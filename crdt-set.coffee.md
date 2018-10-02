Delta-state C(v)RDT

We implement the CvRDT here. The delta-state is handled by the protocol.

Grow-only Set
=============

Page 22 (Specification 11) of Shapiro, Preguiça, Baquero, Zawirski

    class GrowSet

      constructor: ->
        @elements = new Set()

      add: (element) ->
        return null if @elements.has element
        @elements.add element
        element

      has: (element) ->
        @elements.has element

      value: ->
        Array.from @elements.keys()

      fast_merge: (element) ->
        @elements.add element
        return

      all: ->
        @elements.keys()

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
        [ADDED,change]

      remove: (element) ->
        return null unless @has element # `pre lookup(e)` in Shapiro
        change = @removed.add element
        return null unless change?
        [REMOVED,change]

`lookup` in Shapiro

      has: (element) ->
        (@added.has element) and not (@removed.has element)

      value: ->
        @added.value()
        .filter (e) => not @removed.has e

      merge: ([dir,element]) ->
        switch dir
          when ADDED
            if @added.has element
              false
            else
              @added.fast_merge element
              true
          when REMOVED
            if not @has element
              false
            else
              @removed.fast_merge element
              true
          else
            false

      all: ->
        all = []
        for element from @added.all()
          all.push [ADDED,element]
        for element from @removed.all()
          all.push [REMOVED,element]
        all

    module.exports = {GrowSet,TPSet}
