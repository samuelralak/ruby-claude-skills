# dry-rb Complete Reference

## dry-types -- Type System

### Setup

```ruby
# app/types.rb
require "dry-types"

module Types
  include Dry.Types()
end
```

### Built-in Type Categories

**Strict types** (default) -- raise on wrong type:
```ruby
Types::String["hello"]          # => "hello"
Types::String[123]              # => Dry::Types::ConstraintError
Types::Integer[1]               # => 1
Types::Bool[true]               # => true
```

**Coercible types** -- convert via Ruby kernel methods:
```ruby
Types::Coercible::String[123]     # => "123"
Types::Coercible::Integer["42"]   # => 42
Types::Coercible::Float["3.14"]   # => 3.14
Types::Coercible::Symbol["foo"]   # => :foo
```

**Params types** -- for HTTP parameter coercion:
```ruby
Types::Params::Integer["42"]      # => 42
Types::Params::Bool["true"]       # => true
Types::Params::Bool["1"]          # => true
Types::Params::Date["2025-01-15"] # => #<Date: 2025-01-15>
```

**JSON types** -- for JSON payloads:
```ruby
Types::JSON::Date["2025-01-15"]   # => #<Date: 2025-01-15>
Types::JSON::Decimal[3.14]        # => #<BigDecimal:...>
```

### Constraints

```ruby
Types::String.constrained(min_size: 1)                    # non-empty
Types::String.constrained(max_size: 255)                  # max length
Types::String.constrained(format: /\A[a-z]+\z/)           # regex
Types::Integer.constrained(gt: 0)                         # greater than
Types::Integer.constrained(gteq: 0, lteq: 150)           # range
Types::String.constrained(included_in: %w[a b c])         # enum
Types::Integer.constrained(excluded_from: [0])            # exclusion
Types::String.constrained(size: 5)                        # exact size
```

### Sum Types

```ruby
StringOrInteger = Types::String | Types::Integer
NilableString   = Types::Nil | Types::String
# Shorthand:
Types::String.optional  # same as Types::Nil | Types::String
```

### Array and Map Types

```ruby
Types::Array.of(Types::String)                # array of strings
Types::Hash.map(Types::Symbol, Types::String) # symbol => string map
```

### Enum Types

```ruby
PostStatus = Types::String.enum("draft", "published", "archived")
PostStatus["draft"]     # => "draft"
PostStatus["invalid"]   # => ConstraintError
PostStatus.values       # => ["draft", "published", "archived"]

# With mapping
CellState = Types::String.enum("locked" => 0, "open" => 1)
CellState[0]            # => "locked"
```

### Custom Type Constructors

```ruby
StrippedString = Types.Constructor(String) { |v| v.to_s.strip }
Email = Types::String.constructor { |s| s&.strip&.downcase }
  .constrained(format: /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z]+)*\.[a-z]+\z/i)

# Interface (duck typing)
Callable = Types.Interface(:call)

# Instance check
JSONHash = Types.Instance(Hash)
```

---

## dry-struct -- Typed Value Objects

### Basic Usage

```ruby
class UserData < Dry::Struct
  attribute :name,  Types::Strict::String
  attribute :age,   Types::Coercible::Integer
  attribute :email, Types::String.optional     # allows nil, key required
end

user = UserData.new(name: "Jane", age: "30", email: nil)
user.name   # => "Jane"
user.age    # => 30 (coerced)
```

### Optional Keys

```ruby
class UserData < Dry::Struct
  attribute  :name,  Types::String              # required key, must be String
  attribute  :email, Types::String.optional      # required key, can be nil
  attribute? :phone, Types::String.optional      # optional key entirely
end
```

### Defaults

```ruby
class UserData < Dry::Struct
  attribute :role,   Types::String.default("member")
  attribute :active, Types::Bool.default(true)
  attribute :tags,   Types::Array.of(Types::String).default([].freeze)
end
```

### Nested Structs

```ruby
class Order < Dry::Struct
  attribute :id, Types::String

  attribute :address do
    attribute :street, Types::String
    attribute :city,   Types::String
  end

  attribute :items, Types::Array do
    attribute :product_id, Types::String
    attribute :quantity,   Types::Integer
  end
end
```

### Transform Keys (accept string keys)

```ruby
class FlexibleStruct < Dry::Struct
  transform_keys(&:to_sym)
end
```

---

## dry-validation -- Contracts

### Contract Structure

```ruby
class CreateUserContract < Dry::Validation::Contract
  params do
    required(:name).filled(:string, min_size?: 2)
    required(:email).filled(:string)
    optional(:phone).maybe(:string)
  end

  rule(:email) do
    key.failure("has invalid format") unless /\A[\w+\-.]+@/i.match?(value)
  end
end
```

### Schema Types

- `params` -- HTTP form data, coerces strings (`"42"` -> `42`)
- `json` -- JSON API payloads, expects typed values
- `Dry::Schema.define` -- plain hash, no coercion

### Key Macros

```ruby
required(:name).filled(:string)                  # must exist, must be filled
required(:email).maybe(:string)                  # must exist, can be nil
optional(:phone).filled(:string)                 # may be absent; if present, must be filled
optional(:notes).maybe(:string)                  # may be absent; if present, can be nil
```

### Predicates

```ruby
required(:age).filled(:integer, gt?: 17)
required(:name).filled(:string, min_size?: 2, max_size?: 50)
required(:status).filled(:string, included_in?: %w[active inactive])
required(:score).filled(:float, gteq?: 0.0, lteq?: 5.0)
```

### Nested Schemas

```ruby
params do
  required(:address).hash do
    required(:city).filled(:string)
    required(:zip).filled(:string)
  end
  required(:items).array(:hash) do
    required(:product_id).filled(:integer)
    required(:quantity).filled(:integer, gt?: 0)
  end
end
```

### Rules

```ruby
# Single field
rule(:email) do
  key.failure("is taken") if User.exists?(email: value)
end

# Cross-field
rule(:end_date, :start_date) do
  key.failure("must be after start date") if values[:end_date] <= values[:start_date]
end

# Nested key
rule(address: :zip) do
  key.failure("must be 5 digits") unless value.match?(/\A\d{5}\z/)
end

# Base-level (not tied to a field)
rule do
  base.failure("invalid") if suspicious?(values)
end
```

### Macros (Reusable Rules)

```ruby
class ApplicationContract < Dry::Validation::Contract
  register_macro(:email_format) do
    unless /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z]+)*\.[a-z]+\z/i.match?(value)
      key.failure("is not a valid email")
    end
  end
end

class UserContract < ApplicationContract
  params do
    required(:email).filled(:string)
  end

  rule(:email).validate(:email_format)
end
```

### Dependency Injection in Contracts

```ruby
class UniqueEmailContract < Dry::Validation::Contract
  option :user_repo

  params do
    required(:email).filled(:string)
  end

  rule(:email) do
    key.failure("is taken") if user_repo.exists?(email: value)
  end
end

contract = UniqueEmailContract.new(user_repo: UserRepository.new)
```

---

## dry-initializer -- Constructor DSL

### param vs option

```ruby
class MyClass
  extend Dry::Initializer

  param  :name                           # positional, required
  option :role, default: proc { "user" } # keyword, with default
  option :admin, default: proc { false }
end

MyClass.new("Jane", role: "admin")
```

### Type Constraints

```ruby
class MyService
  extend Dry::Initializer

  option :amount,   type: Types::Strict::Decimal
  option :currency, type: Types::String.constrained(included_in: %w[USD EUR])
  option :user,     type: Types.Instance(User)
  option :tags,     type: Types::Array.of(Types::String), default: proc { [] }
end
```

### Reader Visibility

```ruby
option :api_key, reader: :private
option :secret,  reader: :protected
option :internal, reader: false       # no reader, stored in @internal
```

### Optional (truly optional, no default)

```ruby
option :callback, optional: true      # can be omitted entirely
```

---

## dry-configurable -- Configuration Mixin

```ruby
class App
  extend Dry::Configurable

  setting :database_url, default: "sqlite:memory"
  setting :log_level, default: :info

  setting :smtp do
    setting :host, default: "localhost"
    setting :port, default: 25
  end
end

App.configure do |config|
  config.database_url = "postgres://localhost/mydb"
  config.smtp.host = "smtp.example.com"
end

App.config.database_url  # => "postgres://localhost/mydb"
```

---

## Gem Version Reference

```ruby
# Gemfile
gem "dry-validation",  "~> 1.10"
gem "dry-types",       "~> 1.7"
gem "dry-initializer", "~> 3.1"
gem "dry-struct",      "~> 1.6"
gem "dry-configurable", "~> 1.2"
gem "dry-auto_inject", "~> 1.0"   # if using DI containers
gem "dry-system",      "~> 1.0"   # if using application container
```
