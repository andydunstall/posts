# TODO
A list of post ideas that would be fun to research and write about.

## OSS
Looking at how some open source systems work internally (looking at the
low level systems rather than application build ontop of them)

### Seastar
Loads of interesting stuff to learn about here:
* User space networking stack (DPDK, implementing TCP stack etc)
* Shared nothing architecture (why its faster, how it works etc)

### NGINX
* how it keeps cachlines and pagesize for memory caches (`ngx_cacheline_size`, `ngx_pagesize`)

### Linux
Trying to find implementation of some specific functions (probably need to find
more resources about how Linux works generally)
* TCP buffering (SYN_RCVD queue, ESTABLISHED queue, connection in/out bufferes)
* How epoll works

### etcd
* Trace PUT/GET (raft log etc)

### libuv
* How file system operations work (with blocking syscalls in thread pool)

### tokio
* Async runtime
* Scheduler (good chance to learn about OS shedulers too)

### Docker
* how containers work (cgroups, namespaces etc)

## General
* how interprested languages work (js, python...)
* how internet routing works (ISPs, BGP...) - networking tda ch5 has good intro
