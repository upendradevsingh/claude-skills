# Python Testing Examples

Practical pytest examples for every concept in SKILL.md. Each section shows a WRONG and RIGHT pattern.

---

## Test Structure (pytest + AAA)

**WRONG** — no structure, implicit assertions, magic values inline:
```python
def test_extraction():
    e = ExtractionEngine("peakperf")
    result = e.run({"transcript": "John missed Q1 targets"})
    assert result
```

**RIGHT** — explicit AAA, named fixtures, clear assertion:
```python
@pytest.fixture
def performance_engine():
    return ExtractionEngine(profile_name="performance")

def test_extracts_risk_from_missed_targets(performance_engine):
    # Arrange
    transcript = {"segments": [{"text": "John missed Q1 targets", "speaker": "manager"}]}

    # Act
    extractions = performance_engine.extract(transcript)

    # Assert
    risks = [e for e in extractions if e.type == "RISK"]
    assert len(risks) == 1
    assert risks[0].attributes["severity"] == "high"
```

**Factory function** for test data (prefer over shared fixtures for entities):
```python
def make_conversation(tenant_id="tenant-1", status="pending", **kwargs):
    return Conversation(
        id=uuid4(),
        tenant_id=tenant_id,
        status=ConversationStatus(status),
        source_type="text",
        **kwargs,
    )
```

---

## Mocking (unittest.mock)

**WRONG** — patching in the wrong module, no spec:
```python
@patch("unittest.mock.openai.ChatCompletion.create")
def test_calls_llm(mock_create):
    mock_create.return_value = {"choices": [{"message": {"content": "[]"}}]}
    ...
```

**RIGHT** — patch where it's *imported*, use `spec=`, use `AsyncMock` for coroutines:
```python
from unittest.mock import AsyncMock, MagicMock, patch

@patch("app.services.extraction.engine.openai_client")
async def test_engine_calls_llm_with_profile_prompt(mock_client):
    mock_client.chat.completions.create = AsyncMock(return_value=MagicMock(
        choices=[MagicMock(message=MagicMock(content='[{"type":"RISK","text":"...","attributes":{}}]'))]
    ))

    engine = ExtractionEngine(profile_name="performance")
    result = await engine.extract({"segments": [{"text": "Missed targets", "speaker": "A"}]})

    mock_client.chat.completions.create.assert_awaited_once()
    assert result[0].type == "RISK"
```

**When NOT to mock** — if you can use a real object cheaply, do it:
```python
# WRONG: mocking ProfileLoader when it just reads a YAML file
mock_loader = MagicMock()
mock_loader.load.return_value = {...}

# RIGHT: use the real loader with a temp file
def test_profile_loader_merges_tenant_overrides(tmp_path):
    yaml_file = tmp_path / "performance.yaml"
    yaml_file.write_text("extraction_types:\n  - RISK\n  - FOLLOW_UP\n")
    loader = ProfileLoader(profiles_dir=tmp_path)
    profile = loader.load("performance")
    assert "RISK" in profile.extraction_types
```

---

## Database Testing (SQLAlchemy + Testcontainers)

**Session-scoped Postgres container** — start once per test session:
```python
# conftest.py
from testcontainers.postgres import PostgresContainer
import pytest

@pytest.fixture(scope="session")
def pg_container():
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture(scope="session")
def engine(pg_container):
    url = pg_container.get_connection_url()
    eng = create_engine(url)
    Base.metadata.create_all(eng)
    yield eng
    eng.dispose()
```

**Function-scoped rollback** — each test gets a clean slate:
```python
@pytest.fixture
def db_session(engine):
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()
    connection.close()
```

**WRONG** — SQLite in tests against Postgres production code:
```python
# SQLite silently ignores Postgres-specific types (JSONB, UUID, RLS)
engine = create_engine("sqlite:///:memory:")
```

**RIGHT** — always use real Postgres via testcontainers:
```python
# PostgresContainer gives you an isolated real Postgres, no install needed
# JSONB operators, UUID comparisons, and RLS all behave correctly
```

---

## API Testing (FastAPI + httpx)

**Dependency override** for the test database session:
```python
# conftest.py
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.db.session import get_db

@pytest.fixture
async def client(db_session):
    app.dependency_overrides[get_db] = lambda: db_session
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

**JWT auth helper** — generate a real signed token, not a mock:
```python
import jwt

def make_token(tenant_id="tenant-1", user_id="user-1", secret="test-secret"):
    return jwt.encode({"tenant_id": tenant_id, "user_id": user_id}, secret, algorithm="HS256")
```

**Testing error paths:**
```python
async def test_returns_404_for_missing_conversation(client):
    token = make_token()
    resp = await client.get(
        "/api/v1/conversations/00000000-0000-0000-0000-000000000000",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert resp.status_code == 404

async def test_returns_401_without_token(client):
    resp = await client.get("/api/v1/conversations/")
    assert resp.status_code == 401

async def test_returns_422_for_invalid_payload(client):
    token = make_token()
    resp = await client.post(
        "/api/v1/conversations",
        json={"source_type": "invalid_type"},  # not in enum
        headers={"Authorization": f"Bearer {token}"},
    )
    assert resp.status_code == 422
```

---

## Time Testing (freezegun)

**WRONG** — patching `datetime` directly breaks isinstance checks in libraries:
```python
with patch("app.services.auth.datetime") as mock_dt:
    mock_dt.utcnow.return_value = datetime(2026, 1, 1)
    ...
```

**RIGHT** — use `freeze_time` to freeze the whole process clock:
```python
from freezegun import freeze_time

@freeze_time("2026-01-01T00:00:00Z")
def test_expired_token_rejected():
    token = make_token_expiring_at("2025-12-31T23:59:59Z")
    with pytest.raises(ExpiredTokenError):
        validate_jwt(token)

@freeze_time("2026-01-01T00:00:00Z")
def test_token_expiring_exactly_now_is_rejected():
    # boundary: expires_at == now should be treated as expired
    token = make_token_expiring_at("2026-01-01T00:00:00Z")
    with pytest.raises(ExpiredTokenError):
        validate_jwt(token)
```

---

## Async Testing (pytest-asyncio)

**Setup in pyproject.toml:**
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

**WRONG** — mixing event loop scopes causes "Future attached to different loop":
```python
@pytest.fixture(scope="session")
async def client():  # session-scoped async fixture breaks with default event loop scope
    ...
```

**RIGHT** — keep async fixtures at `function` scope (default), or explicitly set loop scope:
```python
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"

@pytest.fixture
async def client(db_session):  # function-scoped, matches event loop scope
    ...

async def test_async_extraction(client):
    resp = await client.post("/api/v1/conversations", json={...}, headers={...})
    assert resp.status_code == 202
```

**AsyncMock vs MagicMock:**
```python
# Use AsyncMock for anything you need to `await`
mock_llm = AsyncMock(return_value="extracted json")
result = await mock_llm("prompt")  # works

# MagicMock raises TypeError if awaited — catches incorrect usage early
mock_sync = MagicMock()
await mock_sync()  # TypeError: object MagicMock is not awaitable
```

---

## Anti-Patterns in Python

**WRONG** — mocking `db.add` / `db.commit` instead of using a real session:
```python
def test_saves_conversation():
    db = MagicMock()
    service = ConversationService(db)
    service.create(payload)
    db.add.assert_called_once()  # proves nothing about correctness
```

**RIGHT** — use a real session with rollback, assert on the queried row:
```python
def test_saves_conversation(db_session):
    service = ConversationService(db_session)
    conv = service.create(ConversationCreate(source_type="text", profile="performance"))
    fetched = db_session.get(Conversation, conv.id)
    assert fetched.status == ConversationStatus.pending
```

**WRONG** — shared mutable fixture mutated across tests (ordering-dependent failures):
```python
@pytest.fixture(scope="module")
def conversations():
    return []  # tests appending to this list bleed into each other
```

**RIGHT** — use a factory or function-scoped fixture that returns a fresh object:
```python
@pytest.fixture
def conversations():
    return []  # fresh list per test
```

**WRONG** — loose mock assertions miss argument changes:
```python
mock_llm.assert_called()  # passes even if called with wrong profile
```

**RIGHT** — assert on the exact arguments that matter:
```python
call_kwargs = mock_llm.call_args.kwargs
assert call_kwargs["model"] == "gpt-4o-mini"
assert "RISK" in call_kwargs["messages"][0]["content"]
```
