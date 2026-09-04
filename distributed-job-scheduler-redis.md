## 1. Project Title

Distributed Job Scheduler with Redis

## 2. Difficulty

Mid-Level

### Rationale
This project requires managing complex asynchronous state across multiple instances. Using Redis as a centralized, high-performance data store forces the developer to implement distributed locking (to prevent concurrent job execution), handle job retries with exponential backoff, and manage task prioritization. It bridges the gap between simple background task queues and enterprise-grade distributed orchestrators.

## 3. Project Overview

The Distributed Job Scheduler is a resilient, multi-node backend system designed to execute scheduled and ad-hoc background tasks. It offloads heavy processing to worker nodes while maintaining a central job registry and execution log in Redis. The system ensures idempotency (jobs don't run twice) even when multiple worker nodes compete for the same job, and handles task failures by implementing retry policies and dead-letter queues.

## 4. Problem Statement

Many modern applications require delayed, recurring, or complex background tasks (sending emails, processing data reports, cleanup scripts). 
- Simple single-process task queues fail when the system needs to scale across multiple instances.
- Multiple instances can inadvertently pick up the same job simultaneously, leading to race conditions.
- If a task crashes, without a central job-tracking mechanism, it remains lost, leaving the system in an inconsistent state.

Providing a reliable, scalable job scheduler is a common engineering challenge that necessitates understanding distributed synchronization and robustness patterns.

## 5. Proposed Solution

The system consists of a central Redis store, one or more worker nodes, and a management API:
1. **Job Definition**: Tasks are defined with a name, payload, scheduled time, and retry policy.
2. **Central Registry**: Redis acts as the source of truth, storing job queues and execution states.
3. **Distributed Locking**: Workers use Redis atomic operations (e.g., `SETNX` or Redlock) to "claim" a job, ensuring only one worker executes it.
4. **Execution**: Workers poll for available jobs, execute them, and update the status in Redis.
5. **Failover**: If a worker crashes mid-execution, the system detects the abandoned lock and moves the job back to the queue for retry.

## 6. Project Goal

To build a job scheduler that guarantees "at-least-once" execution, manages task prioritization, supports delayed/recurring jobs, and provides an API to track job status, all backed by a resilient Redis-based architecture.

## 7. Core Workflow

```text
Management API                      Redis Store                       Worker Nodes
      │                                  │                                  │
      ├──1. Push Job (Enqueue)──────────>│                                  │
      │                                  │<──2. Poll Available Jobs (Loop)──┤
      │                                  │──3. Try Acquire Lock (SETNX)────>│
      │                                  │                                  │
      │                                  │<──4. Lock Acquired (Success)─────┤
      │                                  │                                  │
      │                                  │                           5. Execute Job
      │                                  │                                  │
      │                                  │──6. Update Job Status (Done)────>│
      │                                  │                                  │
```

## 8. Functional Requirements

### Job Management
- **Ad-hoc/Delayed Jobs**: Ability to enqueue a job for immediate execution or at a specific delay.
- **Recurring Jobs**: Ability to enqueue cron-like jobs that repeat based on a schedule.
- **Task Prioritization**: Support job priority levels, ensuring high-priority jobs are picked up before low-priority ones.

### Execution Engine
- **Distributed Locking**: Prevent concurrent execution of the same job across worker nodes.
- **Retry Policy**: Configure automatic retries with exponential backoff on failure.
- **Dead-Letter Queue (DLQ)**: Move persistently failing jobs to a separate DLQ for manual inspection after exceeding maximum retries.

### Monitoring
- **API Status**: Expose endpoints to check the status, history, and error logs of any job.

## 9. Non-Functional Requirements

- **Scalability**: Can add/remove worker nodes dynamically without downtime.
- **Reliability**: Jobs must never be lost, even if a worker crashes during execution.
- **Observability**: Every job state transition (pending, running, failed, completed) must be auditable.

## 10. Main Entities / Data Model

### Job
- **ID**: UUID.
- **Payload**: JSON (data for the worker).
- **Status**: Enum (Pending, Running, Completed, Failed).
- **ScheduledAt**: DateTime.
- **Retries**: Integer (current retry count).
- **MaxRetries**: Integer.

## 11. System Components

- **Management API**: REST interface for enqueuing/monitoring jobs.
- **Redis Queue Store**: Redis Sorted Sets (`ZSET`) or Lists used for job queues.
- **Worker Node**: Daemon that polls Redis, attempts job acquisition, and executes task logic.

## 12. Important Technical Challenges

### Distributed Locking
- **Challenge**: How to ensure two workers don't grab the same job? Using `SETNX` (Set if Not Exists) is standard, but how long should the lock last?
- **Concepts**: Distributed lock timeout, lease renewal (heartbeats).

### Job Polling vs. Push
- **Challenge**: Constant polling (`WHILE(TRUE)`) is inefficient.
- **Research**: Use Redis `BLPOP` or `BRPOPLPUSH` for blocking pops to reduce latency and load.

## 13. Suggested Technology Areas

- **Backend**: Go or Node.js.
- **Datastore**: Redis (with `ioredis` or standard `go-redis`).

## 14. Skills and Knowledge Gained

### Systems Design
- Distributed consensus/coordination basics.
- Job queue mechanics and reliable delivery patterns.

### DevOps
- Redis architecture for high availability.

## 15. Recommended Development Phases

1. **Phase 1 - Redis Queue**: Build basic Enqueue and Dequeue functions in Redis using simple lists.
2. **Phase 2 - Worker Engine**: Build a worker node that polls for a job and executes a simple task (e.g., console log).
3. **Phase 3 - Locking**: Add `SETNX` lock functionality to prevent multiple workers from executing the same job.
4. **Phase 4 - Retries & DLQ**: Implement failure handling, incrementing retry count, and moving to DLQ.
5. **Phase 5 - API & Monitoring**: Build the REST API to enqueue and monitor jobs.

## 16. Testing Requirements

- **Integration Tests**: Spin up 3 worker nodes. Enqueue 100 jobs and verify each job is executed exactly once across the cluster.
- **Fault Tolerance**: Kill a worker node mid-job execution. Verify the lock expires and another worker automatically picks up the job for retry.

## 17. Security Considerations

- **Redis Security**: Secure Redis with a password and bind to local interface.
- **Job Payload Sanitization**: Validate job payloads before execution to prevent arbitrary code execution (ACE) vulnerabilities.

## 18. Possible Extensions

- **Cron-like Syntax**: Support standard Cron syntax for complex recurring schedules.
- **UI Dashboard**: A React-based web dashboard to visualize the queues, active workers, and DLQs in real-time.

## 19. Learning Questions

- How does a distributed lock guarantee exclusive access across independent nodes?
- What is the difference between "at-least-once" and "exactly-once" delivery? 
- Why might a job remain in the queue even if a worker "claims" it?

## 20. Completion Criteria

- [ ] Job Enqueue API works.
- [ ] Multiple workers process jobs without duplication.
- [ ] Failed jobs are retried based on policy.
- [ ] Failed jobs end up in the Dead-Letter Queue.
- [ ] Monitoring API reports accurate job states.