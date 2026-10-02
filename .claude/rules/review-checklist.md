# PR Review Checklist Reference

This file provides the expanded checklist used during PR reviews. Reference this when executing the `/review-pr` command.

---

## Review Procedures (Mandatory)

Steps A–F are **context gathering** — complete them before judging any code. Steps G–I are **verification passes** — run them after you understand the whole diff, before finalizing findings.

| Step | Procedure | Answers |
|---|---|---|
| A | Caller Search | Who calls the changed code — is it still compatible? |
| B | Convention Verification | Is this really a deviation, or the project's own convention? |
| C | Performance Context | Is this performance risk real at this data scale and code path? |
| D | Suggestion Validation | Will the fix I'm suggesting actually compile/run there? |
| E | Callee & Contract Validation | What does the changed code call — do those calls still resolve correctly? |
| F | Reuse / Duplication Search | Does this new code already exist somewhere in the codebase? |
| G | Correctness Trace | Does the code actually do what it's supposed to do? |
| H | Test Coverage | Is this new logic tested, where a suite exists? |
| I | Scope Alignment | Does this change belong in this PR at all? |

### Step A: Caller Search Procedure

Use this whenever a modified (not newly created) function is identified:

1. Run `search_code` with the function name as the query in the repository
2. Collect every file and line that calls this function
3. For each call site, read the surrounding 10–20 lines to understand how the return value, parameters, and side effects are used
4. Compare call site expectations against what the modified function now does
5. Document: "Callers found: X. All compatible: Yes/No. Incompatible callers: [list]"

**If no callers are found via search**: the function may be unused, or the search query may need refinement. Try searching for partial name, aliased imports, or dynamic call patterns before concluding there are no callers.

### Step B: Convention Verification Procedure

Use this before flagging any architecture, naming, or layer placement issue:

1. Identify the pattern you are about to flag (e.g., "DB query in controller", "camelCase variable", "service returning raw model")
2. Search the codebase for 3–5 files in the same layer/type as the changed file
3. Count: how many of those files follow the same pattern you were about to flag?
4. Decision rule:
   - ≥ 2 existing files follow the same pattern → that IS the convention; do NOT flag
   - 0–1 existing files follow the pattern → the new code deviates from convention; flag with the correct example as reference
5. When flagging, cite the specific `file:line` that demonstrates the correct convention

### Step C: Performance Context Assessment

Use this before flagging any N+1 query, algorithm inefficiency, or unbounded growth issue:

1. **Data scale**: What is the maximum realistic size of the collection being iterated?
   - Bounded small set (e.g., always < 20 items, hardcoded list, config values) → low risk
   - Unbounded user-controlled data (e.g., "all orders", "all users") → real risk
2. **ORM/framework batching**: Does the ORM, framework, or library already handle batch fetching for this access pattern?
   - E.g., ActiveRecord lazy-loads only when accessed; Eloquent `$model->relation` in a loop IS N+1
   - E.g., Some GraphQL dataloaders batch automatically
3. **Code path criticality**: Is this code in an HTTP request handler, a background job processing many records, or a one-off admin script?
   - HTTP request / high-frequency job → performance matters; flag if real risk
   - One-off admin script / migration → performance is acceptable; do not flag
4. Only flag after all three context factors confirm a real production risk

### Step D: Suggestion Validation Checklist

Before finalizing any `suggestion` block:

1. **Imports**: Does the suggested code use any class, function, or module that isn't already imported in the file? If yes, the suggestion must include the import line, or explain it separately
2. **Variable scope**: Are all variables referenced in the suggestion already defined and in scope at the comment's line number?
3. **Type compatibility**: If the codebase uses a type system (TypeScript, PHP type hints, Python type annotations), does the suggestion maintain the expected types?
4. **Syntax**: Is the suggested code syntactically correct for the language and version in use?
5. **Indentation**: Does the suggested code match the surrounding indentation style (tabs vs spaces, indent size)?
6. **Line count**: For a `suggestion` block, the replacement must cover exactly the commented lines — not more, not fewer. If the fix is larger, use a regular code block with "Sebelum/Setelah" labels instead

### Step E: Callee & Contract Validation Procedure (outbound dependencies)

Step A finds who **calls** the changed code. This step validates what the changed code **calls**. Run it for every external reference appearing in added/modified lines.

1. **Inventory the references.** From the added/modified lines, list every identifier that is *not* defined in the same file: functions, methods, static/class methods, constants, enum members, classes, types/interfaces, injected services.
2. **Locate each definition.** Use `search_code` for the identifier, then `get_file_contents` on the defining file. Follow the import statement at the top of the changed file to resolve aliases and re-exports.
3. **Read the actual definition** — the signature line plus the body. Never infer behavior from the identifier's name.
4. **Compare against the call site:**

| What to compare | What breaks if it's wrong |
|---|---|
| Function exists & is exported/public | Fatal error / undefined method at runtime |
| Argument count | Missing-argument error, or silently-undefined param |
| Argument order & types | Wrong result, type error, silent coercion bug |
| Optional/default params | Caller assumes a default the callee doesn't have |
| Return type & shape | Caller destructures/indexes something that isn't there |
| Nullability | Null-pointer / undefined dereference downstream |
| Sync vs. async (Promise/Future/coroutine) | Missing `await` → caller gets a Promise, not a value |
| Collection vs. single item | Iterating a single object, or indexing an array as an object |
| Constant/enum member exists & value | Comparison silently never matches |
| Thrown exceptions | Caller has no handler for an exception the callee can raise |
| Real side effects (DB write, cache, event) | Caller assumes persistence that never happens |

5. **If the callee's file is also changed in this PR**, compare the call against the **new** version of the definition, not the old one — and verify the change is coherent from both sides.
6. **Record the result** per reference: `verified-ok`, `mismatch-found`, or `unverified (reason)`. Never leave a reference silently unchecked.
7. **Where the comment goes:** a mismatch is a finding on the **changed calling line**, not on the callee's file. This stays within the changed-code scope rule.

### Step F: Reuse / Duplication Search Procedure

Run this for every newly added function, helper, constant, validator, mapper, DTO, or type.

1. **Name search** — `search_code` for the identifier itself, and for its stem (`formatRupiah` → `format`, `rupiah`).
2. **Behavior search** — 2–3 keywords describing what it does, plus the domain noun (`currency`, `idr`, `money`).
3. **Conventional-location search** — list the folder where this project keeps such utilities (`utils/`, `helpers/`, `services/`, `support/`, `lib/`) and scan for an existing equivalent, even if the new code was placed elsewhere.
4. **Hardcoded-value search** — if the new code hardcodes a string/number that looks like a domain value (status name, role, error code, limit), search whether a named constant or enum for it already exists.
5. **Decide:**
   - Equivalent found and usable → `[MAJOR]` DRY finding; cite the existing `file:line` and recommend reuse
   - Equivalent found but genuinely not usable here → no finding, but say why in the analysis notes
   - Nothing found after all four searches → accept the new code as legitimately new
6. Never conclude "this is new" without having run the searches — an assumption is not a verification.

### Step G: Correctness Trace Procedure

Run this before finalizing, for every meaningfully changed code path.

1. **Restate the intent** in one sentence, taken from the Phase 1 Scope Contract — sourced from the linked issue first, the PR description only if no issue is linked.
2. **Walk the path with real values** — pick a concrete input and follow it line by line through the changed code to the output. Do not skim for shape; actually carry the value.
3. **Run one edge case** from this list, whichever is relevant: empty collection, single element, null/undefined, zero, negative, boundary value (first/last, min/max), duplicate entry, very large input, concurrent access.
4. **Check the classic correctness bugs:**
   - Inverted condition (`&&`/`||` swapped, missing `!`)
   - Boundary operator (`>` where `>=` was meant), off-by-one in loops/slices/pagination
   - Wrong variable used (copy-paste leftover from the block above)
   - Assignment `=` where comparison `==`/`===` was meant
   - Loose vs. strict equality causing unintended type coercion
   - Missing `await` / unhandled promise / unwaited goroutine
   - Early `return`/`break`/`continue` that skips required cleanup or a later branch
   - Mutation of a shared/passed-in object the caller still relies on
   - A branch that falls through without returning or assigning anything
5. **Compare result to intent.** If actual behavior ≠ stated intent, that is `[KRITIS]` or `[MAJOR]` and outranks every style/convention/performance finding.

### Step H: Test Coverage Check

1. Determine whether the repository has a test suite at all (`tests/`, `spec/`, `__tests__/`, `*_test.go`, `*Test.php`, etc.). If it has none, skip this step entirely — introducing one is not this PR's job.
2. If a suite exists, check whether this PR adds or updates tests for the logic it changed.
3. Flag as `[MAJOR]` only when the untested change is non-trivial: a branch, a loop, a parser, or a money/auth/security/permission path.
4. Do NOT flag missing tests for trivial changes (copy text, config values, formatting, pure renames).

### Step I: Scope Alignment Procedure (does this PR stay inside the issue's purpose?)

Every other step asks whether the code is *correct*. This one asks whether the change *belongs in this PR*. Run it last, after you understand the whole diff.

1. **Establish the authority on intent**, in this priority order:
   1. The linked issue (`Closes #12`, `Fixes #12`, `Menutup #12`, a bare `#12`, an issue URL, or a branch named `fix/issue-12-...`) — read it with `issue_read`, including its comments, where scope is often narrowed after the fact
   2. The PR description — the author's *claim* about what they did, which may not match the issue
   3. The diff itself — only when the first two are absent or empty. Mark the intent as **inferred**.
2. **Write the Scope Contract** in three lines: `Tujuan` / `Termasuk scope` / `Di luar scope`.
3. **Classify every changed hunk:**

| Classification | Example | Action |
|---|---|---|
| **Inti** | The bug named in the issue is fixed in the function the issue points at | No scope finding |
| **Pendukung** | A call site updated because the signature had to change; an import added; a test for the new branch; a migration the fix requires | No scope finding — but confirm it really was required |
| **Di luar scope** | An unrelated function renamed; a whole class reformatted; a second, different bug fixed along the way; a new feature nobody asked for; a dependency bumped for no stated reason | `[MINOR]` — ask the author to split it into its own PR/issue |
| **Bertentangan** | Behavior the issue did not mention is changed; the fix solves a different problem than the one reported; an acceptance criterion is contradicted | `[KRITIS]` or `[MAJOR]` — Rule 4 applies |

4. **Check under-delivery, not just over-delivery.** Walk the issue's acceptance criteria / reproduction steps one by one and confirm the diff actually satisfies each. Name any criterion left unmet — a PR that claims `Closes #12` while leaving half of #12 unfixed is a real finding.
5. **Watch for the disguised refactor.** Rule 4 already forbids refactoring working code without justification; this step is where you catch it. A diff that is 10% fix and 90% reformatting makes the fix unreviewable — flag the noise even if every individual line is harmless.
6. **Calibrate confidence to the source of intent.** Intent from a linked issue → flag scope findings normally. Intent inferred from the diff → downgrade to `[SARAN]` and phrase as a question ("Apakah perubahan ini memang bagian dari scope issue ini, atau sebaiknya dipisah?"). Never accuse an author of scope creep based on a guess about what was asked.
7. **Anchoring guard.** Steps E, F, and G send you deep into files this PR never touched. Anything you noticed there that does not relate to a changed line *and* to the Scope Contract is a separate issue — not a comment on this PR. A deep read is not a licence for a codebase audit.

---

## Scope: Review Changed Code Only

Before starting any review, establish the scope clearly:

This rule constrains **where a comment is posted**, not **what you are allowed to read**. Reading unchanged files is mandatory context gathering (Steps A, B, E, F) — it is not scope expansion.

- **ONLY post comments on code that is added, modified, or deleted in the PR diff**
- **Do NOT flag the internal quality of unchanged code** — even if you spot issues in surrounding code that the PR didn't touch
- **DO read unchanged code freely** — callers, callees, existing utilities, constants, conventions. You cannot verify a change without it.
- If you discover a bug or issue in unmodified code that makes the PR's change problematic, that is reviewable (the PR's change depends on the buggy context). Document this clearly.
- If you spot an unrelated issue in unchanged code, note it separately for a new issue, not this PR's review

**Example:**
- PR adds a line `const x = getValue()` — you can review `getValue()` if it's new/modified in this PR
- PR does NOT modify `getValue()`, it just calls it → you MUST still read `getValue()` and verify the call is valid (exists, arg count, return shape, sync/async — Step E). But:
  - The call is invalid (wrong args, `getValue()` returns a Promise and `x` is used as a value) → **flag it**, anchored on the changed calling line
  - The call is valid but `getValue()` has ugly internals, a missing null check of its own, or an N+1 inside it → **do NOT flag**; that belongs in a separate issue

---

## Security Patterns to Always Check

### Injection
- String concatenation used to build SQL queries → SQL Injection
- Unsanitized user input passed to shell commands → Command Injection
- User-controlled data in LDAP filters → LDAP Injection
- Unvalidated XML input with DOCTYPE declarations → XXE

### Authentication & Authorization
- Missing authentication middleware on new routes/endpoints
- Permission check done after the resource is fetched (fetch first, check later = IDOR risk)
- Role checks that use string comparison on user-controlled input
- JWTs not validated (signature, expiry, issuer)
- Hardcoded credentials or API keys in source code

### Data Exposure
- Passwords, secrets, tokens logged at any log level
- Sensitive fields included in serialized API responses without explicit filtering
- Stack traces exposed to end users in error responses
- PII stored unencrypted in database columns that should be encrypted

### CSRF
- State-changing endpoints (POST/PUT/PATCH/DELETE) missing CSRF token validation
- CORS wildcard (`*`) on endpoints that perform mutations

---

## Performance Anti-Patterns

### N+1 Query Examples

**Bad (N+1):**
```
orders = Order.all
orders.each do |order|
  puts order.user.name   # query per order
end
```

**Good (eager load):**
```
orders = Order.includes(:user).all
orders.each do |order|
  puts order.user.name   # no extra query
end
```

---

**Bad (N+1 in loop):**
```
for id in user_ids:
    user = db.query("SELECT * FROM users WHERE id = ?", id)
```

**Good (batch):**
```
users = db.query("SELECT * FROM users WHERE id IN (?)", user_ids)
```

---

### Unnecessary Nesting — Guard Clause Pattern

**Bad (deep nesting):**
```
def process(user):
    if user:
        if user.active:
            if user.has_permission:
                do_work(user)
```

**Good (guard clauses):**
```
def process(user):
    if not user: return
    if not user.active: return
    if not user.has_permission: return
    do_work(user)
```

---

### DRY Violation Detection

Look for:
- Identical or near-identical blocks of code in different methods
- The same validation logic repeated in multiple places instead of a shared validator
- Database query patterns duplicated across service methods
- Same transformation logic written twice instead of a shared mapper function

---

## Behavioral Preservation — What to Check

When a PR modifies an existing function, find ALL callers by searching the codebase. Verify:

1. **Return value compatibility** — if callers expect a specific type/shape, the modified function must still return it
2. **Parameter compatibility** — if new required parameters are added, ALL callers must have been updated
3. **Exception behavior** — if callers assume a function never throws, and the modified version can now throw, that is a breaking change
4. **Side effects** — if the function previously wrote to a database, cache, or file, and the PR removes or conditionally skips that, all callers relying on that side effect are broken

If the PR description does NOT explicitly state that behavior is intentionally changing, treat any behavioral difference as a bug in the PR.

---

## Package & Dependency Security Checklist

Use this when a PR adds, removes, or upgrades dependencies.

### Step 1: Identify Changed Dependency Files

Look for changes in:
- `package.json` / `package-lock.json` / `yarn.lock` (Node.js)
- `composer.json` / `composer.lock` (PHP)
- `requirements.txt` / `Pipfile` / `pyproject.toml` (Python)
- `go.mod` / `go.sum` (Go)
- `Gemfile` / `Gemfile.lock` (Ruby)
- `pom.xml` / `build.gradle` (Java/Kotlin)
- `*.csproj` / `packages.config` (C#/.NET)
- `pubspec.yaml` (Dart/Flutter)
- `Cargo.toml` / `Cargo.lock` (Rust)

### Step 2: Evaluate Each New Package

For every new package added, check:

| Check | How to Assess |
|---|---|
| **Maintenance** | When was the last release? Active = within 1 year. Stale = 2+ years with no archived notice |
| **Popularity** | npm: weekly downloads; GitHub: stars. Low adoption + no known org = higher risk |
| **Org Backing** | Is it maintained by a company/foundation (Google, Meta, Vercel, Apache, OWASP)? More trustworthy |
| **CVEs** | Search package name in [OSV](https://osv.dev), [Snyk Advisor](https://snyk.io/advisor), [npm advisories](https://www.npmjs.com/advisories) |
| **Dependency Footprint** | Does it pull in 50+ transitive deps for a simple task? Bigger surface = bigger risk |
| **Lifecycle Scripts** | Does `package.json` have `postinstall` that runs shell commands? Flag this |
| **Typosquatting** | Is the name suspiciously close to a popular package? (e.g., `express-js` vs `express`) |
| **License** | Is it MIT/Apache/BSD? Avoid GPL in proprietary projects without legal review |

### Step 3: Assign Security Status

**✅ Aman** — All of the following:
- Last release within 12 months
- No known unpatched CVEs
- Reasonable adoption (or backed by reputable org)
- No suspicious lifecycle scripts
- Name is clearly legitimate

**⚠️ Perlu Perhatian** — Any of the following:
- Last release 12-24 months ago but still supported
- CVEs exist but ALL are fixed in the version being used
- Small community but package is from a known developer with good track record
- Slightly high transitive dependency count but well-known packages

**❌ Tidak Disarankan** — Any of the following:
- No release in 2+ years and not archived/deprecated (just abandoned)
- Unpatched CVE exists in the version being added
- < 500 weekly downloads with no reputable org backing
- Suspicious lifecycle scripts with no clear justification
- Name looks like a typosquat
- Multiple major security advisories in package's history

### Step 4: Audit Commands by Ecosystem

Always remind the PR author to run the appropriate command after adding dependencies:

```bash
# Node.js
npm audit
# or
yarn audit

# PHP (Composer 2.4+)
composer audit

# Python
pip-audit
# or
safety check

# Go
govulncheck ./...

# Ruby
bundle audit

# Java (OWASP Dependency Check)
mvn org.owasp:dependency-check-maven:check

# Gradle
gradle dependencyCheckAnalyze

# Rust
cargo audit

# .NET
dotnet list package --vulnerable
```

### Red Flags — Escalate to [KRITIS]

Flag as `[KRITIS]` if:
- A package with a known unpatched CVE is being added
- A package appears to be a typosquat of a popular library
- A `postinstall` script runs curl/wget or downloads external binaries
- A package was published less than 30 days ago with no established author
- A direct dependency is being downgraded to a version with a known vulnerability

---

## Architecture Pattern Examples

### Wrong Layer Placement

| What was added | Where it was added | Where it should be |
|---|---|---|
| Database query | Controller | Repository / DAO |
| Business rule | View / Template | Service / Domain layer |
| HTTP request | Model | Service / Adapter |
| Data transformation | Controller | DTO / Serializer |

### Naming Convention Violations

When reviewing, search the codebase for existing patterns:
- How are service classes named? (`UserService`, `user_service`, `UserSvc`?)
- How are repository methods named? (`findById`, `find_by_id`, `getById`?)
- How are boolean fields named? (`is_active`, `isActive`, `active`?)
- How are event names structured? (`user.created`, `UserCreated`, `USER_CREATED`?)

Flag any new code that deviates from what the majority of the codebase uses.
