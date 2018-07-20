    Immutable = require 'immutable'

BlueRing is a store for (opaque) tickets.

Tickets (as transmitted in the protocol) are expected to be `string`.

    TICKETS = 'tickets'
    EXPIRE  = 'expire'
    VALUE   = 'value'

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

      get_value: (name) ->
        @store.get(name)?.get VALUE

      get_tickets: (name) ->
        @store.get(name)?.get TICKETS

Private operations

      get_local_counter: (name) ->
        expire = @get_expire() ? 0
        if expire > Date.now()
          @store.get name
        else
          @store.delete name
          null

Tool

      add_local_tickets: (name,expire,these_tickets) ->

FIXME These are updated from within `update`, making the function impure.

        forwarded_tickets = Immutable.Set()

        local = @store.get(name) ? Immutable.Map()

        old_tickets = local.get TICKETS, Immutable.Set()
        new_tickets = old_tickets.union these_tickets
        local = local.set TICKETS, new_tickets

        forwarded_tickets = new_tickets.subtract old_tickets

        if expire? and ((not local.has EXPIRE) or (expire > local.get EXPIRE))
          local = local.set EXPIRE, expire

Optimization: avoid re-computing the value if it hasn't changed.

        unless local.has(VALUE) and new_tickets.equals old_tickets
          value = @compute_value new_tickets
          local = local.set VALUE, value

        @store.set name, local
        [forwarded_tickets,local]

Message handlers

      on_new_tickets: (name,expire,expected_value,tickets,socket) ->
        received_tickets = Immutable.Set tickets
        [tickets,local] = @add_local_tickets name, expire, received_tickets
        expire = local.get EXPIRE
        value = local.get VALUE

If the values do not match (some neighbor(s) is visibly missing some of our tickets),
we send a new message (marked as originating from ourselves) containing all of our tickets.

        if not expected_value is value
          tickets = local.get TICKETS
          {name,expire,value,tickets,source:true}

If the values match, we simply forward on behalf of the original sender.

        else
          {name,expire,value,tickets,source:false}

      enumerate_local_counters: (cb) ->
        @store.forEach (local,name) ->
          expire = local.get EXPIRE
          value = local.get VALUE
          tickets = local.get TICKETS
          cb {name,expire,value,tickets}

    module.exports = BlueRing
