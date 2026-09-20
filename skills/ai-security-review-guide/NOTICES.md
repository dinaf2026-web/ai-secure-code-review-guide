# Credits and licensing

`ai-security-review-guide` (formerly `appsec-audit`) is a derivative work. It combines material from two Anthropic
open-source reference implementations with an Android/Kotlin rule pack and a
reporting layer written for this package.

Attribution is recorded here so it survives, and so the upstream terms are met
if this is ever published.

---

## Upstream 1 — claude-code-security-review

- Repository: https://github.com/anthropics/claude-code-security-review
- License: **MIT**
- `Copyright (c) 2025 Anthropic`
- Verified against the live repository on 19-09-2026.

**What was taken:**

- The security audit prompt structure in `claudecode/prompts.py`: the
  three-phase methodology (context → comparative → assessment), the
  vulnerability category list, the severity bands, the confidence scoring
  scale with its 0.7 reporting floor, and the findings JSON schema. Now in
  `references/methodology.md`.
- The hard exclusion rules in `claudecode/findings_filter.py`: DOS and resource
  exhaustion, rate limiting, resource management, regex injection, open
  redirect, memory safety outside C/C++, SSRF in HTML, and findings in Markdown
  files. Now in `references/exclusions.md`.
- The diff-scoped review approach from `.claude/commands/security-review.md`,
  used by `diff` mode.

**What was changed:**

- **Secrets on disk are reported, not excluded.** The upstream prompt excludes
  them on the stated basis that they are handled by another process. There is no
  such process here, so the exclusion is reversed and escalated.
- Intent redirection is explicitly carved out of the open-redirect exclusion.
- Memory safety is re-enabled for bundled JNI and NDK code in Android targets.
- Secrets in Markdown and other non-executed files are carved out of the
  Markdown exclusion.
- Severity is raised one step for health and other sensitive personal data.
- A `provenance` field was added to the findings schema, and
  `areas_not_scanned` was made required.
- The API-based second-pass filter was dropped. Filtering is done in-session
  rather than through a separate Claude API call.

**MIT permission notice**, reproduced as required:

> Permission is hereby granted, free of charge, to any person obtaining a copy
> of this software and associated documentation files (the "Software"), to deal
> in the Software without restriction, including without limitation the rights
> to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
> copies of the Software, and to permit persons to whom the Software is
> furnished to do so, subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in all
> copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
> IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
> FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
> AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
> LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
> OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
> SOFTWARE.

---

## Upstream 2 — defending-code-reference-harness

- Repository: https://github.com/anthropics/defending-code-reference-harness
- License: **Apache License, Version 2.0**
- `Copyright 2026 Anthropic PBC`
- Full license included in [`LICENSE-APACHE-2.0.txt`](LICENSE-APACHE-2.0.txt).
- Official text: https://www.apache.org/licenses/LICENSE-2.0.txt
- The repository contains no `NOTICE` file. Confirmed 19-09-2026.
- Upstream states it is not maintained and not accepting contributions.

**What was taken:**

- The `THREAT_MODEL.md` schema from `.claude/skills/threat-model/schema.md`:
  the eight required sections, the table column orders, and the enum value sets
  for actor, impact, likelihood, and status.
- The four-question framework and the interview method from
  `.claude/skills/threat-model/interview.md`, including the
  `[Code-verified]` / `[Owner-states]` provenance discipline. The framework
  itself is credited upstream to Shostack, *The Four Question Framework for
  Threat Modeling* (2024).
- The pipeline shape: threat-model → scan → triage → report, with parallel
  subagents per focus area, from `.claude/skills/vuln-scan/SKILL.md` and
  `.claude/skills/triage/SKILL.md`.

**What was changed** (Apache 2.0 §4(b) requires stating this):

- The autonomous harness is **not** included. Upstream's `harness/` runs Docker,
  gVisor, and ASAN against C/C++ targets and executes target code. This skill
  never executes the target, so that half was omitted deliberately rather than
  ported. It also requires Docker and gVisor, which are not available on every
  platform.
- The detection-and-response track (`/dnr-hunt`, `/dnr-respond`) is not included.
- Three mobile-specific actor values were added to the threat table enum:
  `malicious_installed_app`, `device_thief`, `backup_extractor`. The upstream
  list is server-oriented and does not cover the common mobile cases.
- Android entry points (exported components, deep links, ContentProvider URIs,
  the backup channel) were added to the entry-points guidance.
- The schema was reformatted as a reference file inside a single skill rather
  than a standalone skill with its own checkpoint script.

Apache 2.0 short notice (the complete license is supplied separately):

> Licensed under the Apache License, Version 2.0 (the "License"); you may not
> use this file except in compliance with the License. You may obtain a copy of
> the License at http://www.apache.org/licenses/LICENSE-2.0
>
> Unless required by applicable law or agreed to in writing, software
> distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
> WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
> License for the specific language governing permissions and limitations under
> the License.

---

## Original material

Written for this skill, not derived from either upstream:

- **`references/android-kotlin.md` in full.** Neither upstream has any Android
  or Kotlin content. Nine focus areas covering manifest and exported components,
  intents and PendingIntent, WebView, data at rest, network and crypto, logging
  and leakage, auth and biometrics, build and signing, and Play policy.
  Written for this package.
- **`references/reporting.md` in full.** XLSX and DOCX deliverable spec,
  formatting floors, the three-destination delivery routine, and the rule that
  provenance survives into the report.
- The three-tag provenance model (`CODE-VERIFIED` / `INFERRED` / `NEEDS-CHECK`)
  and its enforcement in the findings schema.
- The hard rules in `SKILL.md`, including the prohibition on printing secret
  values and on calling an app secure or clean.

---

## If this is ever published

Per the standing skill-repo publishing rule, a public release needs:

1. Skills under `skills/<name>/SKILL.md`
2. `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
3. An MIT `LICENSE` for the original material, © Dina Farhat
4. This `NOTICES.md` and the full `LICENSE-APACHE-2.0.txt` carried forward intact
5. A README Credits section naming both upstream repositories

The MIT copyright and permission notice is reproduced above. The complete
Apache 2.0 license is included in `LICENSE-APACHE-2.0.txt`, downloaded from the
official Apache source on September 20, 2026. The short notice and link alone
were not a complete license copy.

Before distribution, verify the applicable upstream notices and retain them.
Modified files must identify their changes. Including these files does not
establish that every release-packaging requirement has been met.

## Review corrections on September 20, 2026

- `SKILL.md` and `references/methodology.md`: distinguish pending and branch
  scope, include untracked files, and keep patch and inventory comparisons aligned.
- `references/exclusions.md`: preserve security-relevant arithmetic-overflow
  findings in managed languages and inspect reachable native or unsafe operations.
- `references/threat-model.md` and `references/reporting.md`: preserve claim
  provenance inline through scores, exclusions, and summaries.
- `references/android-kotlin.md`: evaluate effective provider permissions and
  release debug settings before reporting exposure.
- `NOTICES.md` and `LICENSE-APACHE-2.0.txt`: correct the license-copy claim and
  supply the full Apache license.

**Confirm before flipping any repository public.** Nothing here has been
published.
