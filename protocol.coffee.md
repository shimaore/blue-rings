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

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

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
          @forward_delay =     97
          @connect_delay =   1511
          @flood_interval = 59141
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

        flood = =>
          @on_connect()

        @pub.bind options.pub ? DEFAULT_PORT
        @timer = setInterval ping, PING_INTERVAL
        @flood = setInterval flood, @flood_interval unless @flood_interval < PING_INTERVAL or @flood_interval is Infinity

Subscribers (receive data)

        @subs = new Map()

Boolean indicating that all subscribers are operational

        @connected = 0

Subscribe to each remote

        subscribe_to?.forEach (o) => @subscribe_to o

        return

      destructor: ->
        super()
        @close()
        @ev.removeAllListeners()
        clearInterval @timer
        clearInterval @flood
        return

Public operations

      add_counter: (name,expire) ->
        data = super name, expire
        @send_data data, [], @pub if data?

      add_amount: (name,amount,expire) ->
        data = super name, amount, expire
        @send_data data, [], @pub if data?

      postpone: (name,delay,f) ->
        if @__sendall.has name
          clearTimeout @__sendall.get name
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

Avoid processing messages we sent, or messages we forwarded (loop avoidance).
Note: the length of `msg.R` (the path the message already followed) is a good indication of the radius of the network.

              return if msg.s is @host
              return if @host in msg.R

Avoid processing expired messages

              return if msg.e < Date.now()

              changes = msg.c.map deserialize

              # console.log 'received', @host, name, msg.e, changes, msg.R

The original code called for only forwarding the original message, updated with our values if they were updated.

However this is not very reliable because things are lossy. It's better to send the entire set every time we get an update, which gives a chance to server which are behind to catch up.

              res = @on_send name, msg.e, changes, sub
              @postpone name, @forward_delay, =>
                @send_data res, msg.R, null

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
          @postpone name, @connect_delay, =>
            @send_data res, [], sub

Rate-limit to 1000 per second.

          await sleep 1
          return
        return

      send: (msg,socket) ->
        @pub.send msg

      send_data: ({name,expire,changes,source},route,socket) ->
        # console.log 'send_data', @host, name, expire, changes, source, route

        serialize = ([dir,host,value]) => [dir,host,(@Value.serialize value)]

        msg =
          n: name
          e: expire
          c: changes.map serialize
          s: source
          R: [@host,route...]
        @send msg, socket

        @sent++
        @sent_changes += BigInt changes.length
        return

    module.exports = BlueRingAxon
