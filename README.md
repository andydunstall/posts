# Posts
This contains a set of posts from my site at [andrewdunstall.com](https://www.andrewdunstall.com/).
These are written to help my own understanding and practice writing.

### [Tracing NGINX: Requesting A Static File](https://github.com/andydunstall/posts/blob/main/tracing-nginx-requesting-a-static-file.md)
This post looks at tracing the simplest NGINX operation, using NGINX as a web
server to serve a static HTML file. I won't look at each function in depth but
will try to give an overview.

### [libuv: Under the hood](https://github.com/andydunstall/posts/blob/main/libuv-under-the-hood.md)
This post looks at how [libuv](https://libuv.org/) works under the hood, tracing
some of the key functions in the C implementation.

### [Tracing Redis Pub/Sub](https://github.com/andydunstall/posts/blob/main/tracing-redis-pub-sub.md)
This post looks at how Redis pub/sub works under the hood by tracing
`PUBLISH`, `SUBSCRIBE` and `UNSUBSCRIBE` calls on the redis server.

### [Redis scripts do not expire keys atomically](https://ably.com/blog/redis-keys-do-not-expire-atomically)
This was a post published at Ably about one of the first bugs I worked on there
(I wrote a rough first draft describing the bug, then the content team at Ably
wrote the final published article, which was of course much easier to read).

### [Scuttlebutt](https://github.com/andydunstall/posts/blob/main/scuttlebutt.md)
This is a high level overview of the Scuttlebutt gossip protocol as described
in [van Renesse et al](https://www.cs.cornell.edu/home/rvr/papers/flowgossip.pdf).

Scuttlebutt is an anti-entropy protocol, which gossip information around
until its made obsolete by newer information.

### [UDP Implementation Using Raw Sockets in Golang](https://github.com/andydunstall/posts/blob/main/udp-implementation-using-raw-sockets-in-golang.md)
This posts implements a write-only UDP ‘socket’ in Golang using raw sockets,
which provide the application direct access to the IP layer.

### [Implementing TD-Gammon with Keras](https://github.com/andydunstall/posts/blob/main/implementing-td-gammon-with-keras.md)
TD-Gammon is an artificial neural network, trained with TD(λ), that learns to play Backgammon by self-play.

This post implements TD-Gammon 0.0 in Python using Keras and Tensorflow.
