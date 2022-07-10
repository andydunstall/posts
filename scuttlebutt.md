# Scuttlebutt
This is a high level overview of the Scuttlebutt gossip protocol as described
in [van Renesse et al](https://www.cs.cornell.edu/home/rvr/papers/flowgossip.pdf).

Scuttlebutt is an anti-entropy protocol, which gossip information around
until its made obsolete by newer information.

## Peers
To make things clearer, I'll use the term node to refer to the process, and
peer to refer to the record holding the known state of a node.

Each node has a set of key-value pairs that must be propagated around the
cluster. For the following examples I'll use the states:
```
# Node A
{
	"foo": 32,
	"bar": 82,
	"baz": 104
}

# Node B
{
	"foo": 212,
	"bar": 2,
	"baz": 92
}

...
```

Nodes maintain a set of known peers (including its local peer), such as if
both nodes are synchronised, node A and B will maintain state:
```
{
	"peer_a": {
		"foo": 32,
		"bar": 82,
		"baz": 104
	},
	"peer_b": {
		"foo": 212,
		"bar": 2,
		"baz": 92
	},
	...
}
```

A node can only directly modify its own local peer, and receives updates to
other peers via gossip.

To query and modify the gossip state, a gossip module may have an interface
like:
```
# List all known peers in the cluster.
peers()

# Lookup a value for a given peer and key.
lookup(peer_id, key)

# Update this nodes local state.
update_local(key, value)
```

## Versioning
To detect when values are stale, each entry has an associated version, which
is increased every time the value changes.

This can be stored as a value-version pair:
```
{
	"peer_a": {
		# key: (value, version)
		"foo": (32, 10),
		"bar": (82, 12),
		"baz": (104, 11)
	},
	"peer_b": {
		"foo": (212, 6),
		"bar": (2, 5),
		"baz": (92, 4)
	},
	...
}
```

This means when a node requests an update from another node, it can send its
key-version pairs for all known peers. The remote node can then compare these
versions with its own state and send any updates the sender is missing.

However, sending all key-version pairs for every known peer is expensive.
Instead we can maintain the maximum version seen for every peer, and ensure
whenever a entries version is updated it is greater than the previous maximum
version seen.

```
{
	"peer_a": {
		# Maximum version seen for peer A.
		"version": 12,
		"entries": {
			"foo": (32, 10),
			"bar": (82, 12),
			"baz": (104, 11)
		},
	},
	"peer_b": {
		"version": 6,
		"entries": {
			"foo": (212, 6),
			"bar": (2, 5),
			"baz": (92, 4)
		}
	},
	...
}
```

When a node wants to update its local peer, it can just set the new entries
version to the peers version plus 1, and set the peers version to this new
version.
```
update_local(key, value)
	new_version = state.local_peer + 1
	state.local_peer.entries[key] = (value, new_version)
	state.local_peer.version = new_version
```

## Protocol
Each node periodically choosed another node to gossip with. This can either
be from a set of known nodes, or a set of configured seed nodes to bootstrap
the nodes entry into the cluster.

Assuming in a round of gossip, node A chooses node B to gossip with, A will
request updates it has not yet seen from B, and B will respond with a set
of deltas containing entries A has not seen.

Node B may then also respond with its own request for node A to send any updates
its seen but B has not.

### Request
As described above each node tracks the maximum version seen for each known
peer, so to request updates it can send a map of known peers and versions
to the remote node.

For example node A may send to node B:
```
{
	"peer_a": 12,
	"peer_b": 6
}
```

### Response
When the remote node receives this request, it can check if it knows about
any peers with a greater version than known by A, using a version of `0`
for peers A does not know about.

Node B then constructs a response containing a list of deltas, in the format
`(peer_id, key, value, version)`. This includes all the entries with a higher
version than A knows about for every peer.

Such as in the above example, say node B has state:
```
{
	"peer_a": {
		# Versions for peer A are less than that known by A.
		"foo": (32, 2),
		"bar": (82, 11),
		"baz": (104, 1)
	},
	"peer_b": {
		# Versions for peer B are greater than that known by A.
		# Node A only knows about updates upto version 6 so is missing
		# the latest versions for "bar" and "baz".
		"foo": (212, 6),
		"bar": (81, 7),
		"baz": (17, 8)
	},
	"peer_c": {
		# Node A does not know about peer C, so has an effective version of 0.
		"foo": (501, 2),
		"bar": (62, 3),
		"baz": (18, 4)
	}
}
```

Node B would respond with the set of deltas:
```
[
	("peer_b", "bar", 81, 7),
	("peer_b", "baz", 17, 8),
	("peer_c", "foo", 501, 2),
	("peer_c", "bar", 62, 3),
	("peer_c", "baz", 18, 4)
]
```

Note its worth emphasising nodes may respond with peers data that are not
local to the node, such as here node B is responding with entries from
peer C.

Note in practice it may not always to possible to transfer all deltas due
to message size constraints. It's important if a delta cannot be included, no
deltas with a greater version for that peer can be included, otherwise
the receiver will update the version for the peer even though it's actually
missing updates prior to that version. The easiest way to handle this is to
ensure all deltas are ordered by update for each peer, then it's easy to
truncate if the message reaches capacity.
