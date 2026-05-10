# Stack detection & profile selection (shared)

Shared logic used by `review-and-fix`, `review-pr`, and `review-and-report-only` to detect the project stack from the diff and pick which review profiles to run.

Inputs the calling command must already have:
- `<base>` — the base branch
- The list of changed files from `git diff origin/<base>...HEAD --name-only`

Outputs this section produces:
- A list of detected languages and frameworks (used in the final report's "Stack detected" line)
- The deduplicated list of profiles to run

## Step A — Detect stack

Read the following files if they exist to detect languages and frameworks in use:
- `package.json` — for JS/TS framework detection
- `pyproject.toml` or `requirements.txt` — for Python framework detection
- `go.mod` — for Go module and framework detection
- `pom.xml` or `build.gradle` — for Java/Spring Boot detection
- `composer.json` — for PHP/Laravel detection
- `*.csproj` — for C#/ASP.NET Core detection
- `Gemfile` — for Ruby/Rails detection

**Language detection** (from changed file extensions):

| Changed files include | Languages detected |
|---|---|
| `*.py` | python |
| `*.ts`, `*.tsx`, `*.js`, `*.jsx`, `*.vue`, `*.svelte` | typescript |
| `*.go` | go |
| `*.java` | java |
| `*.php` | php |
| `*.cs` | csharp |
| `*.rb` | ruby |

**Framework detection** (from config files):

| Signal | Framework skill |
|---|---|
| `package.json` deps contain `"next"` | stack-nextjs |
| `package.json` deps contain `"nuxt"` | stack-nuxt |
| `package.json` deps contain `"@sveltejs/kit"` | stack-sveltekit |
| `package.json` deps contain `"react"` but NOT next/nuxt/sveltekit | stack-react |
| `package.json` deps contain `"vue"` but NOT nuxt | stack-vue |
| `package.json` deps contain `"express"` | stack-express |
| `package.json` deps contain `"fastify"` | stack-fastify |
| `pyproject.toml` or `requirements.txt` contains `django` | stack-django |
| `pyproject.toml` or `requirements.txt` contains `flask` | stack-flask |
| `pyproject.toml` or `requirements.txt` contains `fastapi` | stack-fastapi |
| `go.mod` contains `chi`, `gin`, `echo`, or changed files import `net/http` | stack-go-http |
| `pom.xml` or `build.gradle` contains `spring-boot` | stack-spring |
| `composer.json` contains `laravel/framework` | stack-laravel |
| Any `*.csproj` file exists in the repo | stack-aspnet |
| `Gemfile` contains `rails` | stack-rails |

Build the list of stack skills to run: one per detected language + one per detected framework. A framework skill covers framework-specific rules but does NOT replace the base language skill (e.g., `stack-django` + `stack-python` both run).

## Step B — Select profiles

**If profiles were specified in `$ARGUMENTS`:** run only those (skip auto-detection).

**If no profiles specified**, select profiles based on changed files and detected stack:

| Condition | Profiles to run |
|---|---|
| Any source code changed | quality + security + performance + testing + all detected stack skills |
| Any backend framework detected (django, flask, fastapi, express, fastify, spring, laravel, rails, aspnet, go-http) | rest-api |
| Any `*.py` | stack-python + any detected Python framework skill |
| Any `*.ts`, `*.tsx`, `*.js`, `*.jsx` | stack-typescript + any detected JS framework skill |
| Any `*.go` | stack-go + stack-go-http (if detected) |
| Any `*.java` | stack-spring (if detected) |
| Any `*.php` | stack-laravel (if detected) |
| Any `*.cs` | stack-aspnet |
| Any `*.rb` | stack-rails (if detected) |
| Any `*.vue` | stack-vue + stack-typescript |
| Any `*.svelte` | stack-typescript |
| `Dockerfile*`, `docker-compose*`, `infra/**`, `.github/workflows/**` | security only |
| `.env*` (committed only) | security only |
| `package.json`, `pyproject.toml`, `requirements*.txt`, `go.mod`, `Gemfile`, `composer.json`, `*.csproj`, `pom.xml`, `build.gradle`, `build.gradle.kts` changed | dependencies |

Deduplicate. Always include `quality` if any source code changed. Always include `documentation` if any source code changed.

## Profile ownership boundaries

Each profile ONLY reports findings within its domain. If a finding could belong to multiple profiles, it belongs to the FIRST matching profile in this list:

1. **quality**: DRY violations, dead code, complexity, stdlib reinvention, over-abstraction, size thresholds
2. **stack-***: Language idioms, type safety, error handling conventions, framework-specific patterns
3. **rest-api**: HTTP status codes, Location headers, error envelopes, resource representation, URL design, method semantics
4. **security**: Authentication, authorization, injection, secrets, headers, rate limiting
5. **performance**: Query patterns, caching, async correctness, indexes, frontend rendering
6. **documentation**: Doc accuracy, staleness, completeness
7. **testing**: Test coverage gaps
8. **dependencies**: Outdated packages, frameworks, and runtimes
