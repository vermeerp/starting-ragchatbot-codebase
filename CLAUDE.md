# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

**Start the server:**
```bash
./run.sh
```
Or manually:
```bash
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Access:**
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

**Environment setup:**
Create `.env` in root with:
```
ANTHROPIC_API_KEY=your_key_here
```

## Architecture Overview

This is a RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials using semantic search and Claude AI.

### Data Flow

1. **Document ingestion** (startup):
   - `DocumentProcessor` parses course text files from `/docs` into structured `Course` and `Lesson` objects
   - Text is chunked with overlap (800 chars, 100 char overlap) and enriched with context
   - `VectorStore` stores both course metadata (catalog) and content chunks (content) in separate ChromaDB collections

2. **Query processing** (runtime):
   - User query → `RAGSystem.query()` → `AIGenerator` with tool definitions
   - Claude decides whether to use `CourseSearchTool` based on query type
   - If tool used: `VectorStore.search()` performs semantic search, optionally resolving fuzzy course names
   - Results formatted with course/lesson context and returned to Claude
   - Claude synthesizes final answer using search results + conversation history

3. **Response generation**:
   - `AIGenerator` manages Claude API interaction with strict system prompt
   - Session Manager maintains conversation context (max 2 exchanges)
   - Sources tracked by `CourseSearchTool` and displayed in UI

### Key Components

**RAGSystem** (`rag_system.py`): Central orchestrator
- Initializes all components (processor, vector store, AI generator, session manager, tools)
- `query()`: Main entry point for user questions
- `add_course_folder()`: Batch document ingestion with deduplication

**VectorStore** (`vector_store.py`): ChromaDB wrapper with dual-collection design
- `course_catalog` collection: Stores course metadata for fuzzy course name resolution
- `course_content` collection: Stores text chunks with course/lesson metadata
- `search()`: Unified search interface that handles course name resolution → filtering → semantic search
- Course titles used as unique IDs to prevent duplicates

**DocumentProcessor** (`document_processor.py`): Structured document parser
- Expected format: Header (title, link, instructor) + lessons with numbered sections
- `chunk_text()`: Sentence-aware chunking with configurable overlap
- Adds contextual prefixes to chunks (e.g., "Course X Lesson Y content: ...")

**AIGenerator** (`ai_generator.py`): Claude API client
- System prompt enforces: 1 search max, no meta-commentary, concise responses
- `_handle_tool_execution()`: Manages tool calling workflow
- Temperature=0, max_tokens=800 for consistent, focused answers

**CourseSearchTool** (`search_tools.py`): Tool implementation for Claude
- Defines Anthropic tool schema with query/course_name/lesson_number parameters
- `execute()`: Calls VectorStore.search() and formats results with source tracking
- Results formatted as "[Course - Lesson N]\nContent" blocks

**SessionManager** (`session_manager.py`): Conversation state
- Maintains last N exchanges per session (default 2)
- Provides history as formatted string to AIGenerator

### Document Format

Course text files must follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[lesson content]

Lesson 1: [title]
...
```

## Configuration

All settings in `backend/config.py`:
- `CHUNK_SIZE`: 800 chars (affects retrieval granularity)
- `CHUNK_OVERLAP`: 100 chars (maintains context across chunks)
- `MAX_RESULTS`: 5 search results returned to Claude
- `MAX_HISTORY`: 2 conversation exchanges preserved
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2 (sentence-transformers)
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514

## Development Workflow

**Package management:**
- Always use `uv` for dependency management (NOT pip directly)
- Install dependencies: `uv sync`
- Add new package: `uv add <package-name>`
- Run Python commands: `uv run <command>`

**Running the server during development:**
```bash
cd backend && uv run uvicorn app:app --reload --port 8000
```
The `--reload` flag enables auto-restart on file changes.

**Testing changes:**
1. Make code changes in `/backend`
2. Server auto-reloads (if using `--reload`)
3. Test via web UI at http://localhost:8000 or API docs at http://localhost:8000/docs
4. Frontend changes (HTML/CSS/JS) require browser refresh

**Clearing the vector database:**
Delete `backend/chroma_db/` directory to force re-ingestion of all documents on next startup.

**Debugging:**
- Backend logs appear in terminal running uvicorn
- Frontend errors visible in browser console
- ChromaDB stored in `backend/chroma_db/` (persisted across restarts)

## Adding New Courses

1. Create `.txt` file in `/docs` with proper format
2. Restart server (documents loaded on startup via `app.py:startup_event`)
3. Or call `RAGSystem.add_course_document()` programmatically

System automatically deduplicates by course title.

## Common Modifications

**Change search behavior**: Modify `VectorStore.search()` or `CourseSearchTool.execute()`

**Adjust chunking**: Update `CHUNK_SIZE`/`CHUNK_OVERLAP` in config.py and `DocumentProcessor.chunk_text()`

**Modify AI behavior**: Edit `AIGenerator.SYSTEM_PROMPT` (controls tool usage, response style)

**Add new tools**: Create class inheriting `Tool` ABC, register with `ToolManager` in `RAGSystem.__init__`

**Change vector store**: Collections are hardcoded in `VectorStore.__init__` (catalog + content dual design)