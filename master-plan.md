# Log Analyzer AI Agent — Master Implementation Plan

> **How to use this document:**
> Each phase and subphase is written so you can tell an AI assistant *"execute Phase 0"* or *"execute Phase 1 Subphase 3"* and it has everything it needs — exact commands, exact code, exact file paths, exact test commands and expected output, and a commit message.
>
> Companion files in this folder:
> - `design.md` — *what & why* (architecture, stack, use-cases, WOW-factor)
> - `build-guide.md` — alternate reference with TDD code samples per task
> - **`master-plan.md` (this file)** — *the executable roadmap*

---

## Project summary

An AI agent that takes a **codebase folder + past log files + a design doc**, then — when an operator pastes an **error + log lines** — returns:
1. **Root cause** (why/how it happened)
2. **Ranked suspect files** with line ranges and reasons
3. **Log representation** (template, where to grep, what surrounding lines mean)

Every claim is grounded in real `file:line` or real log lines. Uses the Samsung Gauss Chat API for LLM inference.

**Stack:** Python 3.11 · FastAPI · SQLite · Samsung Gauss Chat API · `all-MiniLM-L6-v2` · Chroma · BM25 · tree-sitter · Drain3 · React + TypeScript + Tailwind

---

## Phase & subphase map

```
Phase 0 — Foundations
  0.1  Hardware & tool check
  0.2  Python environment & deps
  0.3  Samsung Gauss API verification
  0.4  Project folder scaffold
  0.5  Pydantic data contracts
  0.6  Sample data
  0.7  Test infrastructure

Phase 1 — Indexing
  1.1  Code Indexer (tree-sitter chunks + call-site map)
  1.2  Log Indexer (Drain3 templating)
  1.3  Doc Indexer (sliding-window chunks)
  1.4  Vector store + BM25 (Chroma + MiniLM)
  1.5  POST /ingest endpoint
  1.6  Phase 1 integration check

Phase 2 — Analysis Pipeline
  2.1  Error Normalizer (pluggable stack-trace parsers)
  2.2  Hybrid Retriever (BM25 + vector, EnsembleRetriever)
  2.3  Prompt builder
  2.4  Analyzer (ChatOllama + structured output)
  2.5  Grounding Checker
  2.6  Pipeline orchestrator
  2.7  POST /analyze endpoint
  2.8  Phase 2 integration check

Phase 3 — Web App + WOW
  3.1  SQLite persistence (history + feedback)
  3.2  Back-map (log line → source call-site)
  3.3  Additional API endpoints (backmap, incidents, feedback)
  3.4  Frontend scaffold (Vite + React-TS + Tailwind)
  3.5  API client (api.ts)
  3.6  ErrorInput component
  3.7  IncidentReport + SuspectFileCard components
  3.8  LogTimeline WOW component
  3.9  Phase 3 full browser verification

Phase 4 — Eval, Hardening & Demo
  4.1  Labeled eval dataset
  4.2  Eval harness (hit-rate + latency)
  4.3  PII / secret redaction
  4.4  Robustness & edge-case hardening
  4.5  Performance (cache, singleton, persist store)
  4.6  Docker Compose full packaging
  4.7  README & demo script
  4.8  Final demo dry-run

Phase 5 — Stretch (post-internship)
  5.1  Bounded multi-step agent (ReAct, capped)
  5.2  Self-learning incident memory
```

---

## Global constraints (every phase inherits these)

- LLM calls go through the Samsung Gauss Chat API (`POST /openapi/chat/v1/messages`). Keep your credentials in `.env` — never commit them.
- Embeddings model fixed: `sentence-transformers/all-MiniLM-L6-v2` (runs locally).
- MiniLM hard cap ≈ 256 tokens — keep every indexed chunk ≤ ~80 lines.
- All cited `file:line` and log lines must pass the Grounding Checker before surfacing in the UI.
- LangChain = components only (no AgentExecutor, no LangGraph, no long LCEL pipes). Pin all `langchain-*` versions in `requirements.txt`.
- TDD for all backend logic: write failing test → implement → passing test → commit.
- One commit per completed subphase minimum.

---

---

# PHASE 0 — Foundations

**Goal:** Working local environment — Python venv, Samsung Gauss API reachable and responding, project folder created, all Pydantic schemas defined, sample data in place, pytest running green.

**When done:** Samsung Gauss API call returns a valid response; `pytest backend/tests/test_models.py` → PASS.

---

## Phase 0.1 — Hardware & tool check

**Goal:** Confirm the machine can run this project before writing any code.

**Estimated time:** 10 min

### Steps

1. **Check Python version.**
   ```bash
   python --version
   ```
   Expected: `Python 3.11.x` or higher. If lower, install Python 3.11 from python.org.

2. **Check Node.**
   ```bash
   node --version && npm --version
   ```
   Expected: Node 18+ and npm 9+. Install from nodejs.org if missing.

3. **Check Git.**
   ```bash
   git --version
   ```
   Expected: any recent version.

4. **Confirm you have Samsung Gauss API credentials.** You need all four values:
   - `GAUSS_CLIENT_KEY` — the `x-generative-ai-client` header value
   - `GAUSS_API_TOKEN` — the `x-openapi-token` value (include `Bearer ` prefix)
   - `GAUSS_ENDPOINT_URL` — the base endpoint URL from your Open API application
   - `GAUSS_USER_EMAIL` — your portal email for the `x-generative-ai-user-email` header

   If you don't have these yet, apply for the Open API access before continuing.

**Verify:** All checks pass with no errors.

---

## Phase 0.2 — Python environment & dependencies

**Goal:** Isolated venv with all backend packages installed and versions pinned.

**Estimated time:** 20 min

### Steps

1. **Navigate to your project root** (create the folder first if needed).
   ```bash
   mkdir -p ~/Desktop/log-analyzer-agent/backend
   cd ~/Desktop/log-analyzer-agent
   git init
   ```

2. **Create and activate a virtual environment.**
   ```bash
   # Mac/Linux
   python3.11 -m venv .venv
   source .venv/bin/activate

   # Windows (PowerShell)
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   ```
   Expected: your prompt shows `(.venv)`.

3. **Create `requirements.txt`** with these unpinned entries (you will pin after install):
   ```
   fastapi
   uvicorn[standard]
   pydantic>=2
   pydantic-settings
   python-dotenv
   langchain
   langchain-core
   langchain-huggingface
   langchain-chroma
   langchain-community
   chromadb
   sentence-transformers
   rank-bm25
   drain3
   tree-sitter
   tree-sitter-language-pack
   requests
   sseclient-py
   pytest
   httpx
   ```

4. **Install and then immediately pin versions.**
   ```bash
   pip install -r requirements.txt
   pip freeze > requirements.txt
   ```
   This gives you a fully reproducible install. Keep `requirements.txt` in version control.

5. **Create `.env.example`.**
   ```
   GAUSS_ENDPOINT_URL=https://your-endpoint-url-here
   GAUSS_CLIENT_KEY=eyJXXX...
   GAUSS_API_TOKEN=Bearer eyJXXX...
   GAUSS_USER_EMAIL=you@samsung.com
   GAUSS_MODEL_ID=your-text-model-id-here
   EMBED_MODEL=sentence-transformers/all-MiniLM-L6-v2
   CHROMA_DIR=./chroma_store
   TOP_K=6
   ```

**Verify:** `python -c "import langchain_huggingface, chromadb, drain3, tree_sitter, requests, sseclient"` → no errors.

**Commit:**
```bash
git add requirements.txt .env.example
git commit -m "chore: pinned requirements and env template"
```

---

## Phase 0.3 — Samsung Gauss API verification

**Goal:** Confirm your credentials work and you can retrieve a model ID for use in all subsequent phases.

**Estimated time:** 10 min

### Steps

1. **Copy `.env.example` to `.env`** and fill in your real credentials:
   ```bash
   cp .env.example .env
   # edit .env with your actual GAUSS_CLIENT_KEY, GAUSS_API_TOKEN, GAUSS_ENDPOINT_URL, GAUSS_USER_EMAIL
   ```

2. **Fetch available models** to confirm auth works and get your model ID:
   ```python
   # run once from your venv
   import os, requests
   from dotenv import load_dotenv
   load_dotenv()

   headers = {
       "x-generative-ai-client": os.environ["GAUSS_CLIENT_KEY"],
       "x-openapi-token": os.environ["GAUSS_API_TOKEN"],
       "x-generative-ai-user-email": os.environ["GAUSS_USER_EMAIL"],
   }
   resp = requests.get(f"{os.environ['GAUSS_ENDPOINT_URL']}/openapi/chat/v1/models", headers=headers)
   print(resp.json())
   ```
   Expected: a list of model objects each with a `modelId` field. Copy the `modelId` of the text model you want to use and set it as `GAUSS_MODEL_ID` in your `.env`.

3. **Send a test message (non-streaming)** to confirm end-to-end:
   ```python
   body = {
       "modelIds": [os.environ["GAUSS_MODEL_ID"]],
       "contents": ["Reply with only the single word: ok"],
       "isStream": False,
       "llmConfig": {"max_new_tokens": 10, "temperature": 0.1}
   }
   resp = requests.post(
       f"{os.environ['GAUSS_ENDPOINT_URL']}/openapi/chat/v1/messages",
       headers=headers, json=body
   )
   data = resp.json()
   print(data["content"], data["status"])
   ```
   Expected: prints `ok SUCCESS`.

**Verify:** Both API calls succeed and the test message returns `status: SUCCESS`.

**Commit:**
```bash
git add .env.example
git commit -m "chore: Samsung Gauss API verified, env template updated"
```

---

## Phase 0.4 — Project folder scaffold

**Goal:** Full folder structure created (empty files are fine); ready for code.

**Estimated time:** 10 min

### Steps

1. **Create the full directory tree:**
   ```bash
   mkdir -p backend/app/indexing
   mkdir -p backend/app/analysis
   mkdir -p backend/tests
   mkdir -p backend/eval
   mkdir -p sample_data/repo
   mkdir -p sample_data/logs
   mkdir -p frontend/src/components
   ```

2. **Create empty `__init__.py` files** so Python imports work:
   ```bash
   touch backend/app/__init__.py
   touch backend/app/indexing/__init__.py
   touch backend/app/analysis/__init__.py
   touch backend/tests/__init__.py
   ```

3. **Final folder tree should look like:**
   ```
   log-analyzer-agent/
   ├── docker-compose.yml
   ├── requirements.txt
   ├── .env.example
   ├── sample_data/
   │   ├── repo/
   │   ├── logs/
   │   └── design.md          ← create in 0.7
   ├── backend/
   │   ├── app/
   │   │   ├── __init__.py
   │   │   ├── main.py         ← create in 1.5
   │   │   ├── config.py       ← create now
   │   │   ├── models.py       ← create in 0.6
   │   │   ├── db.py           ← create in 3.1
   │   │   ├── indexing/
   │   │   │   ├── __init__.py
   │   │   │   ├── code_indexer.py
   │   │   │   ├── log_indexer.py
   │   │   │   ├── doc_indexer.py
   │   │   │   └── store.py
   │   │   └── analysis/
   │   │       ├── __init__.py
   │   │       ├── normalizer.py
   │   │       ├── retriever.py
   │   │       ├── analyzer.py
   │   │       ├── checker.py
   │   │       ├── backmap.py
   │   │       └── pipeline.py
   │   └── tests/
   │       ├── __init__.py
   │       ├── conftest.py
   │       ├── test_models.py
   │       ├── test_code_indexer.py
   │       ├── test_log_indexer.py
   │       ├── test_normalizer.py
   │       ├── test_retriever.py
   │       ├── test_checker.py
   │       ├── test_backmap.py
   │       └── test_pipeline.py
   └── frontend/
       └── src/components/
   ```

4. **Create `backend/app/config.py`:**
   ```python
   from pydantic_settings import BaseSettings

   class Settings(BaseSettings):
       GAUSS_ENDPOINT_URL: str
       GAUSS_CLIENT_KEY: str
       GAUSS_API_TOKEN: str
       GAUSS_USER_EMAIL: str = ""
       GAUSS_MODEL_ID: str
       EMBED_MODEL: str = "sentence-transformers/all-MiniLM-L6-v2"
       CHROMA_DIR: str = "./chroma_store"
       TOP_K: int = 6

       class Config:
           env_file = ".env"

   settings = Settings()
   ```

**Verify:** `python -c "from app.config import settings; print(settings.GAUSS_MODEL_ID)"` (run from `backend/`) → prints your model ID.

**Commit:**
```bash
git add backend/ sample_data/ frontend/
git commit -m "chore: project folder scaffold and config"
```

---

## Phase 0.5 — Pydantic data contracts

**Goal:** All shared data schemas defined in one file. These are the spine — every module imports from here.

**Estimated time:** 20 min

### Steps

1. **Write the failing test first** (`backend/tests/test_models.py`):
   ```python
   from app.models import (IncidentReport, SuspectFile, LogRepresentation,
                            CodeChunk, CallSite, LogEvent, ErrorSignature, StackFrame)

   def test_incident_report_fields():
       r = IncidentReport(
           root_cause="discount percent exceeded 100",
           suspect_files=[SuspectFile(
               path="buggy_app.py", start_line=5, end_line=8,
               reason="raises ValueError", confidence=0.9)],
           log_representation=LogRepresentation(
               template="Invalid discount percent: <*> for price <*>",
               where_to_grep="Invalid discount percent",
               surrounding_meaning="precedes a 500 response"),
           next_checks=["check percent validation upstream"],
       )
       assert r.suspect_files[0].confidence == 0.9
       assert r.log_representation.where_to_grep == "Invalid discount percent"

   def test_suspect_file_confidence_bounds():
       import pytest
       with pytest.raises(Exception):
           SuspectFile(path="f.py", start_line=1, end_line=2,
                       reason="r", confidence=1.5)   # > 1.0 should fail
   ```

2. **Run it — expect FAIL:**
   ```bash
   cd backend && pytest tests/test_models.py -v
   ```
   Expected output: `ERROR` or `ModuleNotFoundError: No module named 'app.models'`

3. **Implement `backend/app/models.py`:**
   ```python
   from pydantic import BaseModel, Field

   # ── indexing schemas ──────────────────────────────────────────────
   class CodeChunk(BaseModel):
       file: str
       language: str
       symbol: str | None = None
       start_line: int
       end_line: int
       text: str

   class CallSite(BaseModel):
       file: str
       line: int
       text: str           # full source of the logging call
       format_string: str  # the literal/template string passed to logger

   class LogEvent(BaseModel):
       line_no: int
       timestamp: str | None = None
       level: str | None = None
       source: str | None = None
       message: str
       template: str
       template_id: int

   # ── analysis schemas ──────────────────────────────────────────────
   class StackFrame(BaseModel):
       file: str
       line: int
       function: str

   class ErrorSignature(BaseModel):
       exception_type: str | None = None
       message: str
       frames: list[StackFrame] = []
       level: str | None = None
       correlation_ids: list[str] = []
       raw: str

   class SuspectFile(BaseModel):
       path: str
       start_line: int
       end_line: int
       reason: str
       confidence: float = Field(ge=0.0, le=1.0)

   class LogRepresentation(BaseModel):
       template: str
       where_to_grep: str
       surrounding_meaning: str

   class IncidentReport(BaseModel):
       root_cause: str
       suspect_files: list[SuspectFile]
       log_representation: LogRepresentation
       next_checks: list[str] = []
   ```

4. **Run tests — expect PASS:**
   ```bash
   pytest tests/test_models.py -v
   ```
   Expected:
   ```
   tests/test_models.py::test_incident_report_fields PASSED
   tests/test_models.py::test_suspect_file_confidence_bounds PASSED
   2 passed
   ```

**Verify:** Both tests pass.

**Commit:**
```bash
git add backend/app/models.py backend/tests/test_models.py
git commit -m "feat: Pydantic data contracts — all shared schemas"
```

---

## Phase 0.6 — Sample data

**Goal:** A tiny but realistic codebase + log + design doc to drive all tests. Every test references these files.

**Estimated time:** 15 min

### Steps

1. **Create `sample_data/repo/buggy_app.py`:**
   ```python
   import logging

   log = logging.getLogger(__name__)


   def apply_discount(price, percent):
       """Apply a percentage discount to a price."""
       if percent > 100:
           log.error("Invalid discount percent: %s for price %s", percent, price)
           raise ValueError("discount percent cannot exceed 100")
       return price - (price * percent / 100)


   def process_order(order_id, price, discount):
       log.info("Processing order %s: price=%.2f discount=%s", order_id, price, discount)
       try:
           final = apply_discount(price, discount)
           log.info("Order %s completed: final_price=%.2f", order_id, final)
           return final
       except ValueError as e:
           log.error("Order %s failed: %s", order_id, str(e))
           raise
   ```

2. **Create `sample_data/logs/app.log`:**
   ```
   2026-06-20 10:15:00 INFO  app.checkout Processing order abc123: price=99.00 discount=150
   2026-06-20 10:15:00 ERROR app.buggy_app Invalid discount percent: 150 for price 99.0
   2026-06-20 10:15:00 ERROR app.checkout Order abc123 failed: discount percent cannot exceed 100
   2026-06-20 10:15:00 INFO  app.api request_id=abc123 finished with status=500
   2026-06-20 10:16:00 INFO  app.checkout Processing order def456: price=50.00 discount=10
   2026-06-20 10:16:00 INFO  app.checkout Order def456 completed: final_price=45.00
   ```

3. **Create `sample_data/design.md`:**
   ```markdown
   # Order Processing Service — Design

   ## Discount logic
   The `apply_discount(price, percent)` function applies a percentage discount.
   Valid discount values are in the range 0–100 inclusive.
   Any value outside this range is a client error and must raise a ValueError.
   The system must not retry on ValueError; it should return HTTP 400.

   ## Logging policy
   All errors are logged at ERROR level with the order ID and reason.
   Successful completions are logged at INFO level with the final price.
   Log format: `YYYY-MM-DD HH:MM:SS LEVEL source message`
   ```

**Verify:** Files exist and are readable:
```bash
cat sample_data/logs/app.log | wc -l   # should print 6
```

**Commit:**
```bash
git add sample_data/
git commit -m "chore: sample codebase, logs, and design doc for tests"
```

---

## Phase 0.7 — Test infrastructure

**Goal:** `conftest.py` with shared fixtures so every test file can get a built index with one import.

**Estimated time:** 10 min

### Steps

1. **Create `backend/tests/conftest.py`:**
   ```python
   import pytest
   import os

   # Paths relative to the repo root (tests are run from backend/)
   SAMPLE_REPO = os.path.abspath("../sample_data/repo")
   SAMPLE_LOGS = os.path.abspath("../sample_data/logs/app.log")
   SAMPLE_DOC  = os.path.abspath("../sample_data/design.md")

   def pytest_configure(config):
       config.addinivalue_line("markers", "integration: marks tests that call real models")
   ```

2. **Create `backend/pytest.ini`** (so you can run `pytest` from `backend/` without path fuss):
   ```ini
   [pytest]
   testpaths = tests
   python_files = test_*.py
   python_classes = Test*
   python_functions = test_*
   ```

3. **Verify the test suite runs (only `test_models.py` has content yet):**
   ```bash
   cd backend && pytest -v
   ```
   Expected: 2 tests pass, 0 errors.

**Verify:** `pytest` runs cleanly.

**Commit:**
```bash
git add backend/tests/conftest.py backend/pytest.ini
git commit -m "chore: pytest config and shared fixtures"
```

---

### Phase 0 complete ✓

**Milestone check:**
- [ ] Samsung Gauss API test message returns `status: SUCCESS`
- [ ] `cd backend && pytest -v` → 2 tests pass
- [ ] Folder structure matches the tree in Phase 0.4
- [ ] `sample_data/` has all three files

---

---

# PHASE 1 — Indexing

**Goal:** Given a repo folder, a log file, and a design doc, produce a fully searchable library — code function chunks with log call-sites, log event templates, and doc chunks — embedded in Chroma and indexed for BM25.

**When done:** `POST /ingest` on sample data returns non-zero `code_chunks`, `call_sites`, and `log_events`.

---

## Phase 1.1 — Code Indexer

**Goal:** Parse source files via tree-sitter into function-level `CodeChunk`s, and extract every logging call-site into a `CallSite` list (the raw material for the WOW feature).

**Files created:** `backend/app/indexing/code_indexer.py`
**Tests:** `backend/tests/test_code_indexer.py`

### Steps

1. **Write the failing test:**
   ```python
   # backend/tests/test_code_indexer.py
   from tests.conftest import SAMPLE_REPO
   from app.indexing.code_indexer import index_code

   def test_extracts_function_chunks():
       chunks, _ = index_code(SAMPLE_REPO)
       symbols = [c.symbol for c in chunks]
       assert "apply_discount" in symbols
       assert "process_order" in symbols

   def test_all_chunks_have_lines():
       chunks, _ = index_code(SAMPLE_REPO)
       for c in chunks:
           assert c.start_line >= 1
           assert c.end_line >= c.start_line

   def test_extracts_logging_call_sites():
       _, call_sites = index_code(SAMPLE_REPO)
       formats = [cs.format_string for cs in call_sites]
       assert any("Invalid discount percent" in f for f in formats)

   def test_call_site_has_correct_line():
       _, call_sites = index_code(SAMPLE_REPO)
       cs = next(c for c in call_sites if "Invalid discount percent" in c.format_string)
       assert cs.line == 8   # line of log.error in buggy_app.py
   ```

2. **Run — expect FAIL:**
   ```bash
   cd backend && pytest tests/test_code_indexer.py -v
   ```

3. **Implement `backend/app/indexing/code_indexer.py`:**
   ```python
   from pathlib import Path
   from tree_sitter_language_pack import get_parser
   from app.models import CodeChunk, CallSite

   # Map file extension → tree-sitter language name
   EXT_TO_LANG = {
       ".py": "python", ".js": "javascript", ".ts": "typescript",
       ".java": "java", ".go": "go", ".rb": "ruby", ".c": "c", ".cpp": "cpp",
   }

   # Node type that represents a function/method in each language
   DEF_NODES = {
       "python": "function_definition",
       "javascript": "function_declaration",
       "typescript": "function_declaration",
       "java": "method_declaration",
       "go": "function_declaration",
   }

   # Node type that represents a call expression
   CALL_NODES = {
       "python": "call",
       "javascript": "call_expression",
       "typescript": "call_expression",
       "java": "method_invocation",
       "go": "call_expression",
   }

   LOG_HINTS = ("log", "logger", "logging")


   def _text(node, src: bytes) -> str:
       return src[node.start_byte:node.end_byte].decode("utf8", "replace")


   def _walk(node):
       yield node
       for child in node.children:
           yield from _walk(child)


   def _first_string(node, src: bytes) -> str | None:
       for n in _walk(node):
           if "string" in n.type:
               raw = _text(n, src).strip()
               return raw.strip("\"'`")
       return None


   def _fallback_chunks(path: Path, src: bytes) -> list[CodeChunk]:
       """Line-window fallback for languages without a grammar."""
       lines = src.decode("utf8", "replace").splitlines()
       window, overlap = 60, 10
       chunks = []
       i = 0
       while i < len(lines):
           block = lines[i:i + window]
           chunks.append(CodeChunk(
               file=str(path), language="unknown", symbol=None,
               start_line=i + 1, end_line=i + len(block),
               text="\n".join(block),
           ))
           i += window - overlap
       return chunks


   def index_code(root_dir: str) -> tuple[list[CodeChunk], list[CallSite]]:
       chunks: list[CodeChunk] = []
       call_sites: list[CallSite] = []

       for path in sorted(Path(root_dir).rglob("*")):
           if not path.is_file():
               continue
           lang = EXT_TO_LANG.get(path.suffix)
           if not lang:
               continue

           src = path.read_bytes()

           try:
               parser = get_parser(lang)
           except Exception:
               chunks.extend(_fallback_chunks(path, src))
               continue

           tree = parser.parse(src)
           def_type = DEF_NODES.get(lang)
           call_type = CALL_NODES.get(lang)

           for node in _walk(tree.root_node):
               # function/method definition → CodeChunk
               if def_type and node.type == def_type:
                   name_node = next(
                       (c for c in node.children if c.type == "identifier"), None)
                   chunks.append(CodeChunk(
                       file=str(path),
                       language=lang,
                       symbol=_text(name_node, src) if name_node else None,
                       start_line=node.start_point[0] + 1,   # tree-sitter is 0-based
                       end_line=node.end_point[0] + 1,
                       text=_text(node, src)[:4000],          # guard oversized functions
                   ))

               # logging call → CallSite
               if call_type and node.type == call_type:
                   call_text = _text(node, src)
                   callee = call_text.split("(")[0].lower()
                   if any(h in callee for h in LOG_HINTS):
                       fmt = _first_string(node, src)
                       if fmt:
                           call_sites.append(CallSite(
                               file=str(path),
                               line=node.start_point[0] + 1,
                               text=call_text[:500],
                               format_string=fmt,
                           ))

       return chunks, call_sites
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_code_indexer.py -v
   ```
   Expected: 4 tests PASS.
   > If `test_call_site_has_correct_line` fails with line 9 instead of 8, open `buggy_app.py` and count the actual line; update the test to match. tree-sitter counts correctly, but your file might have drifted.

**Commit:**
```bash
git add backend/app/indexing/code_indexer.py backend/tests/test_code_indexer.py
git commit -m "feat: tree-sitter code indexer with function chunks and log call-site extraction"
```

---

## Phase 1.2 — Log Indexer

**Goal:** Parse a log file with Drain3; cluster lines into templates; emit `LogEvent`s with structured fields.

**Files created:** `backend/app/indexing/log_indexer.py`
**Tests:** `backend/tests/test_log_indexer.py`

### Steps

1. **Write the failing test:**
   ```python
   # backend/tests/test_log_indexer.py
   from tests.conftest import SAMPLE_LOGS
   from app.indexing.log_indexer import index_logs

   def test_returns_all_lines():
       events = index_logs(SAMPLE_LOGS)
       assert len(events) == 6   # sample log has 6 lines

   def test_parses_level_and_message():
       events = index_logs(SAMPLE_LOGS)
       err_events = [e for e in events if e.level == "ERROR"]
       assert len(err_events) == 2

   def test_drain_produces_template_with_wildcard():
       events = index_logs(SAMPLE_LOGS)
       err = next(e for e in events if "Invalid discount percent" in e.message)
       assert "<*>" in err.template   # Drain3 replaces variable parts

   def test_template_id_is_non_negative():
       events = index_logs(SAMPLE_LOGS)
       assert all(e.template_id >= 0 for e in events)
   ```

2. **Run — expect FAIL:**
   ```bash
   pytest tests/test_log_indexer.py -v
   ```

3. **Implement `backend/app/indexing/log_indexer.py`:**
   ```python
   import re
   from drain3 import TemplateMiner
   from app.models import LogEvent

   # Matches: 2026-06-20 10:15:00 ERROR app.module message here
   LINE_RE = re.compile(
       r"^(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+"
       r"(?P<level>\w+)\s+(?P<source>\S+)\s+(?P<msg>.+)$"
   )


   def index_logs(log_path: str) -> list[LogEvent]:
       miner = TemplateMiner()
       events: list[LogEvent] = []

       with open(log_path, encoding="utf8", errors="replace") as fh:
           for i, raw_line in enumerate(fh, start=1):
               line = raw_line.rstrip("\n")
               if not line.strip():
                   continue

               m = LINE_RE.match(line)
               message = m.group("msg") if m else line

               result = miner.add_log_message(message)

               events.append(LogEvent(
                   line_no=i,
                   timestamp=m.group("ts") if m else None,
                   level=m.group("level") if m else None,
                   source=m.group("source") if m else None,
                   message=message,
                   template=result["template_mined"],
                   template_id=result["cluster_id"],
               ))

       return events
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_log_indexer.py -v
   ```

**Commit:**
```bash
git add backend/app/indexing/log_indexer.py backend/tests/test_log_indexer.py
git commit -m "feat: Drain3 log indexer with template mining"
```

---

## Phase 1.3 — Doc Indexer

**Goal:** Split a design doc into small overlapping text chunks that fit within MiniLM's 256-token cap.

**Files created:** `backend/app/indexing/doc_indexer.py`

### Steps

1. **Implement `backend/app/indexing/doc_indexer.py`** (simple, no separate test needed — covered by the store test in 1.4):
   ```python
   from pathlib import Path
   from langchain_core.documents import Document


   def index_doc(doc_path: str, window_words: int = 100, overlap_words: int = 20) -> list[Document]:
       """
       Split a plain-text or markdown doc into overlapping word-windows.
       window_words=100 ≈ 130 tokens — safely under MiniLM's 256-token cap.
       """
       text = Path(doc_path).read_text(encoding="utf8", errors="replace")
       words = text.split()
       docs: list[Document] = []
       i = 0
       while i < len(words):
           chunk_words = words[i:i + window_words]
           docs.append(Document(
               page_content=" ".join(chunk_words),
               metadata={
                   "source_type": "doc",
                   "file": doc_path,
                   "start_word": i,
               },
           ))
           i += window_words - overlap_words
       return docs
   ```

**Commit:**
```bash
git add backend/app/indexing/doc_indexer.py
git commit -m "feat: doc indexer with overlapping word-window chunks"
```

---

## Phase 1.4 — Vector store + BM25 (Chroma + MiniLM)

**Goal:** Combine all three index types into a single `Index` dataclass; persist chunks into Chroma + build BM25.

**Files created:** `backend/app/indexing/store.py`

### Steps

1. **Implement `backend/app/indexing/store.py`:**
   ```python
   from dataclasses import dataclass, field
   from langchain_core.documents import Document
   from langchain_huggingface import HuggingFaceEmbeddings
   from langchain_chroma import Chroma
   from app.config import settings
   from app.indexing.code_indexer import index_code
   from app.indexing.log_indexer import index_logs
   from app.indexing.doc_indexer import index_doc
   from app.models import CallSite, LogEvent

   _EMBED_SINGLETON: HuggingFaceEmbeddings | None = None


   def _get_embeddings() -> HuggingFaceEmbeddings:
       global _EMBED_SINGLETON
       if _EMBED_SINGLETON is None:
           _EMBED_SINGLETON = HuggingFaceEmbeddings(
               model_name=settings.EMBED_MODEL,
               model_kwargs={"device": "cpu"},
               encode_kwargs={"normalize_embeddings": True},
           )
       return _EMBED_SINGLETON


   @dataclass
   class Index:
       store: Chroma
       all_docs: list[Document]
       call_sites: list[CallSite]
       log_events: list[LogEvent]
       # convenience view per type
       code_docs: list[Document] = field(default_factory=list)
       log_docs: list[Document] = field(default_factory=list)
       doc_docs: list[Document] = field(default_factory=list)


   def build_index(repo: str, logs: str, doc: str) -> Index:
       # 1. parse raw sources
       chunks, call_sites = index_code(repo)
       events = index_logs(logs)
       design_docs = index_doc(doc)

       # 2. convert to LangChain Documents with metadata
       code_docs: list[Document] = [
           Document(
               page_content=c.text,
               metadata={
                   "source_type": "code",
                   "file": c.file,
                   "symbol": c.symbol or "",
                   "start_line": c.start_line,
                   "end_line": c.end_line,
                   "language": c.language,
               },
           )
           for c in chunks
       ]

       log_docs: list[Document] = [
           Document(
               page_content=e.message,
               metadata={
                   "source_type": "log",
                   "line_no": e.line_no,
                   "level": e.level or "",
                   "template": e.template,
                   "template_id": e.template_id,
               },
           )
           for e in events
       ]

       all_docs = code_docs + log_docs + design_docs

       # 3. embed into Chroma
       store = Chroma.from_documents(
           all_docs,
           _get_embeddings(),
           persist_directory=settings.CHROMA_DIR,
       )

       return Index(
           store=store,
           all_docs=all_docs,
           call_sites=call_sites,
           log_events=events,
           code_docs=code_docs,
           log_docs=log_docs,
           doc_docs=design_docs,
       )
   ```

2. **Smoke test (manual — takes ~60 sec first run while MiniLM embeds):**
   ```bash
   cd backend && python -c "
   from app.indexing.store import build_index
   idx = build_index('../sample_data/repo', '../sample_data/logs/app.log', '../sample_data/design.md')
   print('code_docs:', len(idx.code_docs))
   print('log_docs:', len(idx.log_docs))
   print('call_sites:', len(idx.call_sites))
   "
   ```
   Expected: all three counts > 0.

**Commit:**
```bash
git add backend/app/indexing/store.py
git commit -m "feat: Chroma+MiniLM store builder with Index dataclass"
```

---

## Phase 1.5 — POST /ingest endpoint

**Goal:** A running FastAPI server that accepts folder paths, builds the index, and holds it in app state.

**Files created:** `backend/app/main.py`

### Steps

1. **Implement `backend/app/main.py`:**
   ```python
   from fastapi import FastAPI
   from fastapi.middleware.cors import CORSMiddleware
   from pydantic import BaseModel
   from app.indexing.store import build_index

   app = FastAPI(title="Log Analyzer Agent")

   app.add_middleware(
       CORSMiddleware,
       allow_origins=["*"],   # tighten for production
       allow_methods=["*"],
       allow_headers=["*"],
   )

   # In-memory state — holds the last built index
   STATE: dict = {}


   class IngestRequest(BaseModel):
       repo: str
       logs: str
       doc: str


   @app.post("/ingest")
   def ingest(req: IngestRequest):
       idx = build_index(req.repo, req.logs, req.doc)
       STATE["index"] = idx
       return {
           "status": "ok",
           "code_chunks": len(idx.code_docs),
           "log_events": len(idx.log_events),
           "call_sites": len(idx.call_sites),
           "doc_chunks": len(idx.doc_docs),
       }


   @app.get("/health")
   def health():
       return {"status": "ok", "indexed": "index" in STATE}
   ```

2. **Start the server:**
   ```bash
   cd backend && uvicorn app.main:app --reload --port 8080
   ```

3. **Verify via curl** (in a separate terminal):
   ```bash
   curl -X POST http://localhost:8080/ingest \
     -H "Content-Type: application/json" \
     -d '{
       "repo": "../sample_data/repo",
       "logs": "../sample_data/logs/app.log",
       "doc":  "../sample_data/design.md"
     }'
   ```
   Expected response:
   ```json
   {"status":"ok","code_chunks":2,"log_events":6,"call_sites":3,"doc_chunks":2}
   ```
   *(Exact numbers depend on your sample data size.)*

**Commit:**
```bash
git add backend/app/main.py
git commit -m "feat: FastAPI app with POST /ingest and /health"
```

---

## Phase 1.6 — Phase 1 integration check

**Goal:** Confirm the whole indexing stack works end-to-end before moving to analysis.

### Steps

1. **Run the full backend test suite:**
   ```bash
   cd backend && pytest -v --ignore=tests/test_pipeline.py
   ```
   Expected: all non-integration tests pass (models + code_indexer + log_indexer).

2. **Manual full-stack check:**
   ```bash
   # 1. start api
   uvicorn app.main:app --port 8080 &

   # 2. ingest sample data
   curl -s -X POST http://localhost:8080/ingest \
     -H "Content-Type: application/json" \
     -d '{"repo":"../sample_data/repo","logs":"../sample_data/logs/app.log","doc":"../sample_data/design.md"}' | python -m json.tool

   # 3. confirm health
   curl http://localhost:8080/health
   ```
   Expected: health shows `"indexed": true`.

---

### Phase 1 complete ✓

**Milestone check:**
- [ ] `POST /ingest` returns non-zero counts for code, logs, call_sites
- [ ] All pytest tests pass (no integration tests yet)
- [ ] `GET /health` returns `"indexed": true` after ingest

---

---

# PHASE 2 — Analysis Pipeline

**Goal:** Given an error text, produce a grounded, Pydantic-validated `IncidentReport` containing root cause, ranked suspect files, and log representation — with every citation verified.

**When done:** `POST /analyze` with the sample error returns a report; all five analysis modules have passing tests.

---

## Phase 2.1 — Error Normalizer

**Goal:** Parse raw error text into a structured `ErrorSignature`. Pluggable: one parser per language, registered by key.

**Files created:** `backend/app/analysis/normalizer.py`
**Tests:** `backend/tests/test_normalizer.py`

### Steps

1. **Write the failing test:**
   ```python
   # backend/tests/test_normalizer.py
   from app.analysis.normalizer import normalize

   PY_TRACE = """\
   Traceback (most recent call last):
     File "buggy_app.py", line 9, in apply_discount
       raise ValueError("discount percent cannot exceed 100")
   ValueError: discount percent cannot exceed 100"""

   def test_parses_python_traceback():
       sig = normalize(PY_TRACE)
       assert sig.exception_type == "ValueError"
       assert sig.message == "discount percent cannot exceed 100"
       assert sig.frames[-1].file == "buggy_app.py"
       assert sig.frames[-1].line == 9
       assert sig.frames[-1].function == "apply_discount"

   def test_generic_fallback_for_unknown_format():
       sig = normalize("SomeError: something went wrong")
       assert sig.message == "SomeError: something went wrong"
       assert sig.frames == []

   def test_extracts_correlation_id():
       sig = normalize("Error: boom  request_id=abc123")
       assert "abc123" in sig.correlation_ids
   ```

2. **Run — expect FAIL:**
   ```bash
   pytest tests/test_normalizer.py -v
   ```

3. **Implement `backend/app/analysis/normalizer.py`:**
   ```python
   import re
   from app.models import ErrorSignature, StackFrame

   _PY_FRAME = re.compile(
       r'File "(?P<file>[^"]+)", line (?P<line>\d+), in (?P<func>\S+)'
   )
   _PY_EXC = re.compile(
       r"^(?P<type>[\w.]+(?:Error|Exception|Warning|Fault|Panic)): (?P<msg>.+)$",
       re.MULTILINE,
   )
   _CORR_ID = re.compile(
       r"(?:request_id|correlation_id|trace_id|span_id)[=:]\s*(\w[\w-]*)"
   )

   # ── parsers ────────────────────────────────────────────────────────

   def _parse_python(raw: str) -> ErrorSignature:
       frames = [
           StackFrame(
               file=m.group("file"),
               line=int(m.group("line")),
               function=m.group("func"),
           )
           for m in _PY_FRAME.finditer(raw)
       ]
       exc = _PY_EXC.search(raw)
       return ErrorSignature(
           exception_type=exc.group("type") if exc else None,
           message=exc.group("msg") if exc else raw.strip().splitlines()[-1],
           frames=frames,
           correlation_ids=_CORR_ID.findall(raw),
           raw=raw,
       )

   # Add Java / JS / Go parsers here following the same pattern
   # and register them in PARSERS below.

   PARSERS: dict[str, callable] = {
       "python": _parse_python,
   }

   # ── public API ─────────────────────────────────────────────────────

   def normalize(raw_error: str) -> ErrorSignature:
       if "Traceback (most recent call last)" in raw_error:
           return PARSERS["python"](raw_error)

       # Generic fallback: no frames, keep last line as message
       last_line = raw_error.strip().splitlines()[-1]
       return ErrorSignature(
           message=last_line,
           correlation_ids=_CORR_ID.findall(raw_error),
           raw=raw_error,
       )
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_normalizer.py -v
   ```

**Commit:**
```bash
git add backend/app/analysis/normalizer.py backend/tests/test_normalizer.py
git commit -m "feat: error normalizer with pluggable stack-trace parsers"
```

---

## Phase 2.2 — Hybrid Retriever

**Goal:** Given an `ErrorSignature`, retrieve the most relevant code chunks, log events, and doc chunks using both keyword (BM25) and semantic (vector) search, fused with Reciprocal Rank Fusion.

**Files created:** `backend/app/analysis/retriever.py`
**Tests:** `backend/tests/test_retriever.py`

### Steps

1. **Write the failing test:**
   ```python
   # backend/tests/test_retriever.py
   import pytest
   from tests.conftest import SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC
   from app.indexing.store import build_index
   from app.analysis.normalizer import normalize
   from app.analysis.retriever import retrieve

   @pytest.fixture(scope="module")
   def built_index():
       return build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)

   @pytest.mark.integration
   def test_retrieves_apply_discount_for_valueerror(built_index):
       sig = normalize("ValueError: discount percent cannot exceed 100")
       hits = retrieve(built_index, sig)
       code_text = " ".join(d.page_content for d in hits["code"])
       assert "apply_discount" in code_text

   @pytest.mark.integration
   def test_returns_all_three_keys(built_index):
       sig = normalize("ValueError: discount percent cannot exceed 100")
       hits = retrieve(built_index, sig)
       assert set(hits.keys()) == {"code", "log", "doc"}

   @pytest.mark.integration
   def test_respects_top_k(built_index):
       sig = normalize("ValueError: discount percent cannot exceed 100")
       hits = retrieve(built_index, sig, k=3)
       assert len(hits["code"]) <= 3
   ```

2. **Run — expect FAIL:**
   ```bash
   pytest tests/test_retriever.py -v -m integration
   ```
   *(This calls the real index — it's slow but required.)*

3. **Implement `backend/app/analysis/retriever.py`:**

   > **Import note:** Try `from langchain.retrievers import EnsembleRetriever` first. If that fails, try `from langchain_community.retrievers import EnsembleRetriever`. If both fail, use the built-in RRF fallback function below — it produces identical output.

   ```python
   from langchain_community.retrievers import BM25Retriever
   from langchain_core.documents import Document
   from app.config import settings
   from app.models import ErrorSignature

   def _query_from_sig(sig: ErrorSignature) -> str:
       frame_text = " ".join(
           f"{f.file}:{f.line} {f.function}" for f in sig.frames
       )
       return f"{sig.exception_type or ''} {sig.message} {frame_text}".strip()


   def _docs_by_type(index, source_type: str) -> list[Document]:
       return [d for d in index.all_docs
               if d.metadata.get("source_type") == source_type]


   def _rrf_fuse(ranked_lists: list[list[Document]], k: int, rrf_k: int = 60) -> list[Document]:
       """Reciprocal Rank Fusion — merges multiple ranked lists."""
       scores: dict[str, float] = {}
       by_content: dict[str, Document] = {}
       for ranked in ranked_lists:
           for rank, doc in enumerate(ranked):
               key = doc.page_content
               scores[key] = scores.get(key, 0.0) + 1.0 / (rrf_k + rank + 1)
               by_content[key] = doc
       top = sorted(scores, key=scores.__getitem__, reverse=True)
       return [by_content[key] for key in top[:k]]


   def _hybrid_search(index, source_type: str, query: str, k: int) -> list[Document]:
       docs = _docs_by_type(index, source_type)
       if not docs:
           return []

       # BM25 (keyword)
       bm25 = BM25Retriever.from_documents(docs)
       bm25.k = k
       bm25_results = bm25.invoke(query)

       # Dense vector
       vec_results = index.store.similarity_search(
           query, k=k,
           filter={"source_type": source_type},
       )

       return _rrf_fuse([bm25_results, vec_results], k=k)


   def retrieve(index, sig: ErrorSignature, k: int = settings.TOP_K) -> dict[str, list[Document]]:
       query = _query_from_sig(sig)

       # Always include exact stack-frame files (deterministic pull)
       frame_files = {f.file for f in sig.frames}
       exact_code = [
           d for d in index.code_docs
           if any(d.metadata.get("file", "").endswith(ff) or ff.endswith(d.metadata.get("file", ""))
                  for ff in frame_files)
       ]

       semantic_code = _hybrid_search(index, "code", query, k)

       # Merge: exact first, then semantic (deduplicated by content)
       seen: set[str] = set()
       merged_code: list[Document] = []
       for doc in exact_code + semantic_code:
           if doc.page_content not in seen:
               merged_code.append(doc)
               seen.add(doc.page_content)

       return {
           "code": merged_code[:k],
           "log":  _hybrid_search(index, "log", query, k),
           "doc":  _hybrid_search(index, "doc", query, k),
       }
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_retriever.py -v -m integration
   ```

**Commit:**
```bash
git add backend/app/analysis/retriever.py backend/tests/test_retriever.py
git commit -m "feat: hybrid retriever (BM25 + vector RRF) across code/log/doc"
```

---

## Phase 2.3 — Prompt builder

**Goal:** A pure Python function that assembles the retrieval results into a structured prompt for Qwen3. Tested without the model.

**Files modified:** `backend/app/analysis/analyzer.py` (partial — just `build_prompt`)
**Tests:** `backend/tests/test_analyzer.py` (prompt section only)

### Steps

1. **Write the failing test (prompt only — no LLM):**
   ```python
   # backend/tests/test_analyzer.py
   from langchain_core.documents import Document
   from app.analysis.normalizer import normalize
   from app.analysis.analyzer import build_prompt

   def _fake_hits():
       code_doc = Document(
           page_content="def apply_discount(price, percent):\n    raise ValueError(...)",
           metadata={"source_type":"code","file":"buggy_app.py","start_line":5,"end_line":9},
       )
       return {"code": [code_doc], "log": [], "doc": []}

   def test_prompt_contains_error_text():
       sig = normalize("ValueError: discount percent cannot exceed 100")
       p = build_prompt(sig, _fake_hits())
       assert "discount percent cannot exceed 100" in p

   def test_prompt_contains_file_reference():
       sig = normalize("ValueError: discount percent cannot exceed 100")
       p = build_prompt(sig, _fake_hits())
       assert "buggy_app.py" in p

   def test_prompt_requests_json():
       sig = normalize("ValueError: discount percent cannot exceed 100")
       p = build_prompt(sig, _fake_hits())
       assert "JSON" in p or "json" in p
   ```

2. **Run — expect FAIL.**

3. **Create `backend/app/analysis/analyzer.py`** (prompt part only for now):
   ```python
   from langchain_core.documents import Document
   from app.models import ErrorSignature

   _SYSTEM_PROMPT = """\
   You are a log-analysis assistant for software operators.
   Rules:
   - Use ONLY the provided code, log, and design-doc context below.
   - Cite real file:line ranges and real log line numbers.
   - If you are unsure, say so — never invent file names or line numbers.
   - Respond strictly as JSON matching the schema described in the TASK section.
   """

   def _format_docs(docs: list[Document], label: str) -> str:
       if not docs:
           return f"(no {label} context available)"
       parts = []
       for d in docs:
           m = d.metadata
           if m.get("source_type") == "code":
               loc = f"{m.get('file','?')}:{m.get('start_line','?')}-{m.get('end_line','?')}"
           elif m.get("source_type") == "log":
               loc = f"log line {m.get('line_no','?')}"
           else:
               loc = m.get("file", "doc")
           parts.append(f"[{label} | {loc}]\n{d.page_content}")
       return "\n\n".join(parts)


   def build_prompt(sig: ErrorSignature, hits: dict[str, list[Document]]) -> str:
       return f"""{_SYSTEM_PROMPT}

   ## ERROR INPUT
   {sig.raw}

   ## CODE CONTEXT
   {_format_docs(hits["code"], "code")}

   ## LOG CONTEXT
   {_format_docs(hits["log"], "log")}

   ## DESIGN DOC CONTEXT
   {_format_docs(hits["doc"], "doc")}

   ## TASK
   Return a JSON object with exactly these keys:
   - root_cause (string): why and how the error occurred
   - suspect_files (array): each item has path (string), start_line (int), end_line (int), reason (string), confidence (float 0-1)
   - log_representation: object with template (string), where_to_grep (string), surrounding_meaning (string)
   - next_checks (array of strings): follow-up steps for the operator
   """
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_analyzer.py -v
   ```

**Commit:**
```bash
git add backend/app/analysis/analyzer.py backend/tests/test_analyzer.py
git commit -m "feat: grounded prompt builder for Qwen3"
```

---

## Phase 2.4 — Analyzer (Samsung Gauss API + structured output)

**Goal:** Add the live LLM call to `analyzer.py` using the Samsung Gauss Chat API, parsing the response into a validated `IncidentReport`.

**Files modified:** `backend/app/analysis/analyzer.py` (add `analyze` function)

### Steps

1. **Add `analyze()` to `analyzer.py`:**
   ```python
   # Add these imports at the top of analyzer.py
   import json
   import re
   import time
   import requests
   from app.config import settings
   from app.models import IncidentReport

   def _gauss_headers() -> dict:
       return {
           "x-generative-ai-client": settings.GAUSS_CLIENT_KEY,
           "x-openapi-token": settings.GAUSS_API_TOKEN,
           "x-generative-ai-user-email": settings.GAUSS_USER_EMAIL,
           "Content-Type": "application/json",
       }

   def _call_gauss(prompt_text: str) -> str:
       """
       Call the Samsung Gauss Chat API (non-streaming) and return the content string.
       Raises RuntimeError on non-SUCCESS status.
       """
       body = {
           "modelIds": [settings.GAUSS_MODEL_ID],
           "contents": [prompt_text],
           "isStream": False,
           "llmConfig": {
               "max_new_tokens": 1024,
               "temperature": 0.1,
               "top_p": 0.9,
               "repetition_penalty": 1.04,
           },
       }
       resp = requests.post(
           f"{settings.GAUSS_ENDPOINT_URL}/openapi/chat/v1/messages",
           headers=_gauss_headers(),
           json=body,
           timeout=120,
       )
       resp.raise_for_status()
       data = resp.json()
       if data.get("status") != "SUCCESS":
           raise RuntimeError(f"Gauss API error: {data.get('status')} / {data.get('responseCode')}")
       return data["content"]

   def _extract_json(text: str) -> dict:
       """Strip markdown fences and parse JSON from the model's response."""
       # Remove ```json ... ``` fences if present
       clean = re.sub(r"```(?:json)?", "", text).strip().strip("`").strip()
       return json.loads(clean)

   def analyze(sig: ErrorSignature, hits: dict[str, list[Document]]) -> IncidentReport:
       """
       Call Samsung Gauss with the grounded prompt and return a validated IncidentReport.
       Retries up to 3 times on JSON parse or validation errors.
       """
       prompt_text = build_prompt(sig, hits)
       last_exc = None
       for attempt in range(3):
           try:
               raw = _call_gauss(prompt_text)
               parsed = _extract_json(raw)
               return IncidentReport(**parsed)
           except Exception as e:
               last_exc = e
               if attempt < 2:
                   time.sleep(2)
       raise RuntimeError(f"Gauss analyzer failed after 3 attempts: {last_exc}") from last_exc
   ```

2. **Add a live integration test at the bottom of `test_analyzer.py`:**
   ```python
   import pytest
   from tests.conftest import SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC

   @pytest.mark.integration
   def test_live_analyze_returns_incident_report():
       from app.indexing.store import build_index
       from app.analysis.retriever import retrieve
       from app.analysis.analyzer import analyze

       idx = build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)
       sig = normalize("ValueError: discount percent cannot exceed 100")
       hits = retrieve(idx, sig)
       report = analyze(sig, hits)

       assert report.root_cause
       assert len(report.suspect_files) >= 1
       assert report.log_representation.where_to_grep
   ```

3. **Run the integration test (needs valid Gauss credentials in `.env`):**
   ```bash
   pytest tests/test_analyzer.py -v -m integration
   ```
   Expected: PASS — takes a few seconds for the API round-trip.
   > If you get a `RuntimeError` with `FILTER_INVALID`: the content filter blocked the request. Rephrase the system prompt to be clearly operational/technical. If JSON parse fails, check that `build_prompt` explicitly asks for a raw JSON object (no markdown fences).

**Commit:**
```bash
git add backend/app/analysis/analyzer.py backend/tests/test_analyzer.py
git commit -m "feat: Samsung Gauss API analyzer with JSON parsing and retry"
```

---

## Phase 2.5 — Grounding Checker

**Goal:** Verify every `SuspectFile` citation against the index. Penalize anything unverifiable so the operator can trust what's shown.

**Files created:** `backend/app/analysis/checker.py`
**Tests:** `backend/tests/test_checker.py`

### Steps

1. **Write the failing test:**
   ```python
   # backend/tests/test_checker.py
   from app.analysis.checker import check
   from app.models import IncidentReport, SuspectFile, LogRepresentation

   class _FakeIndex:
       all_docs = [
           type("D", (), {"metadata": {
               "source_type": "code",
               "file": "/full/path/buggy_app.py",
               "start_line": 1, "end_line": 20,
           }})()
       ]

   def _make_report(path, start, end, confidence=0.9):
       return IncidentReport(
           root_cause="test",
           suspect_files=[SuspectFile(path=path, start_line=start, end_line=end,
                                      reason="test", confidence=confidence)],
           log_representation=LogRepresentation(
               template="t", where_to_grep="g", surrounding_meaning="s"),
       )

   def test_verified_file_keeps_confidence():
       report, warns = check(_make_report("buggy_app.py", 5, 8), _FakeIndex())
       assert report.suspect_files[0].confidence == 0.9
       assert warns == []

   def test_nonexistent_file_is_penalized():
       report, warns = check(_make_report("ghost_file.py", 1, 5), _FakeIndex())
       assert report.suspect_files[0].confidence < 0.9
       assert len(warns) > 0

   def test_unverified_reason_is_prefixed():
       report, warns = check(_make_report("ghost_file.py", 1, 5), _FakeIndex())
       assert "[UNVERIFIED]" in report.suspect_files[0].reason
   ```

2. **Run — expect FAIL.**

3. **Implement `backend/app/analysis/checker.py`:**
   ```python
   from app.models import IncidentReport


   def _build_known_ranges(index) -> dict[str, list[tuple[int, int]]]:
       """Map file path (full or basename) → list of (start, end) line ranges."""
       ranges: dict[str, list[tuple[int, int]]] = {}
       for d in index.all_docs:
           m = d.metadata
           if m.get("source_type") != "code":
               continue
           f = m["file"]
           ranges.setdefault(f, []).append((m["start_line"], m["end_line"]))
       return ranges


   def _path_matches(cited: str, known: str) -> bool:
       """True if either path is a suffix of the other."""
       return known.endswith(cited) or cited.endswith(known)


   def check(report: IncidentReport, index) -> tuple[IncidentReport, list[str]]:
       known = _build_known_ranges(index)
       warnings: list[str] = []

       for sf in report.suspect_files:
           matched = next(
               (kpath for kpath in known if _path_matches(sf.path, kpath)),
               None,
           )
           if matched is None:
               warnings.append(f"unverifiable file citation: {sf.path}")
               sf.confidence = round(sf.confidence * 0.5, 3)
               sf.reason = f"[UNVERIFIED] {sf.reason}"

       return report, warnings
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_checker.py -v
   ```

**Commit:**
```bash
git add backend/app/analysis/checker.py backend/tests/test_checker.py
git commit -m "feat: grounding checker penalizes unverifiable citations"
```

---

## Phase 2.6 — Pipeline orchestrator

**Goal:** One function that chains all five steps. The whole "agent brain" is five readable lines.

**Files created:** `backend/app/analysis/pipeline.py`
**Tests:** `backend/tests/test_pipeline.py`

### Steps

1. **Implement `backend/app/analysis/pipeline.py`:**
   ```python
   from app.analysis.normalizer import normalize
   from app.analysis.retriever import retrieve
   from app.analysis.analyzer import analyze
   from app.analysis.checker import check
   from app.config import settings


   def run_analysis(index, raw_error: str) -> dict:
       """
       The full pipeline: 5 steps, plain Python, no framework.
       Returns a dict ready to JSON-serialize over the API.
       """
       sig = normalize(raw_error)
       hits = retrieve(index, sig, k=settings.TOP_K)
       report = analyze(sig, hits)
       report, warnings = check(report, index)
       return {
           "report": report.model_dump(),
           "warnings": warnings,
       }
   ```

2. **Write the integration test:**
   ```python
   # backend/tests/test_pipeline.py
   import pytest
   from tests.conftest import SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC

   @pytest.mark.integration
   def test_full_pipeline_returns_suspect_files():
       from app.indexing.store import build_index
       from app.analysis.pipeline import run_analysis

       idx = build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)
       result = run_analysis(idx, "ValueError: discount percent cannot exceed 100")

       assert "report" in result
       assert result["report"]["suspect_files"]
       assert result["report"]["root_cause"]
       assert result["report"]["log_representation"]
   ```

3. **Run — expect PASS:**
   ```bash
   pytest tests/test_pipeline.py -v -m integration
   ```

**Commit:**
```bash
git add backend/app/analysis/pipeline.py backend/tests/test_pipeline.py
git commit -m "feat: 5-step analysis pipeline orchestrator"
```

---

## Phase 2.7 — POST /analyze endpoint

**Goal:** Expose the pipeline via the API; save each result to history.

**Files modified:** `backend/app/main.py`

### Steps

1. **Add to `main.py`:**
   ```python
   # Add at the top
   from app.analysis.pipeline import run_analysis

   # Add a new request model
   class AnalyzeRequest(BaseModel):
       error: str

   # Add the route
   @app.post("/analyze")
   def analyze_route(req: AnalyzeRequest):
       if "index" not in STATE:
           return {"error": "No index loaded. Call POST /ingest first."}
       return run_analysis(STATE["index"], req.error)
   ```

2. **Verify with curl** (Ollama must be running):
   ```bash
   # 1. ingest first (if not already done)
   curl -X POST http://localhost:8080/ingest \
     -H "Content-Type: application/json" \
     -d '{"repo":"../sample_data/repo","logs":"../sample_data/logs/app.log","doc":"../sample_data/design.md"}'

   # 2. analyze
   curl -X POST http://localhost:8080/analyze \
     -H "Content-Type: application/json" \
     -d '{"error":"ValueError: discount percent cannot exceed 100"}' | python -m json.tool
   ```
   Expected: JSON with `report.root_cause`, `report.suspect_files`, `report.log_representation`.

**Commit:**
```bash
git add backend/app/main.py
git commit -m "feat: POST /analyze wired to analysis pipeline"
```

---

## Phase 2.8 — Phase 2 integration check

### Steps

1. **Run the complete test suite:**
   ```bash
   cd backend && pytest -v
   ```
   Unit tests (no `-m integration`) should all pass. Mark-filtered integration tests require Ollama.

2. **Run only integration tests (end-to-end):**
   ```bash
   pytest -v -m integration
   ```
   Expected: pipeline returns `suspect_files` containing `apply_discount` or `buggy_app.py`.

---

### Phase 2 complete ✓

**Milestone check:**
- [ ] `POST /analyze` returns structured JSON with all three report fields
- [ ] Checker's `[UNVERIFIED]` prefix appears on any hallucinated file
- [ ] All unit tests pass; integration tests pass with Ollama running

---

---

# PHASE 3 — Web App + WOW

**Goal:** A usable browser interface — paste an error, get a rich report, and **click any log line to jump to the exact source code that printed it** (the WOW feature).

---

## Phase 3.1 — SQLite persistence (history + feedback)

**Goal:** Save every analysis result and operator feedback for future "have we seen this?" queries.

**Files created:** `backend/app/db.py`
**Files modified:** `backend/app/main.py` (call `db.save_report` inside `/analyze`)

### Steps

1. **Implement `backend/app/db.py`:**
   ```python
   import sqlite3
   import json

   DB_PATH = "analyzer.db"


   def _conn() -> sqlite3.Connection:
       return sqlite3.connect(DB_PATH)


   def init_db() -> None:
       with _conn() as c:
           c.execute("""
               CREATE TABLE IF NOT EXISTS reports (
                   id      INTEGER PRIMARY KEY AUTOINCREMENT,
                   error   TEXT NOT NULL,
                   result  TEXT NOT NULL,
                   created TEXT DEFAULT (datetime('now'))
               )
           """)
           c.execute("""
               CREATE TABLE IF NOT EXISTS feedback (
                   id        INTEGER PRIMARY KEY AUTOINCREMENT,
                   report_id INTEGER NOT NULL,
                   true_file TEXT,
                   worked    INTEGER,
                   note      TEXT,
                   created   TEXT DEFAULT (datetime('now'))
               )
           """)


   def save_report(error: str, result: dict) -> int:
       with _conn() as c:
           cur = c.execute(
               "INSERT INTO reports(error, result) VALUES(?, ?)",
               (error, json.dumps(result)),
           )
           return cur.lastrowid


   def list_reports() -> list[dict]:
       with _conn() as c:
           rows = c.execute(
               "SELECT id, error, result, created FROM reports ORDER BY id DESC LIMIT 50"
           ).fetchall()
       return [{"id": r[0], "error": r[1],
                "result": json.loads(r[2]), "created": r[3]} for r in rows]


   def save_feedback(report_id: int, true_file: str | None,
                     worked: bool, note: str = "") -> None:
       with _conn() as c:
           c.execute(
               "INSERT INTO feedback(report_id, true_file, worked, note) VALUES(?,?,?,?)",
               (report_id, true_file, int(worked), note),
           )
   ```

2. **Wire `db.init_db()` and `db.save_report()` into `main.py`:**
   ```python
   # At the top of main.py, add:
   from app import db
   db.init_db()   # call at module load

   # Inside analyze_route, before return:
   report_id = db.save_report(req.error, result)
   result["report_id"] = report_id
   ```

3. **Verify** — call `/analyze` twice, then:
   ```bash
   sqlite3 analyzer.db "SELECT id, error FROM reports;"
   ```
   Expected: two rows.

**Commit:**
```bash
git add backend/app/db.py backend/app/main.py
git commit -m "feat: SQLite history and feedback persistence"
```

---

## Phase 3.2 — Back-map (log line → source call-site) — the WOW logic

**Goal:** Given a raw log line, return the `CallSite` (source `file:line`) that emitted it. Works by matching static tokens in the log against the logging call's format string.

**Files created:** `backend/app/analysis/backmap.py`
**Tests:** `backend/tests/test_backmap.py`

### Steps

1. **Write the failing test:**
   ```python
   # backend/tests/test_backmap.py
   import pytest
   from tests.conftest import SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC

   @pytest.mark.integration
   def test_backmaps_error_log_to_source_file():
       from app.indexing.store import build_index
       from app.analysis.backmap import backmap

       idx = build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)
       cs = backmap(idx, "Invalid discount percent: 150 for price 99.0")
       assert cs is not None
       assert cs.file.endswith("buggy_app.py")

   @pytest.mark.integration
   def test_backmap_returns_none_for_unrelated_text():
       from app.indexing.store import build_index
       from app.analysis.backmap import backmap

       idx = build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)
       cs = backmap(idx, "completely unrelated text that matches nothing")
       assert cs is None
   ```

2. **Run — expect FAIL.**

3. **Implement `backend/app/analysis/backmap.py`:**
   ```python
   import re
   from app.models import CallSite

   _NOISE = {"none", "null", "true", "false", "nan", "the", "for", "and"}


   def _static_tokens(text: str) -> set[str]:
       """Extract meaningful word tokens, ignoring numbers, IDs, and noise words."""
       return {
           w.lower() for w in re.findall(r"[A-Za-z]{3,}", text)
           if w.lower() not in _NOISE
       }


   def backmap(index, log_line: str, threshold: float = 0.6) -> CallSite | None:
       """
       Find the CallSite whose format string best matches the given log line.
       Returns None if no match exceeds the threshold.
       """
       target_tokens = _static_tokens(log_line)
       if not target_tokens:
           return None

       best_site: CallSite | None = None
       best_score = 0.0

       for cs in index.call_sites:
           fmt_tokens = _static_tokens(cs.format_string)
           if not fmt_tokens:
               continue
           # Overlap = fraction of the format's tokens found in the log line
           overlap = len(target_tokens & fmt_tokens) / len(fmt_tokens)
           if overlap > best_score:
               best_score = overlap
               best_site = cs

       return best_site if best_score >= threshold else None
   ```

4. **Run — expect PASS:**
   ```bash
   pytest tests/test_backmap.py -v -m integration
   ```

**Commit:**
```bash
git add backend/app/analysis/backmap.py backend/tests/test_backmap.py
git commit -m "feat: log-line to source call-site back-mapper (WOW core logic)"
```

---

## Phase 3.3 — Additional API endpoints

**Goal:** Add `/backmap`, `/incidents`, and `/feedback` routes.

**Files modified:** `backend/app/main.py`

### Steps

1. **Add three routes to `main.py`:**
   ```python
   from app.analysis.backmap import backmap as _backmap

   class BackmapRequest(BaseModel):
       log_line: str

   @app.post("/backmap")
   def backmap_route(req: BackmapRequest):
       if "index" not in STATE:
           return {"error": "No index loaded."}
       cs = _backmap(STATE["index"], req.log_line)
       return cs.model_dump() if cs else {}

   @app.get("/incidents")
   def incidents():
       return db.list_reports()

   class FeedbackRequest(BaseModel):
       report_id: int
       true_file: str | None = None
       worked: bool = False
       note: str = ""

   @app.post("/feedback")
   def feedback(req: FeedbackRequest):
       db.save_feedback(req.report_id, req.true_file, req.worked, req.note)
       return {"ok": True}
   ```

2. **Verify each endpoint:**
   ```bash
   # test backmap
   curl -X POST http://localhost:8080/backmap \
     -H "Content-Type: application/json" \
     -d '{"log_line": "Invalid discount percent: 150 for price 99.0"}'
   # expected: {"file":".../buggy_app.py","line":8,...}

   # test incidents
   curl http://localhost:8080/incidents
   # expected: array of saved reports

   # test feedback
   curl -X POST http://localhost:8080/feedback \
     -H "Content-Type: application/json" \
     -d '{"report_id":1,"true_file":"buggy_app.py","worked":true}'
   # expected: {"ok":true}
   ```

**Commit:**
```bash
git add backend/app/main.py
git commit -m "feat: /backmap, /incidents, /feedback endpoints"
```

---

## Phase 3.4 — Frontend scaffold

**Goal:** Vite + React-TS + Tailwind project created and running.

### Steps

1. **Scaffold the frontend:**
   ```bash
   cd log-analyzer-agent
   npm create vite@latest frontend -- --template react-ts
   cd frontend && npm install
   ```

2. **Install Tailwind:**
   ```bash
   npm install -D tailwindcss postcss autoprefixer
   npx tailwindcss init -p
   ```

3. **Update `tailwind.config.js`:**
   ```js
   export default {
     content: ["./index.html", "./src/**/*.{ts,tsx}"],
     theme: { extend: {} },
     plugins: [],
   }
   ```

4. **Replace `src/index.css` content:**
   ```css
   @tailwind base;
   @tailwind components;
   @tailwind utilities;
   ```

5. **Verify it runs:**
   ```bash
   npm run dev
   ```
   Expected: Vite dev server at `http://localhost:5173` — default React page.

**Commit:**
```bash
git add frontend/
git commit -m "chore: Vite React-TS + Tailwind frontend scaffold"
```

---

## Phase 3.5 — API client (api.ts)

**Goal:** Typed fetch wrappers so components never hardcode URLs.

**Files created:** `frontend/src/api.ts`

### Steps

1. **Create `frontend/src/api.ts`:**
   ```typescript
   const BASE = "http://localhost:8080";

   export async function ingest(repo: string, logs: string, doc: string) {
     const r = await fetch(`${BASE}/ingest`, {
       method: "POST",
       headers: { "Content-Type": "application/json" },
       body: JSON.stringify({ repo, logs, doc }),
     });
     return r.json();
   }

   export async function analyze(error: string) {
     const r = await fetch(`${BASE}/analyze`, {
       method: "POST",
       headers: { "Content-Type": "application/json" },
       body: JSON.stringify({ error }),
     });
     return r.json();
   }

   export async function backmap(logLine: string) {
     const r = await fetch(`${BASE}/backmap`, {
       method: "POST",
       headers: { "Content-Type": "application/json" },
       body: JSON.stringify({ log_line: logLine }),
     });
     return r.json();
   }

   export async function getIncidents() {
     const r = await fetch(`${BASE}/incidents`);
     return r.json();
   }

   export async function sendFeedback(reportId: number, trueFile: string, worked: boolean) {
     const r = await fetch(`${BASE}/feedback`, {
       method: "POST",
       headers: { "Content-Type": "application/json" },
       body: JSON.stringify({ report_id: reportId, true_file: trueFile, worked }),
     });
     return r.json();
   }
   ```

**Commit:**
```bash
git add frontend/src/api.ts
git commit -m "feat: typed API client for all backend endpoints"
```

---

## Phase 3.6 — ErrorInput component

**Goal:** Textarea for the error + submit button + ingest controls.

**Files created:** `frontend/src/components/ErrorInput.tsx`

### Steps

1. **Create `frontend/src/components/ErrorInput.tsx`:**
   ```tsx
   import { useState } from "react";

   interface Props {
     onAnalyze: (error: string) => void;
     loading: boolean;
   }

   export function ErrorInput({ onAnalyze, loading }: Props) {
     const [error, setError] = useState("");

     return (
       <div className="space-y-3">
         <label className="block text-sm font-medium text-gray-700">
           Paste your error + relevant log lines
         </label>
         <textarea
           className="w-full h-48 p-3 font-mono text-sm border rounded-lg
                      focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"
           placeholder={"Traceback (most recent call last):\n  ...\nValueError: ..."}
           value={error}
           onChange={(e) => setError(e.target.value)}
         />
         <button
           onClick={() => onAnalyze(error)}
           disabled={!error.trim() || loading}
           className="px-5 py-2 bg-blue-600 text-white rounded-lg
                      disabled:opacity-40 hover:bg-blue-700 transition-colors"
         >
           {loading ? "Analyzing…" : "Analyze"}
         </button>
       </div>
     );
   }
   ```

---

## Phase 3.7 — IncidentReport + SuspectFileCard components

**Goal:** Display the grounded report — root cause, ranked files with confidence bars, log representation.

**Files created:** `frontend/src/components/SuspectFileCard.tsx`, `frontend/src/components/IncidentReport.tsx`

### Steps

1. **Create `frontend/src/components/SuspectFileCard.tsx`:**
   ```tsx
   interface SuspectFile {
     path: string;
     start_line: number;
     end_line: number;
     reason: string;
     confidence: number;
   }

   export function SuspectFileCard({ file, rank }: { file: SuspectFile; rank: number }) {
     const confidencePct = Math.round(file.confidence * 100);
     const barColor = confidencePct >= 70 ? "bg-red-500"
                    : confidencePct >= 40 ? "bg-yellow-500"
                    : "bg-gray-400";

     return (
       <div className="border rounded-lg p-4 bg-white shadow-sm">
         <div className="flex items-start justify-between gap-2">
           <div>
             <span className="text-xs text-gray-400 mr-2">#{rank}</span>
             <span className="font-mono text-sm font-semibold text-blue-700">
               {file.path}
             </span>
             <span className="text-xs text-gray-500 ml-2">
               lines {file.start_line}–{file.end_line}
             </span>
           </div>
           <span className="text-xs font-medium text-gray-600 whitespace-nowrap">
             {confidencePct}%
           </span>
         </div>
         {/* confidence bar */}
         <div className="mt-2 h-1.5 w-full bg-gray-100 rounded-full">
           <div className={`h-1.5 rounded-full ${barColor}`}
                style={{ width: `${confidencePct}%` }} />
         </div>
         <p className="mt-2 text-sm text-gray-700">{file.reason}</p>
       </div>
     );
   }
   ```

2. **Create `frontend/src/components/IncidentReport.tsx`:**
   ```tsx
   import { SuspectFileCard } from "./SuspectFileCard";
   import { LogTimeline } from "./LogTimeline";  // created in 3.8

   interface Props {
     report: any;
     warnings: string[];
   }

   export function IncidentReport({ report, warnings }: Props) {
     if (!report) return null;

     // Split log_representation.where_to_grep across lines for the timeline
     const logLines = report.log_representation?.where_to_grep
       ? [report.log_representation.where_to_grep]
       : [];

     return (
       <div className="space-y-6 mt-6">
         {warnings.length > 0 && (
           <div className="bg-yellow-50 border border-yellow-300 rounded-lg p-3 text-sm text-yellow-800">
             ⚠ {warnings.join(" · ")}
           </div>
         )}

         {/* Root cause */}
         <section>
           <h2 className="text-lg font-semibold mb-1">Root Cause</h2>
           <p className="text-gray-800 leading-relaxed">{report.root_cause}</p>
         </section>

         {/* Suspect files */}
         <section>
           <h2 className="text-lg font-semibold mb-2">Suspect Files</h2>
           <div className="space-y-2">
             {report.suspect_files?.map((f: any, i: number) => (
               <SuspectFileCard key={i} file={f} rank={i + 1} />
             ))}
           </div>
         </section>

         {/* Log representation */}
         <section>
           <h2 className="text-lg font-semibold mb-1">How It Appears in Logs</h2>
           <div className="bg-gray-900 text-green-300 rounded-lg p-3 font-mono text-sm mb-2">
             {report.log_representation?.template}
           </div>
           <p className="text-sm text-gray-600">
             <strong>Grep for:</strong> <code>{report.log_representation?.where_to_grep}</code>
           </p>
           <p className="text-sm text-gray-600 mt-1">
             {report.log_representation?.surrounding_meaning}
           </p>
         </section>

         {/* WOW — log timeline */}
         <LogTimeline lines={logLines} />

         {/* Next checks */}
         {report.next_checks?.length > 0 && (
           <section>
             <h2 className="text-lg font-semibold mb-1">Next Checks</h2>
             <ul className="list-disc list-inside text-sm text-gray-700 space-y-1">
               {report.next_checks.map((c: string, i: number) => <li key={i}>{c}</li>)}
             </ul>
           </section>
         )}
       </div>
     );
   }
   ```

---

## Phase 3.8 — LogTimeline WOW component

**Goal:** Render log lines; clicking one calls `/backmap` and shows the exact source line that printed it — with the file path and code snippet.

**Files created:** `frontend/src/components/LogTimeline.tsx`

### Steps

1. **Create `frontend/src/components/LogTimeline.tsx`:**
   ```tsx
   import { useState } from "react";
   import { backmap } from "../api";

   interface CallSite {
     file: string;
     line: number;
     text: string;
     format_string: string;
   }

   interface Props {
     lines: string[];
   }

   export function LogTimeline({ lines }: Props) {
     const [hitMap, setHitMap] = useState<Record<number, CallSite | null>>({});
     const [loading, setLoading] = useState<number | null>(null);

     async function handleClick(line: string, idx: number) {
       if (hitMap[idx] !== undefined) return;   // already fetched
       setLoading(idx);
       const result = await backmap(line);
       setHitMap(prev => ({ ...prev, [idx]: result?.file ? result : null }));
       setLoading(null);
     }

     if (lines.length === 0) return null;

     return (
       <section>
         <h2 className="text-lg font-semibold mb-1">
           Log Timeline
           <span className="ml-2 text-xs font-normal text-gray-400">
             click a line to see the source that printed it
           </span>
         </h2>
         <div className="rounded-lg border overflow-hidden">
           {lines.map((ln, i) => (
             <div key={i}>
               <pre
                 onClick={() => handleClick(ln, i)}
                 className={`px-4 py-2 text-xs font-mono cursor-pointer transition-colors
                   ${loading === i ? "bg-blue-50 animate-pulse" : "hover:bg-yellow-50"}`}
               >
                 {ln}
               </pre>
               {hitMap[i] && (
                 <div className="px-4 py-2 bg-gray-900 text-green-300 text-xs font-mono border-t">
                   <div className="text-gray-400 mb-1">
                     ↳ printed by {hitMap[i]!.file}:{hitMap[i]!.line}
                   </div>
                   <div>{hitMap[i]!.text}</div>
                 </div>
               )}
               {hitMap[i] === null && (
                 <div className="px-4 py-1 text-xs text-gray-400 italic border-t">
                   ↳ no matching logging call found
                 </div>
               )}
             </div>
           ))}
         </div>
       </section>
     );
   }
   ```

2. **Wire everything in `frontend/src/App.tsx`:**
   ```tsx
   import { useState } from "react";
   import { analyze } from "./api";
   import { ErrorInput } from "./components/ErrorInput";
   import { IncidentReport } from "./components/IncidentReport";

   export default function App() {
     const [loading, setLoading] = useState(false);
     const [result, setResult]   = useState<any>(null);

     async function handleAnalyze(error: string) {
       setLoading(true);
       setResult(null);
       const data = await analyze(error);
       setResult(data);
       setLoading(false);
     }

     return (
       <div className="max-w-3xl mx-auto px-4 py-8">
         <h1 className="text-2xl font-bold mb-6">Log Analyzer Agent</h1>
         <ErrorInput onAnalyze={handleAnalyze} loading={loading} />
         {result && (
           <IncidentReport
             report={result.report}
             warnings={result.warnings ?? []}
           />
         )}
       </div>
     );
   }
   ```

**Commit:**
```bash
git add frontend/src/
git commit -m "feat: React UI — ErrorInput, IncidentReport, SuspectFileCard, LogTimeline WOW"
```

---

## Phase 3.9 — Phase 3 full browser verification

### Steps

1. **Start everything:**
   ```bash
   # terminal 1 — backend
   cd backend && uvicorn app.main:app --port 8080

   # terminal 2 — frontend
   cd frontend && npm run dev
   ```

2. **Open `http://localhost:5173` in a browser.**

3. **Golden path test (do all of these):**
   - [ ] Paste `ValueError: discount percent cannot exceed 100` → click Analyze → report appears
   - [ ] At least one suspect file shows `buggy_app.py`
   - [ ] Log representation section has a grep pattern
   - [ ] Click a log line in the timeline → a code block appears showing `buggy_app.py:8` (the WOW)
   - [ ] The confidence bar is visible and the color matches the value

4. **Edge cases:**
   - [ ] Submit an empty error → button stays disabled
   - [ ] Submit gibberish → report appears with low-confidence suspects or warnings

---

### Phase 3 complete ✓

**Milestone check:**
- [ ] Browser shows the full incident report
- [ ] Clicking a log line → the source call-site appears inline (WOW works)
- [ ] SQLite has saved reports viewable via `/incidents`

---

---

# PHASE 4 — Eval, Hardening & Demo

---

## Phase 4.1 — Labeled eval dataset

**Goal:** 10–20 labeled cases of `{error, true_file}` pairs built from your real project's past incidents.

**Files created:** `backend/eval/incidents.jsonl`

### Steps

1. **Format:** one JSON object per line:
   ```json
   {"error": "ValueError: discount percent cannot exceed 100", "true_file": "buggy_app.py"}
   {"error": "KeyError: 'user_id' in checkout flow", "true_file": "checkout.py"}
   ```

2. **Collect at least 10 cases from your real project's history** (git log, issue tracker, Slack threads). Ask your mentor for past incidents. The more real cases, the more meaningful the metric.

3. **Start collecting from Week 1** — don't leave this to the last day.

---

## Phase 4.2 — Eval harness

**Goal:** Measure top-1 and top-3 file-localization hit rate and median latency.

**Files created:** `backend/eval/run_eval.py`

### Steps

1. **Create `backend/eval/run_eval.py`:**
   ```python
   import json
   import time
   import sys
   import os

   sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))

   from app.indexing.store import build_index
   from app.analysis.pipeline import run_analysis


   def hit_at_k(suspects: list[dict], true_file: str, k: int) -> bool:
       return any(true_file in s["path"] or s["path"].endswith(true_file)
                  for s in suspects[:k])


   def main(repo: str, logs: str, doc: str, dataset: str):
       print(f"Building index from {repo}…")
       idx = build_index(repo, logs, doc)

       cases = [json.loads(line) for line in open(dataset)]
       print(f"Running eval on {len(cases)} cases…\n")

       top1 = top3 = 0
       latencies = []

       for case in cases:
           t0 = time.time()
           out = run_analysis(idx, case["error"])
           latencies.append(time.time() - t0)

           suspects = out["report"].get("suspect_files", [])
           h1 = hit_at_k(suspects, case["true_file"], 1)
           h3 = hit_at_k(suspects, case["true_file"], 3)
           top1 += h1; top3 += h3

           status = "✓" if h1 else ("~" if h3 else "✗")
           print(f"  [{status}] {case['true_file'][:40]:<40}  {latencies[-1]:.1f}s")

       n = len(cases)
       latencies.sort()
       print(f"\n{'─'*50}")
       print(f"  top-1 hit rate : {top1/n:.0%}  ({top1}/{n})")
       print(f"  top-3 hit rate : {top3/n:.0%}  ({top3}/{n})")
       print(f"  median latency : {latencies[n//2]:.1f}s")


   if __name__ == "__main__":
       main(
           repo  = sys.argv[1],
           logs  = sys.argv[2],
           doc   = sys.argv[3],
           dataset = "eval/incidents.jsonl",
       )
   ```

2. **Run:**
   ```bash
   cd backend && python eval/run_eval.py \
     ../sample_data/repo ../sample_data/logs/app.log ../sample_data/design.md
   ```

3. **Tune until top-3 ≥ 70%.** Knobs to turn: `TOP_K` in config, BM25/vector weights in retriever, chunk size in code indexer.

**Commit:**
```bash
git add backend/eval/
git commit -m "feat: eval harness — top-k file localization hit rate and latency"
```

---

## Phase 4.3 — PII / secret redaction

**Goal:** Strip emails, auth tokens, and long hex strings before anything is embedded or sent to the model.

**Files modified:** `backend/app/analysis/normalizer.py`

### Steps

1. **Add to `normalizer.py`:**
   ```python
   _REDACT_PATTERNS = [
       (re.compile(r"[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z]{2,}"), "[EMAIL]"),
       (re.compile(r"(?:Bearer|Token|token|key|secret)\s*[=:]\s*[\w.\-/+]{8,}",
                   re.IGNORECASE), "[CREDENTIAL]"),
       (re.compile(r"\b[0-9a-fA-F]{32,}\b"), "[HEX_ID]"),
       (re.compile(r"eyJ[\w-]+\.[\w-]+\.[\w-]+"), "[JWT]"),
   ]

   def redact(text: str) -> str:
       for pattern, replacement in _REDACT_PATTERNS:
           text = pattern.sub(replacement, text)
       return text
   ```

2. **Call `redact()` at the top of `normalize()`:**
   ```python
   def normalize(raw_error: str) -> ErrorSignature:
       raw_error = redact(raw_error)
       ...  # rest of the function unchanged
   ```

3. **Add a unit test in `test_normalizer.py`:**
   ```python
   def test_redacts_jwt_from_error():
       jwt = "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWRtaW4ifQ.abc123xyz"
       sig = normalize(f"AuthError: invalid token {jwt}")
       assert jwt not in sig.raw
       assert "[JWT]" in sig.raw
   ```

**Commit:**
```bash
git add backend/app/analysis/normalizer.py backend/tests/test_normalizer.py
git commit -m "feat: PII and credential redaction before embedding and LLM"
```

---

## Phase 4.4 — Robustness & edge cases

**Goal:** Handle the empty-index case, low-confidence results, and malformed model output without crashing.

### Steps

1. **Empty index guard** — already in `/analyze` (`"No index loaded"`). Add a similar guard to `/backmap`.

2. **Low-confidence UI state** — in `IncidentReport.tsx`, if all `suspect_files` have confidence < 0.3, show a banner: *"No confident suspects found — consider expanding the codebase context."*
   ```tsx
   const lowConfidence = report.suspect_files?.every((f: any) => f.confidence < 0.3);
   {lowConfidence && (
     <div className="bg-orange-50 border border-orange-200 rounded-lg p-3 text-sm text-orange-800">
       No high-confidence suspects found. Try ingesting more of the codebase.
     </div>
   )}
   ```

3. **Model JSON retry** — already built into `analyze()` in Phase 2.4 with 3 retries and a 2-second back-off. No further changes needed here.

**Commit:**
```bash
git commit -am "feat: robustness — empty-index guards, low-confidence banner, LLM retry"
```

---

## Phase 4.5 — Performance

**Goal:** Embeddings model loads once; Chroma store is persisted (not rebuilt per server start).

### Steps

1. **Persist and reload index on server start** — if `CHROMA_DIR` already exists, load rather than rebuild:
   ```python
   # In store.py, add a load function:
   def load_index(repo: str, logs: str, doc: str) -> Index:
       """Load existing Chroma index; fall back to build if it doesn't exist."""
       from pathlib import Path
       if Path(settings.CHROMA_DIR).exists():
           _, call_sites = index_code(repo)
           events = index_logs(logs)
           store = Chroma(persist_directory=settings.CHROMA_DIR,
                          embedding_function=_get_embeddings())
           all_docs = store.get()  # lightweight reference
           return Index(store=store, all_docs=[], call_sites=call_sites, log_events=events)
       return build_index(repo, logs, doc)
   ```
   *(For the demo, always call `build_index` on ingest so results are fresh — use `load_index` for warm server restarts.)*

2. **The `_get_embeddings()` singleton in `store.py`** is already implemented. Verify it's only instantiated once by checking the log — you should see the MiniLM download message at most once per server process.

**Commit:**
```bash
git commit -am "perf: embeddings singleton + Chroma persistence between restarts"
```

---

## Phase 4.6 — Full packaging (process-based)

**Goal:** One script starts api + web; Chroma runs as a local process. No Docker required.

### Steps

1. **Install Chroma as a local server** (already in requirements via `chromadb`). Add a startup convenience script `start.sh` at the project root:
   ```bash
   #!/usr/bin/env bash
   set -e

   echo "Starting Chroma..."
   chroma run --path ./chroma_store &
   CHROMA_PID=$!

   echo "Starting API..."
   cd backend
   uvicorn app.main:app --host 0.0.0.0 --port 8080 &
   API_PID=$!
   cd ..

   echo "Starting frontend..."
   cd frontend
   npm run preview -- --port 3000 &
   WEB_PID=$!
   cd ..

   echo "All services started. Press Ctrl+C to stop."
   trap "kill $CHROMA_PID $API_PID $WEB_PID 2>/dev/null" EXIT
   wait
   ```
   ```bash
   chmod +x start.sh
   ```

2. **Build the frontend for production:**
   ```bash
   cd frontend && npm run build
   ```

3. **Verify:**
   ```bash
   ./start.sh
   # open http://localhost:3000 — full app should load
   ```

**Commit:**
```bash
git add start.sh
git commit -m "feat: start.sh — one-command bring-up without Docker"
```

---

## Phase 4.7 — README & demo script

**Goal:** Anyone can clone and run the project; you can present confidently.

### Steps

1. **Create `README.md`** covering:
   - One-sentence description
   - Prerequisites (Python 3.11, Node 18+, Samsung Gauss API credentials)
   - Setup: `git clone` → `pip install -r requirements.txt` → copy `.env.example` to `.env` and fill in credentials → `npm install` in `frontend/` → `./start.sh`
   - Usage: call `/ingest` with paths, then `/analyze` with an error
   - The WOW: click a log line → see its source
   - Eval: how to run `eval/run_eval.py`
   - Architecture diagram (the text flow diagram from design.md)

2. **Write a 5-step demo script** (practice this out loud):
   1. "I'll ingest a Python project with its logs and design doc." → `POST /ingest`
   2. "Now I paste a real error from our issue tracker." → `POST /analyze` → report appears
   3. "The agent identified `apply_discount` in `buggy_app.py:8` as the culprit — with the reason and confidence."
   4. "Watch this — I click the log line…" → source call-site appears inline (WOW moment)
   5. "Here's the eval — top-3 hit rate across 15 past incidents is X%. This is what it can do as-is. Next: bounded multi-step agent for multi-hop bugs."
**Commit:**
```bash
git add README.md
git commit -m "docs: README, setup guide, and demo script"
git tag v1.0
```

---

## Phase 4.8 — Final demo dry-run

### Checklist

- [ ] `./start.sh` from a clean checkout — everything starts, no errors
- [ ] Ingest the sample data — response within 30 sec
- [ ] Analyze the sample error — report appears with suspect files
- [ ] Click a log line — WOW works
- [ ] Run `python eval/run_eval.py` — prints top-k and latency numbers
- [ ] `pytest -v` — all unit tests pass
- [ ] No hardcoded localhost URLs in the frontend (use `BASE` from `api.ts`)
- [ ] Demo script rehearsed — can do it in < 5 minutes

---

### Phase 4 complete ✓

**Milestone:** polished demo + quoted accuracy metric + "where it goes next" story.

---

---

# PHASE 5 — Stretch (post-internship)

These are "where it goes next" — describe them in the demo even if not built.

---

## Phase 5.1 — Bounded multi-step agent (ReAct, capped)

**Goal:** Let Qwen3 issue a few tool calls before answering, so it can handle multi-hop bugs (exception surfaces in File A, real cause is in File B's caller).

**Architecture:**
- Define four tools: `search_code(query)`, `read_file(path, start, end)`, `search_logs(query)`, `get_doc(query)`
- Each wraps the already-built retriever / store methods
- The agent loop: `Qwen3 → tool call → result → Qwen3 → (up to N=5 steps) → IncidentReport`
- **Only here** does LangGraph or a ReAct loop earn its place

**Why it's phase 5:** reliable tool-calling from an 8B model needs careful prompt engineering and evaluation. The MVP pipeline already handles 80% of cases; this adds multi-hop.

---

## Phase 5.2 — Self-learning incident memory

**Goal:** Confirmed fixes become "incident cards" the agent recognizes instantly.

**Architecture:**
- Every time an operator submits feedback with `worked=true`, distill the `{error_signature → true_file → fix summary}` into an incident card and embed it
- At analysis time, before calling Qwen3, check the incident card index first — if similarity > 0.9, return the cached answer instantly with a link to the prior incident
- Demo: show the same error twice — first time takes 15 sec; second time returns in < 1 sec with "seen before"

---

## Quick-reference command table

| Action | Command |
|---|---|
| Start all services | `./start.sh` |
| Start API (dev) | `cd backend && uvicorn app.main:app --reload --port 8080` |
| Start frontend (dev) | `cd frontend && npm run dev` |
| Run unit tests | `cd backend && pytest -v` |
| Run integration tests | `cd backend && pytest -v -m integration` |
| Run eval | `cd backend && python eval/run_eval.py <repo> <logs> <doc>` |
| Ingest sample data | `curl -X POST localhost:8080/ingest -H 'Content-Type: application/json' -d '{"repo":"../sample_data/repo","logs":"../sample_data/logs/app.log","doc":"../sample_data/design.md"}'` |
| Analyze an error | `curl -X POST localhost:8080/analyze -H 'Content-Type: application/json' -d '{"error":"<paste here>"}'` |
