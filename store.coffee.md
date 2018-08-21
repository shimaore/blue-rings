    EXPIRE  = 'E'
    VALUE   = 'V'

    nextTick = -> new Promise process.nextTick

    expired = (expire) -> expire < Date.now()

    collect = (store) ->
      for [name,L] from store.entries()
        expire = L.get EXPIRE
        @store.delete name if expired expire
        await nextTick()
      return

    class BlueRingStore
      constructor: (@Value,@CRDT,@host) ->
        @store = new Map()
        @__collector = setInterval collect, 3600*1000, @store

      destructor: ->
        clearInterval @__collector
        return

Public operations

      add_entry: (name,expire) ->
        @add_local_amount name, expire

      add_amount: (name,amount,expire) ->
        @add_local_amount name, expire, amount

      get_expire: (name) ->
        @store.get(name)?.get EXPIRE

      get_value: (name) ->
        L = @store.get name
        return null unless L?
        expire = L.get EXPIRE
        if not expired expire
          @store.get name
          L.get(VALUE).value()
        else
          @store.delete name
          null

Private operations

      __retrieve: (name,expire) ->
        L = @store.get(name)

        if L?
          if expire?
            L.set EXPIRE, expire if expire > L.get EXPIRE
        else
          L = new Map()
          L.set VALUE, new @CRDT(@Value,@host)
          if expire?
            L.set EXPIRE, expire
            @store.set name, L
          else
            L.set EXPIRE, 0

        L

Tool

      add_local_amount: (name,expire,amount) ->

        L = @__retrieve name, expire

        change = L.get(VALUE).increment amount

        expire_now = L.get EXPIRE
        return if expire is expire_now and not change?
        {name,expire:expire_now,changes:[change],source:@host}

Message handlers

      on_new_changes: (name,expire,changes,source,socket) ->
        L = @__retrieve name, expire
        value = L.get VALUE
        changed = false
        forward = changes
          .map (msg) =>
            [ modified, msg ] = value.update msg
            changed = true if modified
            msg

        expire_now = L.get EXPIRE
        {name,expire:expire_now,changes:forward,changed,source:@host}

      on_send: (name,expire,changes,socket) ->
        L = @__retrieve name, expire
        value = L.get VALUE
        changes.forEach (msg) ->
          value.update msg

        expire = L.get EXPIRE
        changes = value.all()
        {name,expire,changes,source:@host}

      enumerate_local_values: (cb) ->
        for [name,L] from @store.entries()
          expire = L.get EXPIRE
          unless expired expire
            changes = L.get(VALUE).all()
            await cb {name,expire,changes,source:@host}
        return

    module.exports = BlueRingStore
