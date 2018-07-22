    DEFAULT_PORT = 4000
    PING_INTERVAL = 150
    PING_PACKET = ''

    EXPIRE = 'expire'
    HASH = 'hash'

From a protocol perspective, this should use SCTP as the underlying protocol.
However SCTP is not in libuv, and therefor not in Node.js.
There is a Node.js module (relying on raw network access) for SCTP but its author marked it as "not production ready".
There seems to be no SCTP-over-UDP implementations.

For now I'm using Axon but this is highly unsatisfactory since it means we spam the network on every reconnection.

    Axon = require 'axon'
    BlueRing = require './store'
    {EventEmitter2} = require 'eventemitter2'

    wrap = (f) ->
      self = this
      (args...) ->
        try
          f.apply(self,args)
        catch error
          console.error error

    class BlueRingAxon extends BlueRing
      constructor: (options) ->
        super options.Value

Statistics

        @recv = BigInt 0
        @sent = BigInt 0

Options

        {
          @host
          subscribe_to
          @forward_delay = 3
          @flood_delay = 500
          @connect_delay = 1
        } = options

Map of name-to-timer to throttle sending messages out (used to implement the delays)

        @__sendall = new Map()

        @ev = new EventEmitter2

Publisher (sends data out)

        @pub = Axon.socket 'pub'
        @pub.once 'bind', =>
          @ev.emit 'bind'
        @pub.on 'connect', =>
          @on_connect()

        ping = =>
          @send PING_PACKET

        @pub.bind options.pub ? DEFAULT_PORT
        @timer = setInterval ping, PING_INTERVAL

Subscribers (receive data)

        @subs = new Map()

Boolean indicating that all subscribers are operational

        @connected = 0

Subscribe to each remote

        subscribe_to?.forEach (o) => @subscribe_to o

        return

Public operations

      add_counter: (name,expire) ->
        [tickets,local] = super name, expire
        expire = local.get EXPIRE
        hash = local.get HASH
        @send_tickets {name,expire,hash,tickets,source:@host}, @pub

      add_ticket: (name,ticket,expire) ->
        [tickets,local] = super name, ticket, expire
        expire = local.get EXPIRE
        hash = local.get HASH
        @send_tickets {name,expire,hash,tickets,source:@host}, @pub

      invalidate: (name) ->
        if @__sendall.has name
          clearTimeout @__sendall.get name
          @__sendall.delete name

      postpone: (name,delay,f) ->
        @__sendall.set name, setTimeout f, delay

      subscribe_to: (o) ->
        sub = Axon.socket 'sub'
        sub.connect o

        ping_received = 0
        connected = false # State Machine

Message encoding:
- `ping()` is encoded as `true`
- `request-tickets(name)` is encoded as `"#{name}"`
- `new-tickets(name,value,array-of-tickets)` is encoded as :1

        deserialize_ticket = ([key,value]) => [key,(@Value.deserialize value)]

        receive = wrap (msg) =>
          switch
            when msg is PING_PACKET
              ping_received++

            when typeof msg is 'object'
              # console.log 'receive', @host, msg
              @recv++

              name = msg.n

              @invalidate name

Avoid processing messages we sent

              return if msg.h is @host
              return if msg.s is @host

              local = @get_local_counter name
              # assert msg.H?.type is 'Buffer'
              remote_hash = Buffer.from msg.H.data
              if local?
                expire = local.get EXPIRE
                hash = local.get HASH
                return if expire is msg.e and hash.equals remote_hash

              tickets = new Map msg.t.map deserialize_ticket

              # console.log 'received', @host, name, msg.e, remote_hash, tickets
              res = @on_new_tickets name, msg.e, remote_hash, tickets, sub

              sendall = =>
                @send_tickets res, null
                return

Forward

              if res.source is false
                return if res.tickets.size is 0
                res.source = msg.s
                @postpone name, @forward_delay, sendall

Broadcast all of our tickets

              else
                res.source = @host
                @postpone name, @flood_delay, sendall

          return

        monitor = =>
          if ping_received > 0
            ping_received = 0 # Reset counter
            return if connected

            # Transition from not-connected to connected
            connected = true
            @connected++
            # console.log 'connected', @host, o
            # console.log 'all-connected', @host if @coherent()
            @ev.emit 'connected' if @coherent()

            @on_connect sub
          else
            return if not connected

            # Transition from connected to not-connected
            connected = false
            @connected--
            # console.log 'disconnected', @host
            @ev.emit 'disconnected'
          return

        sub.on 'message', receive
        sub.on 'connect', => @on_connect()
        timer = setInterval monitor, PING_INTERVAL*2.3

        @subs.set o,
          sock: sub
          timer: timer
          connected: -> connected

      coherent: ->
        @connected is @subs.size

      connected: ->
        v = 0
        v++ for x in @subs when x.connected
        v

      close: ->
        clearInterval @timer
        @pub.close()
        @pub.server.unref() # hack, why is this require?
        @subs.forEach ({sock,timer}) ->
          sock.close()
          clearInterval timer
        return

Private

      on_connect: (sub) ->
        @enumerate_local_counters (res) =>
          {name} = res
          @invalidate name

          res.source = @host
          @postpone name, @connect_delay, => @send_tickets res, sub

      send: (msg,socket) ->
        @pub.send msg

      send_tickets: ({name,expire,hash,tickets,source},socket) ->
        # console.log 'send_tickets', @host, name, expire, hash.toString('hex'), (if tickets.size > 4 then tickets.size else tickets), source

        serialize_ticket = ([key,value]) => [key,(@Value.serialize value)]

        msg =
          n: name
          e: expire
          H: hash
          t: Array.from(tickets.entries()).map serialize_ticket
          s: source
          h: @host
        @send msg, socket

        @sent++
        return

    module.exports = BlueRingAxon
