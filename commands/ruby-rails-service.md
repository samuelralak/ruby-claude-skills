---
name: ruby-rails:service
description: Generate a new service with contract, types, and tests following dry-rb conventions
---

## Required Reading -- Do This First

Read `~/.claude/skills/ruby-rails/SKILL.md` and `~/.claude/skills/ruby-rails/references/patterns.md`.

## Process

When the user invokes `/ruby-rails:service`, ask for:

1. **Domain** -- The namespace (e.g., `users`, `orders`, `payments`)
2. **Action** -- What the service does (e.g., `create`, `update`, `charge`)
3. **Inputs** -- What parameters the service accepts (name + type)
4. **Steps** -- Brief description of what the service does

### Determine Service Tier

Based on user description:
- Called from controller/job -> Main service in `app/services/{domain}/`
- Called only by other services -> Action in `app/services/{domain}/actions/`
- Complex background worker -> Performer in `app/services/{domain}/performers/`

### Generate Files

Based on user input, generate these files. All files MUST include `frozen_string_literal: true`.

#### 1. Contract

```ruby
# app/contracts/{domain}/{action}_contract.rb
# frozen_string_literal: true

module {Domain}
  class {Action}Contract < ApplicationContract
    params do
      required(:field).filled(:type)
    end

    # Add rules for cross-field or business validation
  end
end
```

#### 2. Service

```ruby
# app/services/{domain}/{action}.rb
# frozen_string_literal: true

module {Domain}
  class {Action} < BaseService
    option :field, type: Types::Strict::{Type}

    def call
      values = yield validate
      # ... steps from user description
      Success(result)
    end

    private

    def validate
      result = {Domain}::{Action}Contract.new.call(field:)
      result.success? ? Success(result.to_h) : Failure(validation: result.errors.to_h)
    end
  end
end
```

#### 3. Service Spec

```ruby
# spec/services/{domain}/{action}_spec.rb
# frozen_string_literal: true

RSpec.describe {Domain}::{Action} do
  subject(:result) { described_class.call(**params) }

  let(:params) { { field: value } }

  context "with valid params" do
    it "returns Success" do
      expect(result).to be_success
    end
  end

  context "with invalid params" do
    let(:params) { { field: invalid_value } }

    it "returns Failure with validation errors" do
      expect(result).to be_failure
      expect(result.failure).to have_key(:validation)
    end
  end
end
```

#### 4. Contract Spec

```ruby
# spec/contracts/{domain}/{action}_contract_spec.rb
# frozen_string_literal: true

RSpec.describe {Domain}::{Action}Contract do
  subject(:result) { described_class.new.call(params) }

  context "with valid params" do
    let(:params) { { field: value } }

    it { is_expected.to be_success }
  end

  context "when field is missing" do
    let(:params) { {} }

    it "returns error" do
      expect(result.errors.to_h).to include(field: ["is missing"])
    end
  end
end
```

### Output

After generating all files, present:

```
Generated:
  app/contracts/{domain}/{action}_contract.rb
  app/services/{domain}/{action}.rb
  spec/contracts/{domain}/{action}_contract_spec.rb
  spec/services/{domain}/{action}_spec.rb

Run: bundle exec rspec spec/services/{domain}/{action}_spec.rb
```

Then run RuboCop on the generated files:
```bash
bundle exec rubocop --autocorrect app/contracts/{domain}/ app/services/{domain}/ spec/
```
