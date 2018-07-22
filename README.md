Blue Rings: distributed counters
--------------------------------

The goal is to provide distributed counters usable for billing, with:
- no centralization / single point of failure
- "light" real-time updates
- able to detect and handle network splits

The underlying protocol is currently Axon, a lightweight, native alternative to ZeroMQ on Node.js.

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
- `Value`, an object describing how numerical values are interpreted and transmitted.

The `Value` option defaults to providing EcmaScript integers as numerical values. Since Node.js 10.7.0 BigInt is also supported natively, and can be activated by using `options.Value = BlueRings.bigint` (the default is the equivalent of `options.Value = BlueRings.integer`).

Here is an example for a service storing Big Rationals (arbitrary precision fractions).

```
    bigRat = require 'big-rational'
    big_rational_values =
      deserialize: (t) -> bigRat t
      serialize: (n) -> n.toString()
      add: (n1,n2) -> n1.add n2
      zero: bigRat.zero

    options.Value = big_rational_values
    const ring = BlueRings(options);
```

Methods
-------

`ring.setup_counter(name,expire) →`
`ring.update_counter(name,amount) → [coherent,new_value]`
`ring.get_counter(name) → [coherent,value]`

This implements a counter `name` by adding value `amount`, keeping it until `expire`. Returns a boolean indicating whether the network is coherent (not-split etc.) and a number representing the new value of the counter.

Note that `amount`, `new_value`, `value` are of the type specified by the `Value` option; by default they are native Javascript numbers but might be `BigInt`, `bigRat`, etc.

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
