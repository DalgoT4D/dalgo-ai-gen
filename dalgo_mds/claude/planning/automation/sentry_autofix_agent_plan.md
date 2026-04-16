# Sentry Autofix Agent - Implementation Plan

## Context

Create a GitHub Actions workflow that runs daily at 10pm, fetches high-priority unresolved issues from Sentry (`dalgo` org / `dalgo-backend` project), uses Claude CLI in headless mode to generate fixes, runs the test suite, creates PRs, and then uses Claude again as a "senior reviewer" to add review comments. The team reviews PRs the next morning.

**Target repo**: `DalgoT4D/DDP_backend`
**Sentry**: org `dalgo`, project `dalgo-backend`, region `https://de.sentry.io`

---

## Architecture Overview

```
[Cron: 10 PM UTC daily]
        |
        v
+-------------------+
| Job: fetch-issues  |  (~30s)
|  - Call Sentry API |
|  - Filter issues   |
|  - Check branches  |
|  - Output matrix   |
+-------------------+
        |
        v (matrix of 0-2 issues)
+-----------------------------------------------+
| Job: fix-issue (per issue, sequential)          |
|                                                 |
| 1. Checkout repo + create branch                |
| 2. Fetch full Sentry event details              |
| 3. Generate rich prompt for Claude              |
| 4. Run Claude CLI to generate fix (~5-10 min)   |
| 5. Run pre-commit (black formatting)            |
| 6. Commit changes                               |
| 7. Apply migrations + run test suite (~5-10 min)|
| 8. Push branch + create PR (draft if tests fail)|
| 9. Run Claude CLI for PR review (~3-5 min)      |
| 10. Post review as PR comment                   |
+-----------------------------------------------+
```

Total expected runtime: 20-50 minutes depending on issue count and complexity.

---

## Files to Create

### 1. `.github/scripts/fetch_sentry_issues.py`

Python script that fetches and filters Sentry issues via REST API.

**Logic:**
- Calls `GET /api/0/projects/dalgo/dalgo-backend/issues/?query=is:unresolved&sort=freq&limit=25`
- For each issue, fetches latest event via `GET /api/0/organizations/dalgo/issues/{id}/events/latest/`
- **Filters** (skip issue if any fail):
  - Events >= 3 (skip transient one-offs)
  - Has first-party frames in `ddpui/` (skip third-party library errors)
  - Not already attempted (check if branch `sentry-autofix/{SHORT_ID}` exists via `git ls-remote`, or PR with label `sentry-autofix` exists via `gh pr list`)
- Selects top 2 issues (configurable via `MAX_ISSUES`)
- Outputs JSON array to `$GITHUB_OUTPUT` for matrix strategy
- Supports manual override via `MANUAL_ISSUE_ID` env var

**Key functions:**
```python
def fetch_issues() -> list:
    """Fetch unresolved issues sorted by frequency from Sentry API."""

def fetch_latest_event(issue_id: str) -> dict:
    """Fetch latest event with full stacktrace, breadcrumbs, tags."""

def has_first_party_frames(event_data: dict) -> bool:
    """Check if any stacktrace frame is in ddpui/ (in_app=True)."""

def is_already_attempted(short_id: str) -> bool:
    """Check git ls-remote + gh pr list for existing branch/PR."""
```

### 2. `.github/scripts/generate_claude_prompt.py`

Python script that builds a rich prompt from Sentry event data.

**Logic:**
- Fetches latest event for the issue (stacktrace, breadcrumbs, tags, local variables)
- Extracts and formats:
  - Exception type + message
  - Full stacktrace with `[APP]` markers for first-party frames
  - Source code context around crash lines
  - Local variables at crash site (first-party frames only, truncated to 200 chars)
  - Last 10 breadcrumbs
  - Tags (environment, browser, runtime)
- Constructs a prompt that tells Claude to:
  - Read `.claude/CLAUDE.md` for project conventions
  - Analyze the root cause (not just suppress the error)
  - Fix the bug with minimal changes
  - Write/update tests that would have caught the bug
  - Follow the layered architecture (API -> Core -> Models)
  - Not add new dependencies
  - Use the project's logging pattern (CustomLogger)
- Writes prompt to `/tmp/claude_fix_prompt.txt`

### 3. `.github/workflows/sentry-autofix.yml`

The main GitHub Actions workflow.

**Triggers:**
- `schedule: cron '0 22 * * *'` (daily 10pm UTC)
- `workflow_dispatch` with optional `issue_id` input for manual testing

**Concurrency:** `group: sentry-autofix, cancel-in-progress: false`

**Permissions:** `contents: write`, `pull-requests: write`, `issues: write`

#### Job 1: `fetch-issues` (~30s)
- Setup Python 3.10, install `requests`
- Run `fetch_sentry_issues.py`
- Outputs `issues` (JSON array) and `issue_count`

#### Job 2: `fix-issue` (per issue, sequential via `max-parallel: 1`)
- Mirrors existing CI environment exactly from `dalgo-ci.yml`:
  - Postgres 15 service, Redis 6, Python 3.10, `uv sync`
  - Same env vars: `DBHOST`, `DBPORT`, `DBNAME`, `DBUSER`, `DBPASSWORD`, `DJANGOSECRET`, etc.
- Steps:
  1. Checkout, setup Python/uv/Redis
  2. `uv sync` to install dependencies
  3. `npm install -g @anthropic-ai/claude-code`
  4. Create branch `sentry-autofix/{SHORT_ID}`
  5. Run `generate_claude_prompt.py`
  6. **Claude generates fix:**
     ```bash
     claude -p "$(cat /tmp/claude_fix_prompt.txt)" \
       --model claude-sonnet-4-20250514 \
       --permission-mode bypassPermissions \
       --allowedTools "Read,Edit,Grep,Glob,Write,Bash(uv run pytest:*),Bash(uv run python:*),Bash(git diff:*),Bash(git status:*)" \
       --max-turns 25 --output-format json
     ```
  7. Run `uv run pre-commit run --all-files` (black formatting)
  8. Check for changes (`git diff --quiet`)
  9. Commit with message: `fix: {ISSUE_TITLE}`
  10. Apply migrations + run full test suite:
      ```bash
      uv run coverage run -m pytest --ignore=ddpui/tests/integration_tests --durations=20
      ```
  11. Push branch, create PR:
      - Tests pass -> normal PR with label `sentry-autofix`
      - Tests fail -> **draft** PR with label `sentry-autofix-tests-failing`
  12. **Claude reviews PR:**
      ```bash
      claude -p "Review this PR diff as a senior Django developer...
        1. Does the fix address the root cause?
        2. Edge cases not handled?
        3. Could it introduce regressions?
        4. Follows project conventions?
        5. Security concerns?
        Provide: Summary, Analysis, Concerns, Suggestions" \
        --model claude-sonnet-4-20250514 \
        --permission-mode bypassPermissions \
        --allowedTools "Read,Grep,Glob" \
        --max-turns 10 --output-format text
      ```
  13. Post review as PR comment via `gh pr comment`
  14. Write workflow summary

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Track attempted issues | Branch naming + `gh pr list` check | Stateless, no tracking file needed, survives workflow re-runs |
| Issues per run | 2 max | Keeps runtime under 60min, avoids PR flood |
| Failed tests | Create draft PR | Team can still see the attempted fix direction |
| No changes from Claude | Skip PR creation, don't push branch | Allows retry on next run |
| Model for fixes | `claude-sonnet-4-20250514` | Good quality/cost/speed balance for automated fixes |
| Tool permissions | Explicit allowlist | Claude can read/edit/grep but only run `uv run pytest` and `git diff/status` via Bash - no destructive commands |
| Issue selection | Events >= 3, first-party frames, frequency-sorted | Fix impactful, actionable bugs only |

---

## Required Secrets

### New secrets to add:

| Secret | Purpose | How to create |
|--------|---------|---------------|
| `SENTRY_AUTH_TOKEN` | Sentry API auth | Settings > Auth Tokens at `de.sentry.io`. Scopes: `project:read`, `event:read`, `org:read` |
| `ANTHROPIC_API_KEY` | Claude CLI API access | From Anthropic Console |
| `DALGO_BOT_PAT` | GitHub PAT for push + PR creation | Fine-grained PAT with `contents:write`, `pull-requests:write` on `DalgoT4D/DDP_backend`. Needed because default `GITHUB_TOKEN` can't trigger CI on created PRs |

### Existing secrets reused:
`CI_DBPASSWORD`, `CI_DBNAME`, `AIRBYTE_API_TOKEN`

### GitHub labels to create:
- `sentry-autofix` — applied to all auto-generated PRs
- `sentry-autofix-tests-failing` — added when tests fail (PRs are drafts)

---

## Edge Cases Handled

| Scenario | Handling |
|----------|----------|
| Claude produces no changes | Detected via `git diff --quiet`, skip PR, don't push branch (retryable next run) |
| Claude times out | `--max-turns 25` + step `timeout-minutes: 20` |
| Sentry issue is config/data problem, not code bug | Claude instructed to only fix code; no changes = no PR |
| Branch already exists from failed prior run | `is_already_attempted()` detects it, issue is skipped |
| Concurrent workflow runs | Prevented by `concurrency: group: sentry-autofix` |
| Pre-existing test failures on main | Noted in PR body; team should diff against main |

---

## Team Workflow (Next Morning)

1. Filter PRs by label `sentry-autofix`
2. Read Claude's auto-review comment on each PR
3. Non-draft = tests pass; Draft = tests fail
4. Review the diff, check Sentry link, merge or close
5. After merge, monitor Sentry issue to see if error rate drops

---

## Implementation Order

1. Create GitHub labels (`sentry-autofix`, `sentry-autofix-tests-failing`)
2. Add secrets (`SENTRY_AUTH_TOKEN`, `ANTHROPIC_API_KEY`, `DALGO_BOT_PAT`)
3. Create `.github/scripts/fetch_sentry_issues.py`
4. Create `.github/scripts/generate_claude_prompt.py`
5. Create `.github/workflows/sentry-autofix.yml`
6. Test with `workflow_dispatch` + a specific Sentry issue ID
7. Iterate on Claude prompts based on fix quality
8. Enable daily cron once validated

---

## Verification

1. **Manual test**: Trigger workflow via `workflow_dispatch` with a known Sentry issue ID
2. **Check**: Branch created, PR opened, Claude review comment posted
3. **Validate**: Fix is reasonable, follows project conventions, tests pass (or draft if not)
4. **PR body**: Has Sentry link, event count, user impact, review checklist
5. **Monitor**: Watch first few automated runs, tune prompts and filters

---

## Future Improvements

- **Webhook trigger**: Instead of daily cron, trigger on new high-severity Sentry issues via webhook
- **Fix validation**: After merge, monitor Sentry issue error rate and auto-comment
- **Sentry issue commenting**: Post a link to the PR back on the Sentry issue
- **Seer integration**: Use Sentry Seer's root cause analysis API to pre-assess fixability before running Claude
- **Cost tracking**: Log Claude API token usage per run
- **Slack notification**: Post to a channel when PRs are created for morning review

---

## Key Reference Files

- `DDP_backend/.github/workflows/dalgo-ci.yml` — CI env vars and test setup to replicate
- `DDP_backend/.claude/CLAUDE.md` — Coding conventions included in Claude's prompt
- `DDP_backend/ddpui/settings.py` — Sentry SDK config (DSN, integrations, sample rates)
