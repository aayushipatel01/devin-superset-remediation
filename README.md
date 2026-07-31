# Devin-Powered Dependency & Security Remediation System

An event-driven system that uses the Devin API to scan a target repository
for dependency vulnerabilities and static-security findings, file GitHub
issues, autonomously remediate them via Devin sessions, and record
observability metrics — with zero human intervention required for a
routine run.

**Target repository:** this system operates against
[aayushipatel01/superset](https://github.com/aayushipatel01/superset) — a
fork of Apache Superset. See that repo for the actual filed and remediated
issues, the merged fixes, and the live-running workflow and Automation.

## Demo

[Loom walkthrough](https://www.loom.com/share/80a56a03e2cb4186abc227be73d4fcca)

## Architecture

Two independent triggers spawn the same underlying orchestrator session:

- **Manual, on-demand:** a `workflow_dispatch` GitHub Action
  (`.github/workflows/devin-scan-orchestrator.yml`) that calls the Devin API
  directly to start a session with `trigger_source=manual`.
- **Scheduled, daily:** a native Devin Automation (Schedule trigger), running
  an equivalent copy of the same prompt with `trigger_source=scheduled`
  hardcoded, using Devin's own Secrets feature for credentials instead of
  GitHub Actions secrets.

Both triggers run the same logical pipeline:

1. **Scan** — runs `pip-audit`, `npm audit`, `yarn audit`, and `bandit` across
   every manifest in the target monorepo, with an OSV.dev fallback where
   `pip-audit` cannot resolve.
2. **Rank** — pools every finding from every scanner onto one normalized
   severity scale and takes the top 5, with a floor rule ensuring a run never
   goes empty-handed even if one scanner returns nothing.
3. **File** — opens one GitHub issue per new finding (`devin-fix` label),
   deduplicated by a stable finding key. A retry rule ensures a finding whose
   issue never produced a successful fix is retried on a later run rather
   than permanently skipped.
4. **Remediate** — spawns up to 5 parallel Devin sessions, each attached to
   the `superset-remediate-one-finding` playbook, each fixing exactly one
   finding: reproduce → apply the minimal fix → re-verify with the same scan
   command → run the target repo's own test/lint suite → open one PR with
   real before/after scan evidence.
5. **Record** — appends run- and session-level records to a committed,
   append-only metrics file in the target repo, which the dashboard reads
   directly.

### Key files in this repository

- `.github/workflows/devin-scan-orchestrator.yml` — the manual trigger
- `.github/devin/scan-orchestrator-prompt.md` — the manual-variant master
  orchestrator prompt (the scheduled variant is identical except for the
  `trigger_source` line, and lives in the Devin Automation's own
  configuration, not as a file)
- `playbook/superset-remediate-one-finding.md` — the remediation playbook
  every child session is attached to
- `dashboard/index.html` — a static, read-only observability dashboard over
  the target repo's committed metrics file
- `dashboard/Dockerfile` — packages the dashboard for local viewing

## Viewing the dashboard

```bash
cd dashboard
docker build -t remediation-dashboard .
docker run -p 8080:80 remediation-dashboard
```

Then open `http://localhost:8080`. (A live version is also hosted via GitHub
Pages directly on the target repo, at
`https://aayushipatel01.github.io/superset/`.)

## Triggering a real run

- **Manual:** in the target repo's **Actions** tab, run the "Devin scan
  orchestrator (manual)" workflow.
- **Scheduled:** fires automatically every day; no action needed.

## What "success" means here

The dashboard's success rate is computed from whether each remediation
session's own `status` is `success` — meaning the vulnerability was
independently re-scanned and confirmed gone — deliberately separate from
`ci_status`, which reflects whether *every* CI check passed, including
checks unrelated to the fix itself (e.g., pre-existing flaky tests or
unrelated lint failures). A run can be marked `partial` due to CI noise
even when every individual fix succeeded; the dashboard's drill-down view
makes this distinction visible rather than conflating the two.

## Known limitations / next steps

- **Merge contention:** parallel fixes touching the same shared manifest
  (e.g., a `package.json` `overrides` block) can produce merge conflicts
  when multiple PRs land close together. A production version would have
  the orchestrator serialize or auto-rebase these rather than surfacing
  conflicts to a human.
- **Network policy:** for this demo, the Devin session's network allowlist
  was opened broadly to move quickly during development. A production
  deployment would harden this back down to a precise, minimal allowlist
  before go-live.
- **Configurable policy per customer:** severity thresholds, which scanners
  apply, and issue-filing rules are currently hardcoded for this repository.
  A production version would make these configurable per target repository
  or customer.
- **Scope discipline over completeness:** one PR in the target repo is
  intentionally left unmerged — Devin correctly identified that fully
  resolving it would require touching test files unrelated to the original
  finding, which its own instructions explicitly forbid. Rather than
  silently expanding scope, it stopped and documented the tradeoff for a
  human decision. This is left as-is deliberately, as evidence of that
  guardrail working as intended.
