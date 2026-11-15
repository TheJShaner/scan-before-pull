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

- `agents/repo-intake.md`: absolute script path, exit `4` described as the default
  (it still said "`--strict` only"), quarantine is the script's own `.quarantine/`,
  full path to `snapshot-claude.sh`, and a note that Socket catches malicious packages
  while Trivy only knows CVEs. Promoted to a live skill on Jonathan's machine, replacing
  the retired `safe-install`. skill-forge validator: 100/100, 0 issues.
- README Tier 3 example: the old one read a `flagged-files.txt` that no longer exists.
  Now NUL-safe stdin, read-only sandbox, verdict written to a file.
- STATUS: stale "repo is private" lines corrected.

### Fixed (2026-11-15, Claude)
- **skillspector results no longer vanish on multi-skill repos.** skillspector prints
  "Warning: Found N skills in this directory. Use --recursive ..." on STDOUT ahead of
  its JSON. The raw output failed `jq`, so Tier 1b reported "ENGINE ERROR — UNKNOWN"
  and exit `3` while the engine had in fact scored the repo. Caught on
  `Jakeschincariol/day-trader-skill` (11 skills): reported UNKNOWN, real result
  100/100 CRITICAL, 34 issues. New `spector_json()` drops everything before the JSON
  object; both call sites (verdict and issue detail) use it. Same repo now exits `2`.
  Verified: `benign` → exit 0 (unchanged); `suspicious` → exit 2 both before and after
  this change (skill-scanner 2.3.2 now rates it HIGH — the "exit 4" below is stale).

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
