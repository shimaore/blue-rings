    {expect} = chai = require 'chai'
    chai.should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout
    nextTick = -> new Promise (resolve) -> process.nextTick resolve

    nanosecond = `1000000000n`

    describe 'The package', ->
      blue_rings = require '..'
      M = blue_rings.run

      port = Math.ceil 22000+10000*Math.random()
      tcp = (p) -> "tcp://127.0.0.1:#{p}"

      describe "when using sets", ->
          it 'should accept values', ->

            m = M
              host: 'α'
              pub: tcp port++

            before -> m.bound
            after -> m.end()

            NAME = 'bear'
            m.setup_set NAME, Date.now()+4000

            v = m.add NAME, 'server a'
            v = m.has NAME, 'server a'
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, true
            v = m.has NAME, 'server b'
            v.should.have.property 1, false
            v = m.value NAME
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            v[1].should.have.property 0, 'server a'

            v = m.add NAME, 'server b'
            (m.has NAME, 'server a').should.have.property 1, true
            (m.has NAME, 'server b').should.have.property 1, true
            await sleep 5
            v = m.value NAME
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 2

            v = m.remove NAME, 'server b'
            (m.has NAME, 'server a').should.have.property 1, true
            (m.has NAME, 'server b').should.have.property 1, false

          it 'should share values across two servers', ->
            port1 = port++
            port2 = port++
            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]

            await Promise.all [m1.bound,m2.bound,m1.connected,m2.connected]

            NAME = 'ant'
            m1.setup_set NAME, Date.now()+8000
            m1.add NAME, 'a'
            v = m1.get_value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            v[1].should.have.property 0, 'a'
            await sleep 5
            v = m2.get_value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            v[1].should.have.property 0, 'a'

            m1.add NAME, 'b'
            v = m1.get_value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 2
            await sleep 5
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 2

            m2.remove NAME, 'c'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 2
            await sleep 5
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 2
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 2

            m2.remove NAME, 'b'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            await sleep 5
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1

            m2.add NAME, 'b'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            await sleep 5
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            v[1].should.have.property 0, 'a'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property 1
            v[1].should.have.length 1
            v[1].should.have.property 0, 'a'

            m1.end()
            m2.end()

          it 'should share values across two disconnected servers', ->
            port1 = port++
            port2 = port++

            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]
            after -> m1.end()

            NAME = 'dog'

            await m1.bound

            m1.setup_set NAME, Date.now()+8000
            m1.add NAME, 'a'
            v = m1.has NAME, 'a'
            v.should.have.property 0, false
            v.should.have.property 1, true

            m1.add NAME, 'b'
            v = m1.has NAME, 'b'
            v.should.have.property 0, false
            v.should.have.property 1, true

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
            after -> m2.end()

            m2.setup_set NAME, Date.now()+8000
            m2.add NAME, 'c'
            v = m1.has NAME, 'c'
            v.should.have.property 0, false  # Time-sensitive, test might fail
            v.should.have.property 1, false  # Time-sensitive, test might fail
            v = m2.has NAME, 'c'
            v.should.have.property 0, false  # Time-sensitive, test might fail
            v.should.have.property 1, true

            await Promise.all [m2.bound,m1.connected,m2.connected]

            await sleep 5
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 3
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 3

            v = m2.add NAME, 'd'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 4
            await sleep 5
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 4
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 4

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
              ping_interval: 20
            after -> m1.end()

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
              ping_interval: 20
            after -> m2.end()

            m3 = M
              host: 'γ'
              pub: tcp port3
              subscribe_to: [
                tcp port2
              ]
              ping_interval: 20
            after -> m3.end()

            NAME = 'ant'

            await Promise.all [m1.bound,m2.bound,m3.bound,m1.connected,m2.connected,m3.connected]

            m1.setup_set NAME, Date.now()+8000
            m1.add NAME, 'a'
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length(1).property 0, 'a'
            await sleep 70
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length(1).property 0, 'a'
            v = m3.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length(1).property 0, 'a'

            m1.add NAME, 'b'
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2
            await sleep 70
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2
            v = m3.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2

            m2.add NAME, 'c'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 3
            await sleep 70
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 3
            v = m3.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 3

            m3.remove NAME, 'a'
            v = m3.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2
            await sleep 70
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2
            v = m3.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length 2

            m2.remove NAME, 'b'
            v = m2.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length(1).property 0, 'c'
            await sleep 70
            v = m1.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length(1).property 0, 'c'
            v = m3.value(NAME)
            v.should.have.property 0, true
            v.should.have.property(1).length(1).property 0, 'c'


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
            after -> m1.end()

            NAME = 'dog'

            await m1.bound

            m1.setup_set NAME, Date.now()+8000

            m1.add NAME, 'a'
            m1.add NAME, 'b'

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
                tcp port3
              ]
            after -> m2.end()

            m2.setup_set NAME, Date.now()+8000
            m2.add NAME, 'c'

            await m2.bound
            await sleep 850

            await sleep 5
            v = m1.has NAME, 'c'
            v.should.have.property 0, false
            v.should.have.property 1, true
            v = m2.has NAME, 'c'
            v.should.have.property 0, false
            v.should.have.property 1, true

            m2.add NAME, 'd'
            await sleep 5
            v = m2.has NAME, 'd'
            v.should.have.property 0, false
            v.should.have.property 1, true
            v = m1.has NAME, 'd'
            v.should.have.property 0, false
            v.should.have.property 1, true

            m1.add NAME, 'e'
            await sleep 5
            v = m2.has NAME, 'e'
            v.should.have.property 0, false
            v.should.have.property 1, true
            v = m1.has NAME, 'e'
            v.should.have.property 0, false
            v.should.have.property 1, true

            m3 = M
              host: 'γ'
              pub: tcp port3
              subscribe_to: [
                tcp port1
                tcp port2
              ]
            after -> m3.end()

            m3.setup_set NAME, Date.now()+8000
            m3.add NAME, 'f'
            v = m3.has NAME, 'f'
            v.should.have.property 0, false # Time-sensitive, might fail
            v.should.have.property 1, true
            v = m3.value NAME
            v.should.have.property 0, false # Time-sensitive, might fail
            v.should.have.property(1).length(1).property 0, 'f'

            await Promise.all [m1.connected,m2.connected,m3.bound,m3.connected]

            await sleep 15
            v = m1.has NAME, 'f'
            v.should.have.property 0, true
            v.should.have.property 1, true
            v = m2.has NAME, 'f'
            v.should.have.property 0, true
            v.should.have.property 1, true
            v = m3.has NAME, 'f'
            v.should.have.property 0, true
            v.should.have.property 1, true

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

            console.timeEnd 'establish connections'

            await sleep (options.ping_interval ? 137)*2

            console.time 'update'

            NAME = 'lion'
            for m in ms
              m.setup_set NAME, Date.now()+80000

            start = Date.now()
            runs = 1000
            for j in [0...runs]
              v = "Q#{[0,1].map(-> HOSTS[Math.floor HOSTS.length*Math.random()]).join ''}"
              x = Math.ceil ms.length * Math.random()
              x = ms.length-1 if x >= ms.length
              what = Math.random() < 0.75
              # console.log "Server #{x} #{i[x]} is going to get updated by #{v} for #{if what then 'add' else 'del'}"
              if what
                ms[x].add NAME, v, Date.now()+80000
                await nextTick()
              else
                ms[x].remove NAME, v, Date.now()+80000
                await nextTick()
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

            boom = (r) -> r.sort().join ','

Note: I don't think there is a reliable way to test both add and delete _and_ satisfactorily compute the expected value, since it must be based on knowledge at the source at the time of insertion (so it's not blackbox).
Therefor we merely make sure that all servers have the same data.

            expected_string = null
            for m,j in ms
              x = m.value(NAME)
              xb = boom x[1]
              expected_string ?= xb
              success = expected_string is xb
              outcome++ if success
              # console.log "Server #{i[j]}", x[1].length, m.statistics(), unless success then '←' else ''

            ms.forEach (m) -> m.end()

            return Promise.reject new Error "Only synchronized #{outcome} servers out of #{ms.length}" unless outcome is ms.length
            return

Note: on my machine a lot of these tests only work because we complete a `flood` cycle during the test. Regular forwarding is lossy, especially at the pace we inject changes in parallel on a single core, while testing.

          it 'should share a whole bunch of values across a whole bunch of servers (full-mesh)', ->
            @timeout 60000
            load 2000, ((l,i1,i2) -> i2 isnt i1)

          it 'should share a whole bunch of values across a whole bunch of servers (95% full-mesh)', ->
            @timeout 90000
            load 16000, ((l,i1,i2) -> i2 isnt i1 and Math.random() < 0.95)

          it 'should share a whole bunch of values across a whole bunch of servers (star)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> if i1 is 0 then i2 isnt 0 else i2 is 0)

          it 'should share a whole bunch of values across a whole bunch of servers (dual star)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1))

          it 'should share a whole bunch of values across a whole bunch of servers (triple star)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1) or (if i1 is 7 then i2 isnt 7 else i2 is 7))

          it 'should share a whole bunch of values across a whole bunch of servers (ring)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l),
              ping_interval: 20

          it 'should share a whole bunch of values across a whole bunch of servers (counter-rotating rings)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l),
              ping_interval: 20

          it 'should share a whole bunch of values across a whole bunch of servers (counter-rotating rings plus one ring)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l or i2 is (i1+7)%l or i2 is (i1-7)%l),
              ping_interval: 20

          it 'should share a whole bunch of values across a whole bunch of servers (sparse counter-rotating rings)', ->
            @timeout 60000
            load 5000, ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-7)%l),
              ping_interval: 20

          it.skip 'should share a whole bunch of values across a whole bunch of servers (half-mesh)', ->
            @timeout 60000
            load 40000, ((l,i1,i2) -> i2 < i1)

          it 'should share a whole bunch of values across a whole bunch of servers (95% connect, including self)', ->
            @timeout 60000
            load 15000, ((l,i1,i2,p1,p2) -> Math.random() < 0.95),
              ping_interval: 20
