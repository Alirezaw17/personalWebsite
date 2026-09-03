# InsightRAG — Document Q&A Engine

A Retrieval-Augmented Generation (RAG) system that answers natural-language
questions about your own documents, grounded in their actual content instead
of an LLM's memorized training data. Built from scratch (no LangChain/LlamaIndex)
so every stage — chunking, embedding, vector search, prompt construction,
generation — is custom code, not a framework black box.

## Tech stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| Chunking | Custom recursive/overlapping text splitter |
| Embeddings | `sentence-transformers` (`all-MiniLM-L6-v2`), local, free, 384-dim |
| Vector search | FAISS (`IndexFlatIP`, cosine similarity) |
| LLM generation | Google Gemini (`gemini-3.6-flash`) via the `google-genai` SDK |
| Web UI | Streamlit |
| Config | `python-dotenv` (`.env` for API keys) |

## How it works

```
Your .txt files
      |
      v
 [chunking.py]  --> splits text into ~800-char overlapping chunks
      |
      v
[embeddings.py] --> turns each chunk into a 384-dim vector (sentence-transformers, local, free)
      |
      v
[vector_store.py] --> stores vectors in a FAISS index for fast similarity search
      |
      v
   (index/ saved to disk)

At question time:
Your question --> embed it --> search FAISS for closest chunks --> stuff them
into a prompt --> Gemini generates an answer grounded in those chunks,
citing sources
```

## Project structure

```
RagProject/
├── data/                     # .txt documents to index (sample: company_policy.txt)
├── index/                    # generated FAISS index (created by ingest.py)
├── src/
│   ├── chunking.py           # splits documents into overlapping chunks
│   ├── embeddings.py         # local embedding model (sentence-transformers)
│   ├── vector_store.py       # FAISS-based similarity search
│   └── rag_pipeline.py       # retrieval + prompt construction + Gemini call
├── ingest.py                 # CLI: build the index
├── query.py                  # CLI: ask a question
├── app.py                    # Streamlit web UI
├── .env                      # GEMINI_API=your-key-here (not committed)
└── requirements.txt
```

## Setup

1. **Install Python 3.9+** if you don't have it.
2. **Create and activate a virtual environment:**
   ```bash
   python -m venv venv
   source venv/bin/activate      # on Windows: venv\Scripts\activate
   ```
3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```
4. **Get a Gemini API key** from https://aistudio.google.com/apikey and add it
   to a `.env` file in the project root:
   ```
   GEMINI_API=your-key-here
   ```
   (Loaded automatically via `python-dotenv` — no need to export it manually.)

## Usage

1. **Add your documents.** Drop `.txt` files into the `data/` folder. A sample
   file (`company_policy.txt`) is already included so you can try it immediately.

2. **Build the index** (run this once, and again whenever you add/change documents):
   ```bash
   python ingest.py
   ```

3. **Ask questions**, either from the terminal:
   ```bash
   python query.py "How many vacation days do employees get?"
   ```
   or with the web UI:
   ```bash
   streamlit run app.py
   ```
   then open http://localhost:8501 in your browser.

## How to test that it's working

1. `python ingest.py` — should print `Indexed company_policy.txt: N chunks`
   and finish with `Done. Index saved to index/`.
2. `python query.py "How many vacation days do employees get?"` — should
   print an `ANSWER:` block referencing 15 (or 20, after 3 years) vacation
   days, plus a `SOURCES USED:` list showing `company_policy.txt` excerpts
   with relevance scores.
3. `streamlit run app.py` — open the browser tab it launches, type a
   question, click **Ask**, and confirm an answer appears with an
   expandable **"Sources used"** section citing the document.

If step 2 or 3 fails with a `GEMINI_API` error, double-check your `.env` file
has the key set correctly and that you're running commands with the `venv`
activated.

## Key concepts

- **Chunking**: documents are split into ~800-character overlapping pieces
  because embedding models produce one vector per input — too much text in
  one chunk blurs its meaning, too little loses context. Overlap prevents
  ideas from being cut in half at chunk boundaries.
- **Embeddings**: each chunk is converted into a 384-dimensional vector using
  `all-MiniLM-L6-v2`, a local open-source model. Semantically similar text
  ends up close together in vector space, even without shared keywords.
- **Vector search**: FAISS (`IndexFlatIP`) finds the chunks whose vectors are
  closest to the question's vector, using cosine similarity.
- **Augmented generation**: the top matching chunks are inserted into the
  prompt sent to Gemini, so the answer is grounded in your actual documents
  rather than the model's memorized training data — this reduces hallucination
  and lets you cite sources.
