# **Content Spark — Full Build Plan (Everything your code-gen AI needs)**

This is a complete, opinionated specification that an expert engineer/agent can implement module-by-module without prior context. It includes a detailed executive summary, high-level architecture, precise data contracts, database DDL, adapter interfaces, orchestration rules, UI specs, linking/RAG math, budget controls, and acceptance tests. All previously agreed constraints are frozen into the design.

---

## **1\) Executive summary**

**Goal.** Content Spark turns raw ideas into published, internally-linked WordPress posts at scale, with human-in-the-loop tuning and deterministic, reversible batch operations. It favors **local models** and **repeatability** while providing **paid-API fallbacks** under a **sliding 24-hour token budget**.

**Two-pass strategy.**

* **Pass 1 (Generation)**: Runs **one idea at a time** through Article → Keywords/Index → SEO. **Images are queued** to ComfyUI (with fallbacks) and **do not block** Pass 1\.

* **Pass 2 (Publish & Link)**: Runs with **N workers** (default 2; adjustable live). Creates categories/tags as needed, resolves slug collisions deterministically, injects internal links with **fairness** and **per-article caps**, and schedules/publishes to WordPress.

**Linking policy (hard).**

* Max **3 links per article**, **≤1 per section**, **no links** in titles/headers.

* **Target fairness**: no target may receive inbound links from more than **5%** of total published posts.

* **Bootstrap**: skip link insertion until there are **≥50** published posts.

**Budgets & fallbacks.**

* **Paid-API sliding 24h cap**: 1,000,000 tokens (local models excluded). If a paid call would exceed the cap, **auto-switch to local** and emit an event.

* **Images**: ComfyUI first, **3-minute timeout** per image, then **Gemini**, then **DALL·E**.

**Vector system & retrieval (for linking & related).**

* Default: **Qdrant** (dense vectors) \+ **Tantivy/BM25** (sparse) via the same **vector adapter** interface. (Chroma also supported behind the adapter.)

* Ingestion: clean → de-dup → chunk (450–650 tok; 60–100 overlap) → **BGE-M3 embeddings** (local) with SQLite cache → upsert vectors & FT payloads.

* Retrieval: **Dense top\_k=200 \+ BM25 top\_k=200**, blend `0.65*dense + 0.35*sparse` → **cross-encoder rerank** (**bge-reranker-base int8**, keep 40\) → **MMR** (λ=0.3, k=12) → stitch adjacent chunks → plan links.

**Human control.**

* **Workbench** (single-item tabs): Article, Keywords/Index, SEO, Images; with template A/B and “Record as Golden” for test snapshots.

* **Command Center**: CSV idea upload, start/stop batches, dry-run with cost/time estimates, batch history and **Undo (local only)**.

* **Attention Required**: grouped errors, quick re-run actions (prefer local, adjust temperature), open in Workbench.

**Determinism & idempotency.**

* `idea_hash` (per profile) & `article_hash` guard against duplicates.

* `slug_locked` prevents future overwrites.

* Link identity is `(source_id, target_id, anchor_text)` (unique).

* **Undo** references `batch_run_id` on every write.

---

## **2\) High-level architecture**

### **2.1 Components**

* **UI**: Streamlit multipage app

  * Module 0: Global Settings & Profiles

  * Module 1: Command Center \+ CSV idea upload

  * Modules 3–6: Workbench tabs (Article, Keywords/Index, SEO, Images)

  * Module 10: Attention Required dashboard

* **Orchestrator** (Python, asyncio): owns sequencing, concurrency, retries, budgets, events, batch lifecycle

* **Adapters** (swappable):

  * `TextModelAdapter`: OpenAI / Gemini / DeepSeek / Ollama (local)

  * `ImageModelAdapter`: ComfyUI (local) → Gemini → DALL·E

  * `VectorStoreAdapter`: Qdrant+Tantivy (default) or Chroma

  * `WordPressAdapter`: WP REST (Application Passwords)

  * `ComfyUIAdapter`: workflow submit \+ job monitor

* **Data**:

  * SQLite (primary DB)

  * Qdrant (vectors), Tantivy/BM25 (sparse)

  * Local filesystem (`/data/images/...`) or configurable media path

### **2.2 Directory layout**

`content-spark/`  
  `app.py                      # Streamlit entry`  
  `orchestrator/`  
    `__init__.py`  
    `pass1.py`  
    `pass2.py`  
    `jobs.py                   # comfy monitor, background tasks`  
    `budgets.py                # sliding 24h cap logic`  
    `events.py`  
  `adapters/`  
    `__init__.py`  
    `text_model.py`  
    `image_model.py`  
    `vector_store.py`  
    `wordpress.py`  
    `comfyui.py`  
  `db/`  
    `schema.sql                # baseline DDL`  
    `migrations/               # versioned SQL migrations`  
    `dal.py                    # typed queries`  
  `modules/`  
    `settings_ui.py            # Module 0`  
    `command_center.py         # Module 1`  
    `workbench_article.py      # Module 3`  
    `workbench_keywords.py     # Module 4`  
    `workbench_seo.py          # Module 5`  
    `workbench_images.py       # Module 6`  
    `attention.py              # Module 10`  
  `tests/`  
    `fixtures/`  
    `test_pass1.py`  
    `test_pass2.py`  
    `test_vector.py`  
    `test_linking.py`  
    `goldens/`  
  `util/`  
    `hashing.py`  
    `text.py                   # HTML parsing, sectionization`  
    `link_insertion.py`  
    `time.py`  
    `http.py`  
  `config/`  
    `defaults.yaml             # default settings`

---

## **3\) Global constraints (frozen)**

* **Inbound link fairness**: 5% (profile-overrideable)

* **Image timeout**: 180 seconds (profile-overrideable)

* **Paid-API sliding 24h cap**: 1,000,000 tokens (profile-overrideable)

* **Pass-2 workers default**: 2 (profile-overrideable)

* **CSV idea columns**: `seed_idea` (required), `context` (opt), `tone` (opt)

* **No readability/plagiarism hard gates**

* **Categories/tags**: auto-create

* **Slug collision**: append `-2`, `-3`, … then `slug_locked=1`

* **Undo**: local DB revert only

* **Re-runs**: reuse prior successful outputs (e.g., images)

---

## **4\) Database schema (DDL)**

Apply these as migrations in order. All `created_at` default to `CURRENT_TIMESTAMP`.

`-- Core settings`  
`CREATE TABLE IF NOT EXISTS app_settings (`  
  `key TEXT PRIMARY KEY,`  
  `value TEXT NOT NULL`  
`);`

`CREATE TABLE IF NOT EXISTS site_profiles (`  
  `id INTEGER PRIMARY KEY,`  
  `name TEXT NOT NULL`  
`);`

`CREATE TABLE IF NOT EXISTS profile_settings (`  
  `id INTEGER PRIMARY KEY AUTOINCREMENT,`  
  `profile_id INTEGER NOT NULL,`  
  `setting_name TEXT NOT NULL,`  
  `setting_value TEXT NOT NULL,`  
  `UNIQUE(profile_id, setting_name),`  
  `FOREIGN KEY (profile_id) REFERENCES site_profiles(id)`  
`);`

`-- Batches, items, events, budgets`  
`CREATE TABLE IF NOT EXISTS batch_runs (`  
  `id TEXT PRIMARY KEY,`  
  `run_type TEXT CHECK(run_type IN ('pass1','pass2')),`  
  `started_at DATETIME, finished_at DATETIME,`  
  `status TEXT CHECK(status IN ('running','completed','failed','canceled')),`  
  `item_count INTEGER, error_count INTEGER, notes TEXT`  
`);`

`CREATE TABLE IF NOT EXISTS batch_items (`  
  `id INTEGER PRIMARY KEY,`  
  `batch_run_id TEXT NOT NULL,`  
  `entity_type TEXT,                 -- 'idea' | 'article' | 'image'...`  
  `entity_id INTEGER,`  
  `step TEXT,                        -- freeform: 'article', 'keywords', ...`  
  `status TEXT,                      -- queued|running|completed|failed|attention_required`  
  `error TEXT,`  
  `started_at DATETIME, finished_at DATETIME,`  
  `FOREIGN KEY (batch_run_id) REFERENCES batch_runs(id)`  
`);`

`CREATE TABLE IF NOT EXISTS events (`  
  `id INTEGER PRIMARY KEY,`  
  `ts DATETIME DEFAULT CURRENT_TIMESTAMP,`  
  `module TEXT, entity_type TEXT, entity_id TEXT,`  
  `action TEXT, ok BOOLEAN, duration_ms INTEGER,`  
  `error_code TEXT, payload_preview TEXT`  
`);`

`CREATE TABLE IF NOT EXISTS token_usage (`  
  `id INTEGER PRIMARY KEY,`  
  `provider TEXT NOT NULL,           -- 'openai','gemini','serpapi', etc.`  
  `tokens INTEGER NOT NULL,`  
  `occurred_at DATETIME NOT NULL`  
`);`  
`CREATE INDEX IF NOT EXISTS ix_token_usage_window ON token_usage(occurred_at);`

`-- Ideas → Articles → Keywords → SEO → Images`  
`CREATE TABLE IF NOT EXISTS generated_ideas (`  
  `id INTEGER PRIMARY KEY,`  
  `profile_id INTEGER NOT NULL,`  
  `seed_idea TEXT NOT NULL,`  
  `context TEXT, tone TEXT,`  
  `idea_hash TEXT,                   -- sha1(lower(trim(seed_idea)) + '|' + profile_id)`  
  `status TEXT DEFAULT 'idea',       -- idea|failed|content_generated|ready_to_publish|...`  
  `created_at DATETIME DEFAULT CURRENT_TIMESTAMP,`  
  `UNIQUE(profile_id, idea_hash),`  
  `FOREIGN KEY (profile_id) REFERENCES site_profiles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS generated_articles (`  
  `id INTEGER PRIMARY KEY,`  
  `idea_id INTEGER NOT NULL,`  
  `profile_id INTEGER NOT NULL,`  
  `article_title TEXT,`  
  `article_body TEXT,`  
  `article_hash TEXT,                -- sha1(lower(title) + '|' + idea_id)`  
  `slug TEXT,`  
  `slug_locked BOOLEAN DEFAULT 0,`  
  `model_used TEXT,`  
  `prompt_version INTEGER DEFAULT 1,`  
  `created_at DATETIME DEFAULT CURRENT_TIMESTAMP,`  
  `FOREIGN KEY (idea_id) REFERENCES generated_ideas(id),`  
  `FOREIGN KEY (profile_id) REFERENCES site_profiles(id)`  
`);`  
`CREATE UNIQUE INDEX IF NOT EXISTS ux_article_hash ON generated_articles(article_hash);`

`CREATE TABLE IF NOT EXISTS article_keywords (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER,`  
  `keyword TEXT,`  
  `kind TEXT CHECK(kind IN ('primary','secondary','entity','question')),`  
  `importance REAL,`  
  `FOREIGN KEY (article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS article_seo (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER,`  
  `meta_title TEXT, meta_description TEXT,`  
  `canonical_url TEXT, noindex BOOLEAN DEFAULT 0,`  
  `schema_json TEXT,`  
  `template_version INTEGER,`  
  `FOREIGN KEY (article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS article_images (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER,`  
  `role TEXT CHECK(role IN ('hero','inline')),`  
  `file_path TEXT,`  
  `model_used TEXT,                  -- 'comfyui'|'gemini'|'dalle'`  
  `status TEXT,                      -- queued|succeeded|failed`  
  `created_at DATETIME DEFAULT CURRENT_TIMESTAMP,`  
  `FOREIGN KEY (article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS comfy_jobs (`  
  `id INTEGER PRIMARY KEY AUTOINCREMENT,`  
  `article_id INTEGER NOT NULL,`  
  `job_id TEXT NOT NULL,`  
  `status TEXT CHECK(status IN ('queued','running','succeeded','failed')) NOT NULL,`  
  `started_at DATETIME, finished_at DATETIME,`  
  `FOREIGN KEY(article_id) REFERENCES generated_articles(id)`  
`);`

`-- WordPress + Links`  
`CREATE TABLE IF NOT EXISTS wordpress_posts (`  
  `id INTEGER PRIMARY KEY,`  
  `article_id INTEGER,`  
  `wp_post_id INTEGER,`  
  `status TEXT,                      -- draft|future|publish`  
  `slug TEXT,`  
  `published_at DATETIME,`  
  `scheduled_date DATETIME,`  
  `FOREIGN KEY (article_id) REFERENCES generated_articles(id)`  
`);`

`CREATE TABLE IF NOT EXISTS internal_links (`  
  `id INTEGER PRIMARY KEY,`  
  `source_article_id INTEGER,`  
  `target_article_id INTEGER,`  
  `anchor_text TEXT,`  
  `source_offset_start INTEGER,`  
  `source_offset_end INTEGER,`  
  `FOREIGN KEY (source_article_id) REFERENCES generated_articles(id),`  
  `FOREIGN KEY (target_article_id) REFERENCES generated_articles(id)`  
`);`  
`CREATE UNIQUE INDEX IF NOT EXISTS ux_link_once`  
  `ON internal_links (source_article_id, target_article_id, anchor_text);`  
`CREATE INDEX IF NOT EXISTS ix_links_target ON internal_links(target_article_id);`

`-- Templates & A/B`  
`CREATE TABLE IF NOT EXISTS templates (`  
  `id INTEGER PRIMARY KEY AUTOINCREMENT,`  
  `kind TEXT NOT NULL,                  -- 'article','seo','image_prompt', etc.`  
  `version INTEGER NOT NULL,`  
  `name TEXT NOT NULL,`  
  `body TEXT NOT NULL,`  
  `status TEXT CHECK(status IN ('draft','published')) DEFAULT 'draft',`  
  `UNIQUE(kind, version)`  
`);`

`CREATE TABLE IF NOT EXISTS prompt_tests (`  
  `id INTEGER PRIMARY KEY AUTOINCREMENT,`  
  `template_id_a INTEGER NOT NULL,`  
  `template_id_b INTEGER NOT NULL,`  
  `article_id INTEGER,`  
  `winner TEXT CHECK(winner IN ('A','B','tie')),`  
  `notes TEXT,`  
  `created_at DATETIME DEFAULT CURRENT_TIMESTAMP`  
`);`

---

## **5\) Settings (effective value rules)**

* **Precedence**: `effective = profile_settings[profile][key] if present else app_settings[key] else default`

* **Top-level keys** (global defaults; all can be overridden per profile):

  * `link.max_inbound_link_share` → default `0.05`

  * `link.max_links_per_article` → default `3`

  * `link.max_links_per_section` → default `1`

  * `link.bootstrap_min_published` → default `50`

  * `image.timeout_seconds` → default `180`

  * `budget.paid_token_cap_24h` → default `1000000`

  * `pass2.default_workers` → default `2`

  * `models.text.*` → per step, provider/model/params

  * `models.image.*` → comfy workflow name/params; fallbacks enabled

  * `wordpress.url`, `wordpress.user`, `wordpress.app_password`

  * `vector.backend` (`qdrant`|`chroma`), `vector.qdrant.url`, `vector.collection` etc.

---

## **6\) Adapters (interfaces & expectations)**

All adapters must be pure (side-effect free except their I/O), fully typed, and raise domain-specific exceptions with error codes for the Attention dashboard.

### **6.1 `TextModelAdapter`**

`class TextRequest(BaseModel):`  
    `prompt: str`  
    `system: str | None = None`  
    `max_tokens: int | None = None`  
    `temperature: float = 0.2`  
    `top_p: float = 0.9`

`class TextResponse(BaseModel):`  
    `text: str`  
    `tokens_in: int`  
    `tokens_out: int`  
    `model_name: str`  
    `provider: Literal["openai","gemini","deepseek","ollama"]`

`class TextModelAdapter(Protocol):`  
    `async def generate(self, req: TextRequest) -> TextResponse: ...`

* When provider is paid, the adapter **must** call `budgets.record_tokens(provider, tokens_in+tokens_out)` and enforce sliding-window cap (`budgets.can_spend`).

### **6.2 `ImageModelAdapter`**

`class ImageGenRequest(BaseModel):`  
    `prompt: str`  
    `size: str = "1024x1024"`  
    `role: Literal["hero","inline"] = "hero"`

`class ImageGenResponse(BaseModel):`  
    `path: str`  
    `model_name: str`  
    `provider: Literal["comfyui","gemini","dalle"]`

`class ImageModelAdapter(Protocol):`  
    `async def generate(self, req: ImageGenRequest) -> ImageGenResponse: ...`

### **6.3 `ComfyUIAdapter`**

`class ComfyUIAdapter(Protocol):`  
    `async def submit(self, prompt: str, size: str) -> str: ...  # returns job_id`  
    `async def status(self, job_id: str) -> Literal["queued","running","succeeded","failed"]: ...`  
    `async def result(self, job_id: str) -> str: ...  # returns local file path`

* The orchestrator monitors jobs; if elapsed \> `image.timeout_seconds`, cancel/fail and hand off to `ImageModelAdapter` fallback chain.

### **6.4 `VectorStoreAdapter`**

`class VectorDoc(BaseModel):`  
    `id: str`  
    `text: str`  
    `meta: dict  # {doc_id, title, url, section_anchor, htrail, doctype, tags, published_at, char_start, char_end, hash5gram}`

`class VectorQuery(BaseModel):`  
    `text: str`  
    `top_k_dense: int = 200`  
    `top_k_sparse: int = 200`

`class HybridCandidate(BaseModel):`  
    `doc_key: str`  
    `score_dense: float`  
    `score_sparse: float`  
    `score_blend: float`  
    `meta: dict`

`class VectorStoreAdapter(Protocol):`  
    `async def upsert(self, docs: list[VectorDoc]) -> None: ...`  
    `async def hybrid_search(self, q: VectorQuery) -> list[HybridCandidate]: ...`

* Qdrant impl: cosine vectors \+ Tantivy BM25 on `text` field; blend inside adapter or return both scores.

### **6.5 `WordPressAdapter`**

`class Term(BaseModel):`  
    `id: int`  
    `name: str`

`class PostCreate(BaseModel):`  
    `title: str`  
    `slug: str`  
    `html: str`  
    `status: Literal["draft","future","publish"] = "draft"`  
    `categories: list[int] = []`  
    `tags: list[int] = []`  
    `canonical_url: str | None = None`  
    `meta_title: str | None = None`  
    `meta_description: str | None = None`  
    `schema_json: str | None = None`  
    `date_gmt: str | None = None      # for scheduling`  
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

* Use WP REST API (`/wp-json/wp/v2/...`) with Basic Auth via Application Password. For SEO fields, if using Yoast/SEOPress, add plugin-specific meta fields via `meta` payload or follow plugin docs (expose this as adapter options).

---

## **7\) Vector/RAG details (ingest & query)**

### **7.1 Ingestion pipeline**

1. **Normalize text** (strip boilerplate, decode HTML, unify whitespace).

2. **Near-dup filter** using rolling 5-gram hash (`hash5gram`) at document level.

3. **Chunk** into 450–650 token segments (60–100 overlap). Provide `char_start/char_end` per chunk.

4. **Embed** with **BGE-M3** locally (llama-cpp or compatible), batch 64–128; persist cache in SQLite:

   * `embeddings_cache(text_sha1 TEXT PRIMARY KEY, dim INT, model TEXT, vec BLOB)`

5. **Upsert** to Qdrant `chunks_main` with cosine metric; store full text for BM25.

### **7.2 Query pipeline (for linking)**

1. Build query from source article: title \+ top H2/H3 \+ primary keywords.

2. **Hybrid search**: dense top\_k=200 \+ BM25 top\_k=200.

3. **Blend** scores: `score = 0.65*dense + 0.35*sparse` → take top \~250.

4. **Cross-encoder rerank** with **bge-reranker-base (int8/onnx)**; keep top **40**.

5. **MMR** diversification with λ=0.3, k=12; cap ≤2–3 chunks per doc; **stitch** adjacent chunks.

6. **Eligibility filters**:

   * Exclude self and same-slug duplicates.

   * Exclude targets scheduled **after** source publish date.

   * Enforce **target fairness**: `inbound(target)/total_published < 0.05`.

7. Produce top candidates with suggested anchor phrases (3–7 words) and insertion offsets.

---

## **8\) Linking engine (insertion rules)**

* **Cap per article**: 3 links; **per section**: ≤1.

* **No headings**: never insert anchors inside `<h1..h6>`.

* **Anchor selection**:

  * Prefer exact/natural phrase occurrence in body (3–7 words), not stop-word heavy.

  * Map phrase to `(start,end)` offsets in plain text and in HTML.

* **Idempotency**: a planned link is unique by `(source_id, target_id, anchor_text)`.

* **Persistence**: write each accepted link to `internal_links` with offsets.

* **HTML mutation**: inject `<a href="...">anchor</a>` at the stored offsets (use an HTML AST to avoid corrupting markup).

---

## **9\) Orchestrator (policies & flows)**

### **9.1 Pass 1 (sequential)**

For each `generated_ideas.status == 'idea'` (or retryable failure):

1. **Article** (Module 3):

   * Build prompt from template \+ idea/context/tone.

   * Call text adapter; persist `generated_articles`, compute `article_hash`.

   * Build slug; do **not** lock yet.

2. **Keywords & Index** (Module 4):

   * Extract primary/secondary/entities/questions.

   * Index article into vector store (title \+ body; doc\_id \= article\_id).

3. **SEO** (Module 5):

   * Generate meta fields; if `slug_locked=1`, do not modify slug.

4. **Images** (Module 6):

   * Submit ComfyUI job(s) (`hero` required; inline optional).

   * Insert row in `comfy_jobs`; **do not block**.

5. Mark idea as `content_generated` on success; on failure, increment retry\_count up to 3 then `attention_required` with `error_code` and human-readable message.

**Events**: emit `started/finished` for each step with `duration_ms`, `ok`, and `payload_preview` (e.g., first 120 chars).

### **9.2 ComfyUI monitor (background)**

* Poll every 5–10 s (exponential backoff capped at 20 s).

* If job `elapsed > image.timeout_seconds (180)` → mark failed and **invoke ImageModelAdapter** fallback chain (Gemini → DALL·E).

* On success/failure, write `article_images` (`status`, `model_used`, `path`) and update `comfy_jobs`.

### **9.3 Pass 2 (concurrent workers, default 2\)**

For each `generated_articles` ready to publish:

1. **Schedule** (if desired) using profile velocity (or publish immediately).

2. **Ensure terms** (categories/tags) via WP adapter; persist term IDs locally.

3. **Slug**: if collision at WP, append `-2`, `-3`, … until unique; then set `slug_locked=1` locally.

4. **Link plan** (JIT):

   * If `published_count < 50`: **skip** linking entirely.

   * Else run linking engine; enforce caps and fairness; apply HTML mutation; persist `internal_links`.

5. **Media**: upload featured image; set `featured_media` ID.

6. **Create post** via WP adapter with SEO fields & canonical/schema as available.

7. Persist returned `wp_post_id`, `slug`, `link`, `status`, `scheduled_date/published_at`.

**Concurrency**: semaphore \= `pass2.default_workers` (UI-adjustable live). If rate limited by a provider, **throttle only that step**.

### **9.4 Budgets (sliding 24h)**

* `budgets.can_spend(provider, tokens_needed)`:

  * sum `token_usage.tokens` where `occurred_at >= now()-24h` for paid providers.

  * compare with `budget.paid_token_cap_24h` effective setting.

* If over cap: **switch** to local adapter for that step, log event `paid_cap_reached`.

---

## **10\) UI specs**

### **10.1 Module 0 — Global Settings & Profiles**

* Sections: Providers (WP, Ollama, ComfyUI, OpenAI/Gemini/DeepSeek), Models, Vector Store, Linking, Images, Budgets, Concurrency.

* **Effective value** display: shows Global value, Profile override, and the resolved value.

* Buttons: **Test Connection** for WP, Ollama, ComfyUI, OpenAI/Gemini/DeepSeek.

* Profile picker at top; overrides saved to `profile_settings`.

### **10.2 Module 1 — Command Center (+ CSV)**

* **CSV upload** (top): choose profile; map CSV columns → preview → import. Report: “X added, Y duplicates skipped.”

* **Dry-Run**: choose run type (Pass 1/2), profile, item limit; show est. tokens/time/cost.

* **Batch controls**: Start/Stop buttons; live **activity feed** (events); progress bars; batch history table (with **Undo**).

* **Undo Batch**: reverts local writes associated with `batch_run_id` (non-destructive remotely).

### **10.3 Workbench (Modules 3–6; single-item)**

* Header: active profile \+ idea/article ID.

* Tabs with **visual stepper**:

  1. Article Forge (prompt editor, preview text)

  2. Keyword Catalyst (list w/ kinds & importance)

  3. SEO Optimizer (meta fields; schema editor)

  4. Image Artisan (prompt, submit, status, gallery)

* **Templates**: choose template version; **A/B** two templates side-by-side; “Select Winner” publishes winning version and bumps template version.

* Button: **Record as Golden** (writes snapshot to `/tests/goldens/...`).

### **10.4 Attention Required**

* Group by `error_code` with counts and example rows.

* Row actions: “Re-run with local model,” “Re-run with lower temperature,” “Open in Workbench.”

* Bulk re-run permitted with a confirmation.

---

## **11\) Module-by-module build detail**

For each module: Purpose, Inputs, Outputs, Side-effects, Policies, Errors, Tests.

### **Module 0 — Global Settings & Profiles**

* **Purpose**: Manage app & profile settings; show effective values; test external connections.

* **Input**: forms (Streamlit); `SaveSettingRequest(scope, profile_id, key, value)`.

* **Output**: writes to `app_settings` or `profile_settings`.

* **Side-effects**: none.

* **Policies**: single-user → secrets visible; tests emit events (`ok`/`fail`).

* **Errors**: adapter exceptions → toast \+ event.

* **Tests**: precedence correctness; all “Test Connection” results logged.

### **Module 1 — Command Center (+ CSV)**

* **Purpose**: Central operations hub.

* **Input**: CSV with headers `seed_idea` (req), `context`, `tone`.

* **Process**:

  * Map columns; compute `idea_hash = sha1(lower(trim(seed_idea)) + '|' + profile_id)`.

  * Insert unique rows to `generated_ideas(status='idea')`.

* **Batch API**:

  * `POST /batches/start` `{run_type, profile_id, limit?}`

  * `POST /batches/stop/{id}`

  * `GET /batches/{id}/progress`

  * `POST /batches/dry_run` `{run_type, profile_id, limit?}` → `{items, est_tokens, est_time_sec}`

* **Output**: UI feedback; DB inserts.

* **Errors**: invalid mapping; empty seed\_idea; show row numbers that failed.

* **Tests**: duplicates skipped; dry-run math; batch start/stop state transitions.

### **Module 2 — Idea Generator**

* **Purpose**: (Optional) Expand seeds to multiple angles; may pull PAA/related via search if configured; otherwise pass-through.

* **Input**: `IdeaIn(profile_id, seed_idea, context?, tone?)`.

* **Output**: New `generated_ideas` rows (status `idea`).

* **Side-effects**: events.

* **Policies**: dedupe by `idea_hash`.

* **Errors**: adapter search failures → event; partial ok.

* **Tests**: deterministic dedupe; multi-row import.

### **Module 3 — Article Forge**

* **Input**: `idea_id`, active profile, template version.

* **Process**: prompt build → `TextModelAdapter.generate()` → persist `generated_articles` (title/body/slug/model\_used/prompt\_version/article\_hash).

* **Output**: `ArticleOut(article_id, title, body, slug)`.

* **Policies**: no links in headings; slug from title (kebab).

* **Errors**: model call fails → `attention_required`.

* **Tests**: idempotency by `article_hash`; slug stability.

### **Module 4 — Keyword Catalyst & Indexer**

* **Input**: `article_id`.

* **Process**: extract keywords/entities/questions; write `article_keywords`. Index to vector store as `VectorDoc(id=<article_id:chunk_n>, text, meta)`.

* **Output**: `KeywordsOut`.

* **Policies**: at least 1 primary; entities/questions reasonable.

* **Errors**: vector upsert failures surfaced; continue if keywords OK.

* **Tests**: stable counts; vector upsert called with right payload.

### **Module 5 — SEO Optimizer**

* **Input**: `article_id`, `template_version`.

* **Process**: generate meta title/description/canonical/schema; **do not modify slug if `slug_locked=1`**.

* **Output**: `SEOOut`.

* **Errors**: schema JSON validation errors → fallback to minimal schema or omit; flag warning.

* **Tests**: slug immutability; schema valid JSON.

### **Module 6 — Image Artisan**

* **Input**: `article_id`, role (`hero` required; `inline` optional), prompt.

* **Process**: `ComfyUIAdapter.submit()` → save `comfy_jobs`(job\_id) → **monitor** in background → on success write `article_images(status='succeeded', model_used='comfyui')`; on timeout/failure → fallbacks via `ImageModelAdapter` (Gemini → DALL·E).

* **Output**: `ImageGenResponse`.

* **Policies**: 180 s timeout; reuse successful image on re-run.

* **Errors**: all surfaced to Attention with `error_code` (`comfy_timeout`, `image_fallback_failed`, etc.).

* **Tests**: simulate timeout → fallback recorded; file path exists.

### **Module 7 — Batch Processor (Pass 1\)**

* **Input**: profile\_id, limit (optional).

* **Process**: strictly sequential per idea: M3 → M4 → M5; enqueue M6; **do not block** on images.

* **Output**: `batch_runs` \+ `batch_items`; updated statuses (`content_generated`).

* **Policies**: retries ≤3; failed → Attention.

* **Errors**: captured per step; continue next idea.

* **Tests**: runs through entire pipeline; events present; retries obey cap.

### **Module 8 — WordPress Publisher (Pass 2\)**

* **Input**: profile\_id; worker count (UI default 2).

* **Process**:

  1. Ensure categories/tags → IDs.

  2. Resolve slug collisions (appenders) → set `slug_locked=1`.

  3. If published \< 50 → **skip** linking.

  4. Else: compute link plan (Module 9\) → mutate HTML → write `internal_links`.

  5. Upload featured image → `featured_media`.

  6. Create/schedule post (status draft/future/publish).

* **Output**: `wordpress_posts` rows (ids, slug, status, timestamps).

* **Policies**: fairness cap, per-article/section caps; no links in headings.

* **Errors**: WP 4xx/5xx capture; partial ok if media uploaded but post failed (retry).

* **Tests**: slug “-2” path; fairness enforced; per-section cap enforced.

### **Module 9 — Internal Linking Engine (JIT)**

* **Input**: `LinkPlanIn(source_article_id, profile_id, max_links=3)`.

* **Process**: Vector/RAG pipeline (Section 7.2) → filter by schedule/fairness → pick anchors → compute offsets → persist plan & apply to HTML.

* **Output**: `LinkPlanOut(chosen=[{target_id, score, anchor_text, range}...])`.

* **Policies**: section cap; no headings; idempotency via `(src, tgt, anchor)` unique index.

* **Errors**: if no eligible targets, skip quietly; log reason code.

* **Tests**: on synthetic corpus, verify fairness math & cap adherence.

### **Module 10 — Attention Required**

* **Input**: all items with status `attention_required` or failed in `batch_items`.

* **Process**: group by `error_code`; show first error snippet; allow bulk or individual re-runs with altered params (e.g., local model, lower temp).

* **Output**: updated statuses; events.

* **Policies**: re-runs **reuse** successful artifacts (e.g., keep images).

* **Errors**: logged; do not loop infinitely.

* **Tests**: clustered UI; re-run clears failures.

---

## **12\) Budgets & rate limiting (implementation detail)**

* **Sliding window**: store each paid call’s token usage in `token_usage`; sum `>= now()-24h`.

* `budgets.can_spend(provider, estimated_tokens) -> bool`

* **Interception**: Orchestrator asks budgets before calling paid adapters. If false → fallback to local provider; add event `{action:'paid_cap_reached', switched_to:'ollama'}`.

* **Rate limits**: Exponential backoff **only for the affected provider/step**; do not stall other workers.

---

## **13\) Error codes (suggested)**

* `model_timeout`, `model_rate_limited`, `model_unavailable`

* `comfy_timeout`, `comfy_failed`

* `image_fallback_failed`

* `wp_auth_failed`, `wp_conflict_slug`, `wp_5xx`

* `vector_upsert_failed`, `vector_query_failed`

* `budget_cap_reached`

* `link_fairness_exceeded`, `link_no_eligible_targets`

---

## **14\) Testing strategy**

* **Goldens**: Workbench button writes canonical JSON/HTML to `/tests/goldens/<module>/<id>.json`.

* **Unit tests**:

  * Pass 1/2 flows (happy & failure)

  * Vector: ingest \+ hybrid \+ CE \+ MMR → stable order on fixture corpus

  * Linking: fairness and caps enforced

  * Budgets: cap breach causes local fallback

  * Slug collision: appends `-2` and locks

* **Acceptance**:

  * With 1,000 published and fairness 5%, no target \> 50 inbound.

  * Comfy timeout triggers fallback at \~180s.

  * Changing Pass-2 workers from 2→5 increases throughput without restart.

  * CSV dedupe produces “added vs skipped” report.

---

## **15\) Pseudocode & key functions**

### **15.1 Orchestrator Pass 1 (sequential)**

`async def run_pass1(profile_id: int, limit: int | None):`  
    `batch = create_batch('pass1')`  
    `ideas = dal.fetch_ideas(profile_id, limit)`  
    `for idea in ideas:`  
        `record_item(batch, 'idea', idea.id, 'article', 'running')`  
        `try:`  
            `art = await build_article(idea)                 # Module 3`  
            `keywords = await extract_keywords(art.id)       # Module 4`  
            `await index_article(art.id)                     # Vector upsert`  
            `seo = await build_seo(art.id)                   # Module 5`  
            `await enqueue_images(art.id)                    # Module 6 (non-blocking)`  
            `mark_item_ok(batch, 'idea', idea.id, 'article')`  
            `dal.set_idea_status(idea.id, 'content_generated')`  
        `except KnownError as e:`  
            `mark_item_fail(batch, 'idea', idea.id, 'article', e)`  
            `continue`  
    `finish_batch(batch)`

### **15.2 Comfy monitor**

`async def monitor_comfy_jobs():`  
    `while True:`  
        `jobs = dal.fetch_pending_comfy_jobs()`  
        `for j in jobs:`  
            `elapsed = now() - j.started_at`  
            `if elapsed.seconds > settings.image.timeout_seconds:`  
                `dal.update_comfy_status(j.id, 'failed')`  
                `await fallback_image_generation(j.article_id)`  
                `continue`  
            `st = await comfy.status(j.job_id)`  
            `if st == 'succeeded':`  
                `path = await comfy.result(j.job_id)`  
                `dal.save_image(j.article_id, 'hero', path, 'comfyui', 'succeeded')`  
                `dal.update_comfy_status(j.id, 'succeeded')`  
            `elif st == 'failed':`  
                `dal.update_comfy_status(j.id, 'failed')`  
                `await fallback_image_generation(j.article_id)`  
        `await asyncio.sleep(next_backoff())`

### **15.3 Pass 2 (workers)**

`async def run_pass2(profile_id: int, workers: int):`  
    `batch = create_batch('pass2')`  
    `sem = asyncio.Semaphore(workers)`  
    `posts = dal.fetch_ready_to_publish(profile_id)`  
    `async def worker(article_id):`  
        `async with sem:`  
            `await publish_article_flow(batch, article_id)`  
    `await asyncio.gather(*(worker(p.id) for p in posts))`  
    `finish_batch(batch)`

### **15.4 Link planning**

`async def link_plan(source_article_id: int, profile_id: int) -> list[LinkCandidate]:`  
    `if dal.published_count(profile_id) < settings.link.bootstrap_min_published:`  
        `return []`  
    `query = build_query_from_article(source_article_id)`  
    `cands = await vector.hybrid_search(VectorQuery(text=query))`  
    `reranked = rerank_ce(cands)[:40]`  
    `diverse = mmr(reranked, k=12, lambda_=0.3)`  
    `eligible = []`  
    `for c in diverse:`  
        `if violates_fairness(c.doc_id, profile_id): continue`  
        `if scheduled_after_source(c.doc_id, source_article_id): continue`  
        `anchor = select_anchor_phrase(source_article_id, c.doc_id)`  
        `if not anchor or duplicates_existing(source_article_id, c.doc_id, anchor): continue`  
        `eligible.append(LinkCandidate(...))`  
        `if len(eligible) == settings.link.max_links_per_article: break`  
    `return enforce_per_section_cap(eligible, settings.link.max_links_per_section)`

---

## **16\) Deployment & ops notes**

* **Runtime**: Python 3.11+, venv, `uvloop` optional.

* **Service ports**: Streamlit (8501), optional FastAPI (8000). Remote Ollama/ComfyUI host/port in settings.

* **File storage**: images under `data/images/<profile>/<article_id>/...`.

* **Backups**: snapshot SQLite and Qdrant collections before major runs (optional CLI).

* **Logging**: file \+ console; events also stored in DB.

---

## **17\) Deliverables checklist (definition of done)**

* All modules implemented and routable from Streamlit.

* All adapters implemented with stubs/mocks for offline tests.

* DDL applied; migrations scripted; seed data for demo profile.

* Workbench A/B and Goldens active.

* Pass-1 and Pass-2 working with the exact policies in this spec.

* Vector ingestion CLI and query tested end-to-end.

* Attention board with actionable errors.

* Acceptance tests pass (Section 14).

