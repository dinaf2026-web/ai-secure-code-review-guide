# Reporting

Read this for `report` mode. This is where the audit becomes a deliverable.

`FINDINGS.json`, `FINDINGS.md`, `TRIAGE.json`, and `THREAT_MODEL.md` are working
files. They stay in the project tree. **They are not the deliverable.** Per the
standing rule, `.md` is never a final deliverable.

Two files ship:

1. `LEADER - Security findings log (dd-mm-yyyy).xlsx` — one row per finding
2. `LEADER - Security audit report (dd-mm-yyyy).docx` — the written assessment

Leader is the project code, for example `MYAPP`. Date is **dd-mm-yyyy**,
day first, in parentheses, last. Plain spaces in the description, one hyphen
after the leader, no underscores, no em dashes.

Use the `xlsx` and `docx` skills to build them.

---

## Formatting — applied explicitly, never library defaults

Same floor as every other deliverable. **Never shrink below 12pt to make
something fit.** Widen the column, go landscape, or split the table.

| Element | Size |
|---|---|
| Body, table cells, bullets, notes, captions | **12pt minimum, never below** |
| Column headers, minor sub-headers | 12pt bold |
| Sub-headers | 13–14pt bold |
| Section headings | 15pt bold |
| Title | 16–17pt bold |

- Font: Arial
- Paragraph spacing: 8pt after
- Spreadsheet cells get the same 12pt floor as the document. Clamp it
  explicitly, e.g. `size=max(sz, 12)`.
- Dates in prose take a comma **after** the year, including attributive:
  "the July 7, 2025, commit". Not in the filename, which stays dd-mm-yyyy.
- No em dashes anywhere in the report text.

---

## The XLSX findings log

One sheet, `Findings`. Freeze the header row. One row per finding.

| Column | Contents |
|---|---|
| ID | `F1`, `F2` … stable across runs |
| Severity | CRITICAL / HIGH / MEDIUM / LOW |
| Base severity | Severity before adjustment, or blank if assigned directly |
| Adjustment | The single trigger used, or blank. Severity must be at most one rank above base |
| Confidence | 0.70 – 1.00 |
| Provenance | CODE-VERIFIED / INFERRED / NEEDS-CHECK |
| Category | e.g. `exported_component`, `plaintext_storage` |
| File | repo-relative path |
| Line | integer |
| Finding | one sentence, plain words. This is the schema's `description` field |
| Evidence | the actual line. **Secrets redacted to shape.** |
| Exploit scenario | concrete actor, input, outcome |
| Recommendation | what to change |
| Threat ref | `T3`, or blank |
| Status | Open / Fixed / Accepted / Not applicable — left for the owner to fill |

Sort by severity, then confidence descending.

**The contract.** Every field in the `FINDINGS.json` schema has exactly one home
on this sheet. `description` becomes `Finding`; everything else keeps its name.
Before delivering, check that the row count equals the retained-findings count
in `TRIAGE.json`, and that no excluded finding and no lead has leaked onto this
sheet. A count that does not reconcile is a bug in the run, not a result.

Second sheet, `Excluded`: ID, reason, rule number. This is what makes the noise
filter auditable rather than a black box.

Third sheet, `Leads` (scan mode only, omit for `diff`): ID, file, line, what
could not be traced, and the single check that would resolve it. No severity
column. **Leads never appear on the `Findings` sheet and are never added to the
severity counts.** If there are no leads, keep the sheet and write "none", so
the reader can tell an empty list from a missing one.

Fourth sheet, `Coverage`: areas scanned, areas not scanned, files reviewed,
whether a threat model was used. **Required.** A findings log without a coverage
sheet invites the reading that an empty category was checked and clean.

---

## The DOCX report

Structure:

1. **Title** — `<App> security audit`, then the date in prose.
2. **Action items** — anything the owner must handle personally, at the very top.
   Rotating a live key goes here, not on page four.
3. **Scope and method** — what was scanned, what was not, what the tool does and
   does not do. Include the scope limits verbatim from `SKILL.md`: this is a
   code review, not a penetration test; it cannot prove absence.
4. **Summary of findings** — counts by severity, in a small table.
5. **Findings** — one sub-section each, severity order. For each: what it is,
   where, the evidence line, the exploit scenario, the fix. Keep provenance
   audible in the sentence, not in a footnote.
6. **Leads** — scan mode only. Open questions with an address, each naming
   what could not be traced and the check that would settle it. Introduce the
   section with one line saying these are not findings and carry no severity.
7. **Recommendations** — defense-in-depth items that are not findings.
8. **Play policy and compliance** — if Android. Kept separate from findings.
9. **What was not checked** — explicit. Dependency advisories without a live
   lookup, anything marked NEEDS-CHECK, any area skipped.

---

## Provenance must survive into the report

This is the rule the whole thing hangs on.

A claim that was `NEEDS-CHECK` in `FINDINGS.json` stays qualified in the report.
It does not quietly become a statement of fact because it reached a summary
table. **A caveat in section 8 does not license a bare assertion in section 5** —
section 8 gets skimmed past.

Keep owner-stated or unverified threat-model premises beside the affected claim
when they influence a finding, score, exclusion, or summary. An owner-stated
control does not become code-verified when it reaches the report.

If a finding is too weak to verify but too load-bearing to hedge, delete it and
state what is missing instead.

Never write that the app is "secure", "clean", "safe to ship", or "has no
vulnerabilities". Those are conclusions this tool cannot reach. Write what is
true: "No finding at or above the confidence floor surfaced in the areas listed
in section 3."

---

## Delivery

Three destinations, in this order, hash-verified at each step:

1. **The project tree** — the source of truth. Write here first.
2. **The backup mirror** — cloud or external. Backup only, never the working copy.
3. **A staging folder** — optional, for attaching the files to an email.

Then open the project-tree copy, never the backup copy. Opening the backup is how
edits end up living only in the backup while the copy you call the source of
truth stays untouched.

Compare hashes before reporting any copy as done. A command returning zero says
it did not crash, not that it wrote the right bytes. Note that `Get-FileHash`
returns null for a missing file, and null equals null is true, so a naive
compare reports a match for two files that do not exist. Assert both paths exist
first.

Keep the explanation after delivery short. The reader reads the document
themselves.
