# prism

A Claude Code plugin that adds multi-profile code review, conventional commits, branching, and PR workflows to any repo. It auto-detects your stack and runs security, quality, performance, documentation, and testing reviewers in parallel. Works both locally via slash commands and as an automated CI reviewer on pull requests.

## Setup

### 1. Install the plugin

```bash
/plugin install SergiCoder/prism
```

### 2. Add automated PR reviews (optional)

```bash
/prism:install-ci-review
```

Then add your Anthropic API key as a repository secret:
- GitHub repo → Settings → Secrets and variables → Actions → New repository secret
- Name: `ANTHROPIC_API_KEY`

The workflow runs automatically on PRs opened by the repo owner, `@prism review` comments, and manual dispatch.

## Examples

```bash
# create a branch with the right prefix and base
/prism:create-branch feature user-avatars

# lint, format, conventional commit, and push in one step
/prism:ship

# report-only review (auto-detects your stack)
/prism:review-and-report-only

# review and auto-fix all findings
/prism:review-and-fix

# review and post inline comments on the PR
/prism:review-pr

# sync with base branch, run tests, and open a pull request
/prism:open-pr

# read inline PR comments, fix the code, reply, and resolve
/prism:address-pr-review

# open a release PR from dev into main
/prism:release
```

## Commands

| Command | Description |
|---|---|
| `/prism:install-ci-review` | Install the automated PR review workflow into the current project |
| `/prism:create-branch <type> <name>` | Create a `feature/`, `fix/`, or `hotfix/` branch from the correct base |
| `/prism:ship` | Pre-ship hygiene check, auto-format, conventional commits, push |
| `/prism:review-and-report-only [profiles]` | Run multi-profile code review — report only, no changes |
| `/prism:review-and-fix [profiles]` | Run multi-profile code review and auto-fix all findings |
| `/prism:review-pr [profiles]` | Review and post inline comments + summary on the PR |
| `/prism:open-pr` | Sync base branch, run tests, open a PR |
| `/prism:address-pr-review` | Address inline PR review comments — validate, fix, reply, and resolve |
| `/prism:release` | Open a release PR from `dev` into `main` (for repos using a dev branch) |

## Code Review

All review commands auto-detect the stack from config files and run only the relevant reviewers. The same profiles run locally and in CI.

```bash
/prism:review-and-report-only                    # report findings locally
/prism:review-and-report-only security,performance  # specific profiles only
/prism:review-and-fix                            # review + auto-fix all severities
/prism:review-pr                                 # post inline comments on the PR
```

### Core Profiles

| Profile | What it checks |
|---|---|
| `security` | Auth, injection, secrets, headers, rate limiting, data exposure |
| `quality` | Dead code, duplication, stdlib/dependency reinvention, verbose patterns, size thresholds |
| `performance` | N+1 queries, indexes, async correctness, caching, frontend rendering |
| `rest-api` | HTTP status codes, Location headers, error envelopes, resource representation, URL design |
| `documentation` | Fixes docs directly — CLAUDE.md, README, docstrings, inline comments |
| `testing` | Writes missing tests directly — happy path, error cases, edge cases |

### Stack Profiles (auto-detected)

| Profile | Language / Framework |
|---|---|
| `stack-python` | Python idioms, type safety, error handling |
| `stack-typescript` | TypeScript/JS idioms, type safety, error handling |
| `stack-go` | Go idioms, concurrency, type safety |
| `stack-django` | Models, queries, views, settings, migrations |
| `stack-flask` | Structure, blueprints, error handling |
| `stack-fastapi` | Routes, dependencies, Pydantic v2 |
| `stack-react` | Hooks, state, memoization, accessibility |
| `stack-vue` | Composition API, reactivity, Pinia |
| `stack-nextjs` | App Router, server/client components, performance |
| `stack-nuxt` | Composables, SSR correctness |
| `stack-sveltekit` | Svelte 5 runes, data loading, form actions |
| `stack-express` | Middleware, async error handling, input validation |
| `stack-fastify` | Plugin encapsulation, schema validation, hooks, reply semantics |
| `stack-spring` | Layers, DI, JPA, error handling, configuration |
| `stack-laravel` | Eloquent, controllers, validation, migrations |
| `stack-aspnet` | Controllers, DI lifetimes, EF Core, configuration |
| `stack-rails` | MVC, ActiveRecord, security, migrations |
| `stack-go-http` | Handlers, middleware, error responses |

Stack detection is automatic from: `package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `pom.xml`, `build.gradle`, `composer.json`, `Gemfile`, `*.csproj`.

## Stack Support

Commands auto-detect the stack from project config files (no configuration needed):
- **Test runners**: pytest, go test, npm/pnpm/yarn test, make test
- **Linters/formatters**: Ruff, ESLint, Biome, golangci-lint, and others
- **Base branch**: `main`

## Plugin Structure

```
prism/
├── .claude-plugin/
│   └── plugin.json              # plugin manifest
├── .github/workflows/
│   └── claude-review.yml        # prism's own CI review (self-review)
├── install/
│   └── claude-review.yml        # installable template for other repos
├── commands/                    # user-invoked slash commands
│   ├── create-branch.md
│   ├── ship.md
│   ├── review-and-report-only.md
│   ├── review-and-fix.md
│   ├── review-pr.md
│   ├── open-pr.md
│   ├── address-pr-review.md
│   ├── release.md
│   └── install-ci-review.md
└── skills/                      # review profiles (used locally and in CI)
    ├── security/SKILL.md
    ├── quality/SKILL.md
    ├── performance/SKILL.md
    ├── rest-api/SKILL.md
    ├── documentation/SKILL.md
    ├── testing/SKILL.md
    ├── stack-python/SKILL.md
    ├── stack-typescript/SKILL.md
    ├── stack-go/SKILL.md
    ├── stack-django/SKILL.md
    ├── stack-flask/SKILL.md
    ├── stack-fastapi/SKILL.md
    ├── stack-react/SKILL.md
    ├── stack-vue/SKILL.md
    ├── stack-nextjs/SKILL.md
    ├── stack-nuxt/SKILL.md
    ├── stack-sveltekit/SKILL.md
    ├── stack-express/SKILL.md
    ├── stack-fastify/SKILL.md
    ├── stack-spring/SKILL.md
    ├── stack-laravel/SKILL.md
    ├── stack-aspnet/SKILL.md
    ├── stack-rails/SKILL.md
    └── stack-go-http/SKILL.md
```

## Contributing

Fork the repo and open a PR against `main`.
