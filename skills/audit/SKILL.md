---
name: codebase-audit
description: >
  Run a comprehensive security and edge-case audit across an entire codebase.
  Trigger this skill whenever the user asks to "audit", "security review", "find vulnerabilities",
  "check for edge cases", "review my codebase for issues", or anything involving systematic
  analysis of security risks, logic flaws, or missing error handling across a project.
  Also trigger for phrases like "what could go wrong", "is my code safe", or
  "review before I ship/deploy". Always use this skill — never just wing a code review.
---

# Codebase Audit Skill

Perform a structured, thorough audit of the user's codebase covering security vulnerabilities,
edge cases, and conceptual/architectural risks. Produce a clear, actionable report.

---

## Phase 1 — Reconnaissance

Before auditing, understand the project:

```bash
# Get a project map — structure, languages, entry points
find . -type f | grep -v node_modules | grep -v .git | grep -v dist | grep -v __pycache__ \
  | sort | head -120

# Identify key config files
ls -la
cat package.json 2>/dev/null || cat pyproject.toml 2>/dev/null || cat Cargo.toml 2>/dev/null \
  || cat go.mod 2>/dev/null || cat composer.json 2>/dev/null || true

# Check for env/secrets patterns
grep -r "process\.env\|os\.environ\|getenv\|SECRET\|API_KEY\|PASSWORD\|TOKEN" \
  --include="*.js" --include="*.ts" --include="*.py" --include="*.go" \
  -l 2>/dev/null | head -20
```

Read enough of the codebase to understand:
- **What the project does** (its core purpose)
- **Tech stack** (language, framework, DB, auth)
- **Entry points** (HTTP handlers, CLI args, message consumers, cron jobs)
- **Trust boundaries** (what data comes from users vs. internal vs. env)

---

## Phase 2 — Audit Execution

Run checks across all six domains. For each finding, note the **file + line** where possible.

### 2A — Secrets & Configuration
```bash
# Hardcoded credentials
grep -rn "password\s*=\s*['\"].\+['\"]" --include="*.py" --include="*.js" --include="*.ts" .
grep -rn "api_key\s*=\s*['\"].\+['\"]" --include="*.py" --include="*.js" --include="*.ts" .
grep -rn "secret\s*=\s*['\"].\+['\"]" --include="*.py" --include="*.js" --include="*.ts" .

# .env files accidentally committed
find . -name ".env" -not -path "*/.git/*"
cat .gitignore 2>/dev/null | grep -i "env\|secret\|key"
```

Check for:
- [ ] Hardcoded secrets, tokens, passwords
- [ ] `.env` files committed to repo
- [ ] Secrets logged or returned in API responses
- [ ] Weak default configs (debug=True in prod, permissive CORS `*`)

### 2B — Input Validation & Injection
```bash
# SQL query construction patterns
grep -rn "query\|execute\|cursor" --include="*.py" --include="*.js" --include="*.ts" . \
  | grep -i "f\"\|f'\|\`\|%s\|format(" | head -30

# Shell command construction
grep -rn "exec\|spawn\|subprocess\|child_process\|os\.system\|shell=True" \
  --include="*.py" --include="*.js" --include="*.ts" . | head -20

# File path construction from user input
grep -rn "join\|open\|readFile\|path\." \
  --include="*.py" --include="*.js" --include="*.ts" . | head -20
```

Check for:
- [ ] SQL injection (string-concatenated queries, no parameterization)
- [ ] Command injection (user input passed to shell)
- [ ] Path traversal (`../` in file operations from user input)
- [ ] XSS (unescaped output in HTML/templates)
- [ ] Template injection
- [ ] Missing input length/type/format validation

### 2C — Authentication & Authorization
```bash
# Auth middleware patterns
grep -rn "middleware\|auth\|require_login\|@login_required\|authenticate\|authorize" \
  --include="*.py" --include="*.js" --include="*.ts" . | head -30

# JWT handling
grep -rn "jwt\|token\|decode\|verify" \
  --include="*.py" --include="*.js" --include="*.ts" . | head -20
```

Check for:
- [ ] Routes/endpoints missing auth checks
- [ ] Insecure direct object reference (IDOR) — user can access other users' data by changing an ID
- [ ] JWT: algorithm confusion, missing expiry, `alg: none`
- [ ] Password hashing (bcrypt/argon2 vs. md5/sha1/plaintext)
- [ ] Session fixation, missing CSRF protection
- [ ] Privilege escalation paths

### 2D — Error Handling & Edge Cases
```bash
# Unhandled promise rejections / bare excepts
grep -rn "\.catch\|try\|except\|rescue" \
  --include="*.py" --include="*.js" --include="*.ts" . | head -30

# Places where errors might leak internals
grep -rn "traceback\|stack\|e\.message\|err\.message\|console\.error\|print(e" \
  --include="*.py" --include="*.js" --include="*.ts" . | head -20
```

Check for:
- [ ] Unhandled exceptions / promise rejections
- [ ] Stack traces / internal errors exposed to end users
- [ ] Missing null/undefined checks before property access
- [ ] Integer overflow, off-by-one errors in loops or pagination
- [ ] Race conditions / TOCTOU (check-then-act on shared state)
- [ ] Unvalidated external API responses

### 2E — Data & Privacy
```bash
# Logging patterns
grep -rn "console\.log\|print\|logger\." \
  --include="*.py" --include="*.js" --include="*.ts" . | grep -i "user\|email\|pass\|token" | head -20

# DB query patterns — mass assignment / no field filtering
grep -rn "\*\|SELECT \*\|\.all()\|findAll" \
  --include="*.py" --include="*.js" --include="*.ts" . | head -20
```

Check for:
- [ ] PII logged (emails, IPs, passwords)
- [ ] Sensitive data in URLs (tokens/passwords as query params)
- [ ] No data retention / deletion strategy
- [ ] Mass assignment vulnerability (accepting all fields from user input into DB)
- [ ] Overly broad DB queries returning more data than needed

### 2F — Dependency & Infrastructure
```bash
# Outdated / known-vulnerable packages
npm audit 2>/dev/null | tail -20 || pip-audit 2>/dev/null | tail -20 || true

# Dockerfile / infra concerns
cat Dockerfile 2>/dev/null
cat docker-compose.yml 2>/dev/null
```

Check for:
- [ ] Known vulnerable dependencies (`npm audit`, `pip-audit`, `cargo audit`)
- [ ] Running containers as root
- [ ] Exposed ports / debug endpoints in production config
- [ ] Missing rate limiting on public endpoints
- [ ] No HTTPS enforcement

---

## Phase 3 — Conceptual / Architectural Review

Beyond code, reason about the project's design against its purpose:

- **Business logic flaws**: Can users bypass intended flows? (e.g., skip payment, re-use single-use codes, abuse free tiers)
- **Concurrency**: Are there shared resources (DB rows, file handles, counters) accessed without locks?
- **Denial of service**: Are there endpoints with no rate limit, or operations that could be made arbitrarily expensive by a user?
- **Third-party trust**: Is data from external APIs/webhooks validated before acting on it?
- **Failure modes**: What happens when the DB, cache, or external service goes down? Does the app fail safe?
- **Scope creep risks**: Does the codebase do more than it should? Unnecessary attack surface?

---

## Phase 4 — App Logic & Completeness Review

This phase steps back from individual code issues and evaluates whether the app's **overall logic is correct, complete, and coherent** — does it actually do what it's supposed to do, end to end, in all the ways that matter?

### 4A — Reconstruct the Intended App Logic

Before critiquing, fully understand the app's purpose by reading:
- README, product docs, or any spec files
- Route/controller names and their groupings
- Data models and their relationships
- Any state machines (order status, user roles, workflow stages)

Then write out — in plain language — what the app is supposed to do:

```
Example reconstruction:
"This is a freelance invoicing app. Users create clients, attach projects to clients,
log time against projects, generate invoices from logged time, send invoices to clients
via email, and mark them as paid when payment arrives. There's a subscription tier that
limits free users to 3 active clients."
```

This reconstruction becomes the baseline you audit against. Every gap between **what it's meant to do** and **what the code actually does** is a finding.

### 4B — Trace Every Core Flow End to End

For each major user-facing flow, trace it through the code from entry point to final side effect:

1. **Map the flow in steps** — e.g. for "send invoice": create → preview → send email → track open → record payment → close invoice
2. **Find where each step is implemented** (or missing)
3. **Check transitions between steps** — is state updated atomically? Can a step be skipped?
4. **Verify the final state is always consistent** — no orphaned records, no half-applied changes

Questions to ask at each step:
- Is this step actually implemented, or just assumed?
- What happens if this step fails partway through — is prior state rolled back?
- Can a user reach step N without completing step N-1?
- Is the outcome of this step visible/auditable somewhere?

### 4C — Identify Missing Logic

Look for flows that are implied by the data model or UI but not implemented:

- **Lifecycle gaps**: entities that can be created but not edited, deleted, or archived
- **Missing notifications**: actions that should trigger an email/webhook/event but don't
- **No cleanup**: resources created as part of a flow but never cleaned up if the flow is abandoned (e.g. uploaded file with no associated record)
- **Unhandled terminal states**: what happens when an order is cancelled mid-fulfillment, a subscription expires, or a user is deleted — are related records handled?
- **Missing admin/ops tooling**: is there any way to recover from bad state, reprocess a failed job, or manually override a stuck workflow?
- **Asymmetric operations**: if you can do X, can you undo X? (create/delete, invite/revoke, enable/disable)

### 4D — Business Rule Verification

Extract the business rules from the README/docs/models and verify each one is enforced in code:

| Business Rule (from spec/readme) | Enforced in code? | Where? | Gap? |
|---|---|---|---|
| Free users limited to 3 clients | Check in client creation handler | `clients/create.ts:44` | ✅ |
| Invoices can't be edited after sending | Status check before update | missing | ❌ Gap |
| ... | | | |

Flag every rule that is:
- **Not enforced at all** — only in the UI, not server-side
- **Enforced inconsistently** — checked in one endpoint but not another that does the same thing
- **Bypassable** — enforced on the main path but not on an edge route (e.g., bulk operations, API vs UI)

### 4E — State Machine Integrity

If the app has entities with status fields (orders, invoices, tickets, subscriptions, jobs):

```bash
# Find status/state fields
grep -rn "status\|state\|stage\|phase" --include="*.py" --include="*.ts" --include="*.js" \
  -l . | head -20
```

For each state machine, verify:
- [ ] All valid states are defined and documented
- [ ] All valid transitions are explicitly listed — invalid transitions are rejected
- [ ] No code sets status directly by string without going through a transition function
- [ ] Terminal states (cancelled, completed, failed) cannot be re-opened accidentally
- [ ] Status changes are logged/auditable

### 4F — Cross-cutting Concerns

Check that important concerns are applied **consistently across all flows**, not just the main ones:

- [ ] **Emails/notifications**: Are all significant events communicated? Are any sent redundantly or missing?
- [ ] **Audit logging**: Are destructive or sensitive actions logged with who did them and when?
- [ ] **Billing/metering**: If the app has usage limits or billing, is every relevant action metered? Can any action bypass the meter?
- [ ] **Soft delete vs hard delete**: Is deletion consistent across entities? Can deleted data be accidentally restored or referenced?
- [ ] **Search/filtering**: Do all list endpoints respect the same ownership/permission scoping as single-item endpoints?
- [ ] **Pagination**: Are all list endpoints paginated? Is the default page size safe at scale?

---

## Phase 5 — Report Format

Output a structured Markdown report. Use exactly this structure:

```markdown
# 🔍 Codebase Audit Report
**Project:** <name>  
**Stack:** <language / framework / DB>  
**Audited:** <timestamp>  
**Total Findings:** <N> (<critical> critical, <high> high, <medium> medium, <low> low, <info> info)

---

## Executive Summary
<2–4 sentences: what the project does, overall risk posture, most urgent concern>

---

## Findings

### [SEV-001] <Title>  <!-- repeat block for each finding -->
| Field | Detail |
|-------|--------|
| **Severity** | 🔴 Critical / 🟠 High / 🟡 Medium / 🔵 Low / ℹ️ Info |
| **Category** | Secrets / Injection / AuthZ / Error Handling / Data / Deps / Architecture |
| **Location** | `path/to/file.ts:42` (or "Multiple files") |

**What's wrong:**  
<Clear explanation of the vulnerability or flaw>

**Attack scenario / impact:**  
<Who could exploit this, and what they could do>

**Fix:**  
```<lang>
// concrete before/after code snippet where possible
```

---

## App Logic & Completeness

### Reconstructed App Intent
<2–4 sentences describing what the app is supposed to do, derived from the code>

### Logic Findings

#### [LOGIC-001] <Title>  <!-- repeat for each logic gap -->
| Field | Detail |
|-------|--------|
| **Type** | Missing flow / Broken transition / Unenforced rule / Lifecycle gap / Inconsistent behavior |
| **Location** | `path/to/file.ts:42` or "No implementation found" |

**What's supposed to happen:**  
<The intended behavior per spec/readme/data model>

**What actually happens:**  
<What the code does — or fails to do>

**Impact:**  
<Data inconsistency / user confusion / revenue loss / silent failure / etc.>

**Fix:**  
<Concrete recommendation>

---

## Real-World Edge Cases

### [EDGE-001] <Title>  <!-- repeat for each edge case -->
| Field | Detail |
|-------|--------|
| **Domain** | Payments / Auth / File Upload / API / Scheduling / etc. |
| **Trigger condition** | <what user action or system state causes this> |
| **Location** | `path/to/file.ts:42` |

**Scenario:**  
<Describe the real-world situation — what a user does, what the system does, what goes wrong>

**Impact:**  
<Data loss / double charge / stuck state / silent failure / etc.>

**Fix:**  
<Concrete recommendation — idempotency key, optimistic lock, queue dedup, timeout + compensation, etc.>

---

## Summary Table
| ID | Severity | Category | Title | File |
|----|----------|----------|-------|------|
| SEV-001 | 🔴 Critical | Injection | SQL injection in search endpoint | `api/search.py:88` |
| LOGIC-001 | 🟠 High | Lifecycle gap | Deleted users' records not cleaned up | No implementation found |
| EDGE-001 | 🟠 High | Payments | Double-charge on checkout retry | `checkout/handler.ts:210` |
...

---

## Recommendations (Priority Order)
1. **Immediate (before next deploy):** ...
2. **Short-term (this sprint):** ...
3. **Longer-term (architectural):** ...
```

---

## Severity Guide

| Level | When to use |
|-------|-------------|
| 🔴 **Critical** | Remote code execution, auth bypass, credential exposure, data breach vector |
| 🟠 **High** | Privilege escalation, stored XSS, IDOR, injections without immediate RCE |
| 🟡 **Medium** | Reflected XSS, missing rate limits, info disclosure, weak crypto |
| 🔵 **Low** | Best practice deviations, verbose errors, minor misconfigs |
| ℹ️ **Info** | Observations worth noting but not exploitable |

---

## Execution Notes

- **Be thorough, not noisy.** Don't flag things that are definitively not issues. Prefer fewer high-signal findings over a long list of low-value ones.
- **Always show the actual vulnerable code** when you identify a finding — never describe it vaguely.
- **For large codebases**, focus first on: auth flows, public-facing input handlers, payment/sensitive-action paths, and anywhere external data is processed.
- **If the project has a README or spec**, read it — some of the most important findings come from understanding what the code is *supposed* to do and finding gaps between intent and implementation.
- After the report, offer to deep-dive on any specific finding or generate a fix PR.
