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

      COUNTER = 1
      REGISTER = 2
      SET = 3

      class Mux

        type: (type) ->
          return if @__type?
          @__type = type
          switch type
            when COUNTER
              @__value = new Counter host
            when REGISTER
              @__value = new Register()
            when SET
              @__value = new TPSet()
            else
              throw new Error "Invalid type #{type}"

          null

        increment: (args...) ->
          unless @__value?
            throw new Error 'You must use `setup_counter` before using `increment`.'
          type = @__type
          change = @__value.increment args...
          return null unless change?
          [type,change]

        assign: (args...) ->
          unless @__value?
            throw new Error 'You must use `setup_register` before using `assign`.'
          type = @__type
          change = @__value.assign args...
          return null unless change?
          [type,change]

        add: (args...) ->
          unless @__value?
            throw new Error 'You must use `setup_set` before using `add`.'
          type = @__type
          change = @__value.add args...
          return null unless change?
          [type,change]

        remove: (args...) ->
          unless @__value?
            throw new Error 'You must use `setup_set` before using `remove`.'
          type = @__type
          change = @__value.remove args...
          return null unless change?
          [type,change]

        value: -> @__value.value()
        has: (element) -> @__value.has element

        merge: ([type,change]) ->
          @type type
          @__value.merge change
          return

        all: ->
          return [] unless @__value?
          type = @__type
          @__value
          .all()
          .map (change) ->
            [type,change]

        @serialize: (a) -> # [type,change]
          # console.log 'serialize', a
          type = a[0]
          switch type
            when COUNTER
              if Counter.serialize?
                Counter.serialize a[1]
            when REGISTER
              Register.serialize a[1]
            when SET
              yes
            else
              throw new Error "Invalid type #{type}"

        @deserialize: (a) -> # [type,change]
          # console.log 'deserialize', a
          type = a[0]
          switch type
            when COUNTER
              if Counter.deserialize?
                Counter.deserialize a[1]
            when REGISTER
              Register.deserialize a[1]
            when SET
              yes
            else
              throw new Error "Invalid type #{type}"

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
