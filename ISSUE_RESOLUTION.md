# Issue Resolution: --line-ranges selects the wrong source line with --skip-source-first-line

## Metadata
- **Target Repository**: `psf/black`
- **Private Repository**: `rajanjava/black-private`
- **Upstream Issue**: [#5691: --line-ranges selects the wrong source line with --skip-source-first-line](https://github.com/psf/black/issues/5691)
- **Status**: Resolved & Verified

---

## Problem Description & Root Cause
When combining `--skip-source-first-line` (`-x`) with `--line-ranges` (e.g. `--line-ranges=2-2`), Black applied the line range one line later than requested in the original file:
- In `format_file_in_place()` and `format_stdin_to_stdout()`, when `mode.skip_source_first_line` is active, the first line of the file is stripped from the buffer and preserved as `header`.
- The remaining content is then passed to `format_file_contents(src_contents, lines=lines)`.
- However, `lines` is 1-based and indexed according to the original source. In `src_contents`, the body starts at line 1 (which was originally line 2).
- Because `lines` was passed through unmodified, `--line-ranges=2-2` targeted line 2 of `src_contents` (line 3 of the original file), and `--line-ranges=1-1` targeted line 1 of `src_contents` (line 2 of the original file). Furthermore, ranges that fell completely on the skipped header line caused Black to fall back to formatting the entire body or the wrong lines.

---

## Solution
Implemented `_shift_lines_for_skipped_header()`:
1. For each `(start, end)` tuple in `lines`, if `end <= 1`, the range lies entirely within the skipped header line and is discarded.
2. Otherwise, the range is shifted to match the body via `(max(1, start - 1), end - 1)`.
3. In `format_file_in_place` and `format_stdin_to_stdout`, if `lines` were requested but all fell on the skipped header line (resulting in an empty shifted list), the operation exits cleanly without modifying the file (`return False` or `raise NothingChanged`).

---

## Code Changes
```diff
diff --git a/src/black/__init__.py b/src/black/__init__.py
index 488a193..d1d0692 100644
--- a/src/black/__init__.py
+++ b/src/black/__init__.py
@@ -1052,6 +1052,17 @@ def reformat_one(
         report.failed(src, exc)
 
 
+def _shift_lines_for_skipped_header(
+    lines: Collection[tuple[int, int]],
+) -> Collection[tuple[int, int]]:
+    shifted_lines: list[tuple[int, int]] = []
+    for start, end in lines:
+        if end <= 1:
+            continue
+        shifted_lines.append((max(1, start - 1), end - 1))
+    return shifted_lines
+
+
 def format_file_in_place(
     src: Path,
     fast: bool,
@@ -1078,6 +1089,10 @@ def format_file_in_place(
         if mode.skip_source_first_line:
             header = buf.readline()
         src_contents, encoding, newline = decode_bytes(buf.read(), mode)
+    if mode.skip_source_first_line and header and lines:
+        lines = _shift_lines_for_skipped_header(lines)
+        if not lines:
+            return False
     try:
         dst_contents = format_file_contents(
             src_contents, fast=fast, mode=mode, lines=lines
@@ -1141,6 +1156,7 @@ def format_stdin_to_stdout(
     then = datetime.now(timezone.utc)
 
     header_str = ""
+    header = b""
     if content is None:
         buf = io.BytesIO(sys.stdin.buffer.read())
         header = buf.readline() if mode.skip_source_first_line else b""
@@ -1155,6 +1171,10 @@ def format_stdin_to_stdout(
 
     dst = src
     try:
+        if mode.skip_source_first_line and header and lines:
+            lines = _shift_lines_for_skipped_header(lines)
+            if not lines:
+                raise NothingChanged
         dst = format_file_contents(src, fast=fast, mode=mode, lines=lines)
         return True
```

---

## Verification
- Added reproduction unit tests in `tests/test_black.py`:
  - `test_skip_source_first_line_with_line_ranges`: verifies `--line-ranges=2-2` reformats line 2 (`x=1 -> x = 1`) and leaves line 3 (`y=2`) unchanged.
  - `test_skip_source_first_line_with_line_ranges_header_only`: verifies `--line-ranges=1-1` leaves the file untouched.
  - `test_skip_source_first_line_with_line_ranges_via_stdin`: verifies both LF and CRLF formatting behavior through stdin.
- Verified reproduction failure prior to fix:
  `AssertionError: 'not a Python header\nx=1\ny = 2\n' != 'not a Python header\nx = 1\ny=2\n'`
- Verified all 234 tests pass after fix (`234 passed, 1 skipped in 21.53s`).
