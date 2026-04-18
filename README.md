# claude-code-laravel-api

A [Claude Code](https://claude.com/claude-code) skill that teaches the AI the conventions for building REST APIs with Laravel, based on the book **"Laravel REST APIs: A Practical Guide"** and the companion project [BookShelf](https://github.com/gitantonio/bookshelf).

With this skill installed, Claude Code knows how to scaffold and modify controllers, form requests, API resources, policies, migrations, factories, seeders and Pest tests in a way that is consistent with the patterns described in the book.

## What is a Claude Code skill

A skill is a Markdown file (with YAML frontmatter) that Claude Code loads as additional context when the description matches what you're asking for. Skills are particularly useful to codify project-specific conventions that Claude Code cannot infer from a generic knowledge base.

This repository provides a single skill, `laravel-api`, whose description triggers when you ask Claude Code to create or modify API resources in a Laravel project.

## Installation

Clone the repository somewhere on your machine:

```bash
git clone https://github.com/gitantonio/claude-code-laravel-api.git
```

Then make the skill available to Claude Code. You have two options.

### Option 1: per project

Copy (or symlink) the `laravel-api/` folder into `.claude/skills/` of the project where you want to use it:

```bash
mkdir -p /path/to/your/laravel-project/.claude/skills
ln -s /path/to/claude-code-laravel-api/laravel-api \
      /path/to/your/laravel-project/.claude/skills/laravel-api
```

### Option 2: globally

Place the skill in your user-level Claude Code configuration so it's available in every project:

```bash
mkdir -p ~/.claude/skills
ln -s /path/to/claude-code-laravel-api/laravel-api \
      ~/.claude/skills/laravel-api
```

Restart Claude Code (or start a new session) and the skill will be listed among the available skills.

## Usage

The skill auto-activates based on its description, so you don't need to invoke it manually. Just ask Claude Code to work on your Laravel API:

```
> Add a Publisher resource with name, country and an optional website
```

Claude Code will pick up the skill and follow the conventions: explicit `$fillable`, Form Request with array syntax validation, API Resource with `whenLoaded()`, Policy, routes split between public reads and authenticated writes, Pest tests, and so on.

## What the skill covers

- Model, migration, factory, seeder conventions
- API Resource structure (`whenLoaded`, `when($request->routeIs(...))`, explicit fields, ISO 8601 dates)
- Controller patterns (authorization, `$request->validated()`, eager loading, status codes)
- Form Request validation (array syntax, `sometimes` vs `nullable`, `exists` for foreign keys)
- Route grouping by authentication
- Policies for ownership checks
- Custom error envelope and `BusinessException`
- Pest tests covering CRUD, validation, authentication and authorization
- Filtering, sorting, pagination and include conventions
- Laravel 13 / PHP 8.3+ specifics

## License

MIT. See [LICENSE](LICENSE).
