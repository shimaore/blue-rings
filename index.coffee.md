    BlueRingAxon = require './protocol'
    assert = require 'assert'

Public API for a service storing EcmaScript numbers (transmitted as base-36 strings).

    run = (options) ->

      options.Value ?= integer_values

      {Value} = options

      {add} = Value

      service = new BlueRingAxon options
      once = (e) -> new Promise (resolve) -> service.ev.once e, resolve
      bound = once 'bind'
      connected = once 'connected'

      get_counter = (name) ->
        assert 'string' is typeof name, 'get_counter: name is required'
        value = service.get_value name
        coherent = service.coherent()
        [coherent,value]

      setup_counter = (name,expire) ->
        assert 'string' is typeof name, 'setup_counter: name is required'
        assert 'number' is typeof expire, 'setup_counter: expire is required'
        service.add_counter name, expire
        return

      update_counter = (name,amount,expire) ->
        assert 'string' is typeof name, 'update_counter: name is required'
        assert amount?, 'update_counter: amount is required'

        service.add_amount name, amount, expire

        get_counter name

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

      {setup_counter,update_counter,get_counter,bound,connected,statistics,end,subscribe_to}

`values` interface for integers

    integer_values =
      deserialize: (t) -> parseInt t, 36
      serialize: (n) -> n.toString 36
      add: (n1,n2) -> n1+n2
      equals: (n1,n2) -> n1 is n2
      abs: (n) -> if n < 0 then -n else n
      subtract: (n1,n2) -> n1-n2
      is_positive: (n) -> n > 0
      is_negative: (n) -> n < 0
      max: (n1,n2) -> if n1 > n2 then n1 else n2
      zero: 0
      accept: Number

`values` interface for big integers (require Node.js 10.7.0 or above)

    bigint_values =
      deserialize: (t) -> BigInt t
      serialize: (n) -> n.toString()
      add: (n1,n2) -> n1+n2
      equals: (n1,n2) -> n1 is n2
      abs: (n) -> if n < BigInt 0 then -n else n
      subtract: (n1,n2) -> n1-n2
      is_positive: (n) -> n > BigInt 0
      is_negative: (n) -> n < BigInt 0
      max: (n1,n2) -> if n1 > n2 then n1 else n2
      zero: BigInt 0
      accept: BigInt

    module.exports = {run,integer:integer_values,bigint:bigint_values}
