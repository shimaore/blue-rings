Delta-state C(v)RDT

We implement the CvRDT here. The delta-state is handled by the protocol.

    class GrowCounter

See e.g. [GrowCounter](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#G-Counter_(Grow-only_Counter))
We store the increments for each node we know about.

      constructor: (@Value,@me) ->
        @increments = new Map()

      increment: (amount) ->
        increment = @Value.add amount, (@increments.get @me) ? @Value.zero
        @increments.set @me, increment
        increment

      update: (source,increment) ->
        previous = (@increments.get source) ? @Value.zero
        new_increment = @Value.max previous, increment
        @increments.set source, new_increment
        new_increment

      value: ->
        value = @Value.zero
        for v from @increments.values()
          value = @Value.add value, v
        value

      all: ->
        @increments.entries()

    PLUS  = '+'
    MINUS = '-'

    class Counter
      constructor: (@Value,@me) ->
        @pluses = new GrowCounter @Value, @me
        @minuses = new GrowCounter @Value, @me

      increment: (amount) ->
        switch
          when @Value.is_positive amount
            [PLUS, @me, @pluses.increment amount]

          when @Value.is_negative amount
            [MINUS, @me, @minuses.increment @Value.abs amount]

          else
            null

      update: (dir,source,increment) ->
        switch dir
          when PLUS
            @pluses.update source, increment
          when MINUS
            @minuses.update source, increment

      value: ->
        @Value.subtract @pluses.value(), @minuses.value()

      all: ->
        all = []
        for [source,increment] from @pluses.all()
          all.push [PLUS,source,increment]
        for [source,increment] from @minuses.all()
          all.push [MINUS,source,increment]
        all

    module.exports = {GrowCounter,Counter}
