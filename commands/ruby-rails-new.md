---
name: ruby-rails:new
description: Set up a new Rails project with all conventions (UUIDs, RuboCop, dry-rb, base service)
---

## Required Reading -- Do This First

Read `~/.claude/skills/ruby-rails/SKILL.md` completely.

## Process

When the user invokes `/ruby-rails:new`, follow these steps:

### 1. Ask for Project Details

Ask the user:
- Project name
- Ruby version (default: latest stable)
- Rails version (default: latest stable)
- Database (default: PostgreSQL)
- API-only? (default: no)
- Any additional gems they want

### 2. Generate the Rails App

```bash
rails new {project_name} --database=postgresql {--api if api-only} --skip-test
```

Use `--skip-test` because we use RSpec.

### 3. Set Up UUID Primary Keys

```bash
cd {project_name}
bin/rails generate migration EnablePgcrypto
```

Edit the migration to add `enable_extension "pgcrypto"`.

Create `config/initializers/generators.rb` with UUID config from SKILL.md.

### 4. Set Up RuboCop + Brakeman

Add to Gemfile (development/test group):
```ruby
group :development, :test do
  gem "rubocop", require: false
  gem "rubocop-rails", require: false
  gem "rubocop-rspec", require: false
  gem "rubocop-performance", require: false
  gem "brakeman", require: false
end
```

Create `.rubocop.yml` with the config from SKILL.md.

Run `bundle install && bundle exec rubocop --autocorrect`.

### 5. Set Up RSpec

Add to Gemfile:
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

Run:
```bash
bundle install
bin/rails generate rspec:install
```

Configure `spec/rails_helper.rb` with FactoryBot and Shoulda Matchers.

### 6. Set Up dry-rb Gems

Add to Gemfile:
```ruby
gem "dry-monads",      "~> 1.6"
gem "dry-validation",  "~> 1.10"
gem "dry-types",       "~> 1.7"
gem "dry-initializer", "~> 3.1"
gem "dry-struct",      "~> 1.6"
```

Run `bundle install`.

Create the foundation files from SKILL.md:
- `app/types.rb`
- `app/services/base_service.rb`
- `app/contracts/application_contract.rb`
- `config/initializers/dry.rb`

### 7. Final Steps

- Run `bin/rails db:create`
- Run `bundle exec rubocop --autocorrect`
- Run `bundle exec rspec` to verify setup
- Run `bundle exec brakeman` to verify security baseline
- Report the setup summary to the user

### Output

Present a summary:
```
Project: {name}
Ruby: {version} | Rails: {version} | DB: PostgreSQL

Configured:
  - UUID primary keys (pgcrypto)
  - RuboCop (rubocop-rails, rubocop-rspec, rubocop-performance)
  - Brakeman security scanner
  - RSpec + FactoryBot + Shoulda + Faker
  - dry-monads + dry-validation + dry-types + dry-initializer + dry-struct
  - BaseService with dry-monads Do notation
  - ApplicationContract base class
  - Types module

Ready to build.
```
