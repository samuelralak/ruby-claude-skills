# Ruby on Rails Skill for Claude Code

A comprehensive Claude Code skill for Ruby on Rails development with dry-rb patterns, strict conventions, and generators-first workflow.

## Installation

Copy the skill and commands to your Claude Code configuration:

```bash
# Clone the repo
git clone git@github.com:samuelralak/ruby-claude-skills.git

# Symlink or copy the skill
ln -s "$(pwd)/ruby-claude-skills" ~/.claude/skills/ruby-rails

# Copy commands to Claude Code commands directory
cp ruby-claude-skills/commands/*.md ~/.claude/commands/
```

## What's Included

### Skill (`SKILL.md`)

Core conventions enforced:

- **Generators first** -- never manually create migrations, models, controllers, etc.
- **UUID primary/foreign keys** everywhere (PostgreSQL pgcrypto)
- **RuboCop + Brakeman** mandatory on every project
- **dry-rb service stack** -- dry-monads (Do notation), dry-initializer (typed options), dry-validation (contracts), dry-types
- **Thin models (<100 lines)** with concerns (`-able` suffix) and class method modules
- **Thin controllers** that delegate to services and pattern match results
- **Clean, simple, idiomatic Ruby** -- guard clauses, safe navigation, frozen string literals

### Commands

| Command | Description |
|---------|-------------|
| `/ruby-rails:new` | Bootstrap a new Rails project with all conventions configured |
| `/ruby-rails:audit` | Audit an existing project for convention violations |
| `/ruby-rails:service` | Generate a service + contract + tests |

### References

| File | Description |
|------|-------------|
| `references/dry-rb.md` | Complete API reference for dry-types, dry-struct, dry-validation, dry-monads, dry-initializer, dry-configurable, dry-auto_inject, dry-system |
| `references/patterns.md` | Advanced service patterns, controller patterns, model patterns, anti-patterns, and testing patterns |

## Key Patterns

### Service Objects

```ruby
module Users
  class Create < BaseService
    option :name,  type: Types::Strict::String
    option :email, type: Types::Strict::String

    def call
      values = yield validate
      user   = yield persist(values)
      Success(user)
    end
  end
end
```

### Failure Convention (hash-style)

```ruby
Failure(validation: errors)
Failure(not_found: "User not found")
Failure(persistence: user.errors.full_messages)
```

### Thin Controllers with Concerns

```ruby
# Controllers use handle_service (from ServiceHandler concern)
def create
  handle_service Users::Create.call(**user_params),
                 success_status: :created,
                 serializer: UserSerializer
end
```

## License

MIT
