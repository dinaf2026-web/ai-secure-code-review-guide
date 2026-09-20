# Threat model

Modified September 20, 2026: Inline provenance and conditional risk assessment corrected.
Modified September 20, 2026: Severity adjustment made single-source, non-cumulative, and capped below CRITICAL.

Read this for `threat-model` mode. Adapted from the `/threat-model` skill in
`anthropics/defending-code-reference-harness` (Apache 2.0). See `../NOTICES.md`.

**Why bother.** A scan without a threat model flags everything at the same
weight. With one, the scan knows that for a cycle tracker the crown jewels are
symptom records at rest, and that a crash in a settings toggle is not the same
class of problem. This step cuts false positives more than any other.

Output: `THREAT_MODEL.md` in the target directory. Markdown so it can be read
and edited by hand, but **the headings and column orders below are a contract** —
the scan and triage stages parse them.

---

## Modes

- **`interview`** — the owner is here. Ask, listen, ground the answers in code.
- **`bootstrap`** — derive it from the code alone, plus git history.
- **`bootstrap-then-interview`** — derive first, then use the interview to close
  the open questions. Best when both the owner and the code are available, which
  is the normal case here.

---

## The four questions

Use this wording when introducing each phase. The phrasing is deliberate.

1. **What are we working on?**
2. **What can go wrong?**
3. **What are we going to do about it?**
4. **Did we do a good job?**

Framework: Shostack, *The Four Question Framework for Threat Modeling* (2024).

**Ask one thing at a time. Do not dump a questionnaire.** If a design doc or a
project brain document exists, read it first and summarize the system back in
four to six sentences, then ask "Is this right, and what did I miss?" That is
faster than asking the owner to describe it cold, and it surfaces drift between
the document and the code.

---

## Provenance

Keep each material claim's basis beside the claim in both working notes and
the final threat model, including tables and summaries:

- `[Code-verified]`: inspected source supports this specific claim; cite file
  and line. Reading a configuration does not prove a deployment uses it.
- `[Owner-states]`: attribute the owner's statement; it has not been independently
  verified. Do not promote it to an observed control or deployment fact.
- `[Unverified]`: an inference or unchecked external fact; name the missing check.
  Where live research was needed but not done, include "Not independently verified."

Keep table columns and enum values unchanged. Put tags beside claims in the
description, controls, or evidence cells. For a likelihood, impact, or status
based on an unverified premise, identify that premise and qualify the score in
the same row's evidence cell. Scores are assessments, not code facts.

Also list unchecked premises affecting scores in `## 6. Open questions`.
That follow-up does not replace the inline qualifier. Do not treat an
owner-stated control as independently verified when reducing risk or excluding
a finding. Carry the qualification through scan, triage, and reporting;
use `NEEDS-CHECK` if an unchecked external fact changes a finding's severity.

---

## Required sections, in order

```markdown
# Threat Model: <system name>

## 1. System context
## 2. Assets
## 3. Entry points & trust boundaries
## 4. Threats
## 5. Deprioritized
## 6. Open questions
## 7. Provenance
## 8. Recommended mitigations
```

### 1. System context
One to three paragraphs. What it is, what it does, who uses it, where it runs.
No table.

### 2. Assets
| asset | description | sensitivity |
|---|---|---|

`sensitivity` ∈ `low` · `medium` · `high` · `critical`

### 3. Entry points & trust boundaries
| entry_point | description | trust_boundary | reachable_assets |
|---|---|---|---|

One row per place untrusted input enters or privilege changes. For Android,
every exported component is an entry point, and so is every deep link, every
`ContentProvider` URI, and the backup channel.

### 4. Threats
The threat model proper. One row per actor-wants-outcome pair.

| id | threat | actor | surface | asset | impact | likelihood | status | controls | evidence |
|---|---|---|---|---|---|---|---|---|---|

- `id` — `T1`, `T2`… Stable. Never renumber when a row is removed.
- `threat` — one sentence, active voice, names the outcome. "Another installed
  app reads cycle records through the exported provider", not "provider is
  exported".
- `actor` ∈ `remote_unauth` · `remote_auth` · `adjacent_network` · `local_user` ·
  `local_admin` · `supply_chain` · `insider`
  For a mobile app add: `malicious_installed_app` · `device_thief` ·
  `backup_extractor`. These three cover most real mobile threats and are missing
  from the server-oriented original list.
- `impact` ∈ `low` · `medium` · `high` · `critical` · `existential`
- `likelihood` ∈ `very_rare` · `rare` · `possible` · `likely` · `almost_certain`
- `status` ∈ `unmitigated` · `partially_mitigated` · `mitigated` · `risk_accepted`
- `evidence` — advisory IDs, issue links, or commit hashes that **instantiate**
  the threat. May be empty. Evidence raises likelihood; it is not the threat.

Sort by (impact, likelihood) descending.

### 5. Deprioritized
| threat | reason |
|---|---|

Explicitly parked threats. "Out of scope", "actor not in model", "risk accepted
by owner". Writing these down is what stops the same argument recurring every
audit.

### 6. Open questions
Bullet list. What the mode could not determine, plus every `[Owner-states]` or
`[Unverified]` premise that moved a score. Keep its qualifier in the affected
table row as well.

### 7. Provenance
```markdown
- mode: interview | bootstrap | bootstrap-then-interview
- date: YYYY-MM-DD
- target: <path or repo url @ commit>
- inputs: <design doc path | "none">
- owner: <name> | <unset>
```

### 8. Recommended mitigations
Optional and additive. One row per **class-level control** that closes or
materially shrinks a whole threat, not a per-finding patch.

---

## How the scan uses it

- A finding that instantiates a modeled threat gets `threat_ref: "T3"` and sorts
  above findings that do not.
- A finding touching a `critical` asset, or instantiating a threat whose
  `impact` is `critical` or `existential`, **qualifies for the severity
  adjustment in `methodology.md`. It does not apply a step of its own.** That
  adjustment is capped at one step in total and cannot reach CRITICAL. A
  finding on health data already qualifies through the sensitive-data trigger,
  so the threat model does not add a second bump.
- Before reducing or excluding a finding because of `## 5. Deprioritized`,
  verify that the recorded reason applies to the observed path. Retain owner
  acceptance as an attributed decision, not proof that the issue is mitigated.
  An unchecked control cannot by itself justify dismissing the finding.
- If `THREAT_MODEL.md` is absent, the scan runs anyway and says once in the
  report that findings were not weighted against a threat model.
