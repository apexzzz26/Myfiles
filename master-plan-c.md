# Master Plan (C / NCM) — Phase 3 → 5

> **This is the single source of truth from Phase 3 onward.** All the C/NCM patches are already folded in — you do **not** need `phase3-onwards-changes.md` anymore (it's superseded by this file).
>
> **Starting point assumed:** you've completed Phases 0–2 of `master-plan.md` **and** applied every change in `c-codebase-changes-before-phase3.md` (MD1): C-aware `code_indexer.py`, C error parsers in `normalizer.py`, pluggable `log_indexer.py`, NCM sample data, and the C-aware Gauss analyzer.

## Prerequisite gate (confirm before starting Phase 3)

- [ ] `cd backend && pytest -v` → all unit tests green
- [ ] Ingest the **real** `NCM/NCMMAIN` → `call_sites > 0` (proves your `LOG_FUNCTION_HINTS` match NCM's logging macros — the WOW feature dies without this)
- [ ] A gdb/assert error through `/analyze` returns suspect files pointing at a real `.c`

> **Note on `redact()`:** it's formally introduced in **Phase 4.3** below. The MD1 C-normalizer already calls `redact()` at the top of `normalize()`. If you applied MD1 but haven't done 4.3 yet, add a temporary stub so imports don't break:
> ```python
> def redact(text): return text   # replaced for real in Phase 4.3
> ```

---

---

# PHASE 3 — Web App + WOW

**Goal:** A browser interface — paste an error, get a grounded report, and **click any log line to jump to the exact C source that printed it**.

---

## Phase 3.1 — SQLite persistence (history + feedback)

**Files created:** `backend/app/db.py` · **Modified:** `backend/app/main.py`

1. **`backend/app/db.py`:**
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
        cur = c.execute("INSERT INTO reports(error, result) VALUES(?, ?)",
                        (error, json.dumps(result)))
        return cur.lastrowid

def list_reports() -> list[dict]:
    with _conn() as c:
        rows = c.execute(
            "SELECT id, error, result, created FROM reports ORDER BY id DESC LIMIT 50"
        ).fetchall()
    return [{"id": r[0], "error": r[1], "result": json.loads(r[2]), "created": r[3]}
            for r in rows]

def save_feedback(report_id: int, true_file: str | None, worked: bool, note: str = "") -> None:
    with _conn() as c:
        c.execute("INSERT INTO feedback(report_id, true_file, worked, note) VALUES(?,?,?,?)",
                  (report_id, true_file, int(worked), note))
```

2. **Wire into `main.py`:**
```python
from app import db
db.init_db()   # at module load

# inside analyze_route, before return:
report_id = db.save_report(req.error, result)
result["report_id"] = report_id
```

3. **Verify:** call `/analyze` twice → `sqlite3 analyzer.db "SELECT id, error FROM reports;"` → two rows.

**Commit:** `git commit -am "feat: SQLite history and feedback persistence"`

---

## Phase 3.2 — Back-map (log line → C source call-site) — the WOW logic

**Goal:** Given a raw log line, return the `CallSite` (`file:line`) that emitted it. **Tuned for C:** printf specifiers (`%d %s %llu`) are stripped before matching, the threshold is 0.5, and ties break on the longest match.

**Files created:** `backend/app/analysis/backmap.py` · **Test:** `backend/tests/test_backmap.py`

1. **Write the failing test (C sample):**
```python
# backend/tests/test_backmap.py
import pytest
from tests.conftest import SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC

@pytest.mark.integration
def test_backmaps_ncm_log_to_source():
    from app.indexing.store import build_index
    from app.analysis.backmap import backmap
    idx = build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)
    cs = backmap(idx, "Invalid discount percent: 150 for price 99")
    assert cs is not None
    assert cs.file.endswith("ncm_sample.c")
    assert cs.line == 6

@pytest.mark.integration
def test_backmap_returns_none_for_unrelated_text():
    from app.indexing.store import build_index
    from app.analysis.backmap import backmap
    idx = build_index(SAMPLE_REPO, SAMPLE_LOGS, SAMPLE_DOC)
    assert backmap(idx, "completely unrelated text matching nothing") is None
```

2. **Implement `backend/app/analysis/backmap.py`:**
```python
import re
from app.models import CallSite

_NOISE = {"none", "null", "true", "false", "nan", "the", "for", "and",
          "with", "from", "err", "error", "warn", "info", "debug"}
# printf/format specifiers: %d %s %-10.2f %llu %p etc.
_FMT_SPEC = re.compile(r"%[-+ #0]*\d*(?:\.\d+)?(?:hh|h|ll|l|L|z|j|t|q)?[diouxXeEfgGaAcspn%]")

def _static_tokens(text: str) -> set[str]:
    text = _FMT_SPEC.sub(" ", text)              # drop %d/%s/... FIRST
    return {w.lower() for w in re.findall(r"[A-Za-z_]{3,}", text)
            if w.lower() not in _NOISE}

def backmap(index, log_line: str, threshold: float = 0.5) -> CallSite | None:
    target = _static_tokens(log_line)
    if not target:
        return None
    best, best_score, best_overlap = None, 0.0, 0
    for cs in index.call_sites:
        toks = _static_tokens(cs.format_string)
        if not toks:
            continue
        overlap = len(target & toks)
        score = overlap / len(toks)              # fraction of the format matched
        if score > best_score or (score == best_score and overlap > best_overlap):
            best, best_score, best_overlap = cs, score, overlap
    return best if best_score >= threshold else None
```

3. **Run:** `pytest tests/test_backmap.py -v -m integration` → PASS.

> **Known limit:** if two call-sites share an *identical* format string, text alone can't disambiguate — Phase 5.1's bounded agent can break ties using the correlation ID / surrounding log lines.

**Commit:** `git commit -am "feat: log-line→C call-site back-map (printf-aware, WOW core)"`

---

## Phase 3.3 — Additional API endpoints

**Modified:** `backend/app/main.py` — add `/backmap`, `/incidents`, `/feedback`.

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

**Verify:**
```bash
curl -X POST localhost:8080/backmap -H "Content-Type: application/json" \
  -d '{"log_line": "Invalid discount percent: 150 for price 99"}'
# expect {"file":".../ncm_sample.c","line":6,...}
curl localhost:8080/incidents
```

**Commit:** `git commit -am "feat: /backmap, /incidents, /feedback endpoints"`

---

## Phase 3.4 — Frontend scaffold

```bash
cd log-analyzer-agent
npm create vite@latest frontend -- --template react-ts
cd frontend && npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```
*(If you land on Tailwind v4, follow its Vite guide instead — it uses `@tailwindcss/postcss`.)*

`tailwind.config.js`:
```js
export default {
  content: ["./index.html", "./src/**/*.{ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
}
```
`src/index.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```
Verify: `npm run dev` → React page at `http://localhost:5173`.

**Commit:** `git commit -am "chore: Vite React-TS + Tailwind scaffold"`

---

## Phase 3.5 — API client (`api.ts`)

**Create `frontend/src/api.ts`:**
```typescript
const BASE = "http://localhost:8080";

export async function analyze(error: string) {
  const r = await fetch(`${BASE}/analyze`, {
    method: "POST", headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ error }) });
  return r.json();
}

export async function backmap(logLine: string) {
  const r = await fetch(`${BASE}/backmap`, {
    method: "POST", headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ log_line: logLine }) });
  return r.json();
}

export async function getIncidents() {
  const r = await fetch(`${BASE}/incidents`);
  return r.json();
}

export async function sendFeedback(reportId: number, trueFile: string, worked: boolean) {
  const r = await fetch(`${BASE}/feedback`, {
    method: "POST", headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ report_id: reportId, true_file: trueFile, worked }) });
  return r.json();
}
```

**Commit:** `git commit -am "feat: typed API client"`

---

## Phase 3.6 — ErrorInput component

**Create `frontend/src/components/ErrorInput.tsx`:**
```tsx
import { useState } from "react";

interface Props { onAnalyze: (error: string) => void; loading: boolean; }

export function ErrorInput({ onAnalyze, loading }: Props) {
  const [error, setError] = useState("");
  return (
    <div className="space-y-3">
      <label className="block text-sm font-medium text-gray-700">
        Paste your error (gdb backtrace, assertion, GoogleTest failure) + log lines
      </label>
      <textarea
        className="w-full h-48 p-3 font-mono text-sm border rounded-lg
                   focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"
        placeholder={"#0 0x... in ncm_apply_acl (...) at ncm_acl.c:142\n#1 ..."}
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

## Phase 3.7 — IncidentReport + SuspectFileCard

**Create `frontend/src/components/SuspectFileCard.tsx`** (shows basename prominently, full nested NCM path on hover):
```tsx
interface SuspectFile {
  path: string; start_line: number; end_line: number;
  reason: string; confidence: number;
}

export function SuspectFileCard({ file, rank }: { file: SuspectFile; rank: number }) {
  const pct = Math.round(file.confidence * 100);
  const bar = pct >= 70 ? "bg-red-500" : pct >= 40 ? "bg-yellow-500" : "bg-gray-400";
  const base = file.path.split("/").pop();   // NCM paths are deeply nested
  return (
    <div className="border rounded-lg p-4 bg-white shadow-sm">
      <div className="flex items-start justify-between gap-2">
        <div>
          <span className="text-xs text-gray-400 mr-2">#{rank}</span>
          <span className="font-mono text-sm font-semibold text-blue-700" title={file.path}>
            {base}
          </span>
          <span className="text-xs text-gray-500 ml-2">
            lines {file.start_line}–{file.end_line}
          </span>
        </div>
        <span className="text-xs font-medium text-gray-600 whitespace-nowrap">{pct}%</span>
      </div>
      <div className="mt-2 h-1.5 w-full bg-gray-100 rounded-full">
        <div className={`h-1.5 rounded-full ${bar}`} style={{ width: `${pct}%` }} />
      </div>
      <p className="mt-2 text-sm text-gray-700">{file.reason}</p>
    </div>
  );
}
```

**Create `frontend/src/components/IncidentReport.tsx`:**
```tsx
import { SuspectFileCard } from "./SuspectFileCard";
import { LogTimeline } from "./LogTimeline";

interface Props { report: any; warnings: string[]; }

export function IncidentReport({ report, warnings }: Props) {
  if (!report) return null;
  const logLines = report.log_representation?.where_to_grep
    ? [report.log_representation.where_to_grep] : [];
  const lowConfidence = report.suspect_files?.every((f: any) => f.confidence < 0.3);

  return (
    <div className="space-y-6 mt-6">
      {warnings.length > 0 && (
        <div className="bg-yellow-50 border border-yellow-300 rounded-lg p-3 text-sm text-yellow-800">
          ⚠ {warnings.join(" · ")}
        </div>
      )}
      {lowConfidence && (
        <div className="bg-orange-50 border border-orange-200 rounded-lg p-3 text-sm text-orange-800">
          No high-confidence suspects found. Try ingesting more of the codebase.
        </div>
      )}

      <section>
        <h2 className="text-lg font-semibold mb-1">Root Cause</h2>
        <p className="text-gray-800 leading-relaxed">{report.root_cause}</p>
      </section>

      <section>
        <h2 className="text-lg font-semibold mb-2">Suspect Files</h2>
        <div className="space-y-2">
          {report.suspect_files?.map((f: any, i: number) =>
            <SuspectFileCard key={i} file={f} rank={i + 1} />)}
        </div>
      </section>

      <section>
        <h2 className="text-lg font-semibold mb-1">How It Appears in Logs</h2>
        <div className="bg-gray-900 text-green-300 rounded-lg p-3 font-mono text-sm mb-2">
          {report.log_representation?.template}
        </div>
        <p className="text-sm text-gray-600">
          <strong>Grep for:</strong> <code>{report.log_representation?.where_to_grep}</code>
        </p>
        <p className="text-sm text-gray-600 mt-1">{report.log_representation?.surrounding_meaning}</p>
      </section>

      <LogTimeline lines={logLines} />

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

**Create `frontend/src/components/LogTimeline.tsx`** — click a line → `/backmap` → show the C source that printed it.

```tsx
import { useState } from "react";
import { backmap } from "../api";

interface CallSite { file: string; line: number; text: string; format_string: string; }
interface Props { lines: string[]; }

export function LogTimeline({ lines }: Props) {
  const [hitMap, setHitMap] = useState<Record<number, CallSite | null>>({});
  const [loading, setLoading] = useState<number | null>(null);

  async function handleClick(line: string, idx: number) {
    if (hitMap[idx] !== undefined) return;
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
          click a line to see the C source that printed it
        </span>
      </h2>
      <div className="rounded-lg border overflow-hidden">
        {lines.map((ln, i) => (
          <div key={i}>
            <pre onClick={() => handleClick(ln, i)}
                 className={`px-4 py-2 text-xs font-mono cursor-pointer transition-colors
                   ${loading === i ? "bg-blue-50 animate-pulse" : "hover:bg-yellow-50"}`}>
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

**Wire `frontend/src/App.tsx`:**
```tsx
import { useState } from "react";
import { analyze } from "./api";
import { ErrorInput } from "./components/ErrorInput";
import { IncidentReport } from "./components/IncidentReport";

export default function App() {
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<any>(null);

  async function handleAnalyze(error: string) {
    setLoading(true); setResult(null);
    setResult(await analyze(error)); setLoading(false);
  }

  return (
    <div className="max-w-3xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold mb-6">NCM Log Analyzer Agent</h1>
      <ErrorInput onAnalyze={handleAnalyze} loading={loading} />
      {result && <IncidentReport report={result.report} warnings={result.warnings ?? []} />}
    </div>
  );
}
```

**Commit:** `git commit -am "feat: React UI + clickable log-to-C-source timeline (WOW)"`

---

## Phase 3.9 — Full browser verification (C / NCM)

1. Start backend (`uvicorn app.main:app --port 8080`) + frontend (`npm run dev`), open `http://localhost:5173`.
2. **Golden path:**
   - [ ] Ingest the real `NCM/NCMMAIN` + an NCM log + the design doc
   - [ ] Paste a gdb backtrace (e.g. `... at ncm_acl.c:142`) → click Analyze → report appears
   - [ ] A suspect file shows a real `ncm_*.c`
   - [ ] Click an `NCM_LOG_*` log line in the timeline → jumps to the exact macro call in `ncm_*.c` (the WOW)
   - [ ] Confidence bar color matches the value
3. **Edge cases:** empty input → button disabled; gibberish → low-confidence banner / warnings.

**Milestone:** browser shows a grounded report; clicking a log line reveals its C source.

---

---

# PHASE 4 — Eval, Hardening & Demo

---

## Phase 4.1 — Labeled eval dataset (C / NCM)

**Create `backend/eval/incidents.jsonl`** — real NCM error shapes. For GoogleTest failures, `true_file` is the **source under test**, not the test file:
```json
{"error": "#0 0x55ad in ncm_apply_acl (r=0x0) at ncm_acl.c:142\n#1 in ncm_main () at ncm_main.c:88", "true_file": "ncm_acl.c"}
{"error": "ncm: ncm_qos.c:90: ncm_qos_set: Assertion `q != NULL' failed.", "true_file": "ncm_qos.c"}
{"error": "../Test/ut/src/test_ncm_nat.cpp:53: Failure\n[  FAILED  ] NcmNatTest.Translate", "true_file": "ncm_nat.c"}
{"error": "SIGSEGV in ncm_netlink_recv at ncm_netlink.c:201", "true_file": "ncm_netlink.c"}
```
Collect ≥10 from NCM git history / the `Test/ut` suite / crash logs. Verify the `_gtest_source_guess` heuristic against your real suite naming and adjust if needed.

**Commit:** `git commit -am "feat: C/NCM eval dataset"`

---

## Phase 4.2 — Eval harness

**Create `backend/eval/run_eval.py`:**
```python
import json, time, sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))
from app.indexing.store import build_index
from app.analysis.pipeline import run_analysis

def hit_at_k(suspects, true_file, k):
    return any(true_file in s["path"] or s["path"].endswith(true_file)
               for s in suspects[:k])

def main(repo, logs, doc, dataset="eval/incidents.jsonl"):
    idx = build_index(repo, logs, doc)
    cases = [json.loads(l) for l in open(dataset)]
    top1 = top3 = 0; lat = []
    for c in cases:
        t0 = time.time(); out = run_analysis(idx, c["error"]); lat.append(time.time() - t0)
        s = out["report"].get("suspect_files", [])
        h1, h3 = hit_at_k(s, c["true_file"], 1), hit_at_k(s, c["true_file"], 3)
        top1 += h1; top3 += h3
        print(f"  [{'✓' if h1 else '~' if h3 else '✗'}] {c['true_file'][:40]:<40} {lat[-1]:.1f}s")
    n = len(cases); lat.sort()
    print(f"\n{'─'*50}")
    print(f"  top-1 hit rate : {top1/n:.0%}  ({top1}/{n})")
    print(f"  top-3 hit rate : {top3/n:.0%}  ({top3}/{n})")
    print(f"  median latency : {lat[n//2]:.1f}s")

if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2], sys.argv[3])
```
Run, then tune `TOP_K` / BM25 weights / `MAX_CHUNK_LINES` until top-3 ≥ 70%.

**Commit:** `git commit -am "feat: eval harness — file-localization hit rate + latency"`

---

## Phase 4.3 — PII / secret redaction (PII + network + IPSec)

**Modified:** `backend/app/analysis/normalizer.py` — add `redact()` with the full pattern set (NCM is a networking daemon with an IPSec module; data now leaves to the Gauss API).

```python
_REDACT_PATTERNS = [
    (re.compile(r"[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z]{2,}"), "[EMAIL]"),
    (re.compile(r"(?:Bearer|Token|token|key|secret)\s*[=:]\s*[\w.\-/+]{8,}", re.I), "[CREDENTIAL]"),
    (re.compile(r"\b[0-9a-fA-F]{32,}\b"), "[HEX_ID]"),
    (re.compile(r"eyJ[\w-]+\.[\w-]+\.[\w-]+"), "[JWT]"),
    # network / IPSec
    (re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b"), "[IPV4]"),
    (re.compile(r"\b(?:[0-9a-fA-F]{1,4}:){2,7}[0-9a-fA-F]{1,4}\b"), "[IPV6]"),
    (re.compile(r"\b(?:[0-9a-fA-F]{2}[:-]){5}[0-9a-fA-F]{2}\b"), "[MAC]"),
    (re.compile(r"(?i)\b(psk|pre-?shared-?key|secret|spi)\b\s*[=:]\s*\S+"), r"\1=[REDACTED]"),
]

def redact(text: str) -> str:
    for pattern, replacement in _REDACT_PATTERNS:
        text = pattern.sub(replacement, text)
    return text
```
Ensure `normalize()` calls `redact(raw_error)` at the top (the MD1 C-normalizer already does).

> **Trade-off:** redacting all IPs removes debugging signal. If your mentor confirms internal logs are safe for Gauss, drop the IPV4/IPV6/MAC patterns and keep only `psk/secret/spi`. Write the decision in the README.

**Test:**
```python
def test_redacts_ipsec_psk():
    sig = normalize("ncm_ipsec: tunnel up psk=SuperSecret123 spi=0xabcd")
    assert "SuperSecret123" not in sig.raw
    assert "[REDACTED]" in sig.raw
```

**Commit:** `git commit -am "feat: PII + network + IPSec redaction"`

---

## Phase 4.4 — Robustness & edge cases

1. **Empty-index guards** — present in `/analyze` and `/backmap`.
2. **Low-confidence banner** — already in `IncidentReport.tsx` (Phase 3.7).
3. **LLM retry** — built into `analyze()` (3 attempts, 2s back-off) from MD1.
4. **C parser-miss check** — add one eval case from a `.c` file that uses macro-wrapped signatures (`STATIC void foo()`); confirm the `_fallback_chunks` path still lets retrieval find it (graceful degradation).

**Commit:** `git commit -am "feat: robustness checks incl. C macro-miss fallback"`

---

## Phase 4.5 — Performance

1. **Reuse the persisted Chroma store** across restarts (`load_index` in `store.py`); rebuild only on `/ingest`.
```python
def load_index(repo, logs, doc):
    from pathlib import Path
    if Path(settings.CHROMA_DIR).exists():
        _, call_sites = index_code(repo)
        events = index_logs(logs)
        store = Chroma(persist_directory=settings.CHROMA_DIR, embedding_function=_get_embeddings())
        return Index(store=store, all_docs=[], call_sites=call_sites, log_events=events)
    return build_index(repo, logs, doc)
```
2. **Embeddings singleton** (`_get_embeddings()`) already in `store.py` — confirm the MiniLM model loads at most once per process.

**Commit:** `git commit -am "perf: Chroma persistence + embeddings singleton"`

---

## Phase 4.6 — Packaging (`start.sh`, no Docker)

**Create `start.sh`:**
```bash
#!/usr/bin/env bash
set -e
echo "Starting Chroma..."; chroma run --path ./chroma_store & CHROMA_PID=$!
echo "Starting API...";    (cd backend && uvicorn app.main:app --host 0.0.0.0 --port 8080) & API_PID=$!
echo "Starting frontend...";(cd frontend && npm run preview -- --port 3000) & WEB_PID=$!
trap "kill $CHROMA_PID $API_PID $WEB_PID 2>/dev/null" EXIT
echo "Up. Ctrl+C to stop."; wait
```
```bash
chmod +x start.sh && (cd frontend && npm run build)
```
Verify: `./start.sh` → app at `http://localhost:3000`.

**Commit:** `git commit -am "feat: start.sh one-command bring-up"`

---

## Phase 4.7 — README & demo script (NCM)

1. **`README.md`:** description; prerequisites (Python 3.11, Node 18+, Gauss credentials); setup (`pip install`, copy `.env.example`→`.env`, `npm install`, `./start.sh`); usage (`/ingest` paths → `/analyze` error); the WOW; eval; architecture diagram; **your redaction decision**.
2. **Demo script (rehearse to <5 min):**
   1. "Ingest the NCM module (`NCM/NCMMAIN`) + a syslog capture + the design doc."
   2. "Paste a gdb backtrace from a SIGSEGV."
   3. "Agent points to `ncm_netlink.c:201` with the reason, grounded in the stack frame + the code."
   4. "Click the `NCM_LOG_ERR` log line → jumps to the exact macro call in `ncm_acl.c`."
   5. "Eval: top-3 file hit-rate across N real NCM incidents = X%. Next: bounded multi-step agent for multi-hop bugs."

**Commit:** `git commit -am "docs: NCM README + demo script" && git tag v1.0`

---

## Phase 4.8 — Final demo dry-run

- [ ] `./start.sh` from a clean checkout — starts with no errors
- [ ] Ingest real NCM slice — completes; `call_sites > 0`
- [ ] Analyze a gdb/assert/GoogleTest error → grounded report
- [ ] Click a log line → C source appears (WOW)
- [ ] `python eval/run_eval.py <repo> <logs> <doc>` → prints metrics
- [ ] `pytest -v` → all unit tests pass
- [ ] Demo rehearsed < 5 min

**Milestone:** polished demo + accuracy metric + "where it goes next" story.

---

---

# PHASE 5 — Stretch (post-internship)

Describe these in the demo even if not built.

## Phase 5.1 — Bounded multi-step agent (ReAct, capped)
Let the model issue a few tool calls before answering, for multi-hop bugs — very common in C daemons (exception in `ncm_netlink.c`, real cause up the `cps_message → ncm_main → handler` chain).
- Tools: `search_code(query)`, `read_file(path, start, end)`, `search_logs(query)`, `get_doc(query)` — each wraps existing retriever/store methods on the C index.
- Loop: `model → tool call → result → … (≤5 steps) → IncidentReport`.
- Also resolves back-map ties (identical format strings) using the correlation ID / surrounding log lines.
- **Only here** does a ReAct loop / LangGraph earn its place.

## Phase 5.2 — Self-learning incident memory
On feedback `worked=true`, distill `{error_signature → true_file → fix summary}` into an "incident card", embed it, and check it before calling the model (similarity > 0.9 → instant cached answer with a link to the prior NCM incident).

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
| Ingest NCM | `curl -X POST localhost:8080/ingest -H 'Content-Type: application/json' -d '{"repo":"/path/NCM/NCMMAIN","logs":"/path/ncm.log","doc":"/path/design.md"}'` |
| Analyze an error | `curl -X POST localhost:8080/analyze -H 'Content-Type: application/json' -d '{"error":"<paste here>"}'` |
| Back-map a log line | `curl -X POST localhost:8080/backmap -H 'Content-Type: application/json' -d '{"log_line":"<paste>"}'` |
