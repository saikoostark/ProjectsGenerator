## 1. Project Title

On-the-Fly Audio Transcoding Streamer

## 2. Difficulty

Junior+

### Rationale
This project introduces developers to processing large multimedia streams in real time. Rather than storing pre-rendered versions of audio files for every possible bit rate or audio format, the application spawns and wraps system command-line processes (specifically FFmpeg) asynchronously, piping incoming audio streams directly through FFmpeg and streaming the output bytes live to HTTP clients. It covers stream piping, process orchestration, backpressure, and local directory scanning.

## 3. Project Overview

The On-the-Fly Audio Transcoding Streamer is a smart local media streaming server. It acts as a personal music server, indexing high-fidelity, uncompressed audio files (e.g., `.wav`, `.flac`, `.alac`) from a specified local directory. When a user requests a file through the browser client, the streaming server inspects the client's network constraints or format preferences, launches a background FFmpeg child process on-the-fly to compress/transcode the audio (e.g., to `.mp3`, `.ogg`, or `.aac` at specific bitrates like `128kbps` or `320kbps`), and streams the compressed output bytes directly over HTTP with support for range requests (seeking).

## 4. Problem Statement

High-fidelity audio formats (like FLAC) sound incredible but consume massive file storage space and significant network bandwidth. If a user tries to stream a raw 100MB FLAC track over a spotty mobile network connection:
- The playback will stutter constantly due to buffer underrun.
- The web browser may not natively support the codec (e.g., older browsers cannot play FLAC natively).
- The user's mobile data plan will be quickly depleted.

Pre-converting thousands of audio files into multiple formats and bitrates (like 128k, 192k, 256k, 320k) on disk solves bandwidth issues but consumes huge quantities of local disk space and is highly redundant. An optimal solution is an on-demand, just-in-time media processor that translates audio formats on-the-fly inside RAM streams, preserving disk space while adapting to network capacities.

## 5. Proposed Solution

The proposed software serves a web client and coordinates system audio files:
1. It scans a designated local music folder, extracting ID3/FLAC metadata to build a local index database (using SQLite).
2. It hosts a Web interface listing albums, tracks, and format configuration controls.
3. When the user plays a track, the frontend sends an HTTP GET request containing query parameters for the target format and quality (e.g., `/api/stream/track_12.mp3?bitrate=128k`).
4. The backend opens a read stream for the source file and spawns a headless `ffmpeg` child process configured with custom parameters.
5. It pipes the source read-stream directly into `ffmpeg`'s standard input (`stdin`).
6. It hooks `ffmpeg`'s standard output (`stdout`) to the client's active HTTP response stream.
7. It monitors stream lifecycle events to terminate the child process immediately if the user pauses, stops, or disconnects.

## 6. Project Goal

To build a lightweight media streaming server that can parse a directory of high-fidelity music files, transcode tracks dynamically based on client selection, support seamless playback/seeking, and handle connection drops gracefully without leaving orphaned background processes.

## 7. Core Workflow

```text
Web Browser Client                     Media Server Daemon                      FFmpeg CLI Process
        │                                       │                                       │
        ├────────1. Request stream URL─────────>│                                       │
        │   (e.g., /stream/track.mp3?br=128k)   │                                       │
        │                                       │                                       │
        │                               Verify File                             │
        │                               Spawn FFmpeg Process ──────────────────>│
        │                                       │                                       │
        │                               Open Source File                        │
        │                               Pipe File Stream ──────────────────────>│ (stdin)
        │                                       │                                       │
        │                                       │<──Pipe Encoded Audio Stream (stdout)──┤
        │                                       │                                       │
        │<───────2. Stream HTTP response ───────┤                                       │
        │                                       │                                       │
        │                                       │                                       │
        │               [IF CLIENT DISCONNECTS / PRESSES PAUSE]                         │
        │──x───3. Socket Closes────────────────>│                                       │
        │                                       │                                       │
        │                                 Kill Child ──────────────────────────>│ [Terminated]
```

## 8. Functional Requirements

### Media Library Ingestion
- **Local Scanner**: Recursively scan a local system directory for common audio files (`.wav`, `.flac`, `.mp3`, `.m4a`).
- **Metadata Parser**: Extract track metadata (Title, Artist, Album, Duration, Track Number) and embedded album artwork.
- **Index Store**: Persist track paths and parsed metadata in an embedded database for quick listing.

### Streaming & Transcoding Engine
- **Child Process Orchestration**: Programmatically spawn the native `ffmpeg` binary using system execution tools.
- **Input/Output Piping**: Stream the raw source file bytes into the child's `stdin` and pipe the process's `stdout` stream into the outgoing HTTP response.
- **Dynamic Parameter Mapping**: Dynamically translate incoming HTTP query parameters (e.g., `?codec=opus&bitrate=96k`) into specific CLI arguments (e.g., `-f ogg -c:a libopus -b:a 96k`).
- **Process Lifecycle Watchdog**: Monitor client network connection closes and force-kill the active child process to reclaim CPU resources.

### Web Audio Player Client
- **Music Browser**: Render a clean catalog interface with albums and tracks search support.
- **Transcode Settings Panel**: Let users toggle stream bitrates (Low - 96kbps, Medium - 192kbps, High - 320kbps, Lossless FLAC) on-the-fly and observe immediate changes.
- **Native Audio Integration**: Integrate with the HTML5 `<audio>` tag to stream playback.

## 9. Non-Functional Requirements

- **Orphan Process Prevention**: Ensure absolutely zero orphaned background `ffmpeg` processes remain running, even during unexpected server crashes or client force-closes.
- **Zero-Latency Startup**: Playback must start playing within 1 second of requesting a track (FFmpeg must start encoding and flushing buffers immediately).
- **Constant Memory Footprint**: No audio data should be fully loaded in RAM. Stream pipes must govern memory usage, keeping memory consumption under 80MB.
- **Seek Support**: Support HTTP Range requests (`multipart/byteranges`) so the frontend audio timeline scrubber can skip back and forth inside transcoded audio.

## 10. Main Entities / Data Model

### Track Metadata (SQLite Schema)
- **ID**: Text (UUID or SHA-256 hash of file path).
- **FilePath**: Text (Absolute path to the local file, e.g., `/Users/music/artist/album/track.flac`).
- **Title**: Text.
- **Artist**: Text.
- **Album**: Text.
- **Duration**: Real (duration in seconds).
- **Format**: Text (original extension, e.g., `flac`).
- **Bitrate**: Integer (source bitrate in kbps).

### Active Stream Session (In-Memory)
- **SessionID**: String.
- **TrackID**: String.
- **ProcessID**: Integer (system PID of running FFmpeg child process).
- **StartedAt**: DateTime.

## 11. System Components

- **Directory Scanner Service**: Periodically crawls configured folders, parsing tags, and managing DB entries.
- **Ingestion Database**: Embeds SQLite to maintain catalog index.
- **Transcoding controller / Router**: Receives playback streams requests, provisions/configures child processes, handles backpressure, and routes output bytes to sockets.
- **Single Page Web Application**: Dynamic audio library catalog built with HTML/CSS and Javascript, controlling bitrates and playing music.

## 12. Important Technical Challenges

### Handling Backpressure in Streams
- **Challenge**: The FFmpeg process can transcode audio faster than the web client can play/download it, or a slow connection may not fetch bytes fast enough. If the server keeps pumping data without waiting, the internal buffer will bloat.
- **Concepts**: TCP Backpressure, Stream buffering, pause/resume stream propagation.
- **Research**: Read how native stream pipes propagate "drain" events when writing to write-streams that are saturated.

### Orphaned Process Leakage
- **Challenge**: In Node.js or Python, spawning a child process runs a separate operating system process. If the client disconnects or the user skips tracks frequently, and the backend code fails to intercept these socket events, dozens of ghost FFmpeg processes will consume 100% of the host CPU.
- **Concepts**: Process signals (`SIGTERM`, `SIGKILL`), exit code monitoring, process tree hunting.
- **Research**: Read about event listeners for network socket termination and how to safely run `child.kill()` or `process.kill()`.

### Supporting Audio Seeking
- **Challenge**: Enabling the user to click ahead in the seek bar for a file that is currently being dynamically generated in real-time is difficult, because the target seek-time does not map directly to a byte offset on disk.
- **Concepts**: HTTP range headers (`Content-Range`), passing timestamp offset arguments to media converters (e.g., `-ss <time>` parameter in FFmpeg).

## 13. Suggested Technology Areas

- **Backend Runtime**: Node.js (highly recommended due to excellent Stream API support), Python (FastAPI/Flask with subprocess streams), or Go.
- **Media Engine**: FFmpeg CLI binary installed on the host system path.
- **Metadata Parsing**: Libraries like `music-metadata` (Node.js) or `mutagen` (Python).
- **Frontend Client**: React, Vue, or Tailwind CSS + plain Javascript.

## 14. Skills and Knowledge Gained

### Backend & Core Systems
- Deep understanding of Child Process execution and System streams piping.
- Working with streams, chunked I/O buffers, and flow-control (backpressure).
- Designing fail-safe lifecycles for external system CLI dependencies.

### Networking & Web
- Understanding multimedia HTTP transfer protocols (Range headers, Audio-seeking mechanics).
- Parsing audio file metadata formats (ID3 tags, Vorbis Comments).

## 15. Recommended Development Phases

1. **Phase 1 - FFmpeg Command Lab**: Verify FFmpeg is installed on your OS. Run CLI commands to convert a local `.flac` file into a compressed `.mp3` directly using command-line prompts.
2. **Phase 2 - Basic Spawning & Piping**: Write a simple backend script. When a client hits `/stream`, programmatically spawn FFmpeg, feed a local high-fidelity track into its stdin, and pipe its stdout to the HTTP response stream.
3. **Phase 3 - Connection Lifecycle Watchdog**: Add event listeners that detect when the client closes the browser or cancels the streaming request. Test to ensure the spawned process is immediately killed.
4. **Phase 4 - File crawling & Metadata Indexing**: Implement the scanner logic. Crawl a music folder, extract ID3/codec tags, and store them inside an SQLite DB. Expose an API endpoint listing all index tracks.
5. **Phase 5 - Seeking (Timeline Scrubber)**: Implement seeking support by parsing `?start_seconds=45` query parameters and instructing FFmpeg to seek forward before transcoding using the `-ss` argument.
6. **Phase 6 - Frontend Catalog**: Build a UI displaying album folders, songs, and a player with a dynamic slider, custom bitrate select menu, and visual spectrums.

## 16. Testing Requirements

- **Unit Tests**: Mock track metadata parsing output. Verify that dynamic HTTP request queries safely and correctly compile into correct FFmpeg parameter array strings without argument injection vulnerabilities.
- **Integration Tests**: Simulate browser socket disconnections during an active stream. Verify programmatically that the corresponding FFmpeg PID is cleanly killed and cleared from the process table.
- **Resource Profiling**: Monitor host CPU and RAM usage while sequentially clicking "Skip Track" 30 times. Ensure the memory footprint remains flat and CPU cycles settle to 0% when music is paused.

## 17. Security Considerations

- **Command Injection Prevention**: NEVER pass raw user query parameters directly into the shell execution string. When spawning FFmpeg, pass arguments as a structured array rather than running inside shell interpolation (e.g., use `spawn('ffmpeg', ['-i', path, ...])` instead of `exec('ffmpeg -i ' + path)`).
- **Directory Traversal Defense**: Strictly validate that requested file IDs map only to absolute paths inside the configured music library directory. Do not allow users to stream random system files (like `/etc/passwd`).

## 18. Possible Extensions

- **Live Audio Visualizer Spectrum**: Compute Fast Fourier Transforms (FFT) on the audio buffer to stream real-time audio frequency data to build canvas-based visualizers.
- **Multi-Room Synchronization**: Create web classrooms/party channels where audio is synchronized across multiple client streams using WebSockets.
- **HL/DASH Segmented Streaming**: Transcode audio into segmented chunk lists (`.m3u8` playlists) for advanced adaptive streaming over web standard HLS.

## 19. Learning Questions

- Why is shell execution (`exec`) highly dangerous compared to direct execution (`spawn`) when integrating command-line binaries in backend services?
- How does backpressure prevent the server from running out of RAM when processing fast file conversions?
- What are HTTP Range requests, and how does the server tell the browser that a stream supports range-seeking?
- What is the difference between a container format (e.g., `.m4a`) and an audio codec (e.g., AAC)?

## 20. Completion Criteria

- [ ] The music directory crawler successfully runs, extracts metadata, and indexes files into the SQLite database.
- [ ] API successfully returns a JSON list of tracks with complete metadata (artist, title, album, format).
- [ ] Music playback begins in the browser player without throwing codec errors.
- [ ] Users can change quality presets (e.g., switching from 96kbps to 320kbps) and observe immediate changes.
- [ ] Clicking on the audio scrub bar to seek forward inside a high-fidelity track updates the playback position without stuttering.
- [ ] Force-closing the browser tab immediately kills the associated FFmpeg process on the host server. No ghost processes are left behind.