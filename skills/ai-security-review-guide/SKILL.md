---
name: ai-security-review-guide
description: Guide an AI-led source-code security review, threat model, finding triage, or review report. This is an instruction guide, not an executable scanner, penetration-testing tool, or security certification. Includes Android and Kotlin guidance. Use for reviewing code for vulnerabilities or assessing pending changes. Not for general code quality or Claude Code configuration reviews.
---

# AI Security Review Guide

Instructions for an AI-led source-code security review: **threat model → review
→ triage → report**, with **diff** mode for pending or branch changes.

This package contains review instructions and reference material. It has no
executable vulnerability scanner. The existing `scan` mode means AI-led
source inspection; it does not run an automated scan. Findings depend on
the evidence the reviewing AI inspects and the checks it actually performs.

Built by combining two Anthropic reference implementations with an Android/Kotlin
layer neither of them has. See `NOTICES.md` for attribution and licensing.

---

## ⛔ Hard rules — read before every run

These override the urge to produce a long, impressive findings list.

1. **A finding without `file:line` evidence is not a finding.** Delete it.
2. **Confidence floor is 0.7 for findings.** Below that, nothing becomes a
   finding. Better to miss a theoretical issue than to bury a real one in
   noise. In `scan` mode only, a 0.5 to 0.7 candidate may be carried as a
   **lead** instead, which is reported separately and never counted as a
   finding. See "Confidence" in `references/methodology.md`. `diff` mode has no
   lead tier.
3. **Never invent** a CVE number, an Android API level, a Play policy name, a
   library version, or a code section. If you did not check it this session,
   say so in the finding. See Provenance below.
4. **Never print a secret you find.** Report the file, the line, and the shape
   (`"a 32-char hex string assigned to ANALYTICS_API_KEY"`). Never the value. If a
   live key is found, say so immediately and tell the owner to rotate it.
5. **Read-only by default.** Scan, triage, and report never modify the target.
   Only `patch` mode writes code, and only with explicit approval per fix.
6. **Never execute the target code.** This skill reads and reasons. It does not
   build, run, install, or send network traffic.
7. **Do not grade on effort already spent.** A shipped app with a real hole is
   still a real hole.

---

## Provenance — every finding carries its basis

This mirrors the standing verification rule. Each finding gets one tag, and the
tag must survive into the report and any summary. A caveat at the bottom of a
document does not preserve it.

| Tag | Means | How to state it |
|---|---|---|
| `CODE-VERIFIED` | You read the source and confirmed the pattern. Cite `file:line`. | State as fact. |
| `INFERRED` | The pattern suggests it but the data flow was not fully traced. | "This looks like X, but I did not trace the input to confirm it is reachable." |
| `NEEDS-CHECK` | Depends on an external fact not checked this session (API level behaviour, library CVE, Play policy wording). | "This depends on <fact>, which I have not verified. Check before acting." |

A finding that would change severity based on an unchecked fact is `NEEDS-CHECK`,
not `CODE-VERIFIED`, no matter how confident the pattern looks.

**There are two tag sets, and they are not interchangeable.** They tag different
objects, so do not merge them or substitute one for the other.

| Object tagged | Tag set | Defined in |
|---|---|---|
| A **finding** in `FINDINGS.json` | `CODE-VERIFIED` · `INFERRED` · `NEEDS-CHECK` | this file |
| A **claim** in `THREAT_MODEL.md` | `[Code-verified]` · `[Owner-states]` · `[Unverified]` | `references/threat-model.md` |

`[Owner-states]` has no finding equivalent on purpose. A finding comes from
code, not from a person. If something the owner said is what leaves a finding
uncertain, the finding is `NEEDS-CHECK` and names that statement as the thing
to verify.

---

## Modes

Invoke with a mode and a target. If the user just says "audit my app", run
`scan` and offer `threat-model` first if no `THREAT_MODEL.md` exists.

| Mode | What it does | Writes |
|---|---|---|
| `threat-model <dir>` | Interview or derive what actually matters for this app. Cuts false positives more than any other single step. | `THREAT_MODEL.md` |
| `scan <dir>` | Whole-tree read-only vulnerability scan. Parallel subagents by focus area. | `FINDINGS.json` + `.md` |
| `triage <findings>` | Apply exclusions, dedupe, re-score severity against the threat model. | `TRIAGE.json` |
| `report <triage>` | Build the deliverables. | `.xlsx` log + `.docx` report |
| `diff [base]` | Review staged, unstaged, and untracked changes. With a base, also review committed changes from its merge base with HEAD. Follow methodology.md for scope and failure handling. | Inline findings |
| `patch <finding-id>` | Propose a fix. Requires explicit approval per fix. | Code edits |

**Load the matching reference file before doing the work.** Do not work from
memory of this table.

| Doing this | Read first |
|---|---|
| any scan or diff | `references/methodology.md` |
| deciding whether to report something | `references/exclusions.md` |
| any Android, Kotlin, or Gradle target | `references/android-kotlin.md` |
| `threat-model` mode | `references/threat-model.md` |
| `report` mode | `references/reporting.md` |

---

## The pipeline

```
threat-model ──→ scan ──→ triage ──→ report
                  ↑                      │
                  └── diff (standalone) ──┘
```

**Why threat-model first.** A scan without one flags everything equally. A scan
with one knows which data in this particular app is the crown jewel, and that a
crash bug in a settings screen is not the same class of problem as sensitive
personal data reaching logcat. This is the single highest-leverage
step, and it is the one most often skipped.

If the user wants to skip it, that is their call. Say once that findings will be
noisier without it, then proceed.

---

## Running a scan

1. **Orient.** Identify language, build system, frameworks, and entry points.
   For Android: read the manifest, the Gradle files, and the permission list
   before any source file.
2. **Load the rule packs.** `references/methodology.md` always;
   `references/android-kotlin.md` if the target is Android or Kotlin.
3. **Fan out.** Use `Task` subagents per focus area so each one reads deeply
   rather than skimming everything. Suggested split for Android:
   - manifest, components, permissions, exported surface
   - data at rest: storage, encryption, backup, keystore
   - network and crypto: TLS, pinning, cipher choice
   - WebView, intents, deep links
   - secrets, logging, PII leakage
   - build, signing, dependencies
4. **Collect.** Merge into `FINDINGS.json` per the schema in
   `references/methodology.md`.
5. **Triage before showing anything.** Raw scanner output is not a deliverable.

---

## What "done" means

Per the standing rule that no error is not the same as correct:

- Every reported finding has been opened in the actual file and the line
  confirmed. Not grepped, opened.
- The count in the summary equals the row count in the XLSX.
- Any `NEEDS-CHECK` finding names what was not checked.
- If a stage was skipped, the report says which and why. Silence about a skipped
  stage is a false report.

---

## Deliverables

Per the standing file rules, `.md` is never the final deliverable. `FINDINGS.json`
and `FINDINGS.md` are working files. What gets delivered is:

- `LEADER - Security findings log (dd-mm-yyyy).xlsx` — one row per finding
- `LEADER - Security audit report (dd-mm-yyyy).docx` — the written assessment

Leader is the project code. Write to the project tree first, mirror to backup
second, and open the project-tree copy rather than the backup. 12pt floor
everywhere, including the spreadsheet. See `references/reporting.md` for the full spec.

---

## Scope limits — say these plainly

This is a code review, not a penetration test and not a security certification.

- It reads source. It does not run the app, fuzz it, or test a live endpoint.
- It cannot prove the absence of a vulnerability. "No finding in category X"
  means the scan did not surface one, not that none exists.
- It does not replace a professional assessment before a regulated launch.
- Dependency CVEs require a live lookup. Without one, dependency findings are
  `NEEDS-CHECK`.

Never tell the user an app is "secure", "clean", or "safe to ship". Those are
conclusions this tool cannot reach. "No high-severity finding surfaced in the
areas scanned" is a statement about the scan, and it is the honest one.
