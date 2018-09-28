    {expect} = chai = require 'chai'
    chai.should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout
    nextTick = -> new Promise (resolve) -> process.nextTick resolve

    nanosecond = `1000000000n`

    describe 'The package', ->
      blue_rings = require '..'
      M = blue_rings.run

      port = Math.ceil 2000+10000*Math.random()
      tcp = (p) -> "tcp://127.0.0.1:#{p}"

      run_with = (type,Value) ->
        describe "when using #{type}", ->
          it 'should accept values', ->

            m = M
              host: 'α'
              pub: tcp port++
              Value: Value

            before -> m.bound
            after -> m.end()

            NAME = 'bear'
            m.setup_counter NAME, Date.now()+4000

            v = m.update_counter NAME, Value.accept 3
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3

            v = m.update_counter NAME, Value.accept 2
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 5
            await sleep 5
            v = m.get_counter NAME
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 5

            v = m.update_counter NAME, Value.accept -10
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept -5

          it 'should accumulate values across two servers', ->
            port1 = port++
            port2 = port++
            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]
              Value: Value
            after -> m1.end()

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
              Value: Value
            after -> m2.end()

            await Promise.all [m1.bound,m2.bound,m1.connected,m2.connected]

            NAME = 'ant'
            m1.setup_counter NAME, Date.now()+8000
            v= m1.update_counter NAME, Value.accept 3
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3
            await sleep 5
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3

            v = m1.update_counter NAME, Value.accept 7
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10
            await sleep 5
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10

            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52

            v = m2.update_counter NAME, Value.accept -30
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 22
            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 22
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 22

          it 'should share values across two servers (with setup in the middle)', ->
            port1 = port++
            port2 = port++
            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]
              Value: Value
            after -> m1.end()

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
              Value: Value
            after -> m2.end()

            await Promise.all [m1.bound,m2.bound,m1.connected,m2.connected]

            NAME = 'ant'
            m1.setup_counter NAME, Date.now()+8000
            v= m1.update_counter NAME, Value.accept 3
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3
            await sleep 5
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3

            v = m1.update_counter NAME, Value.accept 7
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10
            await sleep 5
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10

            m1.setup_counter NAME, Date.now()+8000

            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52

            v = m2.update_counter NAME, Value.accept -30
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 22
            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 22
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 22

          it 'should share values across two disconnected servers', ->
            port1 = port++
            port2 = port++

            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]
              Value: Value
            after -> m1.end()

            NAME = 'dog'

            await m1.bound

            m1.setup_counter NAME, Date.now()+8000
            v = m1.update_counter NAME, Value.accept 3
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 3

            v = m1.update_counter NAME, Value.accept 7
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 10

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
              Value: Value
            after -> m2.end()

            m2.setup_counter NAME, Date.now()+8000
            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, false  # Time-sensitive, test might fail
            v.should.have.property 1, Value.accept 42

            await Promise.all [m2.bound,m1.connected,m2.connected]

            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52

            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 94
            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 94
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 94

          it 'should share values across three servers (ring)', ->
            @timeout 3000

            port1 = port++
            port2 = port++
            port3 = port++
            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port3
              ]
              Value: Value
              ping_interval: 20
            after -> m1.end()

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
              Value: Value
              ping_interval: 20
            after -> m2.end()

            m3 = M
              host: 'γ'
              pub: tcp port3
              subscribe_to: [
                tcp port2
              ]
              Value: Value
              ping_interval: 20
            after -> m3.end()

            NAME = 'ant'

            await Promise.all [m1.bound,m2.bound,m3.bound,m1.connected,m2.connected,m3.connected]

            m1.setup_counter NAME, Date.now()+8000
            v = m1.update_counter NAME, Value.accept 3
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3
            await sleep 70
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3
            v = m3.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 3

            v = m1.update_counter NAME, Value.accept 7
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10
            await sleep 70
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10
            v = m3.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 10

            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            await sleep 70
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52
            v = m3.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 52

            v = m3.update_counter NAME, Value.accept 1
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 53
            await sleep 70
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 53
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 53
            v = m3.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 53


          it 'should share values across three disconnected servers (full-mesh)', ->
            @timeout 3000
            port1 = port++
            port2 = port++
            port3 = port++

            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
                tcp port3
              ]
              Value: Value
            after -> m1.end()

            NAME = 'dog'

            await m1.bound

            m1.setup_counter NAME, Date.now()+8000

            v = m1.update_counter NAME, Value.accept 3
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 3

            v = m1.update_counter NAME, Value.accept 7
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 10

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
                tcp port3
              ]
              Value: Value
            after -> m2.end()

            m2.setup_counter NAME, Date.now()+8000
            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 42

            await m2.bound
            await sleep 850

            await sleep 5
            v = m1.get_counter(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 52
            v = m2.get_counter(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 52

            v = m2.update_counter NAME, Value.accept 42
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 94
            await sleep 5
            v = m2.get_counter(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 94
            v = m1.get_counter(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 94

            v = m1.update_counter NAME, Value.accept 1
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 95
            await sleep 5
            v = m2.get_counter(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 95
            v = m1.get_counter(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, Value.accept 95

            m3 = M
              host: 'γ'
              pub: tcp port3
              subscribe_to: [
                tcp port1
                tcp port2
              ]
              Value: Value
            after -> m3.end()

            m3.setup_counter NAME, Date.now()+8000
            v = m3.update_counter NAME, Value.accept 1
            v.should.have.property 0, false # Time-sensitive, might fail
            v.should.have.property 1, Value.accept 1

            await Promise.all [m1.connected,m2.connected,m3.bound,m3.connected]

            await sleep 15 # fails with 5ms for BigInt on my laptop
            v = m1.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 96
            v = m2.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 96
            v = m3.get_counter(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, Value.accept 96

          HOSTS = 'αβγδεζηθικλμνξοπρςστυφχψω'
          load = (timeout,filter,options={},runs=1000,hosts=HOSTS) ->

            await sleep 1000

            i = hosts.split ''

            console.time 'establish connections'
            ports = i.map -> port++
            ms = i.map (host,i1) ->
              p1 = ports[i1]
              cfg = Object.assign (
                host: host
                pub: tcp p1
                # subscribe_to: ports.filter( (p2,i2) -> filter i.length,i1,i2,p1,p2 ).map tcp
                Value: Value
              ), options
              M cfg
            for m,i1 in ms
              p1 = ports[i1]
              for des in ports.filter( (p2,i2) -> filter i.length,i1,i2,p1,p2 ).map tcp
                await m.subscribe_to des
              await sleep 1

            before ->
              await Promise.all ms.map (x) -> x.bound
              await Promise.all ms.map (x) -> x.connected

            after ->
              ms.forEach (m) -> m.end()

            console.timeEnd 'establish connections'

            await sleep (options.ping_interval ? 150)*2

            console.time 'update'
            sum = 0

            NAME = 'lion'
            for m in ms
              m.setup_counter NAME, Date.now()+80000

            start = Date.now()
            runs = 1000
            for j in [0...runs]
              v = Math.ceil -50+100*Math.random()
              sum += v
              x = Math.ceil ms.length * Math.random()
              x = ms.length-1 if x >= ms.length
              # console.log "Server #{x} #{i[x]} is going to get updated by #{v}"
              ms[x].update_counter NAME, (Value.accept v), Date.now()+80000
            console.timeEnd 'update'

            console.time 'analyze'
            zero = BigInt 0
            prev = [zero,zero]
            t = 500
            last = process.hrtime.bigint()
            for j in [0...timeout/t]
              await sleep t
              now = process.hrtime.bigint()
              delta = now-last
              msgs = ms.reduce (a,n) ->
                {recv,sent} = n.statistics()
                [a[0]+recv,a[1]+sent]
              , [zero,zero]
              console.log "recv #{(msgs[0]-prev[0])*nanosecond/delta} msg/s, ",
                          "sent #{(msgs[1]-prev[1])*nanosecond/delta} msg/s, ",
                          "using #{(delta/BigInt t)/`10000n`-`100n`}% overtime."
              last = now
              prev = msgs

            console.timeEnd 'analyze'
            end = Date.now()
            console.log (Math.ceil (end-start)/ms.length), "ms per process", (Math.ceil timeout/ms.length), "ms wait per process"
            console.log options

            outcome = 0
            for m,j in ms
              x = m.get_counter(NAME)
              success = Value.equals x[1], Value.accept sum
              outcome++ if success
              console.log "Server #{i[j]}", x, sum, m.statistics(), unless success then '←' else ''

            return Promise.reject new Error "Only synchronized #{outcome} servers out of #{ms.length}" unless outcome is ms.length
            return

Note: on my machine a lot of these tests only work because we complete a `flood` cycle during the test. Regular forwarding is lossy, especially at the pace we inject changes in parallel on a single core, while testing.

          it 'should share a whole bunch of values across a whole bunch of servers (full-mesh)', ->
            @timeout 15000
            load 1000, ((l,i1,i2) -> i2 isnt i1)

          it 'should share a whole bunch of values across a whole bunch of servers (95% full-mesh)', ->
            @timeout 90000
            load 12000, ((l,i1,i2) -> i2 isnt i1 and Math.random() < 0.95)

          it 'should share a whole bunch of values across a whole bunch of servers (star)', ->
            @timeout 40000
            load 7000, ((l,i1,i2) -> if i1 is 0 then i2 isnt 0 else i2 is 0)

          it 'should share a whole bunch of values across a whole bunch of servers (dual star)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1))

          it 'should share a whole bunch of values across a whole bunch of servers (triple star)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1) or (if i1 is 7 then i2 isnt 7 else i2 is 7))

          it 'should share a whole bunch of values across a whole bunch of servers (ring)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l),
              ping_interval: 20

          it 'should share a whole bunch of values across a whole bunch of servers (counter-rotating rings)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l),
              ping_interval: 20

          it 'should share a whole bunch of values across a whole bunch of servers (counter-rotating rings plus one ring)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l or i2 is (i1+7)%l or i2 is (i1-7)%l),
              ping_interval: 20

          it 'should share a whole bunch of values across a whole bunch of servers (sparse counter-rotating rings)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-7)%l)

          it.skip 'should share a whole bunch of values across a whole bunch of servers (half-mesh)', ->
            @timeout 40000
            load 5000, ((l,i1,i2) -> i2 < i1)

          it 'should share a whole bunch of values across a whole bunch of servers (95% connect, including self)', ->
            @timeout 90000
            load 12000, ((l,i1,i2,p1,p2) -> Math.random() < 0.95)

      run_with 'native integers', blue_rings.integer

Chai uses JSON.stringify to display content on errors.

      Value = blue_rings.bigint
      Value.accept::toJSON = -> @toString 10
      run_with 'native BigInt', Value
