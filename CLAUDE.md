# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

```bash
# First-time setup
cp .env.example .env   # then add ANTHROPIC_API_KEY

# Start the server (installs deps automatically via uv)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app serves on `http://localhost:8000`. API docs at `/docs`.

There are no tests and no linter configured in this project.

Always use `uv` to manage dependencies and run Python — never use `pip` directly.

```bash
uv add <package>        # add a dependency
uv remove <package>     # remove a dependency
uv sync                 # install all dependencies from pyproject.toml
uv run <command>        # run a command in the project environment
```

## Architecture

This is a full-stack RAG chatbot. The FastAPI backend serves both the REST API and the static frontend files. All backend logic lives in `backend/`; the frontend is plain HTML/JS in `frontend/`.

**Request flow for a user query:**

1. `frontend/script.js` — `sendMessage()` POSTs `{ query, session_id }` to `/api/query`
2. `backend/app.py` — creates a session if needed, delegates to `RAGSystem.query()`
3. `backend/rag_system.py` — fetches conversation history, calls `AIGenerator.generate_response()` with the search tool
4. `backend/ai_generator.py` — makes the first Claude API call; if Claude decides to search (`stop_reason == "tool_use"`), executes the tool and makes a second Claude call to synthesize the answer
5. `backend/search_tools.py` → `backend/vector_store.py` — `CourseSearchTool` calls `VectorStore.search()`, which embeds the query and does a semantic search in ChromaDB, with optional course/lesson filters
6. Sources and answer return up the chain; `SessionManager` records the exchange

**Two ChromaDB collections:**
- `course_catalog` — one entry per course (title, instructor, link, lessons list as JSON)
- `course_content` — chunked lesson text with `course_title` and `lesson_number` metadata for filtering

**Course name resolution:** when Claude passes a `course_name` filter, `VectorStore._resolve_course_name()` does a semantic vector search against `course_catalog` to find the best match before filtering `course_content`.

## Key Configuration (`backend/config.py`)

| Setting | Value | Purpose |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Model used for generation |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence-transformers model for embeddings |
| `CHUNK_SIZE` | 800 chars | Max chunk size for document splitting |
| `CHUNK_OVERLAP` | 100 chars | Sentence overlap between consecutive chunks |
| `MAX_RESULTS` | 5 | Search results returned per query |
| `MAX_HISTORY` | 2 | Conversation exchanges kept in session context |
| `CHROMA_PATH` | `./chroma_db` | Persistent ChromaDB storage (relative to `backend/`) |

## Document Format

Course files in `docs/` must follow this structure for the parser to extract structured metadata:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <title>
Lesson Link: <url>
<lesson content...>

Lesson 1: <title>
...
```

The `course_title` field is the unique identifier — duplicate titles are skipped on startup. To force a re-index, delete `backend/chroma_db/`.

## Adding a New Tool

Tools follow the `Tool` ABC in `backend/search_tools.py`. Implement `get_tool_definition()` (returns an Anthropic tool schema dict) and `execute(**kwargs)`, then register with `tool_manager.register_tool()` in `RAGSystem.__init__()`. The tool will automatically be included in Claude's first API call.
