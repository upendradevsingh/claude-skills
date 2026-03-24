---
name: reviewing-tests
description: Use when reviewing test quality in a PR or codebase audit — applies a critical reviewer mindset to catch anti-patterns, false confidence, and the cognitive biases that cause reviewers to accept bad tests. The inverse of writing-effective-tests.
---

# Reviewing Tests

## Overview

**The Reviewer's Job:** Prove the test suite is wrong. Not check that it passes — assume it passes. Your job is to find the gap between what the tests *claim* to verify and what they *actually* verify.

**The Inverter Principle:** For every claim ("tests tenant isolation", "tests auth", "verifies persistence"), invert it:
- "Does it really test isolation, or just that tenant A sees its own data?"
- "Does it really test auth, or is security disabled in the test config?"
- "Does it really verify persistence, or just verify a mock was called?"

If the inverted question has no test that would catch it, the claim is false confidence.

---

## Language-Specific Examples

Load the review examples file for your project's language:

- **Python**: Read `examples/python.md`
- **Java**: Read `examples/java.md`
- **Node.js/TypeScript**: Read `examples/node.md`

---

## Project-Specific Patterns

Before reviewing tests, check if the current project has additional test conventions:
- `docs/development/TEST_PATTERNS.md` — project-specific test rules, DB constraints, data setup patterns
- `.claude/skills/` in the project root — project-level skills that supplement this one
- `CLAUDE.md` — may contain test-related instructions

Use project-specific patterns as additional review criteria. Flag violations of project patterns at the same severity as general anti-patterns.

**Conflict resolution:** If a project pattern contradicts this skill, flag it: "Note: project pattern X conflicts with general review rule Y. Applying project pattern. Should these be aligned?" This prevents silent drift between global and project conventions.

## Biases to Suppress Before You Start

### 1. Green Bar Bias
"All tests pass, so the suite is good."

Passing tests prove the paths exercised work. They say nothing about paths not exercised. Look for **missing tests**, not passing ones.

### 2. Coverage Number Bias
"95% coverage means we're safe."

Coverage measures lines executed, not behaviors verified. A test that calls a function and asserts nothing is 100% coverage, 0% value. Ask: are the assertions meaningful, not whether the line was hit?

### 3. Volume Bias
"5,000 tests — this codebase is well-tested."

5,000 tests that verify mock wiring are worse than 50 integration tests that verify behavior. Count **behaviors tested**, not test methods.

### 4. Test Name Trust
"The test is called `test_tenant_isolation`, so isolation is tested."

Read the assertions. Names lie. A test file called `TenantIsolationTest` that never queries as a second tenant proves nothing about isolation.

### 5. Mock Confidence Bias
"All dependencies are mocked, so this is isolated and reliable."

Over-mocking tests implementation, not behavior. The more mocks, the less the test proves. Many mocks = high probability of false confidence.

### 6. Security Config Blindness
"Every test sets up an auth user, so auth is tested."

If the test config disables authentication globally, any per-test auth setup is cosmetic decoration. Check the security config, not just test annotations.

---

## Review Checklist

Work through these layers in order. Each layer has targeted questions. Flag every gap.

---

### Layer 0: Test Suite Structure

Before reading a single test, ask:

- What is the unit:integration:E2E ratio? Does it match the architecture?
  - API service / microservice → expect 20% unit, 70% integration, 10% E2E
  - Monolith with rich domain logic → expect 70% unit, 20% integration, 10% E2E
  - Heavy integration bias in an API-heavy codebase is a sign of over-mocking
- Are error paths tested at any layer? (401, 403, 404, 422, 500)
- Is there a regression test for every known production bug? (Check bug tracker → test files)
- Are there tests that document known gaps?

---

### Layer 1: For Each Test File

Open each test file and ask before reading individual tests:

- **What does the test setup do?** Is there a security override, a SQL seed file, or a `beforeEach` hook that might silently invalidate all tests in this file?
- **What are the mocks/patches?** List them. Too many mocks = red flag for testing implementation.
- **Is lenient/non-strict mocking applied globally?** If yes, treat every mock-heavy test as suspect — unused stubs are hidden.

---

### Layer 2: For Each Test

For every test method, ask:

1. **What is actually being asserted?** (Not what the test name says — read the assert lines.)
2. **What is NOT being tested that should be?** (The inverse scenario, the second tenant, the wrong role, the missing field.)
3. **Are stubs being verified?** Asserting that a mock method was called = testing implementation, not behavior. Flag it.
4. **Is the mock boundary correct?** Managed dependencies (repositories, domain objects, your own classes) should NOT be mocked. External APIs, email, queues should be.
5. **Can I rename an internal method and have these tests still pass?** If yes, they're coupled to implementation.

---

### Layer 3: Unit Tests

- Are repositories/DAOs mocked? They should not be for most service tests. Integration test instead.
- Is "verify the data-access method was called" the **primary** assertion? Wrong — check the return value or DB state.
- Is there any assertion on the return value at all, or is the test entirely mock-call verification?
- Are the tests output-based (preferred) or communication-based (verify-heavy)?

**Red flag:** A service test with 4+ mocks and all assertions verify mock calls. That test verifies mock wiring, not service behavior.

---

### Layer 4: Controller / API Tests

- **Check the test security config first.** Does it permit all requests? If yes, auth annotations are invisible and every security test is a false positive.
- Are the following tested for each endpoint?
  - 401 (no auth / invalid token)
  - 403 (authenticated but wrong role)
  - 422 / 400 (invalid input)
  - 404 (resource not found)
  - Happy path
- Is per-test auth setup used alongside a permissive security config? If so, remove it mentally — would the test still pass? If yes, it's cosmetic.
- Does the test verify HTTP concerns (status codes, headers, response shape) or is it testing business logic that belongs in a service test?

---

### Layer 5: Integration Tests

- Is the test database the **same engine as production**? SQLite ≠ PostgreSQL. SQLite silently differs on JSON, constraints, and types. Flag SQLite in a Postgres shop.
- For each test that claims to test tenant isolation:
  - Does it write data as Tenant A **and** query as Tenant B?
  - If it only reads its own data, it proves nothing about isolation.
- Is data setup using a SQL seed file AND a `beforeEach` hook? Two mechanisms = silent collisions. Flag it.
- Are hardcoded IDs (`1`, `2`, `100`) in seed data or tests? They create invisible coupling between test files.
- Can tests run in any order? Is there shared mutable state between tests?
- Does the test framework rollback transactions automatically? If so, async side effects (background jobs, event handlers) may run outside that transaction and be invisible to cleanup.

---

### Layer 6: Security Tests (Standalone Check)

This gets its own layer because security failures are data-integrity and compliance failures.

1. Locate the test security config (or equivalent). Read it fully. Does it have a catch-all permit? If yes, flag as Critical.
2. Find every authorization annotation or middleware in production code.
3. For each one: is there a test that would fail if the annotation/middleware were deleted? If not, the annotation is unverified.
4. Find tests that assert 401. Do they actually omit auth credentials, or do they rely on a config that bypasses auth?
5. Find tests that assert 403. Do they actually switch to an unauthorized role, or is this cosmetic with permissive config?

---

## The Inverter — Applied

Use this table during review. For each test claim, apply the inversion:

| Test Claims To... | The Inversion Question | What To Check |
|---|---|---|
| Test tenant isolation | "Does it query as a different tenant?" | Find the second-tenant query |
| Test authentication | "Is auth actually enforced in test config?" | Read the test security setup |
| Test authorization | "Would the test fail if the role check were removed?" | Delete check mentally, does a test catch it? |
| Verify persistence | "Does it check DB state or just verify a mock?" | Look for DB read after write |
| Test error handling | "Is the error path actually triggered?" | Trace the input that causes the error |
| Test validation | "Are invalid inputs tested?" | Find negative cases |

---

## Review Output Format

Categorize every finding before writing the review. Don't mix severities in prose.

### Critical
Security or data-integrity risk. Production data could be exposed or corrupted. The test suite provides false assurance against a real threat.

Examples:
- Test security config disables all auth — every security test is a false positive
- Tenant isolation test never queries as a second tenant — RLS could be broken and undetected
- Test database is SQLite, production is PostgreSQL — constraint behavior differs

### High
False confidence. The test looks credible but proves nothing about the behavior it names. A real bug in this area would pass the test suite.

Examples:
- Verifying a mock method was called as the primary assertion — tests wiring, not behavior
- Lenient/non-strict mocking applied globally — unused stubs hidden, tests drift from reality
- Test name says "integration" but all dependencies are mocked

### Medium
Anti-pattern that increases maintenance cost or reduces signal quality. Not immediately dangerous but degrades the suite over time.

Examples:
- Hardcoded IDs in test data — invisible coupling between tests
- Competing data setup (SQL seed + beforeEach) — unclear which record wins
- Incorrect assumption about transaction rollback scope with HTTP-level test clients

### Low
Style, naming, or minor improvements. Does not affect correctness.

Examples:
- Test names describe methods rather than behaviors
- No comment explaining why a test documents a known gap
- Missing arrange/act/assert structure making tests hard to scan

---

## Anti-Pattern Reference

These patterns were identified in real code reviews. Each one passed CI with green bars. For language-specific code examples, load the file for your project's language (see top of this document).

| Anti-Pattern | What Happens | What To Check | Severity |
|---|---|---|---|
| Fake Tenant Isolation | Test writes as Tenant A, reads as Tenant A. Passes even if isolation is disabled. | Does the test ever query as a *different* tenant and assert empty results? | Critical |
| Security Config Bypass | Test config permits all requests. Per-test auth setup is cosmetic decoration. | Read the global security config. Does it have a catch-all permit? | Critical |
| Verify-on-DAO | Asserting a data-access mock was called tests wiring, not behavior. Breaks on any rename. | Is mock-call verification the *only* assertion? Check return value or DB state instead. | High |
| Lenient/Non-Strict Mocking | Global non-strict setting hides unused stubs. Tests drift from the code they claim to cover. | Is lenient/non-strict mocking applied at class/file level? Remove it; unused stubs should fail loudly. | High |
| Competing Data Setup | SQL seed file + beforeEach both insert same rows. Silent collisions. Hardcoded IDs couple files. | Is there more than one setup mechanism inserting the same rows? Are IDs hardcoded integers? | Medium |
| FK-Unsafe Cleanup | Parent rows deleted before child rows. FK RESTRICT fires in afterEach, not in the test itself. | Does afterEach delete in FK-safe order (children before parents)? | High |
| Hardcoded Date Time Bombs | Past dates hardcoded in test input. Passes when written, fails months later. | Search for hardcoded year values in test date fields. Use `today + N days` instead. | High |
| Sync Assert on Async Code | Background job triggered, result asserted immediately. Race condition — flaky on CI. | Is async code triggered without a poll/wait before the assertion? | High |
| Array Assert on Paginated Response | Test asserts `response.data` is a flat array after endpoint switched to paginated wrapper. | Does the JSON path match the actual response shape? | Medium |

---

## Quick Reference Card

```
BIAS                    | COUNTER-QUESTION
----------------------- | ------------------------------------------
Green Bar               | What is NOT tested?
Coverage Number         | Are the assertions meaningful?
Volume                  | How many behaviors, not methods?
Test Name Trust         | Read the asserts — what do they actually check?
Mock Confidence         | What does this prove if the mock is removed?
Security Config         | Does the test config permit all? Check first.

LAYER                   | FIRST QUESTION
----------------------- | ------------------------------------------
Suite structure         | What's the unit:integration ratio?
Test file               | Is there a permissive security config or global lenient mocking?
Each test               | What is actually asserted?
Unit tests              | Is mock-call verification the primary assertion?
Controller tests        | Is auth disabled in test config?
Integration tests       | Is the second tenant ever queried?
Security tests          | Would the test fail if I deleted the auth check?

SEVERITY                | DEFINITION
----------------------- | ------------------------------------------
Critical                | Security/data-integrity risk, false assurance
High                    | False confidence — test looks good, proves nothing
Medium                  | Anti-pattern increasing maintenance cost
Low                     | Style, naming, minor
```
