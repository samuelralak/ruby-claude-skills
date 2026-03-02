# Advanced Patterns & Anti-Patterns

## Service Object Patterns

### Pattern: Contract -> Service -> Controller

The standard flow for any user-facing operation:

```
Controller -> Service.call(params)
                |-> Contract validates params (raises on failure)
                |-> Business logic executes (raises on failure)
                |-> Returns result value
           <- Render response (ErrorHandler catches exceptions)
```

### Pattern: Multi-Step Service

```ruby
# app/services/orders/place.rb
# frozen_string_literal: true

module Orders
  class Place < BaseService
    option :user_id,    type: Types::Strict::String
    option :product_id, type: Types::Strict::String
    option :quantity,   type: Types::Strict::Integer

    def call
      values  = validate!
      user    = find_user!(values[:user_id])
      product = find_product!(values[:product_id])
      order   = create_order!(user, product, values)
      send_confirmation(order)
      order
    end

    private

    def validate!
      result = Orders::PlaceContract.new.call(user_id:, product_id:, quantity:)
      raise ValidationError, result.errors.to_h unless result.success?

      result.to_h
    end

    def find_user!(id)
      User.find_by!(id:)
    end

    def find_product!(id)
      Product.find_by!(id:)
    end

    def create_order!(user, product, values)
      Order.create!(user:, product:, quantity: values[:quantity])
    end

    def send_confirmation(order)
      OrderMailer.confirmation(order).deliver_later
    rescue StandardError => e
      Rails.logger.error("Order confirmation email failed: #{e.message}")
    end
  end
end
```

### Pattern: Transactions

Use `ActiveRecord::Base.transaction` for operations that must succeed together. Bang methods (`update!`, `create!`) raise inside transactions, triggering rollback:

```ruby
# app/services/accounts/transfer.rb
# frozen_string_literal: true

module Accounts
  class Transfer < BaseService
    option :from_id, type: Types::Strict::String
    option :to_id,   type: Types::Strict::String
    option :amount,  type: Types::Strict::Decimal

    def call
      validate!

      ActiveRecord::Base.transaction do
        from = Account.lock.find_by!(id: from_id)
        to   = Account.lock.find_by!(id: to_id)
        raise InsufficientFundsError, from.id if from.balance < amount

        from.update!(balance: from.balance - amount)
        to.update!(balance: to.balance + amount)
        { from:, to:, amount: }
      end
    end

    private

    def validate!
      result = Accounts::TransferContract.new.call(from_id:, to_id:, amount:)
      raise ValidationError, result.errors.to_h unless result.success?
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
      charge = process_charge!
      record_payment!(charge)
      charge
    end

    private

    def process_charge!
      result = gateway.charge(amount:, customer_id: user.stripe_id)
      raise PaymentError, result.error unless result.success?

      result
    end

    def record_payment!(charge)
      Payment.create!(user:, amount:, external_id: charge.id)
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

    it "returns the charge" do
      expect(service.call).to respond_to(:id)
    end

    it "creates a payment record" do
      expect { service.call }.to change(Payment, :count).by(1)
    end
  end

  context "when charge fails" do
    before do
      allow(gateway).to receive(:charge)
        .and_return(OpenStruct.new(success?: false, error: "Card declined"))
    end

    it "raises PaymentError" do
      expect { service.call }.to raise_error(PaymentError, "Card declined")
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
      report = Report.find_by!(id: report_id)
      data   = collect_data(report)
      file   = generate_file(report, data)
      attach_file(report, file)
      notify_user(report)
      report.reload
    end

    private

    def collect_data(report)
      report.data_source.query(report.parameters)
    end

    def generate_file(report, data)
      ReportGenerator.new(report.format).generate(data)
    end

    def attach_file(report, file)
      report.file.attach(io: file, filename: "#{report.name}.pdf")
    end

    def notify_user(report)
      ReportMailer.ready(report).deliver_later
    end
  end
end
```

---

## Controller Patterns

Controllers rely on the `ErrorHandler` concern and `before_action` callbacks to stay thin. No pattern matching, no result handling -- just call and render.

### Full CRUD Controller

```ruby
# frozen_string_literal: true

module Api
  module V1
    class OrdersController < ApplicationController
      before_action :set_order, only: %i[show update destroy]

      def index
        orders = Orders::List.call(user_id: params[:user_id])
        render json: orders
      end

      def show
        render json: @order
      end

      def create
        order = Orders::Place.call(**order_params)
        render json: order, status: :created
      end

      def update
        order = Orders::Update.call(order: @order, **order_params)
        render json: order
      end

      def destroy
        Orders::Cancel.call(order: @order)
        head :no_content
      end

      private

      def set_order
        @order = Order.find(params[:id])
      end

      def order_params
        params.require(:order).permit(:product_id, :quantity)
          .to_h.symbolize_keys
      end
    end
  end
end
```

`set_order` uses `find` which raises `RecordNotFound` -- the `ErrorHandler` concern renders 404 automatically. No rescue needed in the action.

### Domain-Specific Errors

When a service raises domain-specific errors, add them to ErrorHandler or rescue locally:

```ruby
# frozen_string_literal: true

module Api
  module V1
    class PaymentsController < ApplicationController
      def create
        charge = Payments::Charge.call(**payment_params)
        render json: charge, status: :created
      rescue PaymentError => e
        render json: { error: e.message }, status: :payment_required
      end

      private

      def payment_params
        params.require(:payment).permit(:amount, :currency).to_h.symbolize_keys
      end
    end
  end
end
```

### Authentication / Authorization via before_action

```ruby
# frozen_string_literal: true

module Api
  module V1
    class BaseController < ApplicationController
      before_action :authenticate_user!

      private

      def authenticate_user!
        @current_user = User.find_by(token: request.headers["Authorization"]&.remove("Bearer "))
        render json: { error: "Unauthorized" }, status: :unauthorized unless @current_user
      end

      attr_reader :current_user
    end
  end
end
```

Resource-scoped controllers inherit and add their own callbacks:

```ruby
# frozen_string_literal: true

module Api
  module V1
    class ProjectsController < BaseController
      before_action :set_project, only: %i[show update destroy]
      before_action :authorize_project!, only: %i[update destroy]

      # ... actions: call service, render result

      private

      def set_project
        @project = Project.find(params[:id])
      end

      def authorize_project!
        render json: { error: "Forbidden" }, status: :forbidden unless @project.owner == current_user
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

CORRECT: Move to a service. Controller only calls the service and renders.

### Swallowing Exceptions Silently

WRONG:
```ruby
def find_user(id)
  User.find(id)
rescue ActiveRecord::RecordNotFound
  nil
end
```

CORRECT: Let exceptions propagate. ErrorHandler deals with them:
```ruby
def find_user!(id)
  User.find_by!(id:)
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
    it "creates user and returns it" do
      expect(result).to be_a(User)
      expect(result).to be_persisted
    end

    it "sends welcome email" do
      expect { result }.to have_enqueued_mail(UserMailer, :welcome)
    end
  end

  describe "failure" do
    context "with invalid params" do
      let(:params) { { name: "", email: "" } }

      it "raises ValidationError" do
        expect { result }.to raise_error(ValidationError)
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
