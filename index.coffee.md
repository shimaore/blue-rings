    counter = require './crdt-counter'
    register = require './crdt-register'
    set = require './crdt-set'
    BlueRingStore = require './store'
    BlueRingAxon = require './protocol'
    assert = require 'assert'

Public API for a service storing EcmaScript counters and text.

    run = (options) ->

      options.Value ?= integer_values

      {Value,host} = options
      assert host?, '`host` is required'

      {Counter} = counter(Value)
      Register = register.LWWRegister
      {TPSet} = set

      COUNTER = 'C'
      REGISTER = 'R'
      SET = 'S'

      class Mux

        type: (type) ->
          return if @__type?
          @__type = type
          switch type
            when COUNTER
              @__cnt = new Counter host
            when REGISTER
              @__reg = new Register()
            when SET
              @__set = new TPSet()
            else
              throw new Error "Invalid type #{type}"

          null

        increment: (args...) ->
          unless @__type is COUNTER and @__cnt?
            throw new Error 'You must use `setup_counter` before using `increment`.'
          change = @__cnt.increment args...
          return null unless change?
          [COUNTER,change...]

        assign: (args...) ->
          unless @__type is REGISTER and @__reg?
            throw new Error 'You must use `setup_register` before using `assign`.'
          change = @__reg.assign args...
          return null unless change?
          [REGISTER,change...]

        add: (args...) ->
          unless @__type is SET and @__set?
            throw new Error 'You must use `setup_set` before using `add`.'
          change = @__set.add args...
          return null unless change?
          [SET,change...]

        remove: (args...) ->
          unless @__type is SET and @__set?
            throw new Error 'You must use `setup_set` before using `remove`.'
          change = @__set.remove args...
          return null unless change?
          [SET,change...]

        value: ->
          switch @__type
            when COUNTER
              @__cnt.value()
            when REGISTER
              @__reg.value()
            when SET
              @__set.value()

        has: (element) -> @__set.has element

        merge: ([type,rest...]) ->
          @type type
          switch type
            when COUNTER
              msg = @__cnt.merge rest
            when REGISTER
              msg = @__reg.merge rest
            when SET
              msg = @__set.merge rest
          msg[1].unshift type
          msg

        all: ->
          type = @__type
          switch type
            when COUNTER
              @__cnt?.all().map((rest) -> [type,rest...]) ? []
            when REGISTER
              @__reg?.all().map((rest) -> [type,rest...]) ? []
            when SET
              @__set?.all().map((rest) -> [type,rest...]) ? []

        @serialize: ([type,rest...]) ->
          switch type
            when COUNTER
              [type].concat Counter.serialize rest
            when REGISTER
              [type].concat Register.serialize rest
            when SET
              [type].concat TPSet.serialize rest
            else
              throw new Error "Invalid type #{type}"

        @deserialize: ([type,rest...]) ->
          switch type
            when COUNTER
              [type].concat Counter.deserialize rest
            when REGISTER
              [type].concat Register.deserialize rest
            when SET
              [type].concat TPSet.deserialize rest

      new_crdt = -> new Mux()

      store = new BlueRingStore new_crdt, host

      service = new BlueRingAxon Mux, store, options
      once = (e) -> new Promise (resolve) -> service.ev.once e, resolve
      bound = once 'bind'
      connected = once 'connected'

      get_value = (name) ->
        assert 'string' is typeof name, 'get_counter: name is required'
        value = store.query name, 'value'
        coherent = service.coherent()
        [coherent,value]

      setup_counter = (name,expire) ->
        assert 'string' is typeof name, 'setup_counter: name is required'
        assert 'number' is typeof expire, 'setup_counter: expire is required'
        service.update name, expire, 'type', [COUNTER]
        return

      increment = (name,amount,expire) ->
        assert 'string' is typeof name, 'update_counter: name is required'
        assert amount?, 'update_counter: amount is required'

        service.update name, expire, 'increment', [amount]

        get_value name

      setup_register = (name,expire) ->
        assert 'string' is typeof name, 'setup_register: name is required'
        assert 'number' is typeof expire, 'setup_register: expire is required'
        service.update name, expire, 'type', [REGISTER]
        return

      assign = (name,text,expire) ->
        assert 'string' is typeof name, 'update_counter: name is required'

        service.update name, expire, 'assign', [text]

        get_value name

      setup_set = (name,expire) ->
        assert 'string' is typeof name, 'setup_set: name is required'
        assert 'number' is typeof expire, 'setup_set: expire is required'
        service.update name, expire, 'type', [SET]
        return

      add = (name,element,expire) ->
        assert 'string' is typeof name, 'add: name is required'
        assert element?, 'add: element is required'

        service.update name, expire, 'add', [element]
        return

      remove = (name,element,expire) ->
        assert 'string' is typeof name, 'remove: name is required'
        assert element?, 'remove: element is required'

        service.update name, expire, 'remove', [element]
        return

      has = (name,element) ->
        assert 'string' is typeof name, 'add: name is required'
        assert element?, 'has: element is required'

        value = store.query name, 'has', element
        coherent = service.coherent()
        [coherent,value]

      statistics = ->
        recv: service.recv
        recv_changes: service.recv_changes
        sent: service.sent
        sent_changes: service.sent_changes

      end = ->
        service.destructor()
        return

      subscribe_to = (port) ->
        service.subscribe_to port

      {
        setup_counter
        increment
        update_counter:increment
        value:get_value
        get_value
        get_counter:get_value
        setup_text:setup_register
        setup_register
        update_text:assign
        assign
        get_text:get_value
        setup_set
        add
        remove
        has

        bound
        connected
        statistics
        end
        subscribe_to
      }

`values` interface for integers

    integer_values =
      deserialize: (t) -> parseInt t, 36  # protocol
      serialize: (n) -> n.toString 36     # protocol
      add: (n1,n2) -> n1+n2
      equals: (n1,n2) -> n1 is n2
      abs: (n) -> if n < 0 then -n else n
      subtract: (n1,n2) -> n1-n2
      is_positive: (n) -> n > 0
      is_negative: (n) -> n < 0
      max: (n1,n2) -> if n1 > n2 then n1 else n2
      zero: 0
      accept: Number # used for testing

`values` interface for big integers (require Node.js 10.7.0 or above)

    bigint_values =
      deserialize: (t) -> BigInt t    # protocol
      serialize: (n) -> n.toString()  # protocol
      add: (n1,n2) -> n1+n2
      equals: (n1,n2) -> n1 is n2
      abs: (n) -> if n < BigInt 0 then -n else n
      subtract: (n1,n2) -> n1-n2
      is_positive: (n) -> n > BigInt 0
      is_negative: (n) -> n < BigInt 0
      max: (n1,n2) -> if n1 > n2 then n1 else n2
      zero: BigInt 0
      accept: BigInt # used for testing

    module.exports = {run,integer:integer_values,bigint:bigint_values}
