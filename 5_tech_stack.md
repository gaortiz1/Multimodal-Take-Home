## 5. **Technology Stack**

### 5.1. Module Breakdown & Architecture

#### 1. API Module  
- **Architecture:** DDD + Event-Driven  
  - **Interfaces (Controllers):** FastAPI exposes REST endpoints and uses Pydantic for input validation.  
  - **Application (Use Cases):** Handles “create task” use case, instantiates a `Task` entity (status=`PENDING`), persists it, and emits an internal `TaskCreated` event.  
  - **Domain:** `Task` aggregate with its status transitions and any business rules.  
  - **Infrastructure:**  
    - **Repository Adapter:** implements read/write against PostgreSQL.  
    - **Event Publisher:** publishes the `TaskCreated` event into RabbitMQ.

#### 2. Worker Module  
- **Architecture:** Hexagonal (Ports & Adapters)  
  - **Core Port (`TaskProcessor`):** defines `process(taskId)` and encapsulates state transitions (`PROCESSING` → analysis → `COMPLETED`/`FAILED`).  
  - **Inbound Adapter:** Celery worker that consumes RabbitMQ messages and invokes `TaskProcessor.process()`.  
  - **Outbound Adapters:**  
    1. **PostgreSQL Repository:** updates task status, timestamps, result or error.  
    2. **LLM/Business Logic Adapter:** either a local word-count implementation or an HTTP client (httpx) to call a third-party LLM.  
  - **Configuration:** adapters are injected into the core at startup, making the business logic independent of Celery, RabbitMQ, or HTTP clients.

### 5.2. Core Technologies

- **Language:** Python 3.10+  
- **API Module:**  
  - **FastAPI** for high-performance async endpoints  
  - **Pydantic** for declarative data validation  
  - **SQLAlchemy + Alembic** for PostgreSQL access and migrations  
  - **Celery client** (via Kombu) to publish `task_id` messages to RabbitMQ  
- **Worker Module:**  
  - **Celery** with RabbitMQ broker (durable queues, `acks_late`)  
  - **PostgreSQL** as the authoritative task store  
  - **httpx** for any external HTTP/LLM calls

