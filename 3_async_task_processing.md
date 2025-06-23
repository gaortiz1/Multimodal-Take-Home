## 3. **Asynchronous Task Processing**

### 3.1. Problem Definition  
When designing an asynchronous task processing system, we face several key challenges:  
- **Decoupling request and execution**  
  - We must avoid blocking the client while heavy work (word counts, LLM calls) runs in the background.  
- **Scalability & parallelism**  
  - The system must handle many tasks in parallel and grow horizontally by adding more workers.  
- **Reliability & resilience**  
  - If a worker crashes or a transient error occurs, we need automatic retries and no lost tasks.  
- **Visibility of state**  
  - Clients must be able to query “Where is my task?” with clear, documented statuses.  
- **Error handling & timeouts**  
  - We must enforce time limits, distinguish fatal vs. transient errors, and report meaningful failure reasons.  

---

### 3.2. Candidate Patterns & Technologies  
To solve this we consider:  
- **Producer–Consumer (Work Queue)** – decouples task submission from execution via RabbitMQ  
- **Worker Pool** – processes tasks in parallel across a pool of Celery workers  
- **Batch Processing** – group N tasks into mini-batches for efficiency (e.g. Celery’s `chunks` or a custom batch size)  
- **Retry with Back-off** – automatically re-enqueue on transient failures  
- **Circuit Breaker / Timeout** – enforce max execution time per task  

---

### 3.3. Why Celery + RabbitMQ (Applying the Patterns)  
1. **Producer–Consumer (Work Queue)**  
   - FastAPI enqueues each `task_id` into a **durable** RabbitMQ queue.  

2. **Worker Pool & Batch Processing**  
   - Celery workers pull tasks with a **prefetch_count** (e.g. 5) to avoid hot-loading.  
   - We use Celery’s **`chunks`** to combine every N=10 tasks into a mini-batch, processing them in a single workflow step when beneficial (e.g., shared setup cost).

3. **Message Routing & Exchanges (AMQP)**  
   - RabbitMQ exchanges and routing keys can direct high-priority or specialized jobs to dedicated worker pools.

4. **Retry & Time Limiting**  
   - Celery retries transient errors up to 3 times with exponential delays, and applies soft/hard time limits (e.g. 120s) to stop runaway tasks.

5. **Observability**  
   - RabbitMQ’s Management UI shows queue depth; Celery’s monitoring tools (Flower) show task rates, latencies, and failures.

---

### 3.4. Addressing Key Concerns

1. **Idempotency**  
   - **Pattern:** Idempotent Consumer  
   - **Solution:** Workers check the DB status before processing; if already `PROCESSING` or `COMPLETED`, they skip re-execution.

2. **Retries**  
   - **Pattern:** Retry with Back-off  
   - **Solution:** Celery’s built-in retry tracks `retry_count` and, after max attempts, marks the task `FAILED` with the error.

3. **Race Conditions**  
   - **Pattern:** Message Acknowledgement + Locking  
   - **Solution:**  
     - RabbitMQ’s `acks_late=True` ensures tasks aren’t dropped on worker failure.  
     - Workers acquire a lightweight DB row lock before updating `PENDING → PROCESSING`, preventing concurrent execution.

---

### 3.5. Trade‐Offs

| Aspect                 | Pros                                                          | Cons                                                     |
|------------------------|---------------------------------------------------------------|----------------------------------------------------------|
| **Complexity**         | Rich feature set (retries, batching, routing, monitoring)     | Additional infrastructure to maintain (RabbitMQ, Celery) |
| **Scalability**        | Easy horizontal scaling via more workers or broker nodes      | Requires tuning prefetch counts and broker clustering    |
| **Reliability**        | Durable queues + acks_late + automatic retries                | Misconfiguration can lead to message duplication or loss |
| **Latency**            | Fast enqueue; workers handle load asynchronously              | Batch processing adds slight delay before execution      |
| **Observability**      | Native management UIs and metrics for both broker and workers | Monitoring these components adds operational overhead    |
| **Development effort** | Minimal custom code for retries and workflows                 | Learning curve for Celery and AMQP concepts              |


### 3.6. Task Lifecycle & State Management  
- **Submission → PENDING**  
  - Client calls `POST /v1/tasks`.  
  - The new task (`task_id`, status=`PENDING`) is saved in a database.  
  - FastAPI enqueues a message with `task_id` into the durable RabbitMQ queue and returns `{ task_id, status: "PENDING" }`.  
- **PENDING → PROCESSING**  
  - A Celery worker pulls the message from RabbitMQ, updates status → `PROCESSING`, and records a start timestamp.  
- **PROCESSING → COMPLETED / FAILED**  
  - The worker runs the analysis logic (see below).  
  - On success: status → `COMPLETED`, result payload attached, finish timestamp recorded.  
  - On fatal error or exhausted retries: status → `FAILED`, error message attached, finish timestamp recorded.  
- **Client Polling**  
  - Clients call `GET /v1/tasks/{task_id}` to fetch the current status and, once `COMPLETED`, the result or, if `FAILED`, the error details.

---

### 3.7. Text Processing Logic Implementation  
- **Simulated Delay**  
  - Pause 5–10 seconds to mimic heavy computation or external API latency.  
- **Tokenization & Filtering**  
  - Split text into words, normalize to lowercase.  
  - Remove common stop words from the token list.  
- **Word Count**  
  - Count the remaining tokens.  
- **Top-5 Words**  
  - Build a frequency map and select the five most frequent tokens.  
- **Result Object**  
  ```json
  {
    "word_count": 123,
    "top_words": ["data", "analysis", "async", "task", "system"]
  }
  
### 3.8. Error Handling During Processing  
- **Transient Errors**  
  - Automatically retried up to 3 times with exponential back-off (e.g., 1s, 2s, 4s).  
  - Each retry increments a `retry_count` for observability.  
- **Fatal Errors**  
  - Detected immediately (e.g., invalid input).  
  - Task is marked `FAILED` and the exception message is recorded.  
  - No further retries.  

---

### 3.9. Impact of Third-Party LLM API Integration  
- **Timeouts**  
  - Increase Celery’s soft and hard time limits (e.g., to 300s) to accommodate external API latency.  
- **Rate-Limit Back-off**  
  - On HTTP 429 responses, apply exponential back-off before retrying.  
- **Idempotency & Caching**  
  - Compute a hash of the input text; return a cached LLM response if available to avoid duplicate calls and extra cost.  
- **Async HTTP Calls**  
  - Use an async-capable HTTP client inside the Celery task, or consider an `asyncio`-based worker model for high-concurrency LLM requests.  

