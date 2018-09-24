    nextTick = -> new Promise process.nextTick

    expired = (expire) -> expire < Date.now()

    class BlueRingNode
      constructor: (@value,@expire) ->
        @expire ?= 0

      set_expire: (expire) ->
        return unless expire?
        @expire = expire if expire > @expire

    collect = (store) ->
      for [name,L] from store.entries()
        expire = L.expire
        @store.delete name if expired expire
        await nextTick()
      return

    class BlueRingStore
      constructor: (@new_crdt,@host) ->
        @store = new Map()
        @__collector = setInterval collect.bind(this), 3600*1000, @store

      destructor: ->
        clearInterval @__collector
        return

      get_expire: (name) ->
        @store.get(name)?.expire

      __retrieve: (name,expire) ->
        L = @store.get(name)

        if L?
          L.set_expire expire
        else
          L = new BlueRingNode @new_crdt(), expire
          if expire?
            @store.set name, L

        L

`query` in Shapiro

      query: (name,op,args...) ->
        L = @store.get name
        return null unless L?
        expire = L.expire
        if not expired expire
          @store.get name
          L.value[op]? args...
        else
          @store.delete name
          null

`update` in Shapiro

      update: (name,expire,op,args) ->

        L = @__retrieve name, expire

        change = L.value[op]? args...

        expire_now = L.expire
        return if expire is expire_now and not change?
        if change?
          {name,expire:expire_now,changes:[change],source:@host}
        else
          {name,expire:expire_now,changes:[],source:@host}

Message handlers

      on_new_changes: (name,expire,changes,source,socket) ->
        L = @__retrieve name, expire
        value = L.value
        changed = false
        forward = changes
          .map (msg) =>
            [ modified, msg ] = value.merge msg
            changed = true if modified
            msg

        expire_now = L.expire
        {name,expire:expire_now,changes:forward,changed,source:@host}

      on_send: (name,expire,changes,socket) ->
        L = @__retrieve name, expire
        value = L.value
        changes.forEach (msg) ->
          value.merge msg

        expire = L.expire
        changes = value.all()
        {name,expire,changes,source:@host}

      enumerate_local_values: (cb) ->
        for [name,L] from @store.entries()
          expire = L.expire
          unless expired expire
            changes = L.value.all()
            yield {name,expire,changes,source:@host}
        return

    module.exports = BlueRingStore
