    crypto = require 'crypto'

    union = (set1,set2) ->
      if set1.size is 0
        return set2
      if set2.size is 0
        return set1

      set = new Map [set1...,set2...]
      if set.size is set1.size
        set1
      else
        set

    diff = (set1,set2) ->
      if set2.size is 0
        return set1
      set = new Map set1
      for k from set2.keys()
        if set1.has k
          set.delete k
      if set.size is set1.size
        set1
      else
        set

BlueRing is a store for (opaque) tickets.

Tickets (as transmitted in the protocol) are expected to be {key:value} pairs.

    TICKETS = 'tickets'
    EXPIRE  = 'expire'
    HASH    = 'hash'
    VALUE   = 'value'

    ALGO = 'md5'

    hash_set = (name,tickets) ->
      h = crypto.createHash(ALGO).update name
      for k from tickets.keys()
        h.update k
      h.digest()

    class BlueRing
      constructor: (@Value,@initial) ->
        @store = new Map()

Public operations

      add_counter: (name,expire) ->
        @add_local_tickets name, expire, new Map()

      add_ticket: (name,ticket,expire) ->
        @add_local_tickets name, expire, new Map [ticket]

      get_expire: (name) ->
        @store.get(name)?.get EXPIRE

      get_hash: (name) ->
        @get_local_counter(name)?.get HASH

      get_value: (name) ->
        @get_local_counter(name)?.get VALUE

Private operations

      get_local_counter: (name) ->
        expire = @get_expire(name) ? 0
        if expire > Date.now()
          @store.get name
        else
          @store.delete name
          null

Tool

      add_local_tickets: (name,expire,these_tickets) ->

        L = @store.get(name) ? new Map()

        old_tickets = L.get(TICKETS) ? new Map()
        new_tickets = union old_tickets, these_tickets
        forwarded_tickets = diff new_tickets, old_tickets

No changes, make sure we're consistent.

        if new_tickets is old_tickets
          unless L.has HASH
            L.set HASH,  hash_set name, new_tickets
          unless L.has VALUE
            L.set VALUE, Array.from(new_tickets.values()).reduce @Value.add, @Value.zero

Changes occurred, update!

        else
          L.set TICKETS, new_tickets
          L.set HASH,  hash_set name, new_tickets
          L.set VALUE, Array.from(forwarded_tickets.values()).reduce @Value.add, (L.get VALUE) ? @Value.zero

        if expire? and ((not L.has EXPIRE) or (expire > L.get EXPIRE))
          L.set EXPIRE, expire

        @store.set name, L
        [forwarded_tickets,L]

Message handlers

      on_new_tickets: (name,expire,expected_hash,received_tickets,socket) ->
        [tickets,L] = @add_local_tickets name, expire, received_tickets
        expire = L.get EXPIRE
        hash = L.get HASH

If the hashes do not match after the update (some neighbor(s) is missing some of our tickets),
we send a new message (marked as originating from ourselves) containing all of our tickets.

        if not expected_hash.equals hash
          tickets = diff L.get(TICKETS), tickets
          {name,expire,hash,tickets,source:true}

If the hashes match, we simply forward on behalf of the original sender.

        else
          {name,expire,hash,tickets,source:false}

      enumerate_local_counters: (cb) ->
        @store.forEach (L,name) ->
          expire = L.get EXPIRE
          hash = L.get HASH
          tickets = L.get TICKETS
          cb {name,expire,hash,tickets}

    module.exports = BlueRing
