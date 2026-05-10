---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(cat*), Bash(grep*), Bash(npm outdated*), Bash(npm audit*), Bash(npm view*), Bash(yarn outdated*), Bash(yarn audit*), Bash(pnpm outdated*), Bash(pnpm audit*), Bash(pip list*), Bash(pip-audit*), Bash(pip index*), Bash(go list*), Bash(go version*), Bash(bundle outdated*), Bash(composer outdated*), Bash(mvn versions*), Bash(dotnet list*), Bash(node --version*), Bash(python --version*), Bash(python3 --version*), Bash(ruby --version*), Bash(php --version*), Bash(java --version*), Bash(dotnet --version*), Read, Edit, Write, Glob, Grep, Agent
description: Run multi-profile code review on the current branch
model: claude-sonnet-4-6
---

Review code changes on the current branch using specialized reviewer profiles.

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
Some profiles **write code directly** (testing, documentation) while others **report findings** (all other profiles). Your SKILL.md specifies which mode applies. Follow its behavior section exactly:
- If your SKILL.md says to write tests, fix docs, or make code changes — do so using the Edit and Write tools. Summarize what you wrote, do not report findings by severity.
- If your SKILL.md says to report findings — follow the Review Standards and False-Positive Validation sections below.

**CI / read-only context:** If you do not have access to Edit or Write tools (e.g., running in CI), fall back to inline comment and reporting mode regardless of your profile. Post what you would have written as findings — missing tests, stale docs — with file paths, suggested content, and severity, using the same inline comment flow as reporting profiles.

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

### Step 5 — Collect and present results

Wait for all agents to complete, then present:

```markdown
# Code Review Report

**Branch:** <branch>
**Base:** <base>
**Files changed:** <count>
**Stack detected:** <languages and frameworks>
**Profiles run:** <list>

---

## Critical & High Findings
(All critical/high across reporting profiles, grouped by file. Deduplicate — keep most detailed, note both profiles.)

## Medium Findings
(Same format)

## Low Findings
(Same format)

## Tests Written
(If testing profile ran — list tests added, not severity counts)

## Documentation Fixes Applied
(If documentation profile ran — list files updated, not severity counts)

## Summary

| Profile | Critical | High | Medium | Low |
|---|---|---|---|---|
| security | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** |

**Do NOT include testing or documentation in the summary table.** They are writing profiles — they write code and docs directly, not report findings by severity. Show their results only in the "Tests Written" and "Documentation Fixes Applied" sections above.
```

### Final output

```
Review complete: X critical, Y high, Z medium, W low findings across N profiles.
```

The finding counts and profile count exclude writing profiles (testing, documentation). Those profiles report what they wrote, not findings.

- Critical findings: "CRITICAL findings must be resolved before opening a PR."
- High or below only: "HIGH findings should be resolved before merge. Run `/prism:review-and-fix` to fix them automatically."
- Clean: "No critical or high findings. Ready for PR."
