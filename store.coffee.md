    EXPIRE  = 'expire'
    PLUS  = '+'
    MINUS = '-'
    COUNTER = 'counter'

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
        if @Value.equals new_increment, previous
          null
        else
          @increments.set source, new_increment
          new_increment

      value: ->
        value = @Value.zero
        for v from @increments.values()
          value = @Value.add value, v
        value

      all: ->
        @increments.entries()

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

    expired = (expire) -> expire < Date.now()

    class BlueRing
      constructor: (@Value,@host) ->
        @store = new Map()

Public operations

      add_counter: (name,expire) ->
        @add_local_amount name, @Value.zero, expire

      add_amount: (name,amount,expire) ->
        @add_local_amount name, amount, expire

      get_expire: (name) ->
        @store.get(name)?.get EXPIRE

      get_value: (name) ->
        L = @store.get name
        return null unless L?
        expire = L.get EXPIRE
        if not expired expire
          @store.get name
          L.get(COUNTER).value()
        else
          @store.delete name
          null

Private operations

      __counter: (name,expire) ->
        L = @store.get(name)

        if L?
          if expire?
            L.set EXPIRE, expire if expire > L.get EXPIRE
        else
          L = new Map()
          L.set COUNTER, new Counter(@Value,@host)
          if expire?
            L.set EXPIRE, expire
            @store.set name, L
          else
            L.set EXPIRE, 0

        L

Tool

      add_local_amount: (name,amount,expire) ->

        L = @__counter name, expire

        change = L.get(COUNTER).increment amount

        expire_now = L.get EXPIRE
        return if expire is expire_now and not change?
        {name,expire:expire_now,changes:[change],source:@host}

Message handlers

      on_new_changes: (name,expire,changes,source,socket) ->
        L = @__counter name, expire
        counter = L.get COUNTER
        forward = changes
          .map ([dir,source,increment]) ->
            increment = counter.update dir,source,increment
            [ dir, source, increment ]
          .filter ([dir,source,increment]) -> increment?

        # console.log 'forward', @host, forward
        expire_now = L.get EXPIRE

        return if expire is expire_now and forward.length is 0
        {name,expire:expire_now,changes:forward,source}

      enumerate_local_counters: (cb) ->
        @store.forEach (L,name) =>
          expire = L.get EXPIRE
          return if expired expire
          changes = L.get(COUNTER).all()
          cb {name,expire,changes,source:@host}

    module.exports = BlueRing
