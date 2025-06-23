## 2. **API Design**

### **API Versioning**

We version the API in the URL path using a clear semantic prefix (`/v1/`) and maintain a `__version__ = "1.0.0"` in the codebase. This strategy:

- **Ensures backward compatibility**: Old clients continue to work with `/v1/` while new clients can migrate to `/v2/` when breaking changes occur.
- **Is human-readable and cache-friendly**: URL-based versioning is easy to understand and works seamlessly with HTTP caches.
- **Supports parallel maintenance**: Multiple versions (e.g., `/v1/tasks`, `/v2/tasks`) can coexist during rollout.
- **Simplifies documentation**: Each version has its own OpenAPI spec and changelog.

```yaml
# Example path versioning in OpenAPI
paths:
  /v1/tasks:
    # v1 endpoints...
  /v2/tasks:
    # v2 endpoints with breaking changes...



### **- Submit a Task**

#### 📌 Endpoint
POST /v1/tasks

#### 💡 What does it do?
This endpoint receives a piece of text and creates an asynchronous task to process it (e.g., word count, frequent words). It returns a unique task ID and a `PENDING` status immediately.

---

### 📥 Request Payload

```json
{
  "text": "Multimodal is building AI-powered tools to automate complex tasks."
}
```

| Field | Type   | Required | Description                          | Constraints       |
|-------|--------|----------|--------------------------------------|-------------------|
| text  | string | ✅ yes   | Text to be processed asynchronously. | Max length: 4000  |

---

### 📤 Synchronous Response

```json
{
  "task_id": "4f147a96-3902-4d9e-a6a2-13eaa39f5c18",
  "status": "PENDING"
}
```

| Field    | Type   | Description                        |
|----------|--------|------------------------------------|
| task_id  | string | UUID v4 identifier for the task    |
| status   | string | Always `"PENDING"` at submission   |

---

### ✅ How do you ensure `task_id` is unique?

Task IDs are generated using [UUID version 4](https://datatracker.ietf.org/doc/html/rfc4122), which produces statistically unique identifiers with no coordination between services. In Python:

```python
import uuid
task_id = str(uuid.uuid4())
```

UUIDs are:
- Globally unique
- Stateless
- Ideal for distributed systems

---

### 📄 Swagger Spec (OpenAPI YAML)

```yaml
paths:
  /v1/tasks:
    post:
      summary: Submit a new text processing task
      operationId: submitTask
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - text
              properties:
                text:
                  type: string
                  description: Text to be processed
                  maxLength: 4000
      responses:
        '200':
          description: Task accepted
          content:
            application/json:
              schema:
                type: object
                properties:
                  task_id:
                    type: string
                    format: uuid
                    description: Unique ID of the submitted task
                  status:
                    type: string
                    enum: [PENDING]
                    description: Initial task status
        '400':
          description: Bad Request – missing or invalid input
        '422':
          description: Unprocessable Entity – text too long or semantically invalid
        '500':
          description: Internal Server Error
```

### **- Check Task Status & Results**

#### 📌 Endpoint  
`GET /v1/tasks/{task_id}`

#### 💡 What does it do?  
This endpoint allows the client to check the status of a previously submitted task and retrieve its result (if completed). It supports tracking of the task lifecycle from submission to processing and final output.

---

### 🔄 Possible Task Status Values

| Status      | Meaning                                                   |
|-------------|-----------------------------------------------------------|
| `PENDING`   | Task has been accepted but not yet picked by a worker     |
| `PROCESSING`| Task is currently being executed                          |
| `COMPLETED` | Task has been processed successfully and has a result     |
| `FAILED`    | An error occurred during task execution                   |

---

### 📤 Response Examples

#### ✅ 1. Task Still Processing

```json
{
  "task_id": "4f147a96-3902-4d9e-a6a2-13eaa39f5c18",
  "status": "PROCESSING"
}
```

#### ✅ 2. Task Completed with Results

```json
{
  "task_id": "4f147a96-3902-4d9e-a6a2-13eaa39f5c18",
  "status": "COMPLETED",
  "result": {
    "text": "Multimodal is building AI-powered tools to automate complex tasks.",
    "word_count": 17,
    "top_words": ["ai", "tools", "automate", "multimodal", "tasks"]
  }
}
```

#### ❌ 3. Task Failed

```json
{
  "task_id": "4f147a96-3902-4d9e-a6a2-13eaa39f5c18",
  "status": "FAILED",
  "error": "Timeout when calling LLM API"
}
```

---

### ❓ What if the `task_id` doesn’t exist?

Return an HTTP `404 Not Found` with a clear message:

```json
{
  "error": "Task with ID 'xyz' not found"
}
```

---

### 📄 Swagger Spec (OpenAPI YAML)

```yaml
paths:
  /v1/tasks/{task_id}:
    get:
      summary: Get task status and result
      operationId: getTaskStatus
      parameters:
        - name: task_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
          description: UUID of the task
      responses:
        '200':
          description: Task status and optional result
          content:
            application/json:
              schema:
                type: object
                required:
                  - task_id
                  - status
                properties:
                  task_id:
                    type: string
                    format: uuid
                  status:
                    type: string
                    enum: [PENDING, PROCESSING, COMPLETED, FAILED]
                  result:
                    type: object
                    properties:
                      text:
                        type: string
                      word_count:
                        type: integer
                      top_words:
                        type: array
                        items:
                          type: string
                  error:
                    type: string
                    description: Error message if the task failed
        '404':
          description: Task not found
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                    example: "Task with ID 'xyz' not found"
        '500':
          description: Internal server error
```

### **List Tasks**

#### 📌 Endpoint  
`GET /v1/tasks`

#### 💡 What does it do?  
This endpoint returns a paginated list of previously submitted tasks. Clients can filter by status and page through results.

---

### 🔍 Query Parameters

| Name    | In     | Type    | Required | Description                              |
|---------|--------|---------|----------|------------------------------------------|
| status  | query  | string  | no       | Filter tasks by status (`PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`) |
| page    | query  | integer | no       | Page number (default: 1)                 |
| limit   | query  | integer | no       | Tasks per page (default: 10, max: 100)   |

---

### 📤 Success Response (`200 OK`)

```json
{
  "tasks": [
    {
      "task_id": "4f147a96-3902-4d9e-a6a2-13eaa39f5c18",
      "status": "COMPLETED",
      "created_at": "2025-06-22T14:20:00Z",
      "updated_at": "2025-06-22T14:21:00Z"
    },
    {
      "task_id": "7b3c2d44-5f6d-4a9c-bb98-1f2a3d4e5f67",
      "status": "PENDING",
      "created_at": "2025-06-22T14:22:00Z",
      "updated_at": "2025-06-22T14:22:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 42,
    "pages": 5
  }
}
```

| Field       | Type     | Description                                                      |
|-------------|----------|------------------------------------------------------------------|
| tasks       | array    | List of task summary objects                                      |
| pagination  | object   | Pagination metadata                                              |
| └ page      | integer  | Current page number                                              |
| └ limit     | integer  | Number of items per page                                         |
| └ total     | integer  | Total number of tasks                                            |
| └ pages     | integer  | Total number of pages (`ceil(total / limit)`)                    |

Each task object:

| Field       | Type    | Description                          |
|-------------|---------|--------------------------------------|
| task_id     | string  | UUID of the task                     |
| status      | string  | Current status of the task           |
| created_at  | string  | ISO 8601 timestamp when created      |
| updated_at  | string  | ISO 8601 timestamp of last update    |

---

### ❓ What if no tasks match?

Return `200 OK` with an empty `tasks` array and appropriate pagination:

```json
{
  "tasks": [],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 0,
    "pages": 0
  }
}
```

---

### 📄 Swagger Spec (OpenAPI YAML)

```yaml
paths:
  /v1/tasks:
    get:
      summary: List tasks with pagination and filtering
      operationId: listTasks
      parameters:
        - name: status
          in: query
          required: false
          schema:
            type: string
            enum: [PENDING, PROCESSING, COMPLETED, FAILED]
          description: Filter tasks by status
        - name: page
          in: query
          required: false
          schema:
            type: integer
            default: 1
          description: Page number
        - name: limit
          in: query
          required: false
          schema:
            type: integer
            default: 10
            maximum: 100
          description: Number of tasks per page
      responses:
        '200':
          description: A list of tasks and pagination info
          content:
            application/json:
              schema:
                type: object
                required:
                  - tasks
                  - pagination
                properties:
                  tasks:
                    type: array
                    items:
                      type: object
                      properties:
                        task_id:
                          type: string
                          format: uuid
                        status:
                          type: string
                          enum: [PENDING, PROCESSING, COMPLETED, FAILED]
                        created_at:
                          type: string
                          format: date-time
                        updated_at:
                          type: string
                          format: date-time
                  pagination:
                    type: object
                    properties:
                      page:
                        type: integer
                      limit:
                        type: integer
                      total:
                        type: integer
                      pages:
                        type: integer
        '500':
          description: Internal server error
```
### **List Tasks**

#### 📌 Endpoint  
`GET /v1/tasks`

#### 💡 What does it do?  
Returns a paginated list of previously submitted tasks. Clients can filter by status, sort by certain fields, and page through results.

---

### 🔍 Query Parameters

| Name    | In    | Type    | Required | Description                                                             | Default |
|---------|-------|---------|----------|-------------------------------------------------------------------------|---------|
| status  | query | string  | no       | Filter tasks by status (`PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`) | —       |
| page    | query | integer | no       | Page number (must be ≥ 1)                                               | 1       |
| limit   | query | integer | no       | Tasks per page (1–100)                                                  | 10      |
| sort_by | query | string  | no       | Field to sort by (`created_at`, `updated_at`, `status`)                 | status  |
| order   | query | string  | no       | Sort direction (`asc` or `desc`)                                        | desc    |

- **Default sort**: newest first by `created_at desc`.

[//]: # (---)

### 📤 Success Response (`200 OK`)

```json
{
  "tasks": [
    {
      "task_id": "4f147a96-3902-4d9e-a6a2-13eaa39f5c18",
      "status": "COMPLETED",
      "created_at": "2025-06-22T14:20:00Z",
      "updated_at": "2025-06-22T14:21:00Z"
    },
    {
      "task_id": "7b3c2d44-5f6d-4a9c-bb98-1f2a3d4e5f67",
      "status": "PENDING",
      "created_at": "2025-06-22T14:22:00Z",
      "updated_at": "2025-06-22T14:22:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 42,
    "pages": 5
  }
}
