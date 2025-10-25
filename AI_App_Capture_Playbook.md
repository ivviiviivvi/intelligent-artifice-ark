# AI App Capture Playbook

This playbook standardizes how to continuously capture transcripts from key AI chat platforms and route them into the unified knowledge base. Each section covers device preparation, export actions, Shortcut automations (for Apple devices), and ingestion hooks so that every conversation is captured with minimal friction.

## Common Device Preparation

1. **Browser Profiles:** Create dedicated browser profiles (or standalone browsers such as Arc or Sidekick) for work and personal use to isolate cookies and extension permissions.
2. **Clipboard Managers:** Install a clipboard manager (e.g., Raycast, Alfred, Paste) and enable history sync so copied transcripts can be recovered during automation failures.
3. **File Sync:** Configure a synced folder (iCloud Drive → `AI Captures/Inbox/`) that Shortcuts and automation scripts can write to. This folder is the staging area that ingestion hooks monitor.
4. **Extensions:** Install the ChatGPT Exporter userscript or extension in supported browsers to simplify multi-format exports.
5. **Shortcuts Permissions:** On iOS/iPadOS/macOS, enable "Allow Running Scripts" and grant file access to the `AI Captures` directory.

## ChatGPT (chatgpt.com)

- **Export Action:** Use ChatGPT Exporter to download Markdown + JSON for each conversation. For manual capture, select the conversation → click the exporter button → choose **Markdown + Metadata**.
- **Shortcut Automation:**
  - Trigger: Share Sheet → "Export to Knowledge Base" shortcut.
  - Actions: Get contents of shared URL → call ChatGPT Exporter API → save Markdown file to `AI Captures/Inbox/chatgpt/` using `${date}-${slug}.md` naming.
- **Ingestion Hook:** `fswatch` (macOS) or `watchexec` monitors the ChatGPT inbox folder and invokes `ingest_chatgpt.py` to normalize speaker tags and push to the knowledge base via REST.

## Claude (claude.ai)

- **Export Action:** From the conversation menu, select **Export → Markdown**. If unavailable, copy the thread using the "Copy conversation" action.
- **Shortcut Automation:**
  - Trigger: Keyboard Maestro macro (`⌃⌥⌘C`).
  - Actions: Execute JavaScript in frontmost browser tab to call `window.__cl_export()` (custom snippet) → save output to `AI Captures/Inbox/claude/Claude-${timestamp}.md`.
- **Ingestion Hook:** A cron job runs `claude_sync.sh` every 15 minutes, converting Markdown to structured JSON and posting to `/ingest/claude` endpoint.

## Gemini (gemini.google.com)

- **Export Action:** Use "Download transcript" from the overflow menu. Ensure "Include sources" is checked.
- **Shortcut Automation:**
  - Trigger: macOS Shortcut "Gemini Capture" via menu bar.
  - Actions: Use `Run JavaScript on Web Page` to call the internal export endpoint → store JSON + Markdown bundle in `AI Captures/Inbox/gemini/`.
- **Ingestion Hook:** Google Drive API watch on the Gemini inbox folder triggers a Cloud Function that calls `gemini_ingest` pipeline, mapping Google citations into the unified schema.

## Perplexity (perplexity.ai)

- **Export Action:** Click **Share** → **Copy Markdown**. Paste into a new file named `Perplexity-${date}.md`.
- **Shortcut Automation:**
  - Trigger: macOS Services menu item "Capture from Perplexity".
  - Actions: Use AppleScript to read clipboard contents, append metadata (query, model, follow-up) → save to `AI Captures/Inbox/perplexity/`.
- **Ingestion Hook:** `perplexity_ingest.ts` parses Markdown, extracts citations, and writes both the conversation and citation graph into the knowledge base.

## Microsoft 365 Copilot (copilot.microsoft.com & M365 apps)

- **Export Action:** In Copilot chat panes, use **Copy** → **Include formatting**. For Outlook/Teams, use the built-in "Export conversation" option.
- **Shortcut Automation:**
  - Trigger: Power Automate flow "Copilot Transcript Capture" initiated by manual button.
  - Actions: Save exported HTML to OneDrive `AI Captures/Inbox/copilot/` → call Azure Function to convert HTML to Markdown.
- **Ingestion Hook:** Azure Function queues normalized transcripts into Service Bus, where the ingestion service polls and merges into the knowledge base with channel tags (`outlook`, `teams`, `edge`).

## Grok (x.com)

- **Export Action:** Use the conversation menu to select **Download transcript**. If missing, copy conversation via developer console script `copyGrokThread()`.
- **Shortcut Automation:**
  - Trigger: iOS Shortcut from the X Share Sheet.
  - Actions: Fetch conversation JSON via X API → convert to Markdown → save to `AI Captures/Inbox/grok/`.
- **Ingestion Hook:** `grok_ingest.py` runs on the ingestion server, enriching posts with referenced tweets and threading context before storing.

## Continuous Ingestion Checklist

- Validate that each inbox folder is included in the watch service configuration.
- Ensure automation credentials (API keys, cookies, auth tokens) are stored in 1Password and referenced via environment variables or Shortcuts secrets.
- Review ingestion logs daily for failures and re-run the relevant automation if errors occur.
- Archive processed files to `AI Captures/Archive/${app}/${year}/` after successful ingestion to prevent re-processing.

## Troubleshooting

| Symptom | Root Cause | Resolution |
| --- | --- | --- |
| Files stuck in inbox folders | Watch service not running | Restart `ingestion-watcher` service and verify logs |
| Shortcut prompts for permissions each run | Missing "Allow Running Scripts" | Enable in Shortcuts → Settings |
| Markdown missing assistant labels | Export action omitted metadata | Re-run export ensuring metadata option selected or patch via `repair_transcript.py` |
| Duplicate entries in knowledge base | File not archived post-ingestion | Move processed files to Archive and re-run dedupe script |

Keeping these automations aligned across all AI chat platforms ensures every transcript is captured, normalized, and searchable within the unified knowledge base.

## Data Preservation & Deduplicated Compilation

To make sure every captured transcript is preserved without redundancy, pair this playbook with the **PRC01 — Deduplicated Compile Process**:

1. **Ingest & Snapshot:** Archive raw exports verbatim to create a Canonical Full Snapshot before any editing.
2. **Tokenize & Tag:** Break snapshots into uniquely identified units (AA01, BB02) using code fences so they can be cross-referenced later.
3. **Dedupe & Merge:** Collapse overlapping units while retaining source trails and optional flags (`GEN+`, `USER_ORIG`, `CROSSREF`).
4. **Fuse & Comment:** Re-sequence deduped units into a Fused Iterative Document, optionally adding comparisons and commentary for version history.
5. **Export & Index:** Publish the fused output alongside a unit index so downstream systems (Notion, Obsidian, internal wikis) can link back to the originals.

Ready-to-use prompts and detailed instructions live in `PRC01_Deduplicated_Compile_Process.md`. Run that workflow whenever you ingest large batches or need an audit-ready knowledge base snapshot.
