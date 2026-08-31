---
title: RCA Investigation Agent — Full Specification
project: Mankind Pharma (DP)
author: Preyas Kulshreshtha, SDM — Celebal Technologies
version: v0.1
date: August 2026
status: Draft — pending leadership approval
---

# RCA Investigation Agent — Full Specification

> This document is the single source of truth for the Foundry-hosted RCA Investigation Agent for the Mankind Pharma DP project. It covers what the agent is, how it works, what it connects to, how it authenticates, what it produces, how it is built, and what comes next.

---

## Table of contents

1. [Background and problem statement](#1-background-and-problem-statement)
2. [What this agent is](#2-what-this-agent-is)
3. [How it differs from the existing agents](#3-how-it-differs-from-the-existing-agents)
4. [Four investigation personas](#4-four-investigation-personas)
5. [Five-stage investigation pipeline](#5-five-stage-investigation-pipeline)
6. [RCA report format](#6-rca-report-format)
7. [System architecture](#7-system-architecture)
8. [ADO repo connections](#8-ado-repo-connections)
9. [Databricks connection](#9-databricks-connection)
10. [Power BI / Fabric connection](#10-power-bi--fabric-connection)
11. [ADF connection](#11-adf-connection)
12. [Azure SQL connection](#12-azure-sql-connection)
13. [Jira connection](#13-jira-connection)
14. [Identity and security model](#14-identity-and-security-model)
15. [What the agent never does](#15-what-the-agent-never-does)
16. [Front end options](#16-front-end-options)
17. [Feasibility on Microsoft Foundry](#17-feasibility-on-microsoft-foundry)
18. [Future roadmap — RAG and graph database](#18-future-roadmap--rag-and-graph-database)
19. [Skill inventory and build plan](#19-skill-inventory-and-build-plan)
20. [Skill carry-over from existing agents](#20-skill-carry-over-from-existing-agents)
21. [Risks and mitigations](#21-risks-and-mitigations)
22. [Success metrics](#22-success-metrics)
23. [Next steps after approval](#23-next-steps-after-approval)
24. [Glossary](#24-glossary)

---

## 1. Background and problem statement

The Mankind Pharma data platform runs across four connected layers:

```
ADF (ingestion) → Databricks (bronze → silver → gold) → Power BI / Fabric (models + reports)
                                                       → Superman Portal / Pulse (reports)
```

When a data issue is reported — a wrong number in a Power BI visual, a mismatch in Pulse, a Databricks pipeline failure, an ADF ingestion error — a senior engineer must manually trace evidence across all four layers to find the root cause. This takes 3–6 hours per incident. There is no consistent output format, no institutional memory of past cases, and no coverage at all for Databricks and ADF issues.

**The goal of this agent is to automate the entire investigation**, produce a structured Root Cause Analysis (RCA) report, and wait for a human to confirm before posting anything to Jira.

---

## 2. What this agent is

The RCA Investigation Agent is a **Microsoft Foundry-hosted AI agent** (Claude, Anthropic Agent SDK) that:

- Is triggered by a Jira ticket entering the `Investigation` status in the DP project, by a manual prompt, or by a scheduled sweep
- Classifies the ticket to one or more investigation planes (PBI, Pulse, Databricks, ADF)
- Runs a five-stage investigation pipeline: symptom capture → data pull → lineage trace → root cause analysis → fix recommendation
- Produces a structured six-section RCA report
- Waits for a human to review and confirm before posting the report as a Jira comment

It is **not a development agent**. It never writes code, opens pull requests, modifies data, or triggers pipeline runs. The only write action in the entire system is posting the confirmed RCA comment to Jira.

It runs as a **Linux container on Microsoft Foundry** (Azure Container Apps), unattended, 24/7, using service principal identities and secrets managed through Azure Key Vault.

---

## 3. How it differs from the existing agents

Two existing agents are relevant context:

### CR-Dev-Agent (Claude Code, local)
A Claude Code workspace that automates the full development lifecycle for the Mankind data platform — from a Jira story through requirement analysis, impact analysis, implementation plan, human approval, code authoring, testing, and pull request. It covers four personas (Dev-Agent, CR-Impact-Analyst, PBI-Support-Agent, Pulse-Support-Agent) and 20 skills. It runs on an engineer's Windows laptop, bound to that person's interactive browser logins.

### Azure Fabric Report agent (Claude Code, local)
A Claude Code workspace pointed at the Azure Fabric Report repo. It supports Power BI and Pulse investigations only, returning a verdict (bug vs working-as-designed) plus a draft Jira reply. It uses the PBI Modeling MCP (a Windows VS Code extension binary), requires interactive AAD sign-in, and has no ADF or Databricks investigation capability.

### How the RCA agent differs

| Dimension | Existing agents | RCA Investigation Agent |
|---|---|---|
| Planes covered | PBI + Pulse only | PBI · Pulse · Databricks · ADF |
| Output | Verdict + draft Jira reply | Structured 6-section RCA with confidence score |
| Hosting | Engineer's Windows laptop, interactive | Foundry Linux container, unattended 24/7 |
| Authentication | Per-user browser logins, stops on token expiry | Service principals, Key Vault, always running |
| Power BI access | VS Code Windows extension (MCP) | Fabric REST + XMLA, no extension dependency |
| Investigation structure | Freeform, engineer-directed | Five mandatory stages, non-skippable |
| Front end | Terminal only | Teams bot / web app / Jira embedded button |
| Learning loop | Manual retro, human-confirmed cases | Persistent case library on git-backed volume |
| Browser / screenshots | Playwright (named-user PII risk) | Not included — log-based evidence only |
| Writes code | Yes (Dev-Agent, approval-gated) | Never |

---

## 4. Four investigation personas

The intake router reads the incoming Jira ticket and dispatches to one or more personas. Tickets that span layers (e.g. a PBI number wrong because a Databricks job failed) can trigger multiple personas in sequence.

### PBI-Investigation
Investigates Power BI and Fabric report issues. Traces from a visual through the DAX measure, the semantic model, the gold table, and back to the ETL notebook that writes it. Uses Fabric REST and XMLA endpoints — no VS Code extension required.

**Typical issues handled:** wrong numbers in visuals, RLS filter problems, dataset refresh failures, measure calculation errors, model table column mismatches.

### Pulse-Investigation
Investigates Superman Portal (Pulse) data issues. Traces from a Pulse view through the gold table, the silver/bronze layers, and the ADF ingestion pipeline. Includes validation queries against the Azure SQL Pulse serving database (`mnkDnAIdb`).

**Typical issues handled:** Pulse report showing stale data, field values not matching source, FLM-to-MR reconciliation mismatches, visit frequency or POB calculation discrepancies.

### DBX-Investigation *(new — no existing analogue)*
Investigates Databricks pipeline and ETL failures. Reads Unity Catalog job run history, cluster event logs, task-level failure reasons, and upstream table snapshots from `mnk_prod`.

**Typical issues handled:** notebook failures, schema drift in bronze/silver tables, UC permission errors, job timeout, upstream data quality failures propagating to gold.

### ADF-Investigation *(new — no existing analogue)*
Investigates Azure Data Factory pipeline failures. Reads pipeline run history, activity-level failure details, trigger state, and linked service connectivity from the ADF REST API.

**Typical issues handled:** ingestion pipeline not running, activity-level failures, linked service credential expiry, trigger not firing, copy activity timeout.

---

## 5. Five-stage investigation pipeline

Every persona runs the same five stages. Stages are not skippable or reorderable. Only the data sources queried in each stage differ by persona.

### Stage 1 — Symptom capture
Reads the full Jira ticket including all comments, attachments, and linked issues. Extracts the reported values, expected values, affected users, and time of first occurrence. Sets the investigation scope.

**Data sources:** Jira REST API (`/issue/{key}`, `/issue/{key}/attachments`), JQL search for related tickets.

### Stage 2 — Data pull
Queries the relevant system for raw evidence. For PBI: DAX query results, dataset refresh history, measure definitions. For Pulse: gold table query results, Azure SQL validation queries. For DBX: job run history, cluster event logs, task failure messages. For ADF: pipeline run history, activity failure details.

**Data sources:** Fabric XMLA + REST, Databricks SQL warehouse, Azure SQL, ADF REST API.

### Stage 3 — Lineage trace
Walks upstream from the affected output to the source. Uses `lineage-trace` and `gold-lineage` skills to map the dependency chain across repos. For PBI: visual → DAX → model table → gold table → ETL notebook. For ADF: failed pipeline → linked dataset → source system.

**Data sources:** ADO repos (read-only) — Azure Fabric Report, Mankind_Databricks_Sql_MigrationWS, MnkSqlDBMigrationADFProd.

### Stage 4 — Root cause analysis
Assembles all evidence collected in stages 1–3. Classifies the failure type from a defined taxonomy. Assigns a confidence score (High / Medium / Low) based on directness of evidence.

**Failure taxonomy** (aligned to DP project bug taxonomy custom fields):
- Data quality — bad source data propagated downstream
- Transform logic — incorrect notebook or DAX logic
- Schema drift — column added, renamed, or dropped upstream
- Pipeline timeout — job or activity exceeded time limit
- Credential expiry — linked service or SP secret expired
- Permission error — UC grant or workspace role missing
- Configuration drift — trigger, parameter, or linked service misconfigured
- Working as designed — behaviour matches spec; expectation is wrong

### Stage 5 — Fix recommendation
Describes what needs to change, which layer owns it, and who should action it. No code is written here. The recommendation is a description, not a diff or a patch.

---

## 6. RCA report format

The agent produces a structured six-section Markdown document, saved to `.claude/rca-cases/<TICKET-KEY>-<YYYY-MM-DD>.md` on the persistent volume and committed to a git branch. After a human reviews and confirms, it is posted as a Jira comment on the originating ticket.

```markdown
## RCA — <TICKET-KEY> — <date>

**Confidence:** High / Medium / Low
**Root cause type:** <from taxonomy>
**Affected layer:** <PBI / Pulse / Databricks / ADF / cross-layer>
**Investigation duration:** <time from trigger to report>

### Summary
One-paragraph verdict. What is wrong, where it originated, and what the impact is.

### Timeline
| Time | Event |
|---|---|
| <timestamp> | Issue first occurred (inferred from logs) |
| <timestamp> | Issue reported in Jira |
| <timestamp> | Investigation triggered |

### Evidence
Quoted log lines, query results, DAX outputs, job run details — all grounded
in data actually retrieved during the investigation. No fabricated evidence.

### Root cause
Specific, classified root cause. Direct evidence linking the symptom to the cause.
Confidence score with justification.

### Impact
Which reports, tables, pipelines, or user groups are affected.
Estimated data gap period if determinable.

### Recommendation
Specific action, owning team, and priority. No code — a description of what needs to change.
```

---

## 7. System architecture

```
┌─ Triggers ─────────────────────────────────────────────────────────┐
│  Jira webhook (new DP ticket)  ·  Manual prompt  ·  Scheduled sweep │
└───────────────────────────────────────────────────────┬────────────┘
                                                        ↓
┌─ Microsoft Foundry — Linux container ──────────────────────────────┐
│                                                                     │
│  ┌─ Intake router ────────────────────────────────────────────┐    │
│  │  Reads ticket · classifies plane · dispatches persona(s)   │    │
│  └────────────────────────────────────────────────────────────┘    │
│       ↓              ↓              ↓              ↓               │
│  PBI-Invest.  Pulse-Invest.  DBX-Invest.    ADF-Invest.           │
│       └──────────────┴──────────────┴──────────────┘               │
│                              ↓                                      │
│  ┌─ Five-stage pipeline ──────────────────────────────────────┐    │
│  │  1 Symptom  →  2 Data pull  →  3 Lineage  →  4 RCA  →  5 Fix│   │
│  └────────────────────────────────────────────────────────────┘    │
│                              ↓                                      │
│  ┌─ RCA report writer ────────────────────────────────────────┐    │
│  │  Summary · Timeline · Evidence · Root cause · Impact · Rec. │    │
│  │  Saved to .claude/rca-cases/ (persistent volume + git)      │    │
│  └────────────────────────────────────────────────────────────┘    │
│                              ↓                                      │
│  ┌─ Human confirm gate ───────────────────────────────────────┐    │
│  │  Review via Teams bot / web app / Jira button               │    │
│  │  Confirm → Jira comment posted  (only write in the system)  │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ read-only
┌─ Data plane ────────────────────────────────────────────────────────┐
│  Power BI / Fabric  ·  Databricks UC  ·  ADO repos ×3              │
│  ADF  ·  Jira (DP)  ·  Azure SQL (mnkDnAIdb)                       │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─ Identity layer ────────────────────────────────────────────────────┐
│  Azure Key Vault — all secrets injected at container startup        │
│  SP-B (Fabric · Databricks · ADF · Azure SQL)                      │
│  SP-C (Jira bot account · DP project scope only)                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. ADO repo connections

Three repositories are cloned at container startup using a read-only ADO deploy key. The agent reads these for lineage context only — it never commits, pushes, or modifies any file in any repo.

| Repo | ADO org | Branch | Used by | Purpose |
|---|---|---|---|---|
| Azure Fabric Report (`AzureFabricReport`) | AzureFabricReport | `prod2_main` | PBI-Investigation | Power BI / Fabric PBIP and TMDL files — DAX measures, model structure |
| Mankind_Databricks_Sql_MigrationWS | DBSQL-MIG | `prod_main` | DBX-Investigation, Pulse-Investigation | ETL notebooks that build gold.* tables |
| MnkSqlDBMigrationADFProd | DBSQL-MIG | `main` | ADF-Investigation | ADF pipeline definitions, triggers, linked services |

**Connection method:** Read-only deploy key (PAT), scoped to these three repos, no push permission. Stored in Key Vault as `ADO_DEPLOY_KEY`. Repos are cloned to known paths at container startup via env vars `AFR_REPO`, `ETL_REPO`, `ADF_REPO`.

**Important note from CR-Dev-Agent architecture:** Two Fabric repos exist (`MankindPharma-PowerBIReports` and `AzureFabricReport`). The RCA agent clones both. Every cross-repo lineage search must scan both or blast radius is silently under-reported.

**Live-synced branches (do not push):** `main` and `HO_Production` on `MankindPharma-PowerBIReports`, `prod2_main` and `main` on `AzureFabricReport` are live-synced to Fabric workspaces — a push to any of them is a production deployment. The RCA agent has no push permission, but this should be validated at the ADO branch policy level, not just at the agent level.

---

## 9. Databricks connection

**Workspace:** `adb-842962623037618.18.azuredatabricks.net`
**SQL warehouse:** `efc9fed520454af6` (Dev_sql_warehouse)
**Catalogs in scope:** `mnk_prod` (read-only), `mnk_dev` (read-only for investigation)
**Schemas in scope:** `gold`, `pulse`

**Connection method:** Databricks Python SDK (`databricks-sdk`), `auth_type="oauth-m2m"` with `DATABRICKS_CLIENT_ID` and `DATABRICKS_CLIENT_SECRET` from Key Vault. This replaces the current `auth_type="azure-cli"` user-delegated token used in the existing agents.

**UC grants required for SP-B:**
- `USE CATALOG` on `mnk_prod`
- `USE SCHEMA` on `mnk_prod.gold` and `mnk_prod.pulse`
- `SELECT` on all tables within `mnk_prod.gold.*` and `mnk_prod.pulse.*`
- `CAN USE` on warehouse `efc9fed520454af6`

**What the agent reads:** Job run history, cluster event logs, task-level failure messages, table row counts and partition snapshots for validation SQL, Unity Catalog metadata (table schemas, lineage).

**What the agent never does:** `INSERT`, `UPDATE`, `DELETE`, `DROP`, `MERGE`, trigger a job, modify a cluster, or change any UC object.

**CLI:** Databricks CLI is used for metadata queries (`databricks jobs list`, `databricks clusters list`). Auth is the same M2M service principal, configured via env vars at startup.

---

## 10. Power BI / Fabric connection

**Workspaces in scope:**
- `MNK-Fabric-Prod-Workspace`
- `MNK-Fabric-PROD-Workspace_2`
- `HO-Production`

**Connection method:** AAD service principal (SP-B) with "Service principals can use Fabric APIs" enabled in the Mankind Pharma tenant. Two access paths:
- **Fabric REST API** — workspace metadata, dataset refresh history, report structure
- **XMLA endpoint** — model metadata (measures, tables, columns, RLS roles), DAX query execution

This replaces the Power BI Modeling MCP (a Windows VS Code extension binary, `powerbi-modeling`) used in the existing agents. The MCP is not available on a Linux container and requires interactive AAD sign-in. The REST + XMLA path has no OS dependency and works with a service principal.

**SP-B role on each workspace:** Workspace Viewer (sufficient to read model structure and run DAX queries; not sufficient to publish, refresh, or edit anything).

**What the agent reads:** Dataset refresh history and failure details, measure definitions, table and column metadata, RLS role definitions, DAX query results for validation, model parameters.

**What the agent never does:** Publish a model, trigger a dataset refresh, modify a report, change workspace settings, or add/remove workspace members.

**Important:** Before Phase 2 goes live, validate that "Service principals can use Fabric APIs" is enabled in the Mankind Pharma AAD tenant settings. This is a tenant-level switch that must be on for XMLA and Fabric REST to work with a service principal. If it is off, Phase 2 is blocked.

---

## 11. ADF connection

**Factory:** `MnkSqlDBmigrationADF-PROD`
**API:** Azure Resource Manager REST API (`management.azure.com`)
**Endpoint pattern:** `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DataFactory/factories/{factory}`

**Connection method:** SP-B with **Reader** role on the ADF resource group. Deliberately not Contributor — with Reader only, `pipeline create-run` cannot succeed even if an agent rule is bypassed.

**What the agent reads:** Pipeline run history (`queryByFactory`), activity-level failure details, trigger state (enabled/disabled, last run, next run), linked service names and connection status.

**What the agent never does:** Trigger a pipeline run, modify a trigger, change a linked service configuration, or update any pipeline definition.

**CLI:** `az datafactory query-by-factory` with `az extension add -n datafactory`. Auth via SP-B service principal credentials from env vars.

---

## 12. Azure SQL connection

**Server:** `mnkdnaisql.database.windows.net`
**Database:** `mnkDnAIdb`

This is the Pulse app serving database. It is queried during Pulse investigations to validate whether data in the serving layer matches what is in the gold tables, helping distinguish between a Databricks issue and a serving/API layer issue.

**Connection method:** SP-B with `db_datareader` role on `mnkDnAIdb`. Connection string stored in Key Vault as `SQL_CONNECTION_STRING`.

**What the agent reads:** Pulse-related tables for validation SQL only. Read-only.

**What the agent never does:** `INSERT`, `UPDATE`, `DELETE`, or modify any schema object.

---

## 13. Jira connection

**Instance:** `mankindpharmaltd.atlassian.net`
**Cloud ID:** `6509a8e3-abf2-431e-a23c-c683f867e08c`
**Project scope:** DP (Mankind Engagements)

**Connection method:** Dedicated bot account (`agent-bot@celebaltech.com` or equivalent) with an Atlassian API token. Basic Auth on all REST calls (`btoa("email:token")`). No browser OAuth, no `mcp-remote`.

This replaces the `mcp-remote` OAuth path used in the existing Azure Fabric Report agent (which had deprecated endpoint issues) and the Copilot Studio RCA Agent OAuth connector.

**Read operations:**
- `GET /rest/api/3/issue/{key}` — ticket detail
- `POST /rest/api/3/search/jql` — JQL search for related tickets
- `GET /rest/api/3/issue/{key}/attachments` — attachment metadata
- `GET /rest/api/3/attachment/{id}` — attachment download (via `jira_attachment.py`)

**Write operation (one, gated):**
- `POST /rest/api/3/issue/{key}/comment` — post the confirmed RCA report as a comment

This is the only write action in the entire agent. It is not in the agent's tool allow-list by default. It is only exposed via the front end confirm workflow — a human must explicitly click confirm before the POST is made.

**Jira bot account permissions required on the DP project:**
- Browse Projects
- View Issues
- Add Comments
- Create Attachments (for downloading attachments)

**Alignment with DP Jira workflow:** The agent is designed to handle tickets in the `Investigation` status. The 13-status DP workflow (`On Hold → To-do → In Dev → In QA → In UAT → Info Required → Investigation → Is Blocked → PR Review → Cancelled → Ready for deployment → Prod Deployed → Closed`) includes `Investigation` as a valid status. The agent does not transition tickets — it only reads the ticket and posts a comment.

**Alignment with DP bug taxonomy:** The RCA report's root cause classification is designed to map directly to the six DP project bug taxonomy custom fields: Root Cause, Resolution Type, Detected In, Affected Layer, Preventability, Recurrence Flag. Once the report is confirmed and posted, a human can update these fields from the RCA content.

---

## 14. Identity and security model

### Service principals

The agent uses two service principals, not the identity of any individual engineer.

**SP-B** (tenant `729ec3a8-…` — the Fabric/Azure tenant):
- Databricks: OAuth M2M, UC grants SELECT on `mnk_prod.gold.*` and `mnk_prod.pulse.*`
- Power BI / Fabric: Workspace Viewer on three workspaces, Fabric API access enabled
- ADF: Reader role on the ADF resource group
- Azure SQL: `db_datareader` on `mnkDnAIdb`

**SP-C** (Atlassian / Jira):
- Dedicated bot account with an API token
- Scoped to DP project only
- Read + comment permissions

### Key Vault secrets

All secrets are injected as environment variables at container startup. No secret is stored in any repo, container image layer, or config file.

| Secret | Used by | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Claude runtime on Foundry | Model API access |
| `DBX_CLIENT_ID` | Databricks SDK (SP-B) | OAuth M2M client ID |
| `DBX_CLIENT_SECRET` | Databricks SDK (SP-B) | OAuth M2M client secret |
| `AAD_TENANT_ID` | Fabric, ADF (SP-B) | Azure tenant ID |
| `PBI_CLIENT_ID` | Fabric XMLA + REST (SP-B) | AAD app client ID |
| `PBI_CLIENT_SECRET` | Fabric XMLA + REST (SP-B) | AAD app client secret |
| `JIRA_EMAIL` | Jira REST (SP-C bot) | Bot account email |
| `JIRA_API_TOKEN` | Jira REST (SP-C bot) | Bot account API token |
| `SQL_CONNECTION_STRING` | Azure SQL (SP-B) | Pulse serving DB connection |
| `ADO_DEPLOY_KEY` | Git repo checkout | Read-only PAT for 3 repos |

### Security principles

- Databricks access is enforced at the platform level (UC grants), not just by agent instruction. If a prompt-level rule is bypassed, the platform grant prevents any write.
- Power BI access is Workspace Viewer — structurally cannot publish, refresh, or edit.
- The Jira bot account is scoped to DP project only and cannot write without the human confirm step.
- All secrets rotate independently of the agent code. Key Vault rotation requires no agent redeployment.
- Every Databricks query, Fabric API call, and Jira write is logged with the originating ticket key to a central observability sink.

---

## 15. What the agent never does

Enforced by UC grants, SP role assignments, and the agent's `settings.json` permission allow-list — not just by instruction. Both layers must agree, and the platform layer is the one that actually holds.

- Does not write, commit, or push to any ADO repo
- Does not modify any Databricks table, schema, job, or cluster configuration
- Does not publish a Power BI model or trigger a dataset refresh
- Does not trigger an ADF pipeline run or modify any trigger or linked service
- Does not post a Jira comment without a human reviewing and confirming the RCA report first
- Does not use Playwright or any browser automation — no named-user data, no screenshot reproduction
- Does not store any credential in any repo, container image layer, or config file
- Does not run `INSERT`, `UPDATE`, `DELETE`, `DROP`, or `MERGE` on any database
- Does not transition Jira ticket statuses
- Does not access any system outside the six listed data plane connections

---

## 16. Front end options

The Foundry agent exposes an HTTP endpoint. Any of the following can serve as a front end:

### Teams bot (recommended starting point)
Foundry agents can publish directly into Microsoft Teams (GA June 2026). A Teams bot is the lowest-friction option — no new tool to install, the confirm gate is a button in the chat, and it reaches the right people immediately.

**Flow:** Jira webhook triggers investigation → agent runs → bot posts RCA report summary in a Teams channel → authorised person clicks Confirm → Jira comment is posted.

### Web app
A React or Next.js interface where someone can:
- Paste a Jira ticket key and trigger a manual investigation
- Watch the pipeline stages progress in real time
- Read the full RCA report before confirming
- Browse past investigations by ticket, root cause type, affected layer, or date
- See pattern analytics across cases (recurring root causes, MTTR trends)

### Jira embedded button (Forge app)
An "Investigate" button added to DP project ticket views via a Jira Forge app. One click triggers the agent; the RCA report comes back as a comment draft the assignee can review and post.

The human confirm step is **not optional in any front end**. The Jira comment write is not in the agent's allow-list — it is only exposed via the front end confirm action.

---

## 17. Feasibility on Microsoft Foundry

All capabilities this agent requires are generally available as of the date of this document.

| Capability | Status | Notes |
|---|---|---|
| Claude on Azure (Foundry) | GA — June 2026 | Messages API, prompt caching, tool streaming, Entra ID auth |
| Hosted agents (Foundry Agent Service) | GA — July 2026 | Linux container, managed endpoint, session state, observability |
| MCP connector for Claude on Azure | Live — August 2026 | Structured outputs, web search, web fetch, tool search |
| Anthropic Agent SDK | Supported | Explicitly listed as a supported framework for hosted agents |
| Azure Key Vault integration | Native | Foundry container apps support Key Vault secret injection |
| OpenTelemetry observability | GA | Every model call, tool invocation, and sub-agent hop traced |

**One known limitation:** Computer use and browser toolsets (`computer_toolset_20260801`, `browser_toolset_20260801`) are not available on Azure-hosted Claude deployments. This is not a blocker — the agent is designed to use log-based evidence only and Playwright is explicitly excluded by design.

**Claude models available on Azure (Hosted on Azure):** Claude Opus 4.8, Claude Haiku 4.5. For features requiring newer models, the "Hosted on Anthropic" option supports the full model catalogue but prompts and completions leave the Azure perimeter.

---

## 18. Future roadmap — RAG and graph database

The Phase 1–3 agent is a live-query agent: it hits Databricks, Fabric, and ADO in real time for every investigation. This is correct while building the case library from scratch. Once 3–6 months of confirmed RCA cases accumulate, the architecture can be extended significantly.

### Phase 4 — RAG on the case library

Before querying Databricks for a new investigation, the agent retrieves the three most semantically similar past cases from an Azure AI Search index. If the same root cause was already solved, the agent can say so in under a minute without touching Databricks at all.

**Implementation:** Azure AI Search index built from `.claude/rca-cases/`. Hybrid search (vector similarity + keyword). Foundry's built-in RAG pattern (`Foundry IQ` or a custom index) integrates natively with the hosted agent.

### Phase 4 — Graph database for lineage

Build a knowledge graph of the entire data platform — nodes for every gold table, silver table, bronze table, DAX measure, ETL notebook, ADF pipeline, and Power BI report, with edges representing the dependency relationships between them.

Instead of the agent traversing ADO repos and running SQL queries to trace lineage (which takes multiple round trips), it queries the graph once. A six-hop lineage trace that currently takes four Databricks queries becomes a single graph traversal.

**Technology:** Azure HorizonDB (Apache AGE) for the graph, `pgvector` for vector search alongside it. Graph-augmented RAG: vector search for semantic matches, then graph traversal for structurally important results that vector search misses.

**Build frequency:** The graph is rebuilt nightly from ADO repo metadata and Unity Catalog lineage APIs. Incremental updates on each commit.

### Phase 5 — GraphRAG

Microsoft's GraphRAG (March 2026 release, integrated with Azure AI Foundry) builds a knowledge graph from unstructured text. Applied here, it would index all RCA reports, Jira ticket histories, Confluence documentation, and ADO commit messages into a hierarchical knowledge graph.

**Benefit:** Multi-hop queries across the full history — "have we seen this pattern before in any related pipeline?" — that baseline RAG answers poorly because it cannot connect disparate facts through shared attributes.

**Cost note:** LazyGraphRAG reduces indexing cost to 0.1% of full GraphRAG while maintaining retrieval quality. At the scale of a single client's data platform, the indexing cost is not a concern.

---

## 19. Skill inventory and build plan

### Phase 1 — Auth and infrastructure (~2 weeks)
Prerequisite for everything. Nothing else can be built or validated until this is complete.

| Skill / asset | Status | What changes | Est. effort |
|---|---|---|---|
| Service principals (SP-B, SP-C) | New | Provision in AAD; configure Key Vault | 2 days |
| `settings.json` guardrails | Rework | Rewrite from scratch — current path globs are inert (see §6.1 of CR-Dev-Agent summary) | 1 day |
| `databricks-query` (`dbx_query.py`) | Rework | Switch from `auth_type="azure-cli"` to OAuth M2M env vars | 1 day |
| Jira REST path | Rework | Replace `mcp-remote` browser OAuth with bot API token | 1 day |
| ADO deploy key + repo checkout | New | Read-only PAT for 3 repos; wired at container startup via env vars | 1 day |

### Phase 2 — PBI and Pulse (~4 weeks)
Highest value. Most existing material. Validate RCA format against known closed DP tickets before go-live.

| Skill / asset | Status | What changes | Est. effort |
|---|---|---|---|
| `lineage-trace` | Carries over | Use as-is | — |
| `gold-lineage` | Carries over | Use as-is | — |
| `jira-intake` | Carries over | Ticket reading logic intact | — |
| `jira_attachment.py` | Carries over | Point to bot account env vars | 0.5 days |
| `pbi-troubleshoot` | Rework | Add Fabric REST + XMLA path; remove VS Code MCP calls | 3 days |
| `pbi-support` → PBI-Investigation | Rework | Restructure to five-stage pipeline + RCA report output | 5 days |
| Pulse-Investigation | New | Full pipeline + cross-layer trace including Azure SQL validation | 5 days |
| Intake router (2-plane) | New | Classify ticket to PBI or Pulse; dispatch persona | 2 days |
| RCA report writer | New | Produces the structured six-section RCA document | 3 days |
| Case library (`rca-cases/`) | Rework | Rename from `support-cases/`, extend schema, mount on persistent volume | 1 day |

### Phase 3 — Databricks and ADF (~3 weeks)
New planes. No existing analogue for either persona.

| Skill / asset | Status | What changes | Est. effort |
|---|---|---|---|
| DBX-Investigation | New | UC job run history, cluster logs, task failures, table snapshots | 5 days |
| ADF-Investigation | New | Pipeline run history, activity failures, trigger state via ADF REST | 4 days |
| `adf-run-reader` | New | Wrapper normalising ADF REST output into RCA evidence format | 2 days |
| Confidence scorer | New | Classifies root cause type; assigns confidence score | 2 days |
| Intake router (4-plane) | Rework | Extend Phase 2 router to include DBX and ADF planes | 1 day |

---

## 20. Skill carry-over from existing agents

### From Azure Fabric Report local agent

| Skill | Verdict | Notes |
|---|---|---|
| `lineage-trace` | Carries over | Read-only, use as-is |
| `pbi-troubleshoot` | Carries over (base) | Add XMLA REST path; remove MCP calls |
| `pbi-support` | Rework | Restructure output to five-stage + RCA format |
| `support-retro` | Carries over (logic) | Case filing pattern reusable; rename and extend schema |
| `update-docs` | Rework | Replace hardcoded Windows paths with env vars |
| `jira_attachment.py` | Carries over | Change credential source to bot account env vars |
| Case library pattern | Carries over | Rename `support-cases/` → `rca-cases/`; extend schema |
| Permission allow/deny structure | Rework | Rewrite globs — current paths match nothing |

### From CR-Dev-Agent

| Skill | Verdict | Notes |
|---|---|---|
| `gold-lineage` | Carries over | Read-only, use as-is |
| `jira-intake` / `jira-extract` | Carries over | Ticket reading logic intact |
| `databricks-query` (`dbx_query.py`) | Rework | Switch auth to M2M OAuth |
| `impact-analysis` | Rework | Scope to read-only evidence assembly; reformat output to RCA sections |
| `CR-Impact-Analyst` persona | Merge | Read-only analysis behaviour merges into DBX-Investigation |

### From Copilot Studio RCA Agent (Mankind Internal Core)

| Asset | Verdict | Notes |
|---|---|---|
| Jira JQL search pattern (`/rest/api/3/search/jql`) | Carries over | Already on REST + Basic Auth — reuse directly |
| Jira OAuth connector | Replace | Switch to bot API token; OAuth had deprecated endpoint issues |
| Auth lesson: `btoa("email:token")` for Jira | Keep | Validated pattern, document in runbook |

---

## 21. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Fabric SP API access disabled by tenant policy | Medium | Validate "Service principals can use Fabric APIs" in Phase 1 — blocks Phase 2 if unresolved |
| Databricks UC grants not tightened before go-live | Medium | Enforce SELECT only on `gold.*` and `pulse.*` before Phase 2; remove prompt-level constraint |
| Azure SQL access not provisioned for Pulse persona | Medium | Confirm SP-B `db_datareader` on `mnkDnAIdb` in Phase 1; Pulse-Investigation depends on it |
| Container restart loses `rca-cases/` history | Low | Persistent volume mounted; cases committed to git branch after each investigation |
| ADF REST scope blocked by subscription policy | Low | Validate SP-B Reader role on ADF resource in Phase 1 — well before Phase 3 |
| Human confirm step bypassed under time pressure | Medium | Gate enforced in code — Jira write not in agent allow-list; only exposed via front end confirm action |
| Jira bot lacks DP project permissions | Low | Create bot and confirm DP read + comment access in Phase 1 before Phase 2 starts |
| Four Fabric branches live-synced to production | Ongoing | ADO branch policies on `main`, `prod_main`, `prod2_main`, `HO_Production` must be enforced at server level — RCA agent has read-only access but this must be validated |
| `settings.json` path globs inert if not rewritten | High (if skipped) | Current globs point to `d:\Mankind-DataPlatform\` which matches nothing in a container — rewrite is Phase 1 item 2 |
| Concurrent investigations sharing one DBX profile | Low | Foundry session isolation handles concurrency; each run gets its own scratch dir under `rca-cases/` |

---

## 22. Success metrics

| Metric | Current baseline | Target |
|---|---|---|
| Mean time to root cause (MTTR) | 3–6 hours (manual) | < 30 minutes (agent draft + human confirm) |
| % DP P1/P2 tickets with structured RCA | ~10% (ad hoc) | > 90% after Phase 2 |
| Senior engineer time per investigation | 3–6 hours | < 30 minutes review time |
| Recurring root cause rate | Not tracked | < 15% recurrence of same root cause within 90 days |
| RCA false positive rate | N/A | < 10% (validated against Phase 2 smoke test on closed tickets) |
| Planes covered | 2 (PBI, Pulse) | 4 (PBI, Pulse, Databricks, ADF) after Phase 3 |

---

## 23. Next steps after approval

All eight items can start in parallel once leadership approves. Items 1–6 are Phase 1 prerequisites.

1. **Provision SP-B and SP-C in AAD.** Create two app registrations in the relevant tenants. Generate client credentials. Record object IDs and confirm each is in the correct tenant (`729ec3a8-…` for SP-B, Atlassian for SP-C).
2. **Configure Azure Key Vault.** Create Key Vault resource (if not existing). Add all secrets from the table in §14. Grant the Foundry container's managed identity read access to the vault.
3. **Create Jira bot account.** Create `agent-bot@celebaltech.com` (or equivalent). Generate API token. Confirm DP project permissions: Browse Projects, View Issues, Add Comments, Create Attachments.
4. **Enable Fabric API for service principals.** Go to Mankind Pharma AAD tenant settings → Power BI / Fabric admin portal → enable "Service principals can use Fabric APIs".
5. **Set Databricks UC grants.** Grant SP-B `USE CATALOG` + `USE SCHEMA` + `SELECT` on `mnk_prod.gold.*` and `mnk_prod.pulse.*`. Grant `CAN USE` on warehouse `efc9fed520454af6`.
6. **Assign Power BI workspace roles.** Add SP-B as Workspace Viewer on `MNK-Fabric-Prod-Workspace`, `MNK-Fabric-PROD-Workspace_2`, `HO-Production`. Grant SP-B `db_datareader` on `mnkDnAIdb` (Azure SQL).
7. **Kick off Phase 1 build.** Auth infrastructure, `settings.json` rewrite, path de-hardcoding, repo checkout wiring.
8. **Smoke test Phase 2 before go-live.** Run PBI-Investigation and Pulse-Investigation against a set of confirmed closed DP tickets. Compare agent verdicts to recorded outcomes. Target false positive rate < 10% before enabling the Jira comment write.

---

## 24. Glossary

| Term | Meaning in this context |
|---|---|
| AAD / Entra ID | Azure Active Directory — Microsoft's identity platform. Manages all Azure and Microsoft 365 identities. |
| ADO | Azure DevOps — Microsoft's git hosting and CI/CD platform. |
| ADF | Azure Data Factory — the ingestion pipeline layer that feeds Databricks. |
| Claude Agent SDK | Anthropic's SDK for building and deploying agents. Supported as a framework for Foundry hosted agents. |
| Databricks UC | Unity Catalog — the governance layer on top of Databricks that manages table-level access control. |
| DP project | The Jira project for Mankind Engagements (project key: DP). |
| Foundry | Microsoft Foundry — Azure's platform for building, hosting, and operating AI agents. |
| Foundry Agent Service | The Foundry component that runs hosted agents as containers with managed endpoints, scaling, and observability. |
| Gold layer | The curated, business-ready data layer in Databricks (`mnk_prod.gold.*`), which feeds Power BI and Pulse. |
| GraphRAG | Microsoft's graph-augmented RAG system. Combines knowledge graphs with vector search for multi-hop queries across complex private datasets. |
| Key Vault | Azure Key Vault — a managed secrets store. Secrets are injected into the container as env vars at startup. |
| M2M OAuth | Machine-to-machine OAuth — a credential flow where a service principal authenticates using a client ID and secret, with no browser or user interaction. |
| MCP | Model Context Protocol — a standard for connecting AI agents to external tools and services. |
| mnk_prod | The production Unity Catalog in Databricks. Contains `gold` and `pulse` schemas used by this agent. |
| Persona | A configured investigation mode within the agent, specialised for a specific data plane. |
| Pulse / Superman Portal | The internal reporting portal for Mankind Pharma field teams. Fed by the Databricks gold layer via ADF and Azure SQL. |
| RAG | Retrieval-Augmented Generation — grounding LLM responses in retrieved private data rather than relying on training data alone. |
| RCA | Root Cause Analysis — a structured investigation of why a problem occurred, as opposed to just fixing the symptom. |
| SP-B | Service Principal B — the agent's data-access identity for Fabric, Databricks, ADF, and Azure SQL. |
| SP-C | Service Principal C — the agent's Jira bot identity, scoped to the DP project only. |
| TMDL | Tabular Model Definition Language — the file format Power BI uses to define semantic models in git. |
| XMLA | XML for Analysis — the protocol used to query Power BI / Fabric semantic models programmatically. |

---

*Document maintained by Preyas Kulshreshtha, SDM — Celebal Technologies. Update this file as the agent evolves. Next planned update: after Phase 1 completion.*
