# STATUS — scan-before-pull

**As of 2026-08-17.** Repo is **PRIVATE** at https://github.com/TheJShaner/scan-before-pull.
Local copy at `~/HQ/DEV/ACTIVE/tools/scan-before-pull/` **is** the git repo — edits here
are edits to the tool. Commit and push; do not fork a private copy elsewhere.

## Why private

It was public for ~20 minutes (0 stars, 0 forks — nothing propagated). Pulled back
because `v1.0.0` was tagged and then its exit-code semantics changed: sub-HIGH findings
now exit `4` instead of `0`. Cleaner to fix unreleased than to ship a correction to a
tag that existed for half an hour.

## Verified working

Tested on this machine, real runs, not assumed:

- Ladder end to end on `AgriciDaniel/secretary`: 27 skills scanned, both engines
  CRITICAL, 25 files routed to Tier 3, exit 2.
- `--installed` sweep: 411 targets, 49 at MEDIUM+ (14 CRITICAL, 7 HIGH).
- Fixtures: `benign` → exit 0, `suspicious` → exit 4, `--advisory` → exit 0.
- Refuses to `rm -rf` a caller-supplied destination (test dir with data survived).
- Preflight refuses to run when an engine is missing.
- Stale quarantine clone removed and re-cloned fresh on re-run.

- **CI is green** (run 32091851323 → fixed in the next push; all three jobs pass):
  shellcheck clean, the no-engines job confirms it refuses rather than partially
  scanning, and the fixture jobs confirm findings always reach the flagged list.
  CI caught two real defects on its first run — a 404ing gitleaks install URL and
  two dead variables — which is the whole reason it exists.
- Linux path now exercised on every push (ubuntu-latest), not just macOS.

## NOT verified

- Portability fallbacks for hosts without PCRE (`perl`/`python` unicode sweep) and
  without `grep --null`: coded, but neither CI nor this Mac takes those branches.
  Both runners have GNU grep with `-P` and `--null`.
- `--installed` sweep is not covered by CI (no skill dirs on a runner).

## Open items

1. ~~Run CI~~ — **done, green.**
2. **Re-tag.** `v1.0.0` still points at the old exit semantics (sub-HIGH returned 0).
   Nobody consumed it — 0 stars, 0 forks — so either delete and re-cut it, or tag
   `v1.1.0`. Must happen before public.
3. **Go public again** when 2 is done. Jonathan has already approved publishing —
   this is a timing hold, not a reversal.
7. **Bump `actions/checkout@v4` → `v5`.** Node 20 deprecation warning on every run;
   currently forced onto Node 24. Not breaking yet.
4. **Enable private vulnerability reporting** in repo settings. `SECURITY.md` already
   points at it, so today it links to a feature that is off. Settings → Security.
5. **Triage the sweep.** 14 CRITICAL / 7 HIGH installed skills in
   `~/HQ/DEV/ACTIVE/tools/INSTALLED-SKILL-SWEEP-20260817.md`. Almost certainly mostly
   false positives (the top scorer is our own `cybersecurity` skill — 100/100, 101
   issues, because it is made of attack patterns). Read-the-line work, not rip-it-out
   work.
6. **Branch protection**, so a PR to a security scanner can't be merged without the
   ladder run on it.

## Known-good facts worth not re-deriving

- Our `skill-scanner` is **`cisco-ai-skill-scanner`** (Cisco AI Defense), running the
  MIT `agent-threat-rules` corpus. Not the PyPI package literally named
  `skill-scanner`, which is a different, older project.
- `skillspector` **exits 1 when it finds things.** Never read its exit status as
  failure — validate the JSON instead.
- On ugrep (Homebrew, shadows system grep on this Mac) **`-Z` means fuzzy matching**,
  not NUL output. Always use `--null`.
- `{ ... } | tee f` runs the block in a subshell and silently discards every variable
  set inside it. This shipped in v2/v3 and made the script print "clean" regardless of
  findings for a week.
- Both engines are noisy on test directories. `env = process.env` reads as privilege
  escalation; test fixtures echoing prompts read as system-prompt leakage. Deliberately
  not suppressed — read, don't mute.

## Provenance of the current state

Reviewed adversarially by Codex (`codex exec`), prompted to find paths reporting CLEAN
when they should not. Nine findings; the two most serious were gitleaks honoring a
scanned repo's own allowlist, and `git clone` running the untrusted repo's
`.gitattributes` smudge filters before any check ran. Both fixed. Full review output
was in the session scratchpad and is not preserved — re-run it after CI changes.
