# claude-code-laravel-api

This repository contains a single Claude Code skill, `laravel-api`, that codifies REST API conventions for Laravel.

## What this repo is (and is not)

- **Is**: a distributable Claude Code skill. The content in `laravel-api/SKILL.md` must work for any Laravel project that wants to follow these conventions, not just one specific codebase.
- **Is not**: a book, a tutorial, a changelog, or a companion to a specific application. Do not include book references, chapter numbers, reading plans, or narrative text.

The skill originated from the book *Laravel REST APIs: A Practical Guide* and its companion project [BookShelf](https://github.com/gitantonio/bookshelf), but it is intended to evolve independently and must stand on its own.

## Reference implementation

[BookShelf](https://github.com/gitantonio/bookshelf) (usually checked out at `../bookshelf/` on the maintainer's machine) is the reference implementation of every pattern documented in this skill.

When adding, changing, or clarifying a convention:

1. **Verify it against the real code in `../bookshelf/`.** If the skill says "use `$request->user()->books()->create(...)`", a controller in BookShelf must actually do that. If BookShelf does something different, either update BookShelf or update the skill: the two must not diverge.
2. **Prefer snippets that compile.** Examples in `SKILL.md` should be copy-paste runnable in a Laravel project, not pseudocode.
3. **Do not copy BookShelf-specific details.** Avoid mentioning `Book`, `Author`, `Publisher` in prescriptive rules; use them only in illustrative code blocks. Rules ("NEVER include `user_id` in `$fillable`") should be domain-agnostic.

## Editing `SKILL.md`

The file has YAML frontmatter with `name` and `description`. Claude Code uses the `description` to decide when to activate the skill, so:

- Keep it specific enough that it only fires for Laravel API work, not any Laravel task.
- Mention the main triggers (controllers, form requests, resources, policies, migrations, factories, seeders, tests).
- Do not reference the book or BookShelf in the description: users who install this skill may have no idea what those are.

Content structure in the body:

- Lead with a checklist for a new resource (high-level, actionable).
- One section per artifact (Model, Migration, Factory, Resource, Controller, Form Request, Route, Policy, Error handling, Tests).
- Each section: short code example, then bullet-point rules.
- End with cross-cutting concerns (security checklist, filtering/sorting/pagination, Laravel version target).

Style:

- Write in English. The skill is public and the rest of the Claude Code ecosystem is English.
- Be prescriptive ("NEVER use `$request->all()`"), not descriptive ("you might want to consider...").
- Keep examples short. A section is a reference, not a tutorial.

## Do not

- Do not add narrative introductions, analogies, or "why this matters" essays. Rules only.
- Do not mention the book or BookShelf in `SKILL.md` itself (they belong in `README.md` for context).
- Do not add unrelated skills to this repo: keep it focused on Laravel REST API conventions. If a new skill is needed, create a separate repo.
- Do not break the single-folder layout (`laravel-api/SKILL.md`): users install via symlink into `.claude/skills/`, and that path is load-bearing.

## Laravel version policy

The skill currently targets **Laravel 13 and PHP 8.3+**. When Laravel releases a new major version:

1. Verify every code snippet in `SKILL.md` against the new version's conventions.
2. Update the "Laravel version" section at the bottom of `SKILL.md`.
3. Update the corresponding section in `../bookshelf/composer.json` and re-verify the reference implementation.

If a convention changes between versions (e.g. a different default behavior, a new helper, a deprecated pattern), document the Laravel 13+ behavior in the skill. Do not keep backward-compatibility shims.
