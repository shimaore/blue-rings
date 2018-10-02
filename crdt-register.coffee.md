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
          true
        else
          false

      value: ->
        @v

      all: ->
        [[@v,@t]]

      # [v,t]
      @deserialize: (a) -> a[1] = BigInt a[1]     # BigInt.fromString t, 36
      @serialize:   (a) -> a[1] = a[1].toString() # t.toString 36

    module.exports = {LWWRegister}
