## Residual Review Findings

Source: `ce-code-review` autofix run on branch `feat/automated-security-scans`
Run artifact: `/tmp/compound-engineering/ce-code-review/20260503-104259-279c3bc4/`
Plan: `docs/plans/2026-05-03-001-feat-automated-security-scans-plan.md`

The autofix pass applied two `safe_auto` fixes inline (CodeQL brace expansion, Trivy `@master` -> `@0.28.0`). The findings below are unresolved actionable work flagged by the multi-agent review (security, project-standards, correctness reviewers, run in parallel) that requires `downstream-resolver` attention before this set of workflows is promoted to required PR checks.

No issue tracker filing was attempted — recording inline as the durable sink because no GitHub PR exists yet for this branch.

### Pending findings

- **[P2] `manual` — `.github/workflows/workflow-lint.yml:42` — `pipx install zizmor` is unpinned**
  - Resolves to whatever the latest published zizmor wheel is at workflow run time. Same supply-chain hardening principle as SHA-pinning third-party Actions.
  - Fix: `pipx install zizmor==<version>` and add `pip` to `.github/dependabot.yml` so Dependabot bumps it. Pick the latest stable from PyPI when implementing.
  - Reviewer: security (P2, 75%), project-standards (low, 80%).

- **[P2] `manual` — Net-new third-party Actions pinned to major-version tags, not commit SHAs**
  - Affects: `codeql.yml` (`github/codeql-action/init@v3`, `analyze@v3`, `upload-sarif@v3`), `dependency-review.yml` (`actions/dependency-review-action@v4`), `gitleaks.yml` (`gitleaks/gitleaks-action@v2`), `scorecard.yml` (`ossf/scorecard-action@v2`, `github/codeql-action/upload-sarif@v3`), `trivy.yml` (`docker/build-push-action@v6`, `github/codeql-action/upload-sarif@v3`), `workflow-lint.yml` (`github/codeql-action/upload-sarif@v3`).
  - All flagged with inline `NOTE:` comments at introduction. Plan documents this as deferred (`docs/plans/2026-05-03-001-feat-automated-security-scans-plan.md` § Deferred to Follow-Up Work).
  - Resolve before promoting any of these workflows to a required PR check. The `ossf/scorecard-action` pin should be prioritized because it runs with `id-token: write` (OIDC issuance is in scope).
  - Reviewer: project-standards (medium, 90%), security (residual risk).

- **[P3] `manual` — `.github/workflows/workflow-lint.yml:36-46` — zizmor `continue-on-error` masks empty SARIF and double-runs the lint**
  - The current pattern runs `zizmor --format sarif . > zizmor.sarif` with `continue-on-error: true`, then runs `zizmor --min-severity high .` again to gate the job. Two issues: (1) if the first invocation crashes, the redirected file is empty/garbage and `upload-sarif` still runs, masking silent regression; (2) doubles wall time per PR.
  - Fix: emit SARIF once, then gate via `jq` on the SARIF (look for `level: "error"` results) instead of re-running zizmor.
  - Reviewer: security (P3, 75%), project-standards (low, 80%), correctness (low, 95%).

### Advisory items (informational, no action required)

- Dependency Review's `pull-requests: write` is downgraded to read-only on fork PRs (GitHub `GITHUB_TOKEN` policy), so the `comment-summary-in-pr: on-failure` summary will not post on external-contributor PRs. The check itself still runs and blocks. Do **not** switch to `pull_request_target` to fix this — it would expose write tokens to fork code.
- Trivy uses `exit-code: '0'` (record-only on `main`) by design — see plan § Key Technical Decisions.
- Scorecard badge in README will 404 until either the first weekly scheduled run lands (Monday 07:00 UTC) or `workflow_dispatch` is fired manually after merge.
- `actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1` was flagged by the correctness reviewer at 70% confidence as a non-existent major. The same pin is in active use in `.github/workflows/release-candidate.yml`, which suggests it resolves correctly; treating as a low-confidence false positive.

### Suggested next step

Open a follow-up issue tracking the three Pending findings together — they're all "harden the security workflows themselves" cleanup. Land before declaring any of these workflows a required PR status check.
