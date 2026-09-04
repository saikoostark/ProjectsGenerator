## 1. Project Title

Secure Zero-Knowledge File Drop Portal

## 2. Difficulty

Junior+

### Rationale
This project combines client-side cryptography, secure full-stack workflows, and ephemeral storage. The developer must implement client-side encryption using the Web Crypto API (AES-GCM) before files are uploaded, store encrypted payloads in an ephemeral database, and provide a secure link-sharing portal with automatic "burn-after-reading" self-destruction semantics.

## 3. Project Overview

The Secure Zero-Knowledge File Drop Portal is a full-stack web application for sharing sensitive files securely. When a user uploads a file, the browser encrypts the file locally using AES-GCM before sending any data to the server. The encryption key is embedded in the URL fragment (hash parameter `#key`), ensuring the server never receives or stores the decryption key (Zero-Knowledge architecture). When the recipient opens the link, the encrypted payload is downloaded, decrypted entirely in their browser, and permanently deleted from the server upon first retrieval ("burn-after-reading").

## 4. Problem Statement

Sharing sensitive documents (API keys, confidential contracts, credentials) via standard email or cloud storage is insecure.
- Cloud storage providers have access to unencrypted files stored on their servers, exposing them to subpoenas or data breaches.
- Standard file links remain active indefinitely, creating long-term security liabilities if compromised.
- Without client-side encryption, the server acts as a trusted custodian of plaintext sensitive data.

Building a zero-knowledge file drop portal teaches developers client-side cryptographic principles, secure URL fragment handling, and ephemeral data lifecycle management.

## 5. Proposed Solution

The proposed architecture consists of:
1. **Client-Side Encryption**: Uses the Web Crypto API (`crypto.subtle`) to generate a random AES-GCM encryption key locally in the browser and encrypt the file before upload.
2. **Ephemeral Backend Storage**: The server stores only the encrypted ciphertext blob and metadata (expiration time, download count). The encryption key is placed in the URL hash (e.g., `https://drop.com/file/123#secret_key`), which browsers never send to servers in HTTP requests.
3. **Burn-After-Reading Engine**: Upon the first successful download request, the server permanently deletes the ciphertext record from the database and disk.

## 6. Project Goal

To build a full-stack secure file drop portal featuring client-side zero-knowledge encryption, URL-fragment key distribution, and automatic burn-after-reading self-destruction.

## 7. Core Workflow

```text
Uploader Browser                       Backend API                        Recipient Browser
     │                                      │                                     │
     ├──1. Select File & Generate AES Key──>│                                     │
     ├──2. Encrypt File Locally (AES-GCM)   │                                     │
     ├──3. POST /api/upload (Ciphertext)───>│                                     │
     │                                      │──4. Save Encrypted Blob to DB──────>│
     │<─────5. Return Drop ID Link──────────┤                                     │
     │   (URL: /drop/xyz#decryption_key)    │                                     │
     │                                      │                                     │
     │                                      │<──────6. GET /api/drop/xyz──────────┤
     │                                      │──7. Stream Ciphertext Blob─────────>│
     │                                      │──8. Permanently Delete Record ──────┤
     │                                      │                                     │
     │                                      │       9. Decrypt Locally with #key  │
     │                                      │       10. Save File to Disk         │
```

## 8. Functional Requirements

### Client-Side Cryptography
- **Key Generation**: Generate secure random AES-256-GCM keys locally using `crypto.subtle`.
- **Local Encryption**: Encrypt files in browser memory before uploading.
- **Local Decryption**: Decrypt downloaded ciphertexts in browser memory using the key from the URL hash.

### Ephemeral Storage & Expiration
- **Burn-After-Reading**: Automatically delete files immediately after their first successful download.
- **Time-Based Expiration**: Optionally expire and delete files after a set timeframe (e.g., 24 hours).

## 9. Non-Functional Requirements

### Security & Privacy
- **Zero-Knowledge Guarantee**: The server must never receive, log, or store the decryption key. If the server is compromised, stored files remain unreadable.

## 10. Main Entities / Data Model

### SecureDrop
- **ID**: UUID (primary key in URL path).
- **CiphertextPath**: String (path to encrypted blob on disk).
- **OriginalFilename**: String (encrypted or stored safely).
- **ExpiresAt**: DateTime.
- **BurnAfterRead**: Boolean.
- **IsDownloaded**: Boolean.

## 11. System Components

- **Frontend SPA**: React/Vue interface handling Web Crypto encryption/decryption and upload/download progress.
- **API Server**: Express/FastAPI backend handling encrypted blob storage and expiration worker cleanup.
- **Database**: SQLite or PostgreSQL tracking drop metadata and expiration timestamps.

## 12. Important Technical Challenges

### Handling URL Fragments (`#key`)
- **Challenge**: Developers must ensure encryption keys are placed in the URL fragment (hash) rather than query parameters, because query parameters are sent to servers in HTTP requests and logged in proxy access logs, whereas URL fragments stay strictly on the client.
- **Concepts**: URL structure (`#hash` vs `?query`), client-side routers.

### Memory Management for Large Files
- **Challenge**: Encrypting large files entirely in browser RAM can crash the tab with Out-Of-Memory errors.
- **Concepts**: Streaming encryption, chunked Web Crypto processing (if applicable).

## 13. Suggested Technology Areas

- **Backend**: Node.js or Python.
- **Frontend**: React, Tailwind CSS, Web Crypto API (`window.crypto.subtle`).
- **Database**: SQLite.

## 14. Skills and Knowledge Gained

### Cryptography & Security
- Client-side cryptography using the browser Web Crypto API.
- Zero-knowledge architectures and threat modeling.
- Ephemeral data lifecycles and secure cleanup routines.

## 15. Recommended Development Phases

1. **Phase 1 - Web Crypto Lab**: Write a standalone frontend script that encrypts and decrypts a test file in browser memory using AES-GCM.
2. **Phase 2 - Backend Encrypted Blob Storage**: Build an API endpoint that accepts binary blobs and stores them with UUID primary keys.
3. **Phase 3 - Burn-After-Reading Logic**: Implement download handlers that delete database records and disk files immediately upon first fetch.
4. **Phase 4 - Frontend Portal**: Build the React UI for uploading files, copying secure links with URL fragments, and downloading/decrypting shared files.

## 16. Testing Requirements

* **Unit Tests**: Test expiration worker logic and cryptographic helper functions.
* **Security Audits**: Verify via network inspection proxies (like Wireshark or browser DevTools) that the decryption key never appears in HTTP request headers or payloads.

## 17. Security Considerations

* **Key Exposure**: Educate users that losing the secure link (including the `#key` fragment) means the file is unrecoverable, as the server cannot reset passwords or decrypt data.

## 18. Possible Extensions

* **Password Protection**: Allow optional user-defined passwords combined with PBKDF2 to derive encryption keys.
* **Download Counter**: Allow configuring exact download limits (e.g., expire after exactly 3 downloads).

## 19. Learning Questions

* Why must the decryption key be stored in the URL fragment (`#key`) rather than a query parameter (`?key=...`)?
* What does "Zero-Knowledge" mean in the context of cloud storage architectures?
* How does AES-GCM provide both confidentiality and integrity verification?

## 20. Completion Criteria

* [ ] Users can upload files encrypted locally in the browser via Web Crypto.
* [ ] The server stores only ciphertext blobs and metadata without ever receiving the encryption key.
* [ ] Secure links include the decryption key in the URL fragment.
* [ ] Recipients can download and decrypt files directly in their browser.
* [ ] Files are permanently deleted from the server upon first download (burn-after-reading).