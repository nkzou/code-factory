# Workspace Auth Setup

Detailed auth validation and setup procedures for Datadog workspaces.
Referenced from SKILL.md Steps 2b, 3e, and 6c.

## Pre-Flight Auth Checks (Local)

Run on the LOCAL machine before creating or entering a workspace.

### SSH Agent Validation

Before any SSH with `-A` (agent forwarding):

```bash
ssh-add -l 2>&1
```

| Result | Action |
|--------|--------|
| Keys listed | Proceed |
| `Could not open...` or empty | Surface fix via AskUserQuestion |

Fix:

```bash
eval $(ssh-agent) && ssh-add
```

After fix, re-check `ssh-add -l` to confirm keys are present.
Block until resolved -- SSH agent forwarding is required for git operations on the workspace.

### GitHub Auth Validation

```bash
ddtool auth github status 2>&1
```

| Result | Action |
|--------|--------|
| Authenticated | Proceed |
| Expired or missing | Run `ddtool auth github login` (may trigger OIDC -- handle per Surfacing Protocol below) |

## Post-Creation Auth Setup (Remote via SSH)

After workspace creation completes and SSH is available,
run these checks ON THE WORKSPACE via SSH.
Each may trigger an OIDC device-code flow requiring user browser interaction.

Run sequentially -- never in parallel. Each OIDC flow requires the user's attention.

### R1: ddtool Staging Auth

```bash
ssh -A workspace-<name> "ddtool auth login --datacenter us1.staging.dog 2>&1"
```

Monitor output for OIDC patterns (see Surfacing Protocol).
If device code appears, surface immediately and wait for user confirmation.

### R2: ddtool Ddbuild Auth

```bash
ssh -A workspace-<name> "ddtool auth login --datacenter us1.ddbuild.io 2>&1"
```

Same OIDC handling as R1.

### R3: GitHub Auth on Workspace

```bash
ssh -A workspace-<name> "gh auth status 2>&1"
```

If not authenticated:

```bash
ssh -A workspace-<name> "gh auth login --hostname github.com --git-protocol https --web 2>&1"
```

This produces a device code. Surface immediately.

### R4: SSH Agent Forwarding Verification

```bash
ssh -A workspace-<name> "ssh-add -l 2>&1"
```

| Result | Action |
|--------|--------|
| Keys listed | Proceed |
| No keys | Warn: "SSH agent forwarding is not working. Ensure ssh-agent is running locally with keys loaded." |

Non-blocking if R1-R3 passed -- the user can fix this later.

### R5: DD_API_KEY Check

```bash
ssh -A workspace-<name> "echo \$DD_API_KEY | head -c4 2>&1"
```

Informational only. If empty, warn but do not block.

## OIDC Device-Code Surfacing Protocol

When running any auth command (locally or via SSH),
the command may produce an OIDC device-code or browser-auth prompt.
Detect and surface these immediately -- codes expire in minutes.

### Detection Patterns

After running an auth command, scan output for:

| Pattern | Example |
|---------|---------|
| `enter code` or `one-time code` | `When prompted, enter code BXYZ-LMNO` |
| `Open the following link` or `open.*browser` | URL on next/same line |
| `github.com/login/device` | GitHub device flow |
| `google.com/device` | Google device flow |
| Device code format | `XXXX-XXXX`, `XXX-XXX-XXXX` |
| Bare URL near auth keywords | `https://...` near login/authenticate/device/verify |

### Surfacing

When detected, present immediately:

```
AskUserQuestion(
  header: "Auth required",
  question: "An auth flow needs your browser.\n\nOpen: <URL>\nCode: <DEVICE_CODE>\n\nComplete the login in your browser, then confirm here.",
  options: [
    "Done" -- I completed the authentication,
    "Retry" -- Run the auth command again,
    "Skip" -- Continue without this auth (may cause failures later)
  ]
)
```

### Post-Confirmation Verification

After "Done", re-run the status check for that auth type to verify success:

| Auth type | Verification command |
|-----------|---------------------|
| ddtool datacenter | `ssh -A workspace-<name> "ddtool auth status --datacenter <dc> 2>&1"` |
| GitHub | `ssh -A workspace-<name> "gh auth status 2>&1"` |
| GitHub (local) | `ddtool auth github status 2>&1` |

If still failing, offer retry (max 3 attempts before moving on with a warning).

## Auth on Entry (Lightweight)

For SSH to an existing workspace (not freshly created), run a lighter check:

1. **Local:** SSH agent validation (same as pre-flight)
2. **Remote (after SSH succeeds):**

```bash
ssh -A workspace-<name> "gh auth status 2>&1 && ssh-add -l 2>&1"
```

3. If either fails:

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

If "Yes", run the full Post-Creation Auth Setup (R1 through R5) above.
