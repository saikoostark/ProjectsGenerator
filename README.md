# ProjectsGenerator

An expert-designed technical project generator and mentor framework that transforms Large Language Models (LLMs) into rigorous software architects, producing production-grade, unique, and non-trivial programming project specifications.

---

## 🎯 Main Purpose

Most developer project guides and portfolio repositories suffer from tutorial fatigue: endless variations of basic CRUD (Create, Read, Update, Delete) applications like standard task managers, simple blog platforms, or basic chat apps. 

**ProjectsGenerator** solves this by establishing a rigorous specification framework for software projects aimed at **Junior to Mid-Level developers**. Its core objectives are:
- **Learning Through Engineering Constraints**: Focusing on real-world engineering challenges (concurrency, distributed systems, streaming, data consistency, caching, low-level networking, and security) rather than superficial UI scaffolding.
- **Deep Technical Diversity**: Exposing developers to diverse technical stacks and architectural patterns (REST, GraphQL, WebSockets, SSE, message brokers, event-driven architectures, search engines, object storage, and selective AI integration).
- **Fostering Critical Thinking**: Forcing developers to weigh architecture trade-offs (e.g., SQL vs. NoSQL, Polling vs. WebSockets, strong vs. eventual consistency).

---

## 🤖 The LLM Synergy: Intelligence Meets Solid Requirements

ProjectsGenerator shines brightest when used as a system prompt / guidance framework combined with a capable **LLM model**. 

While LLMs are naturally creative, they often default to generic, repetitive ideas when asked open-ended questions like *"give me a project idea."* **ProjectsGenerator.md** acts as a strict cognitive constraint and blueprint that unlocks the LLM's full potential:

1. **Systemized Architectural Mentorship**: Instead of a casual chat response, the LLM adopts the persona of an expert technical mentor and systems architect.
2. **Living Knowledge Base Inspection**: Before generating any new project, the LLM scans all existing `.md` project files in the repository. It treats them as a source of truth for what has already been built.
3. **Semantic Anti-Repetition Engine**: The LLM evaluates uniqueness *semantically* rather than via superficial keyword matching. It prevents the common trap of repackaging an inventory CRUD app into a "museum artifact catalog" by ensuring the underlying *engineering challenge* is genuinely novel.
4. **Comprehensive 20-Section Specifications**: Guided by the strict requirements in `ProjectsGenerator.md`, the LLM autonomously outputs exhaustive, professional-grade Markdown specifications complete with system goals, core workflows, ASCII sequence diagrams, functional requirements, technical challenges, and testable completion criteria.

---

## ⚙️ How the Mechanism Works

1. **Inspection**: The model reads existing specification files in the workspace (e.g., `reverse-proxy-and-load-balancer.md`, `custom-lsm-tree-storage-engine.md`).
2. **Ideation & Filtering**: It formulates a candidate project concept and evaluates it against quality gates (Junior–Mid level feasibility, non-CRUD complexity, meaningful engineering constraint, zero artificial novelty).
3. **Specification Generation**: Upon passing validation, the LLM generates a complete `.md` project specification file adhering strictly to the 20-section structure:
   - *Title & Difficulty Rationale*
   - *Overview & Problem Statement*
   - *Proposed Solution & Core Goals*
   - *Core Workflow (with ASCII Sequence Diagrams)*
   - *Functional & Non-Functional Requirements*
   - *Main Entities / Data Model*
   - *System Components & Technical Challenges*
   - *Suggested Technology Areas & Skills Gained*
   - *Recommended Development Phases*
   - *Testing, Security, Extensions & Learning Questions*
   - *Completion Criteria*

---

## 📂 Existing Project Catalog (Sample Index)

The repository contains a diverse library of generated project specifications, including:

- **[Reverse Proxy and Load Balancer](./reverse-proxy-and-load-balancer.md)** *(Mid-Level | Networking)*: Low-level HTTP/TCP gateway featuring round-robin & least-connections load balancing, active health checks, and zero-buffering stream piping.
- **[Custom LSM Storage Engine](./custom-lsm-tree-storage-engine.md)** *(Mid-Level | Data Structures)*: Log-Structured Merge-tree storage engine with memtables, WAL (Write-Ahead Logging), SSTables, and background compaction.
- **[Distributed Job Scheduler with Redis](./distributed-job-scheduler-redis.md)** *(Mid-Level | Distributed Systems)*: Fault-tolerant job queue supporting retries, exponential backoff, cron scheduling, and worker concurrency locks.
- **[Real-Time Audio Frequency Visualizer](./audio-synthesizer-dsp-wasm.md)** *(Junior+ | DSP / WebAssembly)*: Browser-based audio processing pipeline featuring Web Audio API nodes, custom DSP filters, and canvas frequency rendering.
- **[Intelligent Log Aggregation Agent](./intelligent-log-aggregation-agent.md)** *(Mid-Level | DevOps / Systems)*: Lightweight log collector agent featuring regex log parsing, backpressure handling, and batch transmission over TLS.

---

## 🚀 Getting Started & Usage

### Using with an LLM Agent / Assistant
1. Provide the contents of **`ProjectsGenerator.md`** as a system instruction or context prompt to your LLM.
2. Ensure the LLM has access to your project workspace directory containing the existing `.md` specification files.
3. Request a new project generation (e.g., *"Analyze existing projects and generate a new unique backend/networking project specification as a markdown file"*).
4. The LLM will inspect existing files, verify uniqueness against the semantic anti-repetition rules, and write a brand new `<project-title>.md` specification file.
