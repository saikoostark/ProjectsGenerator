## 1. Project Title

Automated Uptime Monitor and Pager

## 2. Difficulty

Junior+

### Rationale
This project introduces infrastructure monitoring, scheduled tasks, and alert orchestration. The developer must build a service that concurrently probes endpoints, maintains state machines for monitoring health (e.g., UP, DOWN, FLAPPING), and integrates with external messaging APIs (like Slack or Twilio) to trigger alerts. It teaches concurrent job scheduling, robust error handling, and basic incident management logic.

## 3. Project Overview

The Automated Uptime Monitor and Pager is a service designed to periodically probe a list of URLs or TCP services for availability and responsiveness. It keeps a history of status checks and implements an incident-tracking state machine. When an endpoint changes status (e.g., from OK to DOWN) and stays down for a configurable period, the system notifies an administrator through an integration (e.g., Webhooks, Slack, or email). It includes a simple dashboard to manage monitored targets, view uptime percentages, and see incident history.

## 4. Problem Statement

System downtime is inevitable. Without proactive monitoring:
- A service could be down for hours without the owner knowing.
- Users are often the first to report issues, harming the system's reputation.
- Distinguishing between temporary network blips (flapping) and true outages is difficult without a state-aware monitor.

Building an uptime monitor teaches how to implement reliable background tasks, manage service health states, and orchestrate alerts efficiently.

## 5. Proposed Solution

The monitor consists of a scheduler, a probe engine, and an alert manager:
1. **Scheduler**: Runs probes at defined intervals (e.g., every minute) using a task queue.
2. **Probe Engine**: Concurrently executes HTTP requests (GET/HEAD) or TCP connection attempts.
3. **State Engine**: Analyzes results. If a service fails, it waits for a retry before declaring it DOWN to avoid false positives.
4. **Alert Manager**: If a service enters the DOWN state, it dispatches notifications to configured channels.

## 6. Project Goal

To build an uptime monitor that can track multiple services, accurately report downtime by filtering out transient network issues, and send alerts when a service is genuinely unreachable.

## 7. Core Workflow

```text
Config Service                      Scheduler Engine                       Alert Service
      │                                    │                                    │
      ├────1. Define Targets──────────────>│                                    │
      │                                    │                                    │
      │                                    ├─────2. Probe Task (Loop)──────────>│
      │                                    │                                    │
      │                                    │<────3. Probe Result (OK/FAIL)──────┤
      │                                    │                                    │
      │                                    │                                    │
      │                                    │──4. State Machine (Check Flapping)─>│
      │                                    │                                    │
      │                                    │────────5. Alert Dispatch──────────>│
```

## 8. Functional Requirements

### Probing
- **Support Types**: HTTP/HTTPS (URL, Method, Expected Code) and TCP (Address, Port).
- **Concurrency**: Probe multiple services in parallel without waiting for one to finish before starting the next.
- **Timeouts**: Configure timeouts for each probe.

### Incident Management
- **State Machine**: Implement states (e.g., UP, DOWN, FLAPPING/PENDING_DOWN).
- **Retries**: Configurable number of retries before marking DOWN.
- **Alerting**: Integrated notifier to send messages on status changes.

### Dashboard
- **Configuration**: CRUD for monitoring targets.
- **Reporting**: Display recent uptime statistics (e.g., 99.9%).

## 9. Non-Functional Requirements

- **Scalability**: Capable of monitoring 100+ services.
- **Reliability**: The monitor itself must be robust and provide heartbeat checks for its own health.

## 10. Main Entities / Data Model

### Service
- **ID**: UUID.
- **URL**: String.
- **Interval**: Integer.
- **Status**: Enum (UP, DOWN, PENDING).

### Incident
- **ID**: UUID.
- **ServiceID**: String.
- **StartTime**: DateTime.
- **EndTime**: DateTime.

## 11. System Components

- **Monitoring Service**: The core scheduler and probe worker.
- **Persistent Store**: Database to keep configuration and incident history.
- **Notifier**: Component for messaging integration.

## 12. Important Technical Challenges

### Handling False Positives (Flapping)
- **Challenge**: Network instability can cause a service to flip between UP and DOWN.
- **Solution**: Implement a "wait and retry" mechanism (e.g., only trigger alert if failure persists for 3 consecutive checks).

### Concurrency
- **Challenge**: Running many probes in parallel requires efficient task management.
- **Concepts**: Goroutines, Node.js worker threads, or `asyncio`.

## 13. Suggested Technology Areas

- **Backend**: Go (excellent for concurrent tasks) or Node.js.
- **Persistence**: PostgreSQL or SQLite.

## 14. Skills and Knowledge Gained

### Backend
- Background tasks and scheduling.
- HTTP client usage and TCP sockets.

### Infrastructure
- Monitoring principles and alerting logic.

## 15. Recommended Development Phases

1. **Phase 1 - Basic Prober**: Create a CLI script that takes a URL, probes it, and outputs "UP" or "DOWN".
2. **Phase 2 - Scheduler**: Add a loop that runs the prober every minute for a list of URLs.
3. **Phase 3 - State Machine & Alerting**: Add logic to handle retries and call a mock alert function on DOWN status.
4. **Phase 4 - Dashboard**: Build the CRUD web UI.

## 16. Testing Requirements

- **Integration**: Mock a service that flips between UP and DOWN to test state machine accuracy.

## 17. Security Considerations

- **Secure Configuration**: Store API keys/secrets for notification services securely.

## 18. Possible Extensions

- **Custom Headers**: Send custom headers/payloads in probes (e.g., for API auth).

## 19. Learning Questions

- How do you differentiate between a temporary network glitch and a genuine service outage?
- Why is it important for the monitor to run independently of the services being monitored?

## 20. Completion Criteria

- [ ] Monitors multiple services simultaneously.
- [ ] Correctly identifies UP/DOWN status.
- [ ] Implements retry logic to avoid false alarms.
- [ ] Dispatches alerts on status change.
- [ ] Web dashboard manages monitoring targets.