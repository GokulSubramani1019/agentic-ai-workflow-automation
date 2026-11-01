# Agentic AI Workflow Automation System

An autonomous AI agent built with **LangChain** and **OpenAI GPT-4** that plans, reasons and executes multi-step data management tasks. It can query a PostgreSQL database, summarize reports and trigger alerts when data conditions are met. Every decision and tool call is written to an audit trail, so each run is explainable and traceable.

## Features

- **Plan, act, observe loop:** the agent writes a short plan, calls tools, reads the results and continues until it can give a final report.
- **Three tools:** `query_database` (read-only SQL), `summarize_report` and `trigger_alert`.
- **Safe SQL:** only a single `SELECT` is accepted. Writes, multiple statements and comments are rejected, and on PostgreSQL the query runs in a read-only transaction.
- **Memory:** the agent remembers earlier turns within a session (`session_id`).
- **Audit trail:** every task, LLM response and tool call is stored in the `audit_log` table and exposed through the API.
- **Step limit:** a run stops after `MAX_AGENT_STEPS` so it can never loop forever.

## Tech Stack

Python, LangChain, OpenAI API (GPT-4), FastAPI, PostgreSQL, SQLAlchemy, Docker, GitHub Actions

## Architecture

```
POST /run  {task, session_id}
      |
      v
WorkflowAgent (LangChain + GPT-4) <--> session memory
      |
      |  each step: LLM decides -> tool runs -> result goes back to the LLM
      +--> query_database   (read-only SELECT on PostgreSQL)
      +--> summarize_report (reads a report, summarizes it)
      +--> trigger_alert    (writes to alerts table, optional webhook)
      |
      v
audit_log table  <--  GET /runs/{run_id}/audit
```

## Project Structure

```
app/
  config.py   settings from environment variables
  db.py       tables + demo data (sales, reports, alerts, audit_log)
  tools.py    the three agent tools and the SQL safety check
  agent.py    the agent loop, memory and audit logging
  audit.py    write and read the audit trail
  main.py     FastAPI endpoints
tests/        21 tests that run offline with a fake LLM
```

## Quick Start (Docker)

```bash
cp .env.example .env        # then put your OPENAI_API_KEY in .env
docker compose up --build
```

The API runs at http://localhost:8000 (interactive docs at `/docs`). Demo data is loaded on first start. It includes a sales drop in the South region over the last 5 days, so the agent has something to find.

## Run Locally Without Docker

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# For a quick local run, you can use SQLite instead of PostgreSQL:
#   DATABASE_URL=sqlite:///./demo.db
uvicorn app.main:app --reload
```

## Example

```bash
curl -X POST http://localhost:8000/run -H "Content-Type: application/json" -d '{
  "task": "Compare the last 5 days of sales per region against the previous 25 days. If any region fell by more than 30 percent, trigger a critical alert and summarize the Weekly Sales Review report."
}'
```

The response contains `run_id`, `final_answer`, number of `steps` and `tool_calls`. Then review exactly what the agent did:

```bash
curl http://localhost:8000/runs/<run_id>/audit
```

The audit trail lists, in order: `task_received`, each `llm_response` (including the plan and requested tool calls), each `tool_call` with its arguments and result, and the `final_answer`.

## Run the Tests

```bash
pip install -r requirements-dev.txt
pytest -q
```

Tests use SQLite and a fake LLM, so they need no API key and no network.

## Design Decisions (good interview talking points)

- **Why a manual tool-calling loop?** It keeps every step visible, which makes logging each action for the audit trail simple and gives full control over the step limit and error handling.
- **Why validate SQL in code?** The LLM is not trusted with write access. The check, the read-only transaction and the row limit are layered protections.
- **Tool errors go back to the model** instead of crashing the run, so the agent can correct its own query.
- **Responsible AI:** the audit trail makes every automated action reviewable after the fact.

## Possible Improvements

- Human approval step before critical alerts
- Authentication on the API
- Scheduled runs for recurring data checks
- Per-tool rate limits and cost tracking

## Author

Gokul Subramani, Chennai, India
