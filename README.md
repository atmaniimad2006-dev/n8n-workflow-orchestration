<h1 align="center">⚡ N8N Workflow Orchestration</h1>

<p align="center">
  <i>Infrastructure as Code (IaC) — Multi-Agent Data Flow Orchestration, Conditional Routing & Rate Limiting</i>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow_Engine-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white" alt="n8n">
  <img src="https://img.shields.io/badge/OpenAI-GPT--4o-412991?style=for-the-badge&logo=openai&logoColor=white" alt="OpenAI">
  <img src="https://img.shields.io/badge/Pinecone-Vector_DB-000000?style=for-the-badge&logo=pinecone&logoColor=white" alt="Pinecone">
  <img src="https://img.shields.io/badge/Redis-Rate_Limiting-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis">
  <img src="https://img.shields.io/badge/Supabase-State_Mgmt-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white" alt="Supabase">
</p>

---

## 📋 Overview

This repository contains **production-grade n8n workflow definitions** exported as JSON (Infrastructure as Code). Each workflow implements a complete data pipeline with conditional routing, state management, and error handling patterns — demonstrating backend orchestration skills applicable to **multi-agent AI supervision**.

> **Note:** All credentials, API keys, and personal identifiers have been sanitized. Replace `YOUR_*` placeholders with your own credentials to deploy.

---

## 🏗️ Architecture & Workflows

### Workflow 1 — AI Support Agent (`support-ai-agent.json`)

**Purpose:** Event-driven WhatsApp support bot with conditional routing, rate limiting, and LLM-powered response generation.

```
Webhook (POST) → Message Validation (If) → Trigger Route (Switch)
                                               ├── Route 1: Audio Response Pipeline
                                               └── Route 2: Rate Limiter → AI Agent → Response Dispatch
```

**Key Architectural Patterns:**

| Pattern | Implementation |
|---|---|
| **Conditional Routing** | `Switch` node with regex-based intent matching on incoming message payloads |
| **Rate Limiting (State Management)** | Supabase table tracking per-user message counts; conditional gate at 15 messages triggers human handoff |
| **LLM Orchestration** | `@n8n/n8n-nodes-langchain.agent` with GPT-4o-mini, structured system prompt with 4 intent categories |
| **Session Memory** | `Window Buffer Memory` (10-message context window) keyed by `remoteJid` for per-user conversation state |
| **Media Guard** | Pre-routing filter rejecting non-text messages before they reach the AI pipeline |
| **Prompt Engineering** | XML-structured system prompt with `<SYSTEM_INSTRUCTIONS>`, `<TONE_AND_LANGUAGE>`, `<KNOWLEDGE_BASE>` sections and anti-injection guardrails |

---

### Workflow 2 — WhatsApp Community Manager (`whatsapp-manager-system.json`)

**Purpose:** Multi-pipeline workflow combining a **task tracking system** (image proof validation + Google Drive archival) with a **RAG-powered AI community support agent**.

#### Pipeline A — Task Tracker (Proof Validation)
```
Webhook → Group/Image Filter → Google Sheets Lookup → Base64 Extraction (JS)
→ Caption Routing (Fajr|Quran|DeepWork) → Validation Gate → Drive Upload
→ Sheets Upsert (appendOrUpdate with UID dedup)
```

#### Pipeline B — RAG AI Support Agent
```
Webhook → Word Count + Text Extract (JS) → Multi-Condition Filter
→ Redis INCR (Rate Limit) → Rate Limit Gate (≤5/hour) → AI Agent (GPT-4.1-mini)
→ "IGNORE" Kill Switch (If) → WhatsApp Reply (with quoted message)
```

#### Pipeline C — Knowledge Ingestion (RAG)
```
Google Drive Trigger (file add) → Download → PDF Extract → Text Cleanup (JS)
→ Smart Chunking (1500 chars) → OpenAI Q&A Extraction (Structured JSON)
→ JSON Parsing (JS) → Pinecone Vector Store Insert (with OpenAI Embeddings)
```

**Key Architectural Patterns:**

| Pattern | Implementation |
|---|---|
| **Rate Limiting** | Redis `INCR` with hourly keys (`rate_limit:{chat_id}:{yyyy-MM-dd-HH}`), gate at ≤5 req/hour. On AI "IGNORE" response, counter is decremented (Redis GET → SET) to avoid penalizing no-ops |
| **Anti-Hallucination Guardrail** | System prompt enforces `IGNORE` keyword when no relevant context found; post-LLM `If` node filters it before sending |
| **RAG Pipeline** | PDF → Clean → Chunk → LLM Q&A extraction → Pinecone vector insert. Retrieval via `vectorStorePinecone` in `retrieve-as-tool` mode |
| **Timezone-Aware Processing** | JavaScript node using `Intl.DateTimeFormat` with per-user timezone from Google Sheets |
| **Idempotent Upserts** | Google Sheets `appendOrUpdate` with composite UID key (`date + phone`) preventing duplicate entries |
| **Error Resilience** | `alwaysOutputData: true` on lookup nodes, `try/catch` in JavaScript nodes with structured `DEBUG_ERROR` output |

---

### Workflow 3 — Invoice Automation (`invoice-automation.json`)

**Purpose:** Event-driven invoice generation pipeline triggered by Google Sheets row additions.

```
Google Sheets Trigger → Status Filter ("Send") → Template Copy (Google Drive)
→ Field Computation (JS) → Template Injection (Google Docs replaceAll)
→ PDF Export → Drive Upload → Sheets Update (invoice link writeback)
```

**Key Architectural Patterns:**

| Pattern | Implementation |
|---|---|
| **Template-as-Code** | Google Docs template with `{{placeholder}}` tokens, programmatically replaced via `replaceAll` operations |
| **Computed Fields** | `Edit Fields` node calculating `Total_sans_livraison` and `Total` from unit price × quantity + shipping |
| **Status-Driven Routing** | `Filter` node gating pipeline execution on `Status === "Send"` |
| **Writeback Loop** | Final node updates the source Google Sheet with the generated invoice link |

---

### Workflow 4 — OCR MVP (`ocr-mvp.json`)

**Purpose:** Intelligent invoice parsing pipeline using OCR + GPT-4o Vision for automated accounting.

```
Webhook (POST /new-invoice) → File Download → Format Router (PDF vs Image)
    ├── PDF: ConvertAPI (PDF→PNG) → Download Converted → GPT-4o Vision
    └── Image: Direct → GPT-4o Vision
→ JSON Parse (JS) → Google Sheets Upsert (Accounting Ledger)
```

**Key Architectural Patterns:**

| Pattern | Implementation |
|---|---|
| **Format-Aware Routing** | `If` node checking `file_path` for `.pdf` extension, branching to conversion pipeline or direct analysis |
| **Structured LLM Output** | GPT-4o Vision with expert-level extraction prompt (Moroccan accounting standards, VAT verification, Plan Comptable categorization) |
| **Mathematical Verification** | Prompt instructs the LLM to self-verify `HT + TVA === TTC` and flag discrepancies |
| **JSON Hardening** | JavaScript node strips markdown fencing and parses raw JSON output from LLM |

---

## 🚀 Deployment

1. Import any `.json` file into your n8n instance via **Settings → Import Workflow**
2. Replace all `YOUR_*` credential placeholders with your own API keys
3. Configure webhook URLs in your messaging provider (Evolution API / Whapi)
4. Activate the workflow

---

## 📁 Repository Structure

```
n8n-workflow-orchestration/
├── workflows/
│   ├── support-ai-agent.json          # AI Support Bot + Rate Limiting
│   ├── whatsapp-manager-system.json   # RAG Agent + Task Tracker
│   ├── invoice-automation.json        # Template-based Invoice Generator
│   └── ocr-mvp.json                   # OCR + GPT-4o Vision Pipeline
└── README.md
```

---

## 🛠️ Tech Stack

- **Orchestration Engine:** n8n (self-hosted)
- **LLMs:** OpenAI GPT-4o, GPT-4o-mini, GPT-4.1-mini
- **Vector Database:** Pinecone (RAG retrieval)
- **State Management:** Redis (rate limiting), Supabase (user state), Google Sheets (data persistence)
- **File Processing:** ConvertAPI (PDF→PNG), Google Drive API
- **Messaging:** WhatsApp via Evolution API & Whapi
