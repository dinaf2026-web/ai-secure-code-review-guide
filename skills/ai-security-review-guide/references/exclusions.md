# Exclusions — the noise filter

Modified September 20, 2026: Arithmetic-overflow and native-call filtering corrected.
Modified September 20, 2026: Severity adjustment made single-source, non-cumulative, and capped below CRITICAL.

Read this before deciding whether a candidate becomes a finding.

Ported from `HardExclusionRules` in `claudecode/findings_filter.py`
(`anthropics/claude-code-security-review`, MIT). See `../NOTICES.md`.

**Why this file exists.** A scanner that reports everything gets ignored, and
then a real vulnerability hides inside a list nobody reads. These categories were
excluded upstream because in practice they produce far more false positives than
true ones. Apply them.

---

## Hard exclusions — drop without reporting

### 1. Denial of service and resource exhaustion
Anything matching: denial of service, DOS attack, resource exhaustion; exhaust /
overwhelm / overload of resource, memory, or CPU; infinite or unbounded loop or
recursion.

**Reason:** availability findings from static review are nearly always
theoretical, and services are not required to be DOS-proof.

### 2. Rate limiting
Missing / lack of / no rate limit; rate limiting missing, required, or not
implemented; "implement rate limiting"; unlimited requests or API calls.

**Reason:** a design recommendation, not a vulnerability. Absence of rate
limiting is not by itself a security defect.

### 3. Resource management
Resource / memory / file leak potential; unclosed resource, file, or connection;
"close / cleanup / release the resource"; potential memory leak; database,
thread, socket, or connection leak.

**Reason:** a correctness and reliability issue, not a security vulnerability.

### 4. Regex injection and ReDoS
Regex or regular expression injection, denial of service, or flooding.

**Reason:** a DOS variant, and almost never exploitable for impact.

### 5. Open redirect
Open redirect, unvalidated redirect, redirect attack / exploit / vulnerability,
malicious redirect.

**Reason:** low impact on its own.

> ⚠️ **Android override — see below.** Android **intent redirection** is a
> different vulnerability that shares a word with this one. Do not let this rule
> suppress it.

### 6. Unsupported memory-corruption claims in managed code

Drop a memory-corruption claim when the inspected operation is bounds-checked
by the managed runtime and no native or unsafe operation is reachable. A thrown
bounds exception alone is not evidence of arbitrary memory access. Do not use
the source-file extension as proof of safety; inspect native and unsafe calls.

**Integer overflow and underflow are not excluded by this rule.** Arithmetic
overflow can occur in Kotlin. Trace attacker-controlled arithmetic to a concrete
security outcome, such as bypassing an authorization limit or corrupting an
amount. Apply the usual impact, reachability, and confidence requirements.
An overflow with no security consequence is not a vulnerability finding.

Kotlin arithmetic behavior checked September 20, 2026:
https://kotlinlang.org/docs/numbers.html.

> ⚠️ **Override — native Android code.** If the target ships JNI or NDK code
> (`src/main/cpp/`, `*.c`, `*.cpp`, `CMakeLists.txt`, `Android.mk`, bundled
> `.so` files), memory safety is **fully in scope** for those files. Check for
> native code before applying this exclusion, and record the result in
> `areas_scanned` / `areas_not_scanned`.

### 7. SSRF in client-side HTML
SSRF or server-side request forgery findings in `.html` files.

**Reason:** client-side markup cannot make a server-side request.

### 8. Findings in Markdown files
Any vulnerability finding whose file is `.md`.

**Reason:** documentation is not executed.

> ⚠️ **Override — secrets.** A credential in a `.md`, `.txt`, `.json`, or any
> other non-executed file **is reportable**, at full severity. See below.

---

## Overrides — where upstream is wrong for this package

Four deliberate departures. Each has a reason grounded in something that already
happened.

### A. Secrets on disk ARE reported. Always.

The upstream prompt says: *"Secrets or sensitive data stored on disk (these are
handled by other processes)"* — excluded. That assumption does not hold here.
There is no other process.

Report every hardcoded credential, at HIGH or CRITICAL:

- API keys, tokens, passwords, connection strings, private keys, keystore
  passwords, signing credentials
- In any file type: source, `.md`, `.json`, `.properties`, `.gradle`, `.gradle.kts`,
  `strings.xml`, `res/raw/`, `.env`, committed `local.properties`, CI config
- Including in comments, test fixtures, and example files

**Escalate to CRITICAL if the file is inside a tree that syncs to `G:` or is
committed to a public repository.** A live key on Drive or in a public repo is a
same-day emergency.

**Never print the value.** Report file, line, variable name, and shape:
`"a 32-character hex value assigned to ANALYTICS_API_KEY"`. If it looks live, say so
plainly and say it needs rotating.

### B. Intent redirection is NOT open redirect

Rule 5 suppresses web open-redirect findings. Android **intent redirection** —
taking an `Intent` from an untrusted extra and passing it to `startActivity`,
`sendBroadcast`, or `bindService` — is a distinct and serious bug that can reach
non-exported components and steal permissions.

**Report it.** It is HIGH. Rule 5 does not apply to it. Do not let the shared
word "redirect" collapse the two.

### C. Memory safety applies to bundled native code

See the override under rule 6. Android apps routinely ship `.so` files via
dependencies. If the app has native code, that code gets the C/C++ ruleset.

### D. Sensitive data qualifies a finding for the severity adjustment

Not an exclusion, but it belongs with them. Sensitive data is one trigger for
the single severity step defined in `methodology.md`, "The severity adjustment
rule". **Apply the step there, not here.** This section only says what counts
as sensitive:

- Health records, cycle and symptom data, medications, diagnoses
- Precise location, biometrics, government IDs
- Auth material and session tokens

Worked through: plain-text storage of health data starts at MEDIUM on the
generic scale, takes the one step, and lands at HIGH. It does not continue to
CRITICAL because the threat model also marks those records critical. That would
be the same fact counted twice. Health data in logcat and health data in an
unencrypted cloud backup land the same way, at HIGH.

---

## Soft filters — judgment, not rules

Drop these unless something specific makes them real:

- **Missing input validation with no demonstrated sink.** If there is no proven
  problem downstream, it is not a finding. Upstream is explicit about this.
- **Defense-in-depth suggestions** phrased as vulnerabilities ("should also
  validate", "consider adding"). These are recommendations. They belong in the
  report's recommendations section, not the findings log.
- **Framework behaviour assumed unsafe without checking.** If a framework
  escapes by default, an unescaped-looking call site is not a finding. Check
  the default before reporting.
- **Anything you cannot write an exploit scenario for.** Concrete actor,
  concrete input, concrete outcome, or drop it.

---

## Recording what you dropped

Do not silently discard. `TRIAGE.json` carries an `excluded` array:

```json
{
  "excluded": [
    { "id": "F7", "reason": "Generic rate limiting recommendation", "rule": 2 }
  ],
  "exclusion_breakdown": { "rate_limiting": 1 }
}
```

Report the count in the summary — "14 candidates, 9 reported, 5 excluded as
known-noise categories" — so coverage is auditable and the filter itself can be
challenged. A filter nobody can inspect is how a real finding disappears.
