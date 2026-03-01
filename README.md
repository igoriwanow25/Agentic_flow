# Agentic Flow

This repository contains an autonomous, self-hosted AI agent ecosystem using `n8n`, `Ollama`, `Mem0`, `Qdrant`, and `PostgreSQL`. It orchestrates a team of multiple specialized AI agents directly on a local machine, integrating a full Memory layer and a Notion knowledge base via RAG.

## Project Structure
- `self-hosted-ai-starter-kit`: The core n8n environment running via Docker Compose. Contains the `n8n` orchestrator, `PostgreSQL` database, `Qdrant` vector store, `Ollama` for local LLMs, and `Mem0` for layered long-term memory.
- `mem0-repo`: A local clone/build of the `mem0` server to support custom architecture builds (e.g., ensuring `linux/amd64` compatibility on Windows hosts).
- `gemini.md`: The architectural blueprint and build instructions for the agent ecosystem.

## Prerequisites
- Docker & Docker Compose
- `ngrok` (for Telegram Webhook testing)
- OpenRouter API Key

## Setup & Run
1. Navigate to the starter kit directory:
   ```bash
   cd self-hosted-ai-starter-kit
   ```
2. Copy the example `.env` file and fill in your secrets (specifically `OPENROUTER_API_KEY` and `POSTGRES_PASSWORD`):
   ```bash
   cp .env.example .env
   ```
3. Start the environment:
   ```bash
   docker compose --profile cpu up -d
   ```
4. Access the `n8n` dashboard at `http://localhost:5679` (Note: Port 5679 is used to prevent conflicts with standard `5678` ports).

### Telegram Webhooks
If integrating with Telegram bots locally, use `ngrok` to tunnel `n8n` to the internet:
```bash
ngrok http 5679
```
Update your `.env` file with the generated `WEBHOOK_URL` and restart the `n8n` container.

## Architecture
- **Main Assistant:** Built with `Claude 3.5 Sonnet` via OpenRouter. Manages tasks and context.
- **Memory Layer:** Uses `Mem0` coupled with `Qdrant` and `n8n`'s Buffer Window memory to maintain conversational state across sessions (identified by Telegram Chat ID).