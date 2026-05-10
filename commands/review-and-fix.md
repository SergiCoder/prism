---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(cat*), Bash(grep*), Bash(npm outdated*), Bash(npm audit*), Bash(npm view*), Bash(npm install*), Bash(npm update*), Bash(yarn outdated*), Bash(yarn audit*), Bash(yarn upgrade*), Bash(pnpm outdated*), Bash(pnpm audit*), Bash(pnpm update*), Bash(pip list*), Bash(pip-audit*), Bash(pip index*), Bash(pip install*), Bash(go list*), Bash(go get*), Bash(go version*), Bash(bundle outdated*), Bash(bundle update*), Bash(composer outdated*), Bash(composer update*), Bash(mvn versions*), Bash(dotnet list*), Bash(dotnet add*), Bash(node --version*), Bash(python --version*), Bash(python3 --version*), Bash(ruby --version*), Bash(php --version*), Bash(java --version*), Bash(dotnet --version*), Bash(npm run*), Bash(npm test*), Bash(yarn*), Bash(pnpm*), Bash(npx tsc*), Bash(tsc*), Bash(pytest*), Bash(mypy*), Bash(ruff*), Bash(go build*), Bash(go test*), Bash(go vet*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read, Edit, Write, Glob, Grep, Agent
description: Review code changes and fix all findings automatically
model: claude-sonnet-4-6
---

Review code changes on the current branch using specialized reviewer profiles, then automatically fix all findings.

## Arguments

- `$ARGUMENTS` — optional: comma-separated profile names (e.g., `security,performance`)

If no profiles specified, auto-detect from changed files.

## Current state

**Current branch:** !`git branch --show-current`

## Instructions

### Step 1 — Determine base branch and diff

Determine the base branch: check if `dev` branch exists — if yes use `dev`, otherwise use `main`.

Get the list of changed files:
```bash
git diff origin/<base>...HEAD --name-only
```

If the diff is empty, stop:
> "No changes compared to `<base>` — nothing to review."

### Step 2 — Detect stack

Follow **Step A** in `commands/_detect-stack.md` to identify the languages and frameworks in use from the diff. Carry forward the resulting list of detected languages, frameworks, and stack skills.

### Step 3 — Select profiles

Follow **Step B** in `commands/_detect-stack.md` to determine which profiles to run. Honor `$ARGUMENTS` if provided. The profile ownership boundaries defined there govern the agent's `## Ownership Boundary` reference in Step 4.

> **Note:** the `dependencies` profile is a writing profile — it upgrades packages automatically.

### Step 4 — Run reviews in parallel

For each selected profile, launch an Agent with `subagent_type: "general-purpose"` and `run_in_background: true` using this prompt:

```
You are reviewing code changes for a pull request.

## Profile
Read and follow `skills/<profile>/SKILL.md`. It defines your scope, rules, behavior, and output format.

## Context
- Read CLAUDE.md (and any sub-CLAUDE.md files) for project rules and architecture decisions.
- Get the diff: `git diff origin/<base>...HEAD`
- Read changed files relevant to your profile's scope.

## Behavior Mode
**All profiles fix issues directly.** After identifying findings, apply fixes using the Edit and Write tools. Do not just report — fix.

- **Writing profiles** (testing, documentation, dependencies): follow your SKILL.md behavior to write tests, fix docs, or upgrade packages directly.
- **Reporting profiles** (all others): identify findings following the Review Standards and False-Positive Validation below, then **fix each finding** using the Edit tool. Summarize what you fixed.

**CI / read-only context:** If you do not have access to Edit or Write tools (e.g., running in CI), fall back to inline comment and reporting mode regardless of your profile. Post what you would have written as findings with file paths, suggested content, and severity, using the same inline comment flow as reporting profiles.

## Review Standards (reporting profiles only)
Before reporting any finding, verify ALL of these:
- You can point to a specific line or range in the diff
- You can name the exact rule from the checklist it violates
- The fix is a concrete code change, not a design opinion
If any criterion fails, do not report the finding.

Each finding must include a root-cause explanation: what structural design problem causes the issue and what the correct design would be.

## False-Positive Validation (reporting profiles only)
Before including any finding in your output, validate it is a real issue:

1. **Code context** — Read the file and surrounding lines (not just the diff hunk). Does the broader context already resolve the concern? (e.g., validation in a caller, a test covering the case, a comment explaining intent)
2. **Project conventions** — Check CLAUDE.md rules and existing patterns. Is the flagged code consistent with established project conventions?
3. **Actual impact** — Is the issue reachable in practice, or is it blocked by type system guarantees, framework behavior, or upstream validation?
4. **Diff ownership** — Is the flagged code actually changed in this PR, or is it pre-existing context?

**Drop** the finding if:
- Surrounding code already handles the concern
- It contradicts project conventions documented in CLAUDE.md
- The flagged code is not changed in the diff (pre-existing)
- The recommendation is speculative ("consider", "might want to") rather than a concrete rule violation

Only report findings that survive this validation.

## Ownership Boundary
Only report findings within your profile's domain.
If a finding crosses domains, defer to the profile listed first in the ownership boundary list.
```

Launch ALL agents in parallel. Do NOT run sequentially.

### Step 5 — Verify nothing broke

After all agents finish applying fixes, run the shared verification checks defined in `commands/_verify.md` to confirm the edits didn't introduce regressions. Record which checks ran and their pass/fail status for the final report.

### Step 6 — Collect and present results

Wait for all agents to complete, then present:

```markdown
# Review & Fix Report

**Branch:** <branch>
**Base:** <base>
**Files changed:** <count>
**Stack detected:** <languages and frameworks>
**Profiles run:** <list>
**Verification:** <which checks ran, pass/fail>

---

## Fixes Applied
(All fixes across all profiles, grouped by file. For each fix: severity, profile, what was wrong, what was changed.)

## Tests Written
(If testing profile ran — list tests added)

## Documentation Fixes Applied
(If documentation profile ran — list files updated)

## Dependencies Updated
(If dependencies profile ran — list packages upgraded with old → new versions)

## Summary

| Profile | Critical | High | Medium | Low | Fixed | Skipped (CI-only) |
|---|---|---|---|---|---|---|
| security | X | X | X | X | X | X |
| quality | X | X | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** | **X** | **X** |

**Do NOT include writing profiles (testing, documentation, dependencies) in the summary table.** Show their results only in their dedicated sections above.
```

### Step 7 — Final output

```
Review & fix complete: X findings fixed, Y tests written, Z docs updated, W packages upgraded across N profiles.
```

- If all findings were fixed and verification passed: "All findings resolved and checks passing. Ready for PR."
- If verification failed: list the failing checks and say "Fixes applied but verification failed — review before opening PR."
- If some findings could not be auto-fixed: list them with explanation and say "These require manual intervention."
