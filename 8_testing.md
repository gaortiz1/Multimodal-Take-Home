## 8. **Testing Strategy**

### 8.1. Overall Testing Approach

The goal of our testing strategy is to ensure correctness, resilience, and integration across system components—especially in an asynchronous, distributed architecture.

We follow a **test pyramid** approach:

- **Unit Tests (Base Layer):** Fast, isolated tests for individual functions/methods.
- **Integration Tests (Middle Layer):** Validate interactions between components (e.g., FastAPI ↔ DB, Celery ↔ RabbitMQ).
- **End-to-End Tests (Top Layer):** Simulate full flows from API request to task completion and result delivery.

---

### 8.2. Types of Tests & Priorities

| Test Type       | Purpose                                                    | Priority | Tools                               |
|-----------------|------------------------------------------------------------|----------|-------------------------------------|
| **Unit Tests**  | Validate pure logic: word count, stop-word filtering, etc. | ✅ High   | `pytest`                            |
| **Integration** | Test Celery ↔ RabbitMQ, DB writes, API ↔ DB                | ✅ High   | `pytest`, `testcontainers`, `httpx` |
| **E2E Tests**   | Simulate full task lifecycle via HTTP API                  | ✅ High   | `pytest`, `httpx`                   |
| Load Tests      | Assess how the system behaves under high task volume       | Medium   | `Locust`, `k6`                      |
| Contract Tests  | Ensure compatibility if other services depend on this API  | Low      | `pact`, `schemathesis`              |

---

### 8.3. Most Critical Components to Test

- **Task lifecycle transitions**
  - Ensure status moves correctly from `PENDING → PROCESSING → COMPLETED/FAILED`.
  - Test retry behavior and idempotency logic.

- **Text processing logic**
  - Validate correct word counting, stop word removal, and top 5 frequency.

- **Failure handling**
  - Simulate transient errors (e.g. LLM timeout, DB disconnection) and ensure proper retries or failure state.

- **Queue integrity**
  - Verify that tasks are enqueued exactly once and not lost or duplicated.
  - Ensure `acks_late` behavior with Celery/RabbitMQ.

- **API contract**
  - Test request validation (Pydantic), required fields, and proper error messages.

---

### 8.4. Additional Practices

- **CI Integration:** Run all tests on every commit using GitHub Actions or GitLab CI.
- **Mocking external dependencies:** Use mocks for LLM APIs and DBs in unit tests to isolate failures.
- **Dockerized test environment:** Use Docker Compose or `testcontainers` to spin up RabbitMQ, PostgreSQL, and Celery locally for full-stack integration testing.

---