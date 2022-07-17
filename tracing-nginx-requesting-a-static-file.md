# Tracing NGINX: Requesting A Static File
NGINX is a load balancer, web server and reverse proxy written in C.

This post looks at tracing the simplest NGINX operation, using NGINX as a web
server to serve a static HTML file. I won't look at each function in depth but
will try to give an overview.

I'm working with NGINX [`1.23.0`](http://hg.nginx.org/nginx/rev/release-1.23.0)
([GitHub mirror](https://github.com/nginx/nginx/releases/tag/release-1.23.0)),
so of course this may change in the future.

I'm running NGINX in a Linux Docker container, so when implementations of NGINX
diverge between platforms I'll focus on the Linux implementation.

## Compiling NGINX
Compiling NGINX is really simple. See [instructions](https://nginx.org/en/docs/configure.html)
for a list of options.

```bash
$ ./auto/configure
$ make
```

This outputs the compiled binary at `objs/nginx`. By default this compiles with
the `-g` flag so this works fine with GDB. It also includes optimization with
`-O` but I'm only using GDB to get a stack trace so this isn't an issue.

## Configuration
My configuration to serve a static HTML file is very similar to the default
NGINX config, except I'm adding `daemon off` so NGINX runs in the foreground to
make debugging easier, and redirecting `access_log` and `error_log` to
`/dev/stdout`. This is saved at `conf/nginx.conf` which is the default location.

```
daemon off;

worker_processes  1;

events {
    worker_connections  1024;
}

error_log /dev/stdout info;

http {
    include       mime.types;
    default_type  application/octet-stream;

    access_log /dev/stdout;

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
```

The [`location`](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)
directive tells NGINX to serve static files from the `html/` directory, and to
return `html/index.html` to requests with path `/`. Therefore I also added a
simple HTML file at `html/index.html`.

```html
<!DOCTYPE html>
<html>
  <body>
    <h1>TRACING NGINX</h1>
  </body>
</html>
```

Running `./objs/nginx -p .` works as expected, returning the above HTML file
in request to a `GET http://localhost` request. I'm setting path prefix, `-p`,
to use the current directory for config and static files.

```bash
$ ./objs/nginx -p .
2022/07/16 11:20:07 [notice] 5394#0: using the "epoll" event method
2022/07/16 11:20:07 [notice] 5394#0: nginx/1.23.1
2022/07/16 11:20:07 [notice] 5394#0: built by gcc 11.3.0 (Ubuntu 11.3.0-3ubuntu1)
2022/07/16 11:20:07 [notice] 5394#0: OS: Linux 5.10.76-linuxkit
2022/07/16 11:20:07 [notice] 5394#0: getrlimit(RLIMIT_NOFILE): 1024:1048576
2022/07/16 11:20:07 [notice] 5394#0: start worker processes
2022/07/16 11:20:07 [notice] 5394#0: start worker process 5395
# ...
```

```bash
$ curl -v http://localhost
*   Trying 127.0.0.1:80...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.23.0
< Date: Sun, 17 Jul 2022 11:45:37 GMT
< Content-Type: text/html
< Content-Length: 69
< Last-Modified: Sat, 16 Jul 2022 11:19:41 GMT
< Connection: keep-alive
< ETag: "62d29ecd-45"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<body>
<h1>TRACING NGINX</h1>
</body>
</html>
```

Note though I'm running as a non-root user, I can still bind to port `80` as
since Docker `20.03.0` the default `sysctl` `net.ipv4.ip_unprivileged_port_start`
for containers is set to `0`.

## Strace
As a starting point to try and understand how NGINX listens for
connections, polls for IO and handles my HTTP request, I'm running again with
[strace](https://strace.io/) to view all system calls made by NGINX.

I'm running the same example as above, starting NGINX with strace
(`strace -f ./objs/nginx -p .`) then curl'ing `http://localhost`. Note using
`-f` flag since NGINX starts a worker process on startup where the request is
handled.

I'm leaving out most of the system calls here, and just focusing on the most
important ones to handle the request. The full output from this strace is
at [output/tracing-nginx-requesting-a-static-file/strace](https://github.com/andydunstall/posts/blob/main/output/tracing-nginx-requesting-a-static-file/strace).

### Startup
On Startup NGINX loads the configuration from `./conf/nginx.conf`.
```
uname({sysname="Linux", nodename="abca1d47ff0f", ...}) = 0
openat(AT_FDCWD, "./conf/nginx.conf", O_RDONLY) = 4
newfstatat(4, "", {st_mode=S_IFREG|0644, st_size=446, ...}, AT_EMPTY_PATH) = 0
pread64(4, "daemon off;\n\nworker_processes  1"..., 446, 0) = 446
```

Next it opens the configured TCP listen socket on `0.0.0.0:80` (with
`SO_REUSEADDR` so the TCP socket can bind to this address which is in a
`TIME_WAIT` state rather than having to wait 2 minutes for it to become
available).

```
socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 5
setsockopt(5, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
ioctl(5, FIONBIO, [1])                  = 0
bind(5, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(5, 511)                          = 0
```

Finally it forks a worker to run the event loop. Strace shows it has followed
to this new process.
```
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD
strace: Process 5540 attached
```

### Worker Process
The worker process seems to run an event loop which polls for IO on each
iteration, and handles the incoming requests.

First it sets up the IO polling mechanism. NGINX supports multiple pollers
depending on the host platform. Since I'm on Linux it uses `epoll`.

```
[pid  5540] epoll_create(512)           = 7
```

Next it adds the above TCP listen socket with file descriptor `5` to the poller to
watch for incoming connections (which have type `EPOLLIN`).

```
[pid  5540] epoll_ctl(7, EPOLL_CTL_ADD, 5, {events=EPOLLIN|EPOLLRDHUP, data={u32=1478918160, u64=139965873160208}}) = 0
```

Once the TCP listen socket has been added, it starts the event loop, checking
`epoll_wait` on each iteration.

On the first iteration an event on the TCP listen socket is fired, which triggers
it to accept the incoming TCP connection socket (file descriptor `3`) and add
this to the poller.

```
[pid  5540] epoll_wait(7, [{events=EPOLLIN, data={u32=1478918160, u64=139965873160208}}], 512, -1) = 1
[pid  5540] accept4(5, {sa_family=AF_INET, sin_port=htons(37202), sin_addr=inet_addr("127.0.0.1")}, [112 => 16], SOCK_NONBLOCK) = 3
[pid  5540] epoll_ctl(7, EPOLL_CTL_ADD, 3, {events=EPOLLIN|EPOLLRDHUP|EPOLLET, data={u32=1478918608, u64=139965873160656}}) = 0
```

On the second iteration the TCP connection socket is fired, this time triggering
reading the incoming HTTP request from the connection, starting with
`GET / HTTP/1.1\r\nHost: localhost...`.

```
[pid  5540] epoll_wait(7, [{events=EPOLLIN, data={u32=1478918608, u64=139965873160656}}], 512, 60000) = 1
[pid  5540] recvfrom(3, "GET / HTTP/1.1\r\nHost: localhost\r"..., 1024, 0, NULL, NULL) = 73
```

Since this has a request path `/`, NGINX opens the configured index file at
`html/index.html` (though never reads it into memory). It also requests the file
stats which I'm guessing are to get the file size and last modified date to set
the `Content-Length` and `Last-Modified` headers.

The HTTP header is then written back to the TCP connection socket.

Finally the file itself is written back to the client using `sendfile`. This
copies data between two file descriptors (in this case the open file to the
TCP connection socket) in the kernel. This means the file is never read into the user
space which is much more efficient.

```
[pid  5540] newfstatat(AT_FDCWD, "./html/index.html", {st_mode=S_IFREG|0644, st_size=69, ...}, 0) = 0
[pid  5540] openat(AT_FDCWD, "./html/index.html", O_RDONLY|O_NONBLOCK) = 9
[pid  5540] newfstatat(9, "", {st_mode=S_IFREG|0644, st_size=69, ...}, AT_EMPTY_PATH) = 0
[pid  5540] writev(3, [{iov_base="HTTP/1.1 200 OK\r\nServer: nginx/1"..., iov_len=236}], 1) = 236
[pid  5540] sendfile(3, 9, [0] => [69], 69) = 69
```

After the client closes the connection, another event on the TCP connection
is fired socket, though now reads 0 bytes as the connection has been closed by
the client, so the server also closes the socket.

```
[pid  5540] setsockopt(3, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid  5540] epoll_wait(7, [{events=EPOLLIN|EPOLLRDHUP, data={u32=1478918608, u64=139965873160656}}], 512, 65000) = 1
[pid  5540] recvfrom(3, "", 1024, 0, NULL, NULL) = 0
[pid  5540] write(4, "2022/07/16 13:30:29 [info] 5540#"..., 83) = 83
[pid  5540] close(3)                    = 0
```

## GDB Stack Trace
To get a stack trace of the key operations in handling the HTTP request, first
just `grep`ing the above system calls in the NGINX source, then running again
with GDB with a breakpoint in each of these functions to get a backtrace.

Note as with strace, since NGINX forks a worker process so need to use
`set follow-fork-mode child`.

The main operations I'm looking for are:
* Opening the TCP listen socket `listen()`
* Setting up the event loop IO polling mechanism (`epoll_create()`)
* Adding the TCP listen socket to the event loop (`epoll_ctl()`);
* Accepting a TCP connection socket and adding it to the event loop (`accept()` and `epoll_ctl())
* Sending the file to the client (`sendfile()`)

I've added the stack traces for these functions below, though trimmed a lot
of the output to make it clearer. I've added the full GDB output to
[output/tracing-nginx-requesting-a-static-file/gdb](https://github.com/andydunstall/posts/blob/main/output/tracing-nginx-requesting-a-static-file/gdb).

### `listen`: Create TCP Listen Socket
When the event loop is initialised in the main process all configured listen
sockets are created in `ngx_open_listening_sockets`.
This function creates the socket, binds to the configured address, and listens
for TCP connections. Note as the event loop is handled in the worker process
this is not yet added to the IO poller.
```
#0  ngx_open_listening_sockets at src/core/ngx_connection.c:408
#1  ngx_init_cycle at src/core/ngx_cycle.c:618
#2  main at src/core/nginx.c:292
```

### `epoll_create`: Create IO Poller
When the event loop in the worker process is initialised it sets up the IO
poller. Given I'm on Linux this uses `ngx_epoll_init(ngx_cycle_t *cycle, ngx_msec_t timer)`.

```
#0  ngx_epoll_init at src/event/modules/ngx_epoll_module.c:324
#1  ngx_event_process_init at src/event/ngx_event.c:669
#2  ngx_worker_process_init at src/os/unix/ngx_process_cycle.c:902
#3  ngx_worker_process_cycle at src/os/unix/ngx_process_cycle.c:706
#4  ngx_spawn_process at src/os/unix/ngx_process.c:199
#5  ngx_start_worker_processes at src/os/unix/ngx_process_cycle.c:344
#6  ngx_master_process_cycle at src/os/unix/ngx_process_cycle.c:130
#7  main at src/core/nginx.c:383
```

### `epoll_ctl`: Adding TCP Listen Socket To IO Poller
As with `epoll_create` the TCP listen socket is added to the IO poller with
the worker process event loop is initialised, in `ngx_epoll_add_event`.

```
#0  ngx_epoll_add_event at src/event/modules/ngx_epoll_module.c:580
#1  ngx_event_process_init at src/event/ngx_event.c:908
#2  ngx_worker_process_init at src/os/unix/ngx_process_cycle.c:902
#3  ngx_worker_process_cycle at src/os/unix/ngx_process_cycle.c:706
#4  ngx_spawn_process at src/os/unix/ngx_process.c:199
#5  ngx_start_worker_processes at src/os/unix/ngx_process_cycle.c:344
#6  ngx_master_process_cycle at src/os/unix/ngx_process_cycle.c:130
#7  main at src/core/nginx.c:383
```

### `epoll_ctl`: Adding TCP Connection Socket To IO Poller
Each iteration of the event loop calls `ngx_process_events_and_timers`, which
on Linux polls for IO with `ngx_epoll_process_events`. As a read event on the
TCP listen socket has fired it calls `ngx_event_accept` to accept the connection
then adds it to the poller with `ngx_epoll_add_event`.

```
#0  ngx_epoll_add_event at src/event/modules/ngx_epoll_module.c:580
#1  ngx_handle_read_event at src/event/ngx_event.c:275
#2  ngx_http_init_connection at src/http/ngx_http_request.c:368
#3  ngx_event_accept at src/event/ngx_event_accept.c:313
#4  ngx_epoll_process_events at src/event/modules/ngx_epoll_module.c:901
#5  ngx_process_events_and_timers at src/event/ngx_event.c:248
#6  ngx_worker_process_cycle at src/os/unix/ngx_process_cycle.c:721
#7  ngx_spawn_process at src/os/unix/ngx_process.c:199
#8  ngx_start_worker_processes at src/os/unix/ngx_process_cycle.c:344
#9  ngx_master_process_cycle at src/os/unix/ngx_process_cycle.c:130
#10 main at src/core/nginx.c:383
```

### `sendfile`: Send File Response
The next event loop iteration again calls `ngx_epoll_process_events`, where
a read event on the TCP connection socket fires. This calls `ngx_http_process_request`
to process the request. Given the stack of HTTP handling is 20 frames long I've
skipped most of the functions. Eventually it sends the file itself (after writing
the HTTP headers) with `ngx_linux_sendfile`.

```
#0  ngx_linux_sendfile at src/os/unix/ngx_linux_sendfile_chain.c:251
#1  ngx_linux_sendfile_chain at src/os/unix/ngx_linux_sendfile_chain.c:177
#2  ngx_http_write_filter at src/http/ngx_http_write_filter_module.c:295
#3  ngx_http_chunked_body_filter at src/http/modules/ngx_http_chunked_filter_module.c:115

... (http processing functions)

#21 ngx_http_handler src/http/ngx_http_core_module.c:858
#22 ngx_http_process_request at src/http/ngx_http_request.c:2094
#23 ngx_http_process_request_headers at src/http/ngx_http_request.c:1496
#24 ngx_http_process_request_line at src/http/ngx_http_request.c:1163
#25 ngx_http_wait_request_handler at src/http/ngx_http_request.c:501
#26 ngx_epoll_process_events at src/event/modules/ngx_epoll_module.c:901
#27 ngx_process_events_and_timers at src/event/ngx_event.c:248
#28 ngx_worker_process_cycle at src/os/unix/ngx_process_cycle.c:721
#29 ngx_spawn_process at src/os/unix/ngx_process.c:199
#30 ngx_start_worker_processes at src/os/unix/ngx_process_cycle.c:344
#31 ngx_master_process_cycle at src/os/unix/ngx_process_cycle.c:130
#32 main at src/core/nginx.c:383
```
