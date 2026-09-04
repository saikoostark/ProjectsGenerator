## 1. Project Title

Intelligent Log Aggregation Agent

## 2. Difficulty

Junior+

### Rationale
This project focuses on filesystem monitoring and stream processing. The challenge lies in creating an efficient agent that can "tail" log files in real-time, implement robust pattern matching (regex), and buffer output streams to minimize I/O overhead before flushing logs to a centralized server. It introduces concepts like file watchers, backpressure, and network serialization.

## 3. Project Overview

The Intelligent Log Aggregation Agent is a background service that monitors local log files, processes them on-the-fly, and transmits them to a centralized collector server. It performs real-time filtering, data anonymization (stripping sensitive info), and rate-limited batching to optimize network throughput. The agent is designed to be lightweight, running as a service on any machine, and includes a dashboard to configure which files to watch and which patterns to filter.

## 4. Problem Statement

In distributed systems, logs are scattered across dozens of nodes. Debugging requires manual SSH access to each machine, which is unsustainable.
- Without a central log aggregator, data is siloed and lost if a machine crashes.
- Direct streaming of all logs consumes massive bandwidth.
- Raw logs often contain sensitive data (passwords, PII) that must be masked before leaving the host machine.
- Manually processing files on many machines is error-prone.

Building an aggregation agent exposes developers to log lifecycle management, data sanitization, and streaming efficiency.

## 5. Proposed Solution

The agent is a cross-platform daemon:
1. **Configurable Watcher**: It reads a YAML config file defining file paths to tail and regex filters.
2. **Real-time Tailing**: Using native file system events (e.g., `inotify` or `fs.watch`), it efficiently detects new log entries.
3. **Processing Pipeline**: Each entry is passed through a pipeline: Filter (discard noise) -> Mask (anonymize) -> Buffer (batch).
4. **Resilient Transmission**: The buffered batch is sent via an encrypted HTTP/TCP stream to the collector. If the collector is down, the agent applies exponential backoff for retries to avoid data loss.

## 6. Project Goal

To build a lightweight, secure agent that reliably tails log files, filters/masks data based on user configuration, and streams logs to a central destination with high efficiency.

## 7. Core Workflow

```text
Log File (on disk)              Log Aggregation Agent               Central Collector
      │                                  │                                  │
      ├──1. New Line Detected───────────>│                                  │
      │                                  │                                  │
      │                               2. Process Pipeline                   │
      │                             (Filter/Mask/Batch)                     │
      │                                  │                                  │
      │                                  │──3. Send Buffered Batch─────────>│
      │                                  │                                  │
```

## 8. Functional Requirements

### Log Processing
- **File Watching**: Tail files continuously, handling file rotations (where files are renamed and recreated).
- **Filtering**: Allow inclusion/exclusion of lines based on regex patterns.
- **Data Anonymization**: Apply masks to sensitive data (e.g., replace email addresses or credit card numbers with `***`).
- **Batching**: Buffer log entries in memory and flush based on timer (e.g., every 5s) or size limit (e.g., 1MB).

### Transmission
- **Resilient Delivery**: If transmission fails, retry with exponential backoff.
- **Secure Transport**: Encrypt the log stream in transit.

## 9. Non-Functional Requirements

- **Low Overhead**: The agent should consume minimal CPU and RAM (e.g., <2% CPU, <50MB RAM).
- **Reliability**: No log entry should be permanently lost due to temporary network issues.

## 10. Main Entities / Data Model

### Config
- **WatchPath**: String.
- **RegexFilter**: String.
- **MaskingRules**: Map<String, String> (Regex to replacement).

### LogEntry
- **Timestamp**: DateTime.
- **Content**: String.
- **Source**: String.

## 11. System Components

- **File System Watcher**: Detects changes.
- **Pipeline Processor**: Filters and masks entries.
- **Streamer**: Manages network connectivity and batching.

## 12. Important Technical Challenges

### Handling File Rotations
- **Challenge**: Log rotation (e.g., `app.log` -> `app.log.1`) breaks simple file watches.
- **Concepts**: Inode watching, file descriptor monitoring.

### Backpressure
- **Challenge**: If logs are generated faster than the network can send them, the agent could consume all RAM.
- **Concepts**: Memory management, disk-backed buffers (optional).

## 13. Suggested Technology Areas

- **Backend**: Go or Rust (best suited for low-overhead background agents).

## 14. Skills and Knowledge Gained

### Systems Programming
- File I/O and OS signals.
- Serialization and streaming.

### DevOps
- Log management patterns.

## 15. Recommended Development Phases

1. **Phase 1 - File Tailing**: Build an agent that watches a file and prints new lines to `stdout`.
2. **Phase 2 - Processing Pipeline**: Add regex filtering and simple masking.
3. **Phase 3 - Batching & Transmission**: Implement the HTTP streamer with batching.
4. **Phase 4 - Rotation Handling**: Ensure the agent detects log rotation and re-opens files correctly.

## 16. Testing Requirements

- **Unit Tests**: Test the masking regex against various inputs.
- **Integration Tests**: Simulate log rotation by renaming/recreating files while the agent is running.

## 17. Security Considerations

- **Masking Effectiveness**: Ensure masking rules are robust and not easily bypassed.
- **Restricted Access**: The agent must only read files it is explicitly configured to watch.

## 18. Possible Extensions

- **Local Storage Buffer**: If the collector is down for long, buffer logs to disk.

## 19. Learning Questions

- How do you handle file rotation without losing data or creating duplicates?
- Why is it better to batch log transmissions rather than sending each line individually?

## 20. Completion Criteria

- [ ] Agent monitors configured files.
- [ ] Masking rules are applied successfully.
- [ ] Logs are batched and sent to a collector.
- [ ] Agent handles log rotation without restarting.
- [ ] Agent retries on network failure.