=== FILENAME: docs/daemon.md ===

Daemon: Runtime, Schedulers, and Workers

This document provides a detailed technical overview of the OCVD's runtime environment, its internal scheduling mechanisms, and the worker architecture.

Daemon Runtime

The OCVD is a long-running, stateful service implemented in a systems language (e.g., Rust, Go) for performance and memory safety. It is designed to be run as a containerized application (Docker) within a secure, isolated environment.

· Startup Sequence:
  1. Load environment variables and configuration file.
  2. Initialize logging and telemetry (metrics, tracing).
  3. Establish connections: PostgreSQL (audit logs), NATS (message queue), Blockchain Node (via RPC).
  4. Fetch and cache on-chain governance parameters.
  5. Spawn scheduler and worker pools.
  6. Start API server and begin listening for jobs.
· Shutdown Sequence: On receiving a termination signal, the daemon stops accepting new jobs, waits for active workers to complete (with a configurable timeout), flushes logs, and gracefully closes all connections.

Schedulers

The scheduler is the brain of the operation, responsible for job distribution and load management.

· Job Intake: The scheduler listens to the main job queue. It validates the incoming request's basic structure and assigns it a unique, traceable Job ID.
· Pipeline Routing: Based on the job_type field (e.g., KYC, CREDIT_ASSESSMENT), the scheduler routes the job to the appropriate pipeline's work queue.
· Rate Limiting & Prioritization: The scheduler enforces global and per-pipeline rate limits to prevent resource exhaustion. It also supports priority queues (e.g., high for time-sensitive compliance checks, low for batch scoring) to ensure critical tasks are processed first.
· Dead Letter Queue (DLQ): Jobs that repeatedly fail processing are moved to a DLQ for manual inspection and debugging.

Workers

Workers are the execution engines. They are stateless, single-purpose, and managed in pools for concurrency.

· Worker Lifecycle:
  1. Poll: The worker listens to its assigned pipeline queue.
  2. Claim: It claims a job, marking it as IN_PROGRESS.
  3. Fetch Rules: It retrieves the latest governance ruleset from the local cache (which is periodically synced with the chain).
  4. Execute: It runs the pipeline logic. This may involve calling internal libraries (e.g., a document parser) or external, trusted services (e.g., a sanctions list API).
  5. Produce Result: The output is a structured JobResult containing the verification status, any generated proofs, and public outputs.
  6. Acknowledge: The worker publishes the JobResult to a results topic and acknowledges to the queue that the job is complete.
· Worker Pools: For each pipeline (e.g., KYC, Credit), a configurable pool of workers runs concurrently. This allows for horizontal scaling within a single daemon instance. Pool sizes are dynamically adjustable based on system load.

Event Queues

The OCVD uses a high-performance message broker (NATS) for all internal communication. This decouples components and ensures reliability.

· jobs.> Topics: For submitting new jobs. (e.g., jobs.kyc, jobs.credit.income).
· results.> Topics: For workers to publish their results. (e.g., results.kyc.completed, results.credit.failed).
· governance.> Topics: For broadcasting updates from the governance monitor (e.g., governance.rules.updated, governance.registry.address.changed).

Configuration Surfaces

The daemon is configured via a combination of a static file and environment variables.

```yaml
# config/ocvd.toml
[server]
host = "0.0.0.0"
port = 8080

[queues]
url = "nats://localhost:4222"

[workers]
max_concurrent = 50

[workers.pools.kyc]
size = 10
timeout_seconds = 120

[workers.pools.credit]
size = 5
timeout_seconds = 300

[database]
audit_log_url = "postgresql://..."

[blockchain]
rpc_url = "${ETH_RPC_URL}" # From environment variable
chain_id = 1

[governance]
core_contract = "0x..."
sync_interval_seconds = 60
```
