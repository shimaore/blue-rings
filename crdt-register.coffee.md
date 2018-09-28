    class LWWRegister # state-based, we're doing CvRDT
      constructor: ->
        @v = null
        @t = `0n`

      assign: (v) ->
        @v = v
        @t = process.hrtime.bigint()
        [@v,@t]

      changed_merge: ([v,t]) ->
        changed = false
        if @t < t
          @v = v
          @t = t
          changed = true
        [ changed, [ @v, @t ] ]

      merge: ([v,t]) ->
        if @t < t
          @v = v
          @t = t
        return

      value: ->
        @v

      all: ->
        [[@v,@t]]

      @deserialize: ([v,t]) -> [v,BigInt(t)] # BigInt.fromString t, 36
      @serialize:   ([v,t]) -> [v,t.toString()] # t.toString 36

    module.exports = {LWWRegister}
