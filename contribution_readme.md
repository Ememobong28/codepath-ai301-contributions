# Contribution 1: Add LogSanitizer and UploadedFileUtils utilities; sanitize filename logging in EDocUtil

**Contribution Number:** 1

**Student:** Immanuella Emem Umoren

**Issue:** https://github.com/carlos-emr/carlos/issues/2267

**My Fork:** https://github.com/Ememobong28/carlos

**Status:** Phase I — Complete

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

[Phase II]

### Steps to Reproduce

[Phase II]

### Reproduction Evidence

[Phase II]

---

## Solution Approach

### Analysis

[Phase II/III]

### Proposed Solution

[Phase II/III]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Phase III]

**Match:** The codebase already uses `LogSafe.sanitize(...)` in `BillingOnRaService.java` — this is the pattern to mirror.

**Plan:**
1. Create `LogSanitizer.java` in `utility/` as a deprecated shim delegating to `LogSafe`.
2. Create `UploadedFileUtils.java` in `utility/` with `getUploadedFile` and `getUploadedFileOrNull`.
3. Wrap filename/path variables in `LogSafe.sanitize(...)` at the five named `EDocUtil.java` lines.
4. Add unit tests for both utility classes.

**Implement:** [Phase III — branch/commit links]

**Review:** [Phase III]

**Evaluate:** [Phase III]

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
