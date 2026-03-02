---
name: ruby-rails:audit
description: Audit a Rails project for convention violations (naming, structure, UUIDs, linting, dry-rb)
---

## Required Reading -- Do This First

Read `~/.claude/skills/ruby-rails/SKILL.md` completely.

## Process

When the user invokes `/ruby-rails:audit`, systematically check the project against all conventions.

### 1. Project Structure Audit

Check for the presence and correctness of:

- [ ] `.rubocop.yml` exists and includes rubocop-rails, rubocop-rspec, rubocop-performance
- [ ] `config/initializers/generators.rb` configures UUID primary keys
- [ ] `app/types.rb` exists with `Dry.Types()` inclusion
- [ ] `app/services/base_service.rb` exists with dry-initializer
- [ ] `app/contracts/application_contract.rb` exists
- [ ] `app/errors/` contains custom error classes (ServiceError, ValidationError, NotFoundError)
- [ ] `app/controllers/concerns/error_handler.rb` exists with rescue_from blocks
- [ ] `ApplicationController` includes `ErrorHandler`
- [ ] Gemfile includes required dry-rb gems (dry-validation, dry-types, dry-initializer, dry-struct)

### 2. Migration Audit

Scan `db/migrate/` for:
- [ ] All tables use `id: :uuid`
- [ ] All `t.references` use `type: :uuid`
- [ ] pgcrypto extension is enabled

### 3. Model Audit

For each model in `app/models/`:
- [ ] File is under 100 lines
- [ ] Follows correct order: extend -> include -> associations -> validations -> scopes
- [ ] Concerns use `-able` suffix and live in `concerns/{model_plural}/`
- [ ] Class methods live in `models/{model_plural}/` (not in concerns)
- [ ] Constants live in `models/{model_plural}/` (not inline in model)
- [ ] No business logic in models (should be in services)

### 4. Service Audit

For each service in `app/services/`:
- [ ] Extends BaseService or uses dry-initializer
- [ ] Uses typed options (Types::Strict::*)
- [ ] Returns values on success, raises on failure (no monads)
- [ ] Has a corresponding contract if it validates input
- [ ] Uses bang methods (`find_by!`, `create!`) to let exceptions propagate

### 5. Controller Audit

For each controller:
- [ ] No business logic (delegates to services)
- [ ] Actions are simple: call service, render result
- [ ] Uses `before_action` for resource loading
- [ ] Uses strong params
- [ ] No inline error handling (ErrorHandler catches exceptions)

### 6. Naming Audit

Check all files follow naming conventions:
- [ ] Concerns: `{Feature}able` suffix
- [ ] Services: `{Domain}::{Action}` pattern (e.g., `Users::Create`)
- [ ] Actions: in `actions/` subdirectory
- [ ] Performers: `{Noun}Performer` suffix in `performers/`

### 7. Code Quality

Run and report:
```bash
bundle exec rubocop
bundle exec rspec
```

### Output

Present findings as a checklist with pass/fail status:

```
Rails Convention Audit
======================

Structure:
  [PASS] .rubocop.yml configured
  [FAIL] Missing UUID generator config
  [PASS] Types module exists
  [PASS] ErrorHandler concern exists
  ...

Models (3 files):
  [PASS] User - 45 lines, correct order
  [FAIL] Order - 150 lines (exceeds 100 line limit)
  [WARN] Event - class methods in concern (should use extend)

Services (5 files):
  [PASS] Users::Create - typed options, raises on failure
  [FAIL] Orders::Process - untyped options
  ...

Migrations (8 files):
  [FAIL] 2 migrations missing id: :uuid
  [PASS] pgcrypto enabled

Summary: 12 pass, 4 fail, 1 warning
```

Then suggest specific fixes for each failure.
