# Android / Kotlin rule pack

Load this for any target containing `AndroidManifest.xml`, `build.gradle.kts`,
or Kotlin sources. This is the layer neither upstream reference implementation
has, and for a mobile app it covers most of the real attack surface.

Modified September 20, 2026: eleven coverage gaps added from an independent
blind review, plus one FileProvider check added separately. Items whose
applicability turns on an API level are marked `NEEDS-CHECK` rather than
asserted.

---

## ⚠️ Version-dependent behaviour — check, do not assert

Android security defaults change by API level, and several of the rules below
hinge on the target's `compileSdk` / `targetSdk`. **Read the Gradle files first**
and record the actual values.

Any finding whose severity depends on an API level, a library's current status,
or a Play policy is `NEEDS-CHECK` unless verified this session. Never state an
API level from memory in a delivered finding.

Specifically flagged as needing a live check before being relied on:
- Current status of `androidx.security:security-crypto`
  (`EncryptedSharedPreferences` / `EncryptedFile`). This library's support
  status has changed. Verify before recommending it as the fix.
- Current Play Console Data safety and health-data policy wording.
- Any CVE or advisory for a specific dependency version.

---

## 1. Manifest, components, permissions

Read `AndroidManifest.xml` completely before any source file. Merge in library
manifests if the build produces a merged manifest.

| Check | Severity | Notes |
|---|---|---|
| `android:exported="true"` on Activity / Service / Receiver with no `android:permission` | HIGH | Any app on the device can invoke it. Ask what it does with its extras. |
| `<intent-filter>` present but `android:exported` absent | HIGH | Explicit declaration is required from API 31. Check `targetSdk`. |
| `ContentProvider` permits an unauthorized caller to read or change sensitive data | Impact-based after tracing access | Resolve effective `permission`, `readPermission`, `writePermission`, path permissions, export state, caller checks, and actual URI grants. Missing `android:permission` or `grantUriPermissions="true"` alone is not a finding. |
| `android:allowBackup="true"` (this is the **default** — absence of the attribute means true) | HIGH for sensitive data | Health records land in cloud backup and in `adb backup`. Set `false`, or scope with `dataExtractionRules` / `fullBackupContent`. |
| `android:debuggable="true"` in a release manifest | CRITICAL | Full runtime access. |
| `android:usesCleartextTraffic="true"` | HIGH | Plaintext HTTP. Also check for a `networkSecurityConfig` that re-permits cleartext. |
| Custom permission with `protectionLevel="normal"` guarding sensitive operations | MEDIUM | Should be `signature`. |
| `android:sharedUserId` present | MEDIUM | Deprecated, and shares a sandbox. |
| `android:taskAffinity` unset or permissive on sensitive activities | MEDIUM | Task hijacking / StrandHogg-class overlay. |
| `QUERY_ALL_PACKAGES` requested | MEDIUM | Privacy over-reach and a Play policy flashpoint. |
| Dangerous permissions requested but unused in code | MEDIUM | Over-permissioning. Grep each one for a call site. |

**Method:** list every component and its `exported` state. Trace activity,
service, and receiver inputs through their sensitive operations. For providers,
trace caller identity, URI, operation, effective permissions, and grant scope.
An enabled URI-grant mechanism permits temporary access to be granted; it does
not by itself grant arbitrary callers access. Inspect the grant recipient,
read/write flags, and reachable data, including grants from non-exported providers.

Provider permission behavior checked September 20, 2026:
https://developer.android.com/guide/topics/manifest/provider-element.

### Components registered in code, not in the manifest

The table above covers manifest-declared components only. Receivers registered
at runtime are a separate surface and are easy to miss.

| Check | Severity | Notes |
|---|---|---|
| `Context.registerReceiver(...)` without `RECEIVER_EXPORTED` or `RECEIVER_NOT_EXPORTED` | `NEEDS-CHECK`, then HIGH if exported | On the API levels where the flag is required, an unflagged context-registered receiver can default to exported, letting any app trigger it. **Verify the enforcing API level and the app's `targetSdk` before asserting this applies.** Grep `registerReceiver`. |
| `setComponentEnabledSetting` toggling a sensitive component | MEDIUM | Can re-expose a component that looks disabled in the manifest. |

### Tapjacking and overlays

| Check | Severity | Notes |
|---|---|---|
| Destructive or sensitive confirmation controls without `android:filterTouchesWhenObscured="true"` or `setFilterTouchesWhenObscured(true)` | MEDIUM | A malicious overlay can harvest a tap on "delete all records" or "share health data" while the user sees something else. Scope to genuinely destructive or data-sharing controls, not every button. |

---

## 2. Intents, deep links, PendingIntent

| Check | Severity | Pattern |
|---|---|---|
| **Intent redirection** | HIGH | An `Intent` obtained from an extra and then launched. Grep `getParcelableExtra` near `startActivity`, `startService`, `sendBroadcast`, `bindService`. This is **not** open redirect — see `exclusions.md` override B. |
| `PendingIntent` without `FLAG_IMMUTABLE` or explicit `FLAG_MUTABLE` | HIGH | Explicit mutability is required from API 31. A mutable PendingIntent handed to another app can be filled in and fired with this app's identity. |
| Sensitive data in an implicit `Intent` | HIGH | Any app with a matching filter receives it. Set a package or component. |
| `sendBroadcast` without a receiver permission | MEDIUM | Use `LocalBroadcastManager`, a package-scoped intent, or a signature permission. |
| Deep link host / path not validated | HIGH | `intent.data` used to build a request, load a URL, or select a record without an allowlist. |
| App Links without `android:autoVerify="true"` | MEDIUM | Another app can claim the link. |
| `setResult` on an exported activity returning sensitive data | MEDIUM | The caller may be hostile. |
| `PendingIntent` built from an **implicit** `Intent` | HIGH | Distinct from the mutability row above. Mutability governs whether the extras can be refilled; it does not govern which component resolves. Any app with a matching intent-filter can receive it. Set an explicit component or package. |

---

## 3. WebView

If the app has no WebView, record that and move on. If it does, this is
high-yield.

| Check | Severity |
|---|---|
| `addJavascriptInterface(...)` with a remote or user-influenced URL | CRITICAL |
| `setJavaScriptEnabled(true)` with content not fully under app control | HIGH |
| `onReceivedSslError { handler.proceed() }` | CRITICAL — TLS silently disabled |
| `setAllowUniversalAccessFromFileURLs(true)` / `setAllowFileAccessFromFileURLs(true)` | HIGH |
| `setAllowFileAccess(true)` with `file://` loading | HIGH |
| `loadUrl` / `loadDataWithBaseURL` with an unvalidated URL | HIGH |
| `setAllowContentAccess(true)` unnecessarily | MEDIUM |
| No `WebViewClient.shouldOverrideUrlLoading` allowlist | MEDIUM |

`onReceivedSslError` calling `proceed()` is the single highest-value grep in an
Android review. It defeats TLS entirely and looks innocuous in a diff.

---

## 4. Data at rest

The centre of gravity for a health app.

| Check | Severity | Notes |
|---|---|---|
| Health, cycle, symptom, or PII data in plain `SharedPreferences` | HIGH | |
| Plain Jetpack `DataStore` (Preferences or Proto) holding sensitive data | HIGH | DataStore does **not** encrypt. A very common misconception. |
| Room / SQLite database unencrypted for sensitive data | HIGH | Consider SQLCipher. Weigh against the offline-only design before recommending. |
| Sensitive files on external storage (`getExternalFilesDir`, `MediaStore`, `Downloads`) | HIGH | World-readable on older APIs; user-accessible on all. |
| Exported / backed-up database via `allowBackup` | HIGH | Cross-reference §1. |
| Hardcoded secret in `local.properties` that is committed | CRITICAL | Check `.gitignore` actually covers it, and check git history. |
| Secrets in `BuildConfig` fields sourced from a committed file | HIGH | `BuildConfig` is trivially readable after decompilation. It hides a key from a grep, not from an attacker. |
| Secrets in `strings.xml`, `res/raw/`, or assets | HIGH | Shipped in the APK in clear. |
| Keystore key without `setUserAuthenticationRequired` where the data warrants it | MEDIUM | |
| Cache or temp files holding sensitive data, never cleared | MEDIUM | |

### Local database queries

Room binds parameters safely on the normal path. The escape hatches do not.

| Check | Severity | Notes |
|---|---|---|
| `@RawQuery`, or `@Query` assembled by string concatenation | HIGH | Any user-controlled text reaching a raw query can read or corrupt the local health database. A symptom-note search or filter feature is the usual route. |
| `SupportSQLiteDatabase.execSQL` / `rawQuery` with concatenated input | HIGH | Same class. Grep `execSQL`, `rawQuery`, `@RawQuery`. |

### Files leaving the app

| Check | Severity | Notes |
|---|---|---|
| `FileProvider` `<paths>` declaring a broad root, such as `<root-path path="/" />` or `<external-path path="." />` | HIGH | Scopes the provider far wider than the one file meant to be shared. Confirm the declared paths cover only the intended directory, then trace which URIs are actually granted. |
| Captured photos shared or exported without stripping EXIF | HIGH for location data | `ExifInterface`, `MediaStore` capture, then `ACTION_SEND` or a `FileProvider` URI. A shared photo can carry precise GPS the user never intended to publish, independent of any permission they granted. |
| Sensitive free-text fields without `android:importantForAutofill="no"` / `setImportantForAutofill` | MEDIUM | A third-party autofill service can capture and persist symptom or mood notes outside the app's storage controls. |
| `openFileOutput` / `openFileInput` with `MODE_WORLD_READABLE` or `MODE_WORLD_WRITEABLE` | `NEEDS-CHECK`, then HIGH if reachable | Any app could read or write the file directly. These constants are deprecated and throw on newer levels, so **check the app's `minSdk` and `targetSdk` before asserting live risk.** |

**On offline-only apps:** "offline" reduces network risk to
near zero. It does **not** reduce at-rest risk, backup exposure, or exported
component risk. Do not let an offline design dismiss this section.

---

## 5. Network and crypto

| Check | Severity |
|---|---|
| Custom `TrustManager` with an empty `checkServerTrusted` | CRITICAL |
| `HostnameVerifier` returning `true` unconditionally (incl. OkHttp `hostnameVerifier { _, _ -> true }`) | CRITICAL |
| No certificate pinning on an endpoint carrying health data or auth | MEDIUM — HIGH depending on threat model |
| Effective release `networkSecurityConfig` permits cleartext for a traced sensitive request | HIGH if the exposure is reachable |
| Debug-only trust anchors are effective in a release with `android:debuggable="true"` and enable a traced trust bypass | Impact-based; do not flag an inactive block |
| Weak algorithms: `DES`, `3DES`, `RC4`, `MD5`, `SHA-1` for security purposes | HIGH |
| `Cipher.getInstance("AES")` — defaults to **ECB** | HIGH |
| Hardcoded IV, or an IV reused across encryptions | HIGH |
| `java.util.Random` where `SecureRandom` is required | HIGH |
| Key derived from a constant, device ID, or package name | HIGH |
| HTTP URL literal in source | HIGH |

A `debug-overrides` block is ignored when `android:debuggable` is false.
Its presence in packaged resources alone is not a release vulnerability.
Inspect the effective release manifest and trust configuration. If those
artifacts are unavailable, qualify the result instead of assuming they are active.
Do not build the target to obtain them under this read-only skill.

Debug override behavior checked September 20, 2026:
https://developer.android.com/privacy-and-security/security-config.

`Cipher.getInstance("AES")` deserves its own note: it looks correct, compiles,
works, and silently selects ECB. Grep for it specifically.

---

## 6. Logging and leakage

| Check | Severity |
|---|---|
| `Log.*` / `println` / `Timber` emitting PII, health data, tokens, or keys | HIGH |
| Logging left enabled in release (no `BuildConfig.DEBUG` guard, no ProGuard log stripping) | MEDIUM |
| Sensitive data in crash reports or analytics payloads | HIGH |
| Missing `FLAG_SECURE` on screens showing health data | MEDIUM | Blocks screenshots and the recents-screen thumbnail. |
| Sensitive data copied to the clipboard without `EXTRA_IS_SENSITIVE` | MEDIUM |
| Stack traces or internal paths surfaced in UI | MEDIUM |
| Exception messages carrying record contents | MEDIUM |

Two leakage surfaces that are not logs:

| Check | Severity | Notes |
|---|---|---|
| Reminder notifications carrying health content, without a redacted lock-screen form | HIGH | Check `NotificationCompat.Builder.setVisibility` and whether `setPublicVersion` supplies a redacted fallback. Do not assume the platform default; read the setting. A cycle or symptom reminder visible on a locked screen exposes health data to anyone nearby. Different surface from `FLAG_SECURE`, which only covers screenshots and recents. |
| Free-text health fields without `textNoSuggestions` (`TYPE_TEXT_FLAG_NO_SUGGESTIONS`) | LOW | The keyboard's personalized suggestion cache can retain fragments of symptom notes outside the app. Obscure, and low priority against the rest of this section. |

Grep the log calls for the app's own domain vocabulary — for a cycle tracker
that means symptom, flow, cycle, period, medication, mood, weight, temperature.
A generic PII grep will miss all of them.

---

## 7. Auth and biometrics

| Check | Severity |
|---|---|
| `BiometricPrompt` without a `CryptoObject` | HIGH | Result is a boolean an attacker can bypass; without a crypto object nothing is actually unlocked. |
| Auth state as a plain boolean in `SharedPreferences` | HIGH |
| PIN or passcode stored reversibly, or compared in plain text | CRITICAL |
| Lock screen bypassable via an exported activity | HIGH | Cross-reference §1. |
| No re-auth after background timeout on sensitive screens | LOW — MEDIUM |

---

## 8. Build, signing, dependencies

| Check | Severity |
|---|---|
| Keystore password, key alias password, or key path in `build.gradle.kts` | CRITICAL |
| `signingConfigs` referencing a committed keystore | CRITICAL |
| Release build using the debug signing config | HIGH |
| `minifyEnabled false` in release | MEDIUM | Not a vulnerability alone; raises the value of every other finding. |
| Non-HTTPS Maven repository, or `mavenLocal()` in a release path | HIGH |
| Dependency with a known advisory | NEEDS-CHECK | Requires a live lookup. Do not assert a CVE from memory. |
| `debugImplementation` leaking into release | MEDIUM |
| v1-only APK signing (`v2SigningEnabled = false`, v3/v4 absent) | `NEEDS-CHECK` | Signature-downgrade class, associated with CVE-2017-13156. Real exposure depends on `minSdk` and which Android versions can still install the app. **Check both before asserting this is live.** |

**Device integrity is a hardening question, not a finding.** Neither app gates
access to local health data on Play Integrity. On a rooted device the sandbox is
defeated and everything in §4 is readable regardless. Raise it once, in the
report's recommendations section, and do not log it as a vulnerability. Many
sound programs scope this out deliberately.

---

## 9. Play policy and sensitive data

Not vulnerabilities, but they block a release, which is the same practical
outcome. Report in a separate section of the report, never mixed into the
findings log.

- Health or medical data triggers additional Play policy obligations. **Verify
  current wording before citing it.**
- Data safety declaration must match what the code actually collects and
  transmits. Cross-check the declaration against §4 and §6 findings.
- Permissions requested must be justified and used.
- Third-party SDKs that phone home must appear in the declaration.
- If no data leaves the device, say so precisely and evidence it — that is a
  strong position, and it needs to be accurate.

---

## Grep starting points

Fast first pass. Every hit is a **candidate**, not a finding. Open the file,
confirm the line, then apply `methodology.md` and `exclusions.md`.

```
android:exported|android:allowBackup|android:debuggable|usesCleartextTraffic
getParcelableExtra|startActivity\(|sendBroadcast\(|PendingIntent\.
addJavascriptInterface|setJavaScriptEnabled|onReceivedSslError|setAllowFileAccess
checkServerTrusted|HostnameVerifier|hostnameVerifier|SSLSocketFactory
Cipher\.getInstance|SecureRandom|java\.util\.Random|MessageDigest\.getInstance
SharedPreferences|DataStore|getExternalFilesDir|openFileOutput
registerReceiver|setComponentEnabledSetting|filterTouchesWhenObscured
@RawQuery|rawQuery|execSQL
FileProvider|root-path|external-path|ExifInterface|ACTION_SEND
importantForAutofill|textNoSuggestions|setVisibility|setPublicVersion
MODE_WORLD_READABLE|MODE_WORLD_WRITEABLE|v2SigningEnabled
Log\.[dveiw]\(|println\(|Timber\.
BiometricPrompt|setUserAuthenticationRequired
storePassword|keyPassword|signingConfig|minifyEnabled
http://
```

Note: if `ripgrep` is not available in your environment, use your agent's
search tool, or `grep -rn`. A search that prints zero because the binary is missing is a false
negative, not a clean result.
