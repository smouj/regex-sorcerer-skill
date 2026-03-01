---
name: regex-sorcerer
version: 1.2.0
description: Master regex patterns to transform and manipulate text with magical precision
author: OpenClaw Team
tags:
  - regex
  - text
  - manipulation
  - automation
  - pattern
  - matching
  - refactoring
  - search-replace
input:
  - text: The text content to process (stdin or file)
  - pattern: Regex pattern to apply
  - flags: Optional regex flags (g, i, m, s, u, y)
  - operation: One of [match, replace, extract, split, validate, count, test]
  - replacement: Replacement string or transformation template
  - multiline: Boolean for multi-line processing mode
output:
  - result: Processed text output
  - matches: Array of found matches with positions
  - count: Number of matches found
  - changed: Boolean indicating if text was modified
  - statistics: Timing and performance metrics
  - errors: Array of validation errors
environment:
  - REGEX_MAX_MATCHES: Maximum matches to return (default 1000)
  - REGEX_TIMEOUT: Timeout in milliseconds (default 5000)
  - REGEX_MAX_FILE_SIZE: Max file size in MB (default 100)
  - REGEX_DEBUG: Enable debug logging (default false)
  - REGEX_BACKUP_EXT: Backup file extension (default .regex-bak)
dependencies:
  - name: grep
    required: true
    version: ">=2.0"
  - name: sed
    required: true
    version: ">=4.0"
  - name: perl
    required: false
    version: ">=5.0"
  - name: ripgrep (rg)
    required: false
    version: ">=13.0"
  - name: jq
    required: false
    version: ">=1.6"
---

# Regex Sorcerer

## Purpose

Regex Sorcerer provides surgical precision for text transformation, bulk data processing, code refactoring, log analysis, and automated text cleanup tasks. It enables complex pattern-based operations that would otherwise require verbose imperative code or manual editing.

**Real use cases:**
- Extract email addresses and phone numbers from unstructured logs
- Rename variables across an entire codebase using capture groups
- Convert CSV/TSV formats with field reordering
- Remove duplicated whitespace, normalize line endings, clean formatting
- Validate configuration files against pattern requirements
- Find TODO/FIXME comments across multiple files
- Generate structured reports from semi-structured text
- Transcode date formats (MM/DD/YYYY to ISO 8601)
- Mask sensitive data (credit cards, SSNs) for deidentification
- Perform multi-file search-and-replace with safety guards

## Scope

### Supported Operations

- **match**: Find all regex matches with capture groups and positions
- **replace**: Replace patterns with replacement string (supports $1-$9 backreferences)
- **extract**: Pull out capture groups into structured format (JSON/CSV)
- **split**: Split text on regex delimiter into array
- **validate**: Check if entire text matches pattern, return boolean
- **test**: Quick boolean check for pattern existence
- **count**: Count pattern occurrences without extracting
- **analyze**: Generate pattern analysis (complexity, AST, common pitfalls)

### Command Forms

```bash
# Match mode (find all matches)
regex-sorcerer match "pattern" < input.txt
regex-sorcerer match -i -g "pattern" < input.txt

# Replace mode (transform text)
regex-sorcerer replace "pattern" "replacement" < input.txt > output.txt
regex-sorcerer replace -i "pattern" "replacement" < input.txt
echo "text" | regex-sorcerer replace -s "\s+" " "  # squeeze whitespace

# Extract mode (get capture groups)
regex-sorcerer extract "pattern" --group=1 --format=json < input.txt
regex-sorcerer extract -g "(\w+).*(\d+)" -f csv < data.txt

# Split mode
regex-sorcerer split "pattern" < input.txt
regex-sorcerer split -l -i ",\s*" < data.txt

# Validate mode
regex-sorcerer validate "pattern" < file.txt
if regex-sorcerer validate "^\d{5}-\d{4}$"; then echo "Valid ZIP"; fi

# Test mode (boolean only)
if regex-sorcerer test "ERROR" < log.txt; then echo "Has errors"; fi

# Count mode
regex-sorcerer count "pattern" < bigfile.log
regex-sorcerer count -i -t "^\w+@\w+\.\w+" < emails.txt

# Analyze mode
regex-sorcerer analyze "pattern"
```

### Flags

- `-i`, `--ignore-case`: Case-insensitive matching
- `-g`, `--global`: Find all matches (default: first only for some ops)
- `-m`, `--multiline`: Treat string as multiple lines (^/$ match per line)
- `-s`, `--dotall`: Dot matches newline (single-line mode)
- `-u`, `--unicode`: Enable Unicode mode
- `-l`, `--line`: Process line-by-line (for split/validate/test)
- `-t`, `--total`: For count, return number instead of verbose output
- `-j`, `--json`: Output results in JSON format
- `-f`, `--format`: Output format: json, csv, lines, raw (for extract)
- `--group=N`: Extract specific capture group (default: all)
- `--max=N`: Limit number of results
- `--timeout=N`: Override REGEX_TIMEOUT for this run
- `--backup`: Create .bak backup before modifying files
- `--dry-run`: Show changes without applying (replace mode only)
- `--line-numbers`: Include line numbers in output
- `--positions`: Include start/end positions in match output
- `--named`: Use named groups as keys in JSON output

## Work Process

### Phase 1: Pattern Validation
1. Parse pattern syntax using PCRE2 (Perl Compatible Regular Expressions)
2. Detect common pitfalls: nested quantifiers, catastrophic backtracking
3. Verify pattern compiles without errors
4. If `REGEX_DEBUG=true`, output AST to stderr

### Phase 2: Input Acquisition
1. Read from stdin OR file path argument
2. Check file size against `REGEX_MAX_FILE_SIZE`
3. Detect encoding (UTF-8 default), convert if needed
4. For `--backup` mode: copy original to `{path}{REGEX_BACKUP_EXT}`

### Phase 3: Execution (per operation)
- **match**: Run `re2::RE2::FindAndConsume` in loop until exhaustion or `REGEX_MAX_MATCHES`
- **replace**: Stream-process line-by-line; for single-line mode replace globally
- **extract**: Collect all capture groups, build structured output
- **split**: Use `re2::RE2::Split` into vector of strings
- **validate**: Check `re2::RE2::FullMatch` returns true
- **test**: Check `re2::RE2::PartialMatch` returns true
- **count**: Increment counter for each match found
- **analyze**: Run static analysis on pattern string

### Phase 4: Output Formatting
1. Format according to `--format` flag
2. For JSON: escape strings, include positions if requested
3. For CSV: quote fields containing commas/newlines
4. Write to stdout; errors/warnings to stderr
5. Exit code:
   - 0: Success (matches found for match/test; validation passed for validate)
   - 1: No matches found OR validation failed
   - 2: Pattern syntax error
   - 3: Input error (file not found, too large, encoding)
   - 4: System error (memory, timeout)

### Phase 5: Post-Processing
- For replace with `--backup`: retain backup until user verifies
- For multi-file operations: continue on per-file errors, report summary

## Golden Rules

1. **Always backup before destructive replace**: Use `--backup` when modifying files in-place. Never operate on original without backups.
2. **Test on small sample first**: Run matching on a tiny sample before full file replace. Use `regex-sorcerer test` for quick validation.
3. **Respect line boundaries**: Unless using `-m`/`-s`, patterns operate per line. Multi-line patterns require explicit flags.
4. **Escape properly**: In shell, single-quote patterns to avoid shell interpretation. Example: `'^\d{3}-\d{4}$'` not `"^\d{3}-\d{4}$"`
5. **Use non-greedy via `?`**: Default greedy matching causes errors. Explicitly use `*?` or `+?` when needed.
6. **Handle Unicode**: Use `-u` for non-ASCII text; ensure terminal encoding matches.
7. **Limit matches**: Always use `--max` for large files to avoid OOM. Default limit 1000 enforced.
8. **Validate replacements**: Use `--dry-run` to preview replace changes before committing.
9. **Check exit codes**: Scripts must test `$?` after regex-sorcerer calls. Don't assume success.
10. **PCRE2 only**: This tool uses RE2 (Google's regex library). No support for backreferences in lookbehind, conditionals, or recursion.
11. **No variable-length lookbehind**: RE2 requires fixed-width lookbehind. Pattern `(?<=\d{2,})` invalid.
12. **Capture group references**: In replacement, use `$1` not `\1`. For literal `$`, escape as `$$`.
13. **Performance timeout**: `REGEX_TIMEOUT` prevents regex DoS. Complex patterns on large input may timeout; simplify pattern if needed.

## Examples

### Example 1: Extract email addresses from logs
**Input** (`access.log`):
```
127.0.0.1 - - [01/Mar/2026:10:00:00] "GET / HTTP/1.1" 200 1234 - user@example.com
192.168.1.1 - - [01/Mar/2026:10:01:00] "POST /api HTTP/1.1" 404 567 admin@test.org
```

**Command**:
```bash
regex-sorcerer extract -i "([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})" --named -f json < access.log
```

**Output** (JSON):
```json
[
  {"email": "user@example.com", "line": 1},
  {"email": "admin@test.org", "line": 2}
]
```

**Purpose**: Bulk extraction of contact data from server logs for analytics.

---

### Example 2: Refactor JavaScript variable names
**Input** (`legacy.js`):
```javascript
var old_count = 0;
function increment_old_count() {
    old_count++;
    console.log("Count is: " + old_count);
}
```

**Command**:
```bash
regex-sorcerer replace -g "old_count" "newCount" --backup < legacy.js > modern.js
```

**Output** (`modern.js`):
```javascript
var newCount = 0;
function increment_newCount() {
    newCount++;
    console.log("Count is: " + newCount);
}
```

**Dry-run preview**:
```bash
regex-sorcerer replace -g "old_count" "newCount" --dry-run < legacy.js
```
Shows diff-style output: `-var old_count = 0;` `+var newCount = 0;`

---

### Example 3: Convert date format in CSV
**Input** (`dates.csv`):
```
John,01/15/2025,Active
Jane,02/28/2024,Inactive
```

**Command**:
```bash
regex-sorcerer replace -g "(\d{2})/(\d{2})/(\d{4})" "$3-$1-$2" < dates.csv
```

**Output**:
```
John,2025-01-15,Active
Jane,2024-02-28,Inactive
```

**Verification**:
```bash
regex-sorcerer match -g "\d{4}-\d{2}-\d{2}" < dates.csv | wc -l  # Should be 2
```

---

### Example 4: Find all TODOs in codebase
**Shell script**:
```bash
find . -name "*.py" -exec sh -c '
  echo "File: $1"
  regex-sorcerer match -i -g --line-numbers "TODO|FIXME|XXX" < "$1" | head -20
' sh {} \;
```

---

### Example 5: Mask credit card numbers
**Input** (`transactions.txt`):
```
Visa 4111-1111-1111-1111 $99.99
MasterCard 5555-5555-5555-4444 $149.99
Amex 3782-822463-10005 $79.99
```

**Command**:
```bash
regex-sorcerer replace -g "(\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4})" "****-****-****-$4" < transactions.txt
```

**Output**:
```
Visa ****-****-****-1111 $99.99
MasterCard ****-****-****-4444 $149.99
Amex ****-****-****-10005 $79.99
```

---

### Example 6: Validate IPv4 addresses
**Command**:
```bash
while read ip; do
  if regex-sorcerer validate "^(?:[0-9]{1,3}\.){3}[0-9]{1,3}$" <<<"$ip" && \
     regex-sorcerer test "^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$" <<<"${ip##*.}"; then
    echo "$ip valid"
  else
    echo "$ip INVALID" >&2
  fi
done < ips.txt
```

**Pattern explanation**: First regex checks format `x.x.x.x`, second ensures each octet 0-255.

---

## Rollback Commands

### For single file replacement with `--backup`
```bash
# Restore original file
cp input.txt{REGEX_BACKUP_EXT} input.txt
# Or if backup is .regex-bak
cp input.txt.regex-bak input.txt
```

### For batch operations without explicit backup
**If you miscalculated replacement**:
```bash
# Use diff to see what changed
diff -u <(regex-sorcerer replace "old" "new" < input.txt) < input.txt
# Then manually reconstruct or use inverse pattern
regex-sorcerer replace "new" "old" < input.txt > input.txt.fixed
mv input.txt.fixed input.txt
```

### Undo a `split` operation
**Original**: comma-separated file
**Command**: `regex-sorcerer split "," < data.csv > parts.txt`
**Rollback**: paste back together (requires saved delimiter)
```bash
# If you used `regex-sorcerer split ","` you need to re-join:
paste -d',' parts.txt > restored.csv  # if exactly 2 fields
# Or programmatically:
awk '{print $1","$2","$3}' parts.txt > restored.csv
```

### Undo `extract` to reconstruct partial file
Extract only emails, lost other data. **No automatic rollback**. Must restore from original source. Always extract to new file (not overwrite) and keep original.

### Cleanup after failed batch operation
```bash
# Find all backups created in directory tree
find . -name "*.regex-bak" -exec sh -c '
  f="{}"
  echo "Restoring $f to ${f%.regex-bak}"
  cp "$f" "${f%.regex-bak}"
' \;
```

### Version control rollback (preferred)
If files are in git:
```bash
# See what changed
git diff --color
# Revert specific file
git checkout -- path/to/file.txt
# Or revert all changes in directory
git checkout -- directory/
```

### Log of operations for forensic recovery
When `REGEX_DEBUG=true`, regex-sorcerer logs to stderr:
```bash
regex-sorcerer replace ... 2>> /var/log/regex-ops.log
# Later review:
grep "pattern.*replacement" /var/log/regex-ops.log | tail -50
```

## Dependencies & Requirements

### System Packages
- **re2-dev** or equivalent (linked dynamically)
- Standard C++ runtime (libstdc++)
- POSIX utilities: grep, sed, awk, cut

### Optional Performance Enhancers
- `ripgrep (rg)` for faster file iteration when used in scripts
- `perl` for advanced PCRE features not in RE2 (use `--perl` flag when available)
- `jq` for JSON output manipulation

### Memory & CPU
- Minimum 50MB RAM per active process
- Timeout default 5000ms per pattern; increase via `REGEX_TIMEOUT`
- Parallel execution: use GNU parallel with care (no shared state)

### Security Constraints
- Pattern input is sanitized against ReDoS attacks via timeout
- File operations respect user permissions; no privilege escalation
- No network access initiated by tool
- All I/O is local; remote protocols require user shell redirection

### Supported Platforms
- Linux (x86_64, ARM64)
- macOS (Intel, Apple Silicon)
- FreeBSD (with compatible re2 library)
- Windows (via WSL or native port; path handling differs)

---

## Troubleshooting

### Pattern compiles but matches wrong text
- Cause: Greedy quantifiers `.*` consume too much.
- Fix: Use non-greedy `.*?` or explicit negated character class `[^"]*`.
- Debug: Use `regex-sorcerer analyze "pattern"` to see complexity.

### No matches found when expecting
- Check `-i` flag for case sensitivity
- Verify pattern anchors: `^` matches start of string, not line unless `-m`
- Test with `regex-sorcerer test` first
- Use `--positions` to see what region was scanned

### "catastrophic backtracking" warning in analyze
- Pattern contains nested quantifiers like `(a+)+`
- Rewrite as atomic group `(?>(a+)+)` or different pattern
- Input length matters: limit with `--max` or use more specific pattern

### Unicode characters don't match
- Add `-u` flag
- Ensure terminal/file is UTF-8 (check with `file -I filename`)
- Use `\p{L}` for any letter across scripts

### Replacement `$1` becomes literal "$1"
- Cause: Using single quotes vs double quotes in shell. Shell removes `$`.
- Fix: Use single quotes around pattern BUT double quotes around replacement OR escape `$` as `\$1` in single-quoted string.
- Example: `regex-sorcerer replace '(\w+)' "\$1_processed"` OR `'(\w+)' '\$1_processed'` (RE2 uses `$1` not `\1`)

### "Pattern too large" error
- RE2 limits installed size (often 1-4MB). Simplify pattern or split into multiple steps.

### Timeout on large file
- Increase timeout: `REGEX_TIMEOUT=30000 regex-sorcerer ...`
- Reduce input size by pre-filtering (e.g., `grep "suspicious" big.log | regex-sorcerer ...`)
- Break into chunks with `split` command

### Exit code 2 but pattern looks correct
- Invalid escape: `\d` not enabled? RE2 supports `\d`, `\w`, `\s` by default. Ensure no stray backslash.
- Unbalanced parentheses: count `(` vs `)`
- Use `perl -e 'pattern'` to test standard compliance

### `--backup` fails: Permission denied
- Destination directory read-only or owned by different user
- Run with appropriate permissions or write to different location

### Output format wrong
- `-f` only affects `extract` mode. For `match`/`replace` use `-j` for JSON.
- `--named` requires regex to have `(?P<name>...)` groups; otherwise numeric keys used.

### "No matches" but human sees text
- Trailing newline missing: `^` may not match beginning if no newline at start
- Use `\A` for absolute start of string, `\z` for absolute end (PCRE)
- RE2 supports `\A` and `\z` as anchors

### Multi-line flag confusion
- `-m`: `^` and `$` match after/before newline inside string
- `-s`: `.` matches newline
- Without either, pattern doesn't cross line boundaries

---

## Performance Tips

1. **Pre-filter with grep** before regex-sorcerer for large datasets:
   ```bash
   grep -l "error" *.log | xargs regex-sorcerer match "ERROR \d{4}"
   ```

2. **Use atomic groups** to prevent backtracking on ambiguous patterns:
   Pattern `(a+)+b` → `(?>a+)+b`

3. **Anchor patterns** to avoid full scans when possible:
   `^ERROR:` only scans beginning of each line, much faster.

4. **Compile once, run many**: For repetitive use, wrap in script that reads pattern once.

5. **Avoid `.` in strict contexts**: `[^"]*` is faster than `.*?` inside quotes.

6. **Parallelize with caution**: Use GNU parallel but ensure each invocation independent:
   ```bash
   parallel --halt soon,fail=1 regex-sorcerer match "pattern" < {} ::: files/*
   ```
   Stops after first failure to avoid cascading errors.

---

## Limitations

- RE2 engine does not support backreferences in lookahead/lookbehind
- No conditional subpatterns `(?(condition)yes|no)`
- No subroutine calls or recursion
- Named capture groups syntax: `(?P<name>...)` only, not `(?<name>...)`
- Maximum pattern size: ~1-4MB depending on compile-time constants
- Maximum match vector: limited by memory but typically ~65536 submatches

## Version History

- **1.2.0** (2026-03-01): Added `--analyze`, improved error messages, JSON streaming
- **1.1.0** (2025-11-15): Added `--dry-run`, backup management
- **1.0.0** (2025-06-01): Initial release with match/replace/extract/validate

---

**Last updated**: 2026-03-01
**Maintainer**: OpenClaw Core Team
```