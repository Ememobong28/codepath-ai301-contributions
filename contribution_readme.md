# Contribution 1: Add LogSanitizer and UploadedFileUtils utilities; sanitize filename logging in EDocUtil

**Contribution Number:** 1

**Student:** Immanuella Emem Umoren

**Issue:** https://github.com/carlos-emr/carlos/issues/2267

**My Fork:** https://github.com/Ememobong28/carlos

**Status:** Phase II — Complete

---

## Why I Chose This Issue

This issue asks me to add two small utility classes (`LogSanitizer` and `UploadedFileUtils`) and to sanitize user-controlled filenames and paths that `EDocUtil.java` currently writes into logs via raw string concatenation, which allows log injection / log forging. It matters because it's a real security fix in a healthcare EMR that handles sensitive patient data, and because the two utility classes are blockers for two other in-progress issues ([#2262](https://github.com/carlos-emr/carlos/issues/2262) and [#2263](https://github.com/carlos-emr/carlos/issues/2263)), so completing it unblocks downstream work. I chose it because it's clearly scoped with a fix pattern already present in the codebase (`LogSafe.sanitize` in `BillingOnRaService.java`), while still being challenging enough to teach me secure logging practices and how to navigate a large legacy Java/Struts project.

My relevant background: I'm comfortable with Java and object-oriented patterns, and my learning goal for this contribution is to understand secure logging (log injection/forging defenses) and how a deprecation-shim migration is rolled out incrementally across a large codebase.

---

## Understanding the Issue

### Problem Description

`EDocUtil.java` logs user-controlled filenames and file paths by concatenating them directly into log message strings. Because the values aren't sanitized, an attacker could inject newlines or fake log entries (log injection / log forging). Separately, two utility classes that other security work depends on — `LogSanitizer.java` and `UploadedFileUtils.java` — exist on the PR 2092 branch but are missing from `develop`.

### Expected Behavior

Filename and path values should be passed through `LogSafe.sanitize()` before being logged, matching the existing safe-logging pattern already used in `BillingOnRaService.java`.

### Current Behavior

Values are concatenated straight into log strings unsanitized, e.g.:
`logger.error("Error resolving file path: " + fileName, e);`

### Affected Components

- `src/main/java/io/github/carlos_emr/carlos/documentManager/EDocUtil.java` — unsanitized logging at lines 1262, 1341, 1352, 1355, 1358 (line numbers confirmed in the issue by the author)
- New file: `src/main/java/io/github/carlos_emr/carlos/utility/LogSanitizer.java` — deprecated shim delegating to `LogSafe`
- New file: `src/main/java/io/github/carlos_emr/carlos/utility/UploadedFileUtils.java` — helpers for extracting canonical `File` handles from Struts `UploadedFile` objects
- Reference implementation to mirror: `src/main/java/io/github/carlos_emr/carlos/.../BillingOnRaService.java` (existing correct `LogSafe.sanitize` usage)

**Out of scope (reserved for other issues):** `NioFileManagerImpl` ([#2213](https://github.com/carlos-emr/carlos/issues/2213)) and the `validateFileName` / `PathValidationUtils` call sites ([#2262](https://github.com/carlos-emr/carlos/issues/2262), [#2263](https://github.com/carlos-emr/carlos/issues/2263)).

### Acceptance Criteria

- [ ] `LogSanitizer.java` created in `utility/` as a `@Deprecated(forRemoval = true)` shim delegating both `sanitize` overloads to `LogSafe`.
- [ ] `UploadedFileUtils.java` created in `utility/` with `getUploadedFile` (throws `IllegalStateException` on null / no backing file) and `getUploadedFileOrNull` (returns null).
- [ ] All five `EDocUtil.java` logging sites (1262, 1341, 1352, 1355, 1358) wrap filename/path variables in `LogSafe.sanitize(...)`.
- [ ] No changes made to `NioFileManagerImpl` or the `validateFileName` / `PathValidationUtils` call sites.
- [ ] Project builds and existing tests pass; new unit tests added for both utility classes.

---

## Reproduction Process

### Environment Setup

CARLOS provides a Docker-based dev container (Java 21, Spring 5, MariaDB), which is the recommended setup. My path:

1. Installed Docker Desktop, VS Code, and the Dev Containers extension.
2. Cloned my fork and opened it in VS Code; used "Reopen in Container" to build the dev container. First build pulled all Maven dependencies (~15-30 min).
3. **Memory tuning:** On an 8 GB Mac, the full `make install` was killed during the `checkstyle` phase (the container ran out of memory — `free -h` showed only ~1.4 GB free). Fix: skip checkstyle with the make script's built-in flag, `make install --skip-checks`. Checkstyle is a lint pass and still runs in CI, so skipping it locally is safe.
4. **Compilation succeeds:** all 4,039 source files compile cleanly with `--skip-checks`, which confirms the environment is sound and let me inspect the affected code directly.
5. **Known build quirk:** the final `prepare-package` (WAR deploy) step fails on an Ant task looking for `target/classes/carlos.properties`. This is a documented build peculiarity (CONTRIBUTING.md notes the raw packaging is non-trivial) and does not affect reproduction — issue #2267 is a code-level logging vulnerability verifiable in source, not a runtime bug.

Working branch: https://github.com/Ememobong28/carlos/tree/fix-issue-2267

### Steps to Reproduce

This is a code-level security issue (log injection / log forging via unsanitized user-controlled filenames), so reproduction is confirmed by inspecting the source against the safe pattern used elsewhere in the codebase.

1. Build/compile the project in the dev container: `make install --skip-checks`.
2. Open `src/main/java/io/github/carlos_emr/carlos/documentManager/EDocUtil.java`.
3. Search the file for `logger.` calls that concatenate a filename/path variable directly into the message string (e.g. `logger.error("... " + fileName, e);`).
4. **Observed:** seven logging sites concatenate a user-controlled filename straight into the log message without sanitization — at lines **1300, 1316, 1383, 1394, 1397, 1400, 1403**.
5. Compare against the safe pattern in `BillingOnRaService.java`, which wraps values in `LogSafe.sanitize(...)` and uses `{}` placeholders. The `EDocUtil` sites do not.
6. **Expected (correct) behavior:** filename/path values should be passed through `LogSafe.sanitize()` before logging, consistent with the project's mandatory security rule "No PHI in logs" and "no string concatenation."

**Note on line numbers:** the original issue cited lines 1262, 1341, 1352, 1355, 1358. On current `develop` the unsanitized logging has shifted and expanded to the seven lines above — the file has changed since the issue was filed.

### Reproduction Evidence

- Working branch: https://github.com/Ememobong28/carlos/tree/fix-issue-2267
- Affected file/lines: `documentManager/EDocUtil.java` lines 1300, 1316, 1383, 1394, 1397, 1400, 1403 (all concatenate a filename into a log call)
- Reference (safe) pattern: `BillingOnRaService.java` using `LogSafe.sanitize(...)`

---

## Solution Approach

### Analysis

The root cause is that `EDocUtil.java` writes user-controlled filenames into log messages via raw string concatenation, with no sanitization. An attacker who controls a filename could inject newlines or forged log lines (log injection / forging). The codebase already has the correct tool (`LogSafe.sanitize`) and a reference usage (`BillingOnRaService.java`); `EDocUtil` simply wasn't migrated. Separately, two helper classes referenced by downstream issues (#2262, #2263) — `LogSanitizer` and `UploadedFileUtils` — don't yet exist on `develop`.

### Proposed Solution

Add the two utility classes specified in the issue, then sanitize every filename/path value logged in `EDocUtil.java` using `LogSafe.sanitize()`, mirroring the existing safe pattern. Keep strictly to the issue's scope.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** `EDocUtil` logs user-controlled filenames unsanitized at 7 sites, enabling log injection. Filenames should be sanitized via `LogSafe.sanitize()` before logging.

**Match:** `BillingOnRaService.java` already logs safely with `LogSafe.sanitize(...)` and `{}` placeholders — this is the pattern to copy.

**Plan:**
1. Create `LogSanitizer.java` in `utility/` as a `@Deprecated(forRemoval = true)` shim delegating both `sanitize` overloads to `LogSafe` (per the issue snippet).
2. Create `UploadedFileUtils.java` in `utility/` with `getUploadedFile` (throws `IllegalStateException` on null / no backing file) and `getUploadedFileOrNull` (returns null).
3. In `EDocUtil.java`, wrap the filename/path variable in `LogSafe.sanitize(...)` at lines 1300, 1316, 1383, 1394, 1397, 1400, 1403.
4. Add JUnit 5 unit tests for both new utility classes.
5. Do **not** modify `NioFileManagerImpl` or the `validateFileName` / `PathValidationUtils` call sites (reserved for #2213 / #2262 / #2263).

**Implement:** [Phase III — commits on https://github.com/Ememobong28/carlos/tree/fix-issue-2267]

**Review:** Self-review against CONTRIBUTING.md — Conventional Commits message format, mandatory DCO sign-off (`git commit -s`), CARLOS copyright header on new files, `io.github.carlos_emr.carlos.*` package namespace, PR targets `develop`.

**Evaluate:** Confirm the project still compiles with `make install --skip-checks`; run the new unit tests (`make install --run-unit-tests`); re-inspect the seven `EDocUtil` sites to verify each now uses `LogSafe.sanitize()`.

**Understand:** [Phase III]

**Match:** The codebase already uses `LogSafe.sanitize(...)` in `BillingOnRaService.java` — this is the pattern to mirror.

**Plan:**
1. Create `LogSanitizer.java` in `utility/` as a deprecated shim delegating to `LogSafe`.
2. Create `UploadedFileUtils.java` in `utility/` with `getUploadedFile` and `getUploadedFileOrNull`.
3. Wrap filename/path variables in `LogSafe.sanitize(...)` at the five named `EDocUtil.java` lines.
4. Add unit tests for both utility classes.

---

## Testing Strategy

### Unit Tests

- [ ] `LogSanitizer.sanitize` delegates to `LogSafe` (normal input)
- [ ] `LogSanitizer.sanitize` handles null and over-length input
- [ ] `UploadedFileUtils.getUploadedFile` returns backing file; throws `IllegalStateException` on null / no backing file
- [ ] `UploadedFileUtils.getUploadedFileOrNull` returns null when upload unavailable

### Integration Tests

[Phase III]

### Manual Testing

[Phase III]

---

## Implementation Notes

[Phase II onward]

---

## Pull Request

**PR Link:** [Phase IV]
**PR Description:** [Phase IV]
**Maintainer Feedback:** [Phase IV]
**Status:** [Phase IV]

---

## Learnings & Reflections

[Phase IV]

---

## Resources Used

- Issue #2267 (this contribution): https://github.com/carlos-emr/carlos/issues/2267
- Opened by maintainer Ben-Heerema; references PR #2092 (origin of the two utility classes)
- Blocked-downstream issues: [#2262](https://github.com/carlos-emr/carlos/issues/2262), [#2263](https://github.com/carlos-emr/carlos/issues/2263)
- Related out-of-scope issue: [#2213](https://github.com/carlos-emr/carlos/issues/2213)
- CARLOS repo: https://github.com/carlos-emr/carlos
- CARLOS `CONTRIBUTING.md` / README (Phase II setup reference)
