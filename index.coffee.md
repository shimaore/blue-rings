    counter = require './crdt-counter'
    register = require './crdt-register'
    BlueRingStore = require './store'
    BlueRingAxon = require './protocol'
    assert = require 'assert'

Public API for a service storing EcmaScript counters and text.

    run = (options) ->

      options.Value ?= integer_values

      {Value,host} = options

      {Counter} = counter(Value)
      Register = register.LWWRegister

      COUNTER = 'C'
      REGISTER = 'R'

      class Mux

        type: (type) ->
          @__type = type
          switch type
            when COUNTER
              @__value = new Counter host
            when REGISTER
              @__value = new Register()
            else
              throw new Error "Invalid type #{type}"

          null

        increment: (args...) ->
          unless @__value?
            throw new Error 'You must use `setup_counter` before using `update_counter`.'
          type = @__type
          change = @__value.increment args...
          return null unless change?
          [type,change...]

        assign: (args...) ->
          unless @__value?
            throw new Error 'You must use `setup_text` before using `update_text`.'
          type = @__type
          change = @__value.assign args...
          return null unless change?
          [type,change...]

        value: -> @__value.value()

        merge: ([type,rest...]) ->
          @type type if not @__type
          msg = @__value.merge rest
          msg[1].unshift type
          msg

        all: ->
          return [] unless @__value?
          type = @__type
          @__value
          .all()
          .map (rest) ->
            [type,rest...]

        @serialize: ([type,rest...]) ->
          switch type
            when COUNTER
              [type].concat Counter.serialize rest
            when REGISTER
              [type].concat Register.serialize rest
            else
              throw new Error "Invalid type #{type}"

        @deserialize: ([type,rest...]) ->
          switch type
            when COUNTER
              [type].concat Counter.deserialize rest
            when REGISTER
              [type].concat Register.deserialize rest

      new_crdt = -> new Mux()

      store = new BlueRingStore new_crdt, host

      service = new BlueRingAxon Mux, store, options
      once = (e) -> new Promise (resolve) -> service.ev.once e, resolve
      bound = once 'bind'
      connected = once 'connected'

      get_value = (name) ->
        assert 'string' is typeof name, 'get_counter: name is required'
        value = store.get_value name
        coherent = service.coherent()
        [coherent,value]

      setup_counter = (name,expire) ->
        assert 'string' is typeof name, 'setup_counter: name is required'
        assert 'number' is typeof expire, 'setup_counter: expire is required'
        service.operation name, expire, 'type', COUNTER
        return

      update_counter = (name,amount,expire) ->
        assert 'string' is typeof name, 'update_counter: name is required'
        assert amount?, 'update_counter: amount is required'

        service.operation name, expire, 'increment', [amount]

        get_value name

      setup_text = (name,expire) ->
        assert 'string' is typeof name, 'setup_text: name is required'
        assert 'number' is typeof expire, 'setup_text: expire is required'
        service.operation name, expire, 'type', REGISTER
        return

      update_text = (name,text,expire) ->
        assert 'string' is typeof name, 'update_counter: name is required'
        assert 'string' is typeof text, 'update_counter: text is required'

        service.operation name, expire, 'assign', [text]

        get_value name

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
        update_counter
        get_counter:get_value
        setup_text
        update_text
        get_text:get_value
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
