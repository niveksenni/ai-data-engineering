# AI Data Engineering Workspace Summary

_Last updated: 2026-02-24_

## 1) What this solution does

This repository provides an **AI-powered data science platform** composed of:

- A reusable Python package (`ai_data_science_team`) of specialized agents
- Multi-agent orchestration flows (including a supervisor-led team)
- Streamlit applications (flagship: AI Pipeline Studio)
- Example notebooks and sample datasets

The solution supports end-to-end workflows:

1. Discover/load data
2. Wrangle and clean data
3. Run EDA and visualization
4. Perform feature engineering
5. Train/evaluate models (H2O AutoML)
6. Track artifacts/runs in MLflow

---

## 2) How it works (high-level architecture)

## 2.1 Core execution pattern

Most coding agents follow this pattern:

1. **Summarize data/context** for prompt grounding
2. **Recommend steps** (optional, can be bypassed)
3. **Generate Python/SQL code** with strict function/query requirements
4. **Execute code** in a sandboxed subprocess (or DB connection for SQL)
5. **Validate outputs** and capture structured artifacts
6. **Retry/fix loop** if errors occur (`max_retries`, `retry_count`)
7. **Return messages + artifacts + error state**

## 2.2 Base framework

- `BaseAgent` in `templates/agent_templates.py` standardizes invoke/stream/state access.
- LangGraph `StateGraph` workflows compile into runnable graphs.
- ReAct-style agents (loader/EDA/MLflow tools) call concrete tool functions.
- Utility modules handle logging, sandbox execution, parser normalization, plot conversion, and pipeline snapshots.

## 2.3 Orchestration layers

- **Single-task agents**: cleaning, wrangling, viz, SQL, feature engineering, etc.
- **Composite analysts**: 
  - `PandasDataAnalyst` (wrangle + optional chart)
  - `SQLDataAnalyst` (SQL + optional chart)
- **Supervisor team** (`supervisor_ds_team.py`): routes user intent across workers, manages shared state/datasets/artifacts, and coordinates workflow planning + execution.

---

## 3) Workspace structure summary

- `ai_data_science_team/`: core library (agents, tools, templates, utils, multiagents)
- `apps/`: Streamlit frontends
  - `ai-pipeline-studio-app/` (flagship, pipeline-first UX)
  - `exploratory-copilot-app/`
  - `pandas-data-analyst-app/`
  - `sql-database-agent-app/`
- `examples/`: notebooks demonstrating each agent/workflow
- `data/`: sample datasets for demos/tests
- `planning_docs/`: architecture and roadmap notes (notably Pipeline Studio evolution)

---

## 4) Agent I/O contract map (second pass)

Notes:
- “Inputs” are the primary state keys expected at invocation time.
- “Outputs” are principal state/artifact keys exposed by graph state/getters.
- Keys shown are based on explicit `TypedDict` state schemas + invoke/getter methods.

## 4.1 Core coding agents

### DataCleaningAgent

**Primary inputs**
- `messages`
- `user_instructions`
- `data_raw`
- `max_retries`
- `retry_count`

**Primary outputs**
- `data_cleaned`
- `data_cleaner_function`
- `recommended_steps`
- `data_cleaning_summary`
- `data_cleaner_error`
- `data_cleaner_error_log_path`

### DataWranglingAgent

**Primary inputs**
- `messages`
- `user_instructions`
- `data_raw` (single dataset dict or list of dataset dicts)
- `max_retries`
- `retry_count`

**Primary outputs**
- `data_wrangled`
- `data_wrangler_function`
- `recommended_steps`
- `data_wrangling_summary`
- `data_wrangler_error`
- `data_wrangler_error_log_path`

### FeatureEngineeringAgent

**Primary inputs**
- `messages`
- `user_instructions`
- `data_raw`
- `target_variable` (optional but commonly used)
- `max_retries`
- `retry_count`

**Primary outputs**
- `data_engineered`
- `feature_engineer_function`
- `recommended_steps`
- `feature_engineer_error`
- `feature_engineer_error_log_path`

### DataVisualizationAgent

**Primary inputs**
- `messages`
- `user_instructions`
- `data_raw`
- `max_retries`
- `retry_count`

**Primary outputs**
- `plotly_graph`
- `data_visualization_function`
- `recommended_steps`
- `data_visualization_summary`
- `data_visualization_warning`
- `data_visualization_error`
- `data_visualization_error_log_path`

### SQLDatabaseAgent

**Primary inputs**
- `messages`
- `user_instructions`
- `max_retries`
- `retry_count`
- external `connection` supplied at construction

**Primary outputs**
- `data_sql`
- `sql_query_code`
- `sql_database_function`
- `recommended_steps`
- `sql_database_error`
- `sql_database_error_log_path`

### H2OMLAgent

**Primary inputs**
- `messages`
- `user_instructions`
- `data_raw`
- `target_variable`
- `max_retries`
- `retry_count`

**Primary outputs**
- `leaderboard`
- `best_model_id`
- `model_path`
- `model_results`
- `h2o_train_function`
- `recommended_steps`
- `h2o_train_error`
- `h2o_train_error_log_path`

---

## 4.2 Tool/ReAct agents

### DataLoaderToolsAgent

**Primary inputs**
- `messages` or `user_instructions`

**Primary outputs**
- `data_loader_artifacts`
- `tool_calls`
- `internal_messages`
- `messages`

### EDAToolsAgent

**Primary inputs**
- `messages` or `user_instructions`
- `data_raw`

**Primary outputs**
- `eda_artifacts`
- `tool_calls`
- `internal_messages`
- `messages`

### MLflowToolsAgent

**Primary inputs**
- `messages` or `user_instructions`
- optional `data_raw` for prediction/tool contexts

**Primary outputs**
- `mlflow_artifacts`
- `tool_calls`
- `internal_messages`
- `messages`

### WorkflowPlannerAgent

**Primary inputs**
- `messages`
- optional `context`
- optional `user_instructions`

**Primary outputs (`get_plan`)**
- `steps` (allowed canonical step IDs)
- `target_variable`
- `questions`
- `notes`

---

## 4.3 Deterministic evaluator

### ModelEvaluationAgent

**Primary inputs**
- `messages`
- `data_raw` (DataFrame)
- optional `model_artifacts` (`model_path`, `best_model_id`)
- `target_variable`
- `test_size`
- `random_state`

**Primary outputs**
- `eval_artifacts`
- `messages`

`eval_artifacts` includes fields such as task type, metrics, model identifiers, and optional Plotly figure payloads.

---

## 4.4 Multi-agent wrappers

### PandasDataAnalyst (wrangle + optional viz)

**Primary state keys**
- Inputs/routing: `messages`, `user_instructions`, `user_instructions_data_wrangling`, `user_instructions_data_visualization`, `routing_preprocessor_decision`, `data_raw`
- Outputs: `data_wrangled`, `data_wrangler_function`, `data_visualization_function`, `plotly_graph`, `plotly_error`

### SQLDataAnalyst (SQL + optional viz)

**Primary state keys**
- Inputs/routing: `messages`, `user_instructions`, `user_instructions_sql_database`, `user_instructions_data_visualization`, `routing_preprocessor_decision`
- SQL outputs: `sql_query_code`, `sql_database_function`, `data_sql`
- Viz outputs: `plotly_graph`, `plotly_error`, `data_visualization_function`

### Supervisor DS Team

Shared team state includes:
- Conversation/routing: `messages`, `next`, `last_worker`, `handled_steps`, `workflow_plan`
- Dataset management: `datasets`, `active_dataset_id`, `active_data_key`, `target_variable`
- Data/artifacts: `data_raw`, `data_sql`, `data_wrangled`, `data_cleaned`, `feature_data`, `viz_graph`, `eda_artifacts`, `model_info`, `eval_artifacts`, `mlflow_artifacts`, `artifacts`

This layer is the main cross-agent coordinator used by Pipeline Studio.

---

## 5) Failure modes and handling

## 5.1 Common cross-agent failures

- LLM-generated code/query is syntactically or semantically invalid
- Output schema mismatch (non-table output where a table is expected)
- Missing required input fields (`data_raw`, `target_variable`, etc.)
- External dependency not installed (`h2o`, `mlflow`, optional EDA libs)
- Token/context pressure for large schemas/datasets

**Handling pattern:**
- store `*_error` keys
- optional log paths (`*_error_log_path`)
- retry/fix cycle through generated patch code
- fallback defaults in some routers (e.g., default to table mode)

## 5.2 SQL-specific safeguards

- SQL validation blocks unsafe/empty queries in safe mode
- schema-aware prompting + optional smart schema pruning reduce invalid column/table references
- explicit no-DML/no-schema-change guidance in prompts

## 5.3 Visualization-specific safeguards

- missing-column extraction and alias suggestions attempt auto-recovery
- Plotly reconstruction checks detect invalid chart payloads
- explicit warning/error separation (`data_visualization_warning` vs `data_visualization_error`)

## 5.4 Modeling/evaluation-specific safeguards

- H2O import/init guarded; target presence/type checks before training/eval
- ModelEvaluationAgent returns actionable messages when model/data/target is missing
- MLflow logging errors are treated as non-fatal in H2O flow where possible

---

## 6) App layer summary

## AI Pipeline Studio (flagship)

The Studio is a pipeline-centric workspace with lineage-aware datasets and artifact views (Table/Chart/EDA/Code/Model/Predictions/MLflow), including:
- visual editor/canvas concepts
- manual + AI node execution
- save/load project modes (metadata-only or full-data)
- reproducible script/spec generation
- lineage operations (replay, stale marking, undo/redo, compare)

## Other apps

- Exploratory Copilot: EDA-first chat workflow over uploaded/demo data
- Pandas Data Analyst App: natural-language wrangling + optional chart routing
- SQL Database Agent App: natural-language SQL querying with table results

---

## 7) Current maturity and gaps

**Mature/strong areas**
- Practical app UX for iterative DS workflows
- Broad task coverage across data prep, EDA, ML, and MLflow
- Structured stateful orchestration with artifact propagation

**Known gaps**
- Top-level `orchestration.py` remains mostly TODO
- Some flows are large/complex and rely on prompt robustness
- Beta status implies potential breaking API/behavior changes

---

## 8) Practical takeaway

This workspace is best viewed as a **modular AI DS operating system**:
- single agents for focused tasks,
- composite agents for common analyst interactions,
- and a supervisor + Pipeline Studio for end-to-end, reproducible, multi-step workflows.

---

## 9) One-page architecture diagram (Mermaid)

```mermaid
flowchart TB
  U[User] --> A[Streamlit Apps]
  A --> APS[AI Pipeline Studio]
  A --> EDAAPP[Exploratory Copilot App]
  A --> PDAPP[Pandas Data Analyst App]
  A --> SQLAPP[SQL Database Agent App]

  APS --> SUP[Supervisor DS Team\nLangGraph StateGraph]

  SUP --> PLAN[WorkflowPlannerAgent]
  SUP --> LOAD[DataLoaderToolsAgent]
  SUP --> WR[DataWranglingAgent]
  SUP --> CLN[DataCleaningAgent]
  SUP --> EDA[EDAToolsAgent]
  SUP --> VIZ[DataVisualizationAgent]
  SUP --> SQL[SQLDatabaseAgent]
  SUP --> FE[FeatureEngineeringAgent]
  SUP --> H2O[H2OMLAgent]
  SUP --> EVAL[ModelEvaluationAgent]
  SUP --> MLF[MLflowToolsAgent]

  PDAPP --> PMA[PandasDataAnalyst\nWrangle + Optional Viz]
  SQLAPP --> SMA[SQLDataAnalyst\nSQL + Optional Viz]

  PMA --> WR
  PMA --> VIZ
  SMA --> SQL
  SMA --> VIZ

  LOAD --> TLOAD[tools/data_loader.py]
  EDA --> TEDA[tools/eda.py]
  SQL --> TSQL[tools/sql.py]
  H2O --> TH2O[tools/h2o.py]
  MLF --> TMLF[tools/mlflow.py]

  WR --> SBX[Sandboxed Code Execution\nutils/sandbox.py]
  CLN --> SBX
  VIZ --> SBX
  FE --> SBX
  H2O --> SBX

  SUP --> DS[(Dataset Registry + Lineage)]
  SUP --> ART[(Artifacts Store)]
  SUP --> LOG[(logs/)]

  DS --> OUT1[Table / Chart / EDA / Code Views]
  ART --> OUT2[Model / Predictions / MLflow Views]

  DATA[(CSV/XLSX/Parquet/DB)] --> LOAD
  DB[(SQL Database)] --> SQL
  MLFLOW[(MLflow Tracking Server)] --> MLF

  subgraph LIB[ai_data_science_team package]
    SUP
    PLAN
    LOAD
    WR
    CLN
    EDA
    VIZ
    SQL
    FE
    H2O
    EVAL
    MLF
    PMA
    SMA
  end
```

---

## 11) DataOps Reference Architecture (Recommended Next Layer)

This section outlines a practical DataOps layer to make the current agentic platform production-ready for reliability, governance, and scale.

### 11.1 Operating Model (DataOps + MLOps + LLMOps)

```mermaid
flowchart LR
  subgraph UX[Experience Layer]
    APPS[Streamlit Apps\nPipeline Studio + Copilots]
    API[Programmatic API / CLI]
  end

  subgraph ORCH[Orchestration Layer]
    SCHED[Workflow Scheduler\nDagster / Prefect / Airflow]
    SUP[Supervisor + Agents\nLangGraph Runtime]
    GATE[Quality & Policy Gates\npre/post step checks]
  end

  subgraph DATA[Data Plane]
    RAW[Raw Data Sources\nFiles + SQL]
    CUR[Curated Datasets\nVersioned snapshots]
    FEAT[Feature/Training Views]
    ART[Artifact Store\nplots/reports/models]
  end

  subgraph OBS[Observability Plane]
    LOGS[Structured Logs]
    METRICS[Metrics + SLOs]
    TRACES[Distributed Traces]
    ALERT[Alerts + On-call]
  end

  subgraph GOV[Governance & Security]
    CATALOG[Data Catalog + Lineage]
    DQ[Data Quality Rules]
    ACCESS[RBAC + Secrets + Audit]
    POLICY[Execution Policies\nSQL/code/PII guardrails]
  end

  subgraph EXP[Experimentation & Delivery]
    REG[MLflow Tracking + Registry]
    EVAL[Offline/Online Eval]
    PROMPT[Prompt/Agent Versioning]
    ROLLOUT[Canary + Rollback]
  end

  APPS --> SUP
  API --> SUP
  SCHED --> SUP
  SUP --> GATE

  RAW --> SUP
  SUP --> CUR
  CUR --> FEAT
  SUP --> ART
  FEAT --> REG

  SUP --> LOGS
  SUP --> METRICS
  SUP --> TRACES
  METRICS --> ALERT

  CUR --> CATALOG
  FEAT --> DQ
  SUP --> ACCESS
  GATE --> POLICY

  REG --> EVAL
  SUP --> PROMPT
  EVAL --> ROLLOUT
```

### 11.2 Capability Map and Priority Sequence

```mermaid
flowchart TB
  subgraph P30[0-30 Days: Foundation]
    SLI[Define SLIs/SLOs\nrun success, latency, retries]
    OBS1[Structured run telemetry\ntrace_id + run_id + cost]
    DASH[Dashboards\nhealth, cost, quality]
    DQ0[Basic quality checks\nschema + freshness + null spikes]
  end

  subgraph P60[31-60 Days: Control]
    ORCH1[Scheduled workflows\nwith retries/backoff]
    LINEAGE[Dataset + artifact lineage\nversioned snapshots]
    POLICY1[Policy gates\nSQL safety, sandbox controls, PII checks]
    TEST1[Contract tests\nagent I/O + routing behavior]
  end

  subgraph P90[61-90 Days: Operate]
    ALERT1[Production alerting\nSLO burn-rate + incidents]
    REL[Release management\ncanary, rollback, change approval]
    FINOPS[Cost controls\nbudgets, model routing strategy]
    AUDIT[Audit-ready operations\nretention, access logs, evidence]
  end

  P30 --> P60 --> P90

  SLI --> ORCH1
  OBS1 --> ALERT1
  DQ0 --> LINEAGE
  POLICY1 --> AUDIT
  TEST1 --> REL
  REL --> FINOPS
```

### 11.3 Minimal Target Stack (fit-for-purpose now)

- Orchestration: Prefect or Dagster for scheduled/managed pipeline execution.
- Observability: OpenTelemetry + Prometheus/Grafana + centralized logs.
- Data quality: Great Expectations or Soda checks at ingestion and pre-model steps.
- Metadata/lineage: OpenLineage-compatible metadata collection and dataset fingerprint tracking.
- Governance: secret management, role-based access, policy checks before SQL/code execution.
- Delivery: prompt/agent versioning with canary rollout and rollback playbooks.

### 11.4 DataOps Control Overlay on C4 Level 2

```mermaid
flowchart TB
  subgraph CORE[C4 Level 2 Containers (Current)]
    APPS[Streamlit Application Suite]
    SUP[Supervisor Orchestration]
    AGENTS[Specialist Agents]
    MULTI[Composite Multi-Agent Flows]
    TOOLS[Tools Layer]
    UTILS[Execution & Utility Layer]
    PSTORE[(Pipeline Store)]
    LSTORE[(Log Store)]
  end

  subgraph CTRL[DataOps Control Plane (Recommended)]
    ORCH[Workflow Orchestration\nSchedules + retries + backoff]
    OBS[Observability\nLogs/Metrics/Traces + SLOs]
    DQ[Data Quality\nSchema/freshness/volume checks]
    LIN[Lineage & Metadata\nOpenLineage + dataset fingerprinting]
    GOV[Governance\nRBAC, secrets, audit, PII policy]
    REL[Release Controls\nPrompt/agent versioning, canary, rollback]
  end

  subgraph EXT[External Systems]
    SRC[Data Sources\nFiles + SQL]
    LLM[LLM Providers\nOpenAI/Ollama]
    MLF[MLflow Tracking]
    ALERT[Alerting/On-call]
  end

  APPS --> SUP
  APPS --> MULTI
  SUP --> AGENTS
  MULTI --> AGENTS
  AGENTS --> TOOLS
  AGENTS --> UTILS
  SUP --> PSTORE
  APPS --> PSTORE
  AGENTS --> LSTORE

  AGENTS --> SRC
  AGENTS --> LLM
  AGENTS --> MLF

  ORCH --> SUP
  ORCH --> MULTI
  OBS --> SUP
  OBS --> AGENTS
  OBS --> LSTORE
  OBS --> ALERT
  DQ --> AGENTS
  DQ --> PSTORE
  LIN --> SUP
  LIN --> PSTORE
  GOV --> SUP
  GOV --> AGENTS
  GOV --> TOOLS
  REL --> SUP
  REL --> AGENTS

  classDef control fill:#eef7ff,stroke:#4a90e2,stroke-width:1px;
  class ORCH,OBS,DQ,LIN,GOV,REL control;
```

---

## 12) Glossary (Project Terminology)

- **Team**: A coordinated group of specialized agents operating under shared state and routing logic to complete end-to-end workflows.
- **Agent**: A role-focused execution unit (e.g., DataCleaningAgent, SQLDatabaseAgent, H2OMLAgent) that performs a specific skill.
- **Worker**: The active agent selected by the orchestrator to run the next step for a given request.
- **Orchestrator / Supervisor**: The routing and coordination layer (supervisor DS team) that decides which worker runs next, manages state, and aggregates artifacts.
- **Tool**: A deterministic callable function used by an agent (e.g., file loading, EDA routines, SQL helpers, MLflow operations).
- **Multi-agent flow**: A wrapper composition of agents for common tasks (e.g., PandasDataAnalyst, SQLDataAnalyst).
- **Artifact**: Structured output produced by runs (tables, plotly graphs, EDA reports, model/eval info, MLflow run details).

---

## 13) DataOps Operating Table (Control → Owner → KPI → Tooling)

| Control Area | Suggested Owner | Core KPI / SLO | Practical Tooling (Current Fit) |
|---|---|---|---|
| Workflow Orchestration | Data Platform / MLOps Engineer | Run success %, median + p95 duration, retry rate | Prefect or Dagster, cron-backed schedules |
| Observability | Platform / SRE | Error rate, SLO burn rate, MTTR, alert precision | OpenTelemetry, Prometheus + Grafana, centralized logs |
| Data Quality | Analytics Engineering / DataOps | Freshness SLA, schema drift incidents, null/volume anomaly rate | Great Expectations or Soda checks |
| Lineage & Metadata | Data Governance / Platform | % runs with complete lineage, reproducibility pass rate | OpenLineage-compatible metadata capture, dataset fingerprints |
| Governance & Security | Security + Data Governance | Access violations, secrets exposure incidents, audit completeness | RBAC, secrets manager, policy checks for SQL/code/PII |
| Agent Quality | AI Engineering | Routing accuracy, invalid-code rate, fallback frequency | Contract tests, regression prompt suites, golden datasets |
| Release Management | AI Engineering + Platform | Change failure rate, rollback time, canary success rate | Versioned prompts/agents, staged rollout, rollback automation |
| Cost / FinOps | Engineering Manager / FinOps | Cost per run, cost per successful run, budget variance | Usage telemetry, model routing policies, budget alerts |

### 13.1 Suggested Review Cadence

- **Daily**: failed runs, high-severity alerts, freshness breaches.
- **Weekly**: routing accuracy, retry hotspots, top-cost workflows.
- **Monthly**: SLO trends, control effectiveness, roadmap reprioritization.

---

## 10) C4 Model Diagrams (Mermaid)

### Level 1: System Context Diagram

```mermaid
C4Context
  title System Context Diagram (L1) - AI Data Science Team Workspace

  Enterprise_Boundary(userBoundary, "Users") {
    Person(dsUser, "Data Scientist / Analyst", "Loads data, asks questions, reviews outputs, and iterates workflows")
  }

  Enterprise_Boundary(workspaceBoundary, "AI Data Science Team Workspace") {
    System(appSuite, "AI Data Apps (Streamlit)", "UI layer: AI Pipeline Studio, Exploratory Copilot, Pandas Analyst, SQL Database Agent")
    System(coreLib, "AI Data Science Team Library", "Agentic data-science engine for loading, wrangling, cleaning, EDA, visualization, modeling, and evaluation")
  }

  Enterprise_Boundary(modelProviders, "LLM Providers") {
    System_Ext(openaiApi, "OpenAI API", "Hosted LLM inference")
    System_Ext(ollamaApi, "Ollama", "Local/self-hosted LLM inference")
  }

  Enterprise_Boundary(dataSystems, "Data & Tracking Systems") {
    System_Ext(fileSystem, "Local Filesystem / Datasets", "CSV, Excel, JSON, Parquet, etc.")
    System_Ext(sqlDb, "SQL Databases", "Queryable relational sources via SQLAlchemy")
    System_Ext(mlflowSrv, "MLflow Tracking", "Experiment tracking, artifacts, model registry/UI")
  }

  Rel(dsUser, appSuite, "Uses interactive apps", "Browser/Streamlit")
  Rel(appSuite, coreLib, "Invokes agents and orchestrated workflows", "In-process Python")
  Rel(coreLib, openaiApi, "Calls LLM models", "API")
  Rel(coreLib, ollamaApi, "Calls local LLM models", "HTTP")
  Rel(coreLib, fileSystem, "Loads/saves datasets, logs, and project artifacts", "File I/O")
  Rel(coreLib, sqlDb, "Queries tabular data", "SQL via SQLAlchemy")
  Rel(coreLib, mlflowSrv, "Logs and retrieves run artifacts/models", "MLflow API")
```

### Level 2: Container Diagram

```mermaid
C4Container
  title Container Diagram (L2) - AI Data Science Team Workspace

  Person(dsUser, "Data Scientist / Analyst", "Runs exploratory and production-like DS workflows")

  System_Boundary(workspace, "AI Data Science Team Workspace") {
    Container(apps, "Streamlit Application Suite", "Python + Streamlit", "Front-end apps for chat, pipeline editing, artifact views, and project save/load")

    Container(supervisor, "Supervisor Orchestration", "LangGraph + LangChain", "Routes requests across specialist agents, manages workflow plans, shared state, lineage, and artifacts")

    Container(singleAgents, "Specialist Agents", "LangGraph agents", "DataLoader, Wrangling, Cleaning, EDA, Visualization, SQL, Feature Engineering, H2O ML, Model Evaluation, MLflow Tools")

    Container(multiAgents, "Composite Multi-Agent Flows", "LangGraph", "PandasDataAnalyst and SQLDataAnalyst wrappers for task-specific orchestration")

    Container(tools, "Tools Layer", "Python tool modules", "Deterministic task tools for file loading, EDA routines, SQL helpers, H2O, and MLflow operations")

    Container(utils, "Execution & Utility Layer", "Python utilities", "Sandboxed subprocess execution, logging, parsers, plotting adapters, and pipeline snapshot utilities")

    ContainerDb(pipelineStore, "Pipeline Store", "Local files (JSON/Parquet/Pickle)", "Dataset registry cache, project snapshots, artifact indexes, layout/code drafts")
    ContainerDb(logStore, "Log Store", "Local files", "Generated code and error logs for agent runs")
  }

  System_Ext(openaiApi, "OpenAI API", "Hosted LLM inference")
  System_Ext(ollamaApi, "Ollama", "Local/self-hosted LLM inference")
  System_Ext(fileSystem, "Local Filesystem / Datasets", "Input and output datasets")
  System_Ext(sqlDb, "SQL Databases", "Relational data sources")
  System_Ext(mlflowSrv, "MLflow Tracking", "Runs, artifacts, registry, UI")

  Rel(dsUser, apps, "Uses", "Browser/Streamlit")
  Rel(apps, supervisor, "Submits workflow requests and receives artifacts/state")
  Rel(apps, multiAgents, "Invokes focused multi-agent flows in specific apps")

  Rel(supervisor, singleAgents, "Invokes specialist agents based on plan + routing")
  Rel(multiAgents, singleAgents, "Composes specialist agents for chart/table and SQL/table flows")

  Rel(singleAgents, tools, "Calls deterministic task tools")
  Rel(singleAgents, utils, "Uses parsers, sandbox execution, and logging")
  Rel(supervisor, utils, "Builds/updates pipeline snapshots and lineage metadata")

  Rel(singleAgents, openaiApi, "Calls LLMs", "API")
  Rel(singleAgents, ollamaApi, "Calls local LLMs", "HTTP")
  Rel(singleAgents, fileSystem, "Loads data / writes outputs", "File I/O")
  Rel(singleAgents, sqlDb, "Reads data", "SQL")
  Rel(singleAgents, mlflowSrv, "Logs + retrieves run artifacts/models", "MLflow API")

  Rel(supervisor, pipelineStore, "Reads/writes dataset registry, artifacts, project state")
  Rel(apps, pipelineStore, "Reads/writes UI state, layout, drafts, and project files")
  Rel(singleAgents, logStore, "Writes generated code + error logs")
```

