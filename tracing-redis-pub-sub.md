# Tracing Redis Pub/Sub
This post looks at how Redis pub/sub works under the hood by tracing
`PUBLISH`, `SUBSCRIBE` and `UNSUBSCRIBE` calls on the redis server. I'll
skip the redis client and protocol parsing to keep the post short, skipping
straight to the functions `publishCommand`, `subscribeCommand` and
`unsubscribeCommand` which accept the parsed query.

Documentation on using pub/sub can be found [here](https://redis.io/docs/manual/pubsub/)

I'm working with redis [`7.0.2`](https://github.com/redis/redis/releases/tag/7.0.2), commit [`0583395`](https://github.com/redis/redis/tree/05833959e3875ea10f9b2934dc68daca549c9531), so of course this may change in the future.

In the first version of this post I'll skip pattern matching, shard channels and
cluster mode, to focus only on the simple case (as this is the case I care
about). I may extend later.

## Overview
Unlike more complex systems like Kafka which includes features like message
retention and subscribers requesting an offset to subscribe from, Redis
channels simply receive a published message, and forward to any connected
subscribers. It will not keep track of messages after they have been sent,
so if the subscriber disconnects it will miss any messages published during
that time.

I'll give a brief overview of the operations here, with a more in depth trace
below.

### `SUBSCRIBE`
1. A subscriber attaches to a channel by sending the `SUBSCRIBE` message with
the name of the channel to subscribe to (or optionally multiple channels),
2. Redis adds the channel the set of the clients subscribed channels,
3. Redis adds the client to a set of the channels subscribers,
4. Sends a `subscribe` message back to the client with the channel name

```
subscribeCommand(client, channelName)
  client.subscribers.add(channelName)
  server.channels[channelName].clients.add(client)
  client.addResponse("subscribe", channelName)
```

### `PUBLISH`
1. A publisher sends a `PUBLISH` message with the channel name and the data to
send,
2. Redis looks up the channel in the server,
3. If the channel is not found, it means there are no subscribers so there's
nothing to do,
4. If the channel is found, the published message is appended to the output
buffers of all clients subscribing to that channel, to be sent by the event
loop when the client is next writable,
5. Redis sends a message containing the number the number of receiving
subscribers to the publisher client

Note this means the `PUBLISH` must wait to append the message to the subscribers
connection send buffers before receiving a response (which doesn't guarantee
the data is ever actually sent to the subscriber).

```
publishCommand(client, channelName, data)
  if channelName in server.channels
    for subscriber in server.channels[channelName].clients
      subscriber.addResponse("message", data)
  publisher.addResponse(n_receivers)
```

### `UNSUBSCRIBE`
Unsubscribe simply removes the entries added in `SUBSCRIBE`. When the client
disconnects this will also unsubscribe all channels when clearing the connection
state.

```
unsubscribeCommand(client, channelName)
  client.subscribers.remove(channelName)
  server.channels[channelName].clients.remove(client)
  client.addResponse("unsubscribe", channelName)
```

## Tracing
Note I'm skipping all protocol query parsing and networking - focusing just
on the functions `publishCommand`, `subscribeCommand` and `unsubscribeCommand`
from `src/pubsub.c`, which the above commands map to.

Each command has the same signature of `void command(client *c)`, accepting
only the client as an argument.

`client` is a structure that defines a connected client, that maintains all
state for the client including its file descriptor, the selected DB, incoming
and outgoing buffers, and much more. This also contains a set of subscribed
channels represented as a hash map (`dict`) with `NULL` values.

When the protocol parser parses the query it writes the command arguments
to `client.argv`.

Each argument is of type `robj`, which is a typedef for `struct redisObject`.
This represents all basic redis data types like strings, lists, sets etc,
where the `type` field describes what type `ptr` presents.

It will also use the global `server` object which maintains the shared state
of the redis server. In our case we care about the `server.pubsub_channels`
`dict` mapping channel names to a list of subscribing clients.

```c
// src/server.h

typedef struct client {
    uint64_t id;            /* Client incremental unique ID. */
    uint64_t flags;         /* Client flags: CLIENT_* macros. */
    // ...
    int argc;               /* Num of arguments of current command. */
    robj **argv;            /* Arguments of current command. */
    // ...
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
    // ...
} client;

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;

struct redisServer {
    // ...
    dict *pubsub_channels;  /* Map channels to list of subscribed clients */
    // ...
}
```

### `subscribeCommand`
`SUBSCRIBE` may contain a list of multiple channels, so `subscribeCommand`
simply iterates through this list of channels and subscribes to each one. This
list is in `client.argv` as this contains all query parameters (starting at
index one as `argv[0]` will contain the command name).

This also sets the client flag `CLIENT_PUBSUB` to put the client into subscribe
mode which normal commands will be blocked by `processCommand` (except
`UNSUBSCRIBE`, `PING` and `QUIT`). Once the client has unsubscribed to all
channels this will be unset so the client can return to normal (see
`unsubscribeCommand`).

```c
// src/pubsub.c

void subscribeCommand(client *c) {
    int j;

    // ...

    for (j = 1; j < c->argc; j++)
        pubsubSubscribeChannel(c,c->argv[j],pubSubType);
    c->flags |= CLIENT_PUBSUB;
}
```

To subscribe to a specific channel, `pubsubSubscribeChannel` will add
the channel to `client.pubsub_channels` set (represented as a `dict` with
`NULL` values), and `client` to the servers map of channels to
active subscribers, `server.pubsub_channels`.

The `channel` argument is a string `redisObject` as described above.
The `type` is just a wrapper for a few related functions for a non-sharded
channel (of type `pubSubType`).

```c
// src/pubsub.c

dict* getClientPubSubChannels(client *c) {
    return c->pubsub_channels;
}

pubsubtype pubSubType = {
    .shard = 0,
    .clientPubSubChannels = getClientPubSubChannels,
    .serverPubSubChannels = &server.pubsub_channels,
    // ...
};

int pubsubSubscribeChannel(client *c, robj *channel, pubsubtype type) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* Add the channel to the client -> channels hash table */
    if (dictAdd(type.clientPubSubChannels(c),channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        /* Add the client to the channel -> list of clients hash table */
        de = dictFind(*type.serverPubSubChannels, channel);
        if (de == NULL) {
            clients = listCreate();
            dictAdd(*type.serverPubSubChannels, channel, clients);
            incrRefCount(channel);
        } else {
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    addReplyPubsubSubscribed(c,channel,type);
    return retval;
}
```

### `publishCommand`
`PUBLISH` sends a channel name and message. I'm ignoring pattern matching,
sharded channels and clustering, so the main function here is
`pubsubPublishMessageInternal`, which takes the parameters `channel` containing
the channel name and `message` for the actual data to publish. Both of these are
`robj` strings as described above.

`dictFind` will look up the channel in `server.pubsub_channels` to get the list
of subscribing clients, then iterates though each adding a reply containing
the published data for each subscriber.

`addReply`, defined in `src/networking.c` just appends the message to the client
connections outgoing buffer, which will be sent in the future by the event loop
when the client is next writeable. This means there are no guarantees of
delivery to the client, as the client could disconnect before sending the data,
or the outgoing buffer could be full and drop the message.

Finally a reply containing the number of subscribers is appended to the
publisher clients outgoing buffer to acknowledge the publish.

```c
// src/pubsub.c

void publishCommand(client *c) {
    // ...

    int receivers = pubsubPublishMessageAndPropagateToCluster(c->argv[1],c->argv[2],0);

    // ...

    addReplyLongLong(c,receivers);
}

int pubsubPublishMessageAndPropagateToCluster(robj *channel, robj *message, int sharded) {
    int receivers = pubsubPublishMessage(channel, message, sharded);

    // ...
  
    return receivers;
}

int pubsubPublishMessage(robj *channel, robj *message, int sharded) {
    return pubsubPublishMessageInternal(channel, message, sharded? pubSubShardType : pubSubType);
}

int pubsubPublishMessageInternal(robj *channel, robj *message, pubsubtype type) {
    int receivers = 0;
    dictEntry *de;
    dictIterator *di;
    listNode *ln;
    listIter li;

    /* Send to clients listening for that channel */
    de = dictFind(*type.serverPubSubChannels, channel);
    if (de) {
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;
            addReplyPubsubMessage(c,channel,message,*type.messageBulk);
            updateClientMemUsage(c);
            receivers++;
        }
    }

    // ...

    return receivers;
}
```

### `unsubscribeCommand`
`unsubscribeCommand` either accepts a list of channels, which it will
unsubscribe to each using `pubsubUnsubscribeChannel`, or no channels in which
case it will unsubscribe to all channels with `pubsubUnsubscribeAllChannels`.

Both call `pubsubUnsubscribeChannel` which is simply the reverse of
`pubsubSubscribeChannel` clearing all state from `client.pubsub_channels`
and `server.pubsub_channels`.

If there are no longer any channels in `client.pubsub_channels` the client
flag `CLIENT_PUBSUB` will be cleared to allow it to use commands as normal.

Note when the client disconnects, `clearClientConnectionState` will be called
in `src/networking.c` which clears all that clients subscriptions with
`pubsubUnsubscribeAllChannels`.

```c
void unsubscribeCommand(client *c) {
    if (c->argc == 1) {
        pubsubUnsubscribeAllChannels(c,1);
    } else {
        int j;

        for (j = 1; j < c->argc; j++)
            pubsubUnsubscribeChannel(c,c->argv[j],1,pubSubType);
    }
    if (clientTotalPubSubSubscriptionCount(c) == 0) c->flags &= ~CLIENT_PUBSUB;
}

int pubsubUnsubscribeChannel(client *c, robj *channel, int notify, pubsubtype type) {
    dictEntry *de;
    list *clients;
    listNode *ln;
    int retval = 0;

    /* Remove the channel from the client -> channels hash table */
    incrRefCount(channel); /* channel may be just a pointer to the same object
                            we have in the hash tables. Protect it... */
    if (dictDelete(type.clientPubSubChannels(c),channel) == DICT_OK) {
        retval = 1;
        /* Remove the client from the channel -> clients list hash table */
        de = dictFind(*type.serverPubSubChannels, channel);
        serverAssertWithInfo(c,NULL,de != NULL);
        clients = dictGetVal(de);
        ln = listSearchKey(clients,c);
        serverAssertWithInfo(c,NULL,ln != NULL);
        listDelNode(clients,ln);
        if (listLength(clients) == 0) {
            /* Free the list and associated hash entry at all if this was
             * the latest client, so that it will be possible to abuse
             * Redis PUBSUB creating millions of channels. */
            dictDelete(*type.serverPubSubChannels, channel);

            // ...
        }
    }
    /* Notify the client */
    if (notify) {
        addReplyPubsubUnsubscribed(c,channel,type);
    }
    decrRefCount(channel); /* it is finally safe to release it */
    return retval;
}
```

## Resources
* [Redis: under the hood](https://www.pauladamsmith.com/articles/redis-under-the-hood.html) and [Tracing a GET & SET](https://www.pauladamsmith.com/articles/redis-under-the-hood.html) are great posts by Paul Smith on the Redis internals (though are now quite out of date)
* [Pub/Sub docs](https://redis.io/docs/manual/pubsub/)
