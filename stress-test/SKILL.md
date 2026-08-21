---
name: stress-test
description: Break features on purpose — generate and run edge case batteries against CLI commands, API endpoints, and DB writes. Auto-discovers what changed, builds attack vectors (XSS, injection, boundary values, type confusion, auth bypass, race conditions), runs them against production, reports pass/fail, and cleans up test artifacts. Use after shipping a feature or before a client-ready check. Pass a feature name, CLI command, or API endpoint as argument (e.g., "iris content", "/api/v1/my/profiles", "upload flow").
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# Stress Test — Break It Before Clients Do

Generate and execute edge case batteries against CLI commands, API endpoints, and database writes. The goal is to find bugs through adversarial input, boundary conditions, and unexpected usage patterns — the same things real users will do accidentally.

## Arguments

`$ARGUMENTS` — What to test. Examples:

- `/stress-test iris content` — test all `iris content` subcommands
- `/stress-test /api/v1/my/profiles` — test a specific API endpoint
- `/stress-test upload flow` — test the upload workflow end-to-end
- `/stress-test <feature>` — auto-discover commands and endpoints from recent commits

## How It Works

### Phase 1: Discovery

Identify what to test by examining:

1. **Recent commits** — `git log --oneline -5` + `git diff --name-only HEAD~3`
2. **CLI commands** — grep for `cmd({` patterns, extract command names and positional args
3. **API endpoints** — grep for `irisFetch`, `Route::get/post`, extract URL patterns
4. **DB writes** — grep for `::create`, `->update`, `->delete`, `POST /api`, `PUT /api`, `DELETE /api`

```bash
# Auto-discover from recent changes
CHANGED_FILES=$(git diff --name-only HEAD~3 2>/dev/null | head -20)

# Find CLI commands in changed files
echo "$CHANGED_FILES" | xargs grep -l "cmd({" 2>/dev/null

# Find API endpoints in changed files
echo "$CHANGED_FILES" | xargs grep -oh "irisFetch(['\"]\/api[^'\"]*" 2>/dev/null | sort -u

# Find DB mutations
echo "$CHANGED_FILES" | xargs grep -n "::create\|->update\|->delete\|->save" 2>/dev/null | head -10
```

### Phase 2: Attack Vector Generation

For each discovered target, generate test cases from these categories:

#### Category 1: Input Boundary Testing

| Vector | What it tests | Example |
|--------|--------------|---------|
| Empty string | Null/empty handling | `iris content get ""` |
| Zero | Off-by-one, division | `--profile 0`, `--limit 0` |
| Negative numbers | Unsigned assumptions | `iris content get -1` |
| Very large numbers | Integer overflow | `iris content get 999999999999` |
| Max length strings | Buffer/truncation | `--title "$(python3 -c "print('A'*10000)")"` |
| Unicode/emoji | Encoding issues | `--search "日本語🔥"` |
| Null bytes | C-string termination | `--title $'\x00hidden'` |
| Whitespace only | Trim failures | `--search "   "` |
| Special URL chars | Encoding issues | `--search "a&b=c?d#e"` |

#### Category 2: Security Testing

| Vector | What it tests | Example |
|--------|--------------|---------|
| XSS in text fields | HTML injection | `--title '<script>alert(1)</script>'` |
| SQL injection | Parameterized queries | `--search "'; DROP TABLE users;--"` |
| Path traversal | File access | `--profile "../../etc/passwd"` |
| Command injection | Shell escaping | `--title "$(whoami)"`, `` --title "`id`" `` |
| Auth bypass | Token handling | Call endpoint without auth header |
| IDOR | Object ownership | Access another user's content by ID |
| Rate limiting | Abuse prevention | 20 rapid sequential calls |

#### Category 3: Type Confusion

| Vector | What it tests | Example |
|--------|--------------|---------|
| String where number expected | Type coercion | `iris content get "abc"` |
| Number where string expected | Type coercion | `--search 12345` |
| Boolean-ish strings | Truthy/falsy | `--profile "false"`, `--profile "null"` |
| Array-like input | Parser confusion | `--type "video,article"` |
| JSON in string field | Injection | `--title '{"key":"value"}'` |

#### Category 4: State & Concurrency

| Vector | What it tests | Example |
|--------|--------------|---------|
| Duplicate create | Idempotency | Upload same URL twice |
| Delete then access | Ghost references | Delete video, then `get` it |
| Modify then diff | Consistency | Push changes, immediately diff |
| Rapid sequential ops | Race conditions | 5 uploads in parallel |
| Stale local file | Sync conflicts | Modify local JSON, API changes independently |

#### Category 5: UX Flow Testing

| Vector | What it tests | Example |
|--------|--------------|---------|
| First-run experience | Empty state | Run command with no data |
| Ctrl+C mid-operation | Cleanup | Interrupt during upload |
| --json piping | Machine-readable output | `cmd --json 2>/dev/null \| python3 -c "import json,sys;json.load(sys.stdin)"` |
| --help completeness | Self-documentation | Every subcommand has --help |
| Error message quality | User guidance | Error says what to do next |
| Non-interactive mode | CI/CD compat | Pipe input, no TTY |

#### Category 6: User Journey Paths (THE IMPORTANT ONE)

This is the category that catches "it works in isolation but the workflow is broken." Walk through the paths a real client takes — in order, with real data, checking that each step's output feeds correctly into the next.

**How to build journey tests**: For each feature, map the 3-5 most common user workflows. Each journey is a sequence of commands where the output of step N informs the input of step N+1. If any step fails OR produces output that can't feed the next step, the journey fails.

##### Journey Templates

**Journey A: Discovery → First Use → Daily Use**
```bash
# Step 1: User discovers the feature
iris <command> --help
# VERIFY: output is clear, shows examples, no jargon

# Step 2: User tries it with no context
iris <command>
# VERIFY: helpful empty state OR guided first-run, NOT a crash or silent empty

# Step 3: User tries the most basic operation
iris <command> <simplest-args>
# VERIFY: produces output they can understand and act on

# Step 4: User tries the most common real operation
iris <command> <realistic-args>
# VERIFY: succeeds, output includes actionable info (URLs, IDs, next steps)

# Step 5: User comes back tomorrow — is the output still useful?
iris <command> <same-args>
# VERIFY: idempotent or shows updated data, not a stale cache
```

**Journey B: Create → Verify → Modify → Delete (full lifecycle)**
```bash
# Step 1: Create something
iris <command> create <args>
# CAPTURE: the ID/URL from output

# Step 2: Verify it exists
iris <command> get <captured-id>
# VERIFY: record matches what was created

# Step 3: Pull to local, modify, push back
iris <command> pull <id>
# Modify the local file
iris <command> diff <id>
# VERIFY: diff shows exactly what changed
iris <command> push <id>
# VERIFY: changes reflected on get

# Step 4: Delete
iris <command> delete <id>
# VERIFY: get returns 404

# Step 5: Re-create same thing (idempotency)
iris <command> create <same-args>
# VERIFY: succeeds (no ghost references blocking)
```

**Journey C: Error Recovery**
```bash
# Step 1: Try with wrong credentials / expired auth
# VERIFY: clear error message with fix instructions

# Step 2: Try with a resource that doesn't exist
# VERIFY: 404 message with helpful suggestion (e.g., "Run iris <list> to see available items")

# Step 3: Try with a partially valid input (e.g., valid URL but wrong profile)
# VERIFY: error identifies WHICH part is wrong, not a generic "request failed"

# Step 4: Interrupt mid-operation (Ctrl+C / timeout)
# VERIFY: no orphaned records, no corrupt local state, can retry cleanly
```

**Journey D: Power User Multi-Step**
```bash
# Step 1: List available items
iris <command> list
# CAPTURE: use first item's ID

# Step 2: Get details
iris <command> get <id-from-step-1> --json
# CAPTURE: extract a field from JSON output

# Step 3: Use captured data in another command
iris <other-command> <field-from-step-2>
# VERIFY: cross-command data flow works

# Step 4: Pipe JSON output to external tool
iris <command> get <id> --json 2>/dev/null | jq '.title'
# VERIFY: JSON is valid, fields are predictable, piping works
```

**Journey E: Onboarding (Brand New User)**
```bash
# Step 1: Fresh install — what does help show?
iris help | grep <feature>
# VERIFY: feature is visible in top-level help

# Step 2: Try the feature with zero setup
iris <command> <args>
# VERIFY: either works OR tells them exactly what setup is needed

# Step 3: Follow the error message's instructions
<run whatever the error message suggested>
# VERIFY: after following instructions, original command now works

# Step 4: Second run — is it faster/easier?
iris <command> <same-args>
# VERIFY: no repeated setup, cached state works
```

##### Journey Test Execution Pattern

```bash
# Journey test template
run_journey() {
  local name="$1"
  echo ""
  echo "  --- Journey: $name ---"
  shift
  local step=1
  local CAPTURED=""
  local JOURNEY_PASS=true

  while [ $# -gt 0 ]; do
    local desc="$1"; local cmd="$2"; local check="$3"
    shift 3

    echo -n "    Step $step ($desc): "
    OUTPUT=$(eval "$cmd" 2>&1)
    EXIT=$?

    case "$check" in
      pass)       [ $EXIT -eq 0 ] && echo "PASS" || { echo "FAIL (exit=$EXIT)"; JOURNEY_PASS=false; } ;;
      grep:*)     echo "$OUTPUT" | grep -q "${check#grep:}" && echo "PASS" || { echo "FAIL"; JOURNEY_PASS=false; } ;;
      capture:*)  CAPTURED=$(echo "$OUTPUT" | grep -o "${check#capture:}" | head -1); echo "PASS (captured: $CAPTURED)" ;;
      no-crash)   [ $EXIT -le 1 ] && echo "PASS" || { echo "FAIL (crash: exit=$EXIT)"; JOURNEY_PASS=false; } ;;
    esac

    step=$((step+1))
  done

  $JOURNEY_PASS && echo "    JOURNEY: PASS" || echo "    JOURNEY: FAIL"
}
```

##### Persona-Based Journey Matrix

Every feature must be tested from **5 personas**. Each persona has different context, expectations, and failure modes:

**Persona 1: New User (Day 1)**
- Just ran `iris init`, has auth but no content/profiles
- Expectation: guided experience, no jargon, clear next steps
- Common failures: empty state crashes, "no data" with no guidance, assumes config exists
```bash
# Simulate: what does a new user see?
# 1. Can they find the feature?
iris help 2>&1 | grep <feature>
# 2. What happens with zero data?
iris <command> list 2>&1          # Should show "No items yet. Create one: iris <command> create"
# 3. What happens when they try to use it without setup?
iris <command> <action> 2>&1      # Should guide them, not crash
# 4. After following instructions, does it work?
<follow the error message's suggestion>
iris <command> <action> 2>&1      # Should now succeed
```

**Persona 2: Existing User (Daily Driver)**
- Has profiles, content, active workflows
- Expectation: fast, reliable, remembers context
- Common failures: slow queries on large datasets, stale cache, pagination breaks
```bash
# Simulate: daily workflow
# 1. List my stuff quickly (should be <3s)
time iris <command> list 2>&1
# 2. Get a specific item I know exists
iris <command> get <known-id> 2>&1
# 3. Upload/create new content (the main action they do)
iris <command> upload <url> --profile <name> 2>&1
# 4. Verify it shows up in list immediately (no cache lag)
iris <command> list 2>&1 | grep <new-item>
# 5. Share it (get the public URL, verify it works)
curl -sI <public_url> | grep "200\|301"
```

**Persona 3: Admin/Operator (Managing for Clients)**
- Managing content across multiple profiles/brands
- Expectation: bulk operations, cross-profile access, --json for scripting
- Common failures: user scoping blocks admin access, no bulk operations, ID confusion between profiles
```bash
# Simulate: admin workflow
# 1. List ALL profiles (not just mine)
iris <command> profiles list 2>&1
# 2. Switch context to a client's profile
iris <command> list --profile <client-pk> 2>&1
# 3. Upload content to client's profile
iris <command> upload <url> --profile <client-pk> 2>&1
# 4. Bulk export for reporting
iris <command> list --profile <pk> --json 2>/dev/null | python3 -c "import json,sys; print(len(json.load(sys.stdin)))"
# 5. Cross-profile search
iris <command> search "keyword" 2>&1
```

**Persona 4: CI/Automation (Non-Interactive)**
- Running in a script, cron job, or Hive task
- Expectation: no prompts, predictable exit codes, parseable output
- Common failures: interactive prompts block execution, ANSI codes in piped output, non-zero exit on success
```bash
# Simulate: CI pipeline
# 1. All commands work without TTY
echo "" | iris <command> list --json 2>/dev/null | python3 -c "import json,sys;json.load(sys.stdin)" && echo "JSON OK"
# 2. Exit codes are meaningful
iris <command> get 14128 2>/dev/null; echo "exit: $?"       # Should be 0
iris <command> get 99999 2>/dev/null; echo "exit: $?"       # Should be non-zero
# 3. No interactive prompts in non-TTY
echo "n" | iris <command> delete 14128 2>&1                  # Should not hang
# 4. --json suppresses all non-JSON output to stderr
iris <command> get 14128 --json 2>/dev/null | head -1 | grep "^{" && echo "Clean JSON"
# 5. Errors go to stderr, data to stdout
iris <command> get 99999 --json 2>/dev/null; echo "stdout clean: $?"
```

**Persona 5: Adversarial User (Pentester/Abuse)**
- Trying to break things, access other users' data, inject payloads
- Expectation: every input sanitized, auth enforced, no 500s
- Common failures: XSS stored, IDOR, SQL injection, command injection, path traversal
```bash
# This is Categories 1-3 from above, but framed as a user intent:
# "What if someone TRIES to break this?"
# See Category 2 (Security) for the full battery
```

##### What Makes a Good Journey Test

| Good | Bad |
|------|-----|
| Tests a sequence of 3-5 related commands | Tests isolated commands (that's Category 1-5) |
| Output of step N feeds step N+1 | Steps are independent |
| Uses real production data | Uses mocked/fake data |
| Tests the path a CLIENT would take | Tests the path a DEVELOPER would take |
| Includes at least one error/recovery step | Only tests the happy path |
| Verifies cleanup (no orphans) | Leaves test data behind |

### Phase 3: Execution

Run each test case and collect results:

```bash
# Template for each test
run_test() {
  local name="$1"
  local cmd="$2"
  local expect="$3"  # "pass" (exit 0), "fail" (exit non-0), "grep:pattern" (output contains)

  echo -n "  $name: "
  OUTPUT=$(eval "$cmd" 2>&1)
  EXIT=$?

  case "$expect" in
    pass) [ $EXIT -eq 0 ] && echo "PASS" || echo "FAIL (exit=$EXIT)" ;;
    fail) [ $EXIT -ne 0 ] && echo "PASS" || echo "FAIL (expected non-zero)" ;;
    grep:*) echo "$OUTPUT" | grep -q "${expect#grep:}" && echo "PASS" || echo "FAIL (pattern not found)" ;;
    no-500) echo "$OUTPUT" | grep -q "500" && echo "FAIL (500 error)" || echo "PASS" ;;
  esac
}
```

### Phase 4: Cleanup

After all tests:
1. Delete any test records created (videos, profiles, etc.)
2. Restore any modified records (push original JSON back)
3. Verify no test artifacts remain in production
4. Report any artifacts that couldn't be cleaned

```bash
# Track created IDs for cleanup
CREATED_IDS=()

# After tests
for id in "${CREATED_IDS[@]}"; do
  iris content delete "$id" --force 2>/dev/null
done
```

### Phase 5: Report

Output format:
```
============================================
  STRESS TEST REPORT — <feature>
============================================

  Category 1 (Boundaries):    8/8 pass
  Category 2 (Security):      6/7 pass  [!] XSS in description field
  Category 3 (Type Confusion): 5/5 pass
  Category 4 (State):         4/4 pass
  Category 5 (UX Flow):       6/6 pass

  TOTAL: 29/30 pass (96.7%)

  FAILURES:
    - [Security] XSS in description: <script> tag stored in DB
      Fix: strip HTML in CLI before sending

  CLEANUP:
    - Deleted test videos: #14131, #14132
    - Restored original: video #14128 description
    - No orphaned artifacts

  VERDICT: WARN — 1 security issue needs fix before ship
============================================
```

## Pre-Built Test Suites

### CLI Command Suite

For any `iris <command>`:

```bash
# 1. Help exists
iris <command> --help 2>&1 | grep -q "describe\|usage\|Options"

# 2. No args (should show help or graceful error)
iris <command> 2>&1  # should not crash

# 3. --json output is valid JSON
iris <command> <valid-args> --json 2>/dev/null | python3 -c "import json,sys;json.load(sys.stdin)"

# 4. Non-existent resource
iris <command> get 99999999 2>&1  # should be 404, not 500

# 5. Empty/invalid args
iris <command> get "" 2>&1       # should be graceful
iris <command> get -1 2>&1       # should be graceful
```

### Sensitive Data Leak Detection

When checking for leaked secrets in source code, use **precise patterns** that match actual credential variables — not display-only words like "tokens" (token counts) or "secret_count".

```bash
# CORRECT — match actual credential assignments/logs, not display strings
# Use word boundaries (\b) and assignment/interpolation context
SENSITIVE=$(grep -Pn '(api[_-]?key|api[_-]?secret|api[_-]?token|auth[_-]?token|bearer|password|secret[_-]?key|private[_-]?key|access[_-]?token|refresh[_-]?token|client[_-]?secret)\s*[:=]' "$FILE" 2>/dev/null | wc -l)
# Also check console.log that interpolates credential-named variables
LOGGED=$(grep -Pn 'console\.(log|error|warn)\(.*\$(api_key|token|password|secret|bearer)' "$FILE" 2>/dev/null | wc -l)

# WRONG — too broad, causes false positives:
# grep 'console.log.*token'    ← matches "12.5K tokens" display
# grep 'console.log.*secret'   ← matches "secret_count" or "no secrets found"
# grep '.*password.*'          ← matches "password requirements" help text
```

**Why this matters:** Broad patterns like `console.log.*token` flag every line that displays token *counts* (e.g., `${tokens}` showing "12.5K tokens"), password *requirements* text, or secret *status* messages. These false positives erode trust in the test suite — teams start ignoring failures. Use assignment/interpolation context to distinguish "logging a credential value" from "displaying a word that contains 'token'".

### API Endpoint Suite

For any `GET/POST/PUT/DELETE /api/v1/<path>`:

```bash
# 1. No auth (should be 401, not 500)
curl -s -o /dev/null -w "%{http_code}" "https://raichu.heyiris.io<path>" -H "Accept: application/json"

# 2. Invalid auth (should be 401)
curl -s -o /dev/null -w "%{http_code}" "https://raichu.heyiris.io<path>" -H "Authorization: Bearer invalid" -H "Accept: application/json"

# 3. Empty body on POST (should be 422, not 500)
curl -s -o /dev/null -w "%{http_code}" -X POST "https://raichu.heyiris.io<path>" -H "Accept: application/json" -H "Content-Type: application/json" -d '{}'

# 4. SQL injection in query params
curl -s -o /dev/null -w "%{http_code}" "https://raichu.heyiris.io<path>?search=%27%3BDROP%20TABLE--" -H "Accept: application/json"

# 5. Very large page/limit
curl -s -o /dev/null -w "%{http_code}" "https://raichu.heyiris.io<path>?per_page=999999" -H "Accept: application/json"

# 6. Response time under 5s
curl -s -o /dev/null -w "%{time_total}" "https://raichu.heyiris.io<path>" -H "Accept: application/json" | awk '{exit ($1 > 5)}'
```

### Upload/Write Suite

For any feature that creates DB records:

```bash
# 1. Create with valid data → verify record exists
# 2. Create duplicate → should detect or handle gracefully
# 3. Create with XSS payload → HTML stripped or escaped
# 4. Create with max-length fields → truncated or rejected
# 5. Delete the created record → verify 404 after delete
# 6. Create then immediately read → consistency check
```

## Integration with Production Deploy

The stress-test skill can be invoked from `/production-deploy client-ready` as part of Gate 2:

```
Gate 2 (QA & Tests):
  - Unit tests .................. PASS
  - E2E spec .................... PASS
  - /stress-test <feature> ...... 29/30 (WARN: 1 security issue)
  - Production errors ........... 0 new
```

To invoke programmatically:
```bash
# From another skill or script
iris skill run stress-test "iris content"
```

## Severity Levels

| Level | Criteria | Action |
|-------|----------|--------|
| **CRITICAL** | 500 error, data loss, auth bypass, command injection | BLOCK ship. Fix immediately. |
| **HIGH** | XSS stored, IDOR, rate limit bypass | Fix before client-ready. |
| **MEDIUM** | Poor error message, missing validation, slow response (>5s) | Fix in follow-up. Ship with known issue. |
| **LOW** | Cosmetic (emoji in output, truncation off-by-one) | Note it. Fix when convenient. |

## Quick Start Examples

```bash
# Test a CLI command
/stress-test iris content

# Test a specific endpoint
/stress-test /api/v1/my/profiles

# Test after a deploy
/stress-test --recent   # auto-discover from last 3 commits

# Test with verbose output
/stress-test iris content --verbose

# Dry run (show test cases without executing)
/stress-test iris content --dry-run
```
