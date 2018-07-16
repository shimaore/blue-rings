    DEFAULT_PORT = 4000
    PING_INTERVAL = 500
    PING_PACKET = ''

From a protocol perspective, this should use SCTP as the underlying protocol.
However SCTP is not in libuv, and therefor not in Node.js.
There is a Node.js module (relying on raw network access) for SCTP but its author marked it as "not production ready".
There seems to be no SCTP-over-UDP implementations.

For now I'm using Axon but this is highly unsatisfactory since it means we spam the network on every reconnection.

    Axon = require 'axon'
    BlueRing = require './store'

    class BlueRingAxon extends BlueRing
      constructor: (options) ->
        super()
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

      coherent: ->
        Object.values(@subs).every (x) -> x.connected

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

    module.exports = BlueRingAxon
