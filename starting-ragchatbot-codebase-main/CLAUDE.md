# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Install dependencies
uv sync

# Start dev server (from project root)
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# Access
# Web UI: http://localhost:8000
# API docs: http://localhost:8000/docs
```

Requires Python 3.13+, `uv`, and `ANTHROPIC_API_KEY` set in a `.env` file at the project root.

## Architecture

This is an **agentic RAG chatbot** — Claude decides whether to search course content (via Anthropic tool-use) rather than always retrieving context.

**Request flow:**
```
Frontend (vanilla JS) → POST /api/query → RAGSystem.query()
  → AIGenerator sends question + tool definition to Claude
  → Claude optionally calls search_course_content tool
    → CourseSearchTool → VectorStore.search() (ChromaDB semantic search)
    → results returned as tool_result → Claude synthesizes final answer
  → response + sources returned to frontend
```

**Key components (all in `backend/`):**
- `app.py` — FastAPI entry point; mounts `frontend/` as static files at `/`; on startup loads all docs from `docs/` into ChromaDB
- `rag_system.py` — Orchestrator wiring all components together
- `ai_generator.py` — Claude API client; handles tool-use loop (single round: initial response → tool execution → final response)
- `vector_store.py` — Two ChromaDB collections: `course_catalog` (one doc per course for fuzzy name resolution) and `course_content` (chunked text for semantic search)
- `document_processor.py` — Parses structured course transcript files, splits into sentence-based chunks (800 chars, 100 overlap)
- `search_tools.py` — Abstract `Tool` base class + `CourseSearchTool` implementation + `ToolManager` registry
- `session_manager.py` — In-memory conversation history (default 2 exchanges)
- `config.py` — All tunable parameters as a dataclass; reads `ANTHROPIC_API_KEY` from `.env`

**Document format** expected in `docs/`:
```
Course Title: ...
Course Link: ...
Course Instructor: ...

Lesson 0: Title
Lesson Link: ...
<transcript text>
```

## Key Design Decisions

- Tool-use is **single-round only**: Claude gets one tool call, then must produce a final answer (no multi-step tool loops).
- Course name resolution uses semantic search on `course_catalog` — partial/fuzzy names like "MCP" resolve to full titles.
- Conversation history is passed as a formatted string in the system prompt, not as structured message history.
- ChromaDB runs as a persistent local client (stored at `backend/chroma_db/`); no external database needed.
- Embeddings use `all-MiniLM-L6-v2` via Sentence Transformers (same model for indexing and querying).
- The frontend is plain HTML/CSS/JS with no build step; served directly by FastAPI's static file mount.
