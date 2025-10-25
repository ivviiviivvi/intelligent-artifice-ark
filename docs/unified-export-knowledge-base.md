# Unified AI Conversation Export & Personal Knowledge Base Guide

This guide consolidates prior research on the ChatGPT Exporter userscript, compares it with other AI conversation export options, and outlines how to unify data from multiple personal storage providers into a deduplicated knowledge base.

## ChatGPT Exporter Capabilities
- **Purpose**: GreasyFork/Tampermonkey userscript tailored for OpenAI ChatGPT web clients, injecting UI elements to archive conversation history for later reuse.
- **Ingestion flow**: Observes DOM mutations (via `sentinel-js`) to insert a sidebar menu, discovers the active conversation using ChatGPT's `/backend-api` endpoints, and appends per-message timestamps during retrieval.
- **Network handling**: Maps the current origin to the appropriate ChatGPT backend host, reuses the authenticated browser session for API calls, and iterates through conversation catalogs (including “projects”/gizmos) for bulk exports.
- **Normalization**: Produces linear user/assistant exchanges enriched with model metadata while hiding system/internal nodes to keep transcripts readable.
- **Export surface**: Offers one-click exports to plain text, HTML, Markdown, PNG, OpenAI JSON, TavernAI/SillyTavern JSONL, and oobabooga formats, alongside bulk-download tooling.
- **UI & settings**: Injected menu adapts between desktop/mobile layouts, with toggles for timestamp styles and metadata capture.
- **Localization**: Bundles i18next translations for English, Spanish, French, Indonesian, Japanese, Russian, Turkish, Simplified Chinese, and Traditional Chinese, persisting language choice via Tampermonkey storage and honoring browser/OpenAI locale hints.
- **Coverage scope**: Hard-codes OpenAI API hosts and model mappings; external providers (Grok, Gemini, Microsoft Copilot, Anthropic Claude, etc.) require new adapters before ingestion/export works.

## Survey of Alternative AI Chat Exporters
- **Browser extensions/userscripts**
  - *ChatGPT Exporter* (this project) for flexible downloads.
  - *ChatGPT to Notion*, *ChatGPT History Exporter*, *ChatGPT Prompt Genius* reuse ChatGPT's APIs but push transcripts into Notion/Markdown or provide search enhancements.
  - *ShareGPT* and *ChatGPT Saver* publish conversations to hosted catalogs for sharing through unique URLs.
- **Cross-provider desktop clients**
  - *OpenAI ChatGPT Desktop* forks, *Raycast AI*, *AnythingLLM*, and *ChatALL* log chats locally by intercepting requests; many support OpenAI, Anthropic, Gemini, and Claude with user-supplied API keys.
  - *LocalSend* and *LlamaIndex Playground* import transcripts (converted to JSONL) from multiple providers.
- **Cloud automation platforms**
  - Zapier, Make (Integromat), and n8n provide connectors to OpenAI, Anthropic, Google AI Studio, etc., so prompts/responses can be archived in Airtable, Notion, or Google Sheets.
  - SaaS offerings like *RetentionEngine* and *Superwhisper* target customer-support bots across GPT, Claude, and other LLMs.
- **CLI/API-focused tooling**
  - *LangSmith* and *OpenPipe* capture prompts/responses via SDK hooks for any LLM call.
  - *Sema4.ai Transcript Extractor* turns exported chat HTML/JSON into JSONL for fine-tuning or vector indexing.
  - Self-hosted UIs such as *ChatGPT-Next-Web* maintain local IndexedDB/SQLite histories that can be exported as JSON.

## Unified JSON Schema Recommendations
- **Core fields**: `conversation_id`, ISO-8601 `timestamp`, `source` identifier, `participants`, and a `messages` array containing `id`, `role`, `content`, `attachments`, and `metadata`.
- **Model provenance**: Store `model`, `temperature`, `top_p`, and other parameters when available.
- **Formats**: Favor JSON Lines (JSONL) for streaming/vector DB ingestion while also supporting hierarchical JSON for nested metadata.
- **Schema enforcement**: Maintain a JSON Schema (Draft 7+) and validate via tools such as `ajv`, Python's `jsonschema`, or TypeScript's `spectypes`.
- **Metadata extensions**: Reserve slots for embeddings, NLP-derived `entities`, user-defined `tags`, and `source_uri` references back to original storage paths.

## Collecting Historical Data Across Storage Providers
1. **Mac local content**
   - Query Spotlight indexes (`~/Library/Metadata/CoreSpotlight`) or app-specific SQLite stores (Notes, Messages) using AppleScript or `sqlite3`.
   - Mirror key directories with `rsync` or ChronoSync; convert `.rtf`, `.pages`, and `.numbers` via `textutil` or `libreoffice` CLI to produce text-friendly formats.
2. **iCloud Drive**
   - Enable "Download originals" and script exports with `rclone` or `icloudpd`.
   - Use `icloudexport` or `apple-notes-exporter` to dump Notes/Reminders as JSON or Markdown.
3. **Dropbox**
   - Leverage `rclone`'s Dropbox backend for selective sync; collect metadata using `rclone lsjson`.
   - Export Dropbox Paper documents through the API (`/docs/list` + `/docs/download`) to Markdown/JSON.
4. **Google Drive**
   - Use Google Takeout for comprehensive archives (Docs→HTML/ODT, Sheets→CSV) and `rclone` or the `drive` CLI for ongoing sync.
   - Convert Google Docs to Markdown with `pandoc` post-export.
5. **AI chat services**
   - **ChatGPT**: Employ this userscript or the official Settings → Data Controls export for JSON archives.
   - **Claude**: Download workspace archives from Anthropic's `/beta/account/export` endpoint.
   - **Gemini**: Pull conversations through Google Takeout's "AI Studio conversations" option.
   - **Copilot**: Request data via Microsoft's Privacy Dashboard exports (CSV/JSON).
   - Normalize every source into the shared schema with targeted ETL scripts.

## Deduplication and Canonicalization Strategy
- **Hashing**: Compute normalized SHA-256 hashes over text plus metadata (timestamp, participants) and store them in a `dedupe_index` (SQLite table or Redis set) to prevent re-imports.
- **Thread reconciliation**: Merge overlapping exports using shared `conversation_id` when possible; otherwise cluster by timestamp proximity (±5 seconds) and text similarity. Apply simhash or minHash for near-duplicate long-form documents.
- **Version control**: Preserve originals; write processed outputs to a `processed/` directory with manifest JSON referencing source paths/hashes, and version datasets with Git or DVC for historical diffing.

## Building the Knowledge Base
1. **Ingestion pipeline**
   - Assemble an ETL workflow (Airflow, Dagster, or cron + scripts) covering discovery → extraction → normalization → dedupe → persistence.
   - Use type-safe validation (`pydantic`, TypeScript interfaces) at each stage.
2. **Storage layers**
   - Primary relational store: SQLite or PostgreSQL for transcripts and metadata.
   - Vector store: Weaviate, Pinecone, Chroma, or PostgreSQL with `pgvector` for semantic search.
   - Object storage: S3-compatible bucket (or local MinIO) for binaries (PDF, audio).
3. **Indexing & enrichment**
   - Run OCR with Tesseract on images/PDFs and transcribe audio via Whisper or macOS dictation APIs.
   - Generate embeddings using OpenAI `text-embedding-3-large`, Cohere `embed-multilingual-v3`, or local models like `Instructor-large`.
   - Apply spaCy or similar NLP pipelines for entity extraction, summarization, and topic tagging.
4. **Access interface**
   - Build a search UI (Next.js, Retool, Streamlit) supporting keyword, tag, and vector queries.
   - Layer retrieval-augmented generation (LangChain, LlamaIndex) for Q&A, returning source citations, timestamps, and confidence scores.
5. **Automation & upkeep**
   - Schedule nightly sync jobs for each provider; monitor with logs and alerts.
   - Use checksum manifests to spot drift between local and cloud copies.
   - Recompute embeddings as model quality improves, versioning them by model identifier.

## Privacy, Security, and Compliance Considerations
- Encrypt storage volumes (FileVault on Mac, encrypted archives for backups).
- Manage API keys/secrets via 1Password or macOS Keychain.
- Restrict sharing permissions and enforce MFA across all connected services.
- Pair Time Machine or Backblaze backups with Git/DVC history for the knowledge base to guard against accidental deletions.

## Logic Audit: Blind Spots, Failure Modes, and Mitigations
- **Authentication drift**: OAuth tokens for cloud drives and AI services expire or are revoked, silently halting exports. Add scheduled health checks that raise alerts when sync jobs collect fewer records than expected or fail token refresh.
- **Schema evolution**: Providers change export formats (e.g., new Gemini JSON fields). Maintain versioned schemas with migration scripts; include automated contract tests against sample payloads from each source to surface breaking changes early.
- **Partial conversation capture**: Browser exporters can miss messages if the DOM has not fully loaded. Require explicit fetch via official APIs when possible and log counts of downloaded messages versus API-reported totals to verify completeness.
- **Attachment loss**: Many exports reference files (images, audio) by URL without downloading them. Extend pipelines to detect attachment references, mirror the binaries into object storage, and link via `source_uri` metadata to avoid broken context.
- **Timestamp ambiguity**: Files originating from different time zones or without timezone data skew chronological ordering. Normalize all timestamps to UTC, store original offsets, and surface conflicts during dedupe to prompt manual resolution.
- **Duplicated-but-divergent edits**: Cloud documents often retain revision histories; exporting only the latest version hides intermediate states. When preservation is critical, pull revision metadata (Google Docs revisions API, Dropbox file history) and tag entries with revision IDs before deduping.
- **Privacy regressions**: Enriched knowledge bases can expose sensitive PII if shared with downstream tooling. Enforce data classification tags, redact high-risk entities during embedding, and restrict vector search endpoints behind authentication and audit logs.
- **Scale constraints**: Embedding and indexing large archives can become cost- or latency-prohibitive. Batch embedding jobs, cache frequently accessed vectors, and evaluate local embedding models as fallbacks when API quotas are tight.
- **Disaster recovery gaps**: Knowledge base corruption or accidental deletion can cascade. Implement immutable backups (object storage versioning, offline snapshots) and practice restore drills to verify RPO/RTO targets.

## Evolution & Continuous Improvement Pathways
- **Feedback loop instrumentation**: Collect metrics on query success rates, answer satisfaction, and dedupe skips to prioritize pipeline tuning.
- **Active learning workflows**: Use human-in-the-loop review to confirm entity extraction, tagging accuracy, and Q&A responses, feeding corrections back into retraining datasets.
- **Domain-specific ontologies**: Layer custom taxonomies (projects, clients, research themes) onto the unified schema to improve retrieval relevance and reporting.
- **Federated AI integrations**: Introduce adapters for new AI providers incrementally, encapsulating each in isolated modules with shared validation suites to prevent regressions.
- **Edge-device capture**: Extend collectors to mobile and offline contexts (e.g., Shortcuts automations, local proxies) to minimize gaps between device usage and central ingestion.
- **Zero-trust posture**: Adopt hardware security keys, per-service least-privilege roles, and continuous secret rotation to reduce attack surface as the knowledge base grows in value.

