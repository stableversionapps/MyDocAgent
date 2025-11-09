Got it. Below is a **detailed, strict, step-by-step, no-assumption technical specification** formatted to instruct a literal Copilot-style model that needs **maximum explicit context**.
This spec targets an **MVP that runs optimally on your MacBook (Intel i9, 16GB RAM)**, with future switches to larger models configurable.

---

# ✅ **SMART DOC AGENT — MVP TECHNICAL SPEC (LINE-BY-LINE)**

**Project Goal:** Build a local RAG (Retrieval Augmented Generation) agent that reads Markdown files, answers questions using them as knowledge, explains answers, and exposes an API — optimized for a low-resource laptop, with pluggable model upgrades.

---

## 1️⃣ **PROJECT SETUP**

1. Create project folder: `smart-doc-agent`
2. Initialize Git repo inside it: `git init`
3. Add `.gitignore` with Python, cache, and model artifacts ignored
4. Use Python 3.10 or 3.11 only (avoid 3.12 for package stability)
5. Create virtual environment: `python3 -m venv venv`
6. Activate: `source venv/bin/activate`
7. Create `requirements.txt` (packages listed later)

---

## 2️⃣ **FOLDER STRUCTURE**

```
smart-doc-agent/
│── app/
│   ├── main.py                # FastAPI entrypoint
│   ├── config.py              # Config for models, paths, params
│   ├── ingestion.py           # Markdown loader, chunking, embeddings
│   ├── vector_store.py        # FAISS or Chroma index handler
│   ├── llm.py                 # Tiny LLM wrapper + swappable config
│   ├── prompts.py             # Prompt templates
│   ├── api_routes.py          # Query endpoints
│   ├── utils.py               # Helpers
│── data/
│   ├── docs/                  # All markdown files go here
│── db/
│   ├── vectors/               # Local vector DB index storage
│── models/                    # Quantized LLMs download folder
│── configs/
│   ├── default.yml            # Config (switch model size here)
│── notebooks/                 # Optional experiments
│── requirements.txt
│── README.md
```

---

## 3️⃣ **CORE REQUIREMENTS**

Add to `requirements.txt`:

```
fastapi
uvicorn
langchain
langchain-community
sentence-transformers
markdown
faiss-cpu
torch
transformers
accelerate
pydantic
python-dotenv
PyYAML
```

Install: `pip install -r requirements.txt`

---

## 4️⃣ **CONFIGURATION FILE (`configs/default.yml`)**

```yaml
model:
  provider: "local"                  # local or api
  local_model_path: "models/tiny-llm.gguf"
  model_type: "tiny"                 # tiny, small, large (future switch)
  temperature: 0.2

embedding:
  model: "sentence-transformers/all-MiniLM-L6-v2"

retrieval:
  chunk_size: 512
  chunk_overlap: 50
  top_k: 5

paths:
  docs: "data/docs/"
  vector_db: "db/vectors/"
```

---

## 5️⃣ **INGESTION PIPELINE (Markdown Loader)**

Implementation rules:

1. Scan all `.md` files recursively in `data/docs/`
2. Extract content while preserving:

   * File name
   * Headings (#, ##, ###)
   * Paragraph structure
3. Chunk text using:

   * `chunk_size = 512 tokens`
   * `chunk_overlap = 50 tokens`
4. Call embedding model to generate vectors
5. Store vectors + metadata:

   ```
   {
    "content": "...",
    "source": "filename.md",
    "heading": "if exists",
    "chunk_index": 3
   }
   ```
6. Save index into `db/vectors/` using FAISS (CPU only)

---

## 6️⃣ **EMBEDDING MODEL (MVP Smallest Stable)**

Use:

```
sentence-transformers/all-MiniLM-L6-v2
```

Constraints:

* Must run CPU
* Embedding dimension: 384
* No GPU dependency
* Load only once and reuse in memory

---

## 7️⃣ **TINY LLM for Local Answering**

MVP model requirements:

| Requirement       | Value                          |
| ----------------- | ------------------------------ |
| Size              | max 1.5B params or 4-bit quant |
| File type         | `.gguf` quantized              |
| Must run CPU      | ✅ Yes                          |
| Fast inference    | ✅ Yes                          |
| Reasoning quality | Good enough for doc QA         |

Recommended model to download into `/models`:

```
TinyLlama-1.1B-Chat-GGUF (Q4_K_M)
```

Load it using `llama-cpp-python`

Future switch rule:

* If config says `model_type: small`, load 3B model
* If `large`, load 7B/13B
* If `provider: api`, switch to OpenAI/Claude endpoint

---

## 8️⃣ **VECTOR RETRIEVAL**

At query time:

1. Convert user question to embedding vector
2. Run FAISS similarity search
3. Return top K results (default 5)
4. Sort by similarity score descending
5. Format retrieved chunks into prompt context block

---

## 9️⃣ **PROMPT RULES (VERY IMPORTANT)**

The system prompt must enforce:

```
You are a documentation assistant.
You must answer ONLY using the provided context.
If the answer is not found, say: "I don’t have enough context in the docs."
You must explain your answer clearly.
You must list source file names you used.
Do NOT hallucinate.
Format response as:

### Answer:
<your explanation>

### Sources:
- filename.md (section if available)
```

---

## 🔟 API SPECIFICATION (FASTAPI)

Expose endpoints:

| Method | Route     | Purpose                                 |
| ------ | --------- | --------------------------------------- |
| POST   | `/ingest` | Read markdown files, build vector index |
| POST   | `/ask`    | Ask question, return answer + sources   |
| GET    | `/config` | View config                             |
| POST   | `/config` | Update config without redeploy          |

---

## 1️⃣1️⃣ `/ask` Request & Response Contract

**Request:**

```json
{
  "query": "How does authentication work in this project?"
}
```

**Response:**

```json
{
  "answer": "Here is the explanation...",
  "sources": ["auth.md", "security.md"],
  "confidence_score": 0.87
}
```

---

## 1️⃣2️⃣ Resource Constraints (MUST FOLLOW)

| Component            | Limit             |
| -------------------- | ----------------- |
| RAM usage            | < 10GB            |
| Model size           | < 5GB             |
| Must work offline    | ✅ Yes             |
| GPU required         | ❌ No              |
| Response time target | < 5 sec per query |

---

## 1️⃣3️⃣ Future-proofing (Switchable via config)

You will later be able to switch:

* `tiny` → `small` → `large` models
* Local → OpenAI/Anthropic API
* FAISS → Chroma → Qdrant vector DB
* CPU → GPU if available
* Markdown → PDF/Text/Docs ingestion

---

## 1️⃣4️⃣ Logging Requirements

Log:

* Ingested file names
* Chunk count per file
* Index build duration
* Query retrieval scores
* Model latency per answer

---

## 1️⃣5️⃣ Success Criteria (MVP Acceptance)

✅ Reads all markdown files
✅ Builds vector index
✅ Answers only from docs (no hallucination)
✅ Returns sources
✅ Runs on 16GB laptop smoothly
✅ Configurable model switching
✅ Exposed API works via Postman/cURL

---

### ✅ Next question for you:

Do you want me to now:

1. **Generate 100% working code for this spec**, or
2. **Generate code + Docker setup**, or
3. **Generate code + UI Chat dashboard**, or
4. **Package everything into ZIP ready to run**?

Reply with just a number: **1, 2, 3, or 4** 🚀
