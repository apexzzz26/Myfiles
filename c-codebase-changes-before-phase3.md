# C / NCM Retarget — Changes to Phases 0–2 (do these BEFORE Phase 3)

> **Context:** You finished Phase 2 of `master-plan.md` against a **Python** sample. The real target is the **NCM** codebase (C source in `.c`/`.h`, C++ GoogleTest unit tests in `Test/ut`). This file lists the changes to the code you already wrote so the indexer, normalizer, and analyzer work on C. Do all of these, re-run the tests, then start Phase 3.
>
> Companion file `phase3-onwards-changes.md` covers what changes in Phase 3+ as a result.

## Why C forces changes (read first)

| Area | Python (what you built) | C / NCM (what's needed) |
|---|---|---|
| Function name | `function_definition` has a direct `name` identifier | C name is nested: `function_definition → declarator(function_declarator) → declarator(identifier)`, sometimes wrapped in `pointer_declarator` |
| Logging | `log.error("…")` — callee contains "log" | Custom macros: `NCM_LOG_ERR`, `NCM_DEBUG`, `ncm_debug_log`, maybe `syslog` — you must feed the real macro names |
| Errors | Python traceback | gdb backtrace, `assert` failures, **GoogleTest** failures, signals (SIGSEGV) |
| Log format | `YYYY-MM-DD LEVEL source msg` | NCM/syslog format — **unknown, you must confirm** |
| Function size | small | NCM functions can be huge → blow past MiniLM's 256-token cap |
| Headers | few | large `hdr/` tree full of structs, enums, error-code macros worth indexing |

## ⚠️ Two unknowns you must resolve as you apply this

1. **Your real log line format.** The current parser assumes the Python format. Change 1.2 makes it pluggable — you must add your actual NCM format regex.
2. **Your real logging macro names.** Grep the codebase first:
   ```bash
   grep -rhoE '\b(NCM_[A-Z_]*LOG[A-Z_]*|NCM_[A-Z_]*DBG[A-Z_]*|NCM_[A-Z_]*DEBUG[A-Z_]*|[a-z_]*log[a-z_]*)\s*\(' NCM/NCMMAIN | sort -u
   ```
   Also open `ncm_debug.h` and `ncm_lib/ncm_debug_log.c` — those define the logging layer. Put the real names into `LOG_FUNCTION_HINTS` (Change 0-A).

---

## Change 0-A — `config.py`: add language + logging settings

**File:** `backend/app/config.py` — add fields to `Settings`.

```python
class Settings(BaseSettings):
    # ... existing GAUSS_* / EMBED_MODEL / CHROMA_DIR / TOP_K ...

    # Languages whose grammars we parse (tree-sitter-language-pack names)
    PRIMARY_LANGUAGES: list[str] = ["c", "cpp"]

    # Substrings (lowercased) that mark a call as a logging call.
    # REPLACE with the real NCM macro names you find via grep.
    LOG_FUNCTION_HINTS: list[str] = [
        "log", "dbg", "debug", "trace", "syslog", "err", "warn", "print",
    ]

    # Max lines per code chunk before we split it (keeps under MiniLM 256-token cap)
    MAX_CHUNK_LINES: int = 80

    class Config:
        env_file = ".env"
```

**Commit:** `git commit -am "feat: config — C language + logging-macro hints + chunk cap"`

---

## Change 0-B — Replace the sample data with an NCM-style C sample

So your Phase 0–2 tests stay meaningful. **Delete** `buggy_app.py`; **create** these.

**`sample_data/repo/ncm_debug.h`**
```c
#ifndef NCM_DEBUG_H
#define NCM_DEBUG_H
void ncm_debug_log(const char *level, const char *fmt, ...);
#define NCM_LOG_ERR(fmt, ...) ncm_debug_log("ERROR", fmt, ##__VA_ARGS__)
#define NCM_LOG_INFO(fmt, ...) ncm_debug_log("INFO", fmt, ##__VA_ARGS__)
#endif
```

**`sample_data/repo/ncm_sample.c`**
```c
#include "ncm_debug.h"

int ncm_apply_discount(int price, int percent)
{
    if (percent > 100) {
        NCM_LOG_ERR("Invalid discount percent: %d for price %d", percent, price);
        return -1;
    }
    return price - (price * percent / 100);
}
```
*(`NCM_LOG_ERR` is on line 6 — the tests below assert that.)*

**`sample_data/logs/app.log`** — ⚠️ replace with your **real** NCM format once known:
```
Jun 25 10:15:02 ncm-host ncm[2451]: ERROR Invalid discount percent: 150 for price 99
Jun 25 10:15:02 ncm-host ncm[2451]: INFO request abc123 finished status=500
Jun 25 10:16:11 ncm-host ncm[2451]: INFO request def456 finished status=200
```

**`sample_data/design.md`**
```markdown
# NCM Discount Module — Design
`ncm_apply_discount(price, percent)` applies a percentage discount.
Valid percent is 0–100 inclusive. Out-of-range percent is a client error:
log at ERROR via NCM_LOG_ERR and return -1 (never crash).
Logs use syslog format: `MON DD HH:MM:SS host proc[pid]: LEVEL message`.
```

**Commit:** `git commit -am "chore: replace sample data with NCM-style C sample"`

---

## Change 1.1 — `code_indexer.py`: make it C-aware (the biggest change)

**File:** `backend/app/indexing/code_indexer.py` — **replace the whole file** with this. It: handles C/C++ function-name nesting, detects logging via `settings.LOG_FUNCTION_HINTS`, splits oversized functions, and indexes header macros/structs/enums/typedefs.

```python
from pathlib import Path
from tree_sitter_language_pack import get_parser
from app.config import settings
from app.models import CodeChunk, CallSite

EXT_TO_LANG = {
    ".c": "c", ".h": "c",          # this repo's headers are C
    ".cpp": "cpp", ".cc": "cpp", ".cxx": "cpp", ".hpp": "cpp",
    ".py": "python", ".js": "javascript", ".ts": "typescript",
    ".java": "java", ".go": "go",
}

DEF_NODES = {
    "c": "function_definition", "cpp": "function_definition",
    "python": "function_definition",
    "javascript": "function_declaration", "typescript": "function_declaration",
    "java": "method_declaration", "go": "function_declaration",
}
CALL_NODES = {
    "c": "call_expression", "cpp": "call_expression",
    "python": "call",
    "javascript": "call_expression", "typescript": "call_expression",
    "java": "method_invocation", "go": "call_expression",
}
# Header-level declarations worth indexing for context (C/C++)
DECL_NODES = {"preproc_def", "preproc_function_def",
              "struct_specifier", "enum_specifier", "type_definition"}


def _text(node, src: bytes) -> str:
    return src[node.start_byte:node.end_byte].decode("utf8", "replace")


def _walk(node):
    yield node
    for child in node.children:
        yield from _walk(child)


def _func_name(node, lang: str, src: bytes) -> str | None:
    if lang == "python":
        n = node.child_by_field_name("name")
        return _text(n, src) if n else None
    # c / cpp: descend the declarator chain to the function_declarator
    decl = node.child_by_field_name("declarator")
    while decl is not None and decl.type != "function_declarator":
        decl = decl.child_by_field_name("declarator")
    if decl is not None:
        ident = decl.child_by_field_name("declarator")
        if ident is not None:
            return _text(ident, src).split("::")[-1]
    for n in _walk(node):                      # fallback
        if n.type in ("identifier", "field_identifier"):
            return _text(n, src)
    return None


def _callee_name(node, src: bytes) -> str:
    fn = node.child_by_field_name("function")
    if fn is not None:
        return _text(fn, src)
    return _text(node, src).split("(")[0]


def _first_string(node, src: bytes) -> str | None:
    for n in _walk(node):
        if n.type in ("string_literal", "string", "concatenated_string"):
            return _text(n, src).strip().strip('"\'`')
    return None


def _decl_name(node, src: bytes) -> str | None:
    n = node.child_by_field_name("name")
    if n:
        return _text(n, src)
    for c in _walk(node):
        if c.type in ("identifier", "type_identifier"):
            return _text(c, src)
    return None


def _emit_chunk(chunks, file, lang, symbol, kind, start, end, text):
    """Split oversized functions so embeddings aren't truncated."""
    cap = settings.MAX_CHUNK_LINES
    if end - start + 1 <= cap:
        chunks.append(CodeChunk(file=file, language=lang, symbol=symbol,
                                start_line=start, end_line=end, text=text[:4000]))
        return
    lines = text.splitlines()
    for i in range(0, len(lines), cap):
        block = lines[i:i + cap]
        chunks.append(CodeChunk(
            file=file, language=lang, symbol=f"{symbol} (part {i // cap + 1})",
            start_line=start + i, end_line=start + i + len(block) - 1,
            text="\n".join(block)))


def _fallback_chunks(path, src: bytes) -> list[CodeChunk]:
    lines = src.decode("utf8", "replace").splitlines()
    win, ov, out, i = 60, 10, [], 0
    while i < len(lines):
        block = lines[i:i + win]
        out.append(CodeChunk(file=str(path), language="unknown", symbol=None,
                             start_line=i + 1, end_line=i + len(block),
                             text="\n".join(block)))
        i += win - ov
    return out


def index_code(root_dir: str) -> tuple[list[CodeChunk], list[CallSite]]:
    chunks: list[CodeChunk] = []
    call_sites: list[CallSite] = []
    hints = tuple(h.lower() for h in settings.LOG_FUNCTION_HINTS)

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
        def_type, call_type = DEF_NODES.get(lang), CALL_NODES.get(lang)

        for node in _walk(tree.root_node):
            if def_type and node.type == def_type:
                _emit_chunk(chunks, str(path), lang,
                            _func_name(node, lang, src), "function",
                            node.start_point[0] + 1, node.end_point[0] + 1,
                            _text(node, src))

            elif lang in ("c", "cpp") and node.type in DECL_NODES:
                name = _decl_name(node, src)
                if name:
                    chunks.append(CodeChunk(
                        file=str(path), language=lang, symbol=name,
                        start_line=node.start_point[0] + 1,
                        end_line=node.end_point[0] + 1,
                        text=_text(node, src)[:2000]))

            if call_type and node.type == call_type:
                callee = _callee_name(node, src).lower()
                if any(h in callee for h in hints):
                    fmt = _first_string(node, src)
                    if fmt:
                        call_sites.append(CallSite(
                            file=str(path), line=node.start_point[0] + 1,
                            text=_text(node, src)[:500], format_string=fmt))

    return chunks, call_sites
```

**Update `test_code_indexer.py`** (replace Python asserts):
```python
from tests.conftest import SAMPLE_REPO
from app.indexing.code_indexer import index_code

def test_extracts_c_function():
    chunks, _ = index_code(SAMPLE_REPO)
    assert any(c.symbol == "ncm_apply_discount" and c.language == "c" for c in chunks)

def test_extracts_ncm_log_macro_call_site():
    _, sites = index_code(SAMPLE_REPO)
    cs = [s for s in sites if "Invalid discount percent" in s.format_string]
    assert len(cs) == 1
    assert cs[0].line == 6
    assert cs[0].file.endswith("ncm_sample.c")

def test_indexes_header_macro():
    chunks, _ = index_code(SAMPLE_REPO)
    assert any(c.symbol == "NCM_LOG_ERR" for c in chunks)
```

**Run:** `cd backend && pytest tests/test_code_indexer.py -v` → 3 PASS.

> **C gotcha:** if some functions aren't found, it's usually a project macro in the signature (e.g. `STATIC void foo()`). tree-sitter mis-parses these. Quick check: `print([c.type for c in node.children])`. The `_fallback_chunks` path still indexes those files, so retrieval degrades gracefully — but add such macros to a strip-list if many functions are missed.

**Commit:** `git commit -am "feat: C/C++-aware code indexer — name nesting, macro logging, chunk splitting, header decls"`

---

## Change 1.2 — `log_indexer.py`: make the line parser pluggable

**File:** `backend/app/indexing/log_indexer.py` — replace the single `LINE_RE` with a pattern list (Drain3 stays the same).

```python
import re
from drain3 import TemplateMiner
from app.models import LogEvent

# Try each pattern in order. ADD YOUR REAL NCM FORMAT AS THE FIRST ENTRY.
LINE_PATTERNS = [
    # syslog: "Jun 25 10:15:02 host proc[pid]: LEVEL message"
    re.compile(r"^(?P<ts>\w{3}\s+\d+\s[\d:]+)\s+(?P<host>\S+)\s+"
               r"(?P<source>\S+?):\s*(?P<msg>.*)$"),
    # ISO: "2026-06-20 10:15:00 LEVEL source message"
    re.compile(r"^(?P<ts>\d{4}-\d{2}-\d{2}[ T][\d:]+)\s+(?P<level>\w+)\s+"
               r"(?P<source>\S+)\s+(?P<msg>.*)$"),
]
_LEVEL_RE = re.compile(r"\b(FATAL|CRIT|ERROR|ERR|WARN|WARNING|NOTICE|INFO|DEBUG)\b")


def _parse_line(line: str) -> dict:
    for pat in LINE_PATTERNS:
        m = pat.match(line)
        if m:
            d = m.groupdict()
            if not d.get("level"):
                lv = _LEVEL_RE.search(d.get("msg", ""))
                d["level"] = lv.group(1) if lv else None
            return d
    return {"ts": None, "level": None, "source": None, "msg": line}


def index_logs(log_path: str) -> list[LogEvent]:
    miner = TemplateMiner()
    events: list[LogEvent] = []
    with open(log_path, encoding="utf8", errors="replace") as fh:
        for i, raw in enumerate(fh, start=1):
            line = raw.rstrip("\n")
            if not line.strip():
                continue
            d = _parse_line(line)
            res = miner.add_log_message(d["msg"])
            events.append(LogEvent(
                line_no=i, timestamp=d.get("ts"), level=d.get("level"),
                source=d.get("source"), message=d["msg"],
                template=res["template_mined"], template_id=res["cluster_id"]))
    return events
```

**Update `test_log_indexer.py`** to your sample (syslog) — assert `level == "ERROR"` for the discount line and `"<*>" in template`.

**Run:** `pytest tests/test_log_indexer.py -v` → PASS.

**Commit:** `git commit -am "feat: pluggable multi-format log line parser (NCM/syslog)"`

---

## Change 2.1 — `normalizer.py`: add C error parsers (critical)

**File:** `backend/app/analysis/normalizer.py` — keep `redact()` and the Python parser; **add** gdb / assert / GoogleTest / signal parsers and route to them.

```python
import re
from app.models import ErrorSignature, StackFrame

# ---- existing: _PY_FRAME, _PY_EXC, _CORR_ID, redact(), _parse_python() ----

_GDB_FRAME = re.compile(
    r"#\d+\s+(?:0x[0-9a-fA-F]+\s+in\s+)?(?P<func>[\w.]+)\s*\([^)]*\)\s+at\s+"
    r"(?P<file>[\w./-]+):(?P<line>\d+)")
_ASSERT = re.compile(
    r"(?P<file>[\w./-]+\.[ch](?:pp)?):(?P<line>\d+):\s*(?P<func>\w+):\s*"
    r"Assertion\s+[`']?(?P<cond>[^'`]+)")
_GTEST_LOC = re.compile(r"(?P<file>[\w./-]+\.(?:cpp|cc|c)):(?P<line>\d+):\s*Failure")
_GTEST_NAME = re.compile(r"\[\s*FAILED\s*\]\s*(?P<suite>\w+)\.(?P<test>\w+)")
_SIGNAL = re.compile(r"(SIGSEGV|SIGABRT|SIGFPE|SIGBUS|Segmentation fault|core dumped)")


def _parse_gdb(raw):
    frames = [StackFrame(file=m.group("file"), line=int(m.group("line")),
                         function=m.group("func")) for m in _GDB_FRAME.finditer(raw)]
    sig = _SIGNAL.search(raw)
    return ErrorSignature(exception_type=sig.group(1) if sig else "crash",
                          message=raw.strip().splitlines()[0][:200],
                          frames=frames, correlation_ids=_CORR_ID.findall(raw), raw=raw)

def _parse_assert(raw):
    m = _ASSERT.search(raw)
    frames = [StackFrame(file=m.group("file"), line=int(m.group("line")),
                         function=m.group("func"))] if m else []
    return ErrorSignature(exception_type="AssertionFailed",
                          message=(m.group("cond") if m else raw.strip().splitlines()[-1]),
                          frames=frames, raw=raw)

def _gtest_source_guess(suite: str) -> str | None:
    # "NcmAclTest" / "TestNcmAcl" -> "ncm_acl.c"  (best-effort heuristic)
    import re as _re
    words = _re.findall(r"[A-Z][a-z0-9]*", suite)
    words = [w for w in words if w.lower() != "test"]
    return ("_".join(w.lower() for w in words) + ".c") if words else None

def _parse_gtest(raw):
    loc = _GTEST_LOC.search(raw)
    name = _GTEST_NAME.search(raw)
    frames = []
    if loc:
        frames.append(StackFrame(file=loc.group("file"), line=int(loc.group("line")),
                                 function=(name.group("test") if name else "test")))
    guess = _gtest_source_guess(name.group("suite")) if name else None
    msg = f"GoogleTest failure: {name.group('suite')+'.'+name.group('test') if name else 'unknown'}"
    if guess:
        msg += f" (likely source: {guess})"
    return ErrorSignature(exception_type="GTestFailure", message=msg,
                          frames=frames, raw=raw)


def normalize(raw_error: str) -> ErrorSignature:
    raw_error = redact(raw_error)
    if "Traceback (most recent call last)" in raw_error:
        return _parse_python(raw_error)
    if _GTEST_LOC.search(raw_error) or "[  FAILED  ]" in raw_error:
        return _parse_gtest(raw_error)
    if _ASSERT.search(raw_error):
        return _parse_assert(raw_error)
    if _GDB_FRAME.search(raw_error) or _SIGNAL.search(raw_error):
        return _parse_gdb(raw_error)
    last = raw_error.strip().splitlines()[-1] if raw_error.strip() else ""
    return ErrorSignature(message=last, correlation_ids=_CORR_ID.findall(raw_error),
                          raw=raw_error)
```

**Add to `test_normalizer.py`:**
```python
def test_parses_gdb_backtrace():
    raw = ("#0  0x000055ad in ncm_apply_discount (price=99, percent=150) at ncm_sample.c:6\n"
           "#1  0x000055bd in main () at ncm_main.c:42")
    sig = normalize(raw)
    assert sig.frames[0].file == "ncm_sample.c"
    assert sig.frames[0].line == 6
    assert sig.frames[0].function == "ncm_apply_discount"

def test_parses_c_assert():
    raw = "ncm: ncm_sample.c:6: ncm_apply_discount: Assertion `percent <= 100' failed."
    sig = normalize(raw)
    assert sig.exception_type == "AssertionFailed"
    assert sig.frames[0].line == 6

def test_parses_gtest_failure():
    raw = ("../Test/ut/src/test_ncm_acl.cpp:42: Failure\n"
           "[  FAILED  ] NcmAclTest.ApplyRule (0 ms)")
    sig = normalize(raw)
    assert sig.exception_type == "GTestFailure"
    assert sig.frames[0].file.endswith("test_ncm_acl.cpp")
    assert "ncm_acl.c" in sig.message  # heuristic source guess
```

**Run:** `pytest tests/test_normalizer.py -v` → PASS.

**Commit:** `git commit -am "feat: C error normalizers — gdb, assert, GoogleTest, signal"`

---

## Change 2.3 / 2.4 — `analyzer.py`: C-aware prompt + use Gauss `systemPrompt` field

Two improvements, both informed by your LLM doc (the API has a dedicated `systemPrompt` field and a `seed` for reproducibility — use them).

**File:** `backend/app/analysis/analyzer.py`

1. **Replace the system prompt** (C-aware; remove the "Qwen3" wording):
```python
_SYSTEM_PROMPT = (
    "You are a log-analysis assistant for operators of a C networking daemon (NCM). "
    "The codebase is C with C++ (GoogleTest) unit tests. Errors may be gdb backtraces, "
    "assertion failures, GoogleTest failures, or signal crashes (SIGSEGV). "
    "Use ONLY the provided code, log, and design-doc context. Cite real file:line ranges "
    "and real log lines. Never invent file names or line numbers. If unsure, say so. "
    "Respond strictly as a single raw JSON object — no markdown fences."
)
```

2. **`build_prompt` returns only the user content** (drop the system prompt from the top — it now goes in its own field):
```python
def build_prompt(sig: ErrorSignature, hits) -> str:
    return (
        f"## ERROR INPUT\n{sig.raw}\n\n"
        f"## CODE CONTEXT\n{_format_docs(hits['code'], 'code')}\n\n"
        f"## LOG CONTEXT\n{_format_docs(hits['log'], 'log')}\n\n"
        f"## DESIGN DOC CONTEXT\n{_format_docs(hits['doc'], 'doc')}\n\n"
        "## TASK\nReturn a JSON object with keys: root_cause (string), "
        "suspect_files (array of {path, start_line, end_line, reason, confidence 0-1}), "
        "log_representation ({template, where_to_grep, surrounding_meaning}), "
        "next_checks (array of strings)."
    )
```

3. **Pass `systemPrompt` + `seed`, and handle `FILTER_INVALID`** in `_call_gauss`:
```python
def _call_gauss(user_content: str, system_prompt: str) -> str:
    body = {
        "modelIds": [settings.GAUSS_MODEL_ID],
        "contents": [user_content],
        "systemPrompt": system_prompt,
        "isStream": False,
        "llmConfig": {"max_new_tokens": 1024, "temperature": 0.1,
                      "top_p": 0.9, "repetition_penalty": 1.04, "seed": 42},
    }
    resp = requests.post(f"{settings.GAUSS_ENDPOINT_URL}/openapi/chat/v1/messages",
                         headers=_gauss_headers(), json=body, timeout=120)
    resp.raise_for_status()
    data = resp.json()
    if data.get("status") == "FILTER_INVALID":
        raise RuntimeError("Gauss content filter blocked the request (FILTER_INVALID). "
                           "Redact sensitive log content or rephrase the error input.")
    if data.get("status") != "SUCCESS":
        raise RuntimeError(f"Gauss error: {data.get('status')} / {data.get('responseCode')}")
    return data["content"]
```

4. **`analyze` passes both:**
```python
def analyze(sig, hits) -> IncidentReport:
    user_content = build_prompt(sig, hits)
    last = None
    for attempt in range(3):
        try:
            raw = _call_gauss(user_content, _SYSTEM_PROMPT)
            return IncidentReport(**_extract_json(raw))
        except Exception as e:
            last = e
            time.sleep(2)
    raise RuntimeError(f"analyzer failed after 3 attempts: {last}") from last
```

**Run:** `pytest tests/test_analyzer.py -v` (prompt tests still pass; update the prompt test if it asserted the old system text). Live: `pytest tests/test_analyzer.py -v -m integration`.

**Commit:** `git commit -am "feat: C-aware analyzer prompt + Gauss systemPrompt/seed + FILTER_INVALID handling"`

---

## Change 2.7 / 2.8 — fix stale wording (carried over from the Ollama→Gauss switch)

Pure text edits — no behavior change:
- Phase 2.7 step 2 comment: `# Ollama must be running` → `# Gauss credentials must be set in .env`.
- Phase 2.8: "integration tests require Ollama" / "with Ollama running" → "require valid Gauss credentials".
- Any commit message / heading still saying "Qwen3" → "the model".

---

## Final verification before Phase 3

```bash
cd backend && pytest -v            # all unit tests green
pytest -v -m integration           # retriever/analyzer/pipeline green (needs Gauss creds)
```
Then manually, against a **real slice of NCM** (point ingest at `NCM/NCMMAIN`):
```bash
uvicorn app.main:app --port 8080 &
curl -X POST localhost:8080/ingest -H "Content-Type: application/json" \
  -d '{"repo":"/path/to/NCM/NCMMAIN","logs":"/path/to/ncm.log","doc":"/path/to/design.md"}'
# expect non-zero call_sites (proves your LOG_FUNCTION_HINTS caught the macros)
```
- [ ] `call_sites` > 0 on the real repo → your logging-macro hints are correct
- [ ] A gdb/assert error returns suspect files pointing at the right `.c`
- [ ] Log parser recognizes your real format (level populated, not all null)

If `call_sites` is 0 → your `LOG_FUNCTION_HINTS` don't match NCM's macros: re-grep and fix Change 0-A.

➡️ Then proceed to `phase3-onwards-changes.md`.
