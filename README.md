# 🧠 Unified Memory Hub

A unified local/cloud memory architecture for developers and AI agents. It serves as a persistent cross-harness semantic layer where your coding models can refer to past handoff documents, read session summaries, query project preferences, and synchronize with Firebase Firestore (dedicated `memory-hub` project) and Supermemory.ai.

---

## 🚀 Key Features

*   **Offline-First Local DB**: Stores vector embeddings and facts locally in `.memory/db.json` with zero-latency in-memory cosine similarity matching.
*   **2-Layer Deduplication**: Normalized content hash match + cosine similarity (default threshold 0.93) catches both exact and paraphrased duplicates during ingest.
*   **Dual-Cloud Sync**:
    *   **Firebase Firestore**: Dedicated `memory-hub` project (separate from `joint-ai-chat`). Bi-directional sync with undefined-field stripping for Firestore compatibility.
    *   **Supermemory.ai**: Syncs local memories to the Supermemory documentation engine using their REST API.
*   **Document Ingestion**: Recursively parses folders of Markdown files (`.md`) and logs (`.txt`), applying semantic chunking to preserve context. Auto-excludes builds, caches, vendor trees, IDE/agent dotdirs, data directories, and more.
*   **Robust CLI (`mem`)**: Query context, insert facts, manage agent-contributed shared memories, or sync databases instantly via command line.
*   **Launchd Watcher Daemon** (macOS): Auto-starts on login, restarts on crash, watches your `/Projects` tree for canonical markdown files and auto-ingests them.
*   **Vibrant Web Dashboard**: A glassmorphic dark-themed visual console built with Vite and Vanilla JS/CSS to manage, search, write summaries, and browse memory tags.

---

## 🛠️ Installation & Setup

1.  **Install Dependencies**:
    ```bash
    npm install
    ```

2.  **Environment Variables** (in `.env.local`):
    *   `VITE_GEMINI_API_KEY`: Primary embedding API — generates embeddings using `gemini-embedding-2` by default.
    *   `OPENAI_API_KEY`: Fallback embedding API — uses `text-embedding-3-large` (3072-dim) when Gemini is unavailable. Ensures queries still work during Gemini outages.
    *   `MEMORY_EMBEDDING_MODEL`: Optional primary embedding-model override.
    *   `MEMORY_FALLBACK_EMBEDDING_MODEL`: Optional Gemini fallback model (defaults to `gemini-embedding-001`).
    *   `OPENAI_EMBEDDING_MODEL`: Optional OpenAI fallback model override (defaults to `text-embedding-3-large`).
    *   `MEMORY_DEDUP_THRESHOLD`: Optional near-dup cosine threshold for ingest (default 0.93).
    *   `VITE_FIREBASE_API_KEY` & Firestore configs: Connects to the **`memory-hub`** Firestore project.
    *   `SUPERMEMORY_API_KEY` (Optional): Syncs memories to your Supermemory.ai account.

3.  **Link the Command globally**:
    ```bash
    npm link
    ```
    This registers the `mem` CLI command globally on your machine, so you can execute it from any folder.

4.  **Install the Launchd Watcher** (one-time):
    ```bash
    cp /path/to/com.memory.watchd.plist ~/Library/LaunchAgents/
    launchctl load ~/Library/LaunchAgents/com.memory.watchd.plist
    ```
    The watcher now auto-starts on login and restarts on crash. **Do not start it manually via `npm run watch`.**

---

## 💻 CLI Commands (`mem`)

Once linked, run `mem` (or `node bin/mem.js`) from any project directory.

### View Stats
```bash
mem stats
```

### Semantic Search
```bash
mem query "what are the styling preferences for playgrounds?" --limit 3 --minScore 0.35
```
Queries can be scoped and returned as compact JSON:
```bash
mem query "morning routine" \
  --source wearabllm-home-assistant \
  --tags personal,preference \
  --tag-mode any \
  --limit 3 \
  --json
```

### Ingest Documents or Folders
```bash
mem ingest ./docs/handoffs/ --tags handoff,ducati
mem ingest . --not AGENT_JOURNAL,journal,corpus   # skip specific dirs
```

**Auto-excluded** (always, no flag needed): `.git`, `node_modules`, `dist`, `build`, `venv`, `__pycache__`, `coverage`, `tmp`/`temp`, `.cache`, `.env*`, lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`), `android`/`ios` build roots, IDE/agent dotdirs (`.idea`, `.vscode`, `.cursor`, `.hermes`, `.claude`), `archive`, `test-results`, `screenshots`, `corpus`, `raw`, `data`, `datasets`, `functions`, `vendor`, `target`.

### Manually Insert Facts
```bash
mem add "We prefer native <dialog> over custom modals for accessibility reasons" --tags preferences,dialog,a11y
```

### Upsert, Update, Delete
Assistant integrations should use a stable key so corrections replace an existing logical fact instead of creating duplicates. Personal assistant memories should be `--local-only` until remote deletion is implemented:

```bash
mem upsert "Corina prefers concise spoken morning summaries." \
  --key morning-summary \
  --source wearabllm-home-assistant \
  --tags personal,preference \
  --local-only \
  --json

mem update <memory-id> --tags personal,preference,voice --local-only true --json
mem delete <memory-id> --json
```

### Sync Memory (Local + Firestore + Supermemory)
```bash
mem sync
```

### List Agent Contributions (Shared Memories)
```bash
mem shared                    # all shared memories
mem shared --source hermes    # filter by author
mem shared --limit 50
```

### List Recent Memories
```bash
mem list --limit 10
```

### Health Check
```bash
mem health              # human-readable output
mem health --json       # machine-readable (for cron jobs)
```
Checks embedding API connectivity (Gemini + fallback), Firestore reachability, embedding cache hit rate, and launchd watcher status. Returns exit code 1 on any failure so you can wire it into alerts.

---

## 🤝 Cross-Agent Knowledge Sharing

Agents (Hermes, Claude, Codex, Grok, Laguna, Gemini) are encouraged to contribute useful facts to the shared pool. This way knowledge compounds across work sessions and agent boundaries.

```bash
mem add "User prefers concise terminal output when debugging" --tags shared,preference --source hermes
mem upsert "BRICK daemon requires localhost:7373 in CSP" --key brick-csp --tags shared,architecture --source hermes
```

**Conventions**:
*   **Source field** = lowercase agent name (`hermes`, `claude`, `codex`, `grok`, `laguna`, `gemini`).
*   **Tags** = always include `shared` plus topic tags.
*   Use `upsert` for evolving facts so old versions get replaced, not duplicated.
*   Use `add` for one-off facts.

**What NOT to share**:
*   Personal/session-specific progress ("I fixed bug X in PR Y" — stale in 7 days)
*   Journal content (journals are personal and subjective)
*   Secrets, keys, credentials
*   Anything the user explicitly marked as private

---

## 🖥️ Web Dashboard

To start just the visual web dashboard (the watcher runs separately via launchd):
```bash
npm run dev
```
Open **`http://localhost:5173`** in your browser.

### Dashboard Highlights
*   **Interactive Semantic Search**: Instantly query memories with custom match thresholds and similarity percentage ratings.
*   **Template Generators**: Handoff template and session summary layouts.
*   **Tag Cloud Explorer**: Filter your database in real-time by clicking active tag chips.

---

## 👀 Launchd Watcher Daemon

A launchd-managed service at `~/Library/LaunchAgents/com.memory.watchd.plist` watches `/Users/corinakaiser/Projects` for changes to canonical markdown files and auto-ingests them.

| Action | Command |
|--------|---------|
| Check status | `launchctl list \| grep memory` |
| Stop | `launchctl unload ~/Library/LaunchAgents/com.memory.watchd.plist` |
| Start | `launchctl load ~/Library/LaunchAgents/com.memory.watchd.plist` |
| Logs | `tail -f ~/Library/Logs/memory-watchd.log` |
| Errors | `tail -f ~/Library/Logs/memory-watchd-error.log` |

**Auto-ingested file patterns**:
`HANDOFF.md`, `SPEC.md`, `BRAINSTORM.md`, `SUMMARY.md`, `COORDINATION.md`, `SESSION_SUMMARIES.md`, `NEXT_STEPS.md`, `UPDATES.md`, `ANALYSIS_PLAN.md`, `TRAINING_PLAN.md`, `RUNLOG.md`, `CHANGELOG.md`, `MEMORY.md`, `HOW_TO.md`, `THEME_GUIDE.md`, `COMPONENT_STRATEGY.md`, and `*_IDEAS.md` files.

**Restart required** when you change `.env.local` (API keys, embedding models).

---

## 🤖 Cross-Harness Integration (Piping to LLMs)

Since this memory uses a standard JSON database and CLI query tool, you can feed query context directly to any CLI coding model:

### Example: Supplying Context to an Agent
```bash
claude "Review this file, keeping in mind the project design rules: $(mem query 'styling preferences' --limit 2)"
```

### Example: Seed Agent Context on Boot
```bash
mem query "last session handoff" --limit 1 > .agent_context.md
```
Then configure your active model runner to read `.agent_context.md` on boot.

---

## 🔧 Local API (Development Only)

The dashboard's localhost Vite API exposes the same operations the CLI uses:

- `POST /api/memories` — add or keyed-upsert a memory
- `PATCH /api/memories/:id` — update content, source, tags, metadata, key, or `localOnly`
- `DELETE /api/memories/:id` — delete a local memory
- `POST /api/query` — semantic search: accepts `query`, `limit`, `minScore`, `tags`, `tagMode`, `source`, `sources`, and `includeLocalOnly`
- `POST /api/sync` — trigger a Firestore sync run
- `GET /api/config` — status of active API key configs

The Vite API is a localhost development service only — do not expose it directly to Home Assistant or the public network. Add a dedicated token-authenticated service first.
