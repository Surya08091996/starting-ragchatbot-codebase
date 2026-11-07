# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

Must be run from the `backend/` directory — the frontend static files are mounted at `../frontend` relative to where the server starts.

App is served at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

**Environment:** Create `.env` in the project root (not in `backend/`):
```
ANTHROPIC_API_KEY=sk-ant-...
```

**Install dependencies:**
```bash
uv sync
```

## Architecture

The backend is a FastAPI app (`backend/app.py`) that serves both the REST API (`/api/*`) and the frontend as static files (`/`).

**Request flow for a user query:**

1. `frontend/script.js` — `sendMessage()` POSTs `{ query, session_id }` to `/api/query`
2. `backend/app.py` — validates with Pydantic, delegates to `RAGSystem.query()`
3. `backend/rag_system.py` — the central orchestrator:
   - Fetches conversation history from `SessionManager`
   - Calls `AIGenerator.generate_response()` with the query, history, and tool definitions
   - After response, collects sources from `ToolManager` and saves the exchange to session history
4. `backend/ai_generator.py` — makes up to **two Claude API calls**:
   - Call 1: with `tool_choice: auto` — Claude decides whether to search
   - If `stop_reason == "tool_use"`: executes tools, then Call 2 without tools to synthesize the final answer
5. `backend/search_tools.py` → `backend/vector_store.py` — `CourseSearchTool` runs a vector similarity search on ChromaDB
6. Response returns `{ answer, sources, session_id }` — sources are rendered as a collapsible section in the UI

**Key architectural patterns:**
- `RAGSystem` owns all subsystem instances and is the only thing `app.py` talks to
- Claude tool use is the retrieval trigger — the AI decides when to search, not the app
- `SessionManager` is in-memory only; sessions are lost on server restart
- ChromaDB has two collections: `course_catalog` (one doc per course, for semantic course name resolution) and `course_content` (chunked text, for similarity search)
- Documents in `docs/` are ingested once at startup; re-runs skip already-indexed courses by title

**Adding a new tool:** Subclass `Tool` in `search_tools.py`, implement `get_tool_definition()` and `execute()`, then call `tool_manager.register_tool(your_tool)` in `RAGSystem.__init__()`.

## Key Configuration (`backend/config.py`)

| Setting | Default | Notes |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-6` | Use valid model IDs from the claude-sonnet-4-6 / claude-haiku-4-5 family |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | SentenceTransformer model; downloaded on first run |
| `CHROMA_PATH` | `./chroma_db` | Relative to `backend/` |
| `CHUNK_SIZE` | `800` | Characters per content chunk |
| `MAX_RESULTS` | `5` | Vector search results returned per query |
| `MAX_HISTORY` | `2` | Conversation turns kept in session |

## Package Manager

This project uses `uv`. Do not use `pip` or `python -m uv`. Always use `uv run` or `uv sync`.
