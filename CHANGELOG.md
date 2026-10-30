# Changelog

Every change, why it was made, and what it breaks. Newest first.

## 2.0.0 — unreleased (2026-10-30)

**Breaking:** `trivy` is now a required engine. Without it the script exits `1` and
refuses to scan, the same as any other missing engine. Verdicts can also change: a
repo that previously exited `0` or `4` now exits `2` if its lockfile pins a version
with a HIGH or CRITICAL CVE.

### Added
- **Tier 2b — known vulnerabilities (Trivy).** Reads lockfiles for every ecosystem
  Trivy supports, lists the worst 30 CVEs with the fixed version, and sets exit `2`
  on any HIGH/CRITICAL. Closes the gap where Gate 1 had no CVE check at all:
  `mvschwarz-openrig` passed v1.0.0 carrying two critical CVEs; v2.0.0 reports
  CRITICAL=2 HIGH=29 on the same clone.
- Self-suppression guard for Trivy, matching the gitleaks one: it runs from an empty
  temp directory with `--config ""` and `--ignorefile ""`, so a scanned repo cannot
  ship a `trivy.yaml` or `.trivyignore` that hides its own CVEs.
- `trivy` row in `--check-engines`, and in the engine versions stamped on every report.
- README: Troubleshooting table of problems people actually hit; `SBP_OUTPUT_DIR` and
  `SBP_REVIEW_MODEL` documented.
- CI installs a pinned Trivy release.

### Changed
- Version string defined once (`SBP_VERSION`) instead of three hard-coded copies.
- Tier 2 note no longer says Go/Rust/Ruby are unchecked; Tier 2b covers their lockfiles.

### Not changed (on purpose)
- CVE findings are not routed to the Tier 3 file list. A model reading a lockfile
  answers nothing; the decision is "upgrade or decline", which the report states.
- Trivy runs `--scanners vuln` only. Its secret scanner duplicates gitleaks, and IaC
  misconfig answers "is this safe to deploy", not "is this safe to install".

### Verified (this machine, before commit)
- `tests/fixtures/benign` → exit 0; `tests/fixtures/suspicious` → exit 4 (unchanged).
- `mvschwarz-openrig` → exit 2, CRITICAL=2 HIGH=29 MEDIUM=52 LOW=7.
- `bash -n` passes. shellcheck runs in CI only (not installed locally).

## 1.0.0 — 2026-10-26

First public release. Tiered ladder (triage → two engines + gitleaks + unicode sweep +
surface grep → dependency list → cheap-model review → main-model review), fail-closed
preflight, sub-HIGH findings exit `4`. History before this lives in `STATUS.md`.
