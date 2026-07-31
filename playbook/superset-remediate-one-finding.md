# Playbook: Remediate ONE scan finding (aayushipatel01/superset)

**Playbook name:** `superset-remediate-one-finding`

**Input:** a single GitHub issue URL in `aayushipatel01/superset` labeled `devin-fix`,
containing exactly one finding (its `stable_finding_key`, `scan_command_that_surfaced_it`,
and `acceptance_check`).

**Hard rules:** fix exactly ONE finding. Open exactly ONE PR. Do not batch findings, do
not opportunistically upgrade neighbouring packages, do not refactor.

**No privileged findings.** Every issue reaching this playbook was selected by the
orchestrator purely on severity rank. There is no pinned, curated, or demo finding, and
no finding gets special handling because of its category or which scanner produced it.
Treat a bandit source finding and a dependency CVE identically: reproduce, minimally fix,
prove with before/after scan output.

---

## Steps

1. **Read the linked issue.** Extract `stable_finding_key`, `category`,
   `package_or_file`, `scan_command_that_surfaced_it`, and `acceptance_check`. If the
   issue is already closed or has a linked open PR, stop immediately and report
   `status: "skipped_duplicate"` — do not open a second PR.

2. **Prepare the workspace.** Work on the fork's default branch, then
   `git checkout -b devin/$(date +%s)-fix-<short-slug>`. Do not commit anything yet.

3. **Reproduce the finding.** Run the issue's `scan_command_that_surfaced_it` verbatim
   and capture the raw output. Confirm the `acceptance_check` line is present.
   - If it is **not** present, the finding has already been fixed or the advisory feed
     moved. Stop, comment on the issue with the scan output, close it as stale, and
     report `status: "not_reproducible"`.
   - Save the captured output as `scan_before` (trim to the relevant lines plus the
     summary counts; do not paste thousands of lines).

4. **Apply the minimal fix.**
   - `static_security`: change only the flagged line(s). For a B324 MD5 flag, for example,
     that means adding `usedforsecurity=False` to the `hashlib` call — do not restructure
     the function, do not swap the hash algorithm (it would invalidate existing cache keys).
   - `dependency_vuln` / `outdated_dep` (Python): edit the constraint in **`pyproject.toml`**,
     raising the lower bound, then regenerate the lockfiles with `uv`. **Never** hand-edit
     `requirements/*.txt`.
   - `dependency_vuln` / `outdated_dep` (JS): detect the package manager by lockfile
     (`package-lock.json` → npm, `yarn.lock` → yarn; `docs/` is yarn), bump only the one
     package, and commit the regenerated lockfile. Prefer a version published >= 7 days ago.
   - Never touch tests to make a scan pass. Never add a blanket suppression in place of a
     real fix unless the issue explicitly says the finding is a false positive.

5. **Re-run the scan and capture after.** Run the *identical* `scan_command_that_surfaced_it`
   again. The `acceptance_check` line MUST be gone. Save as `scan_after`.
   If it is still present, iterate on the fix; after 3 failed attempts, comment the
   evidence on the issue and report `status: "failed"` rather than forcing a workaround.

6. **Run the repo's own verification.** `git add` the changes, then
   `pre-commit run --all-files`. It must pass. Re-stage and re-run after any auto-fix.
   For dependency changes also run the relevant test suite (`pytest tests/unit_tests/`
   for Python, `npm run test` in the affected JS package) to confirm nothing broke.

7. **Open exactly ONE PR.** Fetch `.github/PULL_REQUEST_TEMPLATE.md` first and follow it.
   Conventional-Commits title, e.g. `fix(security): add usedforsecurity=False to MD5 digest`
   or `chore(deps): bump mcp to >=1.28.1`. The body must:
   - contain `Closes #<issue-number>`;
   - contain the `stable_finding_key`;
   - paste `scan_before` and `scan_after` in fenced blocks under a **Before / After**
     heading, so the fix is verifiable without re-running anything.

8. **Watch CI.** Fix failures silently. Never claim a failure is pre-existing or flaky
   without checking the same job on the base branch. After 3 unsuccessful attempts,
   escalate on the PR rather than continuing to push.

9. **Emit the structured output.** Before finishing, emit exactly this object:

```json
{
  "issue": "https://github.com/aayushipatel01/superset/issues/<number>",
  "pr_url": "https://github.com/aayushipatel01/superset/pull/<number>",
  "scan_before": "<verbatim trimmed scan output showing the acceptance_check line>",
  "scan_after": "<verbatim trimmed scan output with that line absent>",
  "status": "success | failed | skipped_duplicate | not_reproducible"
}
```

`status` is `success` only when the PR is open, `acceptance_check` is gone from
`scan_after`, and `pre-commit run --all-files` passed. If `pr_url` could not be created,
set it to `null` and explain why in the final message.
