# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.13.0] - 2026-05-10

### Added
- `stack-fastify` skill — reviews Fastify/Node.js code for plugin encapsulation, schema-driven validation/serialization, lifecycle hooks, reply semantics (no double-send, no `await reply.send`), `setErrorHandler`, `@fastify/*` security plugins (helmet, cors, rate-limit, bodyLimit), pino logging with redaction, and `fastify.inject()` testing patterns. Auto-triggered when `package.json` deps contain `"fastify"`.
- Shared `commands/_detect-stack.md` defines stack detection (config files, language table, framework table) and profile selection (routing table, ownership boundaries) once, referenced from `review-and-fix`, `review-pr`, and `review-and-report-only`. Adding a new framework now requires a single-file edit instead of three.

### Fixed
- `fastify` is now in the backend-framework trigger list for the `rest-api` profile across all three review commands — previously missed when the framework was added.

## [1.12.0] - 2026-05-01

### Changed
- `ship` no longer inlines full `git diff --cached` and `git diff` output in its prompt — only `--stat` summaries are loaded up-front; the full diff is fetched on demand via `git diff <path>` when a commit message needs it. Substantially reduces token cost on non-trivial branches.
- `ship` reordered to fail fast: verify (typecheck/test/lint) now runs before the hygiene scan and commit-message generation, so a broken working tree stops the command before any speculative file reads or commit drafting.
- `ship` hygiene scan narrowed to 0-byte files and obvious scaffolding leftovers; broad orphan/duplicate searches across the tree have been removed.
- All commands now pin a `model:` for predictable cost and quality: `open-pr` runs on Opus 4.7; `address-pr-review`, `create-branch`, `install-ci-review`, `release`, `review-and-fix`, `review-and-report-only`, `review-pr`, and `ship` run on Sonnet 4.6.

## [1.11.0] - 2026-04-15

### Added
- Shared `_verify.md` defines the post-action verification checks (typecheck, test, lint) and is referenced from `ship`, `open-pr`, `release`, `address-pr-review`, and `review-and-fix` so every post-action flow confirms nothing broke before declaring success
- Confirmation gates before high-blast-radius actions: `open-pr` prompts before `git push --force-with-lease` when the rebase rewrote history (skipped on clean fast-forwards) and shows a PR preview before `gh pr create`; `release` confirms the PR body/base/head before creation and the tag name/target before tag push

### Changed
- `address-pr-review` Step 2b now requires an explicit root-cause statement (structural cause / why naive patch fails / correct design) before editing, and Step 2c carries that statement into the reviewer reply so the reasoning is visible — trivial Low-severity one-line mechanical fixes are exempt

## [1.10.0] - 2026-04-15

### Added
- `create-branch` now supports `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, and `build` prefixes in addition to `feature`, `fix`, and `hotfix`
- `address-pr-review` summary now includes a severity × verdict breakdown table (Critical/High/Medium/Low × Fixed/False positive/Skipped) matching the `review-and-fix` format

### Changed
- `ship` no longer resolves inline PR review comments — that behavior is now exclusive to `address-pr-review`

## [1.9.0] - 2026-04-01

### Added
- `review-pr` command — reviews code and posts inline comments + summary comment directly on the PR
- `review-and-fix` command — reviews code and auto-fixes all findings (all severities, no flags needed)
- Dependencies skill now upgrades outdated packages automatically instead of just reporting them
- Security audit commands (`npm audit`, `pip-audit`) added to dependency detection

### Changed
- `review` renamed to `review-and-report-only` — report-only, no fix flags
- `address-review` renamed to `address-pr-review` for consistency with `review-pr`
- Removed `--fix`, `--fix-medium`, `--fix-all` flags from review (fixing is now a separate command)
- Dependencies skill requires CLI output as evidence — no more guessing versions from training data
- Removed silent error suppression (`2>/dev/null`) from all dependency detection commands
- CI review workflow template now references `review-pr` command

### Fixed
- Dependency detection commands were blocked by `review` command's `allowed-tools` — all package manager CLIs now whitelisted

## [1.8.0] - 2026-03-27

### Added
- New `rest-api` review profile: checks HTTP status codes, `Location` headers on 201, error envelope consistency, resource representation (no bare FK IDs), URL design, and method semantics
- `rest-api` auto-runs whenever a backend framework is detected (django, flask, fastapi, express, spring, laravel, rails, aspnet, go-http)

### Fixed
- `release` command now fetches before checking for the remote `dev` branch, and creates a local tracking branch if switching to `dev` for the first time

## [1.7.0] - 2026-03-25

### Added
- Ship command now runs type checkers (tsc, vue-tsc, mypy, pyright, go vet) before committing and blocks on errors

## [1.6.0] - 2026-03-25

### Added
- Testing skill now runs coverage scoped to changed files, prioritizes uncovered lines, and reports before/after coverage in its output

## [1.5.1] - 2026-03-25

### Fixed
- Writing profiles (testing, documentation) no longer appear in the severity summary table — they report what they wrote in their own dedicated sections instead

## [1.5.0] - 2026-03-25

### Added
- Release command now creates and pushes an annotated git tag (`v<version>`) after committing the version bump

## [1.4.1] - 2026-03-25

### Added
- Privacy policy (`PRIVACY.md`)
- Examples section in README with per-command descriptions
- `address-review` command added to README commands table

### Changed
- Updated plugin and README description to 3-sentence format
- Reordered commands table by relevance

## [1.4.0] - 2026-03-24

### Added
- `dependencies` skill — audits project dependencies for outdated packages across npm, pip, Go modules, Bundler, Composer, Maven, and NuGet. Reports findings bucketed by severity: CRITICAL (known CVE), HIGH (2+ major versions behind), MEDIUM (1 major behind), LOW (minor/patch behind). Auto-triggered when manifest files change in a PR.

## [1.3.0] - 2026-03-24

### Added
- Behavior mode routing in review agent prompt — writing profiles (testing, documentation) now receive explicit instructions to use Edit/Write tools locally instead of only reporting findings
- CI fallback for writing profiles — when Edit/Write tools are unavailable (CI), writing profiles fall back to inline comment and reporting mode automatically

### Changed
- Review Standards and False-Positive Validation sections scoped to reporting profiles only
- CI review prompt updated with explicit instructions for writing profile fallback to inline comments

## [1.2.0] - 2026-03-24

### Added
- Security checklist expanded with 13 new items: password hashing, tenant isolation, path traversal, unsafe deserialization, TLS enforcement, minimal containers, encryption at rest, audit trails, and new Cryptography and API Security sections
- Release command now suggests a semver version bump based on conventional commits and updates the changelog

### Changed
- Removed technology-specific references from agnostic skills and commands (security, quality, performance, testing, ship, open-pr, address-review) to keep them truly stack-agnostic

## [1.1.0] - 2026-03-24

### Added
- `address-review` command — fetches unresolved PR inline comments, validates each for false positives, fixes root causes, replies with explanations, resolves threads, and posts a summary table on the PR
- False-positive validation gate in `review` command — each profile agent now validates its own findings against surrounding code context, project conventions, actual reachability, and diff ownership before reporting

## [1.0.5] - 2026-03-24

### Added
- Self-review checklist item in `open-pr` PR body

### Changed
- CI review trigger changed to `@prism review` comment format
- Renamed `branch` command to `create-branch`

## [1.0.4] - 2026-03-23

### Added
- Expanded stack reviewer checklists with deeper coverage across all language/framework skills
- Installable CI review template and `install-ci-review` command

## [1.0.3] - 2026-03-22

### Fixed
- Resolve review threads via GraphQL after replying in `ship` command

## [1.0.2] - 2026-03-21

### Added
- Auto-resolve inline review comments after push in `ship` command

### Changed
- CI review switched to inline comments with deduplication and restricted trigger

## [1.0.1] - 2026-03-20

### Changed
- Simplified contribution model to single-maintainer fork+PR
- CI auto-approve job for maintainer PRs
- Review CI trigger simplified, added `@claude` comment trigger and allowedTools

## [1.0.0] - 2026-03-19

### Added
- Initial release as Claude Code plugin (`prism`)
- `review` command — multi-profile parallel code review with auto-detection of languages and frameworks
- `ship` command — pre-ship hygiene check, conventional commit, and push
- `open-pr` command — sync base branch, run tests, open PR with conventional commit title
- `create-branch` command — create feature/fix/hotfix branches from the correct base
- `release` command — open a release PR from dev into main
- Review profiles: quality, security, performance, documentation, testing
- Per-language stack skills: Python, TypeScript, Go
- Per-framework stack skills: Django, Flask, FastAPI, Next.js, Nuxt, SvelteKit, React, Vue, Express, Spring Boot, Laravel, ASP.NET Core, Rails, Go HTTP
- Profile ownership boundaries to prevent duplicate findings across profiles
- `--fix`, `--fix-medium`, `--fix-all` flags for automated fix mode in review
- GitHub Actions workflow for automated PR reviews
- Marketplace plugin support
