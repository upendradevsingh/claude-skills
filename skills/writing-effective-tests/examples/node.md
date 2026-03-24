# Node.js / TypeScript Test Examples (Jest & Vitest)

Language-specific examples for every concept in the main SKILL.md. Each section shows a WRONG and RIGHT pattern.

---

## Test Structure

### Arrange-Act-Assert with describe/it blocks

```typescript
// WRONG — multiple behaviors in one test, no structure
test("order", () => {
  const o = new Order([{ price: 50 }, { price: 60 }]);
  expect(o.total()).toBe(110);
  expect(o.applyDiscount(10).total()).toBe(99); // two acts = two tests
});

// RIGHT — one behavior per test, AAA, behavior-focused name
describe("Order.applyDiscount", () => {
  it("reduces total by discount percentage", () => {
    const order = new Order([{ price: 50 }, { price: 60 }]); // Arrange
    const discounted = order.applyDiscount(10);              // Act
    expect(discounted.total()).toBe(99);                      // Assert
  });
});
```

### beforeEach/afterEach and factory functions

```typescript
// WRONG — inline objects repeated everywhere; state leaks between tests
test("subscription", () => {
  const sub = { userId: "u1", plan: "pro", expiresAt: new Date("2020-01-01"), status: "active" };
  expect(canAccess(sub, "premium")).toBe(false);
});

// RIGHT — factory with defaults; beforeEach resets state
function buildSubscription(overrides: Partial<Subscription> = {}): Subscription {
  return {
    userId: "user-123", plan: "pro", status: "active",
    expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30d from now
    ...overrides,
  };
}

describe("canAccess", () => {
  afterEach(() => jest.clearAllMocks()); // or vi.clearAllMocks()

  it("blocks expired subscription from premium content", () => {
    const sub = buildSubscription({ expiresAt: new Date("2020-01-01") });
    expect(canAccess(sub, "premium")).toBe(false);
  });
});
```

---

## Mocking

### jest.fn() / vi.fn() — assert on output, not the stub

```typescript
// WRONG — asserting on a stub (verifying incoming data, not an outgoing side-effect)
it("loads user", async () => {
  const findUser = jest.fn().mockResolvedValue({ id: "1", name: "Alice" });
  await getProfile("1", findUser);
  expect(findUser).toHaveBeenCalledWith("1"); // proves nothing about behavior
});

// RIGHT — assert on the return value
it("returns profile with display name", async () => {
  const findUser = jest.fn().mockResolvedValue({ id: "1", name: "Alice" });
  const result = await getProfile("1", findUser);
  expect(result.displayName).toBe("Alice");
});
```

### jest.spyOn() for partial mocking

```typescript
// RIGHT — spy only on the method that crosses the system boundary (email = external)
import * as mailer from "../mailer";

it("sends welcome email after registration", async () => {
  const sendSpy = jest.spyOn(mailer, "sendEmail").mockResolvedValue(undefined);
  await registerUser({ email: "alice@example.com", password: "secret" });
  expect(sendSpy).toHaveBeenCalledWith(
    expect.objectContaining({ to: "alice@example.com", template: "welcome" })
  );
});
```

### Mock boundary: mock fetch/axios, NOT your own services

```typescript
// WRONG — mocking your own service tests wiring, not behavior
jest.mock("../services/userService");
(userService.getUser as jest.Mock).mockResolvedValue({ id: "u1" });

// RIGHT — mock only the external HTTP call; let your services run for real
import axios from "axios";
jest.mock("axios");

it("enriches order with CRM customer tier", async () => {
  (axios.get as jest.Mock).mockResolvedValue({ data: { tier: "gold" } });
  const order = await createOrder({ userId: "u1", items: [{ sku: "A", qty: 1 }] });
  expect(order.customerTier).toBe("gold");
});
```

---

## Database Testing (Testcontainers + Prisma/Drizzle/Knex)

### Real Postgres via @testcontainers/postgresql

```typescript
// WRONG — SQLite silently differs from Postgres (JSON operators, RETURNING, RLS, arrays)
const db = new Database(":memory:");

// RIGHT — same engine as production
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { PrismaClient } from "@prisma/client";

let prisma: PrismaClient;
beforeAll(async () => {
  const container = await new PostgreSqlContainer("postgres:16-alpine").start();
  process.env.DATABASE_URL = container.getConnectionUri();
  prisma = new PrismaClient();
  await prisma.$executeRawUnsafe("-- run migrations here");
}, 60_000);
afterAll(() => prisma.$disconnect());
```

### Transaction rollback per test

```typescript
// WRONG — deleteMany cleanup is slow and order-dependent
beforeEach(async () => { await prisma.user.deleteMany(); });

// RIGHT — roll back the transaction after each test; zero cleanup code
let tx: PrismaClient; // scoped to a transaction via prisma-test-environment or similar

beforeEach(async () => { tx = await beginTestTransaction(prisma); });
afterEach(async  () => { await rollbackTestTransaction(tx); });

it("creates a user", async () => {
  const user = await tx.user.create({ data: { email: "alice@example.com" } });
  expect(user.id).toBeDefined();
});
```

---

## API Testing (supertest)

### Test through HTTP, not through mocked req/res

```typescript
import request from "supertest";
import { app } from "../app";

// WRONG — mocking req/res tests middleware internals, not behavior
it("rejects unauthenticated", () => {
  const req = { headers: {} } as Request;
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() } as unknown as Response;
  authMiddleware(req, res, jest.fn());
  expect(res.status).toHaveBeenCalledWith(401); // brittle
});

// RIGHT — test through HTTP
it("returns 401 for unauthenticated request", async () => {
  const res = await request(app).get("/api/conversations");
  expect(res.status).toBe(401);
});
```

### Testing error paths and JWT helper

```typescript
// Auth token helper — keeps test bodies clean
function makeToken(overrides: Partial<JwtPayload> = {}): string {
  return jwt.sign(
    { tenantId: "tenant-test", userId: "user-test", ...overrides },
    process.env.JWT_SECRET!,
    { expiresIn: "1h" }
  );
}

it("returns 422 when required field is missing", async () => {
  const res = await request(app)
    .post("/api/conversations")
    .set("Authorization", `Bearer ${makeToken()}`)
    .send({});
  expect(res.status).toBe(422);
  expect(res.body.errors).toContainEqual(expect.objectContaining({ field: "text" }));
});

it("returns 401 for expired token", async () => {
  const expired = makeToken({ exp: Math.floor(Date.now() / 1000) - 3600 });
  const res = await request(app).get("/api/profile").set("Authorization", `Bearer ${expired}`);
  expect(res.status).toBe(401);
});
```

---

## Time Testing

### jest.setSystemTime() for fixed dates

```typescript
// WRONG — hardcoded past date is a time bomb
const sub = new Subscription({ expiresAt: new Date("2024-01-01") });
expect(sub.isExpired()).toBe(true); // relies on real clock being after 2024

// RIGHT — freeze the clock; always test the boundary too
beforeEach(() => jest.useFakeTimers());
afterEach(() => jest.useRealTimers());

it("flags subscription as expired when past expiry", () => {
  jest.setSystemTime(new Date("2026-03-24T12:00:00Z"));
  const sub = new Subscription({ expiresAt: new Date("2026-03-23T23:59:59Z") });
  expect(sub.isExpired()).toBe(true);
});

it("considers subscription active at exact expiry boundary", () => {
  jest.setSystemTime(new Date("2026-03-24T00:00:00Z"));
  const sub = new Subscription({ expiresAt: new Date("2026-03-24T00:00:00Z") });
  expect(sub.isExpired()).toBe(false);
});
```

---

## Async Testing

### Await everything; use rejects for error paths

```typescript
// WRONG — fire-and-forget gives a false pass regardless of outcome
it("saves conversation", () => {
  saveConversation({ text: "hello" }); // no await — errors are swallowed
  expect(true).toBe(true);
});

// RIGHT
it("persists extraction to database", async () => {
  await saveExtraction(extraction);
  const found = await db.extraction.findFirst({ where: { type: extraction.type } });
  expect(found).not.toBeNull();
});

it("throws NotFoundError for unknown conversation", async () => {
  await expect(getConversation("nonexistent-id")).rejects.toThrow("NotFoundError");
});
```

### Event emitters and streams

```typescript
// RIGHT — wrap in a Promise that resolves on the event
it("emits processed event with transcript text", async () => {
  const processor = new TranscriptionProcessor();
  const transcript = await new Promise<string>((resolve, reject) => {
    processor.on("processed", resolve);
    processor.on("error", reject);
    processor.start("audio.mp3");
  });
  expect(transcript).toContain("hello");
});
```

---

## Anti-Patterns

### Over-mocking internal modules

```typescript
// WRONG — mocking your own engine in an engine test
jest.mock("../services/extraction/engine");
it("processes conversation", async () => {
  (ExtractionEngine.prototype.extract as jest.Mock).mockResolvedValue([]);
  await processConversation("conv-1");
  expect(ExtractionEngine.prototype.extract).toHaveBeenCalled(); // tests wiring only
});

// RIGHT — run the real engine; mock only the LLM call at the external boundary
jest.mock("openai");
it("extracts ACTION_ITEM from conversation", async () => {
  (openai.chat.completions.create as jest.Mock).mockResolvedValue(fakeLlmResponse);
  const extractions = await processConversation("conv-1");
  expect(extractions).toContainEqual(expect.objectContaining({ type: "ACTION_ITEM" }));
});
```

### Snapshot testing abuse

```typescript
// WRONG — 200-line snapshot gets blindly updated on every UI tweak
expect(renderConversationList(conversations)).toMatchSnapshot();

// RIGHT — assert on specific meaningful properties
it("renders one card per conversation", () => {
  const { getAllByRole, getByText } = render(<ConversationList items={conversations} />);
  expect(getAllByRole("article")).toHaveLength(conversations.length);
  expect(getByText("Q1 Review")).toBeInTheDocument();
});

// Inline snapshots are acceptable for small, stable serialized outputs
it("serializes extraction to expected shape", () => {
  expect(serializeExtraction(extraction)).toMatchInlineSnapshot(
    `{ "type": "ACTION_ITEM", "text": "Follow up by Friday", "confidence": 0.92 }`
  );
});
```

### Shared database state between test files

```typescript
// WRONG — global seed creates hidden coupling; order-dependent, breaks parallelism
// jest.config.ts: globalSetup: "./seed.ts"

// RIGHT — each file creates and tears down its own isolated data
beforeEach(async () => {
  testTenant = await db.tenant.create({ data: { name: `tenant-${Date.now()}` } });
});
afterEach(async () => {
  await db.conversation.deleteMany({ where: { tenantId: testTenant.id } });
  await db.tenant.delete({ where: { id: testTenant.id } });
});
```
