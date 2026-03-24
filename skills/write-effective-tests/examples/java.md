# Java / Spring Boot / JUnit 5 — Test Examples

Language-specific examples for every concept in SKILL.md. Patterns drawn from real Spring Boot reviews.

---

## Test Structure (JUnit 5)

### Arrange-Act-Assert with @DisplayName

```java
@Test
@DisplayName("expired subscription cannot access premium content")
void expiredSubscription_cannotAccessPremiumContent() {
    // Arrange
    Subscription sub = new Subscription(LocalDate.of(2025, 1, 1));
    Content content = new Content(Tier.PREMIUM);

    // Act
    boolean result = sub.canAccess(content);

    // Assert
    assertThat(result).isFalse();
}
```

### @Nested classes + @BeforeEach

```java
@ExtendWith(MockitoExtension.class)
class GoalServiceTest {
    @Mock GoalRepository goalRepository;
    @InjectMocks GoalService goalService;

    private Goal testGoal;

    @BeforeEach
    void setUp() {
        testGoal = GoalFactory.build(); // factory, not hardcoded IDs
    }

    @Nested @DisplayName("access control")
    class AccessControl {
        @Test void expiredSubscription_deniesAccess() { ... }
        @Test void activeSubscription_grantsAccess() { ... }
    }

    @Nested @DisplayName("renewal")
    class Renewal {
        @Test void renewal_extendsExpiry() { ... }
    }
}
```

---

## Mocking (Mockito)

### verify() — correct (outgoing side effect at boundary)

```java
// RIGHT — email is an external side effect; return value can't confirm it was sent
@Test
void passwordReset_sendsEmailToUser() {
    service.requestPasswordReset("alice@example.com");
    verify(emailClient).send(argThat(msg -> msg.getTo().equals("alice@example.com")));
}
```

### verify() — wrong (asserting on a DAO stub)

```java
// WRONG — tests wiring, breaks if impl switches to batchInsert()
@Test
void createGoal_callsDao() {
    when(goalDao.insert(any())).thenReturn(savedGoal);
    service.createGoal(request);
    verify(goalDao).insert(any(Goal.class));
}

// RIGHT — check observable output or real DB state
@Test
void createGoal_returnsPersistedGoal() {
    GoalResponse response = service.createGoal(request);
    assertThat(response.getId()).isNotNull();
    assertThat(goalRepository.findById(response.getId())).isPresent();
}
```

### ArgumentCaptor for inspecting call arguments

```java
@Test
void createUser_sendsVerificationEmail_withToken() {
    service.createUser(new UserRequest("alice@example.com"));

    ArgumentCaptor<EmailMessage> captor = ArgumentCaptor.forClass(EmailMessage.class);
    verify(emailClient).send(captor.capture());

    assertThat(captor.getValue().getBody()).contains("/verify?token=");
}
```

### Strictness.LENIENT is a code smell

```java
// WRONG — suppresses "unnecessary stubbing" globally; dead setup accumulates silently
@MockitoSettings(strictness = Strictness.LENIENT)
class GoalServiceTest {
    @Test void getGoal_returnsGoal() {
        when(userDao.findById(any())).thenReturn(Optional.of(mockUser)); // never called — silent
        when(goalDao.findById(1L)).thenReturn(Optional.of(mockGoal));
        assertThat(goalService.getGoal(1L)).isNotNull();
    }
}

// RIGHT — STRICT_STUBS by default; only stub what the path actually calls
class GoalServiceTest {
    @Test void getGoal_returnsGoal() {
        when(goalDao.findById(1L)).thenReturn(Optional.of(mockGoal));
        assertThat(goalService.getGoal(1L)).isNotNull();
    }
}
```

---

## Database Testing (Testcontainers + Spring)

### @Testcontainers + PostgreSQLContainer

```java
@Testcontainers
@SpringBootTest
class GoalRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine").withReuse(true);

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### @Transactional rollback — JPA only

```java
// Works when service and test share Spring's DataSourceTransactionManager (JPA/Hibernate)
@Transactional
@SpringBootTest
class GoalRepositoryTest {
    @Autowired GoalRepository goalRepository;

    @Test void save_persistsGoal() {
        Goal goal = GoalFactory.build();
        goalRepository.save(goal);
        assertThat(goalRepository.findById(goal.getId())).isPresent();
    } // rolled back — no cleanup needed
}
```

### Explicit cleanup for non-JPA layers (JDBI / jOOQ / JDBC)

`@Transactional` deadlocks when JDBI/jOOQ acquire their own connections. Use explicit cleanup in FK-safe order.

```java
@Execution(ExecutionMode.SAME_THREAD) // prevent parallel data collisions
class OrderIntegrationTest extends BaseIntegrationTest {
    @BeforeEach void setUp() { /* insert test data */ }

    @AfterEach void cleanUp() {
        // children before parents — avoids FK constraint violations
        db.execute("DELETE FROM order_items WHERE tenant_id = ?", TENANT_ID);
        db.execute("DELETE FROM orders WHERE tenant_id = ?", TENANT_ID);
    }
}
```

**Why NOT H2:** H2 silently differs from Postgres on JSON operators, array types, window functions, and constraints. Use Testcontainers with the real Postgres version.

---

## API Testing (MockMvc + Spring Boot Test)

### @WebMvcTest for controller slice tests

```java
@WebMvcTest(GoalController.class)
class GoalControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean GoalService goalService;

    @Test
    @WithMockUser(roles = "MANAGER")
    void getGoal_returnsGoal() throws Exception {
        when(goalService.getGoal(1L)).thenReturn(GoalFactory.buildResponse());
        mockMvc.perform(get("/api/goals/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.data.id").value(1));
    }
}
```

### Testing error paths (401, 403, 404, 422)

```java
@Test
void unauthenticated_returns401() throws Exception {
    mockMvc.perform(get("/api/goals")).andExpect(status().isUnauthorized());
}

@Test
@WithMockUser(roles = "EMPLOYEE")
void employee_cannotAccessAdmin_returns403() throws Exception {
    mockMvc.perform(get("/api/admin/users")).andExpect(status().isForbidden());
}

@Test
@WithMockUser
void missingRequiredField_returns422() throws Exception {
    mockMvc.perform(post("/api/goals").contentType(APPLICATION_JSON)
           .content("""{"title":""}"""))
           .andExpect(status().isUnprocessableEntity())
           .andExpect(jsonPath("$.errors[0].field").value("title"));
}
```

### Why TestSecurityConfig with permitAll() defeats auth testing

```java
// WRONG — every @PreAuthorize is invisible; removing one breaks no test
@TestConfiguration
public class TestSecurityConfig {
    @Bean SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
        return http.build();
    }
}

// RIGHT — keep real security; drive roles via @WithMockUser
@Test
@WithMockUser(roles = "EMPLOYEE")
void employee_cannotDeleteGoal() throws Exception {
    mockMvc.perform(delete("/api/goals/1")).andExpect(status().isForbidden());
}
```

---

## Integration Testing (Spring Boot)

### @SpringBootTest(webEnvironment = RANDOM_PORT) + base class

```java
@Testcontainers
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
public abstract class BaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine").withReuse(true);

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    protected HttpHeaders buildAuthHeaders() {
        HttpHeaders h = new HttpHeaders();
        h.setBearerAuth(JwtTestUtils.generateToken("tenant-a", "user-1"));
        return h;
    }
}
```

### TestRestTemplate vs MockMvc

`MockMvc` runs in-process — `@Transactional` rollback works. `TestRestTemplate` makes real HTTP calls in a separate transaction — service-layer writes are NOT rolled back. Use explicit cleanup when using `TestRestTemplate`.

### TestDataBuilder / factory pattern

```java
public class GoalFactory {
    public static Goal build() {
        return Goal.builder()
            .id(UUID.randomUUID())
            .title("Goal " + System.nanoTime())       // unique per run (safe with withReuse)
            .tenantId("tenant-a")
            .startDate(LocalDate.now().plusDays(1))   // relative — never stale
            .endDate(LocalDate.now().plusMonths(3))
            .build();
    }
}
```

---

## Anti-Patterns in Java/Spring (from real PMS review)

### All DAOs mocked + verify(dao).insert()

```java
// WRONG — breaks when impl switches to batchInsert(); proves nothing about behavior
@Test void createUser_callsDao() {
    userService.createUser(new UserRequest("alice@example.com"));
    verify(userDao).insert(any(User.class));
}

// RIGHT
@Test void createUser_persistsAndReturnsId() {
    UserResponse resp = userService.createUser(new UserRequest("alice@example.com"));
    assertThat(resp.getId()).isNotNull();
    assertThat(userRepository.findById(resp.getId())).isPresent();
}
```

### TestSecurityConfig disabling @PreAuthorize

```java
// WRONG — removing any @PreAuthorize from a controller breaks no test
http.authorizeHttpRequests(a -> a.anyRequest().permitAll());

// RIGHT
@Test @WithMockUser(roles = "EMPLOYEE")
void employee_cannotDeleteGoal() throws Exception {
    mockMvc.perform(delete("/api/goals/1")).andExpect(status().isForbidden());
}
```

### Fake tenant isolation tests

```java
// WRONG — creates as tenant-a, queries as tenant-a; passes even if RLS is fully broken
@Test void tenantCanReadOwnData() {
    setTenant("tenant-a");
    repo.save(new Goal("target", "tenant-a"));
    assertThat(repo.findAll()).hasSize(1); // always passes
}

// RIGHT — verify cross-tenant invisibility
@Test void tenantCannotReadAnotherTenantData() {
    setTenant("tenant-a");
    repo.save(new Goal("target", "tenant-a"));

    setTenant("tenant-b");
    assertThat(repo.findAll()).isEmpty(); // fails if RLS is broken
}
```

### Competing data setup (SQL seed + @BeforeEach)

```java
// WRONG — data.sql and @BeforeEach both insert id=1 with ON CONFLICT DO NOTHING;
// hardcoded IDs couple unrelated test files
@BeforeEach void setUp() {
    jdbcTemplate.update(
        "INSERT INTO users (id, email) VALUES (1, 'alice@example.com') ON CONFLICT DO NOTHING");
}

// RIGHT — one mechanism, generated IDs
@BeforeEach void setUp() {
    testUser = userRepository.save(UserFactory.build()); // UUID, no hardcoded IDs
}
```

### Strictness.LENIENT hiding unused stubs

```java
// WRONG — global lenient; dead stubs accumulate and mask drift
@MockitoSettings(strictness = Strictness.LENIENT)
class GoalServiceTest { ... }

// RIGHT — default STRICT_STUBS; apply lenient() per stub only when genuinely needed
lenient().when(dao.findById(any())).thenReturn(Optional.empty()); // targeted, not global
```
