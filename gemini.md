# AI Agent Team — Build Instructions
## Stack: n8n + OpenRouter + Notion + Qdrant + Mem0 + Telegram

---

## 🎯 What We're Building

A team of 4 autonomous AI agents running 24/7 on a home computer:

| Agent | Role | Model (OpenRouter) |
|-------|------|--------------------|
| Main Assistant | Manages projects, delegates tasks, knows full context | `anthropic/claude-3.5-sonnet` |
| Auditor | Reviews quality of all other agents' outputs | `google/gemini-2.0-flash-exp` |
| SysOps | Monitors infrastructure, reacts to failures overnight | `deepseek/deepseek-chat` |
| Company Agent (Plej) | Answers questions about services using Notion knowledge base | `openai/gpt-4o-mini` |

Each agent has:
- Its own Telegram bot (control from anywhere — couch, tram, bed at 11pm)
- Access to Notion as a knowledge base
- Layered long-term memory (Mem0 + Qdrant)
- Separate permissions and tools

---

## 🖥️ Hardware Requirements

- Any PC — no GPU needed, models run via OpenRouter API
- Min. 8 GB RAM, 50 GB free disk space
- Stable internet connection
- Docker Desktop (Windows/Mac) or Docker Engine (Linux)

---

## 📦 Step 1 — Install the Base Stack

### 1.1 Clone n8n Self-Hosted AI Starter Kit

```bash
git clone https://github.com/n8n-io/self-hosted-ai-starter-kit.git
cd self-hosted-ai-starter-kit
cp .env.example .env
```

### 1.2 Edit .env

```env
POSTGRES_PASSWORD=your_postgres_password
N8N_ENCRYPTION_KEY=generate_random_32_char_string
GENERIC_TIMEZONE=Europe/Warsaw
```

### 1.3 Start (CPU profile — no GPU required)

```bash
docker compose --profile cpu up -d
```

Services available after startup:
- **n8n dashboard**: http://localhost:5678
- **Qdrant dashboard**: http://localhost:6333
- **PostgreSQL**: localhost:5432

---

## 🔑 Step 2 — Configure OpenRouter in n8n

### 2.1 Create credential

1. In n8n go to **Settings → Credentials → Add Credential**
2. Select **OpenRouter Chat Model** (n8n ≥ 1.77.0) OR **OpenAI Chat Model**
3. Paste your **API Key from openrouter.ai**
4. If using OpenAI node: set **Base URL** to `https://openrouter.ai/api/v1`

### 2.2 Note

Ignore any "cannot connect to OpenAI" warning — it's a false alarm.
The credential works correctly with OpenRouter.

### 2.3 Model names to use per agent

```
anthropic/claude-3.5-sonnet       # Main Assistant
google/gemini-2.0-flash-exp       # Auditor
deepseek/deepseek-chat            # SysOps
openai/gpt-4o-mini                # Company Agent
```

Full model list: https://openrouter.ai/models

### 2.4 Cost optimization strategy

Assign cheaper models to simpler tasks:
- DeepSeek (~$0.001/1K tokens) for SysOps log analysis
- Gemini Flash (free tier available) for Auditor
- Reserve Claude/GPT-4o only for the Main Assistant
- Monitor usage in real time at: https://openrouter.ai/activity

---

## 📓 Step 3 — Connect Notion as Knowledge Base

### 3.1 Create a Notion Integration

1. Go to: https://www.notion.com/my-integrations
2. Click **+ New Integration**
3. Name it e.g. `n8n-ai-agent`
4. Copy the **Internal Integration Token**

### 3.2 Share Notion databases with the integration

In each Notion database you want agents to access:
- Click **...** (page menu) → **Add connections** → select `n8n-ai-agent`

### 3.3 Add credential in n8n

1. Settings → Credentials → **Notion API**
2. Paste the Internal Integration Token

### 3.4 Import the ready-made template

Download: https://n8n.io/workflows/2413-notion-knowledge-base-ai-assistant/

This gives you out of the box:
- Notion node → fetches pages from the database
- Text Splitter → chunks content
- AI Agent with Notion search tool

---

## 🧠 Step 4 — Memory Layer (Mem0 + Qdrant)

### 4.1 Add Mem0 to docker-compose.yml

```yaml
mem0:
  image: mem0ai/mem0-server
  ports:
    - "8888:8888"
  environment:
    - OPENAI_API_KEY=${OPENROUTER_API_KEY}
    - OPENAI_BASE_URL=https://openrouter.ai/api/v1
  depends_on:
    - postgres
    - qdrant
```

### 4.2 Memory architecture (4 layers)

| Layer | Description | Tool |
|-------|-------------|------|
| L0 — Working | Current session / context window | n8n built-in memory (session_id) |
| L1 — Consolidation | Real-time extraction of key facts | Mem0 auto-extraction |
| L2 — Long-term | Condensed decisions, patterns, history | Mem0 long-term + pgvector |
| Retrieval | Hybrid semantic + full-text search | Qdrant (vector) + PostgreSQL (FTS) |

### 4.3 Using memory in n8n workflows

In the AI Agent node add **Memory → Window Buffer Memory** (short session memory).
For long-term memory, add HTTP Request to Mem0 API as an agent tool:

```
POST http://mem0:8888/memories          # Save memory
GET  http://mem0:8888/memories/search?query=...&user_id=agent_name  # Recall
```

---

## 🔍 Step 5 — Index Notion → Qdrant (RAG)

For large knowledge bases, create a separate indexing workflow:

```
Schedule Trigger (every 6h)
  → Notion: Get Many Pages
  → Loop Over Items
    → Notion: Get Page Content (HTTP Request to blocks API)
    → Text Splitter (Recursive, chunk=500, overlap=50)
    → Embeddings (OpenRouter: text-embedding-3-small)
    → Qdrant Vector Store Insert
```

In the agent workflow, add a **Vector Store Search** tool (Qdrant):

```
User Query → Embed Query → Qdrant Search → Top-K chunks → Agent Context
```

To keep the knowledge base up to date, add a **Notion Trigger** that fires
on any page change and re-indexes only the modified pages.

---

## 📱 Step 6 — Telegram Bots (one per agent)

### 6.1 Create bots via BotFather

Open Telegram → @BotFather → `/newbot` (repeat 4 times, one per agent)

Save tokens:
```
TELEGRAM_TOKEN_ASSISTANT=...
TELEGRAM_TOKEN_AUDITOR=...
TELEGRAM_TOKEN_SYSOPS=...
TELEGRAM_TOKEN_PLEJ=...
```

### 6.2 Workflow schema in n8n (per agent)

```
Telegram Trigger
  → AI Agent
      ├── Model: OpenRouter (specific per agent)
      ├── Memory: Window Buffer Memory (session_id = chat_id)
      ├── Tools:
      │     ├── Notion Search (Main Assistant + Company Agent)
      │     ├── Qdrant Vector Search (all agents)
      │     ├── Mem0 Memory Read/Write (all agents)
      │     └── HTTP Request (SysOps — check Docker/services status)
      └── System Prompt: [see below]
  → Telegram: Send Message
```

### 6.3 System Prompts per agent

**Main Assistant:**
```
You are Igor's personal assistant. You know his projects, preferences and decision history.
You manage tasks and delegate to other agents when needed.
You have access to the Notion knowledge base. You remember all previous conversations.
Be specific, actionable and concise.
```

**Auditor:**
```
You are a quality auditor. You review and verify the work of other agents.
You have zero conflict of interest — your only goal is accuracy and quality.
Point out errors, inconsistencies and improvement suggestions. Be critical but constructive.
```

**SysOps:**
```
You are a DevOps engineer. You monitor infrastructure and respond to issues.
You have access to check Docker service statuses.
Alert proactively when something is wrong. Provide specific remediation commands.
```

**Company Agent (Plej):**
```
You are Plej's company assistant. You answer questions about services, projects and approach.
Base your answers EXCLUSIVELY on the Notion knowledge base. Do not make things up.
If information is not in the knowledge base, say so directly.
Tone: professional, friendly, concise.
```

### 6.4 Local n8n on localhost — HTTPS tunneling via ngrok

```bash
# Install ngrok and set authtoken
ngrok config add-authtoken YOUR_TOKEN
# Open HTTPS tunnel to n8n
ngrok http 5678
# Copy the URL (e.g. https://abc123.ngrok.io) and set it as the webhook in Telegram Trigger
```

---

## 💰 Monthly Costs

| Item | Cost |
|------|------|
| n8n, Qdrant, Mem0, PostgreSQL | €0 (open source) |
| OpenRouter — LLM tokens | ~€20–30 |
| Electricity (no GPU) | ~€3–5 |
| **Total** | **~€25–35/month** |

---

## 🗺️ Build Roadmap

### Week 1 — Foundation
- [ ] Docker Compose up (n8n + Qdrant + PostgreSQL)
- [ ] OpenRouter credential in n8n
- [ ] First Telegram bot → Main Assistant (no Notion, no memory yet)
- [ ] Verify: send message → receive response

### Week 2 — Notion + RAG
- [ ] Notion Integration Token + n8n credential
- [ ] Import Notion Knowledge Base AI Assistant template
- [ ] Build Notion → Qdrant indexing workflow
- [ ] Add Qdrant Search as tool to Main Assistant

### Week 3 — Memory
- [ ] Add Mem0 to Docker Compose
- [ ] Integrate Mem0 as tool (read/write) for all agents
- [ ] Test: agent recalls decisions from the previous week

### Week 4 — Remaining Agents
- [ ] Auditor (bot + workflow + system prompt)
- [ ] SysOps (bot + workflow + Docker monitoring tools)
- [ ] Company Agent Plej (bot + Plej Notion knowledge base)
- [ ] End-to-end integration tests

---

## 🔗 Key Resources

- n8n Self-Hosted AI Starter Kit: https://github.com/n8n-io/self-hosted-ai-starter-kit
- Notion Knowledge Base template: https://n8n.io/workflows/2413-notion-knowledge-base-ai-assistant/
- OpenRouter models: https://openrouter.ai/models
- OpenRouter activity dashboard: https://openrouter.ai/activity
- Mem0 OSS docs: https://docs.mem0.ai/open-source/overview
- Qdrant docs: https://qdrant.tech/documentation/
- n8n RAG docs: https://docs.n8n.io/advanced-ai/rag-in-n8n/
- ngrok: https://ngrok.com/download
