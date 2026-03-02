---
name: ruby-rails
description: "REQUIRED for all Ruby and Ruby on Rails development. Use when: writing Ruby code, creating Rails models/controllers/services, running migrations, generating scaffolds, working with ActiveRecord, configuring gems, writing RSpec tests, setting up dry-rb gems, creating service objects, working with concerns/modules, or any task involving .rb files or a Rails project. Triggers: Ruby, Rails, ActiveRecord, RSpec, dry-types, dry-validation, Gemfile, migration, model, controller, service object, concern, RuboCop, Sidekiq, background jobs."
---

# Ruby on Rails Development Skill

Build clean, simple, idiomatic Ruby on Rails applications using dry-rb for validation and typing, strict conventions, and generators.

## Core Principles

1. **Clean, simple, idiomatic code** -- Write Ruby that reads like English. Prefer clarity over cleverness. No over-engineering.
2. **Generators first** -- NEVER manually create files that Rails generators can produce (migrations, models, controllers, jobs, mailers, etc.).
3. **UUIDs everywhere** -- All primary keys and foreign keys use UUIDs by default.
4. **Linting is mandatory** -- Every project must have RuboCop properly configured before writing code.
5. **dry-rb for validation and typing** -- Use dry-validation (contracts), dry-types (type system), dry-initializer (typed constructor DSL).
6. **Thin models, thin controllers** -- Business logic lives in services. Models stay under 100 lines.
7. **Plain Ruby services** -- Services return values on success, raise exceptions on failure. No monads, no wrappers.

---

## Mandatory Checks Before Writing Code

Before writing ANY code in a Rails project, verify:

1. **Linter configured?** -- Check for `.rubocop.yml`. If missing, set it up first.
2. **UUID configured?** -- Check `config/initializers/generators.rb` for UUID primary key config. If missing, set it up first.
3. **Types module exists?** -- Check for `app/types.rb`. If missing, create it.
4. **Base service exists?** -- Check for `app/services/base_service.rb`. If missing, create it.
5. **Error classes exist?** -- Check for `app/errors/`. If missing, create them.

---

## Full Project Structure

```
app/
  contracts/                            # dry-validation contracts
    application_contract.rb
    {domain}/
      {action}_contract.rb
  controllers/
    application_controller.rb
    concerns/
      error_handler.rb                  # rescue_from for exceptions
    api/
      v1/
        {resource}_controller.rb
  errors/                               # Custom error classes
    service_error.rb
    validation_error.rb
    not_found_error.rb
  models/
    application_record.rb
    {model}.rb                          # Thin model (<100 lines)
    {model_plural}/                     # Class methods AND constants
      {feature}.rb                      # extend in model
      {constants}.rb                    # extend in model
    concerns/
      {model_plural}/
        {feature}able.rb                # include in model
  services/
    base_service.rb
    {domain}/
      {action}.rb                       # Main service (entry point)
      actions/
        {action}.rb                     # Internal command
      performers/
        {noun}_performer.rb             # Complex background worker
  jobs/
    application_job.rb
    {action}_job.rb
  mailers/
    application_mailer.rb
  types.rb                              # dry-types module
config/
  initializers/
    generators.rb                       # UUID config
```

---

## Rails Generators -- ALWAYS Use Them

**NEVER manually create files for anything Rails provides a generator for.**

```bash
# Models -- UUID is auto-configured via generators.rb, no flag needed
bin/rails generate model User name:string email:string

# Migrations
bin/rails generate migration AddPhoneToUsers phone:string
bin/rails generate migration CreateJoinTableUsersRoles users roles

# Controllers
bin/rails generate controller Api::V1::Users index show create update destroy

# Jobs
bin/rails generate job ProcessPayment

# Mailers
bin/rails generate mailer UserMailer welcome reset_password
```

### Migration Rules

- ALWAYS use `bin/rails generate migration` -- never create migration files by hand
- After generating, edit the migration to add constraints, indexes, or UUID references
- Always specify `dependent:` on `has_many` associations
- Use `null: false` on columns that should never be nil
- Add indexes on all foreign keys and frequently queried columns
- Ensure all `t.references` use `type: :uuid`

```ruby
# frozen_string_literal: true

class CreateOrders < ActiveRecord::Migration[7.2]
  def change
    create_table :orders, id: :uuid do |t|
      t.references :user, null: false, foreign_key: true, type: :uuid
      t.string :status, null: false, default: "pending"
      t.timestamps
    end

    add_index :orders, :status
  end
end
```

Note: Replace `[7.2]` with your project's actual Rails version.

---

## UUID Configuration

### Generator Configuration

```ruby
# frozen_string_literal: true

# config/initializers/generators.rb
Rails.application.config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end
```

### Enable pgcrypto Extension

```bash
bin/rails generate migration EnablePgcrypto
```

Then edit:
```ruby
# frozen_string_literal: true

class EnablePgcrypto < ActiveRecord::Migration[7.2]
  def change
    enable_extension "pgcrypto" unless extension_enabled?("pgcrypto")
  end
end
```

---

## Linting & Formatting -- Mandatory

### Gemfile

```ruby
group :development, :test do
  gem "rubocop", require: false
  gem "rubocop-rails", require: false
  gem "rubocop-rspec", require: false
  gem "rubocop-performance", require: false
  gem "brakeman", require: false
end
```

### .rubocop.yml

```yaml
require:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance

AllCops:
  NewCops: enable
  TargetRubyVersion: 3.2
  Exclude:
    - "db/schema.rb"
    - "db/migrate/*"
    - "bin/**/*"
    - "node_modules/**/*"
    - "vendor/**/*"

Style/Documentation:
  Enabled: false

Style/FrozenStringLiteralComment:
  Enabled: true
  EnforcedStyle: always

Metrics/MethodLength:
  Max: 15

Metrics/ClassLength:
  Max: 100

Layout/LineLength:
  Max: 120
```

### Running

```bash
bundle exec rubocop --autocorrect          # lint
bundle exec brakeman                       # security scan
```

---

## Model Guidelines

### File Size Rules
- Model files: **<100 lines**
- Extract to concern when: >50 lines of instance methods
- Extract to module when: >3 related constants

### Model File Order

```ruby
# frozen_string_literal: true

class Event < ApplicationRecord
  extend Events::Lookup         # 1. Class methods (extend)
  include Events::Filterable    # 2. Instance methods (include)

  belongs_to :user                                    # 3. Associations
  has_many :tags, dependent: :destroy

  validates :name, presence: true                     # 4. Validations

  scope :active, -> { where(active: true) }           # 5. Scopes
end
```

### Naming Conventions

| Type | Pattern | Location | Include Method |
|------|---------|----------|----------------|
| Concern (instance methods) | `{Feature}able` | `app/models/concerns/{model_plural}/` | `include` |
| Class methods module | `{Feature}` | `app/models/{model_plural}/` | `extend` |
| Constants module | `{Noun}s` | `app/models/{model_plural}/` | `extend` |
| Main service | `{Domain}::{Action}` | `app/services/{domain}/` | -- |
| Internal action | `{Domain}::Actions::{Action}` | `app/services/{domain}/actions/` | -- |
| Performer | `{Domain}::{Noun}Performer` | `app/services/{domain}/performers/` | -- |
| Contract | `{Domain}::{Action}Contract` | `app/contracts/{domain}/` | -- |

### Decision Trees

**Where does this model code go?**
```
Is it a class method?    -> app/models/{model_plural}/{feature}.rb (use extend)
Is it an instance method? -> app/models/concerns/{model_plural}/{feature}able.rb (use include)
Is it a constant?         -> app/models/{model_plural}/{constants}.rb (use extend)
Is it business logic?     -> app/services/{domain}/{action}.rb
```

**What service tier?**
```
Called from job/controller? -> Main service: app/services/{domain}/{action}.rb
Called only by services?    -> Action: app/services/{domain}/actions/{action}.rb
Complex background worker?  -> Performer: app/services/{domain}/performers/{noun}_performer.rb
```

### Concern Format (Zeitwerk-compatible)

Use the **nested module** form for Zeitwerk autoloading:

```ruby
# app/models/concerns/events/filterable.rb
# frozen_string_literal: true

module Events
  module Filterable
    extend ActiveSupport::Concern

    def apply_filter(criteria)
      # ...
    end
  end
end
```

### Class Methods Module

```ruby
# app/models/events/lookup.rb
# frozen_string_literal: true

module Events
  module Lookup
    def find_by_pubkey(pubkey)
      where(pubkey:).first
    end
  end
end
```

### Anti-Patterns

WRONG -- Concern without `-able` suffix:
```ruby
module Events::Filter; end           # WRONG name and form
```

WRONG -- Class methods in concerns:
```ruby
# concerns/events/lookup.rb
module Events::Lookup
  def self.find_by_pubkey(pubkey); end  # WRONG: class method in concern dir
end
```

WRONG -- `has_many` without `dependent:`:
```ruby
has_many :tags                          # WRONG: must specify dependent:
```

---

## Service Guidelines

### Required Gems

```ruby
gem "dry-validation",  "~> 1.10"
gem "dry-types",       "~> 1.7"
gem "dry-initializer", "~> 3.1"
gem "dry-struct",      "~> 1.6"
```

### Foundation Files

```ruby
# app/types.rb
# frozen_string_literal: true

require "dry-types"

module Types
  include Dry.Types()
end
```

```ruby
# app/services/base_service.rb
# frozen_string_literal: true

class BaseService
  extend Dry::Initializer

  def self.call(...)
    new(...).call
  end
end
```

```ruby
# app/contracts/application_contract.rb
# frozen_string_literal: true

class ApplicationContract < Dry::Validation::Contract
  config.messages.default_locale = :en
end
```

### Custom Error Classes

```ruby
# app/errors/service_error.rb
# frozen_string_literal: true

class ServiceError < StandardError; end
```

```ruby
# app/errors/validation_error.rb
# frozen_string_literal: true

class ValidationError < ServiceError
  attr_reader :errors

  def initialize(errors)
    @errors = errors
    super("Validation failed")
  end
end
```

```ruby
# app/errors/not_found_error.rb
# frozen_string_literal: true

class NotFoundError < ServiceError; end
```

### Service Example

Services return values on success, raise on failure. No wrappers.

```ruby
# app/services/users/create.rb
# frozen_string_literal: true

module Users
  class Create < BaseService
    option :name,  type: Types::Strict::String
    option :email, type: Types::Strict::String

    def call
      values = validate!
      user = User.create!(values)
      UserMailer.welcome(user).deliver_later
      user
    end

    private

    def validate!
      result = Users::CreateContract.new.call(name:, email:)
      raise ValidationError, result.errors.to_h unless result.success?

      result.to_h
    end
  end
end
```

### With Transactions

```ruby
def call
  ActiveRecord::Base.transaction do
    account = Account.lock.find_by!(id: from_id)
    target  = Account.lock.find_by!(id: to_id)
    account.update!(balance: account.balance - amount)
    target.update!(balance: target.balance + amount)
    { from: account, to: target, amount: }
  end
end
```

`find_by!` raises `RecordNotFound`, `update!` raises `RecordInvalid` -- ErrorHandler catches both.

### Error Handler Concern

Catches all exceptions globally so controllers stay clean:

```ruby
# app/controllers/concerns/error_handler.rb
# frozen_string_literal: true

module ErrorHandler
  extend ActiveSupport::Concern

  included do
    rescue_from ActiveRecord::RecordNotFound do |e|
      render json: { error: e.message }, status: :not_found
    end

    rescue_from ActiveRecord::RecordInvalid do |e|
      render json: { errors: e.record.errors.full_messages }, status: :unprocessable_entity
    end

    rescue_from ActionController::ParameterMissing do |e|
      render json: { error: e.message }, status: :bad_request
    end

    rescue_from ValidationError do |e|
      render json: { errors: e.errors }, status: :unprocessable_entity
    end

    rescue_from NotFoundError do |e|
      render json: { error: e.message }, status: :not_found
    end
  end
end
```

### ApplicationController

```ruby
# app/controllers/application_controller.rb
# frozen_string_literal: true

class ApplicationController < ActionController::API
  include ErrorHandler
end
```

### Controller Example

Controllers are trivial: call service, render result. Errors are handled by ErrorHandler.

```ruby
# app/controllers/api/v1/users_controller.rb
# frozen_string_literal: true

module Api
  module V1
    class UsersController < ApplicationController
      before_action :set_user, only: %i[show update destroy]

      def index
        users = Users::List.call
        render json: users
      end

      def show
        render json: @user
      end

      def create
        user = Users::Create.call(**user_params)
        render json: user, status: :created
      end

      def update
        user = Users::Update.call(user: @user, **user_params)
        render json: user
      end

      def destroy
        Users::Destroy.call(user: @user)
        head :no_content
      end

      private

      def set_user
        @user = User.find(params[:id])
      end

      def user_params
        params.require(:user).permit(:name, :email).to_h.symbolize_keys
      end
    end
  end
end
```

`set_user` uses `find` -- ErrorHandler catches `RecordNotFound` and renders 404 automatically.

---

## Validation with dry-validation

```ruby
# app/contracts/users/create_contract.rb
# frozen_string_literal: true

module Users
  class CreateContract < ApplicationContract
    params do
      required(:name).filled(:string, min_size?: 2)
      required(:email).filled(:string)
      required(:age).filled(:integer, gt?: 17)
      optional(:phone).maybe(:string)
    end

    rule(:email) do
      unless /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z]+)*\.[a-z]+\z/i.match?(value)
        key.failure("has invalid format")
      end
    end
  end
end
```

- Use `params` for HTML form data (string coercion)
- Use `json` for API payloads (typed values, minimal coercion)
- Rules run AFTER schema validation passes
- See `references/dry-rb.md` for full dry-validation API

---

## Testing

### Gemfile

```ruby
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end

group :test do
  gem "shoulda-matchers"
end
```

### Factory Example

```ruby
# spec/factories/users.rb
# frozen_string_literal: true

FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    email { Faker::Internet.email }
  end
end
```

### Testing Services

```ruby
# frozen_string_literal: true

RSpec.describe Users::Create do
  subject(:result) { described_class.call(**params) }

  let(:params) { { name: "Jane", email: "jane@example.com" } }

  context "with valid params" do
    it "creates a user" do
      expect(result).to be_a(User)
      expect(result).to be_persisted
    end
  end

  context "with invalid email" do
    let(:params) { { name: "Jane", email: "invalid" } }

    it "raises ValidationError" do
      expect { result }.to raise_error(ValidationError) do |e|
        expect(e.errors).to have_key(:email)
      end
    end
  end
end
```

### Testing Contracts

```ruby
# frozen_string_literal: true

RSpec.describe Users::CreateContract do
  subject(:result) { described_class.new.call(params) }

  context "with valid params" do
    let(:params) { { name: "Jane", email: "jane@example.com", age: 25 } }

    it { is_expected.to be_success }
  end

  context "when name is missing" do
    let(:params) { { email: "jane@example.com", age: 25 } }

    it "returns error for name" do
      expect(result.errors.to_h).to include(name: ["is missing"])
    end
  end
end
```

### Testing Models

```ruby
# frozen_string_literal: true

RSpec.describe User do
  describe "associations" do
    it { is_expected.to have_many(:orders).dependent(:destroy) }
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:name) }
  end
end
```

---

## Idiomatic Ruby Style Rules

- `frozen_string_literal: true` in **every** `.rb` file
- Short methods (max 15 lines)
- Guard clauses over nested conditionals: `return if invalid?`
- `{ }` for single-line blocks, `do...end` for multi-line blocks
- Shorthand hash syntax: `{ name:, email: }` (Ruby 3.1+)
- Endless methods for simple one-liners: `def name = @name`
- `&.` (safe navigation) over `try` or nil checks
- `.present?` / `.blank?` over nil/empty checks in Rails
- `find_each` for iterating large datasets (not `each`)
- Single quotes unless string interpolation is needed
- `unless` for simple negative conditions
- No unnecessary comments, dead code, or commented-out code

---

## Database Best Practices

- **UUIDs** for all primary and foreign keys (configured globally)
- **Indexes** on all foreign keys and frequently queried columns
- **`null: false`** on columns that should never be nil
- **`unique: true`** constraints at DB level, not just model validations
- **Reversible migrations** -- avoid `execute` without `reversible` block
- **`find_each`** for batch processing, never `.all.each`
- **Eager loading** -- use `includes`/`eager_load`/`preload` to prevent N+1 queries
- **`dependent:`** always specified on `has_many` associations

### N+1 Prevention

WRONG:
```ruby
users = User.all
users.each { |u| puts u.orders.count }  # N+1!
```

CORRECT:
```ruby
users = User.includes(:orders)
users.each { |u| puts u.orders.size }
```

---

## Security Essentials

- **Never** store secrets in code -- use `Rails.application.credentials`
- **Always** use strong parameters in controllers
- **Never** use raw SQL without sanitization
- **Always** use CSRF protection (enabled by default)
- **Run Brakeman** regularly: `bundle exec brakeman`

---

## Callback Discipline

Avoid complex callback chains. Extract to services. Simple callbacks (defaults, normalization) are fine.

WRONG:
```ruby
class Order < ApplicationRecord
  after_create :send_confirmation
  after_create :update_inventory
  after_create :charge_payment
end
```

CORRECT: Use a service that orchestrates all steps explicitly.

---

## Background Jobs

Always use generators: `bin/rails generate job ProcessPayment`

Jobs should be thin, idempotent, and delegate to services:

```ruby
# frozen_string_literal: true

class ProcessPaymentJob < ApplicationJob
  queue_as :default

  def perform(order_id)
    order = Order.find(order_id)
    Payments::Charge.call(order:)
  end
end
```

---

## Commands

- `/ruby-rails:new` -- Set up a new Rails project with all conventions
- `/ruby-rails:audit` -- Audit an existing project for convention violations
- `/ruby-rails:service` -- Generate a new service with contract and tests

## Deep Dives

- `references/dry-rb.md` -- Complete dry-rb gem reference (types, validation, struct, initializer, configurable)
- `references/patterns.md` -- Advanced patterns, anti-patterns, and real-world examples
