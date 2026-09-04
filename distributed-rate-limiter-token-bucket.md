## 1. Project Title

Distributed Rate Limiter (Token Bucket)

## 2. Difficulty

Mid-Level

### Rationale
This project is a classic systems design challenge. It requires implementing a robust rate-limiting algorithm that works across multiple application instances. The developer must use Redis for shared state, handle atomic operations using Lua scripts to prevent race conditions during token replenishment, and design an API middleware that is highly performant (low latency overhead).

## 3. Project Overview

The Distributed Rate Limiter is an HTTP middleware that limits the number of requests a client (by IP or UserID) can make to an API within a specified time window. It uses the "Token Bucket" algorithm, where each client is assigned a bucket that refills with tokens over time. Requests consume tokens, and if the bucket is empty, the request is rejected with a `429 Too Many Requests` status. Because the limiter must work across multiple backend instances, it uses Redis to maintain the token count globally.

## 4. Problem Statement

Public APIs are vulnerable to abuse, bot attacks, and resource exhaustion.
- Without rate limiting, a single rogue client can consume all available server resources, crashing the service for everyone else.
- Simply limiting requests per second on a single server doesn't work in distributed environments, where multiple servers (behind a load balancer) independently calculate usage.
- In-memory rate limiting is not synchronized across nodes, allowing clients to effectively bypass limits by hitting different servers.

Building a distributed rate limiter introduces developers to shared-state consistency and atomic operations in a distributed environment.

## 5. Proposed Solution

The system is designed as lightweight middleware for an API:
1. **Middleware Hook**: Every incoming request passes through the rate-limiter middleware.
2. **Redis Lua Script**: To ensure atomicity (the "check and update" operation must be atomic), the middleware sends a Lua script to Redis to:
    a. Fetch the client's current bucket count.
    b. Refill tokens based on time elapsed.
    c. Consume a token if available.
    d. Return the new count and whether the request is allowed.
3. **Response Handling**: If allowed, the request proceeds; if not, return `429 Too Many Requests`.

## 6. Project Goal

To build a high-performance, middleware-based rate limiter that consistently enforces request quotas across multiple distributed nodes.

## 7. Core Workflow

```text
Client Request      API Middleware      Redis (Lua Script)
      │                   │                   │
      ├────1. Request────>│                   │
      │                   │──2. Execute Lua──>│
      │                   │                   │  3. Calculate New Tokens
      │                   │                   │  4. Update Bucket
      │                   │<─5. Allow/Reject──┤
      │                   │                   │
      │<──6. Return Resp──┤                   │
```

## 8. Functional Requirements

### Rate Limiting Engine
- **Token Bucket Algorithm**: Implement the token bucket logic with configurable refill rate and bucket capacity.
- **Atomic Operations**: Ensure all updates in Redis are atomic.
- **Configurable Limits**: Support per-client (IP/User) limits.

### Middleware
- **Integration**: Build as a reusable middleware for an HTTP framework (e.g., Express, FastAPI, Gin).
- **HTTP Response**: Correctly set `429` status code and informative headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`).

## 9. Non-Functional Requirements

- **Performance**: The rate-limiter check must add negligible latency (e.g., <5ms overhead per request).
- **Consistency**: Limits must be strictly enforced even with 50+ concurrent request sources.

## 10. Main Entities / Data Model

### Redis Bucket Key
- **Key**: `ratelimit:<client_id>`
- **Value**: Two Redis keys (or one Hash): `tokens` (current count) and `last_updated` (timestamp).

## 11. System Components

- **Middleware**: Intercepts requests and calls Redis.
- **Lua Script**: Executes the Token Bucket logic within Redis for atomicity.
- **Redis Instance**: Stores client-specific bucket state.

## 12. Important Technical Challenges

### Atomicity & Lua
- **Challenge**: If you perform a `GET` followed by a `SET` in your code, another request might slip in between, causing a race condition and inaccurate token counts.
- **Concept**: Atomic operations, Redis Lua scripting.

### Redis Latency
- **Challenge**: Adding a network round-trip to Redis for every single API request can drastically increase latency.
- **Concept**: Local caching of limits (with eventual consistency) vs. strict Redis lookup.

## 13. Suggested Technology Areas

- **Backend**: Node.js, Go, or Python.
- **Datastore**: Redis.

## 14. Skills and Knowledge Gained

### Distributed Systems
- Distributed state management using Redis.
- Atomic operations and race condition avoidance.

### Networking
- HTTP middleware design.

## 15. Recommended Development Phases

1. **Phase 1 - Local Rate Limiter**: Build an in-memory token bucket rate limiter (single-node).
2. **Phase 2 - Redis Integration**: Move the bucket state to Redis using simple `GET`/`SET` commands.
3. **Phase 3 - Lua Scripting**: Rewrite the Redis logic using a Lua script to ensure atomicity.
4. **Phase 4 - Middleware**: Wrap the logic into a middleware function.

## 16. Testing Requirements

- **Concurrency**: Write a test that sends 100 simultaneous requests to see if the counter correctly reflects the limit (e.g., limit is 50).

## 17. Security Considerations

- **Redis Security**: Protect the Redis instance.
- **DoS Protection**: Ensure the rate limiter itself can't be DoS'd.

## 18. Possible Extensions

- **Distributed/Global Rate Limiter**: Limit total requests to the *entire* API (not just per-client).
- **Dynamic Policy**: Change limits on-the-fly without restarting the app.

## 19. Learning Questions

- Why is it necessary to run the Token Bucket logic in Lua inside Redis rather than in the application code?
- How does the Token Bucket algorithm handle bursts of traffic differently from a simple "Fixed Window" counter?

## 20. Completion Criteria

- [ ] Rate limiter is integrated as middleware.
- [ ] Requests over the limit return `429`.
- [ ] Bucket correctly refills over time.
- [ ] Multiple nodes accurately enforce the same limit for a single client.