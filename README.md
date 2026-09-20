# AI Security Review Guide

A Claude skill that guides an AI-led source-code security review, end to end:
**threat model, review, triage, report**, plus a **diff** mode for pending or
branch changes.

Language-agnostic core. Deep Android and Kotlin rule pack on top.

---

## Read this first: what it is not

This is an **instruction guide**. It is not an executable scanner, not a
penetration test, and not a security certification.

There is no binary here and nothing to run. The package tells a reviewing AI
how to read source code for vulnerabilities, what to ignore, how to score what
it finds, and how to write it up. Findings depend entirely on the evidence the
reviewing AI actually inspects.

The `scan` mode means AI-led source inspection. It does not launch an automated
scan.

It reads source. It does not run the app, fuzz it, or touch a live endpoint. It
cannot prove the absence of a vulnerability, and it does not replace a
professional assessment before a regulated launch.

**Status, stated plainly:** this is version 0.1.0.

It has been reviewed adversarially, and it has been used to complete two full
audits, both on real Android codebases, on 20 September 2026. One
produced four findings and the other six, none critical in either. In one of
them the review was given an exported launcher activity as a planted false
positive and correctly declined to report it.

What has **not** been exercised: `diff` mode and `patch` mode have never been
run against a real repository. `patch` is the only mode that writes code. Four
rules in the Android rule pack ship marked `NEEDS-CHECK` and want a live
documentation lookup before any finding rests on them: the API level at which
`RECEIVER_EXPORTED` is enforced, the applicability of `MODE_WORLD_READABLE` and
`MODE_WORLD_WRITEABLE`, v1-only signing exposure against `minSdk`, and any
dependency advisory.

Treat it as a method with two runs behind it, not as a track record.

---

## Why it exists

Most AI security reviews fail the same two ways.

**They flag everything.** Without a model of what the application actually
protects, a missing rate limit on a settings screen ranks alongside health
records written to logcat. The real finding drowns.

**They assert what they did not check.** A finding that depends on an Android
API level, a library CVE, or a store policy gets stated as fact when nobody
looked any of them up.

The guide is built against both. A threat model comes first so severity means
something. Every finding carries a provenance tag that survives into the
report, so a reader can tell what was read in the source from what was inferred
from what was never checked at all.

---

## The rules it enforces

1. **A finding without `file:line` evidence is not a finding.** Delete it.
2. **Confidence floor is 0.7.** Below that nothing becomes a finding. In `scan`
   mode only, a 0.5 to 0.7 candidate may be carried as a **lead**, reported
   separately and never counted as a finding. `diff` mode has no lead tier.
3. **Never invent** a CVE number, an API level, a policy name, a library
   version, or a code section.
4. **Never print a secret.** Report the file, the line, and the shape. Never
   the value. If a key looks live, say so immediately and say to rotate it.
5. **Read-only by default.** Only `patch` mode writes, and only with approval
   per fix.
6. **Never execute the target.**
7. **Do not grade on effort already spent.** A shipped app with a real hole
   still has a real hole.

And one rule about language: never call an application secure, clean, or safe
to ship. Those are conclusions this method cannot reach. "No finding at or
above the confidence floor surfaced in the areas listed" is a statement about
the review, and it is the honest one.

---

## Provenance

Every finding carries one tag, and the tag has to survive into the report. A
caveat at the bottom of a document does not preserve it, because the bottom of
a document is what gets skimmed past.

| Tag | Means |
|---|---|
| `CODE-VERIFIED` | The source was read and the pattern confirmed. Cites `file:line`. |
| `INFERRED` | The pattern suggests it, but the data flow was not traced to confirm reachability. |
| `NEEDS-CHECK` | Depends on an external fact not checked this session. Names the fact. |

Threat model claims use a **separate** set, `[Code-verified]` / `[Owner-states]`
/ `[Unverified]`, because they tag a different object. `[Owner-states]` has no
finding equivalent on purpose: a finding comes from code, not from a person.

---

## Modes

| Mode | Does | Writes |
|---|---|---|
| `threat-model <dir>` | Works out what actually matters for this application. Cuts false positives more than any other single step. | `THREAT_MODEL.md` |
| `scan <dir>` | Whole-tree read-only review. Parallel subagents per focus area. | `FINDINGS.json` and `.md` |
| `triage <findings>` | Applies exclusions, dedupes, re-scores against the threat model. | `TRIAGE.json` |
| `report <triage>` | Builds the deliverables. | `.xlsx` log and `.docx` report |
| `diff [base]` | Reviews staged, unstaged and untracked changes. With a base, also the committed changes from its merge base with HEAD. | Inline findings |
| `patch <finding-id>` | Proposes a fix. Explicit approval per fix. | Code edits |

```
threat-model --> scan --> triage --> report
                  ^                      |
                  +--- diff (standalone)-+
```

---

## What is inside

`skills/ai-secure-code-review-guide/SKILL.md` is a lean router. The weight sits in
`references/`, loaded on demand rather than all at once.

| File | Covers |
|---|---|
| `methodology.md` | The three-phase method (context, comparative, assessment), the six vulnerability categories, severity bands and the adjustment rule, the confidence scale and the lead tier, the findings JSON schema, and how `diff` scopes pending versus branch changes |
| `exclusions.md` | The noise filter. Eight hard exclusions dropped without reporting, four documented overrides where upstream is wrong for this package, soft judgment filters, and the requirement to record what was dropped |
| `android-kotlin.md` | Nine focus areas: manifest and exported components, intents and deep links and PendingIntent, WebView, data at rest, network and crypto, logging and leakage, auth and biometrics, build and signing and dependencies, and store policy. Plus grep starting points |
| `threat-model.md` | The four-question framework, the interview method, the eight required sections with their table schemas and enum sets, and how the scan consumes the result |
| `reporting.md` | The XLSX findings log and DOCX report specification, formatting floors, and the rule that provenance survives into the report |

---

## The one override worth knowing about

Upstream deliberately **excludes** secrets stored on disk from its findings, on
the stated basis that another process handles them.

If there is no such process, porting that exclusion verbatim silently disables
the check that matters most. So it is reversed here, and escalated.

Three other departures are documented in `exclusions.md`: intent redirection is
carved out of the open-redirect exclusion, memory safety is re-enabled for
bundled JNI and NDK code, and secrets in Markdown and other non-executed files
are carved out of the Markdown exclusion.

---

## Install

**Claude Code or Cowork, via the marketplace:**

This plugin is listed in the `dina-skills` marketplace. Add that catalog once
and this plugin, plus anything added to it later, becomes installable:

```
/plugin marketplace add dinaf2026-web/dina-skills
/plugin install ai-secure-code-review-guide@dina-skills
```

Or in the UI: Settings, then Add marketplace, then paste
`https://github.com/dinaf2026-web/dina-skills`, then Sync, then Install.

This repository is the plugin itself and is not a marketplace, so adding this
URL as a marketplace will not work.

**Manually:** copy `skills/ai-secure-code-review-guide/` into `~/.claude/skills/`.

No dependencies. There is nothing to build and nothing to run.

`report` mode builds its deliverables through the `xlsx` and `docx` skills. If
those are not available, the findings and the written assessment still exist as
`FINDINGS.json` and `TRIAGE.json`, and the formatting specification in
`reporting.md` says what to build them into.

---

## Quick start

```
threat-model ./my-app      # do this first; it is the step most often skipped
scan ./my-app
triage FINDINGS.json
report TRIAGE.json
```

For a pre-commit gate, skip the pipeline:

```
diff                       # staged, unstaged and untracked
diff main                  # plus committed changes since the merge base
```

If you skip the threat model, findings will be noisier. That is a real
tradeoff, not a scare tactic, and it is your call to make.

---

## Credits and license

This is a derivative work. It combines material from two Anthropic open-source
reference implementations with an Android and Kotlin rule pack and a reporting
layer that neither of them has.

- **[anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review)**,
  MIT, Copyright (c) 2025 Anthropic. Source of the three-phase method, the
  category list, the severity bands, the confidence scale and its 0.7 floor,
  the findings schema, and the hard exclusion rules.
- **[anthropics/defending-code-reference-harness](https://github.com/anthropics/defending-code-reference-harness)**,
  licensed under a **modified** Apache 2.0, Copyright 2026 Anthropic PBC.
  GitHub classifies that repository `NOASSERTION`, not `Apache-2.0`, because its
  LICENSE deviates from the canonical text, including a rewritten indemnification
  clause in section 9. Source of the threat model schema,
  the interview method, and the pipeline shape. Its four-question framework is
  credited upstream to Shostack, *The Four Question Framework for Threat
  Modeling* (2024). Its autonomous harness is **not** included.

Original material: the entire Android and Kotlin rule pack, the reporting
specification, the three-tag provenance model, and the hard rules.

`NOTICES.md` carries the full attribution, the reproduced MIT notice, the
statement of changes that Apache 2.0 section 4(b) requires, and a diff of the
three places the second upstream's license departs from canonical Apache 2.0.

Two license files ship and they are **not** interchangeable.
`LICENSE-upstream-defending-code-reference-harness.txt` is the text that
upstream actually granted and is the operative license for material taken from
it. `LICENSE-APACHE-2.0.txt` is the canonical text from apache.org, included
for comparison only.

Original material is MIT. Copyright 2026 Dina Farhat. See `LICENSE`.
