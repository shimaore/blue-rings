    BlueRingAxon = require './protocol'
    assert = require 'assert'

Public API for a service storing EcmaScript numbers (transmitted as base-36 strings).

    SEPARATOR = ' '
    module.exports = (options) ->

We use BigInt unless otherwise specified.

      {values = javascript_bigint_values} = options

      accumulate = (acc,ticket) ->
        amount = values.parse ticket.split(SEPARATOR)[0]
        values.add acc, amount

      compute_value = (tickets) ->
        values.toString tickets.reduce(accumulate, values.zero)

      service = new BlueRingAxon compute_value, options

      get_counter = (name) ->
        assert 'string' is typeof name, 'get_counter: name is required'
        value = values.parse service.get_value name
        coherent = service.coherent()
        [coherent,value]

      setup_counter = (name,expire) ->
        assert 'string' is typeof name, 'setup_counter: name is required'
        assert 'number' is typeof expire, 'setup_counter: expire is required'
        service.add_counter name, expire

      update_counter = (name,amount,expire) ->
        assert 'string' is typeof name, 'update_counter: name is required'
        assert amount?, 'update_counter: amount is required'

Note how we also use BigInt for hrtime.

        timestamp = process.hrtime.bigint()

        ticket = [
          values.toString amount
          timestamp.toString 36
          options.host
        ].join SEPARATOR

        service.add_ticket name, ticket, expire

        get_counter name

      end = ->
        service.close()

      {setup_counter,update_counter,get_counter,end}

`values` interface for integers

    javascript_integer_values =
      parse: (t) -> if t? then parseInt t, 36 else null
      toString: (n) -> n.toString 36
      add: (n1,n2) -> n1+n2
      zero: 0

`values` interface for big integers (require Node.js 10.7.0 or above)

    javascript_bigint_values =
      parse: (t) -> if t? then BigInt t else null
      toString: (n) -> n.toString 10
      add: (n1,n2) -> n1+n2
      zero: BigInt 0
