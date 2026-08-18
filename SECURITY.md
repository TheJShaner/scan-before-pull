# Security Policy

## Reporting a vulnerability

**Do not open a public issue for a security problem in this tool.**

This is a scanner. A public issue describing a way to make it report CLEAN when it
should not is a working evasion, handed to everyone reading, before there is a fix.

Report privately through GitHub's [private vulnerability
reporting](https://github.com/TheJShaner/scan-before-pull/security/advisories/new).
It goes only to the maintainer, and the report stays private until a fix ships.

Expect a first response within a week. This is maintained by one person, not a
security team — please size your expectations accordingly, and say so in the report
if you have a disclosure deadline.

## What counts as a vulnerability here

The threat model is narrow and specific. **The most serious bug this tool can have is
reporting CLEAN when it should not.** Anything in that class is in scope:

- A scanned repo suppressing its own findings (config files, allowlists, ignore
  directives, crafted filenames, encoding tricks).
- An error path that fails toward "clean" rather than "unknown" — a swallowed exit
  status, an unparsed result read as zero findings, a check that silently no-ops.
- A check that appears to run but covers nothing, while the report implies coverage.
- Findings that never reach the flagged list, so escalation is skipped.
- Anything that lets a scanned repo affect the machine doing the scanning: code
  execution during clone, writes outside the quarantine, path traversal.
- Content from a scanned repo reaching a model as instructions rather than data.

Also in scope, lower severity:

- Crashes or hangs triggered by a crafted repo (denial of scanning).
- Misleading report output that would reasonably cause a person to install something
  they should not have.

## Not vulnerabilities

- **False positives.** The engines are noisy by design and the README documents known
  patterns. A high score on benign code is expected behavior, not a bug.
- **Evasion of the underlying engines' static rules.** A determined attacker who
  studies NVIDIA SkillSpector or Cisco's rule corpus can write code that avoids them.
  That is inherent to static analysis, and belongs upstream — report it to those
  projects. What is in scope is this tool mishandling what the engines *did* find.
- **"The tool did not catch malware X."** Unless the reason is a defect in this
  orchestration layer rather than a gap in the engines' detection.

## Scope

This repository only: `scan-before-pull.sh` and the agent skill file.

Vulnerabilities in the orchestrated tools go to their own maintainers:
[SkillSpector](https://github.com/NVIDIA/skillspector),
[gitleaks](https://github.com/gitleaks/gitleaks),
[agent-threat-rules](https://github.com/Agent-Threat-Rule/agent-threat-rules),
and Cisco AI Skill Scanner via PyPI.

## For contributors

Pull requests to this repository are run through this tool's own ladder before merge —
including Tier 3 review of every changed file. A PR to a security scanner is code
aimed at the thing that judges other code, which makes it the highest-value target in
the project. Expect the review, and please do not take it personally.
