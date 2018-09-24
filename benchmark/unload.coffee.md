A stresser that tries to keep the CPU around 80%.

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    stresser = (name,fun) ->

`fun` is the function under test

We ramp-up until we hit 80%, then up and down for a while.
Obviously `setTimeout` doesn't provide us with exact clocking, so we're keeping time as well.

      delay = 10

      runs = 10
      per_run = 1

      run_once = ->

        console.log "Going to execute #{runs} runs with #{delay}ms delay between #{per_run} loops…"

Run the garbage collector first, if available.

        if gc?
          gc_start = process.hrtime.bigint()
          gc()
          delta_gc = process.hrtime.bigint() - gc_start
          if delta_gc > `10000000n`
            console.log "GC took #{delta_gc/`1000n`}µs"

Collect start time

        start_time = process.hrtime.bigint()
        start_usage = process.cpuUsage()

Run the function

        run_count = 0
        while run_count++ < runs
          per_count = 0
          while per_count++ < per_run
            await fun()
          await sleep delay

Collect diff data

        delta_time = process.hrtime.bigint() - start_time # in nanoseconds
        delta_usage = process.cpuUsage start_usage        # in microseconds

        user   = BigInt delta_usage.user
        system = BigInt delta_usage.system

        pour_mille = (user+system) / (delta_time/`1000000n`)
        counter = BigInt runs * per_run

        console.log "#{runs} runs with #{delay}ms delay between #{per_run} loops: #{user}µs user, #{system}µs system, #{delta_time/`1000n`}µs real, #{counter} calls → #{pour_mille}‰ CPU, #{(counter*`1000000000n`)/delta_time}/s"

        if counter < `1000n`
          if delay > 0
            per_run++
          else
            runs++
        else
          if per_run > 1
            per_run--
          else
            runs--

        if pour_mille < 800 # 80%
          if delay > 0
            delay--
          else
            per_run++
        else
          if per_run > 1
            per_run--
          else
            delay++

        delay

      while true
        await run_once()

    module.exports = stresser
