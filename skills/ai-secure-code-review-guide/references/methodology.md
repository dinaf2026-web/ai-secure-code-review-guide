# Scan methodology

Modified September 20, 2026: Pending and branch diff scopes corrected.
Modified September 20, 2026: Severity adjustment made single-source, non-cumulative, and capped below CRITICAL.

Read this before any `scan` or `diff` run. Adapted from the audit prompt in
`anthropics/claude-code-security-review` (MIT). See `../NOTICES.md`.

---

## Objective

Identify **high-confidence** vulnerabilities with real exploitation potential.

This is not a general code review. Style, naming, architecture, test coverage,
and performance are out of scope unless they create an exploitable condition.
If it would not make a security engineer sit up in a review, it does not belong
in the output.

---

## Three-phase method

Work the phases in order. Skipping Phase 1 is what produces findings that
contradict the codebase's own conventions.

### Phase 1 — Context

Before reading any file for vulnerabilities:

- What security libraries and frameworks are already in use?
- What are the established sanitization, validation, and auth patterns here?
- Where are the trust boundaries? What is the app's own threat model
  (read `THREAT_MODEL.md` if present)?
- What does this app actually protect? Health records read differently from a
  plant photo cache.

### Phase 2 — Comparative

- Compare the code against the patterns found in Phase 1.
- Deviations from an established secure pattern are higher-signal than isolated
  matches against a generic rule.
- Inconsistency is a finding: three call sites parameterized and one not is a
  stronger signal than four that are all unparameterized (which may be a
  deliberate, mitigated design).

### Phase 3 — Assessment

- Trace data flow from untrusted input to sensitive operation. An unsafe sink is
  only a vulnerability if something reaches it.
- Identify where privilege boundaries are crossed.
- For each candidate: can you state a concrete attacker, a concrete input, and a
  concrete outcome? If not, it is not a finding.

---

## Categories

### Input validation
SQL injection · command injection · XXE · template injection · NoSQL injection ·
path traversal · deserialization of untrusted data

### Authentication & authorization
Auth bypass logic · privilege escalation · session management flaws · JWT
verification errors (alg confusion, missing signature check, missing expiry) ·
IDOR · authorization checks on the client only

### Crypto & secrets
Hardcoded keys, tokens, passwords · weak algorithms · improper key storage ·
predictable randomness · certificate validation bypass · hardcoded IV · ECB mode

### Code execution
RCE via deserialization · pickle/YAML injection · `eval` injection · dynamic
class loading from untrusted input · XSS (reflected, stored, DOM)

### Data exposure
Sensitive data in logs · PII handling violations · API over-fetch and leakage ·
debug information exposure · data in crash reports and analytics

### Business logic
Race conditions · TOCTOU · replay · state machine bypass

For Android and Kotlin targets, `android-kotlin.md` adds a large category set on
top of these. Load it.

---

## Severity

| Severity | Bar |
|---|---|
| `CRITICAL` | Remote, unauthenticated, leads to data breach or code execution. No user interaction. |
| `HIGH` | Directly exploitable: RCE, auth bypass, mass data exposure. May require user interaction or local access. |
| `MEDIUM` | Real impact, but requires specific conditions, a chained precondition, or elevated position. |
| `LOW` | Defense-in-depth. Would not be exploitable alone. |

Two adjustments to the generic scale:

- **Local network is not a discount.** Something exploitable only from the local
  network can still be HIGH.
- **Sensitive context raises severity, once.** See the adjustment rule below.

### The severity adjustment rule

Severity is assigned once, here. **This is the only place in this skill that
raises a severity, and it raises it at most one step, however many triggers
apply.**

**Triggers.** Any one of these qualifies a finding for the single step:

- The data at risk is health, cycle or symptom records, medications, diagnoses,
  precise location, biometrics, a government ID, auth material, or a session
  token.
- The finding touches an asset marked `critical` in `THREAT_MODEL.md`.
- The finding instantiates a modeled threat whose `impact` is `critical` or
  `existential`.

**It is not cumulative.** A finding that trips all three still moves one step.
Health data is normally both sensitive data and a critical asset, so counting
those separately would carry a MEDIUM to CRITICAL on a single underlying
property. That is one fact counted twice.

**The step stops below CRITICAL.** Adjustment can raise LOW to MEDIUM and
MEDIUM to HIGH. It cannot produce CRITICAL. CRITICAL has to be earned against
its own row above: remote, unauthenticated, no user interaction. Plain-text
health data on a device needs local access, so it is HIGH, and it stays HIGH
however sensitive the data is. A severity the table's own definition excludes
is not a severity, it is inflation.

**Direct assignment is a different mechanism.** A rule that names an outcome
outright, such as a live credential committed to a public repository, assigns
that severity because the finding meets the bar on its own terms. It is not a
step, so the one-step cap does not apply to it, and the adjustment is not
applied on top of it.

**Record it.** When a severity was adjusted, the finding says so: the base
severity, the trigger used, and the result. An unexplained severity cannot be
argued with.

---

## Confidence

| Range | Meaning |
|---|---|
| 0.9–1.0 | Exploit path identified end to end. |
| 0.8–0.9 | Clear vulnerability pattern, known exploitation method, reachable. |
| 0.7–0.8 | Suspicious pattern; exploitation requires conditions you could not confirm. |
| 0.5–0.7 | Not a finding. May be carried as a **lead** in `scan` mode only. |
| **< 0.5** | **Drop entirely.** Too speculative to be worth a human's time. |

A finding at 0.7–0.8 is reported as `INFERRED`, and the description says what
was not traced.

### Why there is a lead tier, and only in `scan` mode

The 0.7 floor was inherited from a tool built to gate pull requests. There, a
false positive interrupts someone mid-task, the diff is small, and the reviewer
can usually trace the new code to a confident answer. A high floor is right.

A one-time audit of an existing codebase has the opposite cost balance. The
surface is far larger, many data flows cannot be traced to certainty in a single
pass, and a missed vulnerability in a health app costs more than a noisy line in
a report. Applying a gate's floor to an audit silently discards the leads a
human would most want to chase.

So the floor for **findings** does not move. Instead, `scan` mode keeps a second
list.

**Lead rules.** A lead is not a weak finding, it is an open question with an
address:

- It carries `file:line`, exactly as a finding does.
- It names **what could not be traced** and **the one check that would resolve
  it**. "Possible issue here" is not a lead, it is noise.
- It never appears in the findings log, and never counts toward the severity
  totals. Counting leads as findings is how a report becomes unreadable.
- It carries no severity. Severity is for confirmed findings.
- Below 0.5 it is dropped, not demoted.

**`diff` mode has no lead tier.** A gate that emits open questions is not a
gate. In `diff`, a candidate is a finding or it is nothing.

---

## Output schema

`FINDINGS.json`:

```json
{
  "findings": [
    {
      "id": "F1",
      "file": "app/src/main/java/com/example/Db.kt",
      "line": 42,
      "severity": "HIGH",
      "category": "sql_injection",
      "provenance": "CODE-VERIFIED",
      "description": "User-supplied search term concatenated into a raw SQL query.",
      "evidence": "val q = \"SELECT * FROM entries WHERE note LIKE '%$term%'\"",
      "exploit_scenario": "A note synced from an untrusted source containing a quote character alters the query and returns rows belonging to other profiles.",
      "recommendation": "Use a parameterized query: pass term as a bind argument rather than interpolating it.",
      "confidence": 0.9,
      "threat_ref": "T3",
      "severity_base": "MEDIUM",
      "severity_adjustment": "sensitive-data trigger: health records"
    }
  ],
  "analysis_summary": {
    "files_reviewed": 0,
    "critical": 0, "high": 0, "medium": 0, "low": 0,
    "areas_scanned": [],
    "areas_not_scanned": [],
    "review_completed": true
  }
}
```

Field notes:

- `id` — stable. Never renumber when a finding is removed.
- `evidence` — the actual line, quoted. **Redact any secret value.**
- `exploit_scenario` — concrete actor, concrete input, concrete outcome. If you
  cannot write this sentence, drop the finding.
- `threat_ref` — the `THREAT_MODEL.md` threat ID this instantiates, if there is
  one. Findings that map to a modeled threat sort above those that do not.
- `areas_not_scanned` — **required and non-empty in almost every real run.**
  This is how the report stays honest about coverage.
- `severity_base` and `severity_adjustment` — present only when the severity
  adjustment rule was applied. `severity_base` is the severity before the step,
  and `severity_adjustment` names the single trigger used. Both `null` when the
  severity was assigned directly. Because the rule allows one step and one
  trigger, a finding whose `severity` is more than one rank above
  `severity_base` is a bug in the run, not a severe finding. Check that before
  delivering.

---

## Diff mode

For `diff`, the scope narrows but the bar does not change.

- Review only what the change introduces. Do not report pre-existing issues in
  untouched code. If you spot one, mention it once in a single line below the
  findings, not as a finding.
### Pending changes: `diff` without a base

Review the index and working tree separately. Do not combine them into a single
HEAD-to-working-tree diff: an unstaged reversal can hide a staged change.
Run from the repository root, check each exit status, and use these matching
patch and inventory commands:

```text
git diff --cached --no-ext-diff --no-textconv --
git diff --cached --no-ext-diff --no-textconv --name-status -z --
git diff --no-ext-diff --no-textconv --
git diff --no-ext-diff --no-textconv --name-status -z --
git ls-files --others --exclude-standard -z --
```

The last command lists untracked, non-ignored files. Read each as a new-file
candidate; Git diff does not include them. Parse NUL-delimited filenames without
splitting on whitespace. Retain both paths for a rename. For a deleted file,
inspect the removed content in the patch. For a staged finding, inspect the
index version, not just the working copy, and label its evidence as staged.
Redact secrets before any content reaches tool output or a report.

This pending review works without a remote or origin/HEAD. On a repository with
no commits, `git diff --cached` covers staged additions; the other two inventories
still cover unstaged and untracked files. Check for unmerged paths with
`git ls-files --unmerged -z --`; unresolved conflicts prevent a completed gate.
Record ignored files, uninspected binaries, and unreviewed submodule contents as
coverage limits. A failed command is not an empty result.

### Branch changes: `diff <base>`

Use the base explicitly requested by the user. Resolve it to a commit first;
never replace it silently with origin/HEAD. If the base cannot be resolved or
there is no common ancestor with HEAD, report that branch comparison as blocked.
Pending review can still proceed, but does not complete the branch review.

After assigning the resolved base commit to `$auditBase`, use matching
comparisons for the committed patch and file inventory:

```powershell
git diff --no-ext-diff --no-textconv "$auditBase...HEAD" --
git diff --no-ext-diff --no-textconv --name-status -z "$auditBase...HEAD" --
```

Also inspect pending changes with the commands above, labeled separately, unless
the user expressly requested committed changes only. In that case, disclose the
pending changes as outside scope. Record the resolved base and HEAD commits.
Do not fetch, change branches, stage files, or execute diff helpers to review.

Command semantics checked against Git documentation on September 20, 2026:
https://git-scm.com/docs/git-diff and https://git-scm.com/docs/git-ls-files.

- Same confidence floor, same exclusions, same provenance tags.

---

## Reporting discipline

- Sort by severity, then confidence, then file.
- Deduplicate: the same root cause at five call sites is one finding with five
  locations, not five findings.
- If the scan surfaced nothing above the floor, say exactly that: "No finding at
  or above 0.7 confidence surfaced in the areas scanned," then list the areas
  scanned and the areas not scanned. Do not pad the report to look thorough, and
  do not lower the floor to produce output.
