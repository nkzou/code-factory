---
name: review
description: >
  Use when user says "review pr", "review pull request", "pr review", "review #123",
  or provides a PR number, URL, or branch name to review.
  Supports --codex and --claude-codex flags to delegate or cross-check with Codex.
argument-hint: "[PR number, URL, or branch name] [--codex|--claude-codex]"
user-invocable: true
allowed-tools: Bash(git:*), Bash(gh:*), Bash(node:*), Bash(command:*), Read, Grep, Glob, Task, AskUserQuestion
---

# Review PR

Announce: "I'm using the review skill to review a GitHub pull request."

## Context

- Current branch: !`git branch --show-current`
- Repository: !`basename $(git rev-parse --show-toplevel)`
- Remote URL: !`git remote get-url origin 2>/dev/null || echo "no remote"`

## Routing

| If you need... | Use instead |
|----------------|-------------|
| Address PR review feedback | `/pr-fix` |
| Run Codex review against local working tree only | `/codex:review` |
| Delegate an arbitrary task to Codex | `/codex:rescue` |

## Step 1: Parse Arguments

Tokenize `$ARGUMENTS` on whitespace and classify each token:

| Token | Bucket |
|-------|--------|
| `--codex` | `MODE=codex` |
| `--claude-codex` or `--dual` | `MODE=claude-codex` |
| `--no-worktree` | `WORKTREE=false` (diff-only fallback) |
| Pure digits | `PR_REF=<digits>` |
| Matches `github.com/.*/pull/(\d+)` | `PR_REF=$1` |
| Anything else, non-flag | candidate branch name → `PR_REF` |

Rules:
- Default `MODE=claude`, `WORKTREE=true`.
- `--codex` and `--claude-codex` are mutually exclusive — error and ask the user to pick one if both appear.
- If multiple non-flag tokens appear, treat the first as `PR_REF` and warn about the rest.
- If `PR_REF` is empty after parsing, fall through to current-branch detection in Step 3.

## Step 2: Verify Prerequisites

Run in parallel:
- `gh auth status 2>&1`
- `git rev-parse --show-toplevel 2>&1`
- If `MODE` is `codex` or `claude-codex`: `command -v codex 2>&1`

**If `gh` is not installed or not authenticated:** inform the user (`gh auth login`). Stop.

**If not a git repository:** inform the user. Stop.

**If `codex` CLI is unavailable in a Codex mode:**
- `MODE=codex` → warn: "Codex CLI not installed. Falling back to Claude review." Set `MODE=claude`.
- `MODE=claude-codex` → warn: "Codex CLI not installed. Running Claude-only." Set `MODE=claude`.

## Step 3: Identify the PR

Determine the PR from `PR_REF`:

| Input | Action |
|-------|--------|
| Number | Use directly |
| URL extracted in Step 1 | Already a number |
| Branch name | `gh pr list --head <branch> --json number --jq '.[0].number'` |
| Empty | `gh pr view --json number --jq '.number' 2>/dev/null` |

**If no PR found:** use `AskUserQuestion` to request a PR number, URL, or branch name. List recent PRs with `gh pr list --limit 5` as suggestions.

Bind the resolved number to `PR_NUMBER`.

## Step 4: Fetch PR Metadata and Diff

Run in parallel:

```bash
gh pr view "$PR_NUMBER" --json title,body,baseRefName,headRefName,headRefOid,author,additions,deletions,changedFiles,url,labels,closingIssuesReferences,files
gh pr diff "$PR_NUMBER"
```

Bind:
- `PR_TITLE`, `PR_BODY`, `BASE_REF`, `HEAD_REF`, `PR_HEAD_SHA` from the JSON.
- `CHANGED_FILES` from `.files[].path`.
- `LINKED_ISSUES` from `.closingIssuesReferences[].number` (if present).

If the diff is empty or binary-only, report and stop after listing the binary files.

## Step 5: Create Isolated Worktree

If `WORKTREE=false` (explicit `--no-worktree`), skip this step and operate on the diff only.

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_NAME=$(basename "$REPO_ROOT")
WT_DIR="$REPO_ROOT/../worktrees"
mkdir -p "$WT_DIR"
WT_PATH="$WT_DIR/${REPO_NAME}-review-pr-${PR_NUMBER}"

# Idempotent: remove any stale worktree from a previous run
git worktree remove --force "$WT_PATH" 2>/dev/null || true

# Fetch the exact head SHA so the worktree is pinned even if the PR updates mid-review
git fetch origin "pull/${PR_NUMBER}/head"
git worktree add --detach "$WT_PATH" "$PR_HEAD_SHA"
```

The worktree lives in `$REPO_ROOT/../worktrees/`, parallel to the repo, matching the convention used by `/worktree`.
The user's main checkout is untouched — uncommitted changes in their working tree are independent of the worktree.

**If `git worktree add` fails:** warn the user, set `WORKTREE=false`, and continue in diff-only mode. Codex will receive `Worktree: diff-only fallback` in its prompt.

All `Read`, `Grep`, and `Glob` calls during review must anchor on `$WT_PATH` when the worktree exists.

## Step 6: Extract PR Intent

Read in this order:
1. `PR_TITLE`
2. `PR_BODY`
3. Linked issues: `gh issue view <num>` for each entry in `LINKED_ISSUES`
4. Test files in the diff (added or modified `*_test.*`, `*.test.*`, `tests/`)
5. The diff itself as a signal of last resort

State the intent in 2-4 sentences. End with one of:
- "Stated explicitly in the PR body."
- "Inferred from diff — see Uncertainty below."

If inferred, add an **Uncertainty** block listing the signals used and any alternative interpretations rejected.

Keep this intent visible throughout the review.
Judge every change against it.

## Step 7: Build File Inventory

Initialize a coverage checklist from `CHANGED_FILES`:

| # | File | Module | Reviewed |
|---|------|--------|----------|
| 1 | `path/a.go` | <pending> | ☐ |
| 2 | `path/b.go` | <pending> | ☐ |

Every changed file must end the review with `Reviewed: ✓`.
A file is not reviewed until it appears under a Module section in the output.

## Step 8: Infer Logical Modules

Group `CHANGED_FILES` into modules by:
1. Package or import cohesion (e.g., all files importing the same internal package).
2. Naming and directory locality (files under `internal/auth/` typically form one module).
3. Functional intent (e.g., "session handling" may span `auth/`, `middleware/`, and `tests/`).

A module may span directories.
A directory may split into multiple modules if it contains unrelated changes.

Record `(module-name → [files])` and update the coverage table's Module column.

## Step 9: Mode Dispatch

| Mode | Action |
|------|--------|
| `claude` (default) | Step 10a only |
| `codex` | Step 10b only |
| `claude-codex` | Step 10a and Step 10b in parallel; merge in Step 10c |

## Step 10a: Claude Review

Apply the three-level framework (see `references/three-level-framework.md`) module by module.

For each module:
1. Read the files in the module from `$WT_PATH` (or from the diff if `WORKTREE=false`).
2. Walk all three levels (intent, logic, quality).
3. Record findings with Level, Severity (Critical/Major/Minor), Confidence (HIGH/MEDIUM/LOW), Location, Issue, Why, Impact, Best Fix.
4. Identify missing or insufficient tests.
5. Identify follow-up cleanup needed before merge.
6. Tick the coverage checklist for each file reviewed.

Render output using the single-reviewer template in `references/output-format.md`.

## Step 10b: Codex Review

Delegate via the `codex:codex-rescue` subagent.
The exact prompt template lives in `references/codex-delegation.md`.

```
Task(
  subagent_type = "codex:codex-rescue",
  description = "Codex PR review: #<PR_NUMBER>",
  prompt = <template with PR_TITLE, PR Intent, refs, worktree path, framework, output format, and full PR diff>
)
```

Pass through Codex's stdout verbatim.
Do not paraphrase or add commentary before or after.

**If Codex returns empty stdout** (the rescue subagent's failure signal):
- `MODE=codex` → ask the user via `AskUserQuestion` whether to retry with Claude.
- `MODE=claude-codex` → continue with Claude-only output, prefixed with `Codex unavailable; Claude-only`.

## Step 10c: Merge Dual Reviews

When `MODE=claude-codex` and both reviews succeed, render the dual-reviewer template from `references/output-format.md`:

1. Single PR Intent section (Claude's extraction is the source of truth).
2. Single File Coverage Checklist (each row ✓ only if both reviewers covered it).
3. Reviewer Comparison table (verdict, finding counts, unique findings, agreed findings).
4. Reconciliation section if reviewers disagree (1-3 sentences tied to PR Intent).
5. Combined Verdict line: `Claude: <verdict> | Codex: <verdict>`.
6. Full reviews under collapsible `<details>` blocks.

## Step 11: Cleanup Worktree

Always run, even on error:

```bash
git worktree remove --force "$WT_PATH" 2>/dev/null || true
git worktree prune
```

Verify with `git worktree list` — `$WT_PATH` must not appear.

## Step 12: Present Output

Render the review to the user.
Do NOT post it as a GitHub comment automatically.
Apply semantic line breaks: one sentence per line, break after clause-separating punctuation, target 120 characters per line.

If no findings exist at any level, state that and recommend approval.

## Rules

- Reference exact file paths and line numbers.
- Cite findings from the worktree contents when available, not from the diff alone.
- Be constructive: every issue includes a concrete fix.
- Every changed file must end the review marked `Reviewed: ✓`.
- The three-level framework applies to every PR uniformly — depth does not scale down for small PRs.

## Error Handling

| Error | Action |
|-------|--------|
| `gh` not installed/authenticated | Inform user to run `gh auth login`. Stop. |
| Both `--codex` and `--claude-codex` present | Ask user to pick one. Stop. |
| PR not found | Report. List recent PRs with `gh pr list --limit 5`. |
| Empty or binary-only diff | Report "no reviewable text changes". List binary files. Cleanup worktree if created. |
| `git worktree add` fails | Warn, fall back to diff-only mode, continue. |
| `codex` CLI missing in Codex mode | Warn and fall back to Claude (see Step 2). |
| Codex subagent returns empty stdout | Report failure. Offer retry with Claude or continue Claude-only in dual mode. |
| Stale worktree from prior run | `git worktree remove --force` then retry (built into Step 5). |
| Network/API failure | Report `gh` error. Cleanup worktree. Let user retry. |
