# TODO
A list of post ideas that would be interesting to research and write about.

## Tracing
To get an overview of a open source project tracing a few simple operations
is quite interesting
* ~~libuv~~ (see `libuv-under-the-hood.md`)
* ~~Redis PUB/SUB~~ (see `tracing-redis-pub-sub.md`)
* NGINX
* Redpanda
* etcd
* Cassandra (started this but it's so coupled to netty it would really have just
been a netty tutorial...)

## Papers
Writing an overview of different papers is usually a useful way to study
(such as `scuttlebutt.md` is an overview of [van Renesse et al](https://www.cs.cornell.edu/home/rvr/papers/flowgossip.pdf)).
* Chord
* Kafka
* Cassandra
* https://pdos.csail.mit.edu/6.824/schedule.html has a good list

## Other
* how docker containers work (cgroups, namespaces etc)
* how redis cluster works
* memory caches - look at nginx `ngx_cacheline_size`, `ngx_pagesize` etc
* how strace works
