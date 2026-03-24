# Java Test Review Examples

Anti-patterns organized by review checklist layer. Each example shows the WRONG pattern and what to flag.

---

## Layer 3: Unit Tests

### Verify-on-DAO (Wrong)

```java
// WRONG — verifies wiring, not behavior
@Test
void createGoal_callsDao() {
    service.createGoal(new GoalRequest("Q1 target", "tenant-a"));
    verify(goalDao).insert(any(Goal.class));  // breaks if impl switches to batchInsert()
}
```

**Flag:** `verify(dao).method()` as primary assertion. There is no check on the return value or DB state. This test survives a bug that saves the wrong data, as long as `insert` is called.

---

### LENIENT Strictness Hiding Drift (Wrong)

```java
// WRONG — suppresses unused-stub warnings globally
@MockitoSettings(strictness = Strictness.LENIENT)
class GoalServiceTest {
    @Mock GoalRepository goalRepo;
    @Mock NotificationService notificationService; // never called in half the tests

    // stubs accumulate without failing loudly
}
```

**Flag:** `@MockitoSettings(strictness = Strictness.LENIENT)` at the class level. Remove it. If unused stubs now fail, that is the point — those stubs should be deleted or the tests re-examined.

---

## Layer 4: Controller / API Tests

### TestSecurityConfig Disables All Auth (Wrong)

```java
// WRONG — every @PreAuthorize / @Secured annotation is now invisible
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
    return http.build();
}
```

**Flag:** `anyRequest().permitAll()` in test security config. All 403 and 401 tests are false positives. The fix is to import the real `SecurityConfig` and use `@WithMockUser(roles = "ROLE_X")` to drive roles explicitly.

---

### @WithMockUser Alongside Permissive Config (Wrong)

```java
// WRONG — @WithMockUser is cosmetic when config allows everything
@Test
@WithMockUser(roles = "EMPLOYEE")
void employee_cannot_access_admin() throws Exception {
    mockMvc.perform(get("/api/admin/users"))
           .andExpect(status().isForbidden());  // NEVER 403 — config permits all
}
```

**Flag:** Mentally remove `@WithMockUser`. If the test still reaches `isForbidden()`, the annotation is meaningless — the real enforcement is gone.

---

## Layer 5: Integration Tests

### Fake Tenant Isolation (Wrong)

```java
// WRONG — always passes even if RLS is completely disabled
void tenantCanReadOwnGoals() {
    setTenantContext("tenant-a");
    repo.save(new Goal("Q1 target", "tenant-a"));
    assertThat(repo.findAll()).hasSize(1);  // still as tenant-a, proves nothing
}
// RIGHT — add cross-tenant check
setTenantContext("tenant-b");
assertThat(repo.findAll()).isEmpty();
```

---

### @Transactional with Non-JPA Data Access (Wrong)

```java
// WRONG — deadlocks: Spring holds connection A; JDBI acquires connection B → table lock
@Transactional
class OrderIntegrationTest extends BaseIntegrationTest {
    @BeforeEach void setUp() {
        jdbi.useHandle(h -> h.execute("INSERT INTO orders ..."));
    }
}

// RIGHT — explicit @AfterEach cleanup instead of @Transactional rollback
@AfterEach void cleanUp() {
    jdbcTemplate.update("DELETE FROM orders WHERE tenant_id = ?", TENANT_ID);
}
```

**Flag:** `@Transactional` on a test class that also uses JDBI, jOOQ, or raw JDBC. **Severity: Critical** under load.

---

### FK-Unsafe Cleanup Order (Wrong)

```java
// WRONG — FK violation: order_items reference orders
@AfterEach void cleanUp() {
    db.execute("DELETE FROM orders WHERE tenant_id = ?", TENANT_ID);     // parent first → FK error
    db.execute("DELETE FROM order_items WHERE tenant_id = ?", TENANT_ID);
}
// RIGHT: delete order_items first, then orders
```

**Flag:** Delete children before parents in afterEach.

---

### Hardcoded Date Time Bombs (Wrong)

```java
// WRONG — fails once 2025-01-01 is in the past
.startDate(LocalDate.of(2025, 1, 1))
// RIGHT
.startDate(LocalDate.now().plusMonths(1))
```

**Flag:** Search for hardcoded year values in test date fields. Time bombs.

---

### @Transactional with TestRestTemplate (Wrong)

```java
// WRONG — test-side transaction rolls back but server-side already committed separately
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Transactional
class GoalApiTest {
    @Autowired TestRestTemplate restTemplate;
    // ...
}
```

**Flag:** `@Transactional` on a `TestRestTemplate` test. The two transaction contexts are separate. Use explicit `@AfterEach` cleanup.

---

## Layer 5: Integration Tests (continued)

### Competing Data Setup (Wrong)

```java
// WRONG — two insertion mechanisms, hardcoded ID 1
// data.sql: INSERT INTO users (id, email) VALUES (1, 'alice@example.com') ON CONFLICT DO NOTHING;
@BeforeEach void setUp() {
    jdbcTemplate.update("INSERT INTO users (id, email) VALUES (1, 'alice@example.com') ON CONFLICT DO NOTHING");
}
```

**Flag:** Both `data.sql` and `@BeforeEach` insert the same row with hardcoded `id = 1`. Use `userRepository.save(UserFactory.build())` with a generated UUID.

---

## Layer 6: Security Tests

### @PreAuthorize With No Failing Test (Wrong)

```java
// Production: @PreAuthorize("hasRole('MANAGER')") on updateGoal()

// Test — only the authorized path
@WithMockUser(roles = "MANAGER")
void manager_can_update_goal() throws Exception { ... }
// Missing: @WithMockUser(roles = "EMPLOYEE") asserting isForbidden()
```

**Flag:** If `@PreAuthorize` is deleted, nothing fails in CI. Add the forbidden case.

---

### Sync Assert on @Async Code (Wrong)

```java
// WRONG — race condition; @Async method may not have completed
mockMvc.perform(post("/api/conversations/" + id + "/process"));
assertEquals("COMPLETE",
    jdbcTemplate.queryForObject("SELECT status FROM conversations WHERE id = ?", String.class, id));

// RIGHT — poll with Awaitility
await().atMost(10, SECONDS).until(() ->
    "COMPLETE".equals(jdbcTemplate.queryForObject(...)));
```

**Flag:** Async code triggered with no wait/poll before the assertion.
