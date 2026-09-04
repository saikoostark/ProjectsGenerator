## 1. Project Title

P2P Private File Sharing Client

## 2. Difficulty

Mid-Level

### Rationale
This project introduces decentralization, peer discovery, and raw socket manipulation. The developer must design a protocol for finding peers, negotiating data transfers, and implementing file chunking (slicing) for efficient and verifiable data transmission without a central server for file storage.

## 3. Project Overview

The P2P Private File Sharing Client is a terminal-based application that allows users to securely share files with other peers on a local network or via manual IP connection. It bypasses centralized cloud storage by allowing direct machine-to-machine transfer. Peers use a lightweight discovery protocol (UDP broadcast or direct handshake) to identify available resources, slice large files into checksum-verified chunks, and reconstruct them on the receiving end.

## 4. Problem Statement

Sharing large files between machines often relies on centralized cloud providers, which introduce latency, privacy concerns, and storage limits. While protocols like BitTorrent exist, they are often overkill for private, small-scale ad-hoc sharing.

Without a peer-to-peer (P2P) mechanism:
- Users rely on public file-sharing services that require uploading to the cloud first.
- Files cannot be transferred directly between devices on the same local network without traversing external bandwidth.
- Lack of verifiable integrity can lead to corrupted file transfers.

Building a P2P client forces developers to understand decentralized networking, handshake protocols, and error-tolerant file transmission.

## 5. Proposed Solution

The software acts as both a client and a server (a "servent").
1. **Discovery**: Peers find each other via UDP broadcasting on the local network.
2. **Indexing**: Each peer maintains a list of locally shared files and their metadata (name, size, SHA-256 hash).
3. **Chunking**: Files are broken down into fixed-size chunks.
4. **Transfer**: When a peer requests a file, the sender opens a direct TCP connection, transmits chunks, and includes checksums for every chunk to verify integrity.
5. **Reconstruction**: The receiver validates each chunk against the pre-calculated hash and writes them to disk, reconstructing the original file only if all checks pass.

## 6. Project Goal

To build a functional P2P CLI tool that enables two or more machines to discover each other, browse shared file lists, and successfully transfer files with data integrity verification.

## 7. Core Workflow

```text
Peer A (Sharer)                         Peer B (Receiver)
      │                                         │
      │──1. Broadcast "I am alive" (UDP)───────>│
      │                                         │
      │<──2. Request file list (TCP)────────────┤
      │──3. Return file list & hashes──────────>│
      │                                         │
      │<──4. Request file X (TCP)───────────────┤
      │                                         │
      │──5. Stream chunks + Chunk Hashes───────>│
      │                                         │
      │                                   [Verify Integrity]
      │                                   [Assemble File]
```

## 8. Functional Requirements

### Peer Discovery & Protocol
- **Discovery**: Implement a UDP-based broadcast protocol to announce presence on the network.
- **Handshake**: Establish a secure(ish) TCP connection for control/file-transfer operations.
- **File Indexing**: Maintain an in-memory index of shared local files and generate their SHA-256 hashes.

### File Transfer Engine
- **Chunking**: Divide large files into manageable chunks (e.g., 64KB).
- **Data Streaming**: Implement reliable TCP streaming of file chunks.
- **Integrity Verification**: Transmit a hash for each chunk; the receiver must verify before persisting the chunk to disk.
- **Resume Capability**: Support resuming interrupted file transfers based on already verified local chunks.

### CLI Interface
- **Command Set**: `share <file>`, `list`, `download <file-id>`, `status`.
- **Progress Tracking**: Show real-time transfer speed, progress bars, and percentage complete.

## 9. Non-Functional Requirements

- **Efficiency**: Support high-speed transfers limited only by network bandwidth, not CPU.
- **Integrity**: Files must NEVER be corrupt upon reconstruction.
- **Robustness**: If a peer disconnects midway, the receiver must be able to keep partial file chunks for a potential resume.

## 10. Main Entities / Data Model

### PeerNode
- **ID**: UUID.
- **IPAddress**: String.
- **Port**: Integer.
- **LastSeen**: DateTime.

### SharedFile
- **ID**: SHA-256 hash (or similar).
- **Name**: String.
- **Size**: Integer.
- **Hash**: String (total file hash).

### TransferSession
- **File**: SharedFile.
- **ChunksRemaining**: List of indexes.
- **Peer**: PeerNode.

## 11. System Components

- **Discovery Module (UDP)**: Broadcasting and listening for peer announcements.
- **TCP Control/Data Server**: Handles incoming file requests and streaming data.
- **File Index Manager**: Manages the local file list and hashes.
- **Transfer Coordinator**: Orchestrates chunk request/download/verify flow.

## 12. Important Technical Challenges

### Chunked Integrity
- **Challenge**: If a file is 1GB, a single flip of a bit during transfer makes the file unusable. 
- **Concepts**: Hashing algorithms (SHA-256), chunk-based validation, error correction.
- **Engineering Decision**: How to store chunk hashes during transfer? (e.g., a manifest file).

### NAT Traversal (Advanced/Optional)
- **Challenge**: Local network discovery works, but sharing over the internet is blocked by routers (NAT/Firewall).
- **Research**: STUN/TURN protocols for Hole Punching.

## 13. Suggested Technology Areas

- **Backend**: Go (excellent concurrency model), Rust, or Node.js (`dgram`, `net` modules).
- **CLI**: Standard terminal interface libraries (e.g., `cobra` in Go).

## 14. Skills and Knowledge Gained

### Networking
- UDP broadcast for peer discovery.
- TCP stream management for control/data.
- Understanding NAT/Firewalls.

### File Systems & Data
- File chunking and reconstruction.
- Hash-based integrity checking.

## 15. Recommended Development Phases

1. **Phase 1 - UDP Discovery**: Build two instances of the tool that can "find" each other on the local network via UDP broadcasting.
2. **Phase 2 - Basic TCP File Transfer**: Establish a TCP connection between two peers and successfully transfer a small file without chunking or hashes.
3. **Phase 3 - Chunking & Integrity**: Implement chunking and SHA-256 verification per chunk.
4. **Phase 4 - File Browser**: Add the `list` and `download` CLI commands.
5. **Phase 5 - Resume & Error Handling**: Handle peer disconnection and file transfer resumption.

## 16. Testing Requirements

- **Unit Tests**: Test hash verification function for corrupted chunks.
- **Integration Tests**: Simulate peer disconnection mid-transfer and verify file resumption functionality.

## 17. Security Considerations

- **Peer Spoofing**: Validate peer authenticity if necessary.
- **Resource Constraints**: Limit the maximum number of simultaneous transfers to prevent socket exhaustion.
- **Path Sanitization**: Ensure shared files are strictly within a designated "share" directory; prevent peers from downloading sensitive system files via path traversal.

## 18. Possible Extensions

- **Encryption**: Add TLS/AES encryption to the file transfer socket to make transfers private.
- **Multi-source Download**: Download chunks of the same file from multiple peers simultaneously.

## 19. Learning Questions

- Why is a TCP connection better than UDP for file transfer?
- What are the trade-offs between chunking a file into small pieces vs large pieces?
- How does the system ensure a received file matches the original if no central index is used?

## 20. Completion Criteria

- [ ] Two instances can discover each other.
- [ ] Users can list available files from the remote peer.
- [ ] Large files transfer correctly and hashes match.
- [ ] Interrupted transfers resume correctly.
- [ ] Path traversal attempts are blocked.