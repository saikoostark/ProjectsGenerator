## 1. Project Title

Multicast LAN Chat Application

## 2. Difficulty

Junior+

### Rationale
This project introduces UDP networking and the concept of multicast communication, which is fundamentally different from traditional client-server TCP models. It forces developers to understand how to bind to multicast groups, manage broadcast traffic, and design protocols for peer discovery and message delivery without a central intermediary or message broker.

## 3. Project Overview

The Multicast LAN Chat Application is a terminal-based peer-to-peer chat tool that operates over a local network. When launched, instances automatically discover each other by joining a predefined IP multicast group. Users can send messages that are broadcast to all other active instances in the same group. It demonstrates decentralization, real-time messaging, and basic network protocol design.

## 4. Problem Statement

Centralized chat applications (e.g., Slack, Discord) require an internet connection and a central server.
- In secure environments (e.g., offices, home networks), users may need a simple way to communicate that doesn't rely on external infrastructure or internet access.
- Traditional chat requires managing user accounts, authentication servers, and complex client-server infrastructure.

A P2P chat application simplifies this, allowing instant communication without configuration or central dependency.

## 5. Proposed Solution

The application uses UDP Multicast:
1. **Multicast Join**: When the application starts, it joins a specific multicast group address (e.g., `224.0.0.1`).
2. **Broadcasting**: When a user types a message, it is encapsulated in a packet and sent to the multicast group.
3. **Listening**: Every instance listens on the multicast group for incoming packets and displays them immediately in the terminal UI.

## 6. Project Goal

To build a zero-configuration, P2P chat tool where instances running on the same network automatically see and communicate with each other.

## 7. Core Workflow

```text
Instance A (User)               Multicast Network               Instance B (User)
      │                                │                                │
      ├────1. Join Multicast Group────>│                                │
      │                                │<────2. Join Multicast Group────┤
      │                                │                                │
      │────3. Send "Hello" (UDP)──────>│──────4. Deliver "Hello"───────>│
      │                                │                                │
```

## 8. Functional Requirements

### Network Protocol
- **Multicast Management**: Join and leave multicast groups.
- **Message Transmission**: Send and receive messages using UDP.

### UI/UX
- **Terminal Interface**: A simple CLI to type messages and view incoming ones.
- **Presence**: Display connected peers (optionally).

## 9. Non-Functional Requirements

- **Zero-Config**: No server IP or port needs to be configured.
- **Local Network Only**: Must work reliably on a single subnet.

## 10. Main Entities / Data Model

### Message
- **Sender**: String (username).
- **Text**: String.
- **Timestamp**: DateTime.

## 11. System Components

- **Multicast Listener**: Handles incoming UDP packets.
- **Message Sender**: Handles outgoing message broadcast.
- **UI Component**: Terminal interface.

## 12. Important Technical Challenges

### Network Subnets
- **Challenge**: Multicast packets are often blocked by routers or firewalls.
- **Concept**: Subnet restrictions, IGMP protocol.

### Message Ordering/Reliability
- **Challenge**: UDP is unreliable; messages may be dropped, duplicated, or arrive out of order.
- **Concept**: Handling UDP limitations in application logic.

## 13. Suggested Technology Areas

- **Backend**: Go or Node.js (UDP support).
- **CLI**: Standard terminal input/output.

## 14. Skills and Knowledge Gained

### Networking
- Understanding UDP Multicast.
- Peer discovery mechanisms.

### P2P Concepts
- Stateless message exchange.

## 15. Recommended Development Phases

1. **Phase 1 - UDP Echo**: Build a simple sender/receiver using UDP to ensure basic network communication works.
2. **Phase 2 - Multicast Implementation**: Configure the UDP sockets to join a multicast group.
3. **Phase 3 - Chat Interface**: Add terminal UI to input and display messages.

## 16. Testing Requirements

- **Integration**: Run two instances of the application on the same network to verify message exchange.

## 17. Security Considerations

- **Trust**: Assume all packets on the network are untrusted. Sanitize all incoming messages.

## 18. Possible Extensions

- **Username Personalization**: Allow setting nicknames.
- **Private Messaging**: Add a mechanism for directed messages within the multicast group (peer-to-peer over UDP).

## 19. Learning Questions

- How does UDP Multicast differ from Broadcast and Unicast?
- What happens if a message is lost in a P2P chat?

## 20. Completion Criteria

- [ ] Multiple instances discover each other automatically.
- [ ] Messages sent from one instance appear in all others.
- [ ] No central server is required.