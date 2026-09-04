## 1. Project Title

Peer-to-Peer Browser CDN Offloader

## 2. Difficulty

Mid-Level

### Rationale
This project combines browser networking, WebRTC peer-to-peer data channels, and full-stack integration. The developer must build a system where browser clients visiting a web page automatically form a WebRTC mesh network to share static assets (such as images, video chunks, or JavaScript bundles), significantly reducing origin server bandwidth. It requires managing STUN/TURN signaling servers, chunk verification, and fallback mechanisms when peers are unavailable.

## 3. Project Overview

The Peer-to-Peer Browser CDN Offloader is a full-stack content delivery optimization platform. When high-traffic static assets (like large media files or software updates) are requested by multiple simultaneous users, traditional CDNs bear 100% of the bandwidth cost. This system injects a lightweight client-side SDK that connects visitors in a WebRTC P2P mesh. Peers request static asset chunks from each other rather than hitting the origin server, falling back to the origin server only if peers lack the requested chunks. A backend signaling server coordinates initial peer discovery and connection handshakes.

## 4. Problem Statement

Bandwidth costs scale linearly with web traffic, placing a heavy financial burden on websites hosting large static assets (e.g., video streaming, gaming assets, software downloads).
- Origin servers and commercial CDNs become bottlenecks during traffic spikes.
- Visitors sitting on the same local network or regional cluster download identical assets from remote servers redundantly.
- Leveraging idle client-side browser bandwidth through peer-to-peer sharing reduces server load and egress costs, but requires robust signaling, fallback logic, and cryptographic chunk verification.

## 5. Proposed Solution

The proposed architecture consists of:
1. **Signaling Server**: A WebSocket-based backend that orchestrates WebRTC offer/answer handshakes between browser clients requesting the same asset.
2. **Client-Side SDK**: A JavaScript library embedded in web pages that intercepts asset requests, checks if connected peers have the requested chunk via WebRTC DataChannels, and downloads it peer-to-peer.
3. **Origin Fallback Server**: A standard static asset server that serves chunks if no peers are available or if peer downloads timeout.
4. **Dashboard**: A full-stack analytics dashboard displaying origin bandwidth savings, active peer mesh size, and transfer speeds.

## 6. Project Goal

To build a full-stack P2P content delivery offloader where browser clients share static asset chunks via WebRTC, backed by a WebSocket signaling server and an analytics monitoring dashboard.

## 7. Core Workflow

```text
Browser Client A                      Signaling Server (WS)                  Browser Client B
     │                                         │                                    │
     ├──1. Request Asset Chunk ───────────────>│                                    │
     │                                         │──2. Match with Active Peer B──────>│
     │                                         │                                    │
     │<─────3. WebRTC SDP Offer / Answer───────┼───── (Establish DataChannel) ─────>│
     │                                         │                                    │
     │<===== 4. Direct P2P Chunk Transfer (WebRTC DataChannel) ====================>│
     │                                         │                                    │
     │                                         │             [If Peer Fails]        │
     │──────5. Fallback to Origin Server───────┼────────────────────────────────────>│
```

## 8. Functional Requirements

### Signaling & P2P Mesh
- **WebSocket Signaling**: Coordinate WebRTC SDP offers, answers, and ICE candidate exchanges between peers.
- **Mesh Formation**: Connect clients visiting the same resource URL into a decentralized P2P data cluster.
- **Chunk Verification**: Verify cryptographic hashes (SHA-256) of received chunks to prevent malicious data injection by untrusted peers.

### Fallback & Reliability
- **Origin Fallback**: If a peer connection times out or fails checksum verification, automatically fall back to fetching the chunk from the origin server.
- **Metrics Collection**: Report bandwidth savings and peer transfer statistics back to the analytics server.

## 9. Non-Functional Requirements

### Performance & Security
- **Transfer Speed**: P2P chunk transfer speeds should match local network capabilities.
- **Data Integrity**: Zero tolerance for corrupted chunks; failed checksums trigger immediate origin fallback and peer blacklisting.

## 10. Main Entities / Data Model

### AssetChunk
- **AssetID**: String.
- **ChunkIndex**: Integer.
- **Checksum**: String (SHA-256).

### PeerSession
- **PeerID**: UUID.
- **AssetID**: String.
- **ConnectedAt**: DateTime.

## 11. System Components

- **Signaling Server**: Node.js/Go WebSocket server managing peer matchmaking.
- **Origin Server**: Static file server serving asset chunks as fallback.
- **Client SDK**: JavaScript WebRTC data channel manager.
- **Dashboard**: React analytics portal displaying bandwidth savings.

## 12. Important Technical Challenges

### WebRTC NAT Traversal & Signaling
- **Challenge**: Browsers behind strict firewalls and NATs require STUN servers (and potentially TURN servers) to establish direct P2P connections.
- **Concepts**: ICE candidates, SDP negotiation, STUN server configuration.

### Chunk Splitting & Assembly
- **Challenge**: Large assets must be split into uniform binary chunks (e.g., 64KB), distributed across peers, and reassembled accurately in the browser.
- **Concepts**: ArrayBuffers, Blob manipulation, hash validation.

## 13. Suggested Technology Areas

- **Backend / Signaling**: Node.js (ws) or Go.
- **Frontend / SDK**: JavaScript (WebRTC APIs), React for dashboard.
- **Database**: SQLite or Redis for session tracking.

## 14. Skills and Knowledge Gained

### Browser Networking & P2P
- WebRTC peer-to-peer data channels and signaling architectures.
- NAT traversal, STUN/ICE protocols, and fallback strategies.
- Binary data handling in the browser (ArrayBuffer, Blob, crypto hashing).

## 15. Recommended Development Phases

1. **Phase 1 - Signaling Server**: Build a WebSocket server that pairs clients requesting the same room/asset.
2. **Phase 2 - WebRTC DataChannel**: Implement browser-to-browser WebRTC connection establishment and raw byte transfer.
3. **Phase 3 - Chunking & Checksums**: Implement file chunking, SHA-256 verification, and origin server fallback.
4. **Phase 4 - Dashboard & Analytics**: Build the frontend analytics dashboard displaying bandwidth savings.

## 16. Testing Requirements

* **Unit Tests**: Test chunk hashing and reassembly logic.
* **Integration Tests**: Simulate multiple browser clients using automated testing tools (Playwright) to verify P2P mesh transfer and origin fallback.

## 17. Security Considerations

* **Untrusted Peers**: Never trust data received from a peer without verifying its SHA-256 checksum against the manifest provided by the trusted origin server.

## 18. Possible Extensions

* **Incentivized P2P**: Implement a credit system where peers who upload more chunks get prioritized download speeds.
* **Encrypted Chunks**: Encrypt asset chunks at rest on the origin server so intermediate peers only pass ciphertext.

## 19. Learning Questions

* Why are STUN servers necessary for WebRTC connection establishment across different networks?
* How does cryptographic hash verification protect against malicious peers injecting corrupted data in a P2P mesh?
* What are the primary trade-offs between origin bandwidth savings and client battery/CPU consumption in browser P2P networks?

## 20. Completion Criteria

* [ ] Signaling server successfully matches peers requesting the same asset via WebSockets.
* [ ] Browser clients establish direct WebRTC DataChannels and exchange binary asset chunks.
* [ ] Received chunks are cryptographically verified via SHA-256 checksums.
* [ ] Failed peer transfers automatically fall back to the origin server.
* [ ] Analytics dashboard displays accurate origin bandwidth savings metrics.