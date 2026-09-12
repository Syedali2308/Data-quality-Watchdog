# Autonomous Data Quality Watchdog & Pipeline Self-Healer

A lightweight, production-shaped service that:

1. **Profiles** production tables on a schedule (and on demand) using pure,
   deterministic SQL — null-ratio checks, duplicate primary-key checks, and
   `information_schema` schema-drift detection.
2. **Diagnoses** any breach with an LLM (Gemini or OpenAI, your choice) that
   returns a structured root-cause explanation and, where safe, a proposed
   SQL fix.
3. **Never executes anything automatically.** Every proposed fix sits in an
   `AWAITING_APPROVAL` state until a human explicitly approves it via the
   API. Only `UPDATE`/`INSERT` statements with a mandatory `WHERE` clause on
   `UPDATE` are ever accepted — `DROP`, `TRUNCATE`, `ALTER`, `DELETE`, and
   any multi-statement SQL are rejected before they ever reach a human, let
   alone a database.

Built to run comfortably on an 8GB RAM laptop: sync SQLAlchemy + psycopg2
(no async driver overhead), a 3-connection pool, no local model weights —
all diagnosis happens via a cloud LLM API call.

---

## 1. Repository layout

```
dq-watchdog/
├── app/
│   ├── main.py                 # FastAPI app + APScheduler wiring
│   ├── core/
│   │   ├── config.py           # Settings (env vars, thresholds, pool sizes)
│   │   └── database.py         # SQLAlchemy engine/session management
│   ├── models/
│   │   └── schema.py           # ORM: WatchdogIncident, SchemaSnapshot, TargetTableRegistry
│   ├── profiler/
│   │   ├── metrics.py          # Deterministic SQL: null ratio, duplicates, schema drift
│   │   └── engine.py           # Orchestrates checks -> incidents -> diagnosis handoff
│   ├── agent/
│   │   ├── schemas.py          # RootCauseDiagnosis + SQL-safety validation
│   │   └── analyzer.py         # LLM caller (Gemini / OpenAI), strict JSON parsing
│   └── api/
│       └── routes.py           # /health, /incidents, /profiler/run, approve-fix, reject
├── scripts/
│   ├── init_db.sql             # Mock users/orders tables + seed data + registry rows
│   └── simulate_anomalies.py   # Injects null spikes / duplicate keys / schema drift
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

---

## 2. Prerequisites

- Python 3.11+
- PostgreSQL 13+ running locally (or reachable) with an empty database created
- A Gemini API key **or** an OpenAI API key

---

## 3. Setup

### 3.1 Create and activate a virtual environment

```bash
cd dq-watchdog
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
```

### 3.2 Install dependencies

```bash
pip install -r requirements.txt
```

### 3.3 Create the Postgres database

```bash
createdb dq_watchdog
# or, from psql:
# CREATE DATABASE dq_watchdog;
# CREATE USER dq_user WITH PASSWORD 'dq_pass';
# GRANT ALL PRIVILEGES ON DATABASE dq_watchdog TO dq_user;
```

### 3.4 Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and set:
- `DATABASE_URL` to match your Postgres credentials
- `LLM_PROVIDER` to `gemini` or `openai`
- `GEMINI_API_KEY` or `OPENAI_API_KEY` accordingly

---

## 4. Running the demo end-to-end

### Step 1 — Start the API (this also creates the watchdog's own metadata
tables via SQLAlchemy on startup)

```bash
uvicorn app.main:app --reload --port 8000
```

Leave this running. You should see log lines confirming the metadata
tables were created and the APScheduler job was scheduled.

### Step 2 — Seed the mock production tables + register them with the watchdog

In a second terminal (with the venv activated):

```bash
psql -U dq_user -d dq_watchdog -f scripts/init_db.sql
```

This creates `users` and `orders` with clean seed data, and inserts two
rows into `watchdog_target_table_registry` so the profiler knows to watch
them.

### Step 3 — Inject an anomaly

```bash
python scripts/simulate_anomalies.py --scenario null_spike
# or: duplicate_key / schema_drift / all
```

### Step 4 — Trigger the profiler on demand

```bash
curl -X POST http://localhost:8000/api/v1/profiler/run \
  -H "Content-Type: application/json" \
  -d '{"run_diagnosis": true}'
```

This runs the deterministic checks, creates a `WatchdogIncident`, and — if
`run_diagnosis` is true — calls the LLM to produce a root-cause diagnosis
and a proposed fix. The response (and `GET /api/v1/incidents`) will show
the incident sitting in `AWAITING_APPROVAL` with a `suggested_sql` field.

### Step 5 — Review and approve (or reject) the fix

```bash
# List everything awaiting a human decision:
curl "http://localhost:8000/api/v1/incidents?status=AWAITING_APPROVAL"

# Approve (replace {incident_id}):
curl -X POST http://localhost:8000/api/v1/incidents/{incident_id}/approve-fix \
  -H "Content-Type: application/json" \
  -d '{"reviewed_by": "your_name", "reviewer_note": "Looks safe, approving."}'

# ...or reject instead:
curl -X POST http://localhost:8000/api/v1/incidents/{incident_id}/reject \
  -H "Content-Type: application/json" \
  -d '{"reviewed_by": "your_name", "reviewer_note": "Want to investigate manually first."}'
```

Approving transitions the incident to `RESOLVED` and executes the SQL in a
single transaction. Rejecting moves it to `REJECTED` and no SQL is ever run.

### Step 6 — Verify the fix

```bash
curl "http://localhost:8000/api/v1/incidents/{incident_id}"
```

`status` should now read `RESOLVED`, and re-running
`POST /api/v1/profiler/run` should no longer flag the same anomaly (or
should show a much smaller null ratio, depending on scenario).

---

## 5. API reference

| Method | Path                                   | Description                                    |
|--------|-----------------------------------------|-------------------------------------------------|
| GET    | `/api/v1/health`                        | Liveness + DB connectivity check                |
| GET    | `/api/v1/incidents?status=&table_name=` | List incidents, optionally filtered             |
| GET    | `/api/v1/incidents/{id}`                | Single incident detail                          |
| POST   | `/api/v1/profiler/run`                  | Trigger an on-demand profiler pass              |
| POST   | `/api/v1/incidents/{id}/approve-fix`    | HITL approval — executes `suggested_sql`        |
| POST   | `/api/v1/incidents/{id}/reject`         | HITL rejection — no SQL is ever executed        |

Interactive Swagger docs are available at `http://localhost:8000/docs` once
the server is running.

---

## 6. Safety model (defense in depth)

1. **Deterministic detection only.** The LLM is never involved in deciding
   *whether* an anomaly exists — that's pure SQL against configurable
   thresholds.
2. **Structured, validated diagnosis.** The LLM's response is required to be
   JSON matching `RootCauseDiagnosis`; anything that fails to parse or
   validate marks the incident `FAILED`, never partially-applied.
3. **SQL allow-list, not block-list, at the statement level.** Only
   `UPDATE` and `INSERT` are accepted; `UPDATE` additionally requires a
   `WHERE` clause. `DELETE`, `DROP`, `TRUNCATE`, `ALTER`, `GRANT`, `REVOKE`,
   and any multi-statement (`;`-chained) SQL are rejected at validation
   time, in `app/agent/schemas.py`.
4. **Second, independent check at execution time.** `_execute_approved_fix`
   in `app/api/routes.py` re-validates the first token and `WHERE` clause
   immediately before running the statement, in case the stored value was
   ever mutated between diagnosis and approval.
5. **Human-in-the-loop, always.** No incident can reach `RESOLVED` without
   passing through `AWAITING_APPROVAL` and an explicit
   `POST /incidents/{id}/approve-fix` call carrying a `reviewed_by` name.
6. **Full audit trail.** Every incident records its raw metric payload, the
   LLM's raw response, who reviewed it, their note, and any execution error
   — nothing is silently discarded.

---

## 7. Adding your own tables

Insert a row into `watchdog_target_table_registry` for any table you want
watched:

```sql
INSERT INTO watchdog_target_table_registry
    (id, table_name, primary_key_column, null_check_columns, check_duplicates, check_schema_drift, enabled)
VALUES
    (gen_random_uuid()::text, 'your_table', 'your_pk_column',
     '["col_a", "col_b"]'::json, true, true, true);
```

The next profiler run (scheduled or `POST /profiler/run`) will pick it up
automatically — no code changes required.
