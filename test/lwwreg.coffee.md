    {expect} = chai = require 'chai'
    chai.should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout
    nextTick = -> new Promise (resolve) -> process.nextTick resolve

    nanosecond = `1000000000n`

    describe 'The package', ->
        blue_rings = require '..'
        M = blue_rings.run

        port = Math.ceil 12000+10000*Math.random()
        tcp = (p) -> "tcp://127.0.0.1:#{p}"

        describe "when using strings", ->
          it 'should accept values', ->

            m = M
              host: 'α'
              pub: tcp port++

            before -> m.bound
            after -> m.end()

            NAME = 'bear'
            m.setup_text NAME, Date.now()+4000

            v = m.update_text NAME, 'server a'
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, 'server a'

            v = m.update_text NAME, 'server b'
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, 'server b'
            await sleep 5
            v = m.get_text NAME
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, 'server b'

            v = m.update_text NAME, 'server c'
            v.should.have.length 2
            v.should.have.property 0, true
            v.should.have.property 1, 'server c'

          it 'should share values across two servers', ->
            port1 = port++
            port2 = port++
            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]
            after -> m1.end()

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
            after -> m2.end()

            await Promise.all [m1.bound,m2.bound,m1.connected,m2.connected]

            NAME = 'ant'
            m1.setup_text NAME, Date.now()+8000
            v= m1.update_text NAME, 'a'
            v.should.have.property 0, true
            v.should.have.property 1, 'a'
            await sleep 5
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'a'

            v = m1.update_text NAME, 'b'
            v.should.have.property 0, true
            v.should.have.property 1, 'b'
            await sleep 5
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'b'

            v = m2.update_text NAME, 'c'
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'

            v = m2.update_text NAME, 'd'
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'

          it 'should share values across two servers (with setup in the middle)', ->
            port1 = port++
            port2 = port++
            m1 = M
              host: 'α'
              pub: tcp port1
              subscribe_to: [
                tcp port2
              ]
            after -> m1.end()

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
            after -> m2.end()

            await Promise.all [m1.bound,m2.bound,m1.connected,m2.connected]

            NAME = 'ant'
            m1.setup_text NAME, Date.now()+8000
            v= m1.update_text NAME, 'a'
            v.should.have.property 0, true
            v.should.have.property 1, 'a'
            await sleep 5
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'a'

            v = m1.update_text NAME, 'b'
            v.should.have.property 0, true
            v.should.have.property 1, 'b'
            await sleep 5
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'b'

            m1.setup_text NAME, Date.now()+8000

            v = m2.update_text NAME, 'c'
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'

            v = m2.update_text NAME, 'd'
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'

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

            m1.setup_text NAME, Date.now()+8000
            v = m1.update_text NAME, 'a'
            v.should.have.property 0, false
            v.should.have.property 1, 'a'

            v = m1.update_text NAME, 'b'
            v.should.have.property 0, false
            v.should.have.property 1, 'b'

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
              ]
            after -> m2.end()

            m2.setup_text NAME, Date.now()+8000
            v = m2.update_text NAME, 'c'
            v.should.have.property 0, false  # Time-sensitive, test might fail
            v.should.have.property 1, 'c'

            await Promise.all [m2.bound,m1.connected,m2.connected]

            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'

            v = m2.update_text NAME, 'd'
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'

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

            m1.setup_text NAME, Date.now()+8000
            v = m1.update_text NAME, 'a'
            v.should.have.property 0, true
            v.should.have.property 1, 'a'
            await sleep 70
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'a'
            v = m3.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'a'

            v = m1.update_text NAME, 'b'
            v.should.have.property 0, true
            v.should.have.property 1, 'b'
            await sleep 70
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'b'
            v = m3.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'b'

            v = m2.update_text NAME, 'c'
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            await sleep 70
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'
            v = m3.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'c'

            v = m3.update_text NAME, 'd'
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            await sleep 70
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'
            v = m3.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'd'


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

            m1.setup_text NAME, Date.now()+8000

            v = m1.update_text NAME, 'a'
            v.should.have.property 0, false
            v.should.have.property 1, 'a'

            v = m1.update_text NAME, 'b'
            v.should.have.property 0, false
            v.should.have.property 1, 'b'

            m2 = M
              host: 'β'
              pub: tcp port2
              subscribe_to: [
                tcp port1
                tcp port3
              ]
            after -> m2.end()

            m2.setup_text NAME, Date.now()+8000
            v = m2.update_text NAME, 'c'
            v.should.have.property 0, false
            v.should.have.property 1, 'c'

            await m2.bound
            await sleep 850

            await sleep 5
            v = m1.get_text(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, 'c'
            v = m2.get_text(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, 'c'

            v = m2.update_text NAME, 'd'
            v.should.have.property 0, false
            v.should.have.property 1, 'd'
            await sleep 5
            v = m2.get_text(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, 'd'
            v = m1.get_text(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, 'd'

            v = m1.update_text NAME, 'e'
            v.should.have.property 0, false
            v.should.have.property 1, 'e'
            await sleep 5
            v = m2.get_text(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, 'e'
            v = m1.get_text(NAME)
            v.should.have.property 0, false
            v.should.have.property 1, 'e'

            m3 = M
              host: 'γ'
              pub: tcp port3
              subscribe_to: [
                tcp port1
                tcp port2
              ]
            after -> m3.end()

            m3.setup_text NAME, Date.now()+8000
            v = m3.update_text NAME, 'f'
            v.should.have.property 0, false # Time-sensitive, might fail
            v.should.have.property 1, 'f'

            await Promise.all [m1.connected,m2.connected,m3.bound,m3.connected]

            await sleep 15 # fails with 5ms for BigInt on my laptop
            v = m1.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'f'
            v = m2.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'f'
            v = m3.get_text(NAME)
            v.should.have.property 0, true
            v.should.have.property 1, 'f'

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

            after ->
              ms.forEach (m) -> m.end()

            console.timeEnd 'establish connections'

            await sleep (options.ping_interval ? 150)*2

            sum = 0

            NAME = 'lion'
            for m in ms
              m.setup_text NAME, Date.now()+80000

            start = Date.now()
            runs = 1000
            for j in [0...runs]
              v = "boo#{HOSTS[Math.floor HOSTS.length*Math.random()]}"
              x = Math.ceil ms.length * Math.random()
              x = ms.length-1 if x >= ms.length
              # console.log "Server #{x} #{i[x]} is going to get updated by #{v}"
              ms[x].update_text NAME, v, Date.now()+80000

            zero = BigInt 0
            prev = [zero,zero]
            t = 500
            last = process.hrtime.bigint()
            for j in [0...timeout/t]
              await sleep t
              msgs = ms.reduce (a,n) ->
                {recv,sent} = n.statistics()
                [a[0]+recv,a[1]+sent]
              , [zero,zero]
              now = process.hrtime.bigint()
              delta = now-last
              console.log "recv #{(msgs[0]-prev[0])*nanosecond/delta} msg/s, ",
                          "sent #{(msgs[1]-prev[1])*nanosecond/delta} msg/s, ",
                          "using #{(delta/BigInt t)/`10000n`-`100n`}% overtime."
              last = now
              prev = msgs

            end = Date.now()
            console.log (Math.ceil (end-start)/ms.length), "ms per process", (Math.ceil timeout/ms.length), "ms wait per process"
            console.log options

            outcome = 0
            for m,j in ms
              x = m.get_text(NAME)
              success = x[1] is v
              outcome++ if success
              console.log "Server #{i[j]}", x, sum, m.statistics(), unless success then '←' else ''

            return Promise.reject new Error "Only synchronized #{outcome} servers out of #{ms.length}" unless outcome is ms.length
            return

Note: on my machine a lot of these tests only work because we complete a `flood` cycle during the test. Regular forwarding is lossy, especially at the pace we inject changes in parallel on a single core, while testing.

          it 'should share a whole bunch of values across a whole bunch of servers (full-mesh)', ->
            @timeout 15000
            load 1000, ((l,i1,i2) -> i2 isnt i1)

          it 'should share a whole bunch of values across a whole bunch of servers (95% full-mesh)', ->
            @timeout 60000
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
            @timeout 40000
            load 12000, ((l,i1,i2,p1,p2) -> Math.random() < 0.95)
