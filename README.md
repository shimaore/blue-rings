Blue Rings: distributed counters
--------------------------------

The goal is to provide distributed counters usable for billing, with:
- no centralization / single point of failure
- "light" real-time updates
- able to detect and handle network splits

API
---

`setup_counter(name,expire) →`
`update_counter(name,amount) → [coherent,new_value]`

Increments counter `name` by adding value `amount`, keeping it until `expire`. Returns a boolean indicating whether the network is coherent (not-split etc.) and a number representing the new value of the counter.

Tickets
-------

Each counter is treated as an independent database; the database contains a series of tickets which represents changes to the counter's value.

Each API request is stored uniquely in the distributed database for counter `name` as

`ticket(timestamp,host,amount[,unicity])`

If the timestamp+host+amount combination might not be enough to ensure unicity, then the `unicity` field might be used to ensure it (e.g. by adding a random value).

Each ticket must be globally unique (tickets with identical contents are considered identical).

Network Protocol
----------------

The protocol uses three packet types:
- `ping()` is used to detect failures in remotes
- `new-tickets(name,expire,hash,array-of-tickets)`
- `request-tickets(name)`
