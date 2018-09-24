    unload = require './unload'

    BlueRing = require '..'
    M = BlueRing.run
    port = Math.ceil 18000+1000*Math.random()
    tcp = (p) -> "tcp://127.0.0.1:#{p}"

    different = (i) ->
      (j) -> j isnt i
    bound = (l) -> l.bound
    connected = (l) -> l.connected

    bench = (N) ->
      console.log "(un)load Bench #{N}"

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

      NAME = 'foo'
      await unload 'add', ->
        m[0].setup_set NAME, Date.now()+8000
        await m[0].add NAME, 'c'+Math.random()

      await unload 'has', ->
        await m[1].has NAME, 'c'

      m.forEach (mm) -> mm.end()
      return

    do ->
      for N in [2...5]
        await bench N
