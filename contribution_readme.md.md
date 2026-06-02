# Contribution 1: Add LogSanitizer and UploadedFileUtils utilities; sanitize filename logging in EDocUtil

**Contribution Number:** 1

**Student:** Immanuella Emem Umoren

**Issue:** https://github.com/carlos-emr/carlos/issues/2267

**Status:** Phase I — In Progress

---

## Why I Chose This Issue

This issue asks me to add two small utility classes (`LogSanitizer` and `UploadedFileUtils`) and to sanitize user-controlled filenames and paths that `EDocUtil.java` currently writes into logs via raw string concatenation, which allows log injection / log forging. It matters because it's a real security fix in a healthcare EMR that handles sensitive patient data, and because the two utility classes are blockers for two other in-progress issues (#2262 and #2263), so completing it unblocks downstream work. I chose it because it's clearly scoped with a fix pattern already present in the codebase (`LogSafe.sanitize` in `BillingOnRaService.java`), while still being challenging enough to teach me secure logging practices and how to navigate a large legacy Java/Struts project.

---

## Understanding the Issue

### Problem Description

`EDocUtil.java` logs user-controlled filenames and file paths by concatenating them directly into log message strings. Because the values aren't sanitized, an attacker could inject newlines or fake log entries (log injection / log forging). Separately, two utility classes that other security work depends on — `LogSanitizer.java` and `UploadedFileUtils.java` — exist on the PR 2092 branch but are missing from `develop`.

### Expected Behavior

Filename and path values should be passed through `LogSafe.sanitize()` before being logged, matching the existing safe-logging pattern already used elsewhere in the codebase.

### Current Behavior

Values are concatenated straight into log strings unsanitized, e.g.:
`logger.error("Error resolving file path: " + fileName, e);`

### Affected Components

- `src/main/java/io/github/carlos_emr/carlos/documentManager/EDocUtil.java` — lines 1262, 1341, 1352, 1355, 1358
- New file: `src/main/java/io/github/carlos_emr/carlos/utility/LogSanitizer.java`
- New file: `src/main/java/io/github/carlos_emr/carlos/utility/UploadedFileUtils.java`

**Out of scope (reserved for other issues):** `NioFileManagerImpl` (#2213) and the `validateFileName` / `PathValidationUtils` call sites (#2262, #2263).

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

- Issue #2267: https://github.com/carlos-emr/carlos/issues/2267
- CARLOS repo: https://github.com/carlos-emr/carlos
