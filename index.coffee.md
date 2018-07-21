    BlueRingAxon = require './protocol'
    assert = require 'assert'

Public API for a service storing EcmaScript numbers (transmitted as base-36 strings).

    run = (options) ->

      {Value} = options

      {add} = Value

      service = new BlueRingAxon Value, options
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

Note how we also use BigInt for hrtime.

        timestamp = process.hrtime.bigint()

Tickets are [key,value] pairs.

        ticket = [
          [timestamp.toString(36),options.host].join ' '
          amount
        ]

        service.add_ticket name, ticket, expire

        get_counter name

      statistics = ->
        recv: service.recv
        sent: service.sent

      end = ->
        service.close()

      {setup_counter,update_counter,get_counter,bound,connected,statistics,end}

`values` interface for integers

    integer_values =
      parse: (t) -> if t? then parseInt t, 36 else null
      toString: (n) -> n.toString 36
      add: (n1,n2) -> n1+n2
      zero: 0
      accept: Number

`values` interface for big integers (require Node.js 10.7.0 or above)

    bigint_values =
      parse: (t) -> if t? then BigInt t else null
      toString: (n) -> n.toString 10
      add: (n1,n2) -> n1+n2
      zero: BigInt 0
      accept: BigInt

    module.exports = {run,integer:integer_values,bigint:bigint_values}
