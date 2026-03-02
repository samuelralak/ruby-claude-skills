# Advanced Patterns & Anti-Patterns

## Service Object Patterns

### Pattern: Contract -> Service -> Controller

The standard flow for any user-facing operation:

```
Controller -> Service.call(params)
                |-> Contract validates params
                |-> Business logic executes
                |-> Returns Success/Failure (hash-style)
           <- Pattern match result, render response
```

### Pattern: Chained Services with Do Notation

```ruby
# app/services/orders/place.rb
# frozen_string_literal: true

module Orders
  class Place < BaseService
    option :user_id,    type: Types::Strict::String
    option :product_id, type: Types::Strict::String
    option :quantity,   type: Types::Strict::Integer

    def call
      values  = yield validate
      user    = yield find_user(values[:user_id])
      product = yield find_product(values[:product_id])
      order   = yield create_order(user, product, values)
      yield send_confirmation(order)

      Success(order)
    end

    private

    def validate
      result = Orders::PlaceContract.new.call(user_id:, product_id:, quantity:)
      result.success? ? Success(result.to_h) : Failure(validation: result.errors.to_h)
    end

    def find_user(id)
      user = User.find_by(id:)
      user ? Success(user) : Failure(not_found: "User not found")
    end

    def find_product(id)
      product = Product.find_by(id:)
      product ? Success(product) : Failure(not_found: "Product not found")
    end

    def create_order(user, product, values)
      order = Order.create(user:, product:, quantity: values[:quantity])
      order.persisted? ? Success(order) : Failure(persistence: order.errors.full_messages)
    end

    def send_confirmation(order)
      OrderMailer.confirmation(order).deliver_later
      Success(order)
    rescue StandardError => e
      Rails.logger.error("Order confirmation email failed: #{e.message}")
      Success(order)
    end
  end
end
```

### Pattern: Transactions with Do Notation

Do notation raises `Dry::Monads::Do::Halt` on Failure, which triggers transaction rollback:

```ruby
# app/services/accounts/transfer.rb
# frozen_string_literal: true

module Accounts
  class Transfer < BaseService
    option :from_id, type: Types::Strict::String
    option :to_id,   type: Types::Strict::String
    option :amount,  type: Types::Strict::Decimal

    def call
      ActiveRecord::Base.transaction do
        from = yield find_account(from_id)
        to   = yield find_account(to_id)
        yield check_balance(from, amount)
        yield debit(from, amount)
        yield credit(to, amount)

        Success({ from:, to:, amount: })
      end
    end

    private

    def find_account(id)
      account = Account.lock.find_by(id:)
      account ? Success(account) : Failure(not_found: "Account #{id} not found")
    end

    def check_balance(account, amount)
      account.balance >= amount ? Success() : Failure(insufficient_funds: account.id)
    end

    def debit(account, amount)
      account.update!(balance: account.balance - amount)
      Success(account)
    end

    def credit(account, amount)
      account.update!(balance: account.balance + amount)
      Success(account)
    end
  end
end
```

### Pattern: Service with Injected Dependencies (for Testing)

```ruby
# app/services/payments/charge.rb
# frozen_string_literal: true

module Payments
  class Charge < BaseService
    option :amount,  type: Types::Strict::Decimal
    option :user,    type: Types.Instance(User)
    option :gateway, default: proc { StripeGateway.new }

    def call
      charge = yield process_charge
      yield update_records(charge)
      Success(charge)
    end

    private

    def process_charge
      result = gateway.charge(amount:, customer_id: user.stripe_id)
      result.success? ? Success(result) : Failure(payment: result.error)
    end

    def update_records(charge)
      Payment.create!(user:, amount:, external_id: charge.id)
      Success()
    rescue ActiveRecord::RecordInvalid => e
      Failure(persistence: e.message)
    end
  end
end
```

```ruby
# spec/services/payments/charge_spec.rb
# frozen_string_literal: true

RSpec.describe Payments::Charge do
  let(:gateway) { instance_double(StripeGateway) }
  let(:user) { create(:user, stripe_id: "cus_123") }
  let(:service) { described_class.new(amount: BigDecimal("10.0"), user:, gateway:) }

  context "when charge succeeds" do
    before do
      allow(gateway).to receive(:charge)
        .and_return(OpenStruct.new(success?: true, id: "ch_1"))
    end

    it "returns Success" do
      expect(service.call).to be_success
    end
  end

  context "when charge fails" do
    before do
      allow(gateway).to receive(:charge)
        .and_return(OpenStruct.new(success?: false, error: "Card declined"))
    end

    it "returns Failure with payment error" do
      result = service.call
      expect(result).to be_failure
      expect(result.failure).to have_key(:payment)
    end
  end
end
```

### Pattern: Performer for Complex Background Work

```ruby
# app/services/reports/generation_performer.rb
# frozen_string_literal: true

module Reports
  class GenerationPerformer < BaseService
    option :report_id, type: Types::Strict::String

    def call
      report = yield find_report
      data   = yield collect_data(report)
      file   = yield generate_file(report, data)
      yield attach_file(report, file)
      yield notify_user(report)

      Success(report.reload)
    end

    private

    def find_report
      report = Report.find_by(id: report_id)
      report ? Success(report) : Failure(not_found: "Report not found")
    end

    def collect_data(report)
      data = report.data_source.query(report.parameters)
      Success(data)
    rescue StandardError => e
      Failure(data_error: e.message)
    end

    def generate_file(report, data)
      file = ReportGenerator.new(report.format).generate(data)
      Success(file)
    end

    def attach_file(report, file)
      report.file.attach(io: file, filename: "#{report.name}.pdf")
      Success()
    end

    def notify_user(report)
      ReportMailer.ready(report).deliver_later
      Success()
    end
  end
end
```

---

## Controller Patterns

Controllers use the `ServiceHandler` concern (see SKILL.md) so actions stay clean.

### Standard API Controller

```ruby
# frozen_string_literal: true

module Api
  module V1
    class OrdersController < ApplicationController
      def create
        handle_service Orders::Place.call(**order_params),
                       success_status: :created,
                       serializer: OrderSerializer
      end

      def show
        handle_service Orders::Find.call(id: params[:id]),
                       serializer: OrderSerializer
      end

      private

      def order_params
        params.require(:order).permit(:user_id, :product_id, :quantity)
          .to_h.symbolize_keys
      end
    end
  end
end
```

### Custom Handling for Non-Standard Failures

When a service returns domain-specific failures beyond the standard set, override locally:

```ruby
# frozen_string_literal: true

module Api
  module V1
    class PaymentsController < ApplicationController
      def create
        result = Payments::Charge.call(**payment_params)

        case result
        in Failure(payment: message)
          render json: { error: message }, status: :payment_required
        in Failure(rate_limited: message)
          render json: { error: message }, status: :too_many_requests
        else
          handle_service result, success_status: :created
        end
      end

      private

      def payment_params
        params.require(:payment).permit(:amount, :currency).to_h.symbolize_keys
      end
    end
  end
end
```

---

## Model Patterns

### Concern Example (Instance Methods)

```ruby
# app/models/concerns/events/filterable.rb
# frozen_string_literal: true

module Events
  module Filterable
    extend ActiveSupport::Concern

    def matches_filter?(filter)
      filter.all? { |key, value| send(key) == value }
    end

    def visible_to?(user)
      public? || user_id == user.id
    end
  end
end
```

### Module Example (Class Methods)

```ruby
# app/models/events/lookup.rb
# frozen_string_literal: true

module Events
  module Lookup
    def find_by_pubkey(pubkey)
      where(pubkey:).first
    end

    def recent(limit = 10)
      order(created_at: :desc).limit(limit)
    end
  end
end
```

### Constants Module

```ruby
# app/models/events/kinds.rb
# frozen_string_literal: true

module Events
  module Kinds
    METADATA  = 0
    TEXT_NOTE = 1
    REACTION  = 7
    REPOST    = 6

    ALL = [METADATA, TEXT_NOTE, REACTION, REPOST].freeze
  end
end
```

### Thin Model Tying It Together

```ruby
# app/models/event.rb
# frozen_string_literal: true

class Event < ApplicationRecord
  extend Events::Lookup
  extend Events::Kinds
  include Events::Filterable

  belongs_to :user
  has_many :tags, dependent: :destroy

  validates :content, presence: true
  validates :kind, inclusion: { in: Events::Kinds::ALL }

  scope :active, -> { where(deleted: false) }
  scope :by_kind, ->(kind) { where(kind:) }
end
```

---

## Anti-Patterns to Avoid

### Fat Models

WRONG:
```ruby
class User < ApplicationRecord
  # 300+ lines of methods, callbacks, validations...
  def send_welcome_email; end
  def calculate_subscription; end
  def sync_with_stripe; end
  def generate_report; end
end
```

CORRECT: Extract to services and concerns.

### Business Logic in Controllers

WRONG:
```ruby
def create
  user = User.new(user_params)
  if user.save
    UserMailer.welcome(user).deliver_later
    SubscriptionService.create(user)
    render json: user
  else
    render json: { errors: user.errors }, status: :unprocessable_entity
  end
end
```

CORRECT: Move to a service. Controller only handles HTTP concerns.

### Exceptions for Flow Control

WRONG:
```ruby
def find_user(id)
  User.find(id)
rescue ActiveRecord::RecordNotFound
  nil
end
```

CORRECT: Use Result monads:
```ruby
def find_user(id)
  user = User.find_by(id:)
  user ? Success(user) : Failure(not_found: "User #{id} not found")
end
```

### Manual File Creation (Instead of Generators)

WRONG: Creating `db/migrate/20250101000000_create_users.rb` by hand.

CORRECT:
```bash
bin/rails generate migration CreateUsers name:string email:string
```

### Non-UUID Primary Keys

WRONG:
```ruby
create_table :users do |t|  # uses integer id
  t.string :name
end
```

CORRECT:
```ruby
create_table :users, id: :uuid do |t|
  t.string :name
end
```

### Untyped Service Options

WRONG:
```ruby
class CreateUser < BaseService
  option :name
  option :email
end
```

CORRECT:
```ruby
module Users
  class Create < BaseService
    option :name,  type: Types::Strict::String
    option :email, type: Types::Strict::String
  end
end
```

### Missing `dependent:` on Associations

WRONG:
```ruby
has_many :orders
has_many :comments
```

CORRECT:
```ruby
has_many :orders, dependent: :destroy
has_many :comments, dependent: :destroy
```

---

## Testing Patterns

### Service Test Structure

```ruby
# frozen_string_literal: true

RSpec.describe Users::Create do
  subject(:result) { described_class.call(**params) }

  let(:params) { { name: "Jane", email: "jane@example.com" } }

  describe "success" do
    it "creates user and returns Success" do
      expect(result).to be_success
      expect(result.value!).to be_a(User)
    end

    it "sends welcome email" do
      expect { result }.to have_enqueued_mail(UserMailer, :welcome)
    end
  end

  describe "failure" do
    context "with invalid params" do
      let(:params) { { name: "", email: "" } }

      it "returns Failure with validation errors" do
        expect(result).to be_failure
        expect(result.failure).to have_key(:validation)
      end
    end
  end
end
```

### Contract Test Structure

```ruby
# frozen_string_literal: true

RSpec.describe Users::CreateContract do
  subject(:result) { described_class.new.call(params) }

  describe "valid params" do
    let(:params) { { name: "Jane", email: "jane@example.com", age: 25 } }

    it { is_expected.to be_success }
  end

  describe "validations" do
    context "when name is missing" do
      let(:params) { { email: "jane@example.com", age: 25 } }

      it "returns name error" do
        expect(result.errors.to_h).to include(name: ["is missing"])
      end
    end

    context "when age is under 18" do
      let(:params) { { name: "Jane", email: "jane@example.com", age: 16 } }

      it "returns age error" do
        expect(result.errors.to_h).to include(age: ["must be greater than 17"])
      end
    end
  end
end
```

### Model Test Structure

```ruby
# frozen_string_literal: true

RSpec.describe User do
  describe "associations" do
    it { is_expected.to have_many(:orders).dependent(:destroy) }
    it { is_expected.to belong_to(:team).optional }
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:name) }
    it { is_expected.to validate_presence_of(:email) }
  end

  describe "scopes" do
    describe ".active" do
      it "returns only active users" do
        active = create(:user, active: true)
        create(:user, active: false)

        expect(described_class.active).to eq([active])
      end
    end
  end
end
```
