    Immutable = require 'immutable'
    HASH_NAME = 'sha256'

BlueRing is a store for (opaque) tickets.

However each ticket must be mapped to a unique, stable representation (strictly speaking the tickets must be sort()able and acceptable as input for a hash function; really only strings satisfy both conditions).

Moreover, there could be ambiguities as to how AMP parses ticket contents, especially if values are expected to be exact.

Therefor, tickets (as transmitted in the protocol) are expected to be `string`.

    hash_set = (tickets) ->
      hash = tickets
        .toList()
        .sort()
        .reduce (acc,val) -> acc.update val
        , crypto.createHash HASH_NAME
      hash.digest 'latin1'

    class BlueRing
      constructor: ->
        @store = Immutable.Map()

Public operations

      add_counter: (name,expire) ->
        @add_local_tickets name, expire, null

      add_ticket: (name,ticket) ->
        @add_local_tickets name, null, [ticket]

      accumulate: (name,cb,initial) ->
        counter = @get_local_counter name
        counter?.get('tickets').reduce cb, initial

Private operations

      get_local_counter: (name) ->
        store = @store
        return unless store.has name
        expire = store.getIn [name,'expire'], 0
        return unless expire > Date.now()
        return store.get name

Tool

      add_local_tickets: (name,expire,tickets) ->

        these_tickets = Immutable.Set tickets

FIXME These are updated from within `update`, making the function impure.

        forwarded_tickets = Immutable.Set()
        hash = null

        @store = @store.update name, ( local = Immutable.Map() ) ->
          old_tickets = local.get 'tickets', Immutable.Set()
          new_tickets = old_tickets.union these_tickets
          forwarded_tickets = new_tickets.substract old_tickets
          local = local.set 'tickets', new_tickets

          local = local.set 'expire', expire if expire?

          return local if local.has('hash') and new_tickets.equals old_tickets

          hash = hash_set new_tickets
          local.set 'hash', hash

        {tickets: forwarded_tickets, hash}

Message handlers

      on_request_tickets: (name) ->
        local = @get_local_counter name
        return unless local?
        expire = local.get 'expire'
        hash = local.get 'hash'
        tickets = local.get 'tickets'
        {name, expire, hash, tickets: tickets.toJS()}

      on_new_tickets: (name,expire,hash,tickets,socket) ->
        local = @get_local_counter name
        return if hash is local?.get 'hash'

        {tickets,hash} = @add_local_tickets name, expire, tickets

        {name, expire, hash, tickets}

      enumerate_local_counters: (cb) ->
        @store.forEach (local,name) ->
          expire = local.get 'expire'
          hash = local.get 'hash'
          tickets = local.get 'tickets'
          cb {name,expire,hash,tickets:tickets.toJS()}

    module.exports = BlueRing
