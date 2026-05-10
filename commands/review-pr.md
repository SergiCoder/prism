---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(cat*), Bash(grep*), Bash(gh pr view*), Bash(gh api*), Bash(npm outdated*), Bash(npm audit*), Bash(npm view*), Bash(yarn outdated*), Bash(yarn audit*), Bash(pnpm outdated*), Bash(pnpm audit*), Bash(pip list*), Bash(pip-audit*), Bash(pip index*), Bash(go list*), Bash(go version*), Bash(bundle outdated*), Bash(composer outdated*), Bash(mvn versions*), Bash(dotnet list*), Bash(node --version*), Bash(python --version*), Bash(python3 --version*), Bash(ruby --version*), Bash(php --version*), Bash(java --version*), Bash(dotnet --version*), Read, Glob, Grep, Agent
description: Review current branch's PR and post inline comments with a summary
model: claude-sonnet-4-6
---

Review code changes on the current branch and post findings as inline comments on the associated PR.

## Arguments

- `$ARGUMENTS` — optional: comma-separated profile names (e.g., `security,performance`)

If no profiles specified, auto-detect from changed files.

## Current state

**Current branch:** !`git branch --show-current`

**PR info:**
!`gh pr view --json number,url,baseRefName,headRefName 2>/dev/null || echo "NO_PR_FOUND"`

## Instructions

### Step 1 — Validate PR exists

If `NO_PR_FOUND` is shown above, **stop immediately**:
> "No open PR found for this branch. Push your branch and open a PR first, then re-run `/prism:review-pr`."

Extract from the PR info: `<number>`, `<base>`, `<head>`, `<url>`.
Determine the repository owner and name from the URL or from `gh repo view --json owner,name`.

### Step 2 — Get diff and changed files

```bash
git diff origin/<base>...HEAD --name-only
```

If the diff is empty, stop:
> "No changes compared to `<base>` — nothing to review."

### Step 3 — Detect stack

Follow **Step A** in `commands/_detect-stack.md` to identify the languages and frameworks in use from the diff. Carry forward the resulting list of detected languages, frameworks, and stack skills.

### Step 4 — Select profiles

Follow **Step B** in `commands/_detect-stack.md` to determine which profiles to run. Honor `$ARGUMENTS` if provided. The profile ownership boundaries defined there govern the agent's `## Ownership Boundary` reference in Step 6.

### Step 5 — Fetch existing review comments

Before running reviews, fetch existing bot comments on the PR to avoid duplicates:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --jq '[.[] | select(.user.login == "github-actions[bot]" or .user.type == "Bot") | {path, line, body}]'
```

Store this list for deduplication in Step 7.

### Step 6 — Run reviews in parallel

For each selected profile, launch an Agent with `subagent_type: "general-purpose"` and `run_in_background: true` using this prompt:

```
You are reviewing code changes for a pull request. This is a report-only review — do NOT edit or write any files.

## Profile
Read and follow `skills/<profile>/SKILL.md`. It defines your scope, rules, and output format.

## Context
- Read CLAUDE.md (and any sub-CLAUDE.md files) for project rules and architecture decisions.
- Get the diff: `git diff origin/<base>...HEAD`
- Read changed files relevant to your profile's scope.

## Behavior Mode
**All profiles report findings only.** Do not edit or write any files — not even writing profiles (testing, documentation, dependencies). Report what you find with file paths, line numbers, severity, and recommended fix.

## Review Standards
Before reporting any finding, verify ALL of these:
- You can point to a specific line or range in the diff
- You can name the exact rule from the checklist it violates
- The fix is a concrete code change, not a design opinion
If any criterion fails, do not report the finding.

Each finding must include a root-cause explanation: what structural design problem causes the issue and what the correct design would be.

## False-Positive Validation
Before including any finding in your output, validate it is a real issue:

1. **Code context** — Read the file and surrounding lines (not just the diff hunk). Does the broader context already resolve the concern?
2. **Project conventions** — Check CLAUDE.md rules and existing patterns. Is the flagged code consistent with established project conventions?
3. **Actual impact** — Is the issue reachable in practice, or is it blocked by type system guarantees, framework behavior, or upstream validation?
4. **Diff ownership** — Is the flagged code actually changed in this PR, or is it pre-existing context?

**Drop** the finding if:
- Surrounding code already handles the concern
- It contradicts project conventions documented in CLAUDE.md
- The flagged code is not changed in the diff (pre-existing)
- The recommendation is speculative ("consider", "might want to") rather than a concrete rule violation

Only report findings that survive this validation.

## Output Format
For each finding, return a structured block:
- **file**: path relative to repo root
- **line**: line number in the current file (must be within the diff)
- **severity**: CRITICAL, HIGH, MEDIUM, or LOW
- **profile**: <profile name>
- **title**: short title
- **body**: explanation and recommended fix

## Ownership Boundary
Only report findings within your profile's domain.
If a finding crosses domains, defer to the profile listed first in the ownership boundary list.
```

Launch ALL agents in parallel. Do NOT run sequentially.

### Step 7 — Post inline comments on the PR

Wait for all agents to complete. Collect all findings.

For each finding:
1. **Check for duplicates** — compare against existing comments from Step 5. Skip if a comment already exists on the same file and line with the same title.
2. **Verify the line is in the diff** — only lines that appear in the PR diff can receive inline comments. If the finding is on a line outside the diff, save it for the summary comment instead.
3. **Post the inline comment:**
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/comments \
     -f body="**{SEVERITY}** — {profile}: {title}

   {body}" \
     -f path="{file}" \
     -f commit_id="$(git rev-parse HEAD)" \
     -F line={line} \
     -f side="RIGHT"
   ```

### Step 8 — Post summary comment on the PR

After all inline comments are posted, post a single summary comment on the PR:

```bash
gh api repos/{owner}/{repo}/issues/{number}/comments \
  -f body="$(cat <<'EOF'
# Code Review

**Branch:** <head> → <base>
**Files changed:** <count>
**Stack detected:** <languages and frameworks>
**Profiles run:** <list>
**Inline comments posted:** <count>
**Duplicates skipped:** <count>

| Profile | Critical | High | Medium | Low |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |
| **Total** | **X** | **X** | **X** | **X** |

<If findings outside the diff exist, list them here under a "Findings outside diff" section>

---
*Reviewed by [Prism](https://github.com/SergiCoder/prism)*
EOF
)"
```

If there are no findings at all:
```
# Code Review

No issues found. Ready to merge.

---
*Reviewed by [Prism](https://github.com/SergiCoder/prism)*
```

### Final output

```
Review posted to PR #<number> (<url>):
- <X> inline comments posted
- <Y> duplicates skipped
- Total: X critical, Y high, Z medium, W low
```
