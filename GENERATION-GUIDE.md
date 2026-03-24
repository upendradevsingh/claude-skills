# Generation Guide: Adding Language-Specific Examples

## Purpose

Skills encode language-agnostic principles (mock at boundaries, use real databases in tests, control time explicitly). Each skill ships with language-specific example files that show those principles applied to the idioms, frameworks, and tooling of that language. New languages can be added by creating example files that follow the established pattern — no changes to the core skill are needed.

## Structure

```
skills/<skill-name>/examples/<language>.md
```

Examples: `skills/writing-effective-tests/examples/go.md`, `skills/reviewing-tests/examples/rust.md`

## Template: `writing-effective-tests` Examples

Cover these sections in order, matching the structure of the main `SKILL.md`:

### 1. Test Structure
Show the AAA (Arrange / Act / Assert) pattern using the language's primary test framework. Each section must include a WRONG example (5–15 lines), a RIGHT example (5–15 lines), and a 1–2 sentence explanation.

### 2. Mocking
Show the language's mocking library. Demonstrate mock boundary rules: mock at the edge of your system, not inside it.

### 3. Database Testing
Show Testcontainers setup (or equivalent) and the transaction rollback pattern for test isolation.

### 4. API Testing
Show HTTP endpoint testing covering success and error paths (401, 403, 404, 422).

### 5. Time Testing
Show how to freeze or inject time so tests are deterministic.

### 6. Async Testing
If the language has async primitives (goroutines, async/await, futures), show how to test them correctly.

### 7. Anti-Patterns
List 2–4 common mistakes specific to that language's ecosystem, each with WRONG and RIGHT.

---

Each section follows this format:

````markdown
### Section Title

**WRONG**
```<lang>
# 5–15 lines showing the anti-pattern
```

**RIGHT**
```<lang>
# 5–15 lines showing the correct approach
```

One or two sentences explaining why the right approach is better.
````

## Template: `reviewing-tests` Examples

Cover these four topics, same WRONG/RIGHT/explanation format:

1. Mock boundary violations — mocking internal collaborators instead of external edges
2. Security/auth testing gaps — missing 401/403 coverage, assuming happy path only
3. Database testing anti-patterns — in-memory fakes that diverge from real DB behavior
4. Test isolation issues — shared state leaking between tests

## Supported Languages

| Language | Test Framework | Mocking | DB Testing | API Testing | Time |
|----------|----------------|---------|------------|-------------|------|
| Python | pytest | unittest.mock, AsyncMock | Testcontainers + SQLAlchemy | httpx TestClient | freezegun |
| Java | JUnit 5 | Mockito | Testcontainers + @Transactional | MockMvc, WebTestClient | Clock.fixed() |
| Node.js/TS | Jest, Vitest | jest.mock, vi.mock | @testcontainers/postgresql | supertest | jest.useFakeTimers |

When adding a new language, add its row to this table in the same PR.

## Quality Checklist

Before submitting a new language's examples:

- [ ] All sections from the template are covered
- [ ] Every section has a WRONG example and a RIGHT example
- [ ] Code is realistic — not toy examples, not `foo`/`bar`
- [ ] File is under 300 lines
- [ ] Mocking examples respect the same boundary rules as the main skill (mock external edges, not internals)
- [ ] The supported languages table above is updated

## How to Contribute

1. Create `skills/<skill-name>/examples/<language>.md`
2. Follow the template above, section by section
3. Pressure-test the file: give an agent a testing task in that language once without the skill loaded, once with — verify the output improves in the areas the skill targets
4. Submit a PR with the example file and the updated languages table in this guide
