## 1. Project Title

Reverse Proxy and Load Balancer

## 2. Difficulty

Mid-Level

### Rationale
This project requires a deep understanding of low-level HTTP and TCP socket networking. Designing a reverse proxy forces developers to handle connection streams, manipulate HTTP headers, manage connection pools, implement concurrent load balancing algorithms (Round Robin, Weighted Least Connections), set up active background health checks, and implement fault tolerance, retries, and rate limiting.

## 3. Project Overview

The Reverse Proxy and Load Balancer is a lightweight, high-performance gateway daemon. It intercepts client HTTP/HTTPS requests at the edge of a network, inspects the request headers or paths, selects an optimal backend server from a dynamic pool of web servers, and streams the request and response between the client and backend. It features load-balancing algorithms, SSL/TLS termination, background health checkers, dynamic routing, and a real-time monitoring dashboard to track routing and backend load.

## 4. Problem Statement

Single server configurations fail to meet modern scalability and availability standards. If a single server suffers high traffic, it slows down or crashes; if it undergoes maintenance, the system is offline. 

Without a reverse proxy/load balancer:
- Clients must know the specific IP addresses of individual servers, coupling them directly.
- Traffic cannot be distributed dynamically, causing some servers to sit idle while others bottleneck.
- System administrators cannot run seamless rolling updates or server migrations.
- Direct exposure of backend application servers to the internet introduces considerable security vulnerabilities.

Building a custom proxy gateway teaches developers how traffic is multiplexed, encrypted, and distributed across enterprise networks.

## 5. Proposed Solution

The proposed software sits as a gateway on host ports `80` and `443`:
1. It listens for incoming HTTP/TCP connections.
2. It parses incoming request details (Method, Path, Host, Headers).
3. It evaluates path-routing rules (e.g., `/api` maps to API cluster; `/static` maps to Static server).
4. It executes a load balancing algorithm to pick an active, healthy backend server.
5. It rewrites headers (injecting `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Real-IP`).
6. It pipes the client request stream to the chosen backend server's socket.
7. It pipes the backend server response back to the client.
8. It continuously pings backends on a background schedule (Active Health Check) to remove dead servers from the routing rotation.

## 6. Project Goal

To build a robust, RFC-compliant reverse proxy and HTTP load balancer capable of managing traffic for a cluster of local microservices, automatically bypassing dead nodes, and providing detailed routing logs in a real-time CLI or web-based control panel.

## 7. Core Workflow

```text
Client                                  Reverse Proxy                              Backend Servers
  │                                           │                                           │
  ├────────1. HTTP GET /api/users────────────>│                                           │
  │                                           │                                           │
  │                                     Select Backend:                                   │
  │                                  Matches path "/api"                                  │
  │                                 Checks Health Matrix                                  │
  │                                Runs Load-Balancing Algo                               │
  │                                           │                                           │
  │                                   Rewrite Headers:                                    │
  │                                X-Forwarded-For: Client IP                             │
  │                                           │                                           │
  │                                           ├─────────2. Forward Request (Stream)──────>│ (Backend-B chosen)
  │                                           │                                           │
  │                                           │<────────3. Stream Response (Status 200)───┤
  │                                           │                                           │
  │                                     Update metrics                                    │
  │<───────4. Forward Response (Stream)───────┤                                           │
  │                                           │                                           │
  │                                           │ ──Active Background Health Scan (Ping)─> │
  │                                           │ <──────Healthy 200 OK (Keep in pool)───── │
```

## 8. Functional Requirements

### Gateway & Reverse Proxy
- **TCP Connection Handling**: Keep connection streams open, handle concurrent client requests, and stream payload bytes without buffering entire files in memory.
- **HTTP Parsing**: Read and validate HTTP request lines, headers, and chunks.
- **Header Rewriting**: Inject essential proxy headers (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `Via`).
- **Dynamic Routing**: Route requests based on hostname (Virtual Hosts) or request path prefixes (e.g., `/api/*` -> API cluster, `/*` -> Static server).

### Load Balancing
- **Algorithms**: Implement at least three load-balancing strategies:
  - *Round Robin*: Sequential rotation across the pool.
  - *Random*: Randomized distribution.
  - *Least Connections*: Route to the backend server with the fewest active TCP streams.
- **Session Persistence (Sticky Sessions)**: Route requests from the same client (identified via a cookie or IP hash) to the same backend server to maintain session states.

### Health Checker & Orchestration
- **Active Health Check**: Periodically send HTTP pings to a specified health endpoint on each backend (e.g., every 5 seconds).
- **Passive Health Check**: Mark a backend as degraded if it returns an unhandled server error (503/504) or timeouts during actual client routing.
- **Circuit Breaker / Failover**: Dynamically remove failed nodes from the active rotation pool and gracefully re-introduce them once health pings succeed consistently.

### Dashboard & Metrics
- **Dynamic Logging**: Live-stream proxy access logs showing: Client IP, Request URL, Target Backend, HTTP Status, and Execution Time.
- **Cluster Dashboard**: Visual table displaying backend servers, current status (Active, Dead, Degraded), active connections count, and memory/CPU loads.

## 9. Non-Functional Requirements

- **Zero-Buffering Streaming**: Stream requests and responses using chunked transfer encoding, ensuring the proxy can handle giant file uploads/downloads with constant memory overhead.
- **Latency Overhead**: Proxy routing overhead must be under 3 milliseconds per request under normal loads.
- **Concurrency & Scaling**: Efficiently multiplex connections using asynchronous event-loop models (epoll/kqueue) or light threads to handle 1000+ concurrent connections.
- **Graceful Shutdown**: Drain existing active connections before shutting down the proxy daemon.

## 10. Main Entities / Data Model

Because a load balancer is an in-memory traffic cop, the data structures must be optimized for fast, lock-free concurrent lookups.

### Backend Server Node
- **ID**: UUID or short unique key.
- **Address**: URL or socket string (e.g., `http://10.0.0.12:8080`).
- **PathPrefix**: Target mapping path (e.g., `/api`).
- **IsHealthy**: Boolean.
- **ActiveConnections**: Thread-safe atomic Counter.
- **Weight**: Integer (used for weighted routing algorithms).
- **ConsecutiveFailures**: Integer.

### Routing Rule Map
- **Hostname**: Target domain matching rules.
- **PathMap**: Ordered mapping of string paths to Backend Server Nodes.

## 11. System Components

- **Reverse Proxy Gateway**: Core listener engine binding to port 80/443. Parses stream headers, executes the routing tree, tracks connection counts, and pipes sockets.
- **Health Daemon**: Independent timer-driven worker that schedules and evaluates active endpoint pings to update Node status states.
- **Metrics Ingestion Broker**: Logs query events, tracks backend latency averages, and feeds live statistics.
- **Administration API & Dashboard**: Local web dashboard displaying telemetry and allowing temporary overrides (like manually taking a server offline for maintenance).

## 12. Important Technical Challenges

### Streaming Connections without Memory Bloat
- **Challenge**: If a client uploads a 1GB file, and the reverse proxy buffers the entire payload in RAM before forwarding it to the backend, the server will crash due to Out-Of-Memory (OOM) errors.
- **Concepts**: Backpressure, stream piping, chunked transfer encoding, and non-blocking I/O buffers.
- **Research**: Read about Node.js streams backpressure or Go's `io.Copy` buffer limits.

### SSL/TLS Termination
- **Challenge**: To handle HTTPS requests safely, the proxy must terminate the TLS connection at the edge, decrypt the packet, parse the HTTP headers, and optionally forward plaintext HTTP internally to backend servers.
- **Concepts**: Public-key cryptography, certificate handshakes, and SNI (Server Name Indication) routing.
- **Research**: Setting up self-signed certificates or Let's Encrypt certificates within a custom programmatic server.

### Dynamic DNS Resolution
- **Challenge**: If backend servers are identified by domain names instead of static IP addresses, the proxy must resolve these hostnames without locking up request threads.
- **Research**: Asynchronous DNS lookup libraries, caching DNS TTLs locally in-memory.

## 13. Suggested Technology Areas

- **Backend Gateway**: Go (highly recommended due to `net/http/httputil.ReverseProxy` or custom TCP parsing), Rust (hyper/tokio), C#, or Node.js.
- **Web Interface**: Simple HTML5 + vanilla JavaScript + CSS, pulling real-time metrics over Server-Sent Events (SSE).

## 14. Skills and Knowledge Gained

### Networking
- Comprehensive grasp of HTTP specification (headers, status codes, chunked encoding).
- Layer 4 (TCP/UDP) vs. Layer 7 (Application) load balancing.
- SSL/TLS cryptographic handshakes and SNI routing.

### Systems Design & Concurrency
- Writing thread-safe concurrent programs utilizing Atomic Counters, Mutexes, or channels.
- Building resilient systems using Circuit Breakers and Active/Passive health check states.
- Managing stream backpressure across slow clients and fast backends.

## 15. Recommended Development Phases

1. **Phase 1 - Simple TCP Redirect**: Build a raw TCP socket server that listens on port `8080` and redirects all raw data streams directly to another local running web server on port `3000`.
2. **Phase 2 - Basic HTTP Parser & Proxy**: Upgrade the TCP server to read HTTP stream headers. Parse the target host, modify headers (injecting `X-Forwarded-For`), forward it to port `3000`, read the response, and stream it back.
3. **Phase 3 - Multi-Node Round Robin**: Implement a routing configuration loading a list of 3 backend servers. Distribute incoming client requests sequentially (Round-Robin). Test this with multiple active local servers.
4. **Phase 4 - Active Health Checker**: Create a background daemon running a timer that pings backends on `/health`. Maintain a thread-safe health map. Exclude failed nodes from the Round-Robin rotation.
5. **Phase 5 - Advanced Load Balancing & Sticky Sessions**: Add Least Connections selection and Sticky Sessions (using client IP hashing).
6. **Phase 6 - Live Dashboard UI**: Expose port `9000` as an administration panel showing connected nodes, latency statistics, and current healthy nodes list.

## 16. Testing Requirements

- **Unit Tests**: Test the load-balancing selection algorithms under various edge cases (e.g., selecting a node when all but one is offline; weighted distribution accuracy).
- **Integration Tests**: Spin up 3 mock express/python servers that return their host port inside their response. Use `curl` to make 15 sequential requests to the proxy and verify that requests are distributed exactly as expected.
- **Load Testing**: Use load testing engines like `autocannon` or `wrk` on the proxy. Verify that memory consumption remains flat and flat-line latency overhead is minimal (under 5ms).

## 17. Security Considerations

- **DDoS/Rate Limiting**: Incorporate basic IP-bucket rate limiting at the proxy level to drop excessive requests before forwarding them to resource-intensive backends.
- **Header Sanitization**: Strip dangerous client-supplied headers (like pre-existing `X-Forwarded-For` strings) to prevent IP spoofing unless configured to append them.
- **Timeout Management**: Configure aggressive timeouts (e.g., 5-second read timeouts, 10-second idle timeouts) to prevent Slowloris attacks (where clients open thousands of slow, lingering connections).

## 18. Possible Extensions

- **Custom Cache Middleware**: Cache safe static responses (`GET /static/*`) at the proxy level to bypass backend servers completely, respecting HTTP cache headers (`Cache-Control`, `ETag`).
- **Dynamic Cluster Scaling via Webhooks**: Expose an API endpoint that allows container platforms (like Docker or Kubernetes) to dynamically register/deregister backend nodes as containers spin up/down.
- **gRPC Routing Support**: Support load balancing HTTP/2 and gRPC streams.

## 19. Learning Questions

- Why is it important to stream data between client and backend instead of reading it fully into memory?
- What is the difference between active health checks and passive health checks, and when should you use each?
- How does the "Sticky Session" IP-hashing algorithm degrade when all users are routed through a single NAT gateway (such as a corporate Wi-Fi router)?
- What is SNI (Server Name Indication) and why is it crucial for terminating multiple distinct SSL domains on a single reverse proxy IP?

## 20. Completion Criteria

- [ ] Reverse proxy successfully launches, binds to a port, and loads a configuration file of multiple backend servers.
- [ ] Directing client traffic through the proxy correctly routes to healthy backend nodes.
- [ ] Headers are correctly rewritten and received on the backend containing the client's actual IP address.
- [ ] When a backend server is killed, the active health checker detects it, removes it from rotation within 10 seconds, and the proxy continues operating without returning 502 errors to clients.
- [ ] Least Connections and Round Robin algorithms distribute traffic in predictable patterns.
- [ ] A real-time visual web or CLI dashboard displays the dynamic health status and current connection distribution of the cluster.