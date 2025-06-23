## 6. **Scalability and Reliability**

### 6.1. Horizontal Scaling

- **API Layer**  
  - Deploy FastAPI behind a load-balanced cluster (Kubernetes Deployment or Fargate Service).  
  - Use Horizontal Pod Autoscaler (HPA) or AWS Application Auto Scaling to spin up/down API replicas based on CPU, memory, or request rate.  
  - Leverage async endpoints so each replica can handle many concurrent connections efficiently.

- **Worker Layer**  
  - Run Celery workers in a Deployment/Service that can scale replicas independently.  
  - Use KEDA (Kubernetes Event-Driven Autoscaling) or AWS Service Auto Scaling triggered by RabbitMQ queue length (e.g. messages-visible metric) to add workers when backlog grows.

- **Database Layer**  
  - RDS in AWS or serverless data base

---

### 6.2. Ensuring Reliability

- **Durable Queues & Acknowledgements**  
  - RabbitMQ queues are marked durable; messages survive broker restarts.  
  - Workers use `acks_late=True` so a task is only removed when acknowledged on success.

- **Retries & Dead-Letter Queues**  
  - Celery automatically retries transient failures up to N attempts with exponential back-off.  
  - Configure a RabbitMQ Dead-Letter Exchange/Queue (DLX/DLQ) to catch messages that exceed retry limits or hit fatal errors, isolating problematic tasks for manual intervention or alerting.

- **Worker Crash Recovery**  
  - If a worker crashes mid-task (before ack), RabbitMQ redelivers the message to another worker after its visibility timeout.  
  - Combined with idempotent processing and row-locking, this guarantees exactly-once or at-least-once semantics without duplicate side-effects.

- **Health Checks & Self-Healing**  
  - Liveness/readiness probes on API and worker pods ensure Kubernetes/ECS replaces unhealthy instances automatically.  
  - Monitor key metrics (queue length, retry rates, error rates) and trigger alerts when thresholds are crossed.
