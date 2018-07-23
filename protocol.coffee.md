    DEFAULT_PORT = 4000
    PING_INTERVAL = 150
    PING_PACKET = ''

    EXPIRE = 'expire'

From a protocol perspective, this should use SCTP as the underlying protocol.
However SCTP is not in libuv, and therefor not in Node.js.
There is a Node.js module (relying on raw network access) for SCTP but its author marked it as "not production ready".
There seems to be no SCTP-over-UDP implementations.

For now I'm using Axon but this is highly unsatisfactory since it means we spam the network on every reconnection.

    Axon = require 'axon'
    BlueRing = require './store'
    {EventEmitter} = require 'events'

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
        @recv_changes = BigInt 0
        @sent = BigInt 0
        @sent_changes = BigInt 0

Options

        {
          @host
          subscribe_to
          @forward_delay = 1000
          @connect_delay = 1500
        } = options

Map of name-to-timer to throttle sending messages out (used to implement the delays)

        @__sendall = new Map()

        @ev = new EventEmitter

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
        data = super name, expire
        @send_data data, @pub if data?

      add_amount: (name,amount,expire) ->
        data = super name, amount, expire
        @send_data data, @pub if data?

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
- `new-tickets(name,value,array-of-tickets)` is encoded as :1

        deserialize = ([dir,host,value]) => [dir,host,(@Value.deserialize value)]

        receive = wrap (msg) =>
          switch
            when msg is PING_PACKET
              ping_received++

            when typeof msg is 'object'
              # console.log 'receive', @host, msg
              @recv++
              @recv_changes += BigInt msg.c.length

              name = msg.n

              @invalidate name

Avoid processing messages we sent

              return if msg.h is @host
              return if msg.s is @host

Avoid processing expired messages

              return if msg.e < Date.now()

              changes = msg.c.map deserialize

              # console.log 'received', @host, name, msg.e, changes
              res = @on_new_changes name, msg.e, changes, msg.s, sub
              return unless res?

Forward

              @postpone name, @forward_delay, => @send_data res, null

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
          @postpone name, @connect_delay, =>
            console.log 'on_connect', @host
            @send_data res, sub

      send: (msg,socket) ->
        @pub.send msg

      send_data: ({name,expire,changes,source},socket) ->
        console.log 'send_data', @host, name, expire, changes, source

        serialize = ([dir,host,value]) => [dir,host,(@Value.serialize value)]

        msg =
          n: name
          e: expire
          c: changes.map serialize
          s: source
          h: @host
        @send msg, socket

        @sent++
        @sent_changes += BigInt changes.length
        return

    module.exports = BlueRingAxon
