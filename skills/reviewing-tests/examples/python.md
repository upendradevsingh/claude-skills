# Python Test Review Examples

Anti-patterns organized by review checklist layer. Each example shows the WRONG pattern and what to flag.

---

## Layer 3: Unit Tests

### Mocking the Repository (Wrong)

```python
# WRONG — mocking your own class, verifying call instead of behavior
def test_create_goal(mock_goal_repo):
    mock_goal_repo.save.return_value = Goal(id=1, title="Increase revenue")
    service = GoalService(goal_repo=mock_goal_repo)

    service.create_goal(title="Increase revenue", tenant_id="tenant-a")

    mock_goal_repo.save.assert_called_once()  # tests wiring, not behavior
```

**Flag:** `assert_called_once()` on a repository mock is the primary assertion. There is no check on return value or DB state. If `save` is renamed to `persist`, this test breaks for the wrong reason.

---

### Lenient Patching with autospec=False (Wrong)

```python
# WRONG — patch without autospec; method signature changes are invisible
@patch("app.services.goal_service.GoalRepository")
def test_creates_goal(MockRepo):
    MockRepo.return_value.save.return_value = None
    # If GoalRepository.save signature changes, this test still passes
```

**Flag:** `@patch` without `autospec=True` on managed dependencies. If the real method signature changes, the mock silently accepts any call.

---

## Layer 4: Controller / API Tests

### Permissive Auth Fixture (Wrong)

```python
# WRONG — fixture overrides all auth, making every security assertion cosmetic
@pytest.fixture(autouse=True)
def disable_auth(monkeypatch):
    monkeypatch.setattr("app.dependencies.verify_token", lambda token: {"tenant_id": "t1", "role": "admin"})

# Every test below now runs as admin regardless of what token is sent.
def test_employee_cannot_access_admin(client):
    response = client.get("/api/admin/users", headers={"Authorization": "Bearer bad_token"})
    assert response.status_code == 403  # NEVER reached — fixture returns admin for any token
```

**Flag:** `autouse=True` fixture that always returns a privileged identity. 403 and 401 tests are false positives.

---

### Missing Status Code Assertions (Wrong)

```python
# WRONG — only checks response body, not HTTP status
def test_create_goal(client, auth_headers):
    response = client.post("/api/goals", json={"title": "Q1 target"}, headers=auth_headers)
    data = response.json()
    assert data["title"] == "Q1 target"   # passes even if server returns 201, 200, or 500
```

**Flag:** No `assert response.status_code == 201`. A 500 that happens to return a matching body would pass.

---

## Layer 5: Integration Tests

### Fake Tenant Isolation (Wrong)

```python
# WRONG — always passes even if row-level isolation is disabled
def test_tenant_sees_own_goals(db_session):
    db_session.add(Goal(title="G1", tenant_id="tenant-a"))
    db_session.commit()

    set_tenant_context("tenant-a")
    goals = GoalRepository(db_session).find_all()
    assert len(goals) == 1   # still querying as tenant-a — proves nothing
```

**Flag:** Test never switches to a second tenant. Rename to `test_tenant_a_cannot_see_tenant_b_goals` and add:

```python
set_tenant_context("tenant-b")
assert len(GoalRepository(db_session).find_all()) == 0
```

---

### SQLite in a PostgreSQL Shop (Wrong)

```python
# WRONG — conftest uses SQLite
@pytest.fixture
def db_engine():
    return create_engine("sqlite:///:memory:")
```

**Flag:** Production is PostgreSQL. SQLite differs on JSON operators, `ARRAY` types, `ON CONFLICT` syntax, and constraint enforcement. Integration tests on SQLite give false confidence.

---

### Hardcoded Past Date (Wrong)

```python
# WRONG — fails once 2025-01-01 is in the past and the service rejects past dates
def test_create_review_cycle(client, auth_headers):
    response = client.post("/api/review-cycles", json={
        "start_date": "2025-01-01",
        "end_date": "2025-03-31"
    }, headers=auth_headers)
    assert response.status_code == 201
```

**Flag:** Hardcoded dates. Use `datetime.date.today() + timedelta(days=30)`.

---

### Sync Assert on Async Task (Wrong)

```python
# WRONG — race condition; Celery task runs in background
def test_conversation_processed(client, auth_headers):
    response = client.post("/api/conversations", json={...}, headers=auth_headers)
    conv_id = response.json()["id"]

    result = db.query(Extraction).filter_by(conversation_id=conv_id).first()
    assert result is not None   # task may not have run yet
```

**Flag:** No polling or waiting. Use `pytest-celery` task eager mode or poll with timeout:

```python
from tenacity import retry, stop_after_delay, wait_fixed

@retry(stop=stop_after_delay(10), wait=wait_fixed(0.5))
def _wait_for_extraction(conv_id):
    result = db.query(Extraction).filter_by(conversation_id=conv_id).first()
    assert result is not None
```

---

## Layer 6: Security Tests

### No Test for Missing Token (Wrong)

```python
# WRONG — tests valid auth but never tests missing/invalid token
def test_get_goals_authenticated(client, auth_headers):
    response = client.get("/api/goals", headers=auth_headers)
    assert response.status_code == 200

# Missing: what happens with no auth header at all?
```

**Flag:** No corresponding test for `client.get("/api/goals")` (no headers) asserting `status_code == 401`.

---

### No Test for Wrong Tenant Token (Wrong)

```python
# WRONG — never tests that tenant-b token cannot read tenant-a data
def test_get_goal(client, tenant_a_headers):
    goal_id = create_goal(tenant_id="tenant-a")
    response = client.get(f"/api/goals/{goal_id}", headers=tenant_a_headers)
    assert response.status_code == 200

# Missing: client.get(f"/api/goals/{goal_id}", headers=tenant_b_headers) → 404
```

**Flag:** Cross-tenant 404 case is unverified. If object-level isolation is broken, tenant-b would see tenant-a's data.
