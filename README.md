# scan-before-pull

A tiered security scanner for AI agent skills, plugins, and repos — built around one
idea: **most scans should cost nothing.**

Every agent-skill scanner available today is single-tier. You point it at a repo, it
runs everything it has, and you read the output. When that scanner is an LLM-driven
multi-agent audit, "everything it has" can mean 100,000 tokens to answer a question
that was never subtle: *should I let this stranger's code run on my machine?*

This does it the other way around. Cheap deterministic checks run always. Expensive
model-driven checks run **only when the cheap ones flag or disagree.** In practice
that means the overwhelming majority of repos resolve for free, and your token budget
gets spent on the handful that actually earned the attention.

```bash
./scan-before-pull.sh owner/repo
```

---

## This is an agent-operated scanner

Say this plainly up front, because it changes how you should read everything below:
**this tool is designed to be run by an agent that then interprets the findings.**
It is not a pass/fail robot, and its output is not a verdict.

What the script does is bounded and honest: it runs deterministic engines, collects
what they say, routes the flagged files into a list, and tells you which questions
still need answering. What it cannot do is decide whether
`env = process.env` on line 85 is dependency injection or an exfiltration primitive.
That judgment is Tier 3 and Tier 4 — a model reading specific lines — and the entire
tiering exists to make that judgment *cheap enough to actually perform* rather than
skipped because a full audit costs 100k tokens.

Exit codes exist so it can be wired into CI, and they fail closed. But a scan that
ends at the exit code has skipped the part that does the actual work.

`agents/repo-intake.md` is the skill file that teaches an agent to drive it.

## The ladder

| Tier | Cost | What runs | Exit condition |
|---|---|---|---|
| **0** | free, <1s | triage — install hooks, network surface, size | inert by shape |
| **1** | free, ~2s | two independent engines + gitleaks + unicode sweep + surface grep | **both engines clean → done** |
| **2** | free | dependency list emitted for a supply-chain check | deps clean → done |
| **3** | cheap | a *cheaper model* reads only the flagged files | `CLEAN` verdict → done |
| **4** | ~3k tokens | your *main model* reads only the one file Tier 3 called out | human decision |

The saving is Tier 1's consensus rule. Two scanners built on **different rule corpora**
both returning clean is a substantially stronger signal than either alone, and it costs
exactly the same nothing. They escalate only on a flag or a disagreement.

Nothing here spawns an agent fleet. The worst case in the entire ladder is one cheap
model pass over a bounded file list, plus one file read by your main model.

## What it orchestrates

This is a coordination layer, not a detection engine. The detection is done by tools
built by people with far more threat data than you or I have:

| Tool | By | License | Role |
|---|---|---|---|
| [SkillSpector](https://github.com/NVIDIA/skillspector) | NVIDIA | Apache-2.0 | 69 patterns / 17 categories, YARA, OSV supply chain |
| [skill-scanner](https://pypi.org/project/cisco-ai-skill-scanner/) | Cisco AI Defense | see PyPI | behavioral dataflow analysis |
| [agent-threat-rules](https://github.com/Agent-Threat-Rule/agent-threat-rules) | community | MIT | the rule corpus behind skill-scanner — "Sigma, but for AI agents" |
| [gitleaks](https://github.com/gitleaks/gitleaks) | gitleaks | MIT | committed secrets |

Using **two vendor-maintained engines rather than one** is the entire point. NVIDIA and
Cisco built their rule sets independently. When both come back clean you have genuine
corroboration; when they disagree you have a specific, cheap question to escalate.

## Install

```bash
pip install cisco-ai-skill-scanner
uv tool install git+https://github.com/NVIDIA/skillspector.git
brew install gitleaks jq

git clone https://github.com/<you>/scan-before-pull.git
chmod +x scan-before-pull/scan-before-pull.sh
```

The script **refuses to run** if any engine is missing. A silently-absent scanner
reports "clean," which is the most dangerous failure mode a security tool has.

## Usage

```bash
# vet a repo before you clone it anywhere real (clones to quarantine, never installs)
./scan-before-pull.sh owner/repo

# vet a directory already on disk
./scan-before-pull.sh --path ./some-plugin

# sweep everything already installed
./scan-before-pull.sh --installed --min-severity HIGH

# CI: non-zero exit on HIGH/CRITICAL
./scan-before-pull.sh owner/repo --strict
```

Config via environment:

| Variable | Default | Purpose |
|---|---|---|
| `SBP_QUARANTINE` | `./.quarantine` | where repos are staged for scanning |
| `SBP_SKILL_DIRS` | `~/.claude/skills:~/.agents/skills` | dirs swept by `--installed` |

## Driving Tier 3 with a second CLI

Tier 3's job is to read a *bounded* file list with a model that isn't your expensive
one. The script prints a ready-to-paste command. With the Codex CLI:

```bash
codex exec "Security review these files from freshly-cloned repo owner/repo.
For each: does it make network calls, write outside its own tree, touch agent
config dirs or shell init, run at install time, or contain instructions aimed at
an AI agent rather than a human? Verdict per file: CLEAN / SUSPICIOUS / MALICIOUS
+ one line why. Files: $(cat 'repo-flagged-files.txt' | tr '\n' ' ')"
```

Any capable CLI works — Codex, Gemini, a local model, whatever is cheap for you. Two
properties matter and neither is about which vendor you picked:

1. It is **cheaper than your main model.**
2. It reads a **bounded file list**, not an unbounded repo.

`agents/repo-intake.md` in this repo is a drop-in skill file that teaches an agent to
run this ladder correctly — where the binaries live, what to ask at Tier 3, and which
findings are known noise. Adapt the frontmatter to your harness.

## Scores are not verdicts

The engines are noisy, and you need to know *how* before you trust a number.

Scanning [a governance tool for AI agents](https://github.com/AgriciDaniel/secretary)
produced **91/100 CRITICAL** from SkillSpector and 3 CRITICALs from skill-scanner. Both
engines, independently, screaming. Reading four actual lines took about 5,000 tokens
and found this:

| Flagged as | What it actually was |
|---|---|
| Privilege Escalation ×29 | `env = process.env` default parameters — dependency injection, and *good* practice |
| System Prompt Leakage ×2 | test fixtures echoing a prompt back so tests can assert on it |
| Prompt Injection | the project's own `<!-- owned-surface:v1 -->` provenance marker |
| CRITICAL shell execution ×3 | `spawnSync(` inside the test runner |

Entirely noise. The old flow would have spent 100k tokens reaching the same conclusion.

The dominant cause is that **both engines scan test directories**, where prompts,
fixtures, and shell calls live for legitimate reasons. This tool's own Tier 1d grep
excludes them — but the engines' findings are deliberately **not** suppressed, because
a real payload can hide in a test directory just fine. They get *read*, not muted.

Treat any score as a pointer to a line number, never as a verdict.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | both engines zero findings, nothing flagged |
| `1` | usage error, or a required engine is missing (refuses to run) |
| `2` | HIGH/CRITICAL findings — **always**, not only under `--strict` |
| `3` | scan incomplete: a check could not run, or output could not be written |
| `4` | `--strict` only: findings below HIGH |

`3` is the one people get wrong. An incomplete scan is not a pass. If the unicode
sweep could not run, or an engine returned nothing parseable, or a manifest failed
to parse, the result is **unknown** and the script says so.

## Threat model: the repo is scanning-hostile

The thing being scanned is untrusted, and it knows it is being scanned. Handled:

- **Self-suppression.** A repo shipping its own `.gitleaksignore`, `.gitleaks.toml`,
  or `gitleaks:allow` comments can silence every secret finding — the scanned artifact
  choosing its own verdict. Allowlists are ignored, and their presence is reported.
- **Execution during clone.** `git clone` checks files out, which runs the repo's own
  `.gitattributes` smudge filters (Git LFS most commonly) using *your* git config —
  code execution and network access before a single check has run. Cloning is done
  with filters, hooks, and templates disabled.
- **Hostile filenames.** Paths from the repo flow into reports and into the Tier 3
  prompt. They are stored NUL-delimited (a newline in a filename must not evict the
  real target from a line-oriented list) and escaped before rendering, because a
  filename is untrusted text that a model will later read.
- **Poisoning the diagnostic channel.** Library warnings used to be stripped from the
  engine's table output with `grep -v`. That filter also deletes any *finding* row
  containing those strings — and the attacker picks the strings. Verdicts now come
  from JSON; results and diagnostics never share a channel.
- **Destructive paths.** The scanner never `rm -rf`s a caller-supplied destination.
  Only a directory it created inside the quarantine root is removed.
- **Zero coverage.** An engine reporting "0 skills scanned" is not a clean result.
  That is treated as a gap.

Not handled: a determined attacker who knows these engines can evade their static
rules. This raises the floor; it does not make you safe.

## Field notes

Bugs found building this, kept here because they generalize to anyone writing a
scanner in bash:

**`grep -Z` does not mean NUL-output everywhere.** On GNU grep it does. On **ugrep**
— common on macOS via Homebrew, and often shadowing the system grep — `-Z` means
**fuzzy matching**. Using the short flag would have silently switched the scanner to
approximate matching and manufactured false positives. Probe `--null` explicitly.

**A scanner exiting non-zero usually means it found something**, not that it failed.
skillspector exits `1` on findings. Treating that as an engine failure flipped a
confirmed CRITICAL into "unknown". Validate the *output*, not the status.

**`{ ... } | tee report.md` silently discards your results.** The block runs in a
subshell, so every variable set inside it — the failure flag, the finding count — is
gone by the time the verdict prints. This version of the bug shipped for a week and
printed a confident `✓ clean` on scans that had found HIGH severity issues. A scanner
that can only say "clean" is worse than no scanner. Write to a file and `cat` it.

**`grep -c . file || echo 0` emits `0\n0`.** `grep -c` prints `0` *and* exits non-zero
when it matches nothing, so the `||` fires on top of the output. Downstream arithmetic
then dies with a syntax error, and if that arithmetic gated a check, the check silently
skipped.

**`grep -P` doesn't exist on BSD.** The invisible-unicode sweep needs PCRE. On stock
macOS it fails, prints nothing, and reads as clean. This script detects PCRE support
and falls back to perl, then python — and if none exist it reports the sweep as
**SKIPPED**, loudly, rather than passing.

**`mapfile` doesn't exist in bash 3.2**, which is what macOS ships and what
`#!/usr/bin/env bash` resolves to there.

The theme: every one of these failed *toward* "clean." When you write security
tooling, audit your error paths for which direction they fail in.

## Limitations

- Static analysis only by default. A determined attacker who knows these engines can
  evade them; this raises the floor, it does not make you safe.
- Tier 3/4 quality depends entirely on the model you point at it.
- Only `npm`, `PyPI` (`requirements.txt`, `pyproject.toml`) manifests are parsed for
  Tier 2. Go, Rust, Ruby, and others are detected by the engines but not extracted.
- The supply-chain check is a handoff, not an integration — the script emits a list
  and expects you to run it through Socket, OSV, or deps.dev.

## License

MIT — see [LICENSE](LICENSE).

The orchestrated tools carry their own licenses (linked above) and are neither
vendored nor redistributed here; this script calls them as installed binaries.

## Credits

Built by [Jonathan Shaner](https://github.com/TheJShaner) with Claude (Anthropic),
out of a working repo-intake process: quarantine → scan → move → install with a
config-diff tripwire. The tiering exists because the original process kept reaching
for a 100k-token multi-agent audit to answer questions that four lines of `sed` could
settle.

Before release the script was put through its own Tier 3 — an adversarial review by
a second model (OpenAI Codex), prompted specifically to find paths that report CLEAN
when they should not. It found nine, including the gitleaks self-suppression hole and
the smudge-filter execution during clone. Both are fixed above and neither would have
been caught by the author reviewing his own work, which is the entire argument for
the tier existing.
