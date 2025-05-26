# 📚 Session 16 · LLM-Ops in Practice

> *Cache everything, observe everything, ship with confidence.*

## ✈️ Quick Start

```bash
# 1. clone & enter the repo
git clone <your-fork>
cd 16_LLMOps

# 2. create Python 3.10 venv & install deps
python -m venv .venv && source .venv/bin/activate          # Windows: .venv\Scripts\activate
uv pip install -r requirements.lock                        # or: pip install -r requirements.txt

# 3. export your HF token + *two* endpoints
export HF_TOKEN=hf_************************************
export HF_EMBED_EP=https://<…>.aws.endpoints.huggingface.cloud       # embeddings
export HF_LLM_EP=https://<…>.aws.endpoints.huggingface.cloud         # text-gen

# 4. run the notebook
jupyter lab Prototyping_LangChain_Application_with_Production_Minded_Changes_Assignment.ipynb
```

## 🗂️ Repository Layout

| Path | What lives here |
|------|----------------|
| `Prototyping_…ipynb` | the main notebook with Tasks 1-3 |
| `cache/` | on-disk cache for embeddings |
| `cache_vs_no_cache.png` | screenshot used in Activity #3 |
| `requirements.lock` | fully resolved dependency list (via uv) |

## 🏗️ Task-by-Task Recap

### Task 1 · Cache-Backed Embeddings

#### Details
- **Why**: Vectorising the same text twice is wasted latency & $$$
- **How**: CacheBackedEmbeddings wraps a real HuggingFaceEndpointEmbeddings model, using a hashed namespace in a LocalFileStore.
- **Activity #1**: Measured first vs second call.
  - Cold ≈ 110 ms → cache hit ≈ 80 ms.
- **Question #1**: Where is prompt-level caching best suited? → Unit tests / evaluation loops where identical prompts are common.
- **Gotchas & Fixes**: Accidentally pointed at the LLM endpoint → 413 error. Fixed by separating EMBED vs LLM env vars.

### Task 2 · Cache-Backed LLM Calls

#### Details
- **Why**: Hot prompts (health-checks, evaluations) shouldn't hit external endpoints.
- **How**: Replaced classroom FakeListLLM with real Llama-3 HF endpoint and manually wrote each response into the LangChain prompt cache.
- **Activity #2**: First call ≈ 8 s (model spin-up) → cache hit ≈ 0 s.
- **Question #2**: Caching works only for exact prompt + params; any change (temp, system msg) is a cache miss.
- **Issues**: HF SDK switched return schema.
  - Solution: Force details=True and normalise to plain text before caching.

### Task 3 · Production-Minded RAG

#### Details
- **Chunks & Store**: 221 PDF chunks embedded once → vectors stored in an in-memory Qdrant.
- **LCEL Pipeline**: {"context": question | retriever, "question": question} → chat_prompt → hf_llm. Context & question branches run in parallel by default.
- **Compressor**: Added ContextualCompressionRetriever to stay under HF 512-token limit (previous 413 errors).
- **LangSmith Trace**: 
  - Left = cache (≈ 9 s), right = no-cache (≈ 45 s).
- **Activity #3**: Recorded two traces—one warm, one cold—and attached screenshot above.
- **Latency delta**: ~5× speed-up (23.47 s vs 43.64 s in our run).

## ❓ Discussion Answers in Full

### Best suited for cache-backed embeddings/LLM?
Repeatable evaluation, unit tests, demo apps with a fixed prompt set.

### Least suited?
High-traffic, user-generated chat where every prompt is unique; multi-replica deployments.

### Why did LangSmith still show an HF call on cached run?
The first call populates the cache, so one HF request is inevitable. Subsequent identical calls are served locally and don't appear as children in the trace.
