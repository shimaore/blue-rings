Blue Rings: distributed counters and registers
----------------------------------------------

The goal is to provide distributed counters usable for billing, with:
- no centralization / single point of failure
- "light" real-time updates
- able to detect and handle network splits

The underlying protocol is currently Axon, a lightweight, native alternative to ZeroMQ on Node.js.

The module also provides "last-writer wins" text registers.

API
---

```
const BlueRings = require('blue-rings')
const ring = BlueRings(options);
```

The `options` parameter is required; available parameters are:
- `options.host` (required), a string uniquely identifying this host (but normally shorter than the hostname, to reduce memory and bandwidth usage);
- `options.subscribe_to`, an Array containing one or more `tcp://ip:port/` strings suitable to connect to remote Axon/Blue-Rings servers (default remote port is 4000, see below);
- `options.pub`, a port number or `tcp://0.0.0.0:port/` string suitable to bind the local Axon publisher (the default will bind on port 4000 on all interfaces);
- `options.forward_delay`, the number of milliseconds to wait for no updates on a counter before forwarding an update (default: 1000ms);
- `options.connect_delay`, the number of milliseconds to wait for no updates on a counter before sending a full update, when a remote server connects (default: 1500ms);
- `options.Value`, an object describing how numerical values are interpreted and transmitted.

### Constraints on `subscribe_to`

At this time it is recommended that if server A has an entry `subscribe_to` pointing to server B, server B should have an entry `subscribe_to` pointing to server A. In other words subscriptions should be symmetrical. Not doing so would allow you to implement fun topologies (such as a ring) which would also prove very inefficient. This limitation might be removed at some future point if the underlying protocol is changed from Axon to a custom-crafted protocol.

You can also for example implement `receiver-only` schemes by not setting `subscribe_to`; the server will receive updates but not propagate any local changes.

Also note that a server might be subscribed-to other servers and not update counters on its own; in this case it is used only as a message router.

### Timers

The timers `forward_delay` and `connect_delay` are set by default to values adequate for a full-mesh or near-full-mesh setup. If your topology of choice is different, which is probably the case beyond a handful of servers since full-mesh will not scale much, the timers will need to be adapted based on which role you give each server; there are examples in the test suite, and here are some guidelines:
- on a server that does a lot of message forwarding (for example an apex server in a star topology), the `forward_delay` should be kept very low (i.e. 0 or 1ms) to ensure quick propagation of updates;
- in a setup with limited connections between servers, `flood_delay` should be kept relatively low (200ms for example) to ease convergence;
- conversely, on a full-mesh network, all timers should be kept high so that individual updates (which will flood the network) are the primary source of updates;
- obviously the topology is yours to decide; for example you can set up multiple, redundant star topologies on the same network; and even implement a hierarchy, for example redundant stars on each site and full-mesh between the stars' apex servers.

The `Value` option defaults to providing EcmaScript integers as numerical values. Since Node.js 10.7.0 BigInt is also supported natively, and can be activated by using `options.Value = BlueRings.bigint` (the default is the equivalent of `options.Value = BlueRings.integer`).

Here is an example for a service storing Big Integers (arbitrary precision integers).

```
    options.Value = BlueRings.bigint
    const ring = BlueRings(options);
```

Methods
-------

`ring.setup_counter(name,expire) →`
`ring.update_counter(name,amount) → [coherent,new_value]`
`ring.get_counter(name) → [coherent,value]`

This implements a counter `name` by adding value `amount`, keeping it until `expire`. Returns a boolean indicating whether the network is coherent (not-split etc.) and a number representing the new value of the counter.

Note that `amount`, `new_value`, `value` are of the type specified by the `Value` option; by default they are native Javascript numbers but might be `BigInt`, `bigRat`, etc.

`ring.setup_text(name,expire) →`
`ring.update_text (name,text) → [coherent,new_value]`
`ring.get_text(name) → [coherent,value]`

This implements a Last Writer Wins text register, keeping it until `expire`.

`ring.statistics() -> {recv,recv_tickets,sent,sent_tickets}`

`ring.end()` stops all connections and cleans up.

All methods are synchronous.

Promises
--------

`ring.bound` is a Promise that resolves once the server is bound.
`ring.connected` is a Promise that resolves the first time all the remote connections (in `options.subscribe_to`) are successfully established.

Internals
---------

### Tickets

Each counter is treated as an independent database; the database contains a series of tickets which represents changes to the counter's value.

Each API request is stored uniquely in the distributed database for counter `name` as

`ticket(timestamp,host,amount)`

Each ticket must be globally unique: tickets with identical contents are considered identical.

### Network Protocol

The protocol uses two packet types:
- `ping()` is used to detect failures in remotes (and compute the `coherent` boolean flag);
- `new-tickets(name,expire,hash,array-of-tickets)` is used to transmit changes to the database.
