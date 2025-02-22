# Bluepine

![CircleCI](https://img.shields.io/circleci/project/omise/bluepine.svg?style=flat)

`Bluepine` is a DSL for defining API [Schema](#schema)/[Endpoint](#endpoint) with the capabilities to generate `Open API (v3)` spec (other specs is coming soon), validate API request and serialize object for API response based on single schema definition.

## Table of Contents

- [Quick Start](#quick-start)
  - [Defining a Schema](#defining-a-schema)
  - [Serializing Schema](#serializing-schema)
  - [Generating Open API (v3)](#generating-open-api-v3)
- [Installation](#installation)
- [Attributes](#attributes)
  - [Creating Attribute](#creating-attribute)
  - [Attribute Options](#attribute-options)
  - [Custom Attribute](#custom-attribute)
- [Resolver](#resolver)
  - [Manually registering schema/endpoint](#manually-registering-schemaendpoint)
  - [Automatically registering schema/endpoint](#automatically-registering-schemaendpoint)
- [Serialization](#serialization)
  - [Example](#serializer-example)
  - [Conditional Serialization](#conditional-serialization)
  - [Custom Serializer](#custom-serializer)
- [Endpoint](#endpoint)
  - [Method](#endpoint-method)
  - [Params](#endpoint-params)
  - [Validation](#endpoint-validation)
  - [Permitted Params (Rails)](#permitted-params)
- [Validation](#validation)
  - [Conditional Validation](#validator-condition)
  - [Custom Normalizer](#custom-normalizer)
  - [Custom Validator](#custom-validator)
- [Generating API Specifications](#generating-api-specification)
  - [Open API (v3)](#open-api-v3)


## Quick Start

### Defining a schema

Let's start by creating a simple schema. (For a complete list of attributes and its options please see [Attributes](#attributes) section)

> A schema can be created and registered separately, or we can use `Resolver` to create and register in one step.

```ruby
require "bluepine"

# Schema is just an `ObjectAttribute`
Bluepine::Resolver.new do

  # Defines :hero schema
  schema :hero do
    string :name, min: 4

    # recursive schema
    array   :friends, of: :hero

    # nested object
    object :stats do
      number :strength, default: 0
    end

    # reference
    schema :team
  end

  # Defines :team schema
  schema :team do
    string :name, default: "Avengers"
  end
end
```

### Serializing schema

In order to serialize schema, we just pass schema defined in previous step to `Serializer`.

> The object to be serialized can be a `Hash` or any `Object` with method/accessor.

```ruby
hero = {
  name: "Thor",
  friends: [
    {
      name: "Iron Man",
      stats: {
        strength: "9"
      }
    }
  ],
  stats: {
    strength: "8"
  }
}

# or using our own Model class
hero = Hero.new(name: "Thor")

serializer = Bluepine::Serializer.new(resolver)
serializer.serialize(hero_schema, hero)
```

will produce the following result

```ruby
{
  name: "Thor",
  stats: {
    strength: 8
  },
  friends: [
    { name: "Iron Man", stats: { strength: 9 }, friends: [], team: { name: "Avengers" } }
  ],
  team: {
    name: "Avengers"
  }
}
```
> Note: It converts number to string (via `Attribute.serializer`) and automatically adds missing fields and default value

### Validating data

To validate data against defined schema. We just pass it to `Validator#validate` method.

> The payload could be a `Hash` or any `Object`.

```ruby
payload = {
  name: "Hulk",
  friends: [
    { name: "Tony" },
    { name: "Sta"},
  ],
  team: {
    name: "Aven"
  }
}

validator = Bluepine::Validator.new(resolver)
validator.validate(user_schema, payload) # => Result
```

It'll return `Result` object which has 2 attributes `#value` and `#errors`.

In the case of errors, `#errors` will contain all error messages

```ruby
# Result.errors =>
{
  friends: {
    1 => {
      name: ["is too short (minimum is 4 characters)"]
    }
  },
  team: {
    name: ["is too short (minimum is 5 characters)"]
  }
}
```

If there's no errors, `#value` will contain normalized data.

```ruby
# Result.value =>
{
  name: "Thor",
  stats: {
    strength: 0
  },
  friends: [
    {
      name: "Iron Man",
      stats: { strength: 0 },
      friends: [],
      team: {
        name: "Avengers"
      }
    }
  ],
  team: { name: "Avengers" }
}
```

> All default values will be added automatically.

### Generating Open API (v3)

```ruby
generator = Bluepine::Generators::OpenAPI::Generator.new(resolver)
generator.generate # => return Open API v3 Specification
```

## Installation

    gem 'bluepine'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install bluepine

## Attributes

`Attribute` is just a simple class which doesn't have any functionality/logic on its own. With this design, it decouples the logics to `validate`, `serialize`, etc from `Attribute` and let's consumers (e.g. `Validator`, `Serializer`, etc) decide it instead. 

Here're pre-defined attributes that we can use.

* `string` - StringAttribute
* `boolean` - BooleanAttribute
* `number` - NumberAttribute
* `integer` - IntegerAttribute
* `float` - FloatAttribute
* `array` - [ArrayAttribute](#array-attribute)
* `object` - [ObjectAttribute](#object-attribute)
* `schema` - [SchemaAttribute](#schema-attribute)

### Creating Attribute

There're couple of ways to create attributes. We can create it manually or using some other methods.

#### Manually Creating Attribute
Here, we're creating it manually.

```ruby
user_schema = Bluepine::Attributes::ObjectAttribute.new(:user) do
  string :username
  string :password
end
```

#### Using `Attributes.create`

This is equivalent to the above code

```ruby
Bluepine::Attributes.create(:object, :user) do
  string :username
  string :password
end
```

#### Using `Resolver`

This is probably the easiest way to create object attribute. Since it also keeps track of the created attribute for you. (So, we don't have to register it by ourself. See also [Resolver](#resolver))

```ruby
Bluepine::Resolver.new do
  schema :user do
    string :username
    string :password
  end
end
```

### Array Attribute

Array attribute supports an option named `:of` which we can use to describe what kind of data can be contained inside `array`.

For example

```ruby
schema :user do
  string :name

  # Indicates that each item inside must have the same structure
  # as :user schema (e.g. friends: [{ name: "a", friends: []}, ...])
  array  :friends, of: :user

  # i.e. pets: ["Joey", "Buddy", ...]
  array  :pets, of: :string

  # When nothing is given, array can contain any kind of data
  array  :others
end
```

### Object Attribute

Most of the time, we'll be working with this attribute more often. 

```ruby
schema :user do
  string :name

  # nested attribute
  object :address do
    string :street

    # more nested attribute if needed
    object :country do
      string :name
    end
  end
end
```

### Schema Attribute

Instead of declaring a lot of nested object. We can also use `schema` attribute to refer to other previously defined schema (DRY).

It also accepts `:of` option. (it works the same as `Array`)

```ruby
schema :hero do
  string :name

  # This implies `of: :team`
  schema :team

  # If the field name is different, we can specify `:of` option (which work the same way as `Array`)
  schema :awesome_team, of: :team
end

schema :team do
  string :name
end
```

### Attribute Options

All attributes have common set of options avaiable

| Name | type | Description | Serializer | Validator | Open API 
|-|-|-|-|-|-|
| name | `string\|symbol` | Attribute's name e.g. `email` |
| [method](#method-options) | `symbol` | When attribute's `name` differs from target's `name`, we can use this to specify a method that will be used to get the value for the attribute. | read value from specified name instead. See [Serializer `:method`](#serializer-options-method). | | |
| [match](#match-options) | `Regexp` | `Regex` that will be used to validate the attribute's value (`string` attribute) | | validates string based on given `Regexp` | Will add `Regexp` to generated `pattern` property |
| type | `string` | Data type | Attribute's type e.g. `string`, `schema` etc
| native_type | `string` | JSON's data type |
| [format](#format-options) | `string\|symbol ` | describes the format of this value. Could be arbitary value e.g. `int64`, `email` etc. | | | This'll be added to `format` property |
| [of](#of-options) | `symbol ` | specifies what type of data will be represented in `array`. The value could be attribute type e.g. `:string` or other schema e.g. `:user` | serializes data using specified value. See [Serializer `:of`](#serializer-options-of)| validates data using specified value | Create a `$ref` type schema |
| [in](#in-options) | `array` | A set of valid options e.g. `%w[thb usd ...]` | | payload value must be in this list | adds to `enum` property |
| [if/unless](#if-options) | `symbol\|proc` | Conditional validating/serializing result | serializes only when the specified value evalulates to `true`. See [Serializer `:if/:unless`](#serializer-options-if-unless) | validates only when it evalulates to `true` |
| required | `boolean` | Indicates this attribute is required (for validation). Default is `false` | | makes it mandatory | adds to `required` list |
| default | `any` | Default value for attribute | uses as default value when target's value is `nil` | populates as default value when it's not defined in payload | adds to `default` property |
| private | `boolean` | marks it as `private`. Default is `false` | Excludes this attribute from serialized value |
| deprecated | `boolean` | marks this attribute as deprecated. Default is `false` | | | adds to `deprecated` property |
| description | `string` | Description of attribute |
| spec | `string` | Specification of the value (for referencing only) |
| spec_uri | `string` | URI of `spec` |

### Custom Attribute

If you want to add your own custom attribute. Simply create a new class and make it extends from `Attribute` and then register it to `Attributes` registry.

```ruby
class AwesomeAttribute < Bluepine::Attributes::Attribute
  # codes ...
end

# Register it
Bluepine::Attributes.register(:awesome, AwesomeAttribute)
```

Later, we can refer to it like the following.

```ruby
schema :user do
  string  :email
  awesome :cool  # our custom attribute
end
```

## Resolver

`Resolver` acts as a registry that holds references to all `schemas` and `endpoints` that we've defined.

### Manually registering schema/endpoint

```ruby
user_schema = create_user_schema

# pass it to the constructor
resolver = Bluepine::Resolver.new(schemas: [user_schema], endpoints: [])

# or use `#schemas` method
resolver.schemas.register(:user, user_schema)
```

### Automatically registering schema/endpoint

Although we can create a schema and register it manually. It'll become a tedious tasks when there're lot of schemas/endpoints to work with.

```ruby
resolver = Bluepine::Resolver.new do

  # schema is just `ObjectAttribute`
  schema :user do
    # codes
  end

  schema :group do
    # codes
  end

  endpoint "/users" do
    # codes
  end
end
```

## Serialization

`Serializer` was designed in the way that it can serialize any type of `Attribute`. Either it's simple attribute type such as `StringAttribute` or a more complex type like `ObjectAttribute`. The `Serializer` treats it the same way.

### <a name="serializer-example"></a> Example
#### Serializing a simple type

```ruby
attr = Bluepine::Attributes.create(:string, :email)

serializer.serialize(attr, 3.14) # => "3.14"
```

#### Serializing `Array`

```ruby
attr = Bluepine::Attributes.create(:array, :heroes)

serializer.serialize(attr, ["Iron Man", "Thor"]) # => ["Iron Man", "Thor"]
```

#### Serializing `Object`

When serializing object, the data that we want to serialize can be a `Hash` or plain `Object`.

In the following example. We serialize an instance of `Hero` class.

```ruby
attr = Bluepine::Attributes.create(:object, :hero) do
  string :name
  number :power, default: 5
end

# Defines our class
class Hero
  attr_reader :name, :power

  def initialize(name:, power: nil)
    @name  = name
    @power = power
  end

  def name
    "I'm #{@name}"
  end
end

thor = Hero.new(name: "Thor")

# Serializes
serializer.serialize(attr, thor) # =>

{
  name: "I'm Thor",
  power: 5
}
```

### <a name="serializer-options"></a> Options
#### <a name="serializer-options-method"></a> `:method`

*Value: Symbol* - Alternative method name

We can use this option to specify which method of the target object that will be used to get the data from.

```ruby
# Our schema
schema :hero do
  string :name, method: :awesome_name
end

class Hero
  def initialize(name)
    @name = name
  end

  def awesome_name
    "I'm super #{@name}!"
  end
end

hero = Hero.new(name: "Thor")

# Serializes
serializer.serialize(hero_schema, hero)
```

will produce the following result.
```
{
  "name": "I'm super Thor!"
}
```

#### <a name="serializer-options-of"></a> `:of`

*Value: `Symbol`* - Attribute type or Schema name e.g. `:string` or `:user`

This option allows us to refer to other schema from `array` or `schema` attribute.

In the following example. We'll re-use our previously defined `:hero` schema with our new `:team` schema.

```ruby
schema :team do
  array :heroes, of: :hero
end

class Team
  attr_reader :name, :heroes

  def initialize(name: name, heroes: heroes)
    @name   = name
    @heroes = heroes
  end
end

team = Team.new(name: "Avengers", heroes: [
  Hero.new(name: "Thor"),
  Hero.new(name: "Hulk", power: 10),
])

# Serializes
serializer.serialize(team_schema, team)
```

will produce the following result

```ruby
{
  name: "Avengers",
  heroes: [
    { name: "Thor", power: 5 }, # 5 is default value from hero schema
    { name: "Hulk", power: 10 },
  ]
}
```

#### <a name="serializer-options-private"></a> `:private`

*Value: `Boolean`* - Default is `false`

When it's set to `true`. It'll exclude that attribute from serializer's result.

```ruby
schema :hero do
  string :name
  number :secret_power, private: true
end

hero = Hero.new(name: "Peter", secret_power: 99)
serializer.serialize(hero_schema, hero)
```

will exclude `secret_power` from result

```ruby
{
  name: "Peter"
}
```

### Conditional Serialization
#### <a name="serializer-options-private"></a> `:if/:unless`

*Possible value: `Symbol`/`Proc`*

This enables us to serialize value based on `if/unless` conditions.

```ruby
schema :hero do
  string :name

  # :mode'll get serialized only when `dog_dead` is true
  string :mode, if: :dog_dead

  # or we can use `Proc` e.g.
  # string :mode, if: ->(x) { x.dog_dead }
  boolean :dog_dead, default: false
end

hero = Hero.new(name: "John Wick", mode: "Angry")
serializer.serialize(hero_schema, hero) # =>
```

will produce

```ruby
{
  name: "John Wick",
  dog_dead: false
}
```
But if we set `dog_dead: true` the result will include `mode` value.

```ruby
{
  name: "John Wick",
  mode: "Angry",
  dog_dead: true,
}
```

### Custom Serializer

By default, each primitive types e.g. `string`, `integer`, etc. has its own serializer. We can override it by overriding `.serializer` class method.

For example. If we want to extend `boolean` attribute to treat "**on**" as a valid boolean value. We could do it like this.

```ruby
BooleanAttribute.normalize = ->(x) { ["on", true].include?(x) ? true : false }

# Usage
schema :team do
  boolean :active
end

team = Team.new(active: "on")
serializer.serialize(team_schema, team)
```

will produce

```ruby
{
  active: true
}
```

## Endpoint

Endpoint represents the API endpoint and it's operations e.g. `GET`, `POST`, etc.
It groups resource's related operations together and defines a set of valid parameters that the endpoint accepts.

### Defining Endpoint

We could define it manually as follows

```ruby
Bluepine::Endpoint.new "/users" do
  get :read, path: "/:id"
end
```

or defines it via `Resolver`

```ruby
Bluepine::Resolver.new do
  endpoint "/heroes" do
    post :create, path: "/"
  end

  endpoint "/teams" do
    # code
  end
end
```

### <a name="endpoint-method"></a> Method

Endpoint provides a set of http methods such as `get`, `post`, `patch`, `delete`, etc.
Each method expects a name and some other options.

> Note that name must be unique within endpoint

```ruby
method(name, path:, params:)

# e.g.
get  :read,   path: "/:id"
post :create, path: "/"
```

### <a name="endpoint-params"></a> Params

`Params` allows us to define a set of valid parameters accepted by the Endpoint's methods (e.g. `get`, `post`, etc).

We can think of `Params` the same way as `Schema` (i.e. `ObjectAttribute`). It's just a specialize version of `ObjectAttribute`.

#### Defining default params

```ruby
endpoint "/users" do
  # declare default params
  params do
    string :username
    string :password
  end

  # `params: true` will use default params for validating incoming request
  post  :create, params: true

  # this will re-use `username` param from default params
  patch :update, params: %i[username]
end
```

#### Using no params `params: false` (default behaviour)

If we don't want our endpoint's method to re-use default params. We can specify `params: false` to endpoint method's arguments.

> Note: this is the default behaviour. So we can leave it blank.

```ruby
get :index, path: "/" # ignore `params` means `params: false`
```

#### Using default params `params: true`

As we've seen in the example above. Set `params: true` indicates that we want to use default params for this method.

```ruby
post :create, path: "/", params: true
```

#### Using subset of default params's attributes `params: %i[...]`

Let's say we want to use only some of default params's attrbiute e.g. `currency` (but not other attributes). We can specify it like this.

```ruby
patch :update, path: "/:id", params: %i[currency]
```

In this case it will re-use only `currency` attribute for validation.

#### Excluding some of default params's attributes `exclude: true`

Let's say the `update` method doesn't need the `default params`'s `amount` attribute (but still want to use all other attributes). We can specify it as follows.

```ruby
patch :update, path: "/:id", params: %i[amount], exclude: true
```

#### Overriding default params with `params: Proc`

In the case where we want to completely use the new set of params. We can use `Proc` to define it like the following.

```ruby
# inside schema.endpoint block
patch :update, path: "/:id", params: lambda {
  integer :max_amount, required: true
  string  :new_currency, match: /\A[a-z]{3}\z/
}
```

It will use these new params for validating/generating specs.

#### Re-using Params from Other Service `params: Symbol`

We can also re-use params from other endpoint by specifing a `Symbol` that refers to other endpoint's params.

```ruby
endpoint "/search" do
  params do
    string :query
    number :limit
  end
end

endpoint "/blogs" do
  get :index, path: "/", params: :search
end
```

Here we will use `search` endpoint's default params for validating `GET /users` endpoint.

### Endpoint Validation

See [Validation - Validating Endpoint](#validating-endpoint)

## Validation

Once we have our schema/endpoint defined. We can use validator to validate it against any data. (it uses `ActiveModel::Validations` under the hood)

Similar to `Serializer`. We can use `Validator` to validate any type of `Attribute`.

### Example
#### Validating simple attribute

```ruby
attr  = Bluepine::Attributes.create(:string, :email)
email = true

validator.validate(attr, email) # => Result object
```

In this case, it'll just return a `Result.errors` which contain error message

```ruby
["is not string"]
```

#### Validating `Array`

```ruby
attr  = Bluepine::Attributes.create(:array, :names, of: :string)
names = ["john", 1, "doe"]

validator.validate(attr, names) # => Result object
```

It'll return the error messages at exact index position.

```ruby
{
  1 => ["is not string"]
}
```

#### Validating `Object`
Most of the time, we'll work with the object type (instead of simple type like `string`, etc).
```ruby
attr  = Bluepine::Attributes.create(:object, :user) do
  string :username, min: 4
  string :password, min: 10
end

user = {
  username: "john",
  password: true,
}

validator.validate(attr, user) # => Result object
```
Since it's an object, the errors will contain attribute names

```ruby
{
  password: [
    "is not string",
    "is too short (minimum is 10 characters)"
  ]
}
```

### Options

#### <a name="validator-options-required"></a> `:required`

*Value: `Boolean`* - Default is `false`

This option makes the attribute mandatory.

```ruby
schema :hero do
  string :name, required: true
end

hero = Hero.new
validator.validate(hero_schema, hero) # => Result.errors
```

will return

```ruby
{
  name: ["can't be blank"]
}
```

#### <a name="validator-options-match"></a> `:match`

*Value: `Regexp`* - Regular Expression to be tested.

This option will test if string matches against given regular expression or not.

```ruby
schema :hero do
  string :name, match: /\A[a-zA-Z]+\z/
end

hero = Hero.new(name: "Mark 3")
validator.validate(hero_schema, hero) # => Result.errors
```

will return

```ruby
{
  name: ["is not valid"]
}
```

#### <a name="validator-options-min-max"></a> `:min/:max`

*Value: `Number`* - Apply to both `string` and `number` attribute types.

This option sets a minimum and maximum value for attribute.

```ruby
schema :hero do
  string :power, max: 100
end

hero = Hero.new(power: 200)
validator.validate(hero_schema, hero) # => Result.errors
```

will return

```ruby
{
  power: ["must be less than or equal to 100"]
}
```

#### <a name="validator-options-in"></a> `:in`

*Value: `Array`* - Set of valid values.

This option will test if value is in the specified list or not.

```ruby
schema :hero do
  string :status, in: ["Happy", "Angry"]
end

hero = Hero.new(status: "Mad")
validator.validate(hero_schema, hero) # => Result.errors
```

will return

```ruby
{
  status: ["is not included in the list"]
}
```

### <a name="validator-condition"></a> Conditional Validation
#### <a name="validator-options-if-unless"></a> `:if/:unless`

*Possible value: `Symbol`/`Proc`*

This enables us to validate attribute based on `if/unless` conditions.

```ruby
schema :hero do
  string :name

  # or we can use `Proc` e.g.
  # if: ->(x) { x.is_agent }
  string :agent_name, required: true, if: :is_agent

  boolean :agent, default: false
end

hero = Hero.new(name: "Nick Fury", is_agent: true)
validator.validate(hero_schema, hero) # Result.errors =>
```

will produce (because `is_agent` is `true`)

```ruby
{
  agent_name: ["can't be blank"]
}
```

### Custom Validator

Since the validator is based on `ActiveModel::Validations`. This make it easy to add a new custom validator.

In the following example. We create a simple password validator and register it to the password attribute.

```ruby
# Defines custom validator
class CustomPasswordValidator < ActiveModel::Validator
  def validate(record)
    record.errors.add(:password, "is too short") unless record.password.length > 10
  end
end

# Registers
schema :user do
  string :username
  string :password, validators: [CustomPasswordValidator]
end
```

### Custom Normalizer

It's possible to change the logic for normalizing data before passing it to the validator. For example, you might want to normalize `boolean` value before validating it.

Here, we want to normalize string such as `on` or `1` to boolean `true` first.

```ruby
# Overrides default normalizer
BooleanAttribute.normalizer = ->(x) { [true, 1, "on"].include?(x) ? true : false }

schema :hero do
  boolean :berserk
end

hero = Hero.new(berserk: 1)
validator.validate(hero_schema, hero) # Result.value
```
will pass the validation and `Result.value`'ll contain normalized value

```ruby
{
  berserk: true # convert 1 to true
}
```

### Validating `Endpoint`

All examples above also apply to endpoint's parameters validation.

Because the params is part of `Endpoint` and it's non-trivial task to retrieve endpoint's method's params. So, the `Endpoint` provides some helper methods to validate the data.

```ruby
resolver = Bluepine::Resolver.new do
  endpoint "/heroes" do
    post :create, params: lambda {
      string :name, required: true
    }
  end
end

# :create is a POST method name given to the endpoint.
resolver.endpoint(:heroes).method(:create, resolver: resolver).validate(payload) # => Result
```

## Generating Open API (v3)

Once we have all schemas/endpoints defined and registered to the `Resolver`. We can simply pass it to the generator as follows.

```ruby
generator = Bluepine::Generators::OpenAPI::Generator.new(resolver, options)
generator.generate # =>
```

will output Open API (v3) specs

*excerpt from the full result*
```js
  // endpoints
  "/users": {
    "post": {
      "requestBody": {
        "content": {
          "application/x-www-form-urlencoded": {
            "schema": {
              "type": "object",
              "properties": {
                "username": {
                  "type": "string"
                },
                "accepted": {
                  "type": "boolean",
                  "enum": [true, false]
                },
              }
            }
          }
        }
      },
      "responses": {
        "200": {
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/user"
              }
            }
          }
        }
      }
    }
  }

  // schema
  "user": {
    "type": "object",
    "properties": {
      "address": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string",
            "default": "Bangkok"
          }
        }
      },
      "friends": {
        "type": "array",
        "items": {
          "$ref": "#/components/schemas/user"
        }
      }
    }
  }
```

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/omise/bluepine. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
