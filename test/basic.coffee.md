    {expect} = chai = require 'chai'
    chai.should()

Chai uses JSON.stringify to display content

    BigInt::toJSON = -> @toString 10

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'The package', ->
      M = require '..'

      port = Math.ceil 2000+10000*Math.random()

      it 'should accumulate values', ->

        m = M
          host: 'α'
          pub: port++

        after -> m.end()

        NAME = 'bear'
        m.setup_counter NAME, Date.now()+4000
        m.update_counter NAME, BigInt 3
        m.update_counter NAME, BigInt 2
        v = m.get_counter NAME
        v.should.have.length 2
        v.should.have.property 0, true
        v.should.have.property 1, BigInt 5

      it 'should accumulate values across two servers', ->
        port1 = port++
        port2 = port++
        m1 = M
          host: 'α'
          pub: port1
          subscribe_to: [
            "tcp://127.0.0.1:#{port2}"
          ]
        after -> m1.end()

        m2 = M
          host: 'β'
          pub: port2
          subscribe_to: [
            "tcp://127.0.0.1:#{port1}"
          ]
        after -> m2.end()

        await sleep 350

        NAME = 'ant'
        m1.setup_counter NAME, Date.now()+8000
        m1.update_counter NAME, BigInt 3
        m1.get_counter(NAME).should.have.property 1, BigInt 3
        await sleep 30
        m2.get_counter(NAME).should.have.property 1, BigInt 3

        m1.update_counter NAME, BigInt 7
        m1.get_counter(NAME).should.have.property 1, BigInt 10
        await sleep 30
        m2.get_counter(NAME).should.have.property 1, BigInt 10

        m2.update_counter NAME, BigInt 42
        m2.get_counter(NAME).should.have.property 1, BigInt 52
        await sleep 30
        m1.get_counter(NAME).should.have.property 1, BigInt 52


      it 'should accumulate values across two disconnected servers', ->
        port1 = port++
        port2 = port++

        m1 = M
          host: 'α'
          pub: port1
          subscribe_to: [
            "tcp://127.0.0.1:#{port2}"
          ]
        after -> m1.end()

        NAME = 'dog'
        m1.setup_counter NAME, Date.now()+8000
        m1.update_counter NAME, BigInt 3
        m1.get_counter(NAME).should.have.property 1, BigInt 3

        m1.update_counter NAME, BigInt 7
        m1.get_counter(NAME).should.have.property 1, BigInt 10

        m2 = M
          host: 'β'
          pub: port2
          subscribe_to: [
            "tcp://127.0.0.1:#{port1}"
          ]
        after -> m2.end()

        m2.setup_counter NAME, Date.now()+8000
        m2.update_counter NAME, BigInt 42
        m2.get_counter(NAME).should.have.property 1, BigInt 42
        await sleep 350
        m1.get_counter(NAME).should.have.property 1, BigInt 52
        m2.get_counter(NAME).should.have.property 1, BigInt 52

        m2.update_counter NAME, BigInt 42
        await sleep 30
        m2.get_counter(NAME).should.have.property 1, BigInt 94
        m1.get_counter(NAME).should.have.property 1, BigInt 94

      it 'should accumulate values across three servers in a loop', ->
        port1 = port++
        port2 = port++
        port3 = port++
        m1 = M
          host: 'α'
          pub: port1
          subscribe_to: [
            "tcp://127.0.0.1:#{port3}"
          ]
        after -> m1.end()

        m2 = M
          host: 'β'
          pub: port2
          subscribe_to: [
            "tcp://127.0.0.1:#{port1}"
          ]
        after -> m2.end()

        m3 = M
          host: 'γ'
          pub: port3
          subscribe_to: [
            "tcp://127.0.0.1:#{port2}"
          ]
        after -> m3.end()

        await sleep 350

        NAME = 'ant'
        m1.setup_counter NAME, Date.now()+8000
        m1.update_counter NAME, BigInt 3
        m1.get_counter(NAME).should.have.property 1, BigInt 3
        await sleep 30
        m2.get_counter(NAME).should.have.property 1, BigInt 3
        m3.get_counter(NAME).should.have.property 1, BigInt 3

        m1.update_counter NAME, BigInt 7
        m1.get_counter(NAME).should.have.property 1, BigInt 10
        await sleep 30
        m2.get_counter(NAME).should.have.property 1, BigInt 10
        m3.get_counter(NAME).should.have.property 1, BigInt 10

        m2.update_counter NAME, BigInt 42
        m2.get_counter(NAME).should.have.property 1, BigInt 52
        await sleep 30
        m1.get_counter(NAME).should.have.property 1, BigInt 52
        m3.get_counter(NAME).should.have.property 1, BigInt 52

        m3.update_counter NAME, BigInt 1
        m3.get_counter(NAME).should.have.property 1, BigInt 53
        await sleep 30
        m1.get_counter(NAME).should.have.property 1, BigInt 53
        m2.get_counter(NAME).should.have.property 1, BigInt 53

      it 'should accumulate values across two disconnected servers', ->
        port1 = port++
        port2 = port++

        m1 = M
          host: 'α'
          pub: port1
          subscribe_to: [
            "tcp://127.0.0.1:#{port2}"
          ]
        after -> m1.end()

        NAME = 'dog'
        m1.setup_counter NAME, Date.now()+8000
        m1.update_counter NAME, BigInt 3
        m1.get_counter(NAME).should.have.property 1, BigInt 3

        m1.update_counter NAME, BigInt 7
        m1.get_counter(NAME).should.have.property 1, BigInt 10

        m2 = M
          host: 'β'
          pub: port2
          subscribe_to: [
            "tcp://127.0.0.1:#{port1}"
          ]
        after -> m2.end()

        m2.setup_counter NAME, Date.now()+8000
        m2.update_counter NAME, BigInt 42
        m2.get_counter(NAME).should.have.property 1, BigInt 42
        await sleep 350
        m1.get_counter(NAME).should.have.property 1, BigInt 52
        m2.get_counter(NAME).should.have.property 1, BigInt 52

        m2.update_counter NAME, BigInt 42
        await sleep 30
        m2.get_counter(NAME).should.have.property 1, BigInt 94
        m1.get_counter(NAME).should.have.property 1, BigInt 94

      load = (filter) ->
        i = 'αβγδεζηθικλμνξοπρςστυφχψω'.split ''
        ports = i.map (_,i1) -> port++
        ms = i.map (host,i1) ->
          cfg =
            host: host
            pub: ports[i1]
            subscribe_to: ports.filter( (_,i2) -> filter i.length,i1,i2 ).map (p) -> "tcp://127.0.0.1:#{p}"
          M cfg
        after -> ms.forEach (m) -> m.end()

        await sleep 2000

        sum = 0

        NAME = 'lion'

        for j in [0...1000]
          await sleep 2
          v = Math.ceil 100*Math.random()
          sum += v
          x = Math.ceil ms.length * Math.random()
          x = ms.length-1 if x >= ms.length
          # console.log "Server #{x} #{i[x]} is going to get updated by #{v}"
          ms[x].update_counter NAME, (BigInt v), Date.now()+80000

        await sleep 300
        for m,j in ms
          console.log "Server #{i[j]}", m.get_counter(NAME), sum
        for m,j in ms
          m.get_counter(NAME).should.have.property 1, BigInt sum

      it 'should accumulate a whole bunch of values across a whole bunch of servers in double-loop', ->
        @timeout 4000
        load (l,i1,i2) -> i2 is (i1+2)%l or i2 is (i1-1)%l

      it 'should accumulate a whole bunch of values across a whole bunch of servers in extended double-loop', ->
        @timeout 4000
        load (l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-1)%l or i2 is (i1+7)%l or i2 is (i1-7)%l

      it 'should accumulate a whole bunch of values across a whole bunch of servers in sparse double-loop', ->
        @timeout 4000
        load (l,i1,i2) -> i2 is (i1+1)%l or i2 is (i1-7)%l

      it 'should accumulate a whole bunch of values across a whole bunch of servers in star', ->
        @timeout 4000
        load (l,i1,i2) -> if i1 is 0 then i2 isnt 0 else i2 is 0

      it 'should accumulate a whole bunch of values across a whole bunch of servers in dual star', ->
        @timeout 6000
        load (l,i1,i2) -> (if i1 is 0 then i2 isnt 0 else i2 is 0) or (if i1 is 1 then i2 isnt 1 else i2 is 1)

      it 'should accumulate a whole bunch of values across a whole bunch of servers in half-mesh', ->
        @timeout 4000
        load (l,i1,i2) -> i2 < i1

      it 'should accumulate a whole bunch of values across a whole bunch of servers in full-mesh', ->
        @timeout 4000
        load (l,i1,i2) -> i2 isnt i1
