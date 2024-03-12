---
layout: post
title: Using dry-validation with Grape
date: 2024-03-12 01:03 +0200
---
## Intro

[Grape](https://github.com/ruby-grape/grape/) is a popular web
framework for Ruby, an alternative to Rails when writing an API-only
application.

[dry-validation](https://dry-rb.org/gems/dry-validation/) is a
flexible library for implementing data validation logic, something one
tends to need more as their project grows.

When an API deals with a certain entity, the shape of endpoint
parameters in it tends to resemble the shape of the entity itself. And
if you already have validation rules defined for it somewhere else (or
intend to do that later, depending on the order the functionality is
built), you might like to use the same language/library/dsl for
describing the schema and rules in both cases, to be able to reuse
some.

Grape comes with its own terse DSL for describing parameters which
starting with version 1.3.0 is [implemented using dry-types under the
cover](https://github.com/ruby-grape/grape/pull/1920). But that
doesn't make it much easier to use `dry-validation` (a layer above
it). Here's the open [feature
request](https://github.com/ruby-grape/grape/issues/2386).

## Problem statement

Our goal here will be to declare the parameters for an API endpoint
using a `dry-validation` contract, have it check and coerce the
inputs, and in the case when the validation fails, send the response
describe the failure in the same format as the built-in validation
does, let's say, for interoperability.

Here's an example:

```rb
module Apples
  class API < Grape::API
    format :json
    prefix :api

    resource :orders do
      desc 'Create new'
      params do
        requires :order, type: Hash do
          requires :baskets, type: Array do
            requires :color, type: String, values: %w[green red yellow]
            optional :count, type: Integer, default: 10
          end
        end
      end
      post do
        Order.create!(declared(params))
      end
    end
  end
end
```

Our endpoint creates a nice big order of several baskets of apples,
each from one of three varieties.

Here's a faulty request and the response we get (for two attributes we messed up):

```sh
$ curl -H 'Content-Type: application/json' http://localhost:9292/api/orders/ \
       -d '{"order": {"baskets": [{"clor": 10, "count": "red"}]}}'
{"error":"order[baskets][0][color] is missing, order[baskets][0][count] is invalid"}
```

The error value enumerates all the invalid attributes, printing their full paths.

## Custom type

Here's how this structure can be described with `dry-rb`:

```rb
ColorString = Dry::Types['string'].constrained(included_in: %w(green red yellow))

BasketSchema = Dry::Schema.Params do
  required(:color).filled(ColorString)
  optional(:count).filled(Dry::Types['integer'].default(10))
end

class OrderContract < Dry::Validation::Contract
  params do
    required(:baskets).array(BasketSchema)
  end
end
```

One unsatisfying approach is to try the [documented way to use the
custom types](https://github.com/ruby-grape/grape#custom-types-and-coercions)
by defining `self.parse` on the type. It will almost work, but there
is no way to get the name of the "wrapper" attribute (`order` in our
example):

```rb
params do
  requires :order, type: OrderContract
end
```

And Grape will prepend the wrapper's name to the returned error
message, which seems difficult to print right when there are several
validation errors within our structure. So `self.parse` seems to be
better suited for "leaf" values.

## Simple delegation

Okay, referencing the contract in `params` seems problematic. Let's
drop that block entirely and do the validation inside the handler.

We create a new class which will call our contract to validate, raise
an exception (our custom type) to signal failure, and in the case of
success yields to the block. The error is formatted from the structure
the validator gives us:

```rb
  ContractError = Class.new(StandardError)

  class ApiContract < Dry::Validation::Contract
    def validate!(value)
      res = call(value)

      if res.success?
        yield res.to_h
      else
        message = errors_array(res.errors.to_h, false).join(', ')
        raise ContractError.new(message)
      end
    end

    def errors_array(hsh, decorate = true)
      res = []

      hsh.each do |key, value|
        key = if decorate
                "[#{key}]"
              else
                key.to_s
              end

        case value
        when String
          res << [key, value]
        when Array
          value.each do |s|
            res << [key, s]
          end
        else # Should be Hash.
          errors_array(value).each do |subkey, s|
            res << [key + subkey.to_s, s]
          end
        end
      end

      res.map! { |arr| arr.compact.join(' ') }
    end
  end

  # ...

  class CreateOrderContract < ApiContract
    params do
      required(:order).filled(OrderSchema)
    end
  end

  # ...

  class API < Grape::API
    format :json
    prefix :api

    rescue_from ContractError do |e|
      error!({error: e.message}, 400)
    end

    resource :orders do
      desc 'Create new'
      post do
        CreateOrderContract.new.validate!(params) do |attrs|
          order = Order.create!(attrs)
          body order
        end
      end
    end
  end
```

This reaches our goal (correct response format). Whether to actually
use Grape's format or keep the nested structure, is user's choice. But
this way we control the response either way.

## Custom DSL

The above is a little verbose, though. And it adds an extra nesting
level. What if we wanted to make the usage more succinct?

Looking at how the `params` block [is implemented](https://github.com/ruby-grape/grape/blob/master/lib/grape/validations/params_scope.rb),
the class `Grape::Validations::ParamsScope` applies itself to the
current endpoint by setting three "namespace stackable" attributes:
`declared_params`, `renamed_params` and `validations`. We will use
neither, using our own attribute to store the assigned contract.

If we just add a helper to fetch the validated input, we don't need to
convert the schema to what `declared` understands. This could also be
extended later so that the contract and declared params are used
together, if needed.

We also define an error type inheriting from
`Grape::Exceptions::Base`, so Grape handles it appropriately by
default, but that can still be overridden in some (or all) endpoints
using `rescue_from`.

The new implementation:

```rb
module GrapeContract
  class ValidationError < Grape::Exceptions::Base
    # @return [Dry::Validation::MessageSet]
    attr_reader :errors

    def initialize(errors:, headers: nil)
      @errors = errors
      message = errors_array(errors.to_h, false).join(', ')
      super(status: 400, message: message, headers: headers)
    end

    private

    def errors_array(hsh, decorate = true)
      # ...
    end
  end

  module EndpointDSL
    def contract(kontrakt = nil, &block)
      unless kontrakt
        kontrakt = Class.new(Dry::Validation::Contract, &block)
      end

      self.namespace_stackable(:contract, kontrakt)
    end
  end

  module InsideRouteHelper
    def contract_params
      kontrakt = namespace_stackable(:contract).last

      raise "No contract defined" unless kontrakt

      res = kontrakt.new.call(params)

      if res.success?
        res.to_h
      else
        raise ValidationError.new(errors: res.errors, headers: header)
      end
    end
  end

  # Install these globally, for every endpoint.
  ::Grape::API::Instance.extend(EndpointDSL)
  ::Grape::DSL::InsideRoute.include(InsideRouteHelper)
end
```

Now it can be used like

```rb
       desc 'Create new'
       contract CreateOrderContract
       post do
```

or even defining the contract inline:

```rb
      desc 'Create new'
      contract do
        params do
          required(:order).filled(OrderSchema)
        end

        rule(:order) do
          next if value[:baskets].count < 10

          key.failure('contains too many baskets')
        end
      end
      post do
```

See [this repository](https://github.com/dgutov/grape-dry-validation-example)
for the full example.
