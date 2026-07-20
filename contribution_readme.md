# Contribution 1: Add LogSanitizer and UploadedFileUtils utilities; sanitize filename logging in EDocUtil

**Contribution Number:** 1

**Student:** Immanuella Emem Umoren

**Issue:** https://github.com/carlos-emr/carlos/issues/2267

**My Fork:** https://github.com/Ememobong28/carlos

**Status:** Phase IV — Complete (Awaiting merge (maintainer approved, content with current state))

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

- `src/main/java/io/github/carlos_emr/carlos/documentManager/EDocUtil.java` — unsanitized logging at lines 1262, 1341, 1352, 1355, 1358 (original issue line numbers; shifted to 1300, 1316, 1383, 1394, 1397, 1400, 1403 on current `develop`)
- New file: `src/main/java/io/github/carlos_emr/carlos/utility/LogSanitizer.java` — deprecated shim delegating to `LogSafe`
- New file: `src/main/java/io/github/carlos_emr/carlos/utility/UploadedFileUtils.java` — helpers for extracting canonical `File` handles from Struts `UploadedFile` objects
- Reference implementation to mirror: `BillingOnRaService.java` (existing correct `LogSafe.sanitize` usage)

**Out of scope (reserved for other issues):** `NioFileManagerImpl` ([#2213](https://github.com/carlos-emr/carlos/issues/2213)) and the `validateFileName` / `PathValidationUtils` call sites ([#2262](https://github.com/carlos-emr/carlos/issues/2262), [#2263](https://github.com/carlos-emr/carlos/issues/2263)).

### Acceptance Criteria

- [x] `LogSanitizer.java` created in `utility/` as a `@Deprecated(forRemoval = true)` shim delegating both `sanitize` overloads to `LogSafe`.
- [x] `UploadedFileUtils.java` created in `utility/` with `getUploadedFile` (throws `IllegalArgumentException` on null / no backing file) and `getUploadedFileOrNull` (returns null).
- [x] All unsanitized `EDocUtil.java` logging sites wrap filename/path variables in `LogSafe.sanitize(...)`.
- [x] No changes made to `NioFileManagerImpl` or the `validateFileName` / `PathValidationUtils` call sites.
- [x] New unit tests added for both utility classes.

---

## Reproduction Process

### Environment Setup

CARLOS provides a Docker-based dev container (Java 21, Spring 5, MariaDB), which is the recommended setup. My path:

1. Installed Docker Desktop, VS Code, and the Dev Containers extension.
2. Cloned my fork and opened it in VS Code; used "Reopen in Container" to build the dev container. First build pulled all Maven dependencies (~15-30 min).
3. **Memory tuning:** On an 8 GB Mac, the full `make install` was killed during the `checkstyle` phase (the container ran out of memory — `free -h` showed only ~1.4 GB free). Fix: skip checkstyle with the make script's built-in flag, `make install --skip-checks`. Checkstyle is a lint pass and still runs in CI, so skipping it locally is safe.
4. **Compilation succeeds:** all 4,039 source files compiled cleanly with `--skip-checks`, confirming the environment is sound and allowing direct inspection of the affected code.
5. **Known build quirk:** the final `prepare-package` (WAR deploy) step fails on an Ant task looking for `target/classes/carlos.properties`. This is a documented build peculiarity (CONTRIBUTING.md notes the raw packaging is non-trivial) and does not affect reproduction — issue #2267 is a code-level logging vulnerability verifiable in source, not a runtime bug.
6. **Git index issue:** local git commits were blocked by a Docker filesystem deadlock on the javadoc tree (Bus error / Resource deadlock avoided on `git commit`). Resolved by committing via the GitHub web editor, which commits directly to the branch without touching the local git index.

Working branch: https://github.com/Ememobong28/carlos/tree/fix-issue-2267

### Steps to Reproduce

This is a code-level security issue (log injection / log forging via unsanitized user-controlled filenames), so reproduction is confirmed by inspecting the source against the safe pattern used elsewhere in the codebase.

1. Build/compile the project in the dev container: `make install --skip-checks`.
2. Open `src/main/java/io/github/carlos_emr/carlos/documentManager/EDocUtil.java`.
3. Search for `logger.` calls that concatenate a filename/path variable directly into the message string (e.g. `logger.error("... " + fileName, e);`).
4. **Observed:** logging sites concatenate a user-controlled filename straight into the log message without sanitization — at lines **1300, 1316, 1383, 1394, 1397, 1400, 1403** on current `develop`.
5. Compare against the safe pattern in `BillingOnRaService.java`, which wraps values in `LogSafe.sanitize(...)` and uses `{}` placeholders. The `EDocUtil` sites do not.
6. **Expected (correct) behavior:** filename/path values should be passed through `LogSafe.sanitize()` before logging, consistent with the project's mandatory security rule "No PHI in logs" and "no string concatenation."

**Note on line numbers:** the original issue cited lines 1262, 1341, 1352, 1355, 1358. On current `develop` the unsanitized logging has shifted and expanded to 7 sites — the file has changed since the issue was filed. Also noted: line 1316 contains a `SecurityException` throw (not a logger call) with unsanitized filename concatenation — left out of scope per issue #2267 which targets only `logger.*` call sites.

### Reproduction Evidence

- Working branch: https://github.com/Ememobong28/carlos/tree/fix-issue-2267
- Affected file/lines: `documentManager/EDocUtil.java` lines 1300, 1316, 1383, 1394, 1397, 1400, 1403
- Reference (safe) pattern: `BillingOnRaService.java` using `LogSafe.sanitize(...)`

---

## Solution Approach

### Analysis

The root cause is that `EDocUtil.java` writes user-controlled filenames into log messages via raw string concatenation, with no sanitization. An attacker who controls a filename could inject newlines or forged log lines (log injection / forging). The codebase already has the correct tool (`LogSafe.sanitize`) and a reference usage (`BillingOnRaService.java`); `EDocUtil` simply wasn't migrated. Separately, two helper classes referenced by downstream issues (#2262, #2263) — `LogSanitizer` and `UploadedFileUtils` — don't yet exist on `develop`.

Additionally discovered during implementation: the issue's `UploadedFileUtils` snippet used `upload.getFile()`, but the actual Struts API in this codebase uses `upload.getContent()` (returning `Object`, cast to `File`). Confirmed by inspecting `AddEditDocument2Action.java`'s `resolveUploadedContentFile` method. The implementation was adjusted accordingly.

### Proposed Solution

Add the two utility classes specified in the issue, then sanitize every filename/path value logged in `EDocUtil.java` using `LogSafe.sanitize()`, mirroring the existing safe pattern. Keep strictly to the issue's scope.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** `EDocUtil` logs user-controlled filenames unsanitized at 6 sites, enabling log injection. Filenames should be sanitized via `LogSafe.sanitize()` before logging. Two missing utility classes block downstream issues.

**Match:** `BillingOnRaService.java` already logs safely with `LogSafe.sanitize(...)` and `{}` placeholders — this is the pattern to copy.

**Plan:**
1. Create `LogSanitizer.java` in `utility/` as a `@Deprecated(forRemoval = true)` shim delegating both `sanitize` overloads to `LogSafe`.
2. Create `UploadedFileUtils.java` in `utility/` with `getUploadedFile` (throws `IllegalArgumentException` on null / no backing file) and `getUploadedFileOrNull` (returns null). Use `upload.getContent()` not `upload.getFile()` — confirmed from `AddEditDocument2Action.java`.
3. In `EDocUtil.java`, wrap filename/path variables in `LogSafe.sanitize(...)` at the unsanitized logging sites.
4. Add JUnit 5 unit tests for both new utility classes matching the repo's test style (`@Tag`, `@DisplayName`, `@Nested`, AssertJ).
5. Do **not** modify `NioFileManagerImpl` or the `validateFileName` / `PathValidationUtils` call sites (reserved for #2213 / #2262 / #2263).

**Implement:** Commits on https://github.com/Ememobong28/carlos/tree/fix-issue-2267:
- `feat: add LogSanitizer transitional shim over LogSafe`
- `feat: add UploadedFileUtils for Struts UploadedFile handling`
- `fix: sanitize filename logging in EDocUtil to prevent log injection`
- `test: add unit tests for LogSanitizer`
- `test: add unit tests for UploadedFileUtils`
- `fix: add Javadoc to LogSanitizer public methods` *(addressed Gemini bot review)*
- `fix: use IllegalArgumentException for argument validation in UploadedFileUtils` *(addressed Gemini bot review)*
- `fix: update tests to expect IllegalArgumentException in UploadedFileUtilsUnitTest` *(addressed Gemini bot review)*

**Review:** Self-reviewed against CONTRIBUTING.md — Conventional Commits format, DCO sign-off on all commits, CARLOS copyright header on new files, `io.github.carlos_emr.carlos.*` package namespace, PR targets `develop`. Also addressed all three Gemini bot review comments before notifying the human maintainer.

**Evaluate:** Re-inspected all `EDocUtil` logging sites — each now uses `LogSafe.sanitize()` and `{}` placeholders. Unit tests cover null input, normal input, truncation, CRLF injection, file-backed content, and the OrNull variant.

---

## Testing Strategy

### Unit Tests

- [x] `LogSanitizer.sanitize` delegates to `LogSafe` (normal input)
- [x] `LogSanitizer.sanitize` handles null and over-length input
- [x] `LogSanitizer.sanitize` escapes CRLF characters
- [x] `UploadedFileUtils.getUploadedFile` returns backing file
- [x] `UploadedFileUtils.getUploadedFile` throws `IllegalArgumentException` on null upload
- [x] `UploadedFileUtils.getUploadedFile` throws `IllegalArgumentException` on non-file-backed content
- [x] `UploadedFileUtils.getUploadedFileOrNull` returns null when upload is null
- [x] `UploadedFileUtils.getUploadedFileOrNull` returns null when content is not file-backed

### Integration Tests

N/A — changes are pure utility additions and log call site updates with no database or Spring context dependency.

### Manual Testing

Manually verified all unsanitized `logger.*` calls in `EDocUtil.java` now use `LogSafe.sanitize()` and `{}` placeholders, matching the `BillingOnRaService.java` reference pattern.

---

## Implementation Notes

- The issue's `UploadedFileUtils` snippet called `upload.getFile()`, but the actual Struts `UploadedFile` API in this codebase uses `upload.getContent()` (returns `Object`, cast to `File`). Discovered by inspecting `AddEditDocument2Action.java`. Implementation adjusted accordingly.
- Line 1316 in `EDocUtil.java` contains a `SecurityException` throw with unsanitized filename concatenation — intentionally left out of scope as it is not a `logger.*` call site and is not covered by issue #2267.
- Local git commits were blocked by a Docker filesystem deadlock (Bus error on `git commit` due to javadoc indexing on 8 GB Mac). All commits made via GitHub web editor instead, with DCO sign-off present on all commits.
- Gemini bot review flagged three issues (wrong exception type, missing Javadoc) — all addressed and resolved before notifying the human maintainer.

---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/2975

**PR Description:**
Adds `LogSanitizer.java` (deprecated transitional shim over `LogSafe`) and `UploadedFileUtils.java` (helpers for extracting validated `File` handles from Struts `UploadedFile` objects via `PathValidationUtils.validateUploadContent()`). Removes unsanitized filename logging from `EDocUtil.java` — filenames are omitted entirely from log messages to prevent PHI leakage, per maintainer guidance.

**Status:** Phase IV — Awaiting merge (maintainer approved, content with current state)

**Maintainer Feedback:**

- **Jun 21 (Gemini bot):** Flagged wrong exception type (`IllegalStateException` → `IllegalArgumentException`), missing Javadoc on `LogSanitizer` methods, and "canonical" misnomer in `UploadedFileUtils` class Javadoc. All addressed before notifying human maintainer.

- **Jun 25 (Ben-Heerema):** Asked to review and address remaining CI bot comments. Confirmed to address PHI leakage in this PR but limit it to EDocUtil log sites already touched — omit filenames entirely rather than logging sanitized values.

- **Jun 25 (Copilot/CodeRabbit):** Multiple comments addressed:
  - Removed filename values from EDocUtil log messages entirely to prevent PHI leakage
  - Removed unused `LogSafe` import from `EDocUtil.java`
  - Delegated `UploadedFileUtils` to `PathValidationUtils.validateUploadContent()` for proper temp file validation
  - Updated Javadoc ("backing" not "canonical", added validation contract note)
  - Fixed `@since` tags to full date format (2026-06-21)
  - Removed duplicate copyright line from test file headers
  - Updated unit tests to use real temp files and expect `SecurityException` where appropriate

- **[3 days ago] (Ben-Heerema):** "this looks good to me at a glance, are you content with its current state?" — final check-in before merge consideration.
  
- **[2 days ago] (me):** Confirmed content with current state; asked for a recommendation on the next issue to pick up.
  
- **[Latest] (cubic-dev-ai):** Automated review — "No issues found across 5 files," confidence score 5/5.

## Learnings & Reflections

[Phase IV — to be completed after merge/close]

---

## Resources Used

- Issue #2267 (this contribution): https://github.com/carlos-emr/carlos/issues/2267
- Opened by maintainer Ben-Heerema; references PR #2092 (origin of the two utility classes)
- Blocked-downstream issues: [#2262](https://github.com/carlos-emr/carlos/issues/2262), [#2263](https://github.com/carlos-emr/carlos/issues/2263)
- Related out-of-scope issue: [#2213](https://github.com/carlos-emr/carlos/issues/2213)
- CARLOS repo: https://github.com/carlos-emr/carlos
- CARLOS `CONTRIBUTING.md` / README (Phase II setup reference)
- `AddEditDocument2Action.java` — confirmed `UploadedFile.getContent()` API
- `BillingOnRaService.java` — reference safe logging pattern
