# Conversational Assessment Recommender

A conversational AI agent that helps hiring managers and recruiters find the right SHL Individual Test Solutions through natural dialogue. Built as a take-home assignment for the **SHL Labs AI Intern role**.

---

## What It Does

Instead of keyword search, you describe the role in plain language — "I'm hiring a mid-level Java developer who needs to work with stakeholders" — and the agent asks focused clarifying questions, then returns a grounded shortlist of SHL assessments drawn exclusively from the official SHL product catalog.

**Four core behaviors:**
- **Clarify** — asks targeted questions when the query is too vague to act on
- **Recommend** — returns 1–10 assessments once it has enough context, with names and catalog URLs
- **Refine** — updates the shortlist mid-conversation when constraints change (e.g., "add personality tests")
- **Compare** — answers "what's the difference between OPQ32r and Verify G+?" using catalog data only

**Hard guardrails:**
- Never recommends anything outside the scraped SHL catalog
- Refuses general hiring advice, legal questions, and prompt-injection attempts
- Every URL returned is validated against the catalog before being sent

---

## Architecture

```
User
  │
  ▼
POST /chat (FastAPI)
  │  stateless — full conversation history every call
  ▼
Agent (conversation.py)
  ├── Retrieval: FAISS semantic search over 338 catalog embeddings
  ├── Prompt: system prompt + retrieved candidates injected into context
  └── LLM: Gemini 1.5 Flash (google-genai SDK)
        │
        ▼
  Validate response (URL guard, schema clamp)
        │
        ▼
  {"reply": ..., "recommendations": [...], "end_of_conversation": ...}
```

### Key design decisions

| Decision | Choice | Reason |
|---|---|---|
| LLM | Gemini 1.5 Flash | Free tier, fast (fits 30s timeout), supports JSON mode |
| Embeddings | `all-MiniLM-L6-v2` | Runs locally, no API cost, 384-dim fast inference |
| Vector store | FAISS (IndexFlatIP) | No server, embeds in process, cosine similarity after L2 normalization |
| Framework | FastAPI + Pydantic | Required by spec; Pydantic enforces schema at the boundary |
| Catalog | Static JSON (338 products) | Scraped once; prevents runtime hallucination of URLs |
| State | Stateless | Spec requirement; full history sent per call |

---

## Project Structure

```
shl-recommender/
├── catalog/
│   ├── scraper.py          # Web scraper for SHL catalog (run once)
│   └── catalog.json        # 338 Individual Test Solutions (ground truth)
├── retrieval/
│   ├── embedder.py         # Build FAISS index from catalog (run once)
│   ├── retriever.py        # Semantic search + URL validation
│   ├── faiss.index         # Built index (git-ignored, built at deploy)
│   └── metadata.pkl        # Catalog metadata aligned with index
├── agent/
│   ├── prompts.py          # System prompt builder + retrieval context formatter
│   └── conversation.py     # Core: history → retrieve → LLM → validate → response
├── api/
│   ├── main.py             # FastAPI app: GET /health, POST /chat
│   └── schemas.py          # Pydantic request/response models
├── tests/
│   └── test_agent.py       # 25 tests: schema, behavior probes, retriever, API
├── requirements.txt
├── render.yaml             # Render deployment config
└── .env.example
```

---

## API Reference

### `GET /health`

Returns `{"status": "ok"}` with HTTP 200. Used for readiness checks.

### `POST /chat`

**Request:**
```json
{
  "messages": [
    {"role": "user", "content": "I'm hiring a Java developer who works with stakeholders"},
    {"role": "assistant", "content": "What seniority level are you targeting?"},
    {"role": "user", "content": "Mid-level, around 4 years experience"}
  ]
}
```

**Response:**
```json
{
  "reply": "Got it. Here are 5 assessments for a mid-level Java dev with stakeholder focus.",
  "recommendations": [
    {"name": "Java 8 (New)", "url": "https://www.shl.com/products/product-catalog/view/java-8-new/", "test_type": "K"},
    {"name": "Occupational Personality Questionnaire OPQ32r", "url": "https://www.shl.com/products/product-catalog/view/occupational-personality-questionnaire-opq32r/", "test_type": "P"}
  ],
  "end_of_conversation": false
}
```

**Rules:**
- `recommendations` is `[]` while the agent is still gathering context
- `recommendations` has 1–10 items when the agent commits to a shortlist
- `end_of_conversation` is `true` only when the agent considers the task complete
- The schema is non-negotiable — it is validated by an automated evaluator

**Limits:**
- Max 8 turns per conversation (enforced by truncation)
- 30-second timeout per call

---

## Setup & Running Locally

### 1. Clone and install

```bash
git clone <repo-url>
cd shl-recommender
pip install -r requirements.txt
```

### 2. Set your Gemini API key

```bash
cp .env.example .env
# Edit .env and add your key:
# GEMINI_API_KEY=your_key_here
```

Get a free Gemini API key at [aistudio.google.com](https://aistudio.google.com).

### 3. Build the FAISS index

This embeds the 338 catalog items and saves the FAISS index locally. Run once.

```bash
python -m retrieval.embedder
```

Output:
```
Loaded 338 catalog items.
Loading embedding model: all-MiniLM-L6-v2
Encoding documents...
Index saved to retrieval/faiss.index (338 vectors, dim=384)
```

### 4. Start the server

```bash
uvicorn api.main:app --reload --port 8000
```

### 5. Test it

```bash
# Health check
curl http://localhost:8000/health

# Chat
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "I need to hire a Python data scientist"}
    ]
  }'
```

---

## Running Tests

```bash
python -m pytest tests/ -v
```

All 25 tests should pass. Test coverage includes:

| Category | Tests |
|---|---|
| Schema compliance | Response shape, field types, 10-item cap |
| Behavior probes | Vague query → clarify, off-topic refusal, prompt injection, mid-conversation refinement |
| Hallucination guard | URL validation — invalid URLs dropped before response |
| Retriever | FAISS loads, semantic search, name lookup, type filtering |
| Conversation traces | Job description → recommend, comparison query, end-of-conversation flag |
| API endpoints | Health check, schema validation, invalid role rejection |

---

## Retrieval Design

Each catalog item is embedded as a rich text document:

```
Java 8 (New) | Knowledge Skills technical knowledge test | https://...
```

Queries are built from the last 4 user turns (to capture refinements) and searched against the FAISS index using cosine similarity (L2-normalized inner product). Top-15 candidates are retrieved, then the LLM selects the best 1–10 with reasoning.

For comparison queries (detected via regex), both named assessments are retrieved directly and passed to the LLM as context.

---

## Prompt Design

The system prompt embeds:
1. All 338 catalog items (name, test types, remote/adaptive flags, URL)
2. Test type code definitions (A=Ability, K=Knowledge, P=Personality, etc.)
3. Explicit behavioral rules (when to clarify, when to recommend, how to refuse)
4. The exact JSON schema the model must output

Retrieved candidates are injected as `[CATALOG CONTEXT]` at the end of the last user message — grounding the model in relevant data before it reasons.

The model is instructed to output `application/json` via Gemini's `response_mime_type` parameter, which forces structured output and eliminates markdown-wrapping issues.

---

## Deployment (Render)

The `render.yaml` configures a free-tier Render web service:

```yaml
buildCommand: pip install -r requirements.txt && python -m retrieval.embedder
startCommand: uvicorn api.main:app --host 0.0.0.0 --port $PORT
```

The FAISS index is built at deploy time (not committed to git) so the binary stays out of the repository. Cold start is within the 2-minute health-check grace period.

Set `GEMINI_API_KEY` in Render's environment variables dashboard before deploying.

---

## Evaluation Criteria Mapping

| Criterion | How this submission addresses it |
|---|---|
| **Hard evals** — schema compliance | Pydantic enforces schema at API boundary; test suite validates every field |
| **Hard evals** — catalog-only URLs | `is_valid_url()` validates every URL before response; fallback name-lookup recovery |
| **Hard evals** — turn cap | Messages truncated to last 8 if exceeded |
| **Recall@10** | FAISS retrieves 15 candidates; LLM selects best 1–10; type-aware filtering available |
| **Behavior probes** — vague query | System prompt explicitly forbids first-turn recommendations without context |
| **Behavior probes** — off-topic refusal | System prompt enumerates forbidden topics; prompt injection patterns identified |
| **Behavior probes** — refinement | Full history passed each call; LLM instructed to update (not restart) shortlist |
| **Behavior probes** — hallucination | URL guard strips any non-catalog URL; name-recovery fallback for near-misses |

---

## What I Tried That Didn't Work

- **Embedding only the name** — retrieval quality was poor for domain terms like "personality" or "ability". Solved by prepending test type labels to each document.
- **`google-generativeai` SDK** — deprecated, replaced with `google-genai` v2 SDK which uses the `Client` pattern instead of `GenerativeModel`.
- **Calling `genai.configure()` at import time** — broke unit tests because the API key isn't available in CI. Solved by wrapping in `@lru_cache` lazy-init function.

---

## License

This project was built as a take-home assessment for SHL Labs. All assessment names and URLs are property of SHL and its affiliates.
