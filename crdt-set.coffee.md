Delta-state C(v)RDT

We implement the CvRDT here. The delta-state is handled by the protocol.

Grow-only Set
=============

Page 22 (Specification 11) of Shapiro, Preguiça, Baquero, Zawirski

    class GrowSet

      constructor: ->
        @elements = new Set()

      add: (element) ->
        @elements.add element
        [element]

      has: (element) ->
        @elements.has element

      value: ->
        Array.from @elements.keys()

      changed_merge: ([element]) ->
        changed = false
        unless @elements.has element
          @elements.add element
          changed = true
        [ changed, [element] ]

      fast_merge: (element) ->
        @elements.add element
        return

      all: ->
        @elements.keys()

      @deserialize: ([element]) -> [element]
      @serialize:   ([element]) -> [element]

2P-Set
======

Two-Phase Set: elements might be added and removed, but once removed never put back in again.
(Specification 12)

    ADDED = 1
    REMOVED = 2

    class TPSet

      constructor: ->
        @added = new GrowSet
        @removed = new GrowSet

      add: (element) ->
        change = @added.add element
        return null unless change?
        [ADDED,change...]

      remove: (element) ->
        return null unless @has element # `pre lookup(e)` in Shapiro
        change = @removed.add element
        return null unless change?
        [REMOVED,change...]

`lookup` in Shapiro

      has: (element) ->
        (@added.has element) and not (@removed.has element)

      value: ->
        @added.value()
        .filter (e) => not @removed.has e

      merge: ([dir,element]) ->
        switch dir
          when ADDED
            @added.fast_merge element
          when REMOVED
            @removed.fast_merge element
        return

      all: ->
        all = []
        for element from @added.all()
          all.push [ADDED,element]
        for element from @removed.all()
          all.push [REMOVED,element]
        all

      @deserialize: ([dir,element]) -> [dir,element]
      @serialize:   ([dir,element]) -> [dir,element]

    module.exports = {GrowSet,TPSet}
