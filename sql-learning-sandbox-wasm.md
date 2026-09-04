## 1. Project Title

SQL Learning Sandbox WASM

## 2. Difficulty

Junior+

### Rationale
This project focuses on client-side safety and browser-based systems. It teaches developers how to leverage WebAssembly (WASM) to run complex, server-side tools (like SQLite) entirely within the browser's sandbox. It involves working with WASM bridges, SQL AST (Abstract Syntax Tree) parsing for query restriction, and building a responsive UI that updates immediately based on SQL execution results.

## 3. Project Overview

The SQL Learning Sandbox WASM is an interactive in-browser playground for learning SQL. It allows users to write, execute, and visualize SQL queries without a backend server. The engine utilizes an SQLite database engine compiled to WebAssembly, running locally within the browser's thread. It includes a security layer to restrict or whitelist dangerous queries (e.g., `DROP TABLE`, `ATTACH DATABASE`), schema visualization, and real-time execution result rendering.

## 4. Problem Statement

Learning SQL usually requires setting up a local database server, installing drivers, or relying on online, hosted SQL playgrounds. 
- Local setups are intimidating for beginners and hard to reset cleanly.
- Hosted playgrounds are slow, rely on server capacity, and often have strict timeouts, limitations, or privacy issues (data sent to a server).
- Developers lack a safe, immediate environment to experiment with complex `JOIN` or `WINDOW` functions without fearing they will break their database or compromise data.

Providing a fast, offline-first, client-side SQL playground removes friction and promotes safe, rapid experimentation.

## 5. Proposed Solution

The application is a pure front-end web project:
1. **WASM Runtime**: The browser initializes an SQLite WASM build.
2. **Persistence**: The database sits in the browser's IndexedDB or memory.
3. **Execution Layer**: User input is sent to a query-interceptor that restricts forbidden commands.
4. **Sandboxed Executor**: Validated SQL is sent to the WASM SQLite worker.
5. **Renderer**: Results are streamed back and rendered as dynamic, interactive tables.

## 6. Project Goal

To build a fully functional, browser-only SQL playground where users can define a schema, insert data, and run complex `SELECT` queries with real-time feedback, all running purely in client-side memory.

## 7. Core Workflow

```text
User Input (Editor) ──> Security Layer (AST Parser) ──> Forbidden? (Show Error)
                                  │
                               [Allowed]
                                  │
                                  ▼
                        SQL Worker (WASM SQLite)
                                  │
                                  ▼
                        Execution Results (JSON)
                                  │
                                  ▼
                        Result UI (Table/Grid)
```

## 8. Functional Requirements

### SQL Engine (WASM-based)
- **Engine Initialization**: Load and instantiate the SQLite WASM library.
- **Query Execution**: Execute SQL commands against the in-memory database and handle errors (syntax errors, constraint violations).
- **Schema Management**: Provide a mechanism to initialize (create tables) and reset the database.

### Security & Sanitization
- **AST Parsing**: Use a client-side SQL parser to generate an Abstract Syntax Tree (AST).
- **Command Restriction**: Programmatically traverse the AST to detect and reject dangerous keywords (e.g., `DROP`, `DELETE`, `TRUNCATE`, `ALTER`) based on sandbox policy.

### User Interface
- **SQL Editor**: Code editor with syntax highlighting for SQL (using libraries like `Monaco` or `CodeMirror`).
- **Data Viewer**: Tabular view of query results.
- **Schema Visualizer**: Automatically generate a sidebar showing table definitions, column types, and primary keys.

## 9. Non-Functional Requirements

- **Performance**: Query execution must be near-instant (under 100ms) for reasonable result sets.
- **Offline Capabilities**: The entire application (HTML/JS/WASM/SQL library) must function offline.
- **Security**: The security layer must be foolproof—no mechanism should exist to bypass query restriction.

## 10. Main Entities / Data Model

### Sandbox Configuration
- **AllowedQueries**: Enum/List of permitted command types (e.g., `SELECT`, `INSERT`, `UPDATE`).
- **InitialSchema**: Array of `CREATE TABLE` strings.

### ExecutionResult
- **Rows**: Array of Objects (Column Name -> Value).
- **Columns**: Array of Strings (Header names).
- **ExecutionTime**: Number (ms).
- **Error**: String (optional).

## 11. System Components

- **Frontend Application**: Editor, Schema Visualizer, and Result Viewer.
- **Query Interceptor**: AST parser to validate SQL.
- **SQLite Worker**: Web Worker running the WASM SQLite engine.

## 12. Important Technical Challenges

### AST-based Query Restriction
- **Challenge**: Simple string matching (e.g., looking for "DROP") is easily bypassed (e.g., `  DRop `).
- **Concepts**: Abstract Syntax Trees (AST), SQL grammars.
- **Research**: SQL parser libraries for JS/TS.

### Web Worker Communication
- **Challenge**: The WASM runtime should run in a Web Worker thread to prevent blocking the UI during complex queries. Communicating between the UI thread and Worker thread involves serialization overhead.
- **Concepts**: `postMessage`, `SharedArrayBuffer` (for performance), worker life cycles.

## 13. Suggested Technology Areas

- **Frontend**: React, Vue, or Svelte.
- **SQL Parsing**: `sql-parser` or `node-sql-parser` (adapted for browsers).
- **SQL Engine**: `sqlite-wasm`.
- **Editor**: `Monaco Editor` or `CodeMirror`.

## 14. Skills and Knowledge Gained

### Browser Systems
- Running WASM modules and using Web Workers for non-blocking UI.
- Understanding the browser sandbox and local storage (IndexedDB).

### SQL & Algorithms
- Parsing SQL commands into AST structures.
- Designing secure playgrounds for dangerous languages.

## 15. Recommended Development Phases

1. **Phase 1 - SQLite WASM Hello World**: Get `sqlite-wasm` running in a basic HTML page and execute a simple `SELECT 1` query.
2. **Phase 2 - UI & Editor**: Set up a code editor for writing SQL and a table for displaying the output.
3. **Phase 3 - Query Interceptor**: Implement the AST parser. Demonstrate blocking `DROP` commands while allowing `SELECT`.
4. **Phase 4 - Schema Visualizer**: Dynamically query `sqlite_master` to list table names and display them in the sidebar.
5. **Phase 5 - Performance Tuning**: Move the SQL engine into a Web Worker and add query execution time measurement.

## 16. Testing Requirements

- **Unit Tests**: Test the SQL parser's ability to correctly identify and block dangerous queries, even with weird casing/whitespace.
- **Integration Tests**: Verify successful result rendering for various SQL query types (`JOIN`, `GROUP BY`, `UNION`).

## 17. Security Considerations

- **Client-Side-Only Security**: Since the code executes entirely in the client's browser, the developer must explicitly acknowledge that this sandbox cannot protect against a malicious user who simply disables or modifies the client-side JavaScript.
- **Resource Exhaustion**: Large queries (e.g., `SELECT * FROM table CROSS JOIN table ...`) could freeze the browser tab. Implement a client-side execution timeout.

## 18. Possible Extensions

- **Import/Export**: Allow users to import CSV/JSON and export query results.
- **Tutorial Mode**: Build a step-by-step tutorial suite that checks query answers automatically.
- **Graphing/Charting**: Convert SQL results into visual charts using libraries like Chart.js.

## 19. Learning Questions

- How does WebAssembly (WASM) allow running code written in C/C++ inside a browser?
- Why is AST parsing safer than RegEx for validating SQL queries?
- What are the limitations of browser-based security for client-side sandboxes?

## 20. Completion Criteria

- [ ] SQLite WASM successfully executes queries in the browser.
- [ ] A code editor allows writing and running SQL queries.
- [ ] A security layer blocks `DROP` or unauthorized commands.
- [ ] Query results display in a readable, formatted table.
- [ ] The schema visualizer sidebar updates based on the database content.
- [ ] Executing heavy queries does not freeze the UI thread.