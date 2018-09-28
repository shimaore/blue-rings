    DEFAULT_PORT = 4000
    PING_PACKET = ''

From a protocol perspective, this should use SCTP as the underlying protocol.
However SCTP is not in libuv, and therefor not in Node.js.
There is a Node.js module (relying on raw network access) for SCTP but its author marked it as "not production ready".
There seems to be no SCTP-over-UDP implementations.

For now I'm using Axon but this is highly unsatisfactory since it means we spam the network on every reconnection.

    nextTick = -> new Promise (resolve) -> process.nextTick resolve

    Axon = require '@shimaore/axon'
    {EventEmitter} = require 'events'

    class BlueRingAxon
      constructor: ({@serialize,@deserialize},store,options) ->
        @store = store

Statistics

        @recv = BigInt 0
        @recv_changes = BigInt 0
        @sent = BigInt 0
        @sent_changes = BigInt 0

Options

        {
          @host
          subscribe_to
          @ping_interval = 137
        } = options

        @ev = new EventEmitter

Publisher (sends data out)

        @pub = Axon.socket 'pub'
        @pub.once 'bind', =>
          @ev.emit 'bind'

        @pub.on 'error', (error) -> yes

        stream = @enumerate()

        @queue = new Map()

        ping = =>
          max = 10
          while max-- > 0
            {value,done} = stream.next()
            if not done and value? and not @queue.has value.name
              @queue.set value.name, @packet value

          if @queue.size is 0
            @send PING_PACKET
            return

          content = []
          for [name,msg] from @queue.entries()
            content.push msg
            @queue.delete name
            @sent_changes += BigInt msg.c.length

            if content.length > 10
              @send_data content
              content = []
              await nextTick()

          if content.length > 0
            @send_data content
          return

        @pub.bind options.pub ? DEFAULT_PORT
        @timer = setInterval ping, @ping_interval

Subscribers (receive data)

        @subs = new Map()

Boolean indicating that all subscribers are operational

        @connected = 0

Subscribe to each remote

        subscribe_to?.forEach (o) => @subscribe_to o

        return

      destructor: ->
        @store.destructor()
        @close()
        @ev.removeAllListeners()
        clearInterval @timer
        return

Public operations

      update: (name,expire,op,args) ->
        data = @store.update name, expire, op, args
        return unless data?
        process.nextTick =>
          @send_data @packet data
          @sent_changes += BigInt data.changes.length
        return

      subscribe_to: (o) ->
        sub = Axon.socket 'sub'
        sub.on 'error', (error) -> yes
        sub.connect o

        ping_received = 0
        connected = false # State Machine

Message encoding:
- `ping()` is encoded as `true`
- `new-tickets(name,value,array-of-tickets)` is encoded as :1

        receive = (msg) =>
          # console.log 'receive', @host, msg
          ping_received++
          switch
            when Array.isArray msg
              msg.forEach receive_change

            when typeof msg is 'object'
              @recv++
              receive_change msg

        receive_change = (msg) =>
              @recv_changes += BigInt msg.c.length

              name = msg.n

Avoid processing messages we sent, or messages we forwarded (loop avoidance).
Note: the length of `msg.R` (the path the message already followed) is a good indication of the radius of the network.

              return if msg.s is @host
              return if @host in msg.R

Avoid processing expired messages

              return if msg.e < Date.now()

              changes = msg.c.map @deserialize

              # console.log 'received', @host, name, msg.e, changes, msg.R

The original code called for only forwarding the original message, updated with our values if they were updated.

However this is not very reliable because things are lossy. It's better to send the entire set every time we get an update, which gives a chance to server which are behind to catch up.

              process.nextTick =>
                res = @store.on_send name, msg.e, changes, sub
                @queue.set name, @packet res, msg.R

              return

        monitor = =>
          # console.log 'monitor', ping_received
          if ping_received > 0
            ping_received = 0 # Reset counter
            return if connected

            # Transition from not-connected to connected
            connected = true
            @connected++
            # console.log 'connected', @host, o
            # console.log 'all-connected', @host if @coherent()
            @ev.emit 'connected' if @coherent()

          else
            return if not connected

            # Transition from connected to not-connected
            connected = false
            @connected--
            # console.log 'disconnected', @host
            @ev.emit 'disconnected'
          return

        sub.on 'message', (msg) ->
          try
            receive msg
          catch error
            console.error error
        timer = setInterval monitor, @ping_interval*2.3

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
        @pub.server.unref() # hack, why is this required?
        @subs.forEach ({sock,timer}) ->
          sock.close()
          clearInterval timer
        return

Private

      enumerate: ->
        block_size = 32
        while true
          found = false
          for {name,expire,changes,source} from @store.enumerate_local_values()
            b = 0
            while b < changes.length
              bb = b+block_size
              res = {name,expire,changes:changes[b...bb],source}
              yield res
              found = true
              b = bb

Avoid an infinite loop when we don't have any data yet.

          yield null if not found
        return

      send: (msg,socket=null) ->
        # console.log 'send',msg
        @pub.send msg

      send_data: (msg) ->
        @send msg
        @sent++
        return

      packet: ({name,expire,changes,source},route = []) ->
        n: name
        e: expire
        c: changes.map @serialize
        s: source
        R: [@host,route...]

    module.exports = BlueRingAxon
