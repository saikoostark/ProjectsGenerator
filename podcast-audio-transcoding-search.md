## 1. Project Title

Podcast Audio Transcription and Search Platform

## 2. Difficulty

Mid-Level

### Rationale
This project combines media file handling, asynchronous background transcription workers, full-text search indexing, and a full-stack web interface. The developer must manage long-running audio processing pipelines, integrate speech-to-text engines, store timestamped transcripts, and build a full-text search engine that allows users to jump directly to exact spoken moments in audio episodes.

## 3. Project Overview

The Podcast Audio Transcription and Search Platform is a full-stack web application for discovering, listening to, and searching inside podcast episodes. When a podcaster uploads an audio file, a background worker queue ingests the file, runs speech-to-text (STT) transcription, generates timestamped word tokens, and indexes the text in a full-text search engine. Users can search for any keyword across thousands of podcast episodes and click search results to play the audio starting at the exact second the word was spoken.

## 4. Problem Statement

Audio content is traditionally opaque. Unlike text articles or PDF documents, podcasts cannot be easily searched, skimmed, or indexed by standard search engines.
- Listeners must scrub blindly through hours of audio to find specific topics discussed in an episode.
- Content creators lack accessible tools to generate searchable transcripts automatically.
- Building an audio search engine requires coordinating file storage, speech-to-text pipelines, structured database indexing, and interactive audio playback.

## 5. Proposed Solution

The proposed architecture consists of:
1. **Upload & Ingestion API**: Accepts audio uploads and stores them in object storage or local disk.
2. **Transcription Worker Pipeline**: Uses an STT engine (e.g., Whisper AI or Vosk) to convert audio into timestamped text tokens (words and sentences).
3. **Full-Text Search Index**: Stores transcripts in a relational database with full-text search capabilities (e.g., PostgreSQL FTS or SQLite FTS5).
4. **Interactive Web Player**: A frontend player that highlights words in real time as the audio plays and jumps to exact timestamps when search results are clicked.

## 6. Project Goal

To build a full-stack podcast platform where users can upload audio episodes, generate searchable timestamped transcripts via background workers, search for keywords across episodes, and play audio starting at exact timestamp matches.

## 7. Core Workflow

```text
Web Frontend                           Backend API                      Transcription Worker
     │                                      │                                       │
     ├──1. Upload Podcast Audio ───────────>│                                       │
     │                                      │──2. Enqueue Transcription Job────────>│
     │                                      │                                       │
     │<─────3. Return Upload Success────────┤                                       │
     │                                      │                                       │
     │                                      │              4. Run STT (Whisper)     │
     │                                      │              Extract Timestamps       │
     │                                      │                                       │
     │                                      │<────────5. Save Transcript (DB FTS)───┤
     │                                      │                                       │
     │──6. Search Keyword ("database")─────>│                                       │
     │<─────7. Return Timestamp Results─────┤                                       │
     │                                      │                                       │
     │──8. Click Result (Jump to 04:12)────>│                                       │
```

## 8. Functional Requirements

### Audio Ingestion & Transcription
- **File Upload**: Accept audio formats (`.mp3`, `.wav`, `.m4a`).
- **Transcription Pipeline**: Process audio through a speech-to-text model to produce structured JSON containing words, start times, and end times.
- **Async Processing**: Run transcription in background workers to prevent blocking HTTP requests.

### Search Engine
- **Full-Text Indexing**: Index transcripts using database full-text search (FTS) capabilities.
- **Snippet Generation**: Return search results with contextual snippets highlighting matching keywords and episode timestamps.

### Frontend Web App
- **Episode Library**: Browse and play podcast episodes.
- **Interactive Transcript Viewer**: Display live scrolling text matching current audio playback time.
- **Search Bar**: Query transcripts and jump to timestamps instantly on click.

## 9. Non-Functional Requirements

### Performance
- **Search Latency**: Keyword search queries must execute in under 100 milliseconds.

## 10. Main Entities / Data Model

### Episode
- **ID**: UUID.
- **Title**: Text.
- **AudioUrl**: Text.
- **DurationSeconds**: Integer.

### TranscriptSegment
- **ID**: UUID.
- **EpisodeID**: UUID (Foreign Key).
- **StartTime**: Real (seconds).
- **EndTime**: Real (seconds).
- **TextContent**: Text (indexed for Full-Text Search).

## 11. System Components

- **Web Frontend**: React/Vue audio player, search interface, and transcript viewer.
- **API Server**: Express/FastAPI backend handling requests and search queries.
- **Transcription Worker**: Background process executing speech-to-text tasks.
- **Database (FTS)**: PostgreSQL or SQLite with FTS5 enabled.

## 12. Important Technical Challenges

### Aligning Timestamps with Audio Playback
- **Challenge**: The frontend audio player must track `currentTime` and dynamically highlight the active transcript segment in real time without lagging the UI.
- **Concepts**: Audio element event listeners (`timeupdate`), binary search or interval indexing for active segment lookup.

### Managing Long-Running Transcription Jobs
- **Challenge**: Speech-to-text transcription of a 1-hour podcast can take several minutes. HTTP connections would timeout if processed synchronously.
- **Concepts**: Asynchronous worker queues, status polling or WebSocket notifications.

## 13. Suggested Technology Areas

- **Backend**: Node.js or Python (FastAPI).
- **STT Engine**: OpenAI Whisper API, Local Whisper (`whisper.cpp`), or Vosk.
- **Database**: SQLite FTS5 or PostgreSQL.
- **Frontend**: React, Tailwind CSS.

## 14. Skills and Knowledge Gained

### Full-Stack Engineering
- Integrating AI/Speech-to-Text pipelines into web applications.
- Implementing full-text search indexing and query retrieval.
- Building synchronized multimedia UI components (audio time-tracking and text highlighting).

## 15. Recommended Development Phases

1. **Phase 1 - Audio Ingestion & STT**: Write a script that takes an audio file, sends it to a speech-to-text engine, and outputs a JSON file with timestamped words.
2. **Phase 2 - Database & Full-Text Search**: Set up SQLite FTS5 or PostgreSQL FTS, store transcripts, and build a search query endpoint.
3. **Phase 3 - Backend API**: Build episode upload endpoints and tie transcription workers into a background queue.
4. **Phase 4 - Frontend Web App**: Build the episode list, search bar, audio player, and interactive scrolling transcript view.

## 16. Testing Requirements

* **Unit Tests**: Test search query parsing and timestamp matching functions.
* **Integration Tests**: Upload an audio file, verify the background worker processes it, and confirm search queries return correct episode timestamps.

## 17. Security Considerations

* **File Validation**: Validate audio file headers and extensions to prevent arbitrary file upload vulnerabilities.
* **Storage Limits**: Implement file size limits on audio uploads to prevent storage exhaustion attacks.

## 18. Possible Extensions

* **Speaker Diarization**: Detect and label different speakers in the podcast (e.g., "Host", "Guest").
* **AI Episode Summary**: Use an LLM to generate concise show notes and bullet-point summaries from the transcript.

## 19. Learning Questions

* How does Full-Text Search (FTS) indexing differ from standard relational database matching?
* Why is asynchronous background processing essential for speech-to-text transcription workflows?
* How can the frontend efficiently identify the active transcript segment during audio playback without searching the entire array on every `timeupdate` event?

## 20. Completion Criteria

* [ ] Users can upload podcast audio files through the web interface.
* [ ] Background workers successfully transcribe audio and store timestamped segments in the database.
* [ ] Users can search for keywords and receive relevant timestamped search results.
* [ ] Clicking a search result starts audio playback at the exact timestamp.
* [ ] The interactive transcript viewer highlights active text in sync with the audio player.