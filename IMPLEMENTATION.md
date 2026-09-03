# Implementation Guide

Full step-by-step flow for building and shipping the AI-powered portfolio, from an empty folder to a live site on GitHub Pages backed by a free-tier AWS API. Each phase builds on the previous one — don't skip ahead.

---

## Flow overview

Phase 0  Prerequisites

Phase 1  Knowledge base content

Phase 2  Local RAG pipeline (Postgres \+ pgvector \+ Gemini embeddings)

Phase 3  Backend API (chat endpoint)

Phase 4  React chat widget \+ static site

Phase 5  Deploy frontend → GitHub Pages

Phase 6  Deploy backend → AWS (CDK)

Phase 7  Connect frontend ↔ backend (CORS)

Phase 8  Tool use via MCP (GitHub, calendar)

Phase 9  Agent orchestration (LangGraph)

Phase 10 Caching (Upstash Redis) \+ evaluation logging

Phase 11 Polish \+ optional custom domain

---

## Phase 0 — Prerequisites

Install: Node.js, Python 3.11+, Docker Desktop, AWS CLI, `npm install -g aws-cdk`.

git init

mkdir content ingestion api frontend infra docs

Create a GitHub repo and push this initial structure — GitHub Pages needs a repo to deploy from later.

---

## Phase 1 — Knowledge base content

In `/content`, write real markdown files:

- `cv.md`  
- `project-1.md`, `project-2.md`, `project-3.md` (what you built, stack, outcome)  
- `bio.md`

This is the input to RAG — spend real time here before writing any code.

---

## Phase 2 — Local RAG pipeline

1. **Get a free Gemini API key** at [aistudio.google.com](https://aistudio.google.com) → *Get API key* → *Create API key*. Store in `.env`:  
     
   GEMINI\_API\_KEY=your\_key\_here  
     
2. **Run Postgres \+ pgvector locally:**  
     
   docker run \--name pgvector-db \-e POSTGRES\_PASSWORD=devpass \\  
     
     \-p 5432:5432 \-d pgvector/pgvector:pg16  
     
   CREATE EXTENSION vector;  
     
3. **Python environment:**  
     
   python3 \-m venv venv && source venv/bin/activate  
     
   pip install google-genai psycopg2-binary python-dotenv  
     
4. **`/ingestion/ingest.py`** — reads each markdown file, splits into chunks (\~300-500 words), embeds each chunk via Gemini, inserts `(chunk_text, embedding)` into a Postgres table.  
5. **Test retrieval** — run a sample query ("what has he built with AWS?"), embed it the same way, and confirm the nearest-neighbor search returns the right chunk. This proves the RAG foundation works before anything else is built.

---

## Phase 3 — Backend API

In `/api`, build one endpoint: `POST /chat`.

- Input: `{ "message": "..." }`  
- Logic: embed the message → similarity search in pgvector → build a prompt with the retrieved chunks → call Gemini → return the answer  
- This can run as a plain FastAPI/Express app locally first, before it's wrapped for Lambda in Phase 6

Test it locally with `curl` or Postman before touching the frontend.

---

## Phase 4 — Frontend

In `/frontend`, scaffold a React app (Vite recommended for GitHub Pages — smaller build, simpler config):

npm create vite@latest . \-- \--template react

Add a chat widget component that calls `POST /chat` on your local backend and renders the response. Reuse the layout from the homepage template (hero, projects, experience, contact \+ floating chat widget).

---

## Phase 5 — Deploy frontend → GitHub Pages

1. In `frontend/vite.config.js`, set the base path to your repo name:  
     
   export default { base: '/your-repo-name/' }  
     
2. Install the deploy helper:  
     
   npm install \--save-dev gh-pages  
     
3. Add to `frontend/package.json`:  
     
   "scripts": {  
     
     "deploy": "vite build && gh-pages \-d dist"  
     
   }  
     
4. **Recommended: automate it with GitHub Actions** instead of deploying manually. Create `.github/workflows/deploy.yml`:  
     
   name: Deploy frontend  
     
   on:  
     
     push:  
     
       branches: \[main\]  
     
       paths: \['frontend/\*\*'\]  
     
   jobs:  
     
     build-and-deploy:  
     
       runs-on: ubuntu-latest  
     
       steps:  
     
         \- uses: actions/checkout@v4  
     
         \- uses: actions/setup-node@v4  
     
           with: { node-version: 20 }  
     
         \- run: npm ci  
     
           working-directory: frontend  
     
         \- run: npm run build  
     
           working-directory: frontend  
     
         \- uses: peaceiris/actions-gh-pages@v3  
     
           with:  
     
             github\_token: ${{ secrets.GITHUB\_TOKEN }}  
     
             publish\_dir: frontend/dist  
     
5. In the GitHub repo settings → **Pages**, set the source to the `gh-pages` branch. Your site is now live at `https://<username>.github.io/<repo-name>/` — free, and it stays free indefinitely.

---

## Phase 6 — Deploy backend → AWS

In `/infra`, a CDK stack provisions:

- API Gateway \+ Lambda (wraps the FastAPI/Express app from Phase 3\)  
- Aurora Serverless Postgres (pgvector) — replaces the local Docker DB

cd infra

cdk bootstrap

cdk deploy

Note the output API URL (e.g. `https://xxxx.execute-api.eu-west-1.amazonaws.com`) — the frontend will call this directly.

---

## Phase 7 — Connect frontend ↔ backend (CORS)

Since the frontend (`*.github.io`) and backend (AWS API Gateway) are on different domains, CORS must be explicitly enabled:

- In the API Gateway/Lambda response headers, allow the GitHub Pages origin:  
    
  Access-Control-Allow-Origin: https://\<username\>.github.io  
    
- In the React app, point the chat widget's fetch call at the deployed API URL instead of `localhost` (use an environment variable so local dev still works against `localhost`).

---

## Phase 8 — Tool use via MCP

Implement two tools as small Lambda functions, exposed via MCP:

- **GitHub tool** — calls the GitHub API for your latest repos/commits  
- **Calendar tool** — checks availability via a calendar API

---

## Phase 9 — Agent orchestration

Wrap the RAG call and the MCP tools in a **LangGraph** agent so the backend automatically decides, per question, whether to retrieve, call a tool, both, or neither — instead of hardcoding "if question contains X."

---

## Phase 10 — Caching \+ evaluation

- **Upstash Redis** (free tier) — cache repeated questions before they hit Gemini again. Sign up at [upstash.com](https://upstash.com), create a database, and store the REST URL/token in `.env`.  
- **CloudWatch Logs** — log every `{question, retrieved_chunks, answer}` triple so answer quality can be reviewed and improved over time.

---

## Phase 11 — Polish (optional)

- Custom domain (\~$10/yr) pointed at GitHub Pages instead of the default `*.github.io` URL — GitHub Pages supports custom domains for free, you only pay for the domain registration itself.  
- Write the project README/write-up featuring this as your top project.

---

## Cost summary for this flow

| Piece | Cost |
| :---- | :---- |
| GitHub Pages (frontend hosting) | Free, indefinitely |
| GitHub Actions (CI/CD) | Free for public repos |
| AWS Lambda \+ API Gateway | Free tier (1M requests/mo) |
| Aurora Serverless Postgres | Free-tier eligible |
| Upstash Redis | Free tier |
| Gemini API | Free tier (rate-limited, sufficient for portfolio traffic) |
| Custom domain | \~$10/yr — the only non-free item, and optional |

