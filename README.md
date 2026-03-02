# Ruby on Rails Skill for Claude Code

A comprehensive Claude Code skill for Ruby on Rails development with dry-rb typing/validation, strict conventions, and generators-first workflow.

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
- **dry-rb for typing and validation** -- dry-types, dry-initializer (typed options), dry-validation (contracts)
- **Plain Ruby services** -- return values on success, raise on failure. No monads.
- **Thin models (<100 lines)** with concerns (`-able` suffix) and class method modules
- **Thin controllers** -- call service, render result. ErrorHandler catches exceptions.
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
| `references/dry-rb.md` | Complete API reference for dry-types, dry-struct, dry-validation, dry-initializer, dry-configurable |
| `references/patterns.md` | Advanced service patterns, controller patterns, model patterns, anti-patterns, and testing patterns |

## Key Patterns

### Service Objects

```ruby
module Users
  class Create < BaseService
    option :name,  type: Types::Strict::String
    option :email, type: Types::Strict::String

    def call
      values = validate!
      User.create!(values)
    end
  end
end
```

### Thin Controllers

```ruby
def create
  user = Users::Create.call(**user_params)
  render json: user, status: :created
end
```

### ErrorHandler Catches Exceptions

```ruby
# ValidationError  -> 422
# RecordNotFound   -> 404
# RecordInvalid    -> 422
# ParameterMissing -> 400
```

## License

MIT
