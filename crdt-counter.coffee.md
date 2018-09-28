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

        changed_merge: ([source,increment]) ->
          previous = (@increments.get source) ? Value.zero
          new_increment = Value.max previous, increment
          @increments.set source, new_increment
          changed = not Value.equals increment, new_increment
          [ changed, [ source, new_increment ] ]

        fast_merge: (source,increment) ->
          previous = (@increments.get source) ? Value.zero
          new_increment = Value.max previous, increment
          @increments.set source, new_increment
          return

This is called `query` in the same paper.

        value: ->
          value = Value.zero
          for v from @increments.values()
            value = Value.add value, v
          value

        all: ->
          @increments.entries()

        @deserialize: ([host,value]) -> [host,(Value.deserialize value)]
        @serialize:   ([host,value]) -> [host,(Value.serialize   value)]

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
          switch dir
            when PLUS
              @pluses.fast_merge source, increment
            when MINUS
              @minuses.fast_merge source, increment
          return

        value: ->
          Value.subtract @pluses.value(), @minuses.value()

        all: ->
          all = []
          for [source,increment] from @pluses.all()
            all.push [PLUS,source,increment]
          for [source,increment] from @minuses.all()
            all.push [MINUS,source,increment]
          all

        @deserialize: ([dir,host,value]) -> [dir,host,(Value.deserialize value)]
        @serialize:   ([dir,host,value]) -> [dir,host,(Value.serialize   value)]

      {GrowCounter,Counter}
