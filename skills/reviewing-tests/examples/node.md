# Node.js / TypeScript Test Review Examples

Anti-patterns organized by review checklist layer. Each example shows the WRONG pattern and what to flag.

---

## Layer 3: Unit Tests

### Jest Mock Verification as Primary Assertion (Wrong)

```typescript
// WRONG — verifies wiring, not behavior
it("creates a goal", async () => {
  const mockRepo = { save: jest.fn().mockResolvedValue({ id: "uuid-1" }) };
  const service = new GoalService(mockRepo as any);

  await service.createGoal({ title: "Q1 target", tenantId: "tenant-a" });

  expect(mockRepo.save).toHaveBeenCalledTimes(1);  // tests implementation, not outcome
});
```

**Flag:** `toHaveBeenCalledTimes` on a repository mock is the primary assertion. If `save` is renamed to `persist`, this test breaks for the wrong reason. Check the return value or DB state instead.

---

### jest.mock at Module Level Silencing Type Errors (Wrong)

```typescript
// WRONG — jest.mock replaces entire module; real interface changes are invisible
jest.mock("../repositories/goalRepository");
const MockGoalRepository = GoalRepository as jest.MockedClass<typeof GoalRepository>;

// No autoMock spec — method signatures from the real class are not enforced
MockGoalRepository.prototype.save.mockResolvedValue(undefined as any);
```

**Flag:** `as any` casting on mock return values. When the real `save` signature changes (e.g., now returns `{ id: string }` instead of `void`), these casts hide the breakage. Use `jest.spyOn` with the real instance, or `ts-jest` with strict type checking.

---

## Layer 4: Controller / API Tests (Express / Fastify / NestJS)

### Middleware Bypass in Test Setup (Wrong)

```typescript
// WRONG — auth middleware removed from test app; every auth test is a false positive
const app = express();
app.use(express.json());
// authMiddleware intentionally excluded for "simplicity"
app.use("/api", goalRoutes);

it("returns 401 when no token", async () => {
  const res = await request(app).get("/api/goals");
  expect(res.status).toBe(401);  // NEVER 401 — middleware not mounted
});
```

**Flag:** Auth middleware excluded from the test Express/Fastify app. All 401 assertions pass vacuously because the server never checks for a token.

---

### No Error Path Coverage (Wrong)

```typescript
// WRONG — only the happy path is tested
describe("POST /api/goals", () => {
  it("creates a goal", async () => {
    const res = await request(app)
      .post("/api/goals")
      .set("Authorization", `Bearer ${validToken}`)
      .send({ title: "Q1 target" });
    expect(res.status).toBe(201);
  });
  // Missing: 401 (no token), 403 (wrong role), 400 (invalid body), 404 (tenant not found)
});
```

**Flag:** No tests for 401, 403, 400, or 404 for this endpoint. A real bug in any of those paths would pass CI undetected.

---

## Layer 5: Integration Tests

### Fake Tenant Isolation (Wrong)

```typescript
// WRONG — always passes even if row-level security is completely off
it("tenant sees own goals", async () => {
  await db.query(
    "INSERT INTO goals (title, tenant_id) VALUES ($1, $2)",
    ["Q1 target", "tenant-a"]
  );
  const rows = await goalRepo.findAll({ tenantId: "tenant-a" });
  expect(rows).toHaveLength(1);  // still querying as tenant-a — proves nothing
});
```

**Flag:** Test never switches to a second tenant. Add:

```typescript
const crossTenantRows = await goalRepo.findAll({ tenantId: "tenant-b" });
expect(crossTenantRows).toHaveLength(0);
```

---

### beforeEach + SQL Seed File Collision (Wrong)

```typescript
// WRONG — two insertion sources, hardcoded ID 1
// seed.sql: INSERT INTO users (id, email) VALUES (1, 'alice@example.com') ON CONFLICT DO NOTHING;

beforeEach(async () => {
  await db.query(
    "INSERT INTO users (id, email) VALUES ($1, $2) ON CONFLICT DO NOTHING",
    [1, "alice@example.com"]
  );
});
```

**Flag:** Both `seed.sql` and `beforeEach` insert `id = 1`. Hardcoded IDs couple this file to any other test that inserts the same row. Use `uuid()` and insert once via `beforeEach` only.

---

### Hardcoded Past Date (Wrong)

```typescript
// WRONG — fails once the date is in the past and the service rejects it
const res = await request(app)
  .post("/api/review-cycles")
  .send({ startDate: "2025-01-01", endDate: "2025-03-31" });
expect(res.status).toBe(201);
```

**Flag:** Hardcoded date. Use `new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString().slice(0, 10)`.

---

### Sync Assert on Async Job (Wrong)

```typescript
// WRONG — race condition; worker processes the message asynchronously
it("marks conversation as processed", async () => {
  await request(app).post("/api/conversations").send({ ... });
  const row = await db.query("SELECT status FROM conversations WHERE id = $1", [convId]);
  expect(row.rows[0].status).toBe("complete");  // job may not have run yet
});
```

**Flag:** No polling or waiting. Use a retry helper or a library like `wait-for-expect`. Assert inside a `waitFor` loop with a bounded timeout.

---

## Layer 6: Security Tests

### No Test for Missing Authorization Header (Wrong)

```typescript
// WRONG — only tests authenticated path; missing unauthenticated case
it("returns goals", async () => {
  const res = await request(app).get("/api/goals").set("Authorization", `Bearer ${validToken}`);
  expect(res.status).toBe(200);
});
// Missing: request without Authorization header asserting 401
```

**Flag:** If middleware is removed, CI stays green. Add the unauthenticated case.

---

### Guard/Decorator With No Failing Test (Wrong)

```typescript
// Production code
@UseGuards(RolesGuard)
@Roles("manager")
@Get("/admin/goals")
async getAdminGoals() { ... }

// Tests — only the happy path is present
it("manager can access admin goals", async () => { ... });
// Missing: employee asserting 403, unauthenticated asserting 401
```

**Flag:** If `@UseGuards` or `@Roles` were deleted, no test would fail. Add the negative cases.

---

### SKIP_AUTH Env Flag Disabling Auth (Wrong)

```typescript
// WRONG — bypasses all authentication in test environment
process.env.SKIP_AUTH = "true";

// authMiddleware: if (process.env.SKIP_AUTH === "true") return next();
```

**Flag:** Any env flag that short-circuits authentication. Every 401/403 test is a false positive. Use test JWTs signed with a known test secret instead.
