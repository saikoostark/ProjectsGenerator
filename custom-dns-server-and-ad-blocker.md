## 1. Project Title

Custom DNS Server and Ad Blocker

## 2. Difficulty

Mid-Level

### Rationale
This project requires low-level binary buffer manipulation to parse and construct network packets according to the RFC 1035 specification. Managing raw UDP sockets, building concurrent in-memory lookups (such as tries) with high performance, handling caching with TTL expiration, and implementing graceful degradation when upstream servers fail are standard mid-level backend challenges.

## 3. Project Overview

The Custom DNS Server and Ad Blocker is a lightweight, high-performance DNS resolver daemon. It intercepts DNS queries from local devices, matches requested domains against memory-optimized blocklists (compiled from public ad-blocking feeds), and returns `0.0.0.0` or `NXDOMAIN` for blacklisted domains. For legitimate queries, it forwards requests to upstream DNS servers (e.g., Cloudflare or Google), caches the responses locally while honoring TTL (Time-To-Live) constraints, and includes a real-time web dashboard for live monitoring and ad-filtering analytics.

## 4. Problem Statement

Modern web browsing is saturated with intrusive advertisements, tracking scripts, and malicious telemetry. While browser-based ad blockers exist, they:
- Do not protect non-browser applications (smart TVs, IoT devices, mobile apps, command-line tools).
- Consume considerable system resources by filtering content at the rendering stage.
- Require installation on every single client device individually.

Without a centralized network-level filter, local network traffic is cluttered, privacy is compromised, and bandwidth is wasted fetching ad resources before they can even be blocked. Existing hardware solutions like Pi-hole are great but act as a black box for learning. Building a custom DNS server exposes developers to raw networking, binary protocols, caching algorithms, and high-performance concurrent request handling.

## 5. Proposed Solution

The proposed software acts as a local DNS Server (listening on UDP/TCP port 53). All local network devices configure this server as their primary DNS resolver. 

When a query arrives:
1. The server decodes the binary DNS question packet.
2. It matches the domain name against an in-memory Trie-based blocklist.
3. If it matches, the server returns a synthetic IP (`0.0.0.0`) or a custom response immediately.
4. If it doesn't match, the server checks its local cache.
5. If cached (and TTL is valid), it returns the cached record.
6. If uncached, it forwards the packet to a secure upstream resolver, saves the result to the cache, and forwards it back to the client.
7. System statistics are streamed asynchronously to a web dashboard via Server-Sent Events (SSE) or WebSockets.

## 6. Project Goal

To build a fully functional, stable RFC 1035-compliant DNS server that can run continuously on a local network (e.g., on a Raspberry Pi or local container), successfully block ads across multiple devices, and provide a live dashboard demonstrating a significant reduction in ad network requests.

## 7. Core Workflow

```text
Client Device                       Custom DNS Server                      Upstream DNS
      │                                     │                                    │
      ├───────1. DNS Query (UDP 53)────────>│                                    │
      │                                     │                                    │
      │                               2. Parse Query                             │
      │                             Match against blocklist                      │
      │                                     │                                    │
      │                           [IF MATCH (AD DETECTED)]                       │
      │<──────3a. Return "0.0.0.0"──────────┤                                    │
      │                                     │                                    │
      │                           [IF NO MATCH / UNBLOCKED]                      │
      │                               Check Local Cache                          │
      │                                     │                                    │
      │                               [If Uncached]                              │
      │                                     ├─────3b. Forward DNS Query─────────>│
      │                                     │                                    │
      │                                     │<────3c. Binary DNS Response────────┤
      │                                     │                                    │
      │                                 Save to Cache                            │
      │                                 Stream stats                             │
      │<──────3d. Forward Response──────────┤                                    │
      │                                     │                                    │
```

## 8. Functional Requirements

### DNS Engine
- **Packet Parsing**: Correctly decode incoming RFC 1035 binary query packets (headers, question count, resource records, flags).
- **Packet Generation**: Correctly encode outgoing DNS responses with proper headers, flags (QR, AA, RCODE), questions, and answer sections.
- **Protocol Support**: Handle DNS over UDP primarily, with a fallback or simultaneous listener for DNS over TCP for large packets.
- **Upstream Forwarding**: Relay unblocked and uncached queries to upstream servers (e.g., `1.1.1.1` or `8.8.8.8`).

### Ad Blocking & List Management
- **Feeds Ingestion**: Download and parse standard hosts-style blocklists or domains-only blocklists on startup and scheduled cron intervals.
- **Efficient Lookup**: Maintain an in-memory lookup system that can match fully qualified domains or wildcard subdomains instantly under concurrent loads.
- **Query Interception**: Force-return `0.0.0.0` (A records) or `::` (AAAA records) for blocked domains with a short TTL (e.g., 5 seconds) to prevent clients from caching the block indefinitely.

### Cache Engine
- **In-Memory Cache**: Cache A and AAAA records mapped by domain.
- **TTL Expiration**: Automatically purge or refresh cache items whose TTL has expired.
- **Stale-While-Revalidate (Optional)**: Return stale cache records while asynchronously fetching fresh values to minimize client latency.

### Admin Dashboard & Metrics
- **Real-Time Stream**: Expose a REST API and a real-time stream (WebSockets or SSE) delivering queries logged, ads blocked, block ratio, and current active clients.
- **Blocklist Controls**: Toggle blocklists, add manual whitelist/blacklist domain overrides, and trigger manual database updates.
- **Query Log**: Display a tabular view of recent DNS queries including Timestamp, Client IP, Domain, Type (A, AAAA, MX), Status (Allowed, Cached, Blocked), and Latency.

## 9. Non-Functional Requirements

- **Latency**: Core DNS resolution overhead (excluding upstream network latency) must be under 5 milliseconds.
- **Memory Footprint**: Must keep memory consumption under 200MB, even when storing over 100,000 blocked domains.
- **Concurrency**: Must handle at least 500 concurrent UDP queries per second without dropping requests or blocking the main event loop.
- **Robustness**: If an upstream DNS server timeouts, the server must gracefully retry alternative upstream servers before returning a temporary server failure (`SERVFAIL`) response.

## 10. Main Entities / Data Model

Because DNS lookup speeds are highly critical, the data structures are designed to live primarily in memory.

### Blocklist Trie
- **Nodes**: Character or domain-label keys pointing to child nodes.
- **IsBlocked**: Boolean indicating if the path represents a blocked domain.
- *Example*: `doubleclick.net` represented as `net` -> `doubleclick` [IsBlocked: true].

### DNS Cache Record
- **Key**: String combination of `DomainName:QueryType` (e.g., `google.com:A`).
- **Value**: Raw parsed DNS answer payload.
- **ExpiresAt**: Absolute timestamp (calculated as `Now + TTL`).

### Query Metrics Entry (Ephemerally logged or persisted in SQLite)
- **ID**: UUID or Auto-incrementing Integer.
- **Timestamp**: DateTime.
- **Client IP**: String (IPv4 or IPv6 address).
- **Domain**: String.
- **QueryType**: Enum (A, AAAA, MX, TXT, CNAME, etc.).
- **ResponseStatus**: Enum (Blocked, Allowed, Cached).
- **DurationMs**: Integer.

## 11. System Components

- **DNS Resolver Daemon**: Low-level network socket handler (UDP/TCP port 53). Decodes packets, queries blocklists/cache, routes to upstream resolvers, and serializes responses.
- **Blocklist Processor**: Downloads raw text lists, strips comments/whitespaces, deduplicates entries, and compiles them into the live lookup data structure.
- **Background Metrics Worker**: Buffers transaction logs and flushes them to an embedded database (like SQLite) or streams them directly to active dashboard connections.
- **API and Web Dashboard Frontend**: Single-page dashboard displaying network health, block statistics (graphs), active logs, and settings.

## 12. Important Technical Challenges

### Binary Packet Manipulation
- **Challenge**: DNS relies heavily on bitwise operations and custom layouts (e.g., domain-name compression pointers where a domain label can point to an offset of another string).
- **Concepts**: Bitwise shifting, byte offsets, buffer slicing, and manual byte serialization.
- **Engineering Decisions**: The developer must decide whether to use standard built-in buffer libraries or construct custom bit-stream reader/writers.

### High-Performance Trie lookup vs. Hash Sets
- **Challenge**: Checking subdomains can be complex. If `analytics.doubleclick.net` is queried, and only `doubleclick.net` is in the blocklist, a simple hash set lookup for `analytics.doubleclick.net` fails.
- **Concepts**: Prefix Tries, suffix matching, memory layout of pointers.
- **Research**: Memory optimization strategies for tries in garbage-collected languages vs. system languages.

### Port 53 Restrictions
- **Challenge**: Operating systems restrict access to port numbers below 1024 to administrative accounts. Running a DNS server directly on host port 53 during development is tricky and conflicts with built-in system resolvers (like `systemd-resolved` on Linux).
- **Research**: Network namespace mapping, Docker port forwarding, dropping privileges to a non-root user after binding sockets, and using environment overrides (like `resolv.conf`) for manual testing.

## 13. Suggested Technology Areas

- **Backend Daemon**: Go, Rust, Node.js (using `dgram` and `net`), or C#.
- **Frontend Dashboard**: Tailwind CSS + React, Vue, or Vanilla JS.
- **Storage**: SQLite for local query logs, in-memory structures for real-time operations.
- **Real-Time Transport**: Server-Sent Events (SSE) or WebSockets.

## 14. Skills and Knowledge Gained

### Backend & Networking
- Binary parsing of RFC 1035 protocols.
- Low-level UDP and TCP socket programming.
- Client-Server connection life cycles.

### Algorithms & Memory
- Implementing and optimizing a Trie data structure.
- TTL-based eviction algorithms (Least-Recently-Used with timestamp expirations).

### Systems Design & DevOps
- Designing systems around restricted network ports.
- Setting up custom system daemons and Docker-based containerized deployments.

## 15. Recommended Development Phases

1. **Phase 1 - Binary Parsing Lab**: Write a standalone parser that can read a raw hex DNS packet, decode the header and question sections, and format the output into a readable JSON structure.
2. **Phase 2 - UDP Echo Server**: Implement a basic UDP server listening on an unprivileged port (e.g., 5300) that forwards all incoming packets to Google's public DNS (`8.8.8.8:53`) and echoes the responses back to the client.
3. **Phase 3 - Local Resolver and Cache**: Incorporate parsing into the UDP server. Extract domain names, search an in-memory map cache, handle cache inserts, and respect TTL values.
4. **Phase 4 - Trie Blocklist Engine**: Write the blocklist ingestion logic. Load a large hosts file, construct a Trie, and run subdomain matching tests. Route matched domains to a synthetic `0.0.0.0` response directly.
5. **Phase 5 - Dashboard & API API**: Build an embedded HTTP server alongside the DNS resolver. Push metric updates via SSE or WebSockets.
6. **Phase 6 - Frontend & UI**: Create the admin portal displaying charts, dynamic logs, and manual whitelist controls.
7. **Phase 7 - Deployment**: Containerize the app and test on a real local machine or virtual environment.

## 16. Testing Requirements

- **Unit Tests**: Test the DNS packet parser with mock buffers representing standard types (A, CNAME, MX) to ensure correct decoding. Test the Trie subdomain matching under edge cases (e.g., matching `sub.adserver.com` when only `adserver.com` is blocked, but allowing `notadserver.com`).
- **Integration Tests**: Set up a local test environment running the DNS server. Use toolings like `dig` or `nslookup` to verify behavior (e.g., `dig @localhost -p 5300 doubleclick.net` returns `0.0.0.0` while `dig @localhost -p 5300 google.com` returns real IPs).
- **Performance/Load Tests**: Write a benchmark script sending 10,000 UDP DNS queries. Verify memory leaks, CPU spikes, and drop rates.

## 17. Security Considerations

- **DNS Amplification Protection**: Mitigate open resolver vulnerability by ensuring the server only responds to queries originating from RFC 1918 private IP subnets (e.g., `192.168.0.0/16`, `10.0.0.0/8`, `127.0.0.1`).
- **Input Sanitization**: Validate DNS packet fields string lengths carefully. Prevent buffer overflow exploits (especially relevant in C/Rust) or out-of-bounds array reads.
- **Admin Dashboard Auth**: Secure the Web GUI with basic session auth to prevent unauthorized users on the same Wi-Fi from turning off the blocklist.

## 18. Possible Extensions

- **DNS over HTTPS (DoH) / DNS over TLS (DoT)**: Secure upstream communications by forwarding uncached queries over encrypted TLS/HTTPS connections instead of plaintext UDP/TCP.
- **Dynamic DHCP Lease Integration**: Read DHCP reservation logs to resolve local device hostnames in the query log (displaying `Dad's iPad` instead of just `192.168.1.42`).
- **Regex Blocking**: Allow advanced block rules containing custom regular expressions.

## 19. Learning Questions

- What is the difference between a recursive DNS resolver and an authoritative DNS server?
- Why is UDP preferred over TCP for typical DNS queries, and under what circumstances does a client switch to TCP?
- How does the DNS protocol compress strings in response records, and why is this compression essential?
- What are the architectural trade-offs of storing blocklists in a Redis database versus a native, in-memory Trie?

## 20. Completion Criteria

- [ ] The DNS server successfully starts, binds to port 53 (or a custom port with port-forwarding), and does not crash.
- [ ] Running `dig` on a blocked domain returns an A record with `0.0.0.0` and a TTL of less than 10 seconds.
- [ ] Running `dig` on a safe domain returns valid IP addresses identical to upstream responses, with corresponding TTL.
- [ ] Multiple devices can set the DNS server as their system resolver and browse the web without connectivity errors.
- [ ] The real-time web dashboard loads and displays a live-updating stream of queries showing blocked and allowed requests.
- [ ] Memory consumption stays stable under continuous query operations.