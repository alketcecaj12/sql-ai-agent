# SQL AI Agent (DuckDB + local models)

You ask a question in plain English, an LLM writes the SQL, and DuckDB runs it
on the SFO air-traffic dataset. This is a simplified rewrite of the LinkedIn
Learning course repo *Build with AI: Safe and Scalable SQL AI Agents*:

* plain Python files instead of Jupyter notebooks
* DuckDB only. Postgres, Docker Compose, `psycopg` and `ibis` are gone.
* local open-weight models through Ollama by default, so there's no API bill

## Quick start

```bash
pip install -r requirements.txt

# 1. Local model (free). Install Ollama from https://ollama.com/download, then:
ollama pull qwen2.5-coder:7b      # main model, good at SQL
ollama pull llama3.1:8b           # fallback model

# 2. Build the clean dataset (already included, re-run if you change it)
python prepare_data.py

# 3. Ask questions
python ask.py "Which 5 airlines carried the most passengers in 2024?"
python ask.py                      # interactive, supports follow-up questions

# 4. Tests (no model needed, they use a fake LLM)
pytest -q
```

If you'd rather use a paid API or another local server (LM Studio, vLLM,
llama.cpp), you only change the `llm:` block in `config.yaml`. The code stays
the same because they all use the OpenAI chat format.

## Files

| File | What it does |
|---|---|
| `config.yaml` | All settings: model, data file, retries, row limit, memory, log |
| `sql_agent/database.py` | Load the CSV into DuckDB, read the schema and text values, run SQL |
| `sql_agent/prompts.py` | The two prompts (write SQL, fix SQL) and SQL clean-up |
| `sql_agent/validator.py` | Block anything that isn't one read-only query and add a LIMIT |
| `sql_agent/llm.py` | Create the model client (Ollama or any OpenAI-compatible server) |
| `sql_agent/agent.py` | **The agent**: generate, check, run, debug, fallback, memory |
| `sql_agent/query_log.py` | Save every question and its outcome to `logs/query_log.duckdb` |
| `ask.py` | Command-line chat |
| `evaluate.py` | Score one or more models on 20 questions and 5 broken queries |
| `prepare_data.py` | Raw CSV to clean `data/air_traffic_gold.csv` |
| `examples/` | One short script per course lesson (see below) |
| `skills/` | Domain notes added to the prompt (codeshares, transit, and so on) |
| `tests/` | Offline tests |

## How the agent answers a question (`sql_agent/agent.py`)

1. **Generate.** The LLM gets the schema, the real values of the text columns,
   the domain notes, recent questions (memory), and your question. It returns SQL.
2. **Check.** `validator.py` parses the SQL without running it. It blocks
   DROP, DELETE, multiple statements and so on, and it adds `LIMIT 1000`.
   A blocked query stops here and is never retried.
3. **Run.** DuckDB executes the query.
4. **Debug.** If DuckDB raises an error, the LLM sees the query, the error and
   its earlier failed attempts, then tries again (up to 3 times).
5. **Fallback.** If it still fails, the fallback model gets one fresh try.

## Using it as a sub-agent

The agent also fits as a sub-agent in a multi-agent setup. There, a supervisor
(or "super") agent talks to the user, splits the work, and hands each part to
a specialised sub-agent: one for SQL data, one for documents, one for web
search, and so on.

Why it fits:

* **One entry point.** `agent.ask(question)` takes one question and returns a
  structured `Result` (`sql`, `data`, `error`, `blocked`, `attempts`, `model`).
  That is what a supervisor needs from a tool.
* **It handles its own safety and failures.** It allows read-only SQL only,
  adds a LIMIT, retries failed queries with the debugger, and falls back to a
  second model. Problems come back as data in the `Result` rather than as
  crashes, so the supervisor can decide what to do next.
* **Predictable.** It runs the same five fixed steps every time, so its
  behaviour is easy to test (`evaluate.py`) and to trust inside a larger system.

What to add when you wire it in (not included yet):

1. **A tool wrapper:** for example, a LangChain `@tool` function that calls
   `agent.ask(question, verbose=False)` and returns compact JSON (the SQL, the
   first 20 rows, the error), so it doesn't flood the supervisor's context.
2. **A clear tool description:** for example, *"Answers questions about SFO
   passenger traffic using SQL."* The supervisor uses it to decide when to call
   this sub-agent.
3. **Memory off in the sub-agent (`memory_size: 0`).** The supervisor keeps
   the conversation and sends complete, self-contained questions, so the two
   memories can't drift apart.
4. **One sub-agent per dataset.** Create a separate `SqlAgent` for each table
   or database and register each one as its own tool.

```
                 ┌─────────────────────┐
     user  ───▶  │  supervisor agent   │
                 └──────────┬──────────┘
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
   SqlAgent (SFO)    SqlAgent (sales)    other sub-agents
   DuckDB + LLM      DuckDB + LLM        (docs, web, ...)
```

## Notebook to script map

| Course notebook | Now |
|---|---|
| 01_03 Preparing the dataset | `prepare_data.py`, `examples/01_load_data.py` |
| 02_01 Prompt templates | `examples/02_prompt_template.py` |
| 02_02 LangChain chain | `examples/03_first_query.py` |
| 02_04 Additional context | `examples/04_add_context.py` |
| 02_05 Short-term memory | `examples/05_memory.py` |
| 02_06 Skills | `examples/06_skills.py` |
| 03_03 / 03_04 Validation, LIMIT | `examples/07_safety.py`, `sql_agent/validator.py` |
| 04_02 Debugger agent | `examples/08_debugger.py` |
| 04_04 Testing framework | `evaluate.py` |
| 05 Database logging | `sql_agent/query_log.py`, `examples/09_query_log.py` |

Run any example from the project folder, e.g. `python examples/05_memory.py`.
Examples 01, 02 and 07 don't need a model.

## What was removed and why

* **Postgres**: `docker-compose.yaml`, the Postgres branches in every module,
  the `database:` config block, `psycopg`, and `ibis-framework[postgres]`.
* **ibis**: it was only a wrapper that let one code path talk to two
  databases. With DuckDB alone, the `duckdb` package is simpler.
* **Streamlit apps** (`app/`): about 2,000 lines, mostly Postgres wiring. `ask.py`
  replaces the chat app and `examples/09_query_log.py` replaces the log viewer.
* **Custom logging stack** (`logger.py`, `callbacks.py`, `token_utils.py`,
  `log_database.py`): replaced by one DuckDB table you can query with SQL.
* **Multi-provider config loader**: the `llm:` block holds one endpoint now.
  To switch provider, you edit three lines.
# sql-ai-agent
