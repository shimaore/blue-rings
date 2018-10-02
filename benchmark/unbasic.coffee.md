    unload = require './unload'

    BlueRing = require '..'
    M = BlueRing.run
    port = Math.ceil 18000+1000*Math.random()
    tcp = (p) -> "udp4://127.0.0.1:#{p}"

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

      await Promise.all m.map bound
      await Promise.all m.map connected
      console.log "Bench #{N} setup complete"

      last_stats = list.map (i) -> m[i].statistics()
      stats = ->
        list.forEach (i) ->
          current_stats = m[i].statistics()
          console.log "-- recv: #{current_stats.recv-last_stats[i].recv}, recv_changes: #{current_stats.recv_changes-last_stats[i].recv_changes}"
          last_stats[i] = current_stats

      await unload 'add', ->
        name = "q#{Math.floor 500 * Math.random()}"
        m[0].setup_set name, Date.now()+8000
        await m[0].add name, 'c'+Math.random()
      , stats

      m.forEach (mm) -> mm.end()
      return

    do ->
      for N in [2...5]
        await bench N
