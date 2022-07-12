# libuv: Under the hood
[libuv](https://libuv.org/) is a cross-platform C library for asynchronous IO.

This post looks at how libuv works under the hood, focusing on the event loop,
network IO polling, and timers. I'll skip file IO as its implementation is more
complex since there's no platform specific IO polling so libuv runs blocking
file IO system calls in a thread pool.

libuv provides a good [design overview](http://docs.libuv.org/en/v1.x/design.html)
that describes the event loop and a useful [user guide](http://docs.libuv.org/en/v1.x/guide.html),
so I don't repeat what's already in there, but instead look at the actual
implementation.

I'm working with libuv [v1.44.1](https://github.com/libuv/libuv/releases/tag/v1.44.1),
commit [e8b7eb6](https://github.com/libuv/libuv/tree/e8b7eb6908a847ffbe6ab2eec7428e43a0aa53a2),
so of course this may change in the future.

## Cross-Platform
Since libuv is cross-platform, it has different implementations for different
platforms. The implementation for the UNIX and Windows platforms are
fairly independent, defined in `src/unix/` and `src/win/` respectively.

libuv includes the correct implementation with macros (and the build system will
only build the right one).

```c
// include/uv.h

#if defined(_WIN32)
# include "uv/win.h"
#else
# include "uv/unix.h"
#endif
```

libuv's implementation on UNIX is fairly common across all UNIX platforms.
Though some functions will be implemented using OS specific system calls, such
as the IO polling mechanism (eg `epoll` on Linux, `kqueue` on Darwin/maxOS).

Similar to UNIX vs Windows, the correct implementation is included with macros
in `include/uv/unix.h`.

```c
// include/uv/unix.h

#if defined(__linux__)
# include "uv/linux.h"
// ...
#elif defined(__APPLE__)
# include "uv/darwin.h"
#elif defined(__DragonFly__)       || \
      defined(__FreeBSD__)         || \
      defined(__FreeBSD_kernel__)  || \
      defined(__OpenBSD__)         || \
      defined(__NetBSD__)
# include "uv/bsd.h"
// ...
#endif
```

Where the implementations diverge, I'll focus on the Linux implementation.

## Event Loop
The libuv event loop is key to libuv. It is already well described in the
libuv [design overview](http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)
so I won't repeat what's already covered there. I'll focus on how the network
IO polling and timers work (both polled by the event loop).

### Loop Setup
The loop struct, `uv_loop_s` is defined in `include/uv.h`, though is extended
by platform specific fields (from `include/unix/uv.h` on UNIX).

```c
// include/uv.h (I've extended with fields from include/unix/uv.h)

struct uv_loop_s {
  // ...
  int backend_fd;
  // ...
  void* watcher_queue[2];
  uv__io_t** watchers;
  unsigned int nwatchers;
  unsigned int nfds;
  // ...
  uint64_t time;
  // ...
}
```

Unfortunately libuv seems to have quite bad commenting, so I'll add some notes
on my understanding of some of the key fields.

* `int backend_fd`: The file descriptor of the IO poller (such as the return
value of `epoll_create` on Linux),
* `void* watcher_queue[2]`: A queue of watchers waiting to be added to the
poller in `uv__io_poll`,
* `watchers`: Contains all active watchers (registered IO event and callback),
* `time`: The current time used by the event loop,

A watcher is of type `struct uv__io_s` which includes the registered file
descriptor, callback and events to watch for (`POLLIN` and/or `POLLOUT`).

```c
typedef struct uv__io_s uv__io_t;

struct uv__io_s {
  uv__io_cb cb;
  // ...
  unsigned int pevents; /* Pending event mask i.e. mask at next tick. */
  unsigned int events;  /* Current event mask. */
  int fd;
  // ...
};
```

Each loop is initialised by `uv_loop_init`. This initialises the loop fields to
they're zero-values, updates the current loop time (described below) and
initialises the platform specific IO poller with `uv__platform_loop_init`. 

```c
int uv_loop_init(uv_loop_t* loop) {
  // ...

  uv__update_time(loop);

  // ...

  err = uv__platform_loop_init(loop);
  if (err)
    goto fail_platform_init;

  // ...
}
```

On Linux `uv__platform_loop_init` is defiend in `src/unix/linux-core.c`, which
just initialises the epoll poller with `uv__epoll_init`.

```c
// src/unix/linux-core.c

int uv__platform_loop_init(uv_loop_t* loop) {
  // ...

  return uv__epoll_init(loop);
}
```

```c
// src/unix/epoll.c

int uv__epoll_init(uv_loop_t* loop) {
  int fd;
  fd = epoll_create1(O_CLOEXEC);

  /* epoll_create1() can fail either because it's not implemented (old kernel)
   * or because it doesn't understand the O_CLOEXEC flag.
   */
  if (fd == -1 && (errno == ENOSYS || errno == EINVAL)) {
    fd = epoll_create(256);

    if (fd != -1)
      uv__cloexec(fd, 1);
  }

  loop->backend_fd = fd;
  if (fd == -1)
    return UV__ERR(errno);

  return 0;
}
```

Note `uv_default_loop` just returns a global loop, initialised when its first
accessed.

```c
// src/uv-common.c

uv_loop_t* uv_default_loop(void) {
  if (default_loop_ptr != NULL)
    return default_loop_ptr;

  if (uv_loop_init(&default_loop_struct))
    return NULL;

  default_loop_ptr = &default_loop_struct;
  return default_loop_ptr;
}
```

### Loop Iteration
The event loop iteration is covered in the libuv [design overview](http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)
as mentioned above. The loop iteration itself is defined in `uv_run`, though
the functions called nicely follow the steps in the design overview so don't
really need to be described further.

I'll look at the implementations of the network IO polling (`uv__io_poll`)
and timers (`uv__run_timers`) below.

```c
// src/unix/core.c

int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv__backend_timeout(loop);

    uv__io_poll(loop, timeout);

    /* Run one final update on the provider_idle_time in case uv__io_poll
     * returned because the timeout expired, but no events were received. This
     * call will be ignored if the provider_entry_time was either never set (if
     * the timeout == 0) or was already updated b/c an event was received.
     */
    uv__metrics_update_idle_time(loop);

    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

## Network IO
Network IO all uses non-blocking sockets and is polled by the event loop
with `uv__io_poll`.

Sockets to poll are registered with `uv__io_start`. This adds the watcher
(described above) and the type of event to listen for (either `POLLIN` to
fire when the socket is readable, or `POLLOUT` to fire when the socket it
writeable)

`uv__io_start` will add the watcher to `loop->watcher_queue` and
`loop->watchers`. As described above, `watcher_queue` contains the handles yet
to be added to the poller, and `watchers` contains all active watchers.

`uv__io_poll` is called by the event loop to poll for any events on the
watched sockets. The implementation of this function is platform specific.
It will add the watchers in `loop->watcher_queue` to the poller, wait a
`timeout` for events to occur on the watched file descriptors, then fire the
appropriate watcher callbacks.

Looking at the `epoll` implementation used by Linux, can see this uses
`epoll_ctl(loop->backend_fd, EPOLL_CTL_ADD, ...)` to add watchers to the
event loop, `epoll_wait(loop->backend_fd, ..., timeout)` to wait for events,
then looks up the watcher in `loop->watchers[fd]` to call.

```c
// src/unix/epoll.c

void uv__io_poll(uv_loop_t* loop, int timeout) {
  // ...

  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    assert(w->pevents != 0);
    assert(w->fd >= 0);
    assert(w->fd < (int) loop->nwatchers);

    e.events = w->pevents;
    e.data.fd = w->fd;

    if (w->events == 0)
      op = EPOLL_CTL_ADD;
    else
      op = EPOLL_CTL_MOD;

    if (epoll_ctl(loop->backend_fd, op, w->fd, &e)) {
      // ...
    }

    w->events = w->pevents;
  }

  // ...

  for (;;) {
    // ...

      nfds = epoll_wait(loop->backend_fd,
                        events,
                        ARRAY_SIZE(events),
                        timeout);

    // ...

    for (i = 0; i < nfds; i++) {
      pe = events + i;
      fd = pe->data.fd;

      /* Skip invalidated events, see uv__platform_invalidate_fd */
      if (fd == -1)
        continue;

      // ...

      w = loop->watchers[fd];

      if (w == NULL) {
        epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
        continue;
      }

      /* Give users only events they're interested in. Prevents spurious
       * callbacks when previous callback invocation in this loop has stopped
       * the current watcher. Also, filters out events that users has not
       * requested us to watch.
       */
      pe->events &= w->pevents | POLLERR | POLLHUP;

      // ...

      if (pe->events != 0) {
        /* Run signal watchers last.  This also affects child process watchers
         * because those are implemented in terms of signal watchers.
         */
        if (w == &loop->signal_io_watcher) {
          have_signals = 1;
        } else {
          uv__metrics_update_idle_time(loop);
          w->cb(loop, w, pe->events);
        }

        nevents++;
      }
    }

    // ...
  }
}
```

## Timers
Timers invoke a callback after a certain time has elapsed or at regular
intervals. See [libuv user guide](http://docs.libuv.org/en/v1.x/guide/utilities.html#timers)
for usage.

As shown above, expired timers are run on every iteration of the event loop
in `uv__run_timers`. This section looks at how timers are registered and how
expired timers are found.

From the above user guide, the main timer functions are `uv_timer_start` and
`uv_timer_stop`.

```c
uv_timer_t timer_req;
uv_timer_init(uv_default_loop(), &timer_req);
uv_timer_start(&timer_req, callback, 5000, 2000);

// ...

uv_timer_stop(&timer_req);
```

Timers are implemented in `src/timer.c` which is shared across all platforms
(since libuv handles polling for expired timers itself rather than a platform
system call like Linux's `timer_create`).

`uv_timer_init` will simply zero the handles values to zero.
```c
// src/timer.c

int uv_timer_init(uv_loop_t* loop, uv_timer_t* handle) {
  uv__handle_init(loop, (uv_handle_t*)handle, UV_TIMER);
  handle->timer_cb = NULL;
  handle->timeout = 0;
  handle->repeat = 0;
  return 0;
}
```

`uv_timer_start` registers the handles callback, initial timeout and repeat
timeout.

Each loop maintains a heap `loop.timer_heap` of all active timers. The
[heap](https://en.wikipedia.org/wiki/Heap_(data_structure)) is used as a
priority queue of active timers ordered by timeout. This means you can
iterate through until you find a non-expired handle (since entries are sorted you
know all the rest are also not expired) and easily find the next timeout to
expire (as its the first entry).

```c
// src/timer.c

int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb,
                   uint64_t timeout,
                   uint64_t repeat) {
  // ...

  handle->timer_cb = cb;
  handle->timeout = clamped_timeout;
  handle->repeat = repeat;
  /* start_id is the second index to be compared in timer_less_than() */
  handle->start_id = handle->loop->timer_counter++;

  heap_insert(timer_heap(handle->loop),
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  uv__handle_start(handle);

  return 0;
}
```

The event loop runs `uv_timer_start` on every iteration of
the event loop. Since we're using a priority queue this just iterates the
loop's active timers, calling the callbacks of all expired timers and re-registering
them if repeating.

```c
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min(timer_heap(loop));
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
```

`uv_timer_stop` simply removes the timer from the loop's active timers so it
won't run again.

```c
int uv_timer_stop(uv_timer_t* handle) {
  if (!uv__is_active(handle))
    return 0;

  heap_remove(timer_heap(handle->loop),
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  uv__handle_stop(handle);

  return 0;
}
```
