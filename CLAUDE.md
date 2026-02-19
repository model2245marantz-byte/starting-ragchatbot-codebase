# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Rules

- Always use `uv` to run Python and manage dependencies. Never use `pip` directly.

## Commands

```bash
# Install dependencies
uv sync

# Run the application (from project root)
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# Run a single backend module directly
cd backend && uv run python <module>.py
```

The app serves at http://localhost:8000 (web UI) and http://localhost:8000/docs (API docs).

## Environment

Requires a `.env` file in the project root with `ANTHROPIC_API_KEY`. Copy from `.env.example`.

## Architecture

This is a full-stack RAG chatbot for querying course materials. Python 3.13+, uv package manager, no test framework configured.

### Backend (`backend/`)

The system uses a **tool-calling RAG pattern** — Claude decides autonomously whether to search via a `search_course_content` tool, rather than pre-fetching context on every query.

**Orchestration flow:**
1. `app.py` — FastAPI endpoints (`POST /api/query`, `GET /api/courses`). Entry point. Mounts `frontend/` as static files at `/`.
2. `rag_system.py` — `RAGSystem` ties all components together. Its `query()` method wraps the user question, retrieves session history, calls the AI generator with tools, collects sources, and updates the session.
3. `ai_generator.py` — `AIGenerator` makes **two Claude API calls** per tool-using query: first with tools attached (`tool_choice: auto`), then a follow-up with the tool results but **no tools** to force a text response. Conversation history is injected into the system prompt, not as message turns.
4. `search_tools.py` — `Tool` (ABC) → `CourseSearchTool` → `ToolManager`. The tool manager registers tools, routes execution by name, and tracks sources from the last search for the response.
5. `vector_store.py` — `VectorStore` wraps ChromaDB with two collections: `course_catalog` (course metadata, used for fuzzy course name resolution) and `course_content` (chunked text for semantic search). Course name resolution does a semantic search against the catalog to map partial names to full titles.
6. `document_processor.py` — Parses structured `.txt` files (header: Course Title/Link/Instructor, then `Lesson N:` markers) into `Course` and `CourseChunk` models. Chunks text on sentence boundaries with configurable size (800 chars) and overlap (100 chars).
7. `session_manager.py` — In-memory per-session conversation history, capped at 2 exchanges (4 messages). History is formatted as a plain string (`"User: ...\nAssistant: ..."`) appended to the system prompt.
8. `config.py` — Singleton `Config` dataclass loaded from env vars. All tunables (chunk size, overlap, max results, model name, ChromaDB path) live here.
9. `models.py` — Pydantic models: `Lesson`, `Course`, `CourseChunk`.

### Frontend (`frontend/`)

Vanilla HTML/CSS/JS with a dark theme. No build step. `script.js` manages session state, sends queries via fetch, and renders markdown responses with `marked.js`. Served as static files by FastAPI.

### Data (`docs/`)

Course transcript `.txt` files loaded automatically on startup via `app.py:startup_event`. Already-loaded courses are skipped by checking titles against ChromaDB.

### Key Design Details

- ChromaDB is persisted at `./chroma_db` (relative to `backend/`).
- Embeddings use `all-MiniLM-L6-v2` via sentence-transformers, configured through ChromaDB's `SentenceTransformerEmbeddingFunction`.
- The AI model is `claude-sonnet-4-20250514` with `temperature=0` and `max_tokens=800`.
- Sources shown in the UI come from `CourseSearchTool.last_sources`, not from the AI response itself.
