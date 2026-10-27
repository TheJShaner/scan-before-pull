# STATUS — scan-before-pull

**As of 2026-10-26.** Repo is **PUBLIC** at https://github.com/TheJShaner/scan-before-pull.
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

### Fixed 2026-08-24

- **Tier 1e no longer reads binaries as text.** Files were decoded UTF-8 with
  `errors='ignore'`, so random bytes became real codepoints — some landing on
  U+200B/200C/202E. Any repo carrying media reported "invisible/bidi unicode
  found," set `FAIL=1`, and exited `2` **while the report body still read "No
  HIGH+ findings from either engine."** The exit code contradicted the report.
  Caught on `InlitX/Obsidian-Dashboard-Gallery`: 5 "findings" were 4 mp4s and a
  JPEG. Now skips a file whose first 8KB contains NUL (the `grep -I` heuristic);
  `grep` branches got `-I`. Same repo now reports clean and exits `4`.
  Verified detection preserved: a real zero-width char in a `.md` is still
  caught, while identical bytes inside a `.mp4` are ignored.
- **Codex review (2026-08-24) found 5 issues in the same-day changes; all fixed.**
  The P1 was a regression introduced by the binary-skip fix itself: skipping any
  file with a NUL in its first 8KB meant **a crafted text file could silence
  Tier 1e with one byte**. Replaced with a strict utf-8 decode — NUL is valid
  utf-8, so text still decodes and still gets scanned, while real binaries raise
  UnicodeDecodeError and are skipped. `grep -I` has the same NUL flaw and cannot
  be patched, so python3 is now the PREFERRED sweep engine and grep the fallback
  (this also removes a branch listed under NOT verified). Verified: a file with
  an early NUL and a zero-width payload later is now caught; a real .mp4 is not.
  The four P2s — INCOMPLETE outranking STALE, an unreadable installed version
  counting as incomplete rather than stale, version stamping in `--installed`
  reports, and the truncation note no longer claiming all HIGH/CRITICAL survived
  a 40-row cap — are fixed and verified (simulated unreadable version exits 3).
- **Engine versions are recorded and checkable.** Nothing tracked which ruleset
  ran. skillspector shipped 2.9.3→2.9.6 in eight days; the 8/17 install was
  current that day and stale the next, silently, for a week. Reports now stamp
  all three versions, and `--check-engines` compares against upstream (exit 4
  stale, exit 3 on failed lookup — never "current" by default).
  Verified: `--check-engines` returns all-current at skill-scanner 2.0.13 /
  skillspector 2.9.6 / gitleaks 8.30.1.

- **CI is green** (run 32091851323 → fixed in the next push; all three jobs pass):
  shellcheck clean, the no-engines job confirms it refuses rather than partially
  scanning, and the fixture jobs confirm findings always reach the flagged list.
  CI caught two real defects on its first run — a 404ing gitleaks install URL and
  two dead variables — which is the whole reason it exists.
- Linux path now exercised on every push (ubuntu-latest), not just macOS.

### OPEN — pick this up on fresh eyes (2026-08-24)

**skillspector 2.9.6 escalates on binary files, and we cannot patch its engine.**
On `InlitX/Obsidian-Dashboard-Gallery` it now returns `74/100 HIGH` driven entirely
by `[HIGH] AE3 analysis-evasion` on four `.mp4`s and a JPEG. On **2.9.5 that repo had
no HIGH from either engine** — so tonight's engine update *introduced* these. Same
false-positive class Tier 1e was fixed for, one layer up.

Our Tier 1e now handles this correctly: binary hits are listed, routed to Tier 3, and
do NOT set FAIL. skillspector's do, so any media-carrying repo exits 2.

Decision still owed (annotate vs suppress). Recommendation: **annotate, never suppress** —
a payload can hide in a binary, and the README already argues findings get read, not
muted. Proposed: when every HIGH/CRITICAL lands on binary/media files, say so in the
Tier 1 consensus block so the verdict is legible rather than silently downgraded.

### DRAFTED, NOT FILED — upstream issue to NVIDIA/skillspector

Decision 2026-08-24: **file the issue, name this tool, do NOT link it, do NOT go public
yet.** The mention is not a pitch — it is the true account of how the bug surfaced, and
naming the tool is the whole value of saying it. One sentence, stated plainly, no ask. The link waits until the public gate below is met — pointing NVIDIA
engineers at a scanner that has yielded a P1 on two consecutive review passes is the
"fails before it is even public" outcome we are trying to avoid. Add the link as a
follow-up comment once public. The wording works read now or read later.

Repro (two files, deterministic):

```bash
mkdir -p /tmp/minimal
printf -- '---\nname: demo\ndescription: minimal repro\n---\n\n# Demo\n' > /tmp/minimal/SKILL.md
python3 -c "open('/tmp/minimal/screenshot.png','wb').write(b'\x89PNG\r\n\x1a\n'+bytes(range(256))*300)"
skillspector scan /tmp/minimal --no-llm --format json
# -> score 30, HIGH AE3 analysis-evasion on screenshot.png, MEDIUM AE4 on the same file
```

Draft body:

> **AE3/AE4 fire HIGH on any binary asset in a skill directory**
>
> A skill containing a single PNG screenshot produces `HIGH AE3 analysis-evasion` and
> `MEDIUM AE4`, both against the image itself.
>
> Minimal repro: a directory with a normal `SKILL.md` and one `.png` scores 30 with a
> HIGH finding. (commands above)
>
> Real-world: `InlitX/Obsidian-Dashboard-Gallery` (four `.mp4` and a JPEG under
> `images/`) scores **74/100 HIGH**, driven entirely by AE3/AE4 on media files. On
> **2.9.5 that same repo produced no HIGH from the engine**, so this looks like a 2.9.6
> regression rather than long-standing behaviour.
>
> Expected: documentation screenshots and demo videos should not read as analysis
> evasion. Entropy is indistinguishable between compressed media and obfuscated
> payloads, so file-type gating (or a severity downgrade for non-text assets) seems
> closer to intent than a HIGH.
>
> Found while building `scan-before-pull`, a tiered pre-install scanner that runs
> SkillSpector alongside skill-scanner and gitleaks and only escalates to a model when
> the free engines disagree. Happy to send a PR if you have a preferred shape for the fix.

Why an issue and not a PR: AE3 is their taxonomy. Whether the right fix is file-type
gating, a severity downgrade, or magic-byte detection is a design call they own, and an
outside PR to a detection ruleset risks fixing it against their intent. Offer the PR in
the issue instead.

Filing is an outward, public action under Jonathan's GitHub account — his to authorize.

## NOT verified

- Portability fallbacks for hosts without PCRE (`perl`/`python` unicode sweep) and
  without `grep --null`: coded, but neither CI nor this Mac takes those branches.
  Both runners have GNU grep with `-P` and `--null`.
- `--installed` sweep is not covered by CI (no skill dirs on a runner).

## Open items

Nothing is required of Jonathan to hold the current state. The repo is private and
stays private until someone changes it.

**DONE 2026-10-26 — repo is PUBLIC.** `v1.0.0` re-cut at 3e8b338 (exit-4 semantics),
private vulnerability reporting enabled via API, visibility flipped. Shared with
AgriciDaniel (offered a star). Items 1–2 below are history. Next: link the repo in the
NVIDIA/skillspector issue once filed.

**Before it could go public (both small):**

1. **Re-cut the tag.** `v1.0.0` points at the old exit semantics, where sub-HIGH
   findings returned 0. Current HEAD returns 4. Nobody consumed the tag — 0 stars,
   0 forks, ~20 minutes public — so delete and re-cut it rather than shipping a
   correction: `git tag -d v1.0.0 && git push origin :refs/tags/v1.0.0`, then
   re-tag at HEAD.
2. **Enable private vulnerability reporting.** Repo Settings → Security. `SECURITY.md`
   already tells reporters to use it, so right now it points at a switched-off
   feature. Requires a human in the web UI.

Then flip public: `gh repo edit TheJShaner/scan-before-pull --visibility public
--accept-visibility-change-consequences`. **Jonathan already approved publishing** —
this was a timing hold, not a reversal. Do not re-litigate it.

**Independent of publishing:**

3. **Triage the sweep.** 14 CRITICAL / 7 HIGH in
   `~/HQ/DEV/ACTIVE/tools/INSTALLED-SKILL-SWEEP-20260817.md`. Probably mostly false
   positives — the top scorer is our own `cybersecurity` skill at 100/100 with 101
   issues, because it is made of attack patterns. This is read-the-flagged-line work
   (Tier 3/4), not rip-it-out work. More useful now than at scan time, since engine
   findings route into the flagged list as of v1.0.0.
4. **Branch protection**, so a PR to a security scanner cannot be merged without the
   ladder run on it. `SECURITY.md` promises this; nothing enforces it yet.
5. **Bump `actions/checkout@v4` → `v5`.** Node 20 deprecation warning on every run;
   currently force-run on Node 24. Not breaking yet.
6. **Gate 1 has no CVE check — known-vulnerable deps pass silently (found 2026-09-24,
   Jonathan: "we will have to play with that later").** The dependency inventory only
   says "feed this to Socket/OSV"; nothing runs it. Proof: `mvschwarz-openrig`
   (downloaded 9/24) pins vitest 3.2.4 (GHSA-5xrq-8626-4rwp critical) and seroval 1.5.1
   (GHSA-mv8w-475r-vwqw critical, runtime via @tanstack/router-core) — its SCAN-REPORT
   never mentions either. Fix idea: run `socket scan create <dir> --report` (CLI already
   authed, org `jsv`) and/or OSV in Gate 1; surface critical/high CVEs in the report and
   exit code. Not started.

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
