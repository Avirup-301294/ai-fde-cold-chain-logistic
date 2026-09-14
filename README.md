# AI FDE Cold-Chain Logistics

An AI-assisted dispatch console for investigating cold-chain shipment incidents. The project turns raw fleet telemetry into a protected SQL semantic layer, combines it with live corridor weather and retrieved SOP guidance, and presents an auditable operational response through a Streamlit interface.

## What it does

The application supports a dispatcher workflow such as: find shipments near a location, inspect cargo temperature and risk indicators, check local weather conditions, retrieve the relevant compliance rule, and recommend the next action.

The pipeline has five main parts:

1. **Legacy ingestion** — loads a CSV into a SQL Server table with legacy-style column names.
2. **Security and semantic layer** — exposes a clean read-only view for the AI agent while denying access to the raw table.
3. **SOP retrieval** — parses policy documents and indexes their embeddings in Pinecone.
4. **Agent orchestration** — a LangGraph agent selects among telemetry, weather, and SOP-retrieval tools.
5. **Dispatch console and audit trail** — a Streamlit app shows the agent’s tool traces and stores execution records in SQL Server.

## Architecture

```text
CSV telemetry ──> MSSQL dbo.TBL_SC_FLEET_HIST_RAW
                          │
                          └──> FDE_VIEWS.VW_ACTIVE_FLEET (agent read-only view)
                                         │
SOP files ──> Pinecone vector index ────┤
                                         v
Open-Meteo live weather ──> LangGraph dispatch agent ──> Streamlit console
                                                   │
                                                   └──> FDE_VIEWS.AgentAuditLog
```

## Repository layout

| Path | Purpose |
| --- | --- |
| `scripts/ingest_legacy_data.py` | Reads the fleet CSV and replaces `dbo.TBL_SC_FLEET_HIST_RAW` in SQL Server. |
| `scripts/setup_security_views` | SQL script that creates the clean semantic view and restricted agent login. |
| `scripts/ingest_sop_pinecone.py` | Parses policy files, creates/updates a Pinecone index, and maintains an ingestion hash cache. |
| `src/agent_tools.py` | The agent’s SQL, weather, and SOP search tools. |
| `src/orchestrator.py` | Defines and compiles the LangGraph tool-calling agent. |
| `src/ui.py` | Streamlit dispatch and audit-log interface. |
| `src/prompts/system_prompt.txt` | Required agent workflow and business-response format. |
| `data/raw/` | Expected location for `dynamic_supply_chain_logistics_dataset.csv`. |
| `data/policy/` | SOP files to index (`.md`, `.txt`, `.pdf`, `.csv`, or `.xlsx`). |
| `data/cache/` | Local, generated SOP ingestion-hash cache. |
| `.env-template` | Starting point for local database configuration. |
| `docs/instructions.md` | Original phase-by-phase project notes. |

## Prerequisites

- Python **3.12.13** or newer (the project requires `>=3.12.13`).
- Docker Desktop, for local SQL Server.
- Microsoft **ODBC Driver 18 for SQL Server** and `unixODBC`.
- A Pinecone account and API key.
- An LLM provider:
  - **Ollama** with `qwen2.5:7b` for the default local mode, or
  - OpenAI, or
  - DeepSeek.

On macOS with Homebrew, install the SQL Server driver with:

```bash
brew install unixodbc
brew tap microsoft/mssql-release https://github.com/microsoft/homebrew-mssql-release
brew trust --formula microsoft/mssql-release/msodbcsql18
brew install msodbcsql18
odbcinst -q -d
```

The final command should list `ODBC Driver 18 for SQL Server`.

## Setup

### 1. Create a Python environment and install dependencies

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Alternatively, use the checked-in `uv.lock` with your preferred `uv` workflow.

### 2. Start SQL Server locally

Choose a strong password for SQL Server’s `sa` account and keep it only in your local `.env` file.

```bash
docker run --name legacy-mssql \
  -e ACCEPT_EULA=Y \
  -e MSSQL_SA_PASSWORD='<choose-a-strong-password>' \
  -p 1433:1433 \
  -d mcr.microsoft.com/mssql/server:2022-latest
```

Wait for SQL Server to finish starting before running ingestion:

```bash
docker logs -f legacy-mssql
```

### 3. Create local configuration

Copy the template and extend it with the settings used by the agent:

```bash
cp .env-template .env
```

Use values appropriate to your environment. Do not commit `.env`.

```dotenv
# SQL Server administrator — used only for CSV ingestion and audit-log viewing
SQL_SERVER_HOST=localhost
SQL_SERVER_PORT=1433
SQL_ADMIN_USER=sa
SQL_ADMIN_PASSWORD=<the MSSQL_SA_PASSWORD value>

# Restricted SQL identity — created by scripts/setup_security_views
SQL_AGENT_USER=USR_FDE_RO
SQL_AGENT_PASSWORD=<agent-login-password>

# Required for SOP indexing and retrieval
PINECONE_API_KEY=<your-pinecone-api-key>

# Embedding mode: LOCAL (default) or OPENAI
Embeddings_model=LOCAL
Local_Embedding_Model=BAAI/bge-m3

# Reasoning mode: OLLAMA (default), OPENAI, or DEEPSEEK
Agent_llm=OLLAMA

# Required only for the corresponding cloud mode
# OPENAI_API_KEY=<your-openai-api-key>
# DEEPSEEK_API_KEY=<your-deepseek-api-key>
```

If using the local defaults, install and start Ollama, then make the configured model available:

```bash
ollama pull qwen2.5:7b
```

### 4. Add project data

Place the fleet data at:

```text
data/raw/dynamic_supply_chain_logistics_dataset.csv
```

Place one or more SOP/compliance documents in `data/policy/`. The SOP ingestion script supports Markdown, text, PDF, CSV, and Excel files.

## Run the pipeline

Run these commands from the repository root, in order.

### 1. Ingest fleet telemetry

```bash
source .venv/bin/activate
python scripts/ingest_legacy_data.py
```

This maps source fields to legacy names and writes the result to `dbo.TBL_SC_FLEET_HIST_RAW`. It uses `if_exists='replace'`, so each run replaces the table.

### 2. Create the secure semantic layer

Open `scripts/setup_security_views` in a SQL Server client (for example, the VS Code SQL Server extension) and execute it against the `master` database.

It creates:

- `FDE_VIEWS.VW_ACTIVE_FLEET`, a clean view over the raw legacy table.
- `USR_FDE_RO`, the limited agent login.
- Grants to select from the clean view only, plus explicit denials on the raw table and `dbo` mutations.

> The supplied SQL script contains a demonstration agent password. Change it before using anything beyond a local demo, and update `SQL_AGENT_PASSWORD` in `.env` to match.

### 3. Create the audit table

The Streamlit console records agent actions in `FDE_VIEWS.AgentAuditLog`. Create it once in the same SQL Server database:

```sql
CREATE TABLE FDE_VIEWS.AgentAuditLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    Timestamp DATETIME DEFAULT GETDATE(),
    SessionID VARCHAR(50),
    NodeExecuted VARCHAR(50),
    ToolName VARCHAR(100),
    Content NVARCHAR(MAX)
);

GRANT INSERT ON FDE_VIEWS.AgentAuditLog TO USR_FDE_RO;
```

### 4. Index SOP documents

```bash
python scripts/ingest_sop_pinecone.py
```

The script creates the selected Pinecone index when necessary:

| Embedding mode | Index | Dimension |
| --- | --- | ---: |
| `LOCAL` | `fde-sop-index-local` | 1024 |
| `OPENAI` | `fde-sop-index-openai` | 1536 |

It hashes each file and skips unchanged files. Changed or deleted files are synchronized to Pinecone. Changing embedding mode uses a separate index; do not point two embedding models with different vector dimensions at the same index.

### 5. Test the agent in the terminal

```bash
python src/orchestrator.py
```

Type a dispatcher question and enter `exit` or `quit` to close the session.

### 6. Run the dispatch console

```bash
streamlit run src/ui.py
```

The browser UI offers:

- **Dispatch Console** — chat interface with visible tool inputs and outputs.
- **Security & Audit Logs** — administrator-authenticated view of `FDE_VIEWS.AgentAuditLog`.

## Agent tools and guardrails

| Tool | Data source | Behavior |
| --- | --- | --- |
| `query_telemetry_db` | SQL Server | Executes queries through the agent identity. The application blocks commands that do not begin with `SELECT`, and fetches at most 10 rows. |
| `fetch_corridor_conditions` | Open-Meteo API | Retrieves current temperature and wind for supplied coordinates, then produces a simple corridor-risk indicator. |
| `search_compliance_sop` | Pinecone | Retrieves the two most relevant indexed SOP chunks with source metadata. |

The system prompt directs the agent to check telemetry first, then local conditions, then SOP guidance. Its final response is structured as an executive summary, a telemetry/environment table, and an SOP-cited action plan.

## Example dispatcher prompt

```text
Find active shipments near Los Angeles (latitude approximately 33.8,
longitude approximately -118.1). Check local weather and determine whether
the current cargo temperature violates the SOP for fresh perishables.
```

## Verify the database

As an administrator, check that ingestion succeeded:

```sql
SELECT COUNT(*) AS total_rows
FROM dbo.TBL_SC_FLEET_HIST_RAW;
```

Test the agent’s intended boundary while connected as `USR_FDE_RO`:

```sql
-- Expected: succeeds
SELECT TOP 5 * FROM FDE_VIEWS.VW_ACTIVE_FLEET;

-- Expected: permission denied
SELECT TOP 5 * FROM dbo.TBL_SC_FLEET_HIST_RAW;
```

## Troubleshooting

### `Library not loaded: ... libodbc.2.dylib`

Install `unixodbc`, then reinstall `pyodbc` inside the active virtual environment:

```bash
brew install unixodbc
pip uninstall -y pyodbc
pip install --no-cache-dir pyodbc
```

### `Can't open lib 'ODBC Driver 18 for SQL Server'`

Install the Microsoft SQL Server ODBC driver, then confirm it is registered:

```bash
brew install msodbcsql18
odbcinst -q -d
```

### `Login failed for user 'None'`

One or both database credential variables are missing. Ensure `.env` defines `SQL_ADMIN_USER` and `SQL_ADMIN_PASSWORD` for ingestion, plus `SQL_AGENT_USER` and `SQL_AGENT_PASSWORD` for the agent and UI.

### Pinecone or embedding dimension errors

Use the same `Embeddings_model` setting for SOP ingestion and application execution. The project maintains separate 1024-dimensional local and 1536-dimensional OpenAI indexes to avoid dimension conflicts.

## Security notes

- `.env`, credential files, private keys, and the `secrets/` directory are ignored by Git.
- Treat the passwords in the original project notes and SQL setup script as local-demo values, not production credentials.
- The application-level `SELECT` prefix check is a guardrail, not a substitute for database permissions. Keep the agent on the restricted `USR_FDE_RO` identity and preserve the SQL grants/denials.
- Review the contents of any policy documents before indexing them in a third-party vector database.
- The weather integration sends only latitude and longitude to Open-Meteo; it does not send database credentials or fleet rows.

## Technology stack

Python 3.12+, pandas, SQLAlchemy, pyodbc, Microsoft SQL Server 2022, Pinecone, LangChain, LangGraph, OpenAI/Hugging Face embeddings, Ollama/OpenAI/DeepSeek LLMs, Streamlit, and Open-Meteo.

## License

No license file is currently included. Add one before distributing or reusing the project outside its intended environment.
