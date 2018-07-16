    BlueRingAxon = require './protocol'
    SEPARATOR = ' '

Public API for a service storing JavaScript Integers (transmitted as base-36 strings).

    javascript_integer_values =
      parse: (t) -> parseInt t, 36
      toString: (n) -> n.toString 36
      add: (n1,n2) -> n1+n2
      zero: 0

    module.exports = (options) ->

      {values = javascript_integer_values} = options

      accumulate = (acc,ticket) ->
        amount = values.parse ticket.split(SEPARATOR)[0]
        values.add acc, amount

      service = new BlueRingAxon options

      get_counter = (name) ->
        value = service.reduce name, accumulate, values.zero
        coherent = service.coherent()
        return [coherent,value]

      setup_counter = (name,expire) ->
        service.add_counter name, expire

      update_counter = (name,amount) ->
        timestamp = process.hrtime()

        ticket = [
          values.toString amount
          timestamp[0].toString 36
          timestamp[1].toString 36
          options.host
        ].join SEPARATOR

        service.add_ticket name, ticket

        get_counter name

      {setup_counter,update_counter,get_counter}
