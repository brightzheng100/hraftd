hraftd
======
[![Circle CI](https://circleci.com/gh/otoolep/hraftd/tree/master.svg?style=svg)](https://circleci.com/gh/otoolep/hraftd/tree/master)
[![AppVeyor](https://ci.appveyor.com/api/projects/status/github/otoolep/hraftd?branch=master&svg=true)](https://ci.appveyor.com/project/otoolep/hraftd)
[![Go Reference](https://pkg.go.dev/badge/github.com/otoolep/hraftd.svg)](https://pkg.go.dev/github.com/otoolep/hraftd)
[![Go Report Card](https://goreportcard.com/badge/github.com/otoolep/hraftd)](https://goreportcard.com/report/github.com/otoolep/hraftd)

_For background on this project check out this [blog post](http://www.philipotoole.com/building-a-distributed-key-value-store-using-raft/)._

_You should also check out the GopherCon2023 talk "Build Your Own Distributed System Using Go" ([video](https://www.youtube.com/watch?v=8XbxQ1Epi5w), [slides](https://www.philipotoole.com/gophercon2023)), which explains step-by-step how to use the Hashicorp Raft library._

## What is hraftd?

hraftd is a reference example use of the [Hashicorp Raft implementation](https://github.com/hashicorp/raft). [Raft](https://raft.github.io/) is a _distributed consensus protocol_, meaning its purpose is to ensure that a set of nodes -- a cluster -- agree on the state of some arbitrary state machine, even when nodes are vulnerable to failure and network partitions. Distributed consensus is a fundamental concept when it comes to building fault-tolerant systems.

A simple example system like hraftd makes it easy to study the Raft consensus protocol in general, and Hashicorp's Raft implementation in particular. It can be run on Linux, macOS, and Windows.

## Reading and writing keys

The reference implementation is a very simple in-memory key-value store.
You can set a key by sending a request to the HTTP bind address (which defaults to `localhost:11000`):

```bash
curl -XPOST localhost:11000/key -d '{"foo": "bar"}'
```

You can read the value for a key like so:
```bash
curl -XGET localhost:11000/key/foo
```

## Building `hraftd`

*Building hraftd requires Go 1.20 or later. [gvm](https://github.com/moovweb/gvm) is a great tool for installing and managing your versions of Go.*

```bash
git clone https://github.com/brightzheng100/hraftd.git
cd hraftd

go mod tidy
go install
```

## Running `hraftd`

### As single node

#### Starting the node

```bash
./hraftd -id node0 -haddr localhost:10000 -raddr localhost:20000 ./node0
```

#### Playing with it

You can now set a key and read its value back:

```bash
curl -XPOST localhost:10000/key -d '{"user1": "batman"}'
curl -XGET localhost:10000/key/user1
```

### As multi-node cluster

For example, with 3 nodes.

#### Starting the nodes

In console #1:

```bash
./hraftd -id node0 -haddr localhost:10000 -raddr localhost:20000 ./node0
```

In console #2:

```bash
./hraftd -id node1 -haddr localhost:10001 -raddr localhost:20001 -join localhost:10000 ./node1
```

In console #3:

```bash
./hraftd -id node2 -haddr localhost:10002 -raddr localhost:20002 -join localhost:10000 ./node2
```

#### Playing with it

You can now set a key ONLY to the "leader" node.

How to determine whether the node is leader or follower?

```bash
curl -XGET localhost:10001/status | jq .
```

```json
{
  "me": {
    "id": "node1",
    "address": "localhost:20001"
  },
  "leader": {
    "id": "node0",
    "address": "127.0.0.1:20000"
  },
  "followers": [
    {
      "id": "node1",
      "address": "localhost:20001"
    },
    {
      "id": "node2",
      "address": "localhost:20002"
    }
  ]
}
```

Now we know the leader is `node0`, and we can both `set` and `get` key-value to it:

```bash
curl -XPOST localhost:10000/key -d '{"user1": "batman"}'
curl -XGET localhost:10000/key/user1
```

Please note that setting key to non-leader node won't take any effect and it will error out.
For example:

```bash
curl -XPOST localhost:10001/key -d '{"user1": "batman2"}'
```

The service serving on port `10001` will prompt up the error message and the new value won't be set / updated.

```log
2025/05/10 23:08:47 Error when setting key: not leader
```

However, as of the current design, we can retrieve the value from ANY of the nodes, including follower nodes.


#### Tolerating failure

Kill the leader process and watch one of the other nodes be elected leader. The keys are still available for query on the other nodes, and you can set keys on the new leader. Furthermore, when the first node is restarted, it will rejoin the cluster and learn about any updates that occurred while it was down.

A 3-node cluster can tolerate the failure of a single node, but a 5-node cluster can tolerate the failure of two nodes. But 5-node clusters require that the leader contact a larger number of nodes before any change e.g. setting a key's value, can be considered committed.

## Production use of Raft

For a production-grade example of using Hashicorp's Raft implementation, to replicate a SQLite database, check out [rqlite](https://github.com/rqlite/rqlite).


## Known issues / limitations

### Stale reads

Because any node will answer a GET request, and nodes may "fall behind" updates, stale reads are possible. Again, hraftd is a simple program, for the purpose of demonstrating a distributed key-value store. If you are particularly interested in learning more about issue, you should check out [rqlite](https://rqlite.io/). rqlite allows the client to control [read consistency](https://rqlite.io/docs/api/read-consistency/), allowing the client to trade off read-responsiveness and correctness.

Read-consistency support could be ported to hraftd if necessary.

### Leader-forwarding

Automatically forwarding requests to set keys to the current leader is not implemented. The client must always send requests to change a key to the leader or an error will be returned.
