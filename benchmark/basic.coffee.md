    Benchmark = require 'benchmark'

    BlueRing = require '..'
    M = BlueRing.run
    port = Math.ceil 18000+1000*Math.random()
    tcp = (p) -> "tcp://127.0.0.1:#{p}"

    different = (i) ->
      (j) -> j isnt i
    bound = (l) -> l.bound
    connected = (l) -> l.connected

    bench = (N) ->
      console.log "Bench #{N}"

      m = null

      list = [0...N]

      ports = list.map -> port++
      m = list.map (i) ->
        M
          host: 'α'
          pub: tcp ports[i]
          subscribe_to: list.filter(different(i)).map (j) -> tcp ports[j]
          connect_delay: 0

      await Promise.all m.map bound
      await Promise.all m.map connected
      console.log "Bench #{N} setup complete"

      suite = new Benchmark.Suite

      NAME = 'foo'

      suite
      .add 'add', ->
        m[0].setup_set NAME, Date.now()+8000
        await m[0].add NAME, 'c'+Math.random()
      .add 'has', ->
        await m[1].has NAME, 'c'
      .on 'start', ->
        console.log "Start Bench #{N}"
      .on 'cycle', (event) ->
        console.log "Bench #{N} #{event.target}"
      .on 'complete', ->
        console.log "Complete Bench #{N}"
        m.forEach (mm) -> mm.end()
      .run defer:true

    do ->
      for N in [2...5]
        await bench N
