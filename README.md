# devsecops-pipelines

**Reusable GitHub Actions security pipelines.** Four gates, written once, called
from every repository.

```yaml
# .github/workflows/security.yml in any Node repo
name: Security
on:
  push: { branches: [main] }
  pull_request:
  schedule: [{ cron: "17 6 * * 1" }]

permissions:
  contents: read

jobs:
  security:
    uses: Noirmiles/devsecops-pipelines/.github/workflows/security-node.yml@main
    permissions:
      contents: read
      security-events: write
```

That is the whole integration. Ten lines per repo, and changing the Trivy
severity threshold or bumping the pinned gitleaks version updates every
repository at once instead of requiring fifteen identical pull requests.

## The gates

| Gate | Tool | Catches |
|---|---|---|
| `secrets` | gitleaks (full history) | credentials committed by mistake |
| `deps` | npm audit / pip-audit | known CVEs in code we imported |
| `sast` | Semgrep | flaws in code we wrote |
| `scan` | **Trivy** | vulnerabilities, secrets, misconfiguration |

**They are separate jobs, not steps in one job.** A Semgrep failure still tells
you whether your dependencies are clean. A single combined job stops at the
first failure, hides the other three results, and turns one fix into three
sequential pipeline runs.

## Inputs

Both workflows accept:

| Input | Default | Notes |
|---|---|---|
| `working-directory` | `.` | for monorepos, e.g. `apps/web` |
| `trivy-severity` | `CRITICAL,HIGH` | severities Trivy blocks on |
| `semgrep-config` | language ruleset | space-separated Semgrep rulesets |
| `blocking` | `true` | `false` reports findings without failing the build |

`security-node.yml` adds `node-version` (default `20`) and `npm-audit-level`
(default `high`). `security-python.yml` adds `python-version` (default `3.12`)
and `install-command`.

Each gate can also be turned off individually — `enable-secrets`, `enable-deps`,
`enable-sast`, `enable-scan`, all defaulting to `true`. This exists for repos
that already run a gate of their own and only want the ones they are missing;
running two secret scanners over the same history is cost without signal.

```yaml
    with:
      enable-deps: false      # this repo already audits prod deps in its own CI
```

### Onboarding a repo with an existing backlog

Start with `blocking: false`, triage what it finds, then remove the line:

```yaml
    with:
      blocking: false
```

Turning four gates on across a mature codebase surfaces a backlog immediately.
Blocking from day one means every repo goes red at once, which is how teams
learn to ignore CI.

## Design notes

**Actions are pinned to commit SHAs, not tags.** `@v4` is a mutable pointer — a
promise from the maintainer not to change what it references. The SHA is the
only immutable reference GitHub offers.

**gitleaks is installed from its release archive with a verified SHA-256**,
rather than via `gitleaks-action`. The action derives a commit range from the
push event; when there is no valid range — a repository's first push, for
instance — it scans zero bytes, reports "no leaks found in partial scan", and
still exits non-zero. That is the worst failure mode a security gate can have:
indistinguishable from a real finding while having checked nothing.

**gitleaks runs with `fetch-depth: 0`.** A secret deleted from the working tree
is still live if it is reachable in an earlier commit. Scanning only the tip is
the most common way secret scanning gives false assurance.

**Semgrep uses explicit rulesets, never `--config=auto`.** Auto resolves rules
over the network at run time, so the checks applied to a given commit are
neither reproducible nor reviewable.

**Trivy runs twice, on purpose.** The first pass emits SARIF and never fails, so
findings reach the Security tab even when the gate trips. The second pass is the
gate. One pass cannot do both, because a non-zero exit skips the upload step.

**SARIF uploads are `continue-on-error`.** Private repositories need GitHub
Advanced Security for Code Scanning upload. The scan itself already failed the
job, so a failed upload must never mask a passing scan.

**Least privilege.** `permissions: contents: read` at the top; only the jobs that
upload SARIF request `security-events: write`.

### The GitHub-expression trap these files are written around

This looks like a ternary and is not:

```yaml
run: npm audit ${{ inputs.blocking && '' || '|| true' }}
```

An empty string is **falsy** in GitHub expressions. With `blocking: true`, the
`&&` branch evaluates to `''`, which is falsy, so evaluation falls through to
the `||` branch and appends `|| true` — silently disabling the gate it was meant
to enforce. The build reports green while checking nothing.

So every blocking decision inside a `run:` step is made in shell against an env
var instead:

```yaml
env:
  BLOCKING: ${{ inputs.blocking }}
run: |
  if [ "$BLOCKING" = "true" ]; then npm audit --audit-level=high
  else npm audit --audit-level=high || true; fi
```

The expression form survives only where both branches are non-empty strings and
there is no shell to be explicit in — the `exit-code:` inputs to `trivy-action`.

## Requirements

This repository is public, so any workflow may call it as-is. If you fork it and
keep the fork **private**, the fork has to opt in before other repositories can
call it:

```bash
gh api -X PUT repos/<owner>/<fork>/actions/permissions/access \
  -f access_level=user
```

## Adoption

Five repositories call these workflows today. The per-gate toggles exist because
adoption is rarely all-or-nothing — a repo whose own CI already runs `npm audit`,
or a monorepo that would otherwise be scanned twice, enables only the gates it is
actually missing.

| Repository | Workflow | Directory | Notes |
|---|---|---|---|
| Python + Next.js monorepo | python + node | `apps/core`, `apps/web` | two callers; `web` disables secrets + scan so the repo is not scanned twice |
| Next.js marketing site | node | `.` | all four gates |
| Next.js marketing site | node | `.` | all four gates |
| Next.js marketing site | node | `.` | secrets + deps disabled; covered by the repo's own `ci.yml` |
| Next.js marketing site | node | `.` | all four gates |

All callers block on CRITICAL/HIGH and expose `workflow_dispatch` for on-demand runs.
