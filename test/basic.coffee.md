    {expect} = chai = require 'chai'
    chai.should()

Chai uses JSON.stringify to display content

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'The package', ->
      blue_rings = require '..'
      M = blue_rings.run
      values = blue_rings.integer # or M.bigint
      # values = blue_rings.bigint
      # values.accept::toJSON = -> @toString 10

      port = Math.ceil 2000+10000*Math.random()
      tcp = (p) -> "tcp://127.0.0.1:#{p}"

      it.only 'should accumulate values', ->

        m = M
          host: 'α'
          pub: tcp port++
          values: values

        before -> m.bound
        after -> m.end()

        NAME = 'bear'
        m.setup_counter NAME, Date.now()+4000
        v = m.update_counter NAME, values.accept 3
        v.should.have.length 2
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 3
        v = m.update_counter NAME, values.accept 2
        v.should.have.length 2
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 5
        v = m.get_counter NAME
        v.should.have.length 2
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 5

      it.only 'should accumulate values across two servers', ->
        port1 = port++
        port2 = port++
        m1 = M
          host: 'α'
          pub: tcp port1
          subscribe_to: [
            tcp port2
          ]
          values: values
        after -> m1.end()

        m2 = M
          host: 'β'
          pub: tcp port2
          subscribe_to: [
            tcp port1
          ]
          values: values
        after -> m2.end()

        await Promise.all [m1.bound,m2.bound,m1.connected,m2.connected]

        NAME = 'ant'
        m1.setup_counter NAME, Date.now()+8000
        v= m1.update_counter NAME, values.accept 3
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 3
        await sleep 5
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 3

        v = m1.update_counter NAME, values.accept 7
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 10
        await sleep 5
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 10

        v = m2.update_counter NAME, values.accept 42
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52
        await sleep 5
        v = m1.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52


      it.only 'should accumulate values across two disconnected servers', ->
        port1 = port++
        port2 = port++

        m1 = M
          host: 'α'
          pub: tcp port1
          subscribe_to: [
            tcp port2
          ]
          values: values
        after -> m1.end()

        NAME = 'dog'

        await m1.bound

        m1.setup_counter NAME, Date.now()+8000
        v = m1.update_counter NAME, values.accept 3
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 3

        v = m1.update_counter NAME, values.accept 7
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 10

        m2 = M
          host: 'β'
          pub: tcp port2
          subscribe_to: [
            tcp port1
          ]
          values: values
        after -> m2.end()

        m2.setup_counter NAME, Date.now()+8000
        v = m2.update_counter NAME, values.accept 42
        v.should.have.property 0, false  # Time-sensitive, test might fail
        v.should.have.property 1, values.accept 42

        await Promise.all [m2.bound,m1.connected,m2.connected]

        await sleep 5
        v = m1.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52

        v = m2.update_counter NAME, values.accept 42
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 94
        await sleep 5
        v = m1.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 94
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 94

      it.only 'should accumulate values across three servers (ring)', ->
        port1 = port++
        port2 = port++
        port3 = port++
        m1 = M
          host: 'α'
          pub: tcp port1
          subscribe_to: [
            tcp port3
          ]
          values: values
        after -> m1.end()

        m2 = M
          host: 'β'
          pub: tcp port2
          subscribe_to: [
            tcp port1
          ]
          values: values
        after -> m2.end()

        m3 = M
          host: 'γ'
          pub: tcp port3
          subscribe_to: [
            tcp port2
          ]
          values: values
        after -> m3.end()

        NAME = 'ant'

        await Promise.all [m1.bound,m2.bound,m3.bound,m1.connected,m2.connected,m3.connected]

        m1.setup_counter NAME, Date.now()+8000
        v = m1.update_counter NAME, values.accept 3
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 3
        await sleep 15
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 3
        v = m3.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 3

        v = m1.update_counter NAME, values.accept 7
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 10
        await sleep 15
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 10
        v = m3.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 10

        v = m2.update_counter NAME, values.accept 42
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52
        await sleep 15
        v = m1.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52
        v = m3.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 52

        v = m3.update_counter NAME, values.accept 1
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 53
        await sleep 15
        v = m1.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 53
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 53
        v = m3.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 53


      it.only 'should accumulate values across three disconnected servers (full-mesh)', ->
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
          values: values
        after -> m1.end()

        NAME = 'dog'

        await m1.bound

        m1.setup_counter NAME, Date.now()+8000

        v = m1.update_counter NAME, values.accept 3
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 3

        v = m1.update_counter NAME, values.accept 7
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 10

        m2 = M
          host: 'β'
          pub: tcp port2
          subscribe_to: [
            tcp port1
            tcp port3
          ]
          values: values
        after -> m2.end()

        m2.setup_counter NAME, Date.now()+8000
        v = m2.update_counter NAME, values.accept 42
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 42

        await m2.bound
        await sleep 850

        await sleep 5
        v = m1.get_counter(NAME)
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 52
        v = m2.get_counter(NAME)
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 52

        v = m2.update_counter NAME, values.accept 42
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 94
        await sleep 5
        v = m2.get_counter(NAME)
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 94
        v = m1.get_counter(NAME)
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 94

        v = m1.update_counter NAME, values.accept 1
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 95
        await sleep 5
        v = m2.get_counter(NAME)
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 95
        v = m1.get_counter(NAME)
        v.should.have.property 0, false
        v.should.have.property 1, values.accept 95

        m3 = M
          host: 'β'
          pub: tcp port3
          subscribe_to: [
            tcp port1
            tcp port2
          ]
          values: values
        after -> m3.end()

        m3.setup_counter NAME, Date.now()+8000
        v = m3.update_counter NAME, values.accept 1
        v.should.have.property 0, false # Time-sensitive, might fail
        v.should.have.property 1, values.accept 1

        await Promise.all [m1.connected,m2.connected,m3.bound,m3.connected]

        await sleep 5
        v = m1.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 96
        v = m2.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 96
        v = m3.get_counter(NAME)
        v.should.have.property 0, true
        v.should.have.property 1, values.accept 96

      HOSTS = 'αβγδεζηθικλμνξοπρςστυφχψω'
      load = (timeout,filter,options={},runs=1000,hosts=HOSTS) ->
        i = hosts.split ''
        ports = i.map (_,i1) -> port++
        ms = i.map (host,i1) ->
          p1 = ports[i1]
          cfg = Object.assign (
            host: host
            pub: tcp p1
            subscribe_to: ports.filter( (p2,i2) -> filter i.length,i1,i2,p1,p2 ).map tcp
            values: values
          ), options
          M cfg
        after -> ms.forEach (m) -> m.end()

        sum = 0

        NAME = 'lion'

        await Promise.all ms.map (x) -> x.bound
        console.log 'all bound'
        await Promise.all ms.map (x) -> x.connected
        console.log 'all connected'

        runs = 1000
        for j in [0...runs]
          await sleep 1
          v = Math.ceil 100*Math.random()
          sum += v
          x = Math.ceil ms.length * Math.random()
          x = ms.length-1 if x >= ms.length
          # console.log "Server #{x} #{i[x]} is going to get updated by #{v}"
          ms[x].update_counter NAME, (values.accept v), Date.now()+80000

Note: in full-mesg with long timers we do not need as much

        await sleep timeout

        for m,j in ms
          console.log "Server #{i[j]}", m.get_counter(NAME), sum, m.statistics()
        for m,j in ms
          m.get_counter(NAME).should.have.property 1, values.accept sum

      it 'should accumulate a whole bunch of values across a whole bunch of servers in random connect', ->
        @timeout 3000
        load (l,i1,i2,p1,p2) -> p1 < p2

      it 'should accumulate a whole bunch of values across a whole bunch of servers (ring)', ->
        @timeout 40000
        load ((l,i1,i2) -> i2 is (i1+1)%l),
          forward_delay: 0, flood_delay: 2000, connect_delay: 1500

      it 'should accumulate a whole bunch of values across a whole bunch of servers (counter-rotating rings)', ->
        @timeout 40000
        load ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l),
          forward_delay: 1, flood_delay: 2000, connect_delay: 1500

      it 'should accumulate a whole bunch of values across a whole bunch of servers in extended double-loop', ->
        @timeout 3000
        load ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l or i2 is (i1+7)%l or i2 is (i1-7)%l),
          forward_delay: 1, flood_delay: 2000, connect_delay: 1500

      it 'should accumulate a whole bunch of values across a whole bunch of servers in sparse double-loop', ->
        @timeout 30000
        load ((l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-7)%l),
          forward_delay: 1, flood_delay: 1200, connect_delay: 1500

      it 'should accumulate a whole bunch of values across a whole bunch of servers in star', ->
        @timeout 3000
        load (l,i1,i2) -> if i1 is 0 then i2 isnt 0 else i2 is 0

      it 'should accumulate a whole bunch of values across a whole bunch of servers (dual star)', ->
        @timeout 15000
        load ((l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1)),
          forward_delay: 20, flood_delay: 1200, connect_delay: 1500

      it 'should accumulate a whole bunch of values across a whole bunch of servers (triple star)', ->
        @timeout 15000
        load ((l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1) or (if i1 is 7 then i2 isnt 7 else i2 is 7)),
          forward_delay: 20, flood_delay: 1200, connect_delay: 1500

      it 'should accumulate a whole bunch of values across a whole bunch of servers in half-mesh', ->
        @timeout 3000
        load (l,i1,i2) -> i2 < i1

      it.only 'should accumulate a whole bunch of values across a whole bunch of servers (full-mesh)', ->
        @timeout 80000
        load 1000, ((l,i1,i2) -> i2 isnt i1), forward_delay: 1000, flood_delay: 1200, connect_delay: 1500
