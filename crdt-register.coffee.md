    class LWWRegister # state-based, we're doing CvRDT
      constructor: ->
        @v = null
        @t = 0

      assign: (v) ->
        @v = v
        @t = process.hrtime.bigint()
        [@v,@t]

      merge: ([v,t]) ->
        changed = false
        if @t < t
          @v = v
          @t = t
          changed = true
        [ changed, [ @v, @t ] ]

      value: ->
        @v

      all: ->
        [[@v,@t]]

      @deserialize: ([v,t]) -> [v,BigInt(t)]
      @serialize:   ([v,t]) -> [v,t.toString()]

    module.exports = {LWWRegister}
