---
name: show-time
model: claude-opus-4-6
description: Full release pipeline — commit, docs, security review, PR, and adversarial code review.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, TodoWrite, SlashCommand, mcp__github__get_me, mcp__github__create_pull_request, mcp__github__merge_pull_request, mcp__github__pull_request_read, mcp__github__update_pull_request
---

# Show Time Skill

Orchestrate the full release pipeline: commit, update docs, security review, create PR, assign PR, adversarial code review, and push.

## When to Use

Use this skill when:
- User says "show time", "ship it", "release", "let's go", "launch"
- User wants to commit, create PR, and merge in one flow
- User asks for the full release pipeline
- User wants to go from code to merged PR

## Prerequisites

Before running this skill, ensure:
- All code changes are complete and working
- Lint and typecheck pass locally

## Core Principle: Self-Healing Pipeline

**Never stop and ask the user to fix failures. Diagnose, fix, and retry automatically.**

- Every step that can fail has a diagnose → fix → retry loop
- CI failures and code review fixes are handled by the PR Fixer agents (separate cron)
- Self-review uses a 3-agent adversarial system (Enthusiast → Adversary → Referee) before pushing
- Max **3 retry cycles** per step before escalating to the user
- Only escalate when truly stuck (needs credentials, access, or a decision only a human can make)

## Pipeline Steps

### Step 1: Commit by Context

Stage and commit files grouped by their context, each with its own conventional commit message. Do NOT lump everything into one commit.

**Process:**
1. Run `git status` to see all changes (never use `-uall`)
2. Run `git diff` and `git diff --cached` to understand all changes
3. Run `git log --oneline -5` to see recent commit style
4. **Group files by context** — analyze the changes and cluster related files together. Common groupings:
   - Backend API changes (routes, services, types)
   - Frontend page/component changes
   - i18n / translation changes
   - Analytics event changes
   - Database migrations
   - Configuration / infrastructure changes
   - Documentation changes
5. For each group, stage only its files and commit with a descriptive message
6. Never stage `.env`, credentials, or large binaries

```bash
# Example: group 1 — backend
git add apps/api/src/services/foo.service.ts apps/api/src/routes/foo.routes.ts
git commit -m "$(cat <<'EOF'
feat(api): add foo service and routes
EOF
)"

# Example: group 2 — frontend
git add apps/dashboard/src/components/foo/ apps/dashboard/src/app/foo/
git commit -m "$(cat <<'EOF'
feat(dashboard): add foo page and components
EOF
)"

# Example: group 3 — i18n
git add apps/dashboard/src/lib/i18n/translations.ts
git commit -m "$(cat <<'EOF'
feat(dashboard): add foo translations
EOF
)"
```

**Commit message format:**
- `feat(<scope>): <description>` for new features
- `fix(<scope>): <description>` for bug fixes
- `refactor(<scope>): <description>` for refactoring
- `docs(<scope>): <description>` for documentation
- `chore(<scope>): <description>` for maintenance

**Grouping guidelines:**
- Prefer smaller, focused commits over large ones
- Each commit should be self-contained and make sense on its own
- If two files are tightly coupled (e.g., a service and its types), they belong in the same commit
- Migrations get their own commit
- i18n changes can be grouped with the feature they support or stand alone if they span multiple features

**If commit fails** (pre-commit hooks): read the hook output, fix the issues, re-stage, and create a NEW commit (never amend).

### Step 2: Update Documentation

Invoke the docs-update skill to update all relevant documentation.

```
/docs-update-skill
```

This handles: OpenAPI specs, ERD diagrams, FEATURES.md, i18n translations, FAQ, sidebar, and any other documentation affected by the changes.

If the docs-update skill produces changes, commit them separately:
```bash
git add <doc-files>
git commit -m "$(cat <<'EOF'
docs: update documentation for <feature>
EOF
)"
```

### Step 3: Security Review

Perform a security review of all changes before creating the PR.

**Process:**
1. Review all changed files for security vulnerabilities:
   ```bash
   git diff main --name-only
   ```

2. **Check for OWASP Top 10 issues:**
   - **Injection** — SQL injection, NoSQL injection, command injection in any user input handling
   - **Broken Authentication** — JWT misuse, session handling, credential exposure
   - **Sensitive Data Exposure** — Hardcoded secrets, API keys, tokens in code or logs
   - **XSS** — Unsanitized user input rendered in HTML/React components
   - **Broken Access Control** — Missing authorization checks, IDOR vulnerabilities
   - **Security Misconfiguration** — Permissive CORS, debug mode, default credentials
   - **CSRF** — Missing CSRF tokens on state-changing endpoints

3. **Check for common vulnerabilities:**
   - No hardcoded secrets, API keys, or credentials in code
   - No `.env` files, `credentials.json`, or sensitive files staged
   - Proper input validation on all user-facing endpoints (Zod schemas, etc.)
   - Authentication/authorization enforced on all protected routes
   - No `eval()`, `dangerouslySetInnerHTML`, or unsafe dynamic execution
   - Parameterized queries for all database operations (Prisma handles this)
   - Rate limiting on authentication endpoints
   - Proper error handling that doesn't leak stack traces or internal details

4. **Frontend-specific checks:**
   - No sensitive data stored in localStorage/sessionStorage
   - CSP headers configured for embedded content
   - External URLs validated before navigation
   - Form inputs properly sanitized

5. **Report and auto-fix:**
   - **PASS** — No security issues found, proceed to PR creation
   - **WARN** — Minor issues found, fix them and proceed
   - **FAIL** — Critical security issues found, fix them immediately

**Always fix any findings** before proceeding:
```bash
git add <fixed-files>
git commit -m "$(cat <<'EOF'
fix(security): address security review findings
EOF
)"
```

### Step 4: Create PR

Invoke the create-pr skill to run quality checks and create the pull request.

```
/create-pr-skill
```

This handles: lint, typecheck, unit tests, integration tests, migration validation, test coverage for new code, plan file management, and PR creation.

**Capture the PR number and URL** from the output — they're needed for subsequent steps.

### Step 5: Assign PR to User

After the PR is created, assign it to the authenticated GitHub user.

**Process:**
1. Get the authenticated user via `mcp__github__get_me`
2. Assign the PR using `gh`:

```bash
gh pr edit <PR_NUMBER> --add-assignee <username>
```

### Step 7: Adversarial Code Review

Instead of watching CI (the PR Fixer agents handle CI failures and code review fixes separately), the implementer runs a self-review before pushing the PR. This uses a three-agent adversarial system to find real bugs before CI even runs.

**Agent 1: The Enthusiast (Bug-finder)**
- Role: Hyper-enthusiastic bug hunter. Rewarded with points for finding bugs based on severity (Critical = 10pts, Medium = 5pts, Low = 1pt).
- Behavior: Reviews the entire diff and produces a massive list of potential bugs, security issues, race conditions, and edge cases. Will find both real and speculative issues to maximize its score.
- Prompt: "You are a bug bounty hunter reviewing this code diff. You earn points for every real bug you find. Critical bugs = 10 points. Medium = 5. Low = 1. Find as many as you can. Be thorough and aggressive."

**Agent 2: The Adversary (Debunker)**
- Role: Skeptical challenger. Gains points for successfully debunking a bug from Agent 1's list, but faces strict penalties (-5pts) for incorrectly dismissing a real bug. This makes it aggressive but cautious.
- Behavior: Goes through each item in Agent 1's list and tries to disprove it. Checks if the "bug" is actually handled elsewhere, if the edge case is impossible given the data model, or if the concern is purely theoretical.
- Prompt: "You are a code defense attorney. For each bug reported below, prove it is NOT a real bug. You earn 3 points for each bug you successfully debunk. But you LOSE 5 points if you dismiss a bug that turns out to be real. Be aggressive but careful."

**Agent 3: The Referee (Judge)**
- Role: Impartial judge. Evaluates the competing claims of Agents 1 and 2 to determine the ground truth. Rewarded for accuracy, penalized for errors.
- Behavior: For each disputed item, reads both arguments, examines the actual code, and renders a verdict: REAL BUG (must fix), FALSE POSITIVE (ignore), or WORTH NOTING (optional improvement).
- Prompt: "You are a senior staff engineer judging a code review debate. For each item, you have the bug report and the counterargument. Examine the actual code and render a verdict: REAL BUG, FALSE POSITIVE, or WORTH NOTING. You earn 10 points for matching the ground truth and lose 10 for errors. Be precise."

**Flow:**
```
1. Run Agent 1 (Enthusiast) on the full diff → produces bug list
2. Run Agent 2 (Adversary) on Agent 1's list → produces counterarguments
3. Run Agent 3 (Referee) on both outputs → produces final verdict
4. Fix all items marked as REAL BUG
5. Optionally address WORTH NOTING items
6. Commit fixes and push
```

**Why this works:** Each agent has a different incentive structure. The Enthusiast over-reports to maximize points. The Adversary aggressively filters to earn debunking points but is careful not to dismiss real issues (penalty is higher than reward). The Referee has no bias toward either side and is incentivized purely for accuracy. The result is a high-fidelity list of actual bugs with very few false positives or missed issues.

**After pushing:** CI and the Claude Code Review will run automatically. Any remaining issues will be caught and fixed by the PR Fixer agents on their next cycle. The implementer does NOT wait for CI.

## Example Workflow

**User:** "Show time!"

**Assistant:**
1. Groups changes by context and creates focused commits:
   - `feat(api): add payment method CRUD service and routes`
   - `feat(dashboard): add payment methods page with list, panel, and tile components`
   - `feat(dashboard): add payment method edit form with SearchableSelect dropdowns`
   - `feat(dashboard): add payment method translations and analytics events`
   - `feat(database): add crypto emoji migration`
2. Runs docs-update skill → commits doc changes
3. Runs security review → PASS (no issues found)
4. Runs create-pr skill → quality checks pass → PR #142 created
5. Assigns PR #142 to the user
7. Reports: "PR #142 created and ready for review!"

## Checklist

- [ ] Code changes committed (grouped by context, multiple commits)
- [ ] Documentation updated and committed
- [ ] Security review passed (all findings fixed)
- [ ] Quality checks pass (lint, typecheck, tests)
- [ ] PR created with proper title and description
- [ ] PR assigned to user
- [ ] CI checks all green (auto-fixed if needed)
- [ ] Claude Code Review completed with no 🔴 critical issues (auto-fixed if needed)

## Playground Test

To verify the skill triggers and runs correctly, use this command in Claude Code:

**Command:**
```
/show-time-skill
```

**Precondition:** Have at least one uncommitted change in the working tree (e.g., a trivial edit to a file).

**Expected Result:**
1. **Step 1 (Commit by Context)** — Claude runs `git status`, `git diff`, and `git log`, then groups changed files by context and creates one or more focused conventional commits (e.g., `feat(api): ...`, `fix(dashboard): ...`).
2. **Step 2 (Update Documentation)** — The `docs-update-skill` is invoked. If it produces doc changes, they are committed separately with a `docs:` prefix.
3. **Step 3 (Security Review)** — Claude reviews the diff against OWASP Top 10 and common vulnerability checks. Output is one of: `PASS`, `WARN` (auto-fixed), or `FAIL` (auto-fixed). No user intervention expected.
4. **Step 4 (Create PR)** — The `create-pr-skill` is invoked. Lint, typecheck, and tests run. A GitHub PR is created with a title and description summarizing the changes. The PR number and URL are printed.
5. **Step 5 (Assign PR)** — The PR is assigned to the authenticated GitHub user via `gh pr edit --add-assignee`.
6. **Step 6 (Adversarial Code Review)** — Three agents run sequentially (Enthusiast → Adversary → Referee). The final verdict lists items as `REAL BUG`, `FALSE POSITIVE`, or `WORTH NOTING`. Any `REAL BUG` items are auto-fixed, committed, and pushed.

**Success criteria:** The pipeline completes end-to-end without asking the user to fix anything (self-healing). The PR is created, reviewed, and pushed.
