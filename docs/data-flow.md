# Local Cocoa Service — Data Flow & Architecture

> Generated: 2026-04-09

---

## 1. System Overview

**Local Cocoa Service** is a FastAPI-based backend for a local-first AI document intelligence platform. It ingests files from watched folders, extracts and indexes their content, and exposes hybrid search, RAG Q&A, agentic reasoning, and memory extraction over the indexed corpus.

**Runtime stack:**

| Layer | Technology |
|---|---|
| HTTP server | FastAPI + Uvicorn (`127.0.0.1:8890`) |
| Metadata / FTS | SQLite (FTS5) |
| Vector store | Qdrant (local on-disk) |
| Embedding model | Qwen3-Embedding-0.6B (local HTTP, port 8005) |
| Vision model | Qwen3VL-2B-Instruct (local HTTP, port 8007) |
| Reranker | bge-reranker-v2-m3 (local HTTP, port 8006) |
| LLM / Chat | llama-cpp or remote (OpenAI / Anthropic / Gemini) |
| Speech-to-text | Whisper (local HTTP, port 8080) |

---

## 2. High-Level Module Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOCAL COCOA SERVICE                                  │
│                                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Indexer    │  │   Search     │  │    Agent     │  │   Memory     │    │
│  │  (Pipeline)  │  │  (Engine)    │  │ (Orchestrat) │  │  (Service)   │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                  │                  │             │
│  ┌──────▼──────────────────▼──────────────────▼──────────────────▼───────┐  │
│  │                         Storage Layer                                   │  │
│  │          SQLite (metadata + FTS5)  +  Qdrant (vectors)                 │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Parser     │  │   Chunker    │  │  LLM Client  │  │   Plugins    │    │
│  │ (Extractors) │  │  (Splitters) │  │ (Local/Remot)│  │(mail/notes/..)│   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Module Boundaries

```mermaid
graph TB
    subgraph ENTRY["Entry Points"]
        HTTP["HTTP API\n(FastAPI :8890)"]
        CLI["CLI Shell\n(python -m cli)"]
        POLL["Background Poll\n(folder scanner)"]
    end

    subgraph INDEXER["Indexer Module\n(app/services/indexer/)"]
        ORCH["Orchestrator"]
        SCAN["Scanner\n(file discovery)"]
        SCHED["TwoRoundScheduler"]
        FAST_TXT["FastTextStage\n(text extraction)"]
        FAST_EMB["FastEmbedStage\n(embedding)"]
        DEEP["DeepStage\n(vision analysis)"]
    end

    subgraph PARSERS["Parser Module\n(app/services/parser/)"]
        PDF["PDF Parser\n(PyMuPDF)"]
        DOCX["DOCX Parser"]
        IMG["Image Parser\n(OCR)"]
        AUD["Audio Parser\n(Whisper)"]
        VID["Video Parser\n(Whisper)"]
        PDF_VIS["PDF Vision Parser\n(VLM)"]
    end

    subgraph CHUNKER["Chunker Module\n(app/services/chunker/)"]
        CHUNK["Chunking Engine\n(fixed/semantic/hierarchical)"]
    end

    subgraph SEARCH["Search Module\n(app/services/search/)"]
        ENG["SearchEngine"]
        STRAT["Strategies\n(vector/lexical/hybrid/multi-path)"]
        COMP["Components\n(intent/verify/synthesize)"]
        PROG["Progressive\n(staged delivery)"]
    end

    subgraph AGENT["Agent Module\n(app/services/agent/)"]
        AGT_ORCH["AgentOrchestrator\n(ReAct loop)"]
        EXEC["Tool Executor"]
        REG["Tool Registry"]
        RUNMGR["Run Manager\n(external runs)"]
    end

    subgraph MEMORY["Memory Module\n(app/services/memory/)"]
        MEM_SVC["MemoryService"]
        CELL_EXT["MemCellExtractor"]
        MEM_EXT["MemoryExtractor\n(episodic/profile)"]
        PROFILE["ProfileManager"]
        CLUSTER["ClusterManager"]
    end

    subgraph STORAGE["Storage Layer"]
        SQLITE["SQLite\n(metadata + FTS5)"]
        QDRANT["Qdrant\n(vectors)"]
    end

    subgraph LLM_LAYER["LLM / Model Layer\n(app/services/llm/ + app/core/model_manager.py)"]
        EMB["Embedding Model\n(:8005)"]
        VLM["Vision Model\n(:8007)"]
        RERANK["Reranker\n(:8006)"]
        LLM["LLM / Chat\n(:8007 or remote)"]
        WHISPER["Whisper\n(:8080)"]
    end

    subgraph PLUGINS["Plugins\n(plugins/)"]
        MAIL["synvo_ai_mail\n(email)"]
        NOTES["synvo_ai_notes\n(notes)"]
        MCP["synvo_ai_mcp\n(MCP server)"]
    end

    HTTP --> INDEXER
    HTTP --> SEARCH
    HTTP --> AGENT
    HTTP --> MEMORY
    HTTP --> STORAGE
    CLI --> HTTP
    POLL --> ORCH

    ORCH --> SCAN
    ORCH --> SCHED
    SCHED --> FAST_TXT
    SCHED --> FAST_EMB
    SCHED --> DEEP
    FAST_TXT --> PARSERS
    DEEP --> PDF_VIS
    FAST_TXT --> CHUNKER
    FAST_EMB --> EMB
    DEEP --> CHUNKER
    DEEP --> EMB
    FAST_TXT --> SQLITE
    FAST_EMB --> QDRANT
    FAST_TXT --> MEMORY

    AUD --> WHISPER
    VID --> WHISPER
    PDF_VIS --> VLM

    ENG --> STRAT
    STRAT --> QDRANT
    STRAT --> SQLITE
    STRAT --> LLM
    STRAT --> RERANK
    ENG --> COMP
    COMP --> LLM

    AGT_ORCH --> REG
    AGT_ORCH --> LLM
    EXEC --> SEARCH
    EXEC --> STORAGE
    RUNMGR --> AGT_ORCH

    MEM_SVC --> CELL_EXT
    MEM_SVC --> MEM_EXT
    MEM_EXT --> PROFILE
    MEM_EXT --> CLUSTER
    MEM_SVC --> SQLITE

    CELL_EXT --> LLM
    MEM_EXT --> LLM

    PLUGINS --> SQLITE
    PLUGINS --> HTTP
```

---

## 4. Data Flow — File Indexing Pipeline

This is the primary **data production** path. Raw files are transformed into searchable, retrievable knowledge.

```mermaid
flowchart TD
    A([User adds folder via POST /folders]) --> B

    B["Scanner\nWalk directory tree\nFingerprint files\n(path + mtime + size)"]

    B -->|"New / changed file"| C["SQLite: INSERT file record\nstatus = pending"]
    B -->|"Unchanged file"| Z1([Skip — already indexed])

    C --> D["TwoRoundScheduler\ndequeues pending files"]

    subgraph FAST["⚡ FAST ROUND (Round 1)"]
        D --> F1

        F1["FastTextStage\n──────────────\n• Parser dispatched by extension\n• PDF → PyMuPDF\n• DOCX → python-docx\n• Image → OCR\n• Audio/Video → Whisper\n• Yields: TextBlockInfo[]"]

        F1 -->|"★ KEY OUTPUT: raw text blocks"| F2

        F2["Chunker\n──────────────\n• Split text into chunks\n• Respect semantic boundaries\n• Annotate: page_num, bbox,\n  source_regions, token_count"]

        F2 -->|"★ KEY OUTPUT: ChunkSnapshot[]"| F3

        F3["SQLite: INSERT chunks\n+ FTS5 index populated\n(enables keyword search)"]

        F3 --> F4

        F4["FastEmbedStage\n──────────────\n• Batch-send chunk texts\n  to Embedding Model (:8005)\n• Returns float[] vector\n  per chunk"]

        F4 -->|"★ KEY OUTPUT: embedding vectors"| F5

        F5["Qdrant: UPSERT vectors\n(chunk_id → float[])"]

        F5 --> F6["SQLite: file.fast_stage = done\nfile.index_status = fast_complete"]
    end

    F1 -.->|"if memory extraction enabled"| M1

    subgraph MEM["🧠 MEMORY EXTRACTION (async, during Fast Round)"]
        M1["MemCellExtractor\n• LLM call per chunk\n• Produce raw MemCell"]
        M1 -->|"★ KEY OUTPUT: MemCell[]"| M2
        M2["MemoryExtractor\n• Classify: episodic / profile\n• Extract structured records"]
        M2 -->|"★ KEY OUTPUT: episode, profile,\nforesight, event_log records"| M3
        M3["SQLite: INSERT memory tables\nfile.memory_status = extracted"]
    end

    subgraph DEEP["🔬 DEEP ROUND (Round 2, optional)"]
        F6 -->|"if deep_indexing enabled"| D1
        D1["DeepStage\n──────────────\n• PDF pages → images\n• VLM (:8007) analyses each page\n• Extracts tables, charts, layout\n• Produces richer TextBlockInfo[]"]
        D1 -->|"★ KEY OUTPUT: enriched text\n(with visual context)"| D2
        D2["Re-Chunk → Re-Embed\n(same pipeline as Fast Round\nbut with deep version flag)"]
        D2 --> D3["SQLite: file.deep_stage = done\nQdrant: vectors updated (version=deep)"]
    end

    F5 & D3 & M3 --> END([File fully indexed\nAvailable for search + agent + memory])
```

---

## 5. Data Flow — Search Query

```mermaid
flowchart TD
    A([Client: GET /search?q=...]) --> B

    B["SearchEngine.search()\n──────────────────\nParse: query, limit, filters,\nstrategy, privacy_level"]

    B --> C["Query Rewriting\n(optional LLM call)\n★ KEY: canonical query form"]

    C --> D{Strategy?}

    D -->|"hybrid (default)"| E1
    D -->|"multi-path"| E2
    D -->|"lexical"| E3
    D -->|"vector"| E4

    E1["Hybrid Search\n• Vector search → Qdrant\n• FTS5 keyword → SQLite\n• Merge & deduplicate hits"]

    E2["Multi-Path Pipeline\n• LLM decomposes query\n  into N sub-queries\n• Run each sub-query\n• Merge all result sets"]

    E3["FTS5 Lexical Search\n• BM25 ranking\n• Snippet extraction"]

    E4["Vector Search\n• Cosine similarity\n• Top-K from Qdrant"]

    E1 & E2 & E3 & E4 --> F

    F["Reranker (:8006)\n• Score (query, chunk) pairs\n• Re-sort by cross-encoder score\n★ KEY OUTPUT: ranked SearchHit[]"]

    F --> G["LLM Verification Component\n(optional)\n• Confirm each hit answers query\n• Add: has_answer, confidence,\n  analysis_comment"]

    G -->|"★ KEY OUTPUT: verified hits\nwith LLM commentary"| H

    H["Assemble SearchResponse\n• file metadata\n• snippet, page_num, bbox\n• score, confidence\n• source_regions"]

    H --> END([Return JSON to client])

    subgraph QA["POST /search/qa — RAG Q&A extension"]
        H --> QA1["LLM Synthesis\n• Top-K hits as context\n• Prompt: answer from context\n★ KEY OUTPUT: natural language answer\n+ citations"]
    end
```

---

## 6. Data Flow — Agent (ReAct Loop)

```mermaid
flowchart TD
    A([Client: POST /agent/stream\n{ query, history }]) --> B

    B["AgentOrchestrator.run()\n• Build system prompt\n  (native tool-call or fallback)\n• Inject available tools list"]

    B --> C["LLM Call\n(local llama-cpp or remote)"]

    C --> D{Response type?}

    D -->|"Final answer"| END([Stream: final answer tokens\nstatus=done])

    D -->|"Tool call"| E["Tool Executor\n──────────────\nDispatch by tool name:"]

    E --> T1["workspace_search\n→ SearchEngine.search()\n★ KEY: retrieves relevant chunks"]
    E --> T2["workspace_qa\n→ Full RAG pipeline\n★ KEY: LLM-synthesised answer"]
    E --> T3["get_document_chunks\n→ Storage.get_chunks(file_id)\n★ KEY: raw text of specific file"]
    E --> T4["list_files\n→ Storage.list_files()\n★ KEY: corpus overview"]
    E --> T5["get_file_content\n→ File bytes download"]

    T1 & T2 & T3 & T4 & T5 --> F["Tool result appended\nto conversation context"]

    F -->|"Next iteration"| C

    subgraph STREAM["NDJSON Event Stream"]
        S1["thinking_step"]
        S2["tool_call"]
        S3["tool_result"]
        S4["token (answer chars)"]
        S5["status"]
    end

    C -.->|"stream events"| STREAM
```

---

## 7. Data Flow — Memory Extraction (Standalone)

```mermaid
flowchart TD
    A([POST /memory/extract\n{ file_id, user_id }]) --> B

    B["MemoryService\n• Load file chunks from SQLite\n• Apply chunking strategy\n  (original chunks or re-chunk)"]

    B --> C["For each chunk →\nMemCellExtractor\n(LLM call)\n★ KEY OUTPUT: MemCell\n{ original_data, summary,\n  timestamp, metadata }"]

    C --> D["MemoryExtractor\n• Classify MemCells\n• Run extraction prompts"]

    D --> E1["EpisodicExtractor\n★ KEY OUTPUT: EpisodeRecord\n{ title, summary, episode,\n  timestamp, participants }"]

    D --> E2["ProfileExtractor\n★ KEY OUTPUT: ProfileRecord\n{ topic, subtopic,\n  profile_info }"]

    D --> E3["EventLogExtractor\n★ KEY OUTPUT: EventLogRecord\n{ event, entities, timestamp }"]

    D --> E4["ForesightExtractor\n★ KEY OUTPUT: ForesightRecord\n{ foresight (prediction) }"]

    E1 & E2 & E3 & E4 --> F["ProfileManager\n• Merge/update existing profiles\n• ClusterManager groups\n  related memories"]

    F --> G["SQLite: INSERT/UPDATE\nepisodes, profiles,\nevent_logs, foresights tables\nfile.memory_status = extracted"]

    G --> END([POST /memory/search\nreturns relevant memories\nfor a given user + query])
```

---

## 8. Key Information Production Summary

The table below lists every point in the system where **new structured knowledge** is synthesised or generated — the "value-add" transformations.

| # | Stage | Input | ★ Key Output | Where Stored |
|---|---|---|---|---|
| 1 | **FastTextStage** | Raw file bytes | `TextBlockInfo[]` — page-aware text blocks | in-memory only |
| 2 | **Chunker** | TextBlockInfo[] | `ChunkSnapshot[]` — semantic text units with spatial coords | SQLite `chunks` + FTS5 index |
| 3 | **FastEmbedStage** | Chunk texts | Float vectors (dense embeddings) | Qdrant |
| 4 | **DeepStage (VLM)** | PDF page images | Enriched TextBlockInfo with visual context (tables, charts) | in-memory → re-chunked → Qdrant |
| 5 | **Whisper transcription** | Audio / video | Transcript text | fed into Chunker → SQLite + Qdrant |
| 6 | **Query rewriting (LLM)** | Raw user query | Canonical / expanded query | in-memory only |
| 7 | **Multi-path decomposition (LLM)** | Single query | N sub-queries covering different aspects | in-memory only |
| 8 | **Reranker** | (query, chunk) pairs | Relevance scores → sorted `SearchHit[]` | returned in response |
| 9 | **Verification component (LLM)** | Query + top hits | `has_answer`, `confidence`, `analysis_comment` per hit | returned in response |
| 10 | **RAG synthesis (LLM)** | Top-K chunks as context | Natural language answer + citations | returned in response |
| 11 | **Agent ReAct loop (LLM)** | User task + tool results | Iterative reasoning trace + final answer | streamed + (optionally) stored in chat |
| 12 | **MemCellExtractor (LLM)** | Document chunk | `MemCell` — distilled atomic fact | in-memory |
| 13 | **EpisodicExtractor (LLM)** | MemCells | `EpisodeRecord` — time-bound event with participants | SQLite `episodes` |
| 14 | **ProfileExtractor (LLM)** | MemCells | `ProfileRecord` — durable user/entity trait | SQLite `profiles` |
| 15 | **EventLogExtractor (LLM)** | MemCells | `EventLogRecord` — structured event entity | SQLite `event_logs` |
| 16 | **ForesightExtractor (LLM)** | MemCells | `ForesightRecord` — predictive insight | SQLite `foresights` |
| 17 | **Whisper (audio/video)** | Audio stream | Transcript with timestamps | fed into indexing pipeline |
| 18 | **Scanner (file fingerprint)** | File system stat | Dedup hash (path+mtime+size) | SQLite `files` |

---

## 9. Inter-Process Communication

All ML inference is performed by separate processes exposed as HTTP services. The main FastAPI process is thin — it orchestrates, not infers.

```
┌─────────────────────────────────────────────────────────┐
│             Main FastAPI Process (:8890)                 │
│                                                           │
│  Indexer ──HTTP──► Embedding Model      :8005            │
│  Parser  ──HTTP──► Vision Model (VLM)   :8007            │
│  Search  ──HTTP──► Reranker             :8006            │
│  Agent   ──HTTP──► LLM / Chat           :8007 (or remote)│
│  Parser  ──HTTP──► Whisper              :8080            │
│                                                           │
│  All services ──► SQLite       (file-local)              │
│  All services ──► Qdrant       (file-local)              │
└─────────────────────────────────────────────────────────┘
```

Remote providers (OpenAI, Anthropic, Gemini, DeepInfra) replace the local ports when configured. Settings are persisted to `<runtime>/provider_config.json` and live-reloaded.

---

## 10. Privacy Boundary

```mermaid
flowchart LR
    subgraph Sources
        LOCAL_UI["Local UI\n(request_source=local_ui)"]
        EXT["External Client\n(request_source=external)"]
        MCP_SRC["MCP Client\n(request_source=mcp)"]
        PLUGIN_SRC["Plugin\n(request_source=plugin)"]
    end

    subgraph Gate["Privacy Gate (auth.py)"]
        PUB["PrivacyLevel = normal\n→ ALL sources allowed"]
        PRIV["PrivacyLevel = private\n→ local_ui ONLY"]
    end

    LOCAL_UI -->|"all files"| PUB & PRIV
    EXT -->|"normal files only"| PUB
    MCP_SRC -->|"normal files only"| PUB
    PLUGIN_SRC -->|"normal files only"| PUB
    EXT -.->|"blocked"| PRIV
    MCP_SRC -.->|"blocked"| PRIV
    PLUGIN_SRC -.->|"blocked"| PRIV
```

`PrivacyLevel` is set at folder or file level and inherited by every `ChunkSnapshot` — the gate is enforced inside both SQLite queries (FTS5 search) and Qdrant vector filters.

---

## 11. Plugin Data Boundary

Each plugin runs with an isolated SQLite schema (prefixed tables) and registers its own FastAPI router. Plugins can call back into core services (storage, LLM client) but core services never import from plugin code — the dependency arrow is one-directional.

```
plugin (synvo_ai_mail)
    ├── plugin.json          — manifest
    ├── router.py            — registers /mail/* routes
    ├── storage.py           — prefixed SQLite tables (mail_*)
    └── service.py           → imports app.core.*  (one-way)
                             → imports app.services.storage.*
                             ✗ app/ never imports plugins/*
```

---

*End of document.*
