    Immutable = require 'immutable'
    crypto = require 'crypto'

BlueRing is a store for (opaque) tickets.

Tickets (as transmitted in the protocol) are expected to be `string`.

    TICKETS = 'tickets'
    EXPIRE  = 'expire'
    HASH    = 'hash'
    VALUE   = 'value'

    ALGO = 'sha1'

    hash_set = (name,tickets) ->
      return Buffer.from tickets.hashCode().toString 36
      ###
      return Buffer.from name + tickets.toArray().sort().join ' '

      h = crypto.createHash(ALGO).update name
      # tickets.toArray().sort().forEach (t) -> h.update t
      h.update tickets.toArray().sort().join ' '
      h.digest()
      ###

    class BlueRing
      constructor: (@compute_value,@initial) ->
        @store = new Map()

Public operations

      add_counter: (name,expire) ->
        @add_local_tickets name, expire, Immutable.Set()

      add_ticket: (name,ticket,expire) ->
        @add_local_tickets name, expire, Immutable.Set [ticket]

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

FIXME These are updated from within `update`, making the function impure.

        forwarded_tickets = Immutable.Set()

        LL = (@store.get(name) ? Immutable.Map()).withMutations (L) =>

          old_tickets = L.get TICKETS, Immutable.Set()
          new_tickets = old_tickets.union these_tickets

No changes, make sure we're consitent.

          if new_tickets.equals old_tickets
            L = L.set HASH,  hash_set name, new_tickets unless L.has HASH
            L = L.set VALUE, @compute_value new_tickets unless L.has VALUE

Changes occurred, update!

          else
            L = L
              .set TICKETS, new_tickets
              .set HASH,  hash_set name, new_tickets
              .set VALUE, @compute_value new_tickets

          forwarded_tickets = new_tickets.subtract old_tickets

          if expire? and ((not L.has EXPIRE) or (expire > L.get EXPIRE))
            L.set EXPIRE, expire

        @store.set name, LL
        [forwarded_tickets,LL]

Message handlers

      on_new_tickets: (name,expire,expected_hash,tickets,socket) ->
        received_tickets = Immutable.Set tickets
        [tickets,local] = @add_local_tickets name, expire, received_tickets
        expire = local.get EXPIRE
        hash = local.get HASH

If the hashes do not match after the update (some neighbor(s) is missing some of our tickets),
we send a new message (marked as originating from ourselves) containing all of our tickets.

        if not expected_hash.equals hash
          tickets = local.get(TICKETS).subtract tickets
          {name,expire,hash,tickets,source:true}

If the hashes match, we simply forward on behalf of the original sender.

        else
          {name,expire,hash,tickets,source:false}

      enumerate_local_counters: (cb) ->
        @store.forEach (local,name) ->
          expire = local.get EXPIRE
          hash = local.get HASH
          tickets = local.get TICKETS
          cb {name,expire,hash,tickets}

    module.exports = BlueRing
