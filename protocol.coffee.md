    Immutable = require 'immutable'
    HASH_NAME = 'sha256'

BlueRing is a store for (opaque) tickets

    hash_set = (tickets) ->
      hash = tickets
        .toList()
        .map JSON.stringify # FIXME probably not stable enough for sort() and hash()
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

    DEFAULT_PORT = 4000
    PING_INTERVAL = 500
    PING_PACKET = ''

From a protocol perspective, this should use SCTP as the underlying protocol.
However SCTP is not in libuv, and therefor not in Node.js.
There is a Node.js module (relying on raw network access) for SCTP but its author marked it as "not production ready".
There seems to be no SCTP-over-UDP implementations.

For now I'm using Axon but this is highly unsatisfactory since it means we spam the network on every reconnection.

    Axon = require 'axon'

    class BlueRingAxon extends BlueRing
      constructor: (options) ->
        @pub = Axon.socket 'pub'
        @subs = {}

        ping = => @send PING_PACKET

        @pub.bind options.pub ? DEFAULT_PORT
        @timer = setInterval ping, PING_INTERVAL

        options.subscribe_to?.forEach (o) => @subscribe_to o

      subscribe_to: (o) ->
        sub = Axon.socket 'sub'
        sub.connect o

        ping_received = 0
        connected = false # State Machine

Message encoding:
- `ping()` is encoded as `true`
- `request-tickets(name)` is encoded as `"#{name}"`
- `new-tickets(name,hash,array-of-tickets)` is encoded as :1

        receive = (msg) =>
          switch
            when msg is PING_PACKET
              ping_received++
            when typeof msg is 'string'
              res = @on_request_tickets msg, sub
              @send_tickets res, socket if res?
            when typeof msg is 'object'
              res = @on_new_tickets msg.n, msg.e, msg.h, msg.t, sub
              @send_tickets res, null if res? # all but socket
          return

        monitor = ->
          if ping_received > 0
            ping_received = 0 # Reset counter
            return if connected
            # Transition from not-connected to connected
            connected = true
            @enumerate_local_counters (res) => @send_tickets res, sub
          else
            connected = false
          return

        sub.on 'message', receive
        timer = setInterval monitor, PING_INTERVAL*2.3

        @subs[o] =
          sock: sub
          timer: timer
          connected: -> connected

      disconnect: ->
        clearInterval @timer
        @pub.close()
        @subs.forEach ({sock,timer}) ->
          sock.close()
          clearInterval timer

      send: (msg,socket) ->
        @pub.send msg

      send_tickets: ({name,expire,hash,tickets},socket) ->
        msg =
          n: name
          e: expire
          h: hash
          t: tickets
        @send msg, socket

    module.exports = (options) ->

      service = new BlueRingAxon options

Public API

      setup_counter: (name,expire) ->
        service.add_counter name, expire

      update_counter: (name,amount) ->
        ticket =
          timestamp: process.hrtime()
          host: options.host
          amount: amount

        service.add_ticket name, ticket

        return [coherent,new_value]

      get_counter: (name) ->
        return [coherent,value]
