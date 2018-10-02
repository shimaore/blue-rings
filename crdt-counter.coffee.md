Delta-state C(v)RDT

We implement the CvRDT here. The delta-state is handled by the protocol.

    module.exports = (Value) ->

Grow-only counter
=================

      class GrowCounter

See e.g. [GrowCounter](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#G-Counter_(Grow-only_Counter))
We store the increments for each node we know about.

        constructor: (@me) ->
          @increments = new Map()

This is called `increment` in [Shapiro,Preguiça,Baquero,Zawirski(2011)] Specification 6.

        increment: (amount) ->
          amount ?= Value.zero
          previous = (@increments.get @me) ? Value.zero
          increment = Value.add previous, amount
          @increments.set @me, increment
          increment

This is called `merge` in the same paper, with X = existing local payload, Y = incremental payload, Z = new local payload.

        merge: (source,increment) ->
          previous = (@increments.get source) ? Value.zero
          new_increment = Value.max previous, increment
          changed = not Value.equals increment, new_increment
          @increments.set source, new_increment
          changed

This is called `query` in the same paper.

        value: ->
          value = Value.zero
          for v from @increments.values()
            value = Value.add value, v
          value

        all: ->
          @increments.entries()

      # [host,value]
      if Value.deserialize?
        GrowCounter.deserialize = (a) -> a[1] = Value.deserialize a[1]
      if Value.serialize?
        GrowCounter.serialize   = (a) -> a[1] = Value.serialize   a[1]

PN-Counter
==========

      PLUS  = 1
      MINUS = 2

      class Counter

        constructor: (@me) ->
          @pluses = new GrowCounter @me
          @minuses = new GrowCounter @me

The INRIA paper offers `increment` and `decrement` (Specification 7), we combine the two.

        increment: (amount) ->
          switch
            when Value.is_positive amount
              [PLUS, @me, @pluses.increment amount]

            when Value.is_negative amount
              [MINUS, @me, @minuses.increment Value.abs amount]

            else
              null

        merge: ([dir,source,increment]) ->
          # console.log "PN-Counter #{@me} merge", dir, source, increment
          switch dir
            when PLUS
              @pluses.merge source, increment
            when MINUS
              @minuses.merge source, increment
            else
              false

        value: ->
          Value.subtract @pluses.value(), @minuses.value()

        all: ->
          all = []
          for [source,increment] from @pluses.all()
            all.push [PLUS,source,increment]
          for [source,increment] from @minuses.all()
            all.push [MINUS,source,increment]
          all

      # [dir,host,value]
      if Value.deserialize?
        Counter.deserialize = (a) -> a[2] = Value.deserialize a[2]
      if Value.serialize?
        Counter.serialize   = (a) -> a[2] = Value.serialize   a[2]

      {GrowCounter,Counter}
