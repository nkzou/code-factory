---
name: workspace
description: >
  Use when the user wants to manage Datadog workspaces (remote cloud development
  environments): create, list, delete, SSH, connect IDE, or validate setup.
  Also use when the `/do` skill needs a remote workspace instead of a local worktree.
  Triggers: "workspace", "create workspace", "list workspaces", "delete workspace",
  "ssh workspace", "datadog workspace", "remote dev environment", "connect workspace".
argument-hint: "[create|list|delete|ssh|connect|validate] [workspace-name]"
user-invocable: true
---

# Datadog Workspace Manager

Announce: "I'm using the workspace skill to manage Datadog workspaces."

Manage remote cloud development environments (CDEs) via the `workspaces` CLI. Workspaces are dev containers running on dedicated EC2 instances with pre-configured tools and repo access.

## Step 1: Parse Mode

Parse `$ARGUMENTS` to determine the operation:

| Argument prefix | Mode |
|----------------|------|
| `create <name>` | Create a new workspace |
| `list` | List existing workspaces |
| `delete <name>` | Delete a workspace |
| `ssh <name>` | SSH into a workspace |
| `connect <name>` | Connect IDE to a workspace |
| `validate` | Validate prerequisites and setup |
| No arguments | Ask user which operation |

If no arguments:

```
AskUserQuestion(
  header: "Workspace operation",
  question: "What would you like to do?",
  options: [
    "Create" -- Create a new Datadog workspace,
    "List" -- List existing workspaces,
    "Delete" -- Delete a workspace,
    "SSH" -- SSH into a workspace,
    "Connect" -- Connect IDE to a workspace,
    "Validate" -- Check prerequisites and setup
  ]
)
```

## Step 2: Validate Prerequisites

Run before `create`, `ssh`, or `validate` modes. Skip for `list`, `delete`, `connect`.

### 2a: CLI and Network

Check in parallel:

```bash
which workspaces
workspaces list 2>&1
```

| Check | Pass | Fail action |
|-------|------|-------------|
| `workspaces` CLI | Binary found | `brew update && brew install datadog-workspaces` |
| Appgate VPN | `workspaces list` succeeds | "Connect to Appgate VPN before continuing" |
| GitHub auth | `workspaces list` succeeds | `ddtool auth github login` |

If CLI not installed, offer to install:

```bash
brew update && brew install datadog-workspaces
```

### 2b: Pre-Flight Auth Checks

Run the local pre-flight checks from [references/auth-setup.md](references/auth-setup.md):

1. **SSH agent** -- verify `ssh-add -l` shows keys before any `-A` forwarding.
   If no keys, offer fix (`eval $(ssh-agent) && ssh-add`). Block until resolved.
2. **GitHub auth** -- verify `ddtool auth github status`.
   If expired, run `ddtool auth github login` (may trigger OIDC -- surface device codes immediately per auth-setup.md).

Surface failures with AskUserQuestion and offer fixes per auth-setup.md procedures.
Stop and fix blocking failures before proceeding.

For `validate` mode: report all check results and stop.

## Step 3: Create Workspace

### 3a: Gather Parameters

Determine workspace parameters from arguments and current context:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
REPO_NAME=$(basename "$REPO_ROOT" 2>/dev/null)
CURRENT_BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
WS_PREFIX=$(whoami | cut -d. -f1)
```

The workspace name is always prefixed with the user's first name (extracted from the OS username before the first `.`), followed by `-`. For example, if `whoami` returns `rodrigo.fernandes` and the user provides `my-feature`, the final name is `rodrigo-my-feature`.

If the user-provided name already starts with the prefix, do not double it.

| Parameter | Source | Default |
|-----------|--------|---------|
| `name` | `$WS_PREFIX-<slug>` where slug is from arguments after "create" | Ask user for slug |
| `--repo` | Current git repo name | Ask user if not in a repo |
| `--branch` | New feature branch via `/branch` | Created and pushed to remote |
| `--region` | User preference | `eu-west-3` |
| `--shell` | Always | `fish` |
| `--instance-type` | Always | `aws:m6gd.4xlarge` (ARM Graviton2) |

If no name provided:

```
AskUserQuestion(
  header: "Workspace name",
  question: "Name for the new workspace? (will be prefixed with '$WS_PREFIX-')",
  options: []
)
```

After deriving the final workspace name, check for conflicts:

```bash
workspaces list 2>/dev/null | grep -q "<final-name>"
```

If a workspace with that name already exists, ask the user:

```
AskUserQuestion(
  header: "Name conflict",
  question: "Workspace '<final-name>' already exists. What would you like to do?",
  options: [
    "Use a different name" -- I'll pick a new name,
    "Delete and recreate" -- Delete the existing workspace first,
    "Connect to existing" -- SSH or connect IDE to the existing workspace
  ]
)
```

### 3b: Optionally Create Feature Branch

Ask whether to create a new branch for this workspace:

```
AskUserQuestion(
  header: "Feature branch?",
  question: "Create a new feature branch for this workspace?",
  options: [
    "Yes — create new branch" -- Create and push a new branch,
    "No — use current branch '<CURRENT_BRANCH>'" -- Work from the current branch
  ]
)
```

If yes, create and push:

```
Skill(skill="branch", args="<name or feature description>")
```

```bash
git push -u origin <branch-name>
```

If no: use `$CURRENT_BRANCH` as `<branch-name>` throughout the remaining steps.

### 3c: Create Workspace

Run the create command **in the background** (`run_in_background: true`) — it takes 10-20 minutes:

```bash
workspaces create <name> \
  --repo <repo> \
  --branch <branch-name> \
  --region eu-west-3 \
  --instance-type aws:m6gd.4xlarge \
  --shell fish
```

Omit `--repo` if not in a git repo and user doesn't specify one.

Dotfiles are auto-applied from `DataDog/workspaces-dotfiles/users/<first>.<last>/`.
Do not pass `--dotfiles` here — it would override the auto-apply chain.

**Do NOT wait for the command to finish.** Report immediately after launching.
The background task will notify when complete — do not poll or sleep.

### 3d: Report Creation Status

Report immediately after launching the background create:

```
Workspace "<name>" is being created on branch "<branch-name>" (takes ~10-20 min).
I'll start a tmux session when it's ready.

Status:  workspaces list
```

### 3e: Post-Creation Auth Setup

After SSH config is verified and before starting tmux with Claude,
run the post-creation auth setup from [references/auth-setup.md](references/auth-setup.md).

Execute remote auth checks sequentially (each may trigger OIDC device-code flows).
Warn the user before starting -- device codes expire in minutes:

```
AskUserQuestion(
  header: "Auth setup",
  question: "About to run auth setup on the workspace. This may prompt for browser-based OIDC login. Please be ready. Proceed?",
  options: [
    "Proceed" -- I'm ready for browser auth prompts,
    "Skip auth" -- Continue without auth setup (may cause failures later)
  ]
)
```

If "Proceed", run each check sequentially per auth-setup.md:

1. **ddtool staging:** `ssh -A workspace-<name> "ddtool auth login --datacenter us1.staging.dog 2>&1"`
2. **ddtool ddbuild:** `ssh -A workspace-<name> "ddtool auth login --datacenter us1.ddbuild.io 2>&1"`
3. **GitHub on workspace:** `ssh -A workspace-<name> "gh auth status 2>&1"` (login if needed)
4. **SSH forwarding:** `ssh -A workspace-<name> "ssh-add -l 2>&1"`

For each command, monitor output for OIDC patterns per the OIDC Device-Code Surfacing Protocol
in auth-setup.md. When a device code or browser URL appears, surface it immediately via
AskUserQuestion and wait for user confirmation before proceeding to the next auth command.

If all checks pass, proceed to 3f.
If any check fails after retry, warn the user but proceed -- auth can be fixed later.

### 3f: Start Tmux Session with Claude

When the background `workspaces create` task completes successfully,
SSH into the workspace, cd into the repo, and start a detached tmux session running Claude Code:

```bash
ssh -A workspace-<name> "cd /workspaces/<repo> && git checkout <branch-name> && tmux new-session -d -s main -c /workspaces/<repo> claude"
```

The workspace is created with `--branch`, so the checkout is a no-op in the happy path
but ensures correctness if the workspace defaulted to a different branch.

If SSH fails, run `workspaces ssh-config <name>` first and retry.

Then print the join command for the user:

```
Workspace "<name>" is ready on branch "<branch-name>".

Join the session with Claude open:
  ssh -A workspace-<name> -t "tmux new-session -A -s main"

iTerm2 users can add -CC for native window integration:
  ssh -A workspace-<name> -t "tmux -CC new-session -A -s main"

Other commands:
  IDE:     workspaces connect <name> --editor intellij
  Status:  workspaces list
  Delete:  workspaces delete <name>

The workspace will be garbage collected after 20 days of inactivity
and has a hard TTL of 6 months.
```

## Step 4: List Workspaces

```bash
workspaces list
```

Report the list including workspace names, status, and expiration dates.

## Step 5: Delete Workspace

If name not provided, run `workspaces list` first and ask user to pick.

Confirm before deleting:

```
AskUserQuestion(
  header: "Delete workspace",
  question: "Delete workspace '<name>'? This removes everything including the home directory.",
  options: [
    "Yes, delete" -- Permanently delete the workspace,
    "Cancel" -- Keep the workspace
  ]
)
```

If confirmed:

```bash
workspaces delete <name>
```

## Step 6: SSH into Workspace

### 6a: Pre-SSH Auth Check

Run the pre-flight checks from [references/auth-setup.md](references/auth-setup.md):

1. Verify SSH agent has keys (`ssh-add -l`)
2. If no keys, offer fix before proceeding

### 6b: Connect

```bash
ssh -A workspace-<name>
```

The prefix `workspace-` is required. Tab completion works.
Use `-A` for agent forwarding so git operations work on the workspace.

If SSH fails:

```bash
workspaces ssh-config <name>
```

Then retry SSH.

### 6c: Post-Entry Auth Validation

After successful SSH, run a lightweight auth check on the workspace:

```bash
ssh -A workspace-<name> "gh auth status 2>&1 && ssh-add -l 2>&1"
```

If either fails:

```
AskUserQuestion(
  header: "Workspace credentials",
  question: "Some credentials on the workspace need refresh. Run full auth setup?",
  options: [
    "Yes" -- Run auth setup now (may require browser interaction),
    "Skip" -- Continue without refreshing
  ]
)
```

If "Yes", run the full Post-Creation Auth Setup (Step 3e procedures).

## Step 7: Connect IDE

```bash
workspaces connect <name> --editor <editor>
```

Default editor is `intellij` if not specified.

## Error Handling

| Error | Action |
|-------|--------|
| `workspaces` CLI not found | Install: `brew update && brew install datadog-workspaces` |
| Connection refused / timeout | Check Appgate VPN is connected |
| Auth error | Run `ddtool auth github login` |
| SSH connection refused | Run `workspaces ssh-config <name>` to fix SSH config |
| Workspace not found | Run `workspaces list` to show available workspaces |
| Create fails | Check Appgate VPN, GitHub auth, and instance type availability |
| SSH agent forwarding fails | Verify ssh-agent is running and keys are added (`ssh-add -l`). Suggest `eval $(ssh-agent) && ssh-add`. |
| Missing `workspace-` prefix in SSH | Use `ssh workspace-<name>`, not `ssh <name>` |
| OIDC device-code prompt | Surface URL and code immediately via AskUserQuestion. See [references/auth-setup.md](references/auth-setup.md). |

## Reference

For auth setup procedures (pre-flight checks, post-creation auth, OIDC surfacing), see [references/auth-setup.md](references/auth-setup.md).

For advanced configuration (secrets, instance types, IDE templates, Claude Code setup), see [references/advanced.md](references/advanced.md).
