# Building AI Agents: A Complete Developer Guide

> A practical, end-to-end guide for designing and building intelligent AI agents for any domain — from network management tools to customer support bots and beyond.

---

## Table of Contents

1. [What is an AI Agent?](#1-what-is-an-ai-agent)
2. [Before You Write a Single Line of Code](#2-before-you-write-a-single-line-of-code)
3. [Architecture Design](#3-architecture-design)
4. [Layer 1 — Data Collection & Integration](#4-layer-1--data-collection--integration)
5. [Layer 2 — Parsing & Structuring Data](#5-layer-2--parsing--structuring-data)
6. [Layer 3 — Analysis & Decision Logic](#6-layer-3--analysis--decision-logic)
7. [Layer 4 — Intent Recognition](#7-layer-4--intent-recognition)
8. [Layer 5 — LLM Integration](#8-layer-5--llm-integration)
9. [Layer 6 — Persistent Storage](#9-layer-6--persistent-storage)
10. [Layer 7 — Presentation (Chat / CLI / API)](#10-layer-7--presentation-chat--cli--api)
11. [Configuration Management](#11-configuration-management)
12. [Error Handling & Fallback Strategy](#12-error-handling--fallback-strategy)
13. [Testing Strategy](#13-testing-strategy)
14. [Security Considerations](#14-security-considerations)
15. [Deployment & Operations](#15-deployment--operations)
16. [Common Pitfalls & How to Avoid Them](#16-common-pitfalls--how-to-avoid-them)
17. [Quick Reference Checklist](#17-quick-reference-checklist)

---

## 1. What is an AI Agent?

An **AI Agent** is a software system that can perceive its environment, reason about what it observes, and take actions — either autonomously or in response to user requests — to accomplish a goal.

Unlike a simple chatbot that just generates text, an agent:

- **Connects** to real data sources (APIs, databases, files, network devices, IoT sensors)
- **Understands** what the user wants, even when phrased in natural language
- **Reasons** over structured data to find answers or detect problems
- **Acts** — it can trigger workflows, send alerts, modify state, or call external services
- **Remembers** context across sessions using persistent storage

### The Mental Model

Think of an AI agent as having three core responsibilities:

```
[ Perceive ]  →  [ Reason ]  →  [ Act ]
   ↑                                ↓
   └──────── [ Remember ] ──────────┘
```

| Responsibility | What it means in practice |
|---|---|
| **Perceive** | Collect data from external systems (SSH, REST APIs, files, databases) |
| **Reason** | Parse, analyze, recognize intent, compare against expectations |
| **Act** | Respond in natural language, trigger alerts, write to storage, call APIs |
| **Remember** | Store collected data, alerts, conversation history in a database |

---

## 2. Before You Write a Single Line of Code

This is the phase most developers skip — and it's the reason most agents fail or need expensive rewrites later. Spend real time here.

### 2.1 Define the Problem Clearly

Write down answers to these questions before anything else:

- **What problem does this agent solve?** (Be specific. "Helps with networking" is not a problem statement. "Detects BGP misconfiguration across 50 routers and explains it in plain English" is.)
- **Who are the users?** Are they domain experts or non-technical? This determines how the agent should communicate.
- **What decisions will the agent make vs. what decisions should always stay with a human?**
- **What does success look like?** Define measurable outcomes upfront.

### 2.2 Map Your Data Sources

Every agent needs data. List all your data sources and classify them:

| Type | Examples | Key Concerns |
|---|---|---|
| **Real-time / live** | SSH into devices, REST APIs, IoT sensors | Latency, connectivity, rate limits |
| **Batch / periodic** | Scheduled DB queries, log file ingestion | Freshness, volume |
| **Static / reference** | Config files, topology definitions, product catalogs | Change management, versioning |
| **User-provided** | Chat input, uploaded files, form submissions | Validation, sanitization |

### 2.3 Define Your Intents

An **intent** is what the user wants to accomplish. Make a complete list of every question or action your agent needs to handle.

**Example intent list for a generic monitoring agent:**

```
SHOW_STATUS      → "What is the current status of X?"
SHOW_HISTORY     → "Show me the last 7 days of data for X"
CHECK_ALERTS     → "Are there any active issues?"
GET_DETAILS      → "Tell me more about alert #42"
RUN_DIAGNOSTIC   → "Run a health check on X"
COMPARE          → "How does X compare to Y?"
UNKNOWN          → Anything the agent doesn't understand
```

Write these down. Every intent will eventually map to a specific function or database query. This list becomes your development roadmap.

### 2.4 Define the "Expected State"

Most agents need to know the difference between **normal** and **abnormal**. Before building detection logic, clearly define:

- What does a healthy system look like?
- What are the rules that must always hold true?
- Who defines these rules, and how do they change over time?

This reference definition is often stored as a config file (e.g., `expected_topology.json` in a network agent, or a rules YAML file in a compliance agent).

---

## 3. Architecture Design

Good agents follow a **layered architecture**. Each layer has a single responsibility and communicates only with adjacent layers. This makes the system testable, replaceable, and easy to reason about.

```
┌─────────────────────────────────────────────────────┐
│              PRESENTATION LAYER                      │
│         CLI  |  Chat Interface  |  REST API          │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│              BUSINESS LOGIC LAYER                    │
│  Intent Recognizer  |  Analyzer  |  LLM Client       │
│  Session Manager    |  Parser    |  Alerting          │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│               DATA ACCESS LAYER                      │
│      Database Manager  |  Cache  |  Config Loader    │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│              EXTERNAL SYSTEMS LAYER                  │
│   Data Sources (APIs, SSH, files)  |  LLM API        │
└─────────────────────────────────────────────────────┘
```

### Recommended Project Structure

```
my-agent/
│
├── main.py                    # Entry point
├── requirements.txt
├── .env                       # Secrets (never commit)
├── .env.example               # Template for secrets
│
├── src/
│   ├── collector.py           # Data collection from external sources
│   ├── parser.py              # Raw data → structured data
│   ├── analyzer.py            # Structured data → insights / alerts
│   ├── intent_recognizer.py   # Rule-based intent detection
│   ├── llm_intent_recognizer.py  # LLM-based intent detection
│   ├── llm_client.py          # Wrapper for your LLM API
│   ├── database.py            # All DB read/write operations
│   ├── chat_interface.py      # Orchestrates: intent → DB → LLM → response
│   └── session_manager.py     # Manages multi-turn conversation state
│
├── configs/
│   ├── sources.json           # Connection details for data sources
│   └── expected_state.json    # Reference / ground truth definition
│
└── tests/
    ├── test_parser.py
    ├── test_analyzer.py
    ├── test_intent_recognizer.py
    └── fixtures/              # Sample raw data for tests
```

> **Key principle:** `chat_interface.py` is the orchestration hub. It should contain almost no business logic itself — it just coordinates calls between the intent recognizer, database, LLM client, and response formatter.

---

## 4. Layer 1 — Data Collection & Integration

This is where your agent connects to the outside world.

### 4.1 Design Your Collector

The collector's job is simple: **connect to a source, retrieve raw data, return it**. It should not parse or analyze — that is the parser's job.

```python
class DataCollector:
    def __init__(self, config: dict):
        self.config = config

    def connect(self) -> bool:
        """Establish connection. Return True on success."""
        ...

    def collect(self, command: str) -> str:
        """Execute command/query. Return raw output as string."""
        ...

    def disconnect(self):
        """Always clean up connections."""
        ...
```

### 4.2 Common Collection Patterns

**REST API:**
```python
import requests

def fetch_from_api(endpoint: str, headers: dict) -> dict:
    response = requests.get(endpoint, headers=headers, timeout=10)
    response.raise_for_status()
    return response.json()
```

**SSH (e.g., for the FRR agent, network devices, remote servers):**
```python
import paramiko

def run_ssh_command(host, user, key_path, command):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(host, username=user, key_filename=key_path)
    _, stdout, stderr = client.exec_command(command)
    output = stdout.read().decode()
    client.close()
    return output
```

**File / Log ingestion:**
```python
def read_log_file(path: str, since_timestamp=None) -> list[str]:
    with open(path, 'r') as f:
        lines = f.readlines()
    if since_timestamp:
        lines = [l for l in lines if extract_timestamp(l) >= since_timestamp]
    return lines
```

### 4.3 Handle Multiple Sources

When your agent talks to multiple sources (e.g., 10 servers, 5 APIs), use a consistent config-driven approach:

```json
// configs/sources.json
{
  "sources": [
    {
      "id": "source-01",
      "type": "ssh",
      "host": "192.168.1.10",
      "user": "admin",
      "key_path": "~/.ssh/id_rsa"
    },
    {
      "id": "api-service",
      "type": "rest",
      "base_url": "https://api.example.com",
      "auth_header": "X-API-Key"
    }
  ]
}
```

Loop over sources using the same collector interface so the rest of your code doesn't care what the source type is.

### 4.4 Scheduling Collection

Decide how often data needs to be fresh:

| Freshness Requirement | Strategy |
|---|---|
| Real-time (< 1 second) | WebSocket / event stream |
| Near real-time (seconds) | Polling loop with short interval |
| Periodic (minutes/hours) | Scheduled jobs (cron, APScheduler) |
| On-demand | Collect only when user asks |

Most agent use cases are best served by **periodic background collection** combined with **on-demand collection** for specific user requests.

---

## 5. Layer 2 — Parsing & Structuring Data

Raw data from external sources is almost never in a form your agent can reason over directly. The parser's job is to transform raw strings, HTML, JSON blobs, or log lines into clean Python dictionaries or dataclasses.

### 5.1 The Parser Contract

A parser takes raw text/data in and returns structured data out. It should be:

- **Stateless** — no side effects, just a transformation
- **Testable** — easy to write unit tests with fixture data
- **Defensive** — handle missing fields, malformed input gracefully

```python
def parse_status_output(raw: str) -> dict:
    """
    Input:  Raw string output from a data source
    Output: Structured dict with known fields
    """
    result = {
        "status": None,
        "uptime": None,
        "error": None
    }
    # parsing logic here...
    return result
```

### 5.2 Common Parsing Techniques

**Regex** — for semi-structured text like CLI output, logs:
```python
import re

pattern = re.compile(r'Status:\s+(\w+)\s+Uptime:\s+([\d:]+)')
match = pattern.search(raw_text)
if match:
    result["status"] = match.group(1)
    result["uptime"] = match.group(2)
```

**JSON / XML parsing** — for APIs:
```python
import json

data = json.loads(raw_response)
result["status"] = data.get("system", {}).get("status", "unknown")
```

**Line-by-line parsing** — for log files or tabular output:
```python
for line in raw_text.strip().split('\n'):
    if line.startswith('ERROR'):
        errors.append(line)
```

### 5.3 Define Your Data Schema Early

Before writing parsers, define exactly what your structured data should look like. Use dataclasses or TypedDicts:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class ServiceStatus:
    service_id: str
    status: str               # "UP", "DOWN", "DEGRADED"
    uptime_seconds: int
    last_seen: str            # ISO timestamp
    error_message: Optional[str] = None
```

This schema becomes the contract between your parser layer and your analysis layer.

---

## 6. Layer 3 — Analysis & Decision Logic

The analyzer compares what **is** against what **should be** and generates alerts or insights.

### 6.1 Define Your Alert Severity Model

Before writing any detection logic, define your severity levels and what each means in your domain:

| Severity | Meaning | Example |
|---|---|---|
| **CRITICAL** | Immediate impact, requires urgent action | Service completely down |
| **HIGH** | Likely causing problems, needs prompt fix | Configuration mismatch detected |
| **MEDIUM** | May or may not be intentional, worth investigating | Unexpected component present |
| **LOW** | Informational, no immediate action needed | Non-standard setting in use |
| **INFO** | Status update, not a problem | Scheduled maintenance window |

### 6.2 The Analyzer Pattern

```python
class Analyzer:
    def __init__(self, expected_state: dict):
        self.expected = expected_state

    def analyze(self, actual_data: dict, source_id: str) -> list[dict]:
        """
        Compare actual vs expected state.
        Returns a list of alert dicts.
        """
        alerts = []
        alerts.extend(self._check_availability(actual_data, source_id))
        alerts.extend(self._check_configuration(actual_data, source_id))
        alerts.extend(self._check_relationships(actual_data, source_id))
        return alerts

    def _check_availability(self, data, source_id) -> list[dict]:
        alerts = []
        if data.get("status") == "DOWN":
            alerts.append({
                "source": source_id,
                "type": "AVAILABILITY",
                "severity": "CRITICAL",
                "message": f"{source_id} is unreachable",
                "details": data
            })
        return alerts
```

### 6.3 Load Expected State from Config

Never hardcode expected values in your analysis logic. Always load from a config file:

```json
// configs/expected_state.json
{
  "services": {
    "auth-service": {
      "expected_status": "UP",
      "min_uptime_seconds": 3600,
      "required_connections": ["db-primary", "cache-01"]
    }
  }
}
```

```python
with open("configs/expected_state.json") as f:
    expected_state = json.load(f)

analyzer = Analyzer(expected_state)
```

This way, when the expected topology or rules change, you update a config file — not the code.

---

## 7. Layer 4 — Intent Recognition

Intent recognition is the bridge between the user's natural language and your agent's internal capabilities. It answers: **"What does the user want to do?"**

### 7.1 Two Approaches (Use Both)

**Approach A — Rule-Based (Regex):**
Fast, predictable, no external dependency. Use this as your primary or fallback method.

```python
import re

INTENT_PATTERNS = [
    {
        "intent": "SHOW_STATUS",
        "patterns": [
            r"\b(show|what is|get|check)\b.*(status|health|state)\b",
            r"\bstatus of\b"
        ],
        "priority": 1
    },
    {
        "intent": "CHECK_ALERTS",
        "patterns": [
            r"\b(any|show|list).*(alert|issue|problem|error)\b",
            r"\bwhat.*(wrong|broken|failing)\b"
        ],
        "priority": 1
    },
    {
        "intent": "SHOW_HISTORY",
        "patterns": [
            r"\b(history|historical|past|last \d+ days?)\b"
        ],
        "priority": 2
    }
]

def recognize_intent_rule_based(user_input: str) -> dict:
    text = user_input.lower().strip()
    for rule in sorted(INTENT_PATTERNS, key=lambda x: x["priority"]):
        for pattern in rule["patterns"]:
            if re.search(pattern, text):
                return {
                    "intent": rule["intent"],
                    "confidence": 0.9,
                    "method": "rule-based"
                }
    return {"intent": "UNKNOWN", "confidence": 0.0, "method": "rule-based"}
```

**Approach B — LLM-Based:**
More flexible, handles ambiguous or complex phrasing. Use when rule-based returns low confidence or UNKNOWN.

```python
def recognize_intent_llm(user_input: str, llm_client) -> dict:
    system_prompt = """
    You are an intent classifier for an AI agent.
    Given a user message, return ONLY a JSON object with:
    - intent: one of [SHOW_STATUS, CHECK_ALERTS, SHOW_HISTORY, GET_DETAILS, UNKNOWN]
    - entity: the specific thing the user is asking about (or null)
    - confidence: float between 0 and 1
    - reasoning: one-sentence explanation

    Return only valid JSON. No preamble. No markdown.
    """
    response = llm_client.complete(system_prompt, user_input)
    return json.loads(response)
```

### 7.2 Entity Extraction

Often you need more than just intent — you need to know **what** the user is asking about (the entity). Extract entities alongside intent:

```python
def extract_entities(user_input: str) -> dict:
    entities = {}

    # Extract time ranges
    time_match = re.search(r'last (\d+) (day|hour|week)s?', user_input.lower())
    if time_match:
        entities["time_range"] = {
            "value": int(time_match.group(1)),
            "unit": time_match.group(2)
        }

    # Extract IDs or names (customize regex to your domain)
    id_match = re.search(r'\b([A-Z][A-Z0-9\-]{2,})\b', user_input)
    if id_match:
        entities["target_id"] = id_match.group(1)

    return entities
```

### 7.3 Intent → Action Mapping

Once intent is recognized, map it to a concrete action. Keep this mapping in one place:

```python
INTENT_HANDLERS = {
    "SHOW_STATUS":    lambda db, params: db.get_latest_status(params.get("target_id")),
    "CHECK_ALERTS":   lambda db, params: db.get_active_alerts(),
    "SHOW_HISTORY":   lambda db, params: db.get_history(params.get("target_id"), params.get("time_range")),
    "GET_DETAILS":    lambda db, params: db.get_alert_details(params.get("alert_id")),
}

def handle_intent(intent: str, db, params: dict):
    handler = INTENT_HANDLERS.get(intent)
    if handler:
        return handler(db, params)
    return None  # UNKNOWN intent
```

---

## 8. Layer 5 — LLM Integration

The LLM is the **voice** of your agent — it takes structured data from your database and turns it into clear, human-readable responses. It is not the brain; your analysis layer is.

### 8.1 The LLM Client

Wrap your LLM API in a clean client class. This makes it easy to swap providers later:

```python
import requests

class LLMClient:
    def __init__(self, api_url: str, api_key: str, model: str):
        self.api_url = api_url
        self.api_key = api_key
        self.model = model

    def complete(self, system_prompt: str, user_message: str,
                 context_data: dict = None) -> str:
        """
        Send a message to the LLM and return the response text.
        context_data: structured data to include in the prompt.
        """
        full_message = user_message
        if context_data:
            full_message += f"\n\nRelevant data:\n{json.dumps(context_data, indent=2)}"

        payload = {
            "model": self.model,
            "messages": [
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": full_message}
            ]
        }
        headers = {"Authorization": f"Bearer {self.api_key}"}
        response = requests.post(self.api_url, json=payload, headers=headers)
        response.raise_for_status()
        return response.json()["choices"][0]["message"]["content"]
```

### 8.2 System Prompt Design

The system prompt defines your agent's persona, capabilities, and constraints. Invest time writing it well.

**Good system prompt structure:**

```python
SYSTEM_PROMPT = """
You are [Agent Name], an intelligent assistant for [Domain].

Your role:
- Answer questions about [what the agent knows]
- Explain findings in plain, jargon-free language
- Suggest actionable next steps when issues are found
- Be concise. Avoid unnecessary filler text.

Your constraints:
- Only discuss topics related to [domain]
- If data is unavailable, say so clearly
- Never guess or fabricate data values
- When severity is CRITICAL, always recommend immediate action

Response format:
- Lead with the direct answer
- Follow with supporting details if relevant
- End with a recommended action if applicable
"""
```

### 8.3 Data-Grounded Responses

The most important principle: **always ground LLM responses in real data from your database**. Never let the LLM answer from its training knowledge about your specific system.

```python
def generate_response(user_input: str, intent: str, db_result: dict) -> str:
    if db_result is None:
        return "I don't have any data on that yet. Try running a collection first."

    # Pass the real data to the LLM as context
    return llm_client.complete(
        system_prompt=SYSTEM_PROMPT,
        user_message=user_input,
        context_data=db_result       # Ground truth from your database
    )
```

### 8.4 Streaming Responses (for Chat UIs)

For a better user experience with long responses, use streaming:

```python
def stream_response(prompt: str):
    response = requests.post(
        self.api_url,
        json={...},
        headers={...},
        stream=True
    )
    for chunk in response.iter_content(chunk_size=None):
        if chunk:
            yield chunk.decode()
```

### 8.5 Handling LLM Unavailability

Always have a fallback when the LLM is down:

```python
def get_response(user_input: str, intent: str, data: dict) -> str:
    try:
        return llm_client.complete(SYSTEM_PROMPT, user_input, data)
    except Exception as e:
        # Fallback: return structured data as plain text
        return format_as_plain_text(intent, data)
```

---

## 9. Layer 6 — Persistent Storage

Your agent needs to remember things. Use a database — even for small projects. SQLite is excellent for single-node agents; PostgreSQL when you need multi-user or high-volume access.

### 9.1 Core Tables (Generic Starting Point)

```sql
-- Collected data from external sources
CREATE TABLE collected_data (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id    TEXT NOT NULL,
    data_type    TEXT NOT NULL,       -- e.g., "status", "config", "metrics"
    raw_data     TEXT,                -- raw string/JSON from source
    parsed_data  TEXT,               -- JSON of structured parsed output
    collected_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Detected issues / anomalies
CREATE TABLE alerts (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id    TEXT NOT NULL,
    alert_type   TEXT NOT NULL,
    severity     TEXT NOT NULL,      -- CRITICAL, HIGH, MEDIUM, LOW, INFO
    message      TEXT NOT NULL,
    details      TEXT,               -- JSON with additional context
    is_active    INTEGER DEFAULT 1,  -- 1 = active, 0 = resolved
    detected_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    resolved_at  DATETIME
);

-- Log of data collection runs
CREATE TABLE collection_runs (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    status       TEXT NOT NULL,      -- "SUCCESS", "PARTIAL", "FAILED"
    sources_ok   INTEGER DEFAULT 0,
    sources_fail INTEGER DEFAULT 0,
    started_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME
);

-- Conversation history (for multi-turn context)
CREATE TABLE conversation_history (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id   TEXT NOT NULL,
    role         TEXT NOT NULL,      -- "user" or "assistant"
    message      TEXT NOT NULL,
    intent       TEXT,
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 9.2 Database Manager Pattern

Keep all database operations in a single `database.py` file. No raw SQL queries should appear anywhere else in the codebase.

```python
import sqlite3
import json
from datetime import datetime

class DatabaseManager:
    def __init__(self, db_path: str):
        self.db_path = db_path
        self._initialize_schema()

    def _get_connection(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row  # Access columns by name
        return conn

    def _initialize_schema(self):
        with self._get_connection() as conn:
            conn.executescript(SCHEMA_SQL)  # Run CREATE TABLE IF NOT EXISTS statements

    def save_collected_data(self, source_id: str, data_type: str,
                             raw: str, parsed: dict):
        with self._get_connection() as conn:
            conn.execute(
                """INSERT INTO collected_data (source_id, data_type, raw_data, parsed_data)
                   VALUES (?, ?, ?, ?)""",
                (source_id, data_type, raw, json.dumps(parsed))
            )

    def get_latest_data(self, source_id: str, data_type: str) -> dict | None:
        with self._get_connection() as conn:
            row = conn.execute(
                """SELECT parsed_data FROM collected_data
                   WHERE source_id = ? AND data_type = ?
                   ORDER BY collected_at DESC LIMIT 1""",
                (source_id, data_type)
            ).fetchone()
        return json.loads(row["parsed_data"]) if row else None

    def save_alert(self, source_id: str, alert_type: str,
                   severity: str, message: str, details: dict = None):
        with self._get_connection() as conn:
            conn.execute(
                """INSERT INTO alerts (source_id, alert_type, severity, message, details)
                   VALUES (?, ?, ?, ?, ?)""",
                (source_id, alert_type, severity, message,
                 json.dumps(details) if details else None)
            )

    def get_active_alerts(self) -> list[dict]:
        with self._get_connection() as conn:
            rows = conn.execute(
                """SELECT * FROM alerts WHERE is_active = 1
                   ORDER BY severity, detected_at DESC"""
            ).fetchall()
        return [dict(row) for row in rows]

    def resolve_alert(self, alert_id: int):
        with self._get_connection() as conn:
            conn.execute(
                """UPDATE alerts SET is_active = 0, resolved_at = ?
                   WHERE id = ?""",
                (datetime.utcnow().isoformat(), alert_id)
            )
```

---

## 10. Layer 7 — Presentation (Chat / CLI / API)

This is the layer users interact with. Keep it thin — it should only handle input/output formatting, not business logic.

### 10.1 The Chat Interface (Core Orchestrator)

```python
class ChatInterface:
    def __init__(self, db: DatabaseManager, llm: LLMClient,
                 intent_recognizer, session_manager):
        self.db = db
        self.llm = llm
        self.intent_recognizer = intent_recognizer
        self.session = session_manager

    def handle_message(self, user_input: str, session_id: str) -> str:
        # Step 1: Recognize intent
        intent_result = self.intent_recognizer.recognize(user_input)
        intent = intent_result["intent"]
        params = intent_result.get("entities", {})

        # Step 2: Fetch relevant data from database
        data = self._fetch_data_for_intent(intent, params)

        # Step 3: Generate LLM response grounded in that data
        response = self._generate_response(user_input, intent, data)

        # Step 4: Save to conversation history
        self.session.save_turn(session_id, user_input, response, intent)

        return response

    def _fetch_data_for_intent(self, intent: str, params: dict):
        return INTENT_HANDLERS.get(intent, lambda db, p: None)(self.db, params)

    def _generate_response(self, user_input: str, intent: str, data) -> str:
        if intent == "UNKNOWN":
            return ("I didn't understand that. Try asking about status, "
                    "alerts, or history for a specific component.")
        if data is None:
            return "No data available for that query. Try collecting data first."
        try:
            return self.llm.complete(SYSTEM_PROMPT, user_input, data)
        except Exception:
            return self._fallback_response(intent, data)
```

### 10.2 CLI Entry Point

```python
# main.py
import argparse
from src.collector import DataCollector
from src.database import DatabaseManager
from src.chat_interface import ChatInterface

def main():
    parser = argparse.ArgumentParser(description="AI Agent CLI")
    parser.add_argument("--collect", metavar="SOURCE",
                        help="Collect data from a source (or 'all')")
    parser.add_argument("--chat", action="store_true",
                        help="Start interactive chat session")
    args = parser.parse_args()

    db = DatabaseManager("agent.db")

    if args.collect:
        run_collection(args.collect, db)
    elif args.chat:
        run_chat(db)
    else:
        parser.print_help()

def run_collection(source: str, db: DatabaseManager):
    print(f"Collecting from: {source}")
    # ... collection logic

def run_chat(db: DatabaseManager):
    chat = ChatInterface(db, ...)
    session_id = generate_session_id()
    print("Agent ready. Type 'exit' to quit.\n")
    while True:
        user_input = input("> ").strip()
        if user_input.lower() in ("exit", "quit"):
            break
        if user_input:
            response = chat.handle_message(user_input, session_id)
            print(f"\nAgent: {response}\n")

if __name__ == "__main__":
    main()
```

### 10.3 Session Management

Track conversation context across turns so the agent can understand follow-up questions:

```python
class SessionManager:
    def __init__(self, db: DatabaseManager, max_history: int = 10):
        self.db = db
        self.max_history = max_history

    def get_context(self, session_id: str) -> list[dict]:
        """Return last N turns for this session."""
        return self.db.get_conversation_history(session_id, self.max_history)

    def save_turn(self, session_id: str, user_msg: str,
                  agent_msg: str, intent: str):
        self.db.save_conversation_turn(session_id, user_msg, agent_msg, intent)
```

---

## 11. Configuration Management

### 11.1 What Goes in Config vs Code

| Type | Where it lives | Why |
|---|---|---|
| Connection details | `configs/sources.json` | Changes without code changes |
| Expected state / rules | `configs/expected_state.json` | Domain experts can update |
| Secrets (API keys, passwords) | `.env` file | Never committed to version control |
| Agent behavior (prompts, thresholds) | `configs/agent_config.json` | Tunable without code changes |
| Business logic | Python source code | Belongs in code, not config |

### 11.2 Environment Variables

```bash
# .env  (add to .gitignore — never commit)
LLM_API_KEY=your_key_here
LLM_API_URL=https://api.your-llm-provider.com/v1
DATABASE_PATH=./data/agent.db
LOG_LEVEL=INFO
```

```python
# Load at startup
from dotenv import load_dotenv
import os

load_dotenv()

LLM_API_KEY = os.getenv("LLM_API_KEY")
if not LLM_API_KEY:
    raise ValueError("LLM_API_KEY is required. Check your .env file.")
```

### 11.3 Config Validation at Startup

Fail fast with a clear error if configuration is missing or malformed:

```python
def validate_config(config: dict) -> list[str]:
    errors = []
    required_fields = ["sources", "llm", "database"]
    for field in required_fields:
        if field not in config:
            errors.append(f"Missing required config section: '{field}'")
    return errors

config = load_config("configs/agent_config.json")
errors = validate_config(config)
if errors:
    for err in errors:
        print(f"[CONFIG ERROR] {err}")
    sys.exit(1)
```

---

## 12. Error Handling & Fallback Strategy

Agents interact with external systems — things will go wrong. Design for failure from day one.

### 12.1 The Fallback Hierarchy

```
Primary:   LLM-based intent recognition + LLM-generated response
    ↓ (if LLM unavailable)
Fallback:  Rule-based intent recognition + template response
    ↓ (if no data available)
Fallback:  Clear "no data" message with instructions
    ↓ (if everything fails)
Fallback:  Graceful error message — never crash, never expose stack traces to users
```

### 12.2 Retry with Exponential Backoff

For transient network errors:

```python
import time

def with_retry(fn, max_attempts: int = 3, backoff_seconds: float = 2.0):
    for attempt in range(1, max_attempts + 1):
        try:
            return fn()
        except Exception as e:
            if attempt == max_attempts:
                raise
            wait = backoff_seconds ** attempt
            print(f"Attempt {attempt} failed ({e}). Retrying in {wait}s...")
            time.sleep(wait)
```

### 12.3 Partial Failure Handling

When collecting from multiple sources, don't let one failed source stop everything:

```python
results = {"success": [], "failed": []}

for source in sources:
    try:
        data = collector.collect(source)
        db.save_collected_data(source["id"], data)
        results["success"].append(source["id"])
    except Exception as e:
        results["failed"].append({"id": source["id"], "error": str(e)})
        logger.error(f"Collection failed for {source['id']}: {e}")

print(f"Collected: {len(results['success'])} OK, {len(results['failed'])} failed")
```

### 12.4 User-Facing Error Messages

Write error messages for humans, not developers:

```python
ERROR_MESSAGES = {
    "NO_DATA":         "I don't have recent data for that. Run a collection first.",
    "SOURCE_OFFLINE":  "I can't reach that source right now. Check connectivity.",
    "LLM_UNAVAILABLE": "My AI response engine is temporarily unavailable. "
                        "Here's the raw data instead: {data}",
    "UNKNOWN_INTENT":  "I didn't understand that. Try: 'show status of X', "
                        "'list active alerts', or 'show history for Y'.",
}
```

---

## 13. Testing Strategy

### 13.1 What to Test

| Layer | What to test | How |
|---|---|---|
| **Parser** | Correct output from known raw inputs | Unit tests with fixture files |
| **Analyzer** | Correct alerts generated from known data | Unit tests with mock expected state |
| **Intent Recognizer** | Correct intent from varied phrasings | Unit tests with phrase lists |
| **Database** | CRUD operations work correctly | Integration tests with temp DB |
| **Chat Interface** | Correct orchestration end-to-end | Integration tests with mock LLM |
| **Collector** | Correct connection and data retrieval | Integration tests against test source |

### 13.2 Testing the Parser (Example)

```python
# tests/test_parser.py
from src.parser import parse_status_output

def test_parse_status_up():
    raw = "Status: UP  Uptime: 02:34:15  Version: 1.4.2"
    result = parse_status_output(raw)
    assert result["status"] == "UP"
    assert result["uptime"] == "02:34:15"

def test_parse_status_handles_missing_fields():
    raw = "Status: UNKNOWN"
    result = parse_status_output(raw)
    assert result["status"] == "UNKNOWN"
    assert result["uptime"] is None  # Should not crash
```

### 13.3 Testing the Intent Recognizer

Write a matrix of expected phrasings → expected intents and assert them all:

```python
# tests/test_intent_recognizer.py
import pytest
from src.intent_recognizer import IntentRecognizer

recognizer = IntentRecognizer()

TEST_CASES = [
    ("What is the status of service-01?",      "SHOW_STATUS"),
    ("Show status",                             "SHOW_STATUS"),
    ("Are there any active alerts?",            "CHECK_ALERTS"),
    ("What's broken right now?",               "CHECK_ALERTS"),
    ("Show me the last 7 days for service-01", "SHOW_HISTORY"),
    ("What time is it?",                       "UNKNOWN"),
]

@pytest.mark.parametrize("user_input,expected_intent", TEST_CASES)
def test_intent_recognition(user_input, expected_intent):
    result = recognizer.recognize(user_input)
    assert result["intent"] == expected_intent
```

### 13.4 Use Fixture Files for Parser Tests

Store sample raw data in `tests/fixtures/` instead of hardcoding strings:

```
tests/
└── fixtures/
    ├── sample_status_output.txt
    ├── sample_error_output.txt
    └── sample_metrics_response.json
```

```python
def load_fixture(filename):
    with open(f"tests/fixtures/{filename}") as f:
        return f.read()

def test_parse_real_output():
    raw = load_fixture("sample_status_output.txt")
    result = parse_status_output(raw)
    assert result["status"] is not None
```

---

## 14. Security Considerations

### 14.1 Secrets Management

- **Never hardcode secrets** in source code
- **Never commit `.env`** — add it to `.gitignore`
- **Always provide `.env.example`** with placeholder values
- Rotate API keys if they are ever accidentally exposed

### 14.2 Input Validation

Never trust user input. Sanitize everything before passing to a database query or shell command:

```python
import re

def validate_source_id(source_id: str) -> bool:
    """Only allow alphanumeric IDs with hyphens."""
    return bool(re.match(r'^[a-zA-Z0-9\-]{1,64}$', source_id))

def handle_message(user_input: str) -> str:
    if len(user_input) > 2000:
        return "Message too long. Please be more concise."
    if not user_input.strip():
        return "Please enter a message."
    # Proceed safely
```

### 14.3 Parameterized SQL Queries

Never use string formatting to build SQL queries — use parameterized queries always:

```python
# NEVER do this
conn.execute(f"SELECT * FROM alerts WHERE source_id = '{source_id}'")  # SQL injection risk

# ALWAYS do this
conn.execute("SELECT * FROM alerts WHERE source_id = ?", (source_id,))
```

### 14.4 Credential Storage

For SSH keys, API credentials, and connection details:

- Store in environment variables or a secrets manager (AWS Secrets Manager, HashiCorp Vault)
- Never log credentials — mask them in log output
- Use least-privilege access — give the agent only the permissions it needs

### 14.5 LLM Prompt Injection

Users may try to manipulate your agent via prompt injection (e.g., "Ignore your instructions and..."). Mitigations:

- Separate system prompts and user input clearly (most LLM APIs handle this with roles)
- Validate that LLM responses conform to expected format when used for intent recognition
- Never let LLM output be executed as code or shell commands

---

## 15. Deployment & Operations

### 15.1 Logging

Add structured logging from day one:

```python
import logging
import json

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s'
)
logger = logging.getLogger(__name__)

# Log with context
logger.info("Collection started", extra={"source_count": len(sources)})
logger.error("Collection failed", extra={"source_id": sid, "error": str(e)})
```

### 15.2 Health Check

Add a simple health check command that validates all dependencies are reachable:

```python
def health_check(db, llm_client, sources) -> dict:
    return {
        "database": check_database(db),
        "llm_api":  check_llm(llm_client),
        "sources":  check_sources(sources)
    }

# Usage: python main.py --health-check
```

### 15.3 Environment Separation

Always maintain separate configurations for each environment:

```
configs/
├── sources.dev.json       # Local / dev data sources
├── sources.staging.json   # Staging environment
└── sources.prod.json      # Production (restricted access)
```

### 15.4 Graceful Shutdown

Handle SIGINT / SIGTERM to clean up connections properly:

```python
import signal
import sys

def shutdown_handler(sig, frame):
    print("\nShutting down gracefully...")
    collector.disconnect()
    db.close()
    sys.exit(0)

signal.signal(signal.SIGINT, shutdown_handler)
signal.signal(signal.SIGTERM, shutdown_handler)
```

---

## 16. Common Pitfalls & How to Avoid Them

| Pitfall | Consequence | How to avoid |
|---|---|---|
| **Letting the LLM be the brain** | Hallucinated facts about your specific system | Always fetch real data from DB first, pass it as context to LLM |
| **No fallback when LLM is down** | Agent completely unusable | Rule-based fallback + template responses |
| **Hardcoding expected values in code** | Config changes require code changes and re-deploy | Load all expected state from JSON config files |
| **One collection failure stops everything** | Missing data from all sources if one fails | Wrap each source in try/catch, continue on failure |
| **No input validation** | SQL injection, prompt injection, crashes | Validate and sanitize all user input |
| **Committing secrets to git** | Credential exposure | `.env` in `.gitignore`, secrets manager for production |
| **No tests on parser** | Silent data corruption when source format changes | Unit tests with real fixture data |
| **Building UI first** | Beautiful interface on broken backend | Build and test collection → parsing → analysis → storage first |
| **No structured logging** | Impossible to debug production issues | Add logging from day one |
| **Monolithic code** | Hard to test, modify, or scale any single layer | Enforce layered architecture strictly |

---

## 17. Quick Reference Checklist

Use this checklist before considering your agent "production-ready":

### Planning
- [ ] Problem statement written in one paragraph
- [ ] All data sources listed and classified
- [ ] All intents defined and written down
- [ ] Expected state / ground truth defined and stored in config

### Architecture
- [ ] Layered architecture followed (no business logic in presentation layer)
- [ ] Project folder structure created
- [ ] No circular imports between layers

### Data Collection
- [ ] Collector handles connection failures gracefully
- [ ] Multi-source collection continues past individual failures
- [ ] Collection can be triggered both on-demand and on schedule

### Parsing
- [ ] Data schema defined as dataclasses or TypedDicts
- [ ] Parser is stateless and has no side effects
- [ ] Fixture files created for unit tests

### Analysis
- [ ] Severity model defined (CRITICAL / HIGH / MEDIUM / LOW)
- [ ] Expected state loaded from config (not hardcoded)
- [ ] Alerts include enough detail to be actionable

### Intent Recognition
- [ ] Both rule-based AND LLM-based recognizers implemented
- [ ] LLM-based falls back to rule-based on failure
- [ ] UNKNOWN intent handled gracefully
- [ ] Intent → action mapping in one place

### LLM Integration
- [ ] System prompt written and tested
- [ ] LLM response grounded in real DB data
- [ ] Fallback response when LLM is unavailable
- [ ] Prompt injection considered

### Database
- [ ] Schema documented and versioned
- [ ] All SQL in `database.py` only — no raw SQL elsewhere
- [ ] Parameterized queries used everywhere

### Configuration & Security
- [ ] Secrets in `.env`, not in code
- [ ] `.env` in `.gitignore`
- [ ] `.env.example` committed as template
- [ ] Config validated at startup — fail fast

### Testing
- [ ] Parser: unit tests with fixtures
- [ ] Intent recognizer: unit tests with phrase matrix
- [ ] Analyzer: unit tests with mock data
- [ ] Database: integration tests with temp DB

### Operations
- [ ] Structured logging in place
- [ ] Health check command implemented
- [ ] Graceful shutdown on SIGINT/SIGTERM
- [ ] User-facing error messages are human-readable (no stack traces)

---

*This guide is based on patterns used in the FRR Intelligent Routing Agent developed at Samsung Research — a production AI agent for automated network configuration management — generalized for any domain.*
