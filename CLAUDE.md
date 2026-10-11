# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A RAG chatbot that answers questions about course transcripts in `docs/`. FastAPI backend + vanilla JS frontend, ChromaDB vector store, Anthropic Claude for generation. Python 3.13, managed with `uv`.

## Commands

```bash
uv sync                                         # install deps (creates .venv)
cd backend && uv run uvicorn app:app --reload --port 8000   # run server
./run.sh                                        # same, from repo root (Git Bash on Windows)
```

- UI at http://localhost:8000, Swagger at http://localhost:8000/docs.
- **The server must be started from `backend/`** — `app.py` and `config.py` use cwd-relative paths (`../docs`, `../frontend`, `./chroma_db`).
- Requires `ANTHROPIC_API_KEY` in `.env` at repo root (see `.env.example`). It is read once at import; `--reload` only watches `.py` files, so restart after editing `.env`.
- To re-ingest documents, stop the server and delete `backend/chroma_db/`. Startup skips any course whose title already exists in the catalog, so edits to an existing `docs/` file are otherwise ignored.
- Quick API check: `curl -X POST localhost:8000/api/query -H "Content-Type: application/json" -d '{"query":"..."}'`. `/api/query` wraps every exception as a 500 with the real error in `detail`.

There is no test suite, linter, or formatter configured. `main.py` at the root is an unused stub.

## Architecture

`RAGSystem` (`backend/rag_system.py`) is the orchestrator; `app.py` holds a single module-level instance and exposes `POST /api/query`, `GET /api/courses`, and mounts `frontend/` as static files at `/`.

**Tool-based (agentic) retrieval, not retrieve-then-generate.** `AIGenerator.generate_response` sends the query to Claude with the `search_course_content` tool. Claude decides whether to search:
- No tool call → the first response is returned directly (general-knowledge questions).
- Tool call → `_handle_tool_execution` runs the tool via `ToolManager`, then makes a **second API call without tools**, so at most one round of searching per query. The system prompt in `ai_generator.py` also instructs "one search per query maximum".

**Sources are passed out-of-band.** `CourseSearchTool._format_results` stores `last_sources` on the tool instance; `RAGSystem.query` reads them via `ToolManager.get_last_sources()` after generation and then calls `reset_sources()`. New tools that should surface sources need a `last_sources` attribute. Register new tools in `RAGSystem.__init__` by subclassing `Tool` (`search_tools.py`).

**Two ChromaDB collections** (`vector_store.py`), both embedded with `all-MiniLM-L6-v2`:
- `course_catalog` — one doc per course; ID and document text are the course title; lessons stored as a JSON string in `lessons_json` metadata. Used by `_resolve_course_name` to fuzzy-match a partial course name (e.g. "MCP") to an exact title via vector search (top-1, no threshold).
- `course_content` — chunks with `course_title`, `lesson_number`, `chunk_index` metadata; IDs are `{title_with_underscores}_{chunk_index}`. `search()` resolves the course name, builds a `where` filter, and queries this collection.

**Document format** expected by `DocumentProcessor.process_course_document`:
```
Course Title: ...
Course Link: ...
Course Instructor: ...

Lesson 0: Title
Lesson Link: ...      (optional, line right after the lesson marker)
...transcript...
```
Text is split into sentence-based chunks (`CHUNK_SIZE=800` chars, `CHUNK_OVERLAP=100`). Note the inconsistent chunk prefixes: the first chunk of each lesson gets `"Lesson N content: "`, but every chunk of the *last* lesson gets `"Course {title} Lesson N content: "`.

**Sessions** (`session_manager.py`) are in-memory only, IDs are sequential (`session_1`, …) and reset on server restart. The server creates an ID on the first query with no `session_id`; the frontend keeps it in a JS variable, so each page load/tab is a new session. History is capped at `MAX_HISTORY` *exchanges* (×2 messages) and is injected as plain text into the **system prompt**, not as `messages` entries. Only the raw user query and final answer are stored.

## Config notes

All tunables live in `backend/config.py`. The current model (`claude-sonnet-5-5`) rejects the `temperature` parameter, so `AIGenerator.base_params` deliberately omits it; check parameter compatibility when changing `ANTHROPIC_MODEL`.
