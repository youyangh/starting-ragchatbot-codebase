# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Course Materials RAG (Retrieval-Augmented Generation) system - a full-stack web application that enables intelligent querying of educational course materials using semantic search and AI-powered responses.

**Stack**: FastAPI backend, vanilla JavaScript frontend, ChromaDB vector database, Anthropic Claude API, SentenceTransformer embeddings.

## Package Management

**IMPORTANT**: This project uses `uv` as the package manager. **ALWAYS use `uv` commands, NEVER use `pip` directly.**

- Install packages: `uv sync` (not `pip install`)
- Run Python scripts: `uv run <command>` (not `python <script>`)
- Run server: `uv run uvicorn app:app --reload` (not `python -m uvicorn`)

## Environment Setup

1. **Install uv** (Python package manager):
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. **Install dependencies**:
   ```bash
   uv sync
   ```

3. **Configure environment variables**:
   ```bash
   cp .env.example .env
   # Edit .env and add your ANTHROPIC_API_KEY
   ```

## Running the Application

**Quick start**:
```bash
./run.sh
```

**Manual start**:
```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

Access at:
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

## Architecture Overview

### RAG Query Flow

The system implements an **agentic RAG pattern** where Claude autonomously decides when to search:

1. **Frontend** (`frontend/script.js`) → POST `/api/query` with user question
2. **API Endpoint** (`backend/app.py`) → Validates request, manages session
3. **RAG Orchestrator** (`backend/rag_system.py`) → Coordinates all components
4. **AI Generator** (`backend/ai_generator.py`) → Makes **two Claude API calls**:
   - First call: Claude decides whether to use search tool
   - If tool needed: Execute search and make second call with results
5. **Tool Manager** (`backend/search_tools.py`) → Executes `search_course_content` tool
6. **Vector Store** (`backend/vector_store.py`) → Performs semantic search:
   - Resolves course name (handles partial matches like "MCP" → "Introduction to MCP Servers")
   - Builds metadata filters (course_title, lesson_number)
   - Searches `course_content` collection with ChromaDB
7. **Response** → Returns formatted answer with source citations

### Key Components

**Core Processing Pipeline**:
- `rag_system.py` - Main orchestrator that ties everything together
- `ai_generator.py` - Manages Claude API interaction with tool calling
- `vector_store.py` - ChromaDB interface with two collections:
  - `course_catalog`: Course metadata (titles, instructors) for fuzzy name matching
  - `course_content`: Chunked course material with embeddings
- `search_tools.py` - Tool definitions and execution for Claude's tool use
- `document_processor.py` - Parses course documents, extracts lessons, creates chunks
- `session_manager.py` - Maintains conversation history per user

**Data Models** (`backend/models.py`):
- `Course`: title, instructor, course_link, lessons[]
- `Lesson`: lesson_number, title, lesson_link
- `CourseChunk`: content, course_title, lesson_number, chunk_index

**Configuration** (`backend/config.py`):
- Model: `claude-sonnet-4-20250514`
- Embeddings: `all-MiniLM-L6-v2`
- Chunk size: 800 chars with 100 char overlap
- Max results: 5 chunks per search
- Conversation history: Last 2 exchanges

### Document Format

Course documents in `/docs` must follow this structure:

```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: [lesson title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [lesson title]
Lesson Link: [url]
[lesson content...]
```

The `DocumentProcessor` parses this format to extract:
- Course metadata (stored in `course_catalog` collection)
- Individual lessons with numbers
- Text chunks with course/lesson context prepended

### Startup Behavior

On server startup (`app.py:startup_event`):
1. Loads all `.pdf`, `.docx`, `.txt` files from `../docs`
2. Processes each document into chunks
3. Generates embeddings and stores in ChromaDB
4. Skips documents already in the database (based on course title)

The ChromaDB database persists in `./chroma_db/` directory.

## Development Notes

### Package Management in Development

**Always use `uv` for all Python operations**:
- Adding dependencies: Edit `pyproject.toml` then run `uv sync`
- Running scripts: `uv run python script.py`
- Installing new packages: `uv add package-name`
- Never use `pip install` or bare `python` commands

### Adding New Tools

To add a new tool for Claude to use:

1. Create a class implementing the `Tool` abstract base class in `search_tools.py`
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` with tool logic
4. Register in `rag_system.py` via `tool_manager.register_tool()`

### Modifying Search Behavior

Search logic is in `vector_store.py:search()`:
- Course name resolution uses semantic search on `course_catalog`
- Content search uses ChromaDB's `query()` with metadata filters
- Results are limited by `config.MAX_RESULTS` (default: 5)

### Changing Chunk Strategy

Chunking logic is in `document_processor.py:chunk_text()`:
- Uses sentence-based splitting (regex pattern for sentence boundaries)
- Respects `config.CHUNK_SIZE` and `config.CHUNK_OVERLAP`
- First chunk of each lesson gets context prefix: "Lesson {N} content: ..."

### Session Management

Sessions are ephemeral (in-memory only):
- New session created on first query without session_id
- History limited to `config.MAX_HISTORY * 2` messages (4 messages = 2 exchanges)
- Sessions lost on server restart

## Common Patterns

**Clearing and rebuilding the database**:
```python
# In rag_system.py
rag_system.add_course_folder("../docs", clear_existing=True)
```

**Accessing ChromaDB directly**:
```python
# Vector store exposes ChromaDB collections
vector_store.course_catalog  # Course metadata
vector_store.course_content  # Chunked content
```

**Modifying AI behavior**:
Edit the `SYSTEM_PROMPT` in `ai_generator.py` - this instructs Claude when to use tools and how to format responses.

## Windows Compatibility

Windows users must use Git Bash to run shell scripts like `run.sh`. The application itself is cross-platform compatible.
