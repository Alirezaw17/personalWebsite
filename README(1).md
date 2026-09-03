# AI-Powered Personal Portfolio

A personal portfolio website with an embedded AI chat assistant that answers questions about my background using retrieval-augmented generation (RAG), can call live tools (GitHub, calendar) through an agent, and is built end-to-end on **free-tier services** — $0/month to run at portfolio-scale traffic.

---

## 1\. Objective

Give recruiters and hiring managers a way to interact with my background directly — not just read a static page — while the project itself serves as a live, working demonstration of the AI/LLM engineering skills currently in demand: RAG, agentic tool use, MCP, and LLM evaluation, built on a full-stack

+ AWS foundation.

---

## 2\. Features

- **Standard portfolio sections** — about, projects, experience, contact  
- **Grounded Q\&A (RAG)** — chat answers are retrieved from real CV/project content stored as embeddings, not hallucinated  
- **Live tool calls** — the assistant can fetch real GitHub activity or check calendar availability, exposed via MCP  
- **Agent routing** — a LangGraph orchestrator decides whether a question needs retrieval, a tool call, both, or neither  
- **Caching** — repeated questions are served from a Redis cache  
- **Evaluation/logging** — every conversation is logged for quality review

---

## 3\. Tech stack (all free tier)

| Layer | Choice | Why / free-tier notes |
| :---- | :---- | :---- |
| Frontend | React | Existing skill |
| Frontend hosting | **GitHub Pages** | 100% free, no AWS cost — deployed via GitHub Actions on push |
| Backend | Node/FastAPI | Runs inside Lambda |
| Backend hosting | API Gateway \+ Lambda | AWS Always Free tier (1M free requests/mo) |
| Database | PostgreSQL \+ `pgvector` | Local: Docker. Prod: Aurora Serverless (free-tier eligible) |
| Cache | Redis via **Upstash** | AWS ElastiCache is *not* free — Upstash has a genuine free tier |
| LLM | **Google Gemini API** (`google-genai`) | Free tier, no credit card, 1M-token context |
| Agent orchestration | LangGraph | Open source, no cost |
| Tool protocol | MCP (Model Context Protocol) | Open protocol, no cost |
| IaC / deployment | AWS CDK | Existing skill |
| Logging / eval | CloudWatch Logs | AWS free tier |
| Domain (optional) | Custom domain | \~$10/yr — only non-free item, skip and use a free `*.github.io` or `*.vercel.app` subdomain if $0 is a hard requirement |

**Backup LLM option:** Groq (`groq.com`) — free tier, no card, very fast, useful if Gemini rate limits are hit during development.

---

## 4\. Architecture

 React frontend (GitHub Pages)

        │

        ▼

 API Gateway \+ Lambda  ──────────────► Upstash Redis (cache)

        │

        ▼

 Agent orchestrator (LangGraph)  ────► CloudWatch (eval/logs)

        │

   ┌────┴─────┐

   ▼          ▼

 RAG        Tool calls via MCP

 (pgvector) (GitHub, calendar)

   │          │

   └────┬─────┘

        ▼

 Gemini API (LLM)

        │

        ▼

   Response back to user

---

## 5\. Project structure

/content     → CV, project write-ups, bio (markdown source for RAG)

/ingestion   → chunking \+ embedding scripts

/api         → backend (Lambda handlers, agent logic)

/frontend    → React app

/infra       → AWS CDK stack

---

## 6\. Local setup

### Prerequisites

- Node.js  
- Python 3.11+  
- Docker Desktop  
- AWS CLI \+ `npm install -g aws-cdk`

### Steps

1. **Clone and set up the repo**  
     
   git init  
     
   mkdir content ingestion api frontend infra  
     
2. **Get a free Gemini API key**  
     
   - Go to [aistudio.google.com](https://aistudio.google.com)  
   - Sign in → *Get API key* → *Create API key*  
   - Store it in `.env` (already in `.gitignore`):  
       
     GEMINI\_API\_KEY=your\_key\_here

     
3. **Run Postgres \+ pgvector locally**  
     
   docker run \--name pgvector-db \-e POSTGRES\_PASSWORD=devpass \\  
     
     \-p 5432:5432 \-d pgvector/pgvector:pg16  
     
   Then enable the extension once connected:  
     
   CREATE EXTENSION vector;  
     
4. **Set up Python environment**  
     
   python3 \-m venv venv  
     
   source venv/bin/activate  
     
   pip install google-genai psycopg2-binary python-dotenv langgraph  
     
5. **Write knowledge base content** Add real markdown files to `/content`: CV, 3–5 project write-ups, short bio.  
     
6. **Run the ingestion script** Chunks the markdown files, embeds each chunk via Gemini, and stores the chunk \+ vector in Postgres. Test with a sample query to confirm the right chunk comes back.  
     
7. **Set up Upstash Redis (free tier)**  
     
   - Sign up at [upstash.com](https://upstash.com)  
   - Create a Redis database → copy the REST URL/token into `.env`

---

## 7\. Deployment

### Frontend → GitHub Pages (free)

Pushed automatically via GitHub Actions on every push to `main`. Full workflow file and setup steps: see [**`docs/IMPLEMENTATION.md`](http://docs/IMPLEMENTATION.md)**.

### Backend → AWS (free tier)

cd infra

cdk bootstrap

cdk deploy

This provisions: API Gateway \+ Lambda (backend), Aurora Serverless Postgres (pgvector), and CloudWatch log groups — all within AWS Always Free limits at this traffic scale. The frontend on GitHub Pages calls this API over HTTPS (CORS configured to allow the `*.github.io` origin — details in the docs).

**Cost checklist to stay at $0:**

- ✅ Frontend on GitHub Pages — no AWS cost at all for hosting  
- ✅ No EC2, no NAT Gateway (common hidden-cost traps)  
- ✅ Aurora **Serverless**, not provisioned RDS  
- ✅ Redis on Upstash, not AWS ElastiCache  
- ⚠️ Gemini free tier has daily rate limits — fine for portfolio traffic  
- ⚠️ AWS's $100–200 signup credit lasts 6 months, but Always Free services don't depend on it

📄 **Full step-by-step build guide:** [`docs/IMPLEMENTATION.md`](http://docs/IMPLEMENTATION.md)

---

## 8\. Roadmap

- [ ] Write real CV/project content  
- [ ] Local RAG pipeline working (ingestion \+ retrieval test)  
- [ ] React chat widget wired to a basic `/api/chat` endpoint  
- [ ] GitHub \+ calendar tools implemented via MCP  
- [ ] LangGraph agent routing (RAG vs tools vs both)  
- [ ] Redis caching wired in  
- [ ] Conversation logging for evaluation  
- [ ] Deployed via CDK to AWS  
- [ ] Custom domain pointed at CloudFront (optional)

---

## 9\. Contact

- Email:  
- LinkedIn:  
- GitHub:

