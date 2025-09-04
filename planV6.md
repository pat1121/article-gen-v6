# **Content Spark — Full Build Plan (Everything your code-gen AI needs)**

This is a complete, opinionated specification that an expert engineer/agent can implement module-by-module without prior context. It includes a detailed executive summary, high-level architecture, precise data contracts, database DDL, adapter interfaces, orchestration rules, UI specs, linking/RAG math, budget controls, and acceptance tests. All constraints below are **frozen** unless explicitly stated as profile-overrideable.

---

## **1\) Executive summary**

**Goal.** Content Spark turns raw ideas into published, internally-linked WordPress posts at scale, with human-in-the-loop tuning and deterministic, reversible batch operations. It favors **local models** and **repeatability** while providing **paid-API fallbacks** under a **sliding 24-hour token budget**.

**Two-pass strategy.**

* **Pass 1 (Generation)**: Runs **one idea at a time** through Article → Keywords/Index → SEO. **Images are queued** to ComfyUI (with fallbacks) and **do not block** Pass 1\.

* **Pass 2 (Publish & Link)**: Runs with **N workers** (default 2; UI-adjustable). Ensures categories/tags, resolves slug collisions deterministically, injects internal links with **fairness** and **per-article caps**, and schedules/publishes to WordPress.

**Linking policy (hard).**

* Max **3 links/article**, **≤1/section**, **no links** in headings.

* **Target fairness**: no target may receive inbound links from \> **5%** of total published posts.

* **Bootstrap**: skip link insertion until there are **≥50** published posts.

**Budgets & fallbacks.**

* **GLOBAL paid-API sliding 24h cap**: 1,000,000 tokens across profiles (local models excluded). If a paid call would exceed the cap, **auto-switch to local** and emit `paid_cap_reached`.

* **Images**: ComfyUI first, **3-minute timeout** per image, then **Gemini**, then **DALL·E**.

**Vector system & retrieval.** Qdrant (dense) \+ Tantivy/BM25 (sparse) via a single adapter; ingest with BGE-M3; retrieve via **hybrid \+ rerank \+ MMR**; these power JIT internal linking & related content.

**Human control.** Workbench (single-item tabs), Command Center (CSV \+ batches \+ **Undo**), Attention Required dashboard.

**Determinism & idempotency.** `idea_hash`, `article_hash`, `slug_locked`, and link identity `(source_id, target_id, anchor_text)` ensure stable re-runs.

---

## **2\) Architecture & layout**

### **2.1 Components**

* **UI**: Streamlit multipage app: Settings/Profiles, Command Center, Workbench tabs (Article, Keywords/Index, SEO, Images), Attention dashboard.

* **Orchestrator** (Python/asyncio): sequencing, concurrency, retries, budgets, events, batch lifecycle.

* **Adapters**: TextModelAdapter (OpenAI/Gemini/DeepSeek/Ollama), ImageModelAdapter (ComfyUI→Gemini→DALL·E), VectorStoreAdapter (Qdrant+Tantivy; Chroma optional), WordPressAdapter, ComfyUIAdapter.

* **Data**: SQLite primary DB; Qdrant \+ Tantivy for retrieval; local filesystem for media.

### **2.2 Directory layout**

Canonical project tree with orchestrator modules, adapters, DB schema/migrations, tests (goldens) and utilities.

---

## **3\) Global constraints (frozen)**

* **Inbound link fairness**: 5%

* **Image timeout**: 180 s

* **GLOBAL paid-API sliding 24h cap**: 1,000,000 tokens

* **Pass-2 workers default**: 2

* **CSV columns**: `seed_idea` (required), `context` (opt), `tone` (opt)

Note: We enforce the budget **globally** (not per profile), per your decision.

---

## **4\) Database schema (DDL)**

All tables that are touched by a batch now carry a `batch_run_id` for deterministic **Undo**. We added **dual offset domains** for internal links (plain text & HTML), **chunked vector IDs**, **mutated HTML** persistence \+ **history**, **retry & error fields**, **prompt\_tests FKs**, **image usage** ledger, and expanded **token usage**.

Run with `PRAGMA foreign_keys=ON;` and version these as migrations.

`-- ---- Core entities --------------------------------------------------------`

`CREATE TABLE IF NOT EXISTS profiles (`  
  `id INTEGER PRIMARY KEY,`  
  `name TEXT NOT NULL`  
`);`

`CREATE TABLE IF NOT EXISTS generated_ideas (`  
  `id INTEGER PRIMARY KEY,`  
  `profile_id INTEGER NOT NULL,`  
  `seed_idea TEXT NOT NULL,`  
  `context TEXT,`  
  `tone TEXT,`  
  `idea_hash TEXT NOT NULL,                -- deterministic per profile`  
  `status TEXT NOT NULL DEFAULT 'idea',    -- idea | content_generated | attention_required`  
  `created_at INTEGER NOT NULL,`  
  `updated_at INTEGER,`  
  `batch_run_id INTEGER,                   -- NEW: for Undo`  
  `FOREIGN KEY(profile_id) REFERENCES profiles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS generated_articles (`  
  `id INTEGER PRIMARY KEY,`  
  `profile_id INTEGER NOT NULL,`  
  `idea_id INTEGER NOT NULL,`  
  `title TEXT NOT NULL,`  
  `article_body TEXT NOT NULL,             -- raw/generated text (canonical)`  
  `article_body_html_final TEXT,           -- NEW: mutated HTML after linking`  
  `mutated_at INTEGER,                     -- NEW`  
  `mutated_by_batch_id INTEGER,            -- NEW`  
  `slug TEXT NOT NULL,`  
  `slug_locked INTEGER NOT NULL DEFAULT 0,`  
  `model_used TEXT,`  
  `prompt_version TEXT,`  
  `article_hash TEXT NOT NULL,`  
  `created_at INTEGER NOT NULL,`  
  `updated_at INTEGER,`  
  `batch_run_id INTEGER,                   -- NEW`  
  `FOREIGN KEY(profile_id) REFERENCES profiles(id),`  
  `FOREIGN KEY(idea_id) REFERENCES generated_ideas(id)`  
`);`

`CREATE UNIQUE INDEX IF NOT EXISTS ux_article_hash ON generated_articles(article_hash);`

`CREATE TABLE IF NOT EXISTS article_html_history ( -- NEW: for Undo of HTML mutations`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER NOT NULL,`  
  `batch_run_id INTEGER NOT NULL,`  
  `prev_html TEXT,`  
  `changed_at INTEGER NOT NULL,`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS article_keywords (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER NOT NULL,`  
  `kind TEXT NOT NULL,                     -- primary | secondary | entity | question`  
  `value TEXT NOT NULL,`  
  `importance INTEGER DEFAULT 0,`  
  `batch_run_id INTEGER,                   -- NEW`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS article_seo (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER NOT NULL,`  
  `meta_title TEXT,`  
  `meta_description TEXT,`  
  `canonical_url TEXT,`  
  `schema_json TEXT,`  
  `batch_run_id INTEGER,                   -- NEW`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS article_images (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER NOT NULL,`  
  `role TEXT NOT NULL,                     -- hero | inline`  
  `status TEXT NOT NULL,                   -- pending | succeeded | failed`  
  `model_used TEXT,`  
  `file_path TEXT,                         -- keep name consistent`  
  `created_at INTEGER NOT NULL,`  
  `batch_run_id INTEGER,                   -- NEW`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS comfy_jobs (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER NOT NULL,`  
  `role TEXT NOT NULL,`  
  `job_id TEXT NOT NULL,`  
  `status TEXT NOT NULL,                   -- pending | succeeded | failed`  
  `started_at INTEGER NOT NULL,`  
  `updated_at INTEGER,`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`-- ---- WordPress publishing -------------------------------------------------`

`CREATE TABLE IF NOT EXISTS wordpress_posts (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER NOT NULL,`  
  `wp_post_id INTEGER,`  
  `slug TEXT,`  
  `link TEXT,`  
  `status TEXT NOT NULL,                   -- draft | future | publish`  
  `scheduled_gmt TEXT,`  
  `published_at INTEGER,`  
  `batch_run_id INTEGER,                   -- NEW`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`-- ---- Internal links + reservations (fairness under concurrency) ----------`

`CREATE TABLE IF NOT EXISTS internal_links (`  
  `id INTEGER PRIMARY KEY,`  
  `source_article_id INTEGER NOT NULL,`  
  `target_article_id INTEGER NOT NULL,`  
  `anchor_text TEXT NOT NULL,`  
  `-- dual offset domains (NEW)`  
  `text_start INTEGER,`  
  `text_end   INTEGER,`  
  `html_start INTEGER,`  
  `html_end   INTEGER,`  
  `batch_run_id INTEGER,                   -- NEW`  
  `UNIQUE (source_article_id, target_article_id, anchor_text),`  
  `FOREIGN KEY(source_article_id) REFERENCES generated_articles(id),`  
  `FOREIGN KEY(target_article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE INDEX IF NOT EXISTS idx_internal_links_target ON internal_links (target_article_id);`  
`CREATE INDEX IF NOT EXISTS idx_internal_links_batch  ON internal_links (batch_run_id);`

`CREATE TABLE IF NOT EXISTS link_reservations (       -- NEW: fairness reservations`  
  `id INTEGER PRIMARY KEY,`  
  `batch_run_id INTEGER NOT NULL,`  
  `source_article_id INTEGER NOT NULL,`  
  `target_article_id INTEGER NOT NULL,`  
  `reserved_at INTEGER NOT NULL DEFAULT (strftime('%s','now'))`  
`);`

`-- ---- Vector indexing & cache ---------------------------------------------`

`CREATE TABLE IF NOT EXISTS embeddings_cache (`  
  `text_sha1 TEXT PRIMARY KEY,`  
  `dim INT NOT NULL,`  
  `model TEXT NOT NULL,`  
  `vec BLOB NOT NULL`  
`);`

`CREATE TABLE IF NOT EXISTS vector_chunks (           -- NEW: BM25 text + metadata mirror`  
  `id TEXT PRIMARY KEY,                               -- "<article_id:chunk_n>"`  
  `article_id INTEGER NOT NULL,`  
  `chunk_no INTEGER NOT NULL,`  
  `char_start INTEGER,`  
  `char_end INTEGER,`  
  `text TEXT NOT NULL,`  
  `last_ingested_at INTEGER`  
`);`  
`CREATE INDEX IF NOT EXISTS idx_vc_article ON vector_chunks (article_id, chunk_no);`

`-- ---- Budgets & usage ledgers ---------------------------------------------`

`CREATE TABLE IF NOT EXISTS token_usage (`  
  `id INTEGER PRIMARY KEY,`  
  `provider TEXT NOT NULL,      -- openai | google | deepseek | etc. | local providers excluded from cap`  
  `model TEXT,`  
  `operation TEXT,              -- generate | rerank | embed | ...`  
  `entity_type TEXT,            -- article | link-plan | ...`  
  `entity_id INTEGER,`  
  `tokens_in INTEGER DEFAULT 0,`  
  `tokens_out INTEGER DEFAULT 0,`  
  `occurred_at INTEGER NOT NULL,`  
  `batch_run_id INTEGER`  
`);`  
`CREATE INDEX IF NOT EXISTS idx_token_usage_window ON token_usage (provider, occurred_at);`  
`CREATE INDEX IF NOT EXISTS idx_token_usage_batch  ON token_usage (batch_run_id);`

`CREATE TABLE IF NOT EXISTS image_usage (            -- NEW: separate ledger for images`  
  `id INTEGER PRIMARY KEY,`  
  `provider TEXT NOT NULL,       -- comfyui | gemini | dalle`  
  `model TEXT,`  
  `operation TEXT NOT NULL,      -- txt2img | img2img | upscale`  
  `images INTEGER NOT NULL,`  
  `cost_cents INTEGER,           -- optional: for paid providers`  
  `occurred_at INTEGER NOT NULL,`  
  `batch_run_id INTEGER`  
`);`  
`CREATE INDEX IF NOT EXISTS idx_image_usage_window ON image_usage (provider, occurred_at);`  
`CREATE INDEX IF NOT EXISTS idx_image_usage_batch  ON image_usage (batch_run_id);`

`-- ---- Batches, retries, attention -----------------------------------------`

`CREATE TABLE IF NOT EXISTS batch_runs (`  
  `id INTEGER PRIMARY KEY,`  
  `run_type TEXT NOT NULL,       -- pass1 | pass2`  
  `profile_id INTEGER,           -- helpful for history/filters`  
  `started_at INTEGER NOT NULL,`  
  `finished_at INTEGER`  
`);`

`CREATE TABLE IF NOT EXISTS batch_items (`  
  `id INTEGER PRIMARY KEY,`  
  `batch_run_id INTEGER NOT NULL,`  
  `entity_type TEXT NOT NULL,    -- idea | article | image | post | link`  
  `entity_id INTEGER,`  
  `step TEXT NOT NULL,           -- article | keywords | seo | images | publish | link-plan | ...`  
  `status TEXT NOT NULL,         -- running | ok | failed`  
  `retry_count INTEGER DEFAULT 0, -- NEW`  
  `error_code TEXT,               -- NEW`  
  `error_message TEXT,            -- NEW`  
  `started_at INTEGER NOT NULL,`  
  `finished_at INTEGER,`  
  `FOREIGN KEY(batch_run_id) REFERENCES batch_runs(id)`  
`);`

`-- ---- Templates & prompt tests --------------------------------------------`

`CREATE TABLE IF NOT EXISTS templates (`  
  `id INTEGER PRIMARY KEY,`  
  `name TEXT NOT NULL,`  
  `version TEXT NOT NULL,`  
  `body TEXT NOT NULL`  
`);`

`CREATE TABLE IF NOT EXISTS prompt_tests (`  
  `id INTEGER PRIMARY KEY,`  
  `template_id INTEGER NOT NULL,`  
  `article_id INTEGER,            -- optional: the article written with this variant`  
  `variant_label TEXT NOT NULL,   -- A | B | ...`  
  `prompt TEXT NOT NULL,`  
  `created_at INTEGER NOT NULL,`  
  `FOREIGN KEY (template_id) REFERENCES templates(id) ON DELETE CASCADE,`  
  `FOREIGN KEY (article_id) REFERENCES generated_articles(id) ON DELETE SET NULL`  
`);`

`-- ---- Settings (global and per profile) ------------------------------------`

`CREATE TABLE IF NOT EXISTS app_settings (`  
  `key TEXT PRIMARY KEY,`  
  `value TEXT NOT NULL`  
`);`

`CREATE TABLE IF NOT EXISTS profile_settings (`  
  `id INTEGER PRIMARY KEY,`  
  `profile_id INTEGER NOT NULL,`  
  `key TEXT NOT NULL,`  
  `value TEXT NOT NULL,`  
  `UNIQUE(profile_id, key),`  
  `FOREIGN KEY(profile_id) REFERENCES profiles(id)`  
`);`

---

## **5\) Adapter contracts (interfaces)**

### **WordPress**

`class PostCreate(BaseModel):`  
    `title: str`  
    `slug: str`  
    `html: str                                # supply article_body_html_final if present`  
    `status: Literal["draft","future","publish"] = "draft"`  
    `categories: list[int] = []`  
    `tags: list[int] = []`  
    `canonical_url: str | None = None`  
    `meta_title: str | None = None`  
    `meta_description: str | None = None`  
    `schema_json: str | None = None`  
    `date_gmt: str | None = None             # scheduling`  
    `featured_media: int | None = None`

`class PostResult(BaseModel):`  
    `id: int`  
    `slug: str`  
    `link: str`

`class WordPressAdapter(Protocol):`  
    `async def test_connection(self) -> bool: ...`  
    `async def ensure_category(self, name: str) -> Term: ...`  
    `async def ensure_tag(self, name: str) -> Term: ...`  
    `async def upload_media(self, path: str, title: str) -> int: ...`  
    `async def create_or_update_post(self, data: PostCreate) -> PostResult: ...`

Use WP REST API with Application Passwords; expose SEO plugin meta via `meta` if needed.

**Publish source of truth:** Always send **`article_body_html_final`** if not NULL, else fall back to `article_body`. (Ensures idempotent linking.)

### **Image model adapters**

Primary: **ComfyUI**, then **Gemini**, then **DALL·E**; background job monitor enforces **180 s** timeout and triggers fallback chain. Include filesystem storage util shown in module doc.

### **Vector store adapter**

Qdrant \+ Tantivy/BM25 behind a single interface; hybrid search \+ score blending with rerank and MMR. Provide `VectorDoc(id, text, meta, vector?)` and `VectorQuery(text, top_k_dense=200, top_k_sparse=200, filters?)`.

**Doc ID policy (LOCKED):** `VectorDoc.id = "<article_id:chunk_n>"` **for all ingest/query** (chunked IDs). (Also mirrored in `vector_chunks.id`.)

---

## **6\) Vector/RAG details (ingest & query)**

### **6.1 Ingestion pipeline (chunked IDs)**

1. Normalize text; 2\) 5-gram near-dup; 3\) **Chunk 450–650 tokens** (60–100 overlap) with `char_start/char_end`; 4\) **Embed BGE-M3 locally** with SQLite cache; 5\) Upsert to **Qdrant** (`chunks_main`, cosine) and store full‐text for BM25.

### **6.2 Query pipeline (for linking)**

Hybrid dense+BM25 (top\_k 200 each) → blend `0.65*dense + 0.35*sparse` → **bge-reranker-base (int8/onnx)** keep 40 → **MMR** λ=0.3, k=12, ≤2–3 chunks per doc, stitch adjacent → eligibility filters (exclude self/same-slug; exclude future-scheduled targets; enforce 5% inbound fairness) → candidates with anchor phrases (3–7 words) \+ insertion offsets.

---

## **7\) Internal linking engine (insertion rules)**

* **Caps**: 3 links/article, ≤1/section; **no headings**.

* **Anchor selection**: prefer natural 3–7 word spans; **map offsets in BOTH domains**: `(text_start,text_end)` **and** `(html_start,html_end)`.

* **Idempotency**: unique `(source_id, target_id, anchor_text)`.

* **Persistence & mutation**: write to `internal_links`, then inject `<a>` at stored **HTML offsets** using an HTML AST; persist final HTML to `generated_articles.article_body_html_final`.

---

## **8\) Orchestrator (policies & flows)**

### **8.1 Pass 1 (sequential)**

For each idea: **Article** → **Keywords/Index** (vector upsert with chunked IDs) → **SEO** (don’t touch slug if locked) → **Images** (enqueue; non-blocking). Success → `content_generated`; failures retry ≤3 then `attention_required` with `error_code`. Emit step events.

### **8.2 ComfyUI monitor (background)**

Poll 5–10 s (capped backoff); if elapsed \> 180 s, mark failed and trigger Gemini→DALL·E fallback; on completion write `article_images(status, model_used, file_path)`; update comfy job status.

### **8.3 Pass 2 (concurrent workers; default 2\)**

Schedule/publish; ensure terms; resolve slug collisions (`-2, -3, …`) then **lock slug**; execute JIT **link plan** (skip if published \< 50), enforce caps \+ fairness; apply HTML mutation; upload featured image; create post; persist returned `wp_post_id`, `slug`, `link`, `status`, `published_at/scheduled`. Concurrency: throttle only the affected provider when rate-limited.

### **8.4 Budgets (sliding 24h; GLOBAL cap)**

`budgets.can_spend(provider, tokens_needed)` sums paid `token_usage` in last 24h and compares to the **global** cap. If breach → **switch to local** & log `paid_cap_reached`. Exponential backoff for the rate-limited provider only.

---

## **9\) Fairness under concurrency (5% inbound)**

* Use **`link_reservations`** table to reserve a target before committing a link.

* At reservation time (transaction): compute `(current_inbound + current_reservations)/total_published < 0.05`; only then insert reservation.

* On finalize: write `internal_links` with both offset domains; delete reservation atomically.

* Skip all linking until published\_count ≥ 50\.

---

## **10\) Budgets & usage ledgers**

* **Token usage**: record **provider**, **model**, **operation**, **entity\_type/id**, **tokens\_in/out**, **occurred\_at**, **batch\_run\_id**.

* **Budget enforcement**: global sliding 24h cap (1,000,000 tokens). Paid providers only; local excluded. If `can_spend` is false → **fallback to local** and emit an event; do not stall other steps.

* **Image usage**: separate ledger (**`image_usage`**) with **provider**, **operation**, **images**, optional **cost\_cents**, **occurred\_at**, **batch\_run\_id** (since images are billed per-image/time, not tokens).

Per your decision: **no token buckets** beyond exponential backoff.

---

## **11\) UI specs**

### **11.1 Module 0 — Global Settings & Profiles**

Effective value display (Global vs Profile override vs Resolved), Test Connection buttons, profile picker; saves to `app_settings` / `profile_settings`.

### **11.2 Module 1 — Command Center (+ CSV)**

CSV import, Dry-Run (token/time/cost), batch controls with live events, **Undo** for a batch (local DB only).

### **11.3 Workbench (Modules 3–6; single-item)**

Tabs: Article Forge, Keyword Catalyst, SEO Optimizer, Image Artisan. Template **A/B**, “**Record as Golden**”.

### **11.4 Attention Required**

Group by `error_code`, row actions (Re-run local, lower temp, open in Workbench), bulk re-runs allowed.

---

## **12\) Module-by-module build detail (I/O, policies, errors, tests)**

The following mirrors your module breakdown, with the **agreed changes applied** (chunked vector IDs, retry fields, etc.).

### **Module 0 — Global Settings & Profiles**

Purpose: settings & effective values; Input: `SaveSettingRequest`; Output: writes to settings tables; Policies: single-user secrets visible; Test Connection emits events; Errors: adapter exceptions toast+event; Tests: precedence & connection logs.

### **Module 1 — Command Center (+ CSV)**

CSV headers `seed_idea`, `context`, `tone`; compute `idea_hash = sha1(lower(trim(seed_idea)) + '|' + profile_id)`; insert unique ideas; Batch API: start/stop/progress/dry\_run; Errors: invalid mapping; Tests: duplicates skipped, dry-run math, start/stop transitions.

### **Module 2 — Idea Generator**

Optional expansion; dedupe by `idea_hash`; partial OK on adapter failures.

### **Module 3 — Article Forge**

Persist `generated_articles(title, body, slug, model_used, prompt_version, article_hash)`; Policies: slug from title (kebab); Errors: model fail → attention; Tests: idempotency by `article_hash`, slug stability.

### **Module 4 — Keyword Catalyst & Indexer**

Write `article_keywords`; **index to vector store as `VectorDoc(id=<article_id:chunk_n>)`**; Tests: stable counts; correct vector payload.

### **Module 5 — SEO Optimizer**

Generate meta/canonical/schema; **do not modify slug if `slug_locked=1`**; Tests: slug immutability; valid JSON schema.

### **Module 6 — Image Artisan**

ComfyUI job submit → background monitor → 180 s timeout → Gemini→DALL·E fallback; write `article_images(status, model_used, file_path)` and log in **`image_usage`**; reuse success on re-run; Errors surfaced to Attention (`comfy_timeout`, etc.).

### **Module 7 — Batch Processor (Pass 1\)**

Sequential M3→M4→M5; enqueue M6; Output: `batch_runs` \+ `batch_items`; Policies: **retries ≤3**; Errors: captured per step; Tests: events present; retries obey cap.

### **Module 8 — WordPress Publisher (Pass 2\)**

Ensure terms; resolve & **lock slug**; if published \< 50 skip linking; else compute link plan → mutate HTML (persist `article_body_html_final`) → write `internal_links`; upload featured image; create/schedule post; Output: `wordpress_posts` rows; Policies: fairness & caps; Tests: slug “-2” path; fairness & per-section cap.

### **Module 9 — Internal Linking Engine (JIT)**

Input: `LinkPlanIn(source_article_id, profile_id, max_links=3)`; Process: Section 6.2 pipeline; **compute both text & HTML offsets**; persist & mutate HTML; Output: `LinkPlanOut`; Policies: idempotency via `(src,tgt,anchor)`; Errors: skip quietly if none eligible; Tests: fairness & cap adherence.

### **Module 10 — Attention Required**

Group by `error_code`; actions to re-run (local/lower temp); reuse successful artifacts; no infinite loops; Tests: clustered UI & clears failures.

---

## **13\) Budgets & rate limiting (implementation detail)**

* **Sliding window:** each paid call logs to `token_usage`; sum last 24h; `budgets.can_spend(provider, estimated_tokens) -> bool`.

* **Interception:** Orchestrator checks budget **before** paid adapters; if over, fallback to local provider; log `{action:'paid_cap_reached', switched_to:'ollama'}`.

* **Rate limits:** Exponential backoff **only for the affected provider/step** (no global stalls).

---

## **14\) Determinism, idempotency & Undo**

* **Idempotent publish:** mutated HTML persisted in `article_body_html_final`; downstream publish always uses that if present (prevents duplicate link insertion).

* **Undo (local DB revert only):** use `batch_run_id` on all mutated tables to (a) restore previous HTML from `article_html_history`, (b) delete `internal_links`, `article_images`, `article_keywords`, `article_seo`, `wordpress_posts`, and (c) delete any articles created by the batch. UI labels this clearly in Command Center (non-destructive remotely).

---

## **15\) Testing strategy & acceptance**

**Unit tests:** Pass1/Pass2 flows (happy & failure); Vector ingest→hybrid→CE→MMR produce stable order; Linking fairness & caps; Budgets cause local fallback; Slug collision appends `-2` and locks. **Acceptance:** with 1,000 published and fairness 5%, no target \> 50 inbound; Comfy timeout → fallback at \~180s; increasing Pass-2 workers improves throughput live; CSV dedupe shows “added vs skipped.”

---

## **16\) Orchestrator pseudocode (key flows)**

### **Pass 1 (sequential)**

See reference skeleton (create batch, iterate ideas, record items, build article → keywords → vector index → SEO → enqueue images; mark ok/fail with retries).

### **Comfy monitor**

Poll, enforce timeout, update statuses, trigger fallback, save image path/model.

### **Pass 2 (workers)**

Create batch, semaphore(workers), gather publish tasks; each flow ensures terms, resolves slug & locks, links (skip \<50), media, publish, persist.

---

