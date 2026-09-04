## 1. Project Title

Custom Git Visualizer and Object Parser

## 2. Difficulty

Junior+

### Rationale
This project introduces developers to low-level file format parsing and directory traversal. Since Git objects are compressed with zlib and have a specific header structure, the developer must read binary headers, parse commit DAG (Directed Acyclic Graph) trees, and construct internal directory trees from Git's proprietary format. This provides excellent exposure to file-system operations, compression, and tree-traversal algorithms without requiring advanced networking or complex system architectures.

## 3. Project Overview

The Custom Git Visualizer and Object Parser is a local desktop or web-based exploration tool that directly inspects and reconstructs any local `.git` repository folder. Instead of using high-level CLI commands (such as `git log` or `git show`), this tool reads, decompresses, and parses Git's raw database objects (blobs, trees, commits, and tags) stored under `.git/objects/`. It maps out the repository's historical DAG (Directed Acyclic Graph), lets users browse old tree states like a standard file explorer, shows diffs, and renders an interactive visual timeline of commit branches.

## 4. Problem Statement

Most developers use Git daily but treat it as a black box of magical commands. When merge conflicts, detached HEAD states, or corrupted repositories occur, they lack the conceptual model to understand what went wrong because they do not comprehend Git's underlying data model. 

Without understanding that:
- Commits are simple text metadata referencing trees.
- Trees are snapshots of directories referencing other trees or blobs.
- Blobs are content-only, deduplicated byte arrays.
- Git directories are just Content-Addressable Key-Value stores.

Developers struggle to navigate Git's power effectively. Typical online git tutorials are dry and visual representations are static. By building an application that reads a repository's actual binary metadata directly off the disk, developers learn exactly how version control works from the ground up.

## 5. Proposed Solution

The proposed application parses the `.git` folder of a selected directory on the system:
1. It reads reference files under `.git/refs/heads/` to find branch heads.
2. It locates Git objects in `.git/objects/` using the 40-character SHA-1 hash (where the first 2 characters are the folder name, and the remaining 38 are the file name).
3. It decompresses the binary objects using a `zlib` / `deflate` decoder.
4. It parses the object metadata structure (Header + Size + NULL byte + Content Payload).
5. It traverses the commit parents recursively to build a custom in-memory Directed Acyclic Graph (DAG) representation of repository history.
6. It displays this graph in an interactive visual interface, allowing developers to inspect file trees and file content at any historical point.

## 6. Project Goal

To build a standalone explorer tool that can parse any valid, non-packed `.git` directory, construct its historical commit tree, show file snapshot structures for selected commits, and render an interactive GUI mapping out branches, commits, and file contents.

## 7. Core Workflow

```text
Local Path (.git) ────> Scan Ref Heads (.git/refs/heads/)
                             │
                             ▼
                     Extract Commit Hash ───> Find Object in .git/objects/xx/yyyy...
                                                   │
                                                   ▼
                                           Decompress (zlib)
                                                   │
                                                   ▼
                                           Verify Header Type:
                                  ┌────────────────┴────────────────┐
                             [If Commit]                       [If Tree]
                                  │                                 │
                   Parse: Tree ID, Parent IDs,        Iterate binary entries:
                     Author, Commit Message            SHA Hash, File Mode, Name
                                  │                                 │
                                  ├─────────────────────────────────┘
                                  ▼
                    Reconstruct Tree Snapshot
                                  │
                                  ▼
               Render Timeline Graph & File Explorer
```

## 8. Functional Requirements

### Git Object Parser
- **Decompression**: Implement a buffer stream decoder that decompresses Git's `zlib` compressed files.
- **Header Classification**: Read the leading ASCII characters to identify whether an object is a `blob`, `tree`, `commit`, or `tag`, extract the size of the payload, and process the payload accordingly.
- **Commit Parsing**: Extract the root `tree` hash, parent commit hashes, author/committer names, emails, timestamps, and commit message strings from a commit payload.
- **Tree Parsing**: Decode the binary list of entries within a tree object. Each entry includes file permission modes, entry names (file or folder), and their respective 20-byte SHA-1 hash.
- **Blob Extraction**: Extract raw binary or text data from blob objects for user preview.

### Repository Explorer & Graph
- **Branch/Ref Resolution**: Scan `.git/refs/heads/` and `.git/HEAD` to resolve branch references and find the current checked-out commit.
- **Commit History DAG Construction**: Follow parent commit hashes to compile a fully traversed commit tree. Detect and map branches, merges, and head states.
- **Visual Graph Viewer**: Display an interactive timeline diagram (using canvas, SVG, or a terminal visualizer) drawing commits as nodes, connecting them to parental nodes, and clearly highlighting branch divergence and merge actions.
- **Interactive File Snapshot Explorer**: Clicking on a commit node loads its respective `tree` object. The user can navigate nested subdirectories (sub-trees) and select files (blobs) to read their historical content.

## 9. Non-Functional Requirements

- **Parsing Performance**: Decompression and graph building of a local repository with up to 500 commits must take less than 1.5 seconds.
- **Local Sandbox Access**: The tool must run with secure local file system permissions, and only open directories explicitly pointed to by the user.
- **Accuracy**: Decoded objects must perfectly match the output of native Git commands (e.g. `git cat-file -p <hash>`).
- **Memory Footprint**: Blobs (which can contain large files) must be read and rendered on-demand, rather than buffering all file contents in memory during the history scan.

## 10. Main Entities / Data Model

### GitObject (Base)
- **Hash**: String (40-char SHA-1 hex).
- **Type**: Enum (`blob`, `tree`, `commit`, `tag`).
- **Size**: Integer (uncompressed byte size).

### GitCommit (extends GitObject)
- **TreeHash**: String (reference to the snapshot root tree).
- **ParentHashes**: List of Strings (references to previous commits).
- **Author**: String.
- **Committer**: String.
- **Timestamp**: DateTime/Unix timestamp.
- **Message**: String.

### GitTreeEntry
- **Mode**: String (file system permission mode, e.g., `100644` for files, `040000` for sub-directories).
- **Name**: String (filename or folder name).
- **Hash**: String (SHA-1 hash of the target blob or sub-tree).
- **Type**: Enum (`blob` or `tree`).

## 11. System Components

- **File System Watcher / Scanner**: Responsible for locating `.git` folders, scanning refs, and loading raw file bytes into the buffer memory stream.
- **Zlib Decompression Engine**: Utilizes native or built-in decompression modules to expand packed buffers.
- **DAG Compiler**: Runs the recursive ancestor-tracking traversal algorithm to sort and build the structural history timeline list.
- **User Interface (UI)**:
  - **Commit Graph Timeline**: Graphical flow representing branches and merges.
  - **Snapshot Directory Tree**: Interactive folder list representing the folder state during the selected commit.
  - **File Viewer panel**: Highlighting text file previews.

## 12. Important Technical Challenges

### Parsing Binary Tree Objects
- **Challenge**: Unlike commits which are human-readable plaintext strings, Git tree objects are compressed binary arrays. They separate files using the NULL byte (`0x00`) followed immediately by the raw 20-byte SHA-1 hash (which is NOT hex-encoded but actual 20 binary bytes).
- **Concepts**: Byte array reading, converting raw 20-byte binary arrays into 40-character hex strings, indexing with offset pointers.
- **Research**: Read Git's internal documentation on the physical file format layout of Tree objects.

### Graph Rendering Layout (DAG)
- **Challenge**: Drawing a branch diagram is mathematically complex when there are multiple parallel branches and recursive merge commits. Determining which commit sits on which "swimlane" or column without overlapping requires a topological sorting strategy.
- **Concepts**: Topological sorting, depth-first search, layout algorithms for graphs.
- **Research**: Coordinate generation for timeline swimlanes.

### Git Packfiles (Scope Management)
- **Challenge**: As repositories grow, Git run-times compress objects into binary monoliths called `.pack` files along with index `.idx` offsets. Parsing packfiles is highly complex and beyond junior-plus scope.
- **Engineering Decision**: Restrict the project to loose objects (stored under `.git/objects/xx/yyyy...`). Guide the student on how to write a safety fallback or clear warning if a repository is packed (instructing them to run `git unpack-objects` locally for manual testing).

## 13. Suggested Technology Areas

- **Backend / Parser Engine**: Node.js (with standard library `fs` and `zlib`), Python, Go, or Rust.
- **Frontend / Visualization**: 
  - *Web Option*: React or Vue, with D3.js or HTML5 Canvas for graph rendering.
  - *Terminal Option*: blessed/blessed-contrib, termimad, or bubbletea (Go TUI).

## 14. Skills and Knowledge Gained

### Programming & File Systems
- Raw binary stream reading, parsing custom file headers, and buffer slicing.
- Working with file system APIs and scanning directories recursively.

### Algorithms & Systems
- Implementing graph structures (DAGs) and tree traversals.
- Understanding compression mechanics (zlib / deflate).
- Mastering Git's internal data structures and storage mechanics (Ref heads, Commit objects, Trees, and Blobs).

## 15. Recommended Development Phases

1. **Phase 1 - Decompress & Type Check**: Write a script that accepts a file path to a loose git object, decompresses it, parses the header prefix, prints the object type and size, and writes the raw decrypted content to console.
2. **Phase 2 - Parse Individual Objects**: Expand the parser to support parsing:
   - Plaintext Commit files (extract tree pointer, author, message, parents).
   - Binary Tree files (safely scan and print file permissions, file names, and target object hashes).
3. **Phase 3 - Ref Resolution & History Traversal**: Read `.git/refs/heads/main` to find the latest commit. Decompress it, read parents, and repeatedly traverse historical commits back to the initial root commit, storing them in a linked-list history.
4. **Phase 4 - DAG and Branch Tracker**: Handle multiple branches. Scan all files in `.git/refs/heads/`, combine histories, track where commits diverge, and construct a multi-branch commit graph model.
5. **Phase 5 - Simple GUI/CLI Graph Viewer**: Render the commits. For TUI, draw nodes using Unicode characters (`*`, `|`, `\`, `/`). For Web UI, draw SVG nodes linked on custom swimlanes.
6. **Phase 6 - Interactive File Snapshot**: Add functionality so clicking a commit node lists its files. Double-clicking folders opens directories. Selecting files loads and previews their content.

## 16. Testing Requirements

- **Unit Tests**: Mock zlib-compressed buffers representing predefined Git blobs, trees, and commits. Confirm the parser decodes entries, authors, and permissions with perfect accuracy.
- **Integration Tests**: Set up a script that dynamically initializes a temporary local git repository, creates a few commits, branches, and merges, and then runs the Visualizer on this newly created repository to verify compatibility.
- **Boundary Conditions**: Test handling of empty files, highly nested sub-folders, and commits with multi-line messages or non-ASCII characters.

## 17. Security Considerations

- **Path Traversal Protection**: Ensure the folder reader restricts itself to the selected `.git` workspace. It must not parse paths pointing outside of the root boundary (e.g., if a directory tree references symlinks or malicious parent directory paths).
- **Safe Execution Environment**: Since the visualizer parses arbitrary local directories, it must treat user files as untrusted. Strictly sanitize and escape text files rendered in the code panel to prevent script injection (XSS) if written as a web client.

## 18. Possible Extensions

- **Index File Parser**: Parse the binary `.git/index` file to show currently staged changes that differ from the `HEAD` commit.
- **Custom Git Diff Engine**: Write a line-by-line diff engine comparing two different hashes of a blob to highlight added/removed lines manually instead of calling external tools.
- **Lightweight Write Support**: Allow creating commits or writing trees directly through the tool!

## 19. Learning Questions

- How does Git handle file renaming under the hood, and why is there no "rename" object type?
- What are Git "loose objects" versus "packfiles", and how does Git optimize storage for highly repetitive files?
- Why does the hashing of a commit change when the commit time changes, even if the file contents and messages are identical?
- How does Git manage merge commits with multiple parents in the commit graph?

## 20. Completion Criteria

- [ ] The application can load a local directory, locate its `.git` folder, and read active branches.
- [ ] Successfully parses loose `blob`, `tree`, and `commit` binary formats without throwing decoding exceptions.
- [ ] Traverses commits recursively to reconstruct and visualize a complete branch history with at least two parallel branches.
- [ ] Selecting a commit displays its root folder snapshot, letting users navigate nested directories.
- [ ] User can click on file blobs inside the visual snapshot to read their exact plaintext contents.
- [ ] The app handles files safely and ignores packed repositories without crashing, providing clear user feedback.