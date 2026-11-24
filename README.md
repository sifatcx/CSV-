# AI Data Analyst

An autonomous, full-stack business intelligence application powered by **Gemini Managed Agents** (`antigravity`). Upload one or more CSV datasets (or link large datasets via Google Cloud Storage `gs://` URIs), ask a natural-language business question, and watch an isolated Python agent profile your data, run Pandas/scikit-learn analysis, render charts, and assemble an interactive executive dashboard.

---

## Key Features

- **Flexible Dataset Ingestion**
  - **Inline CSV Upload**: Drag-and-drop or browse multiple related CSV files (up to 1 MB per file) for relational or single-table analysis.
  - **Google Cloud Storage (`gs://`) Support**: Paste a `gs://bucket/object` URI to analyze larger datasets without hitting the inline upload limit, fetched securely inside the sandbox via a static JSON manifest and optional allowlisted server proxy.
- **Autonomous Python Analysis Sandbox**
  - Executes inside a remote Managed Agents sandbox with an explicit agent workflow (`Explore -> Analyze -> Visualize -> Report`).
  - Streams live Server-Sent Events (SSE) for agent thinking, code execution tool calls, outputs, and stage indicators.
- **Interactive Executive Dashboard**
  - **Overview**: Top KPI cards, AI executive summary, featured charts, and priority actions.
  - **Visualizations**: Full gallery of Matplotlib PNG charts (`bar`, `barh`, `line`, `scatter`, `pie`, `heatmap`) with lightbox zoom.
  - **Data Tables**: Searchable, scrollable supporting tables extracted from `data/analysis/*.csv`.
  - **Insights & Actions**: Ranked metrics and strategic recommendations.
  - **Export & Print**: One-click multi-page PDF export (`jspdf` + `jspdf-autotable`), raw JSON report download, and print preview.
- **Conversational Follow-Ups**
  - Reuses the active sandbox environment (`environmentId`) so follow-up questions build directly on the staged CSVs and `profile.json` without re-uploading data.

---

## Tech Stack

- **Frontend**: React 19, TypeScript, Vite 6, Tailwind CSS v4, Motion (`motion/react`), Lucide Icons, `react-markdown` + `remark-gfm`, `jspdf`, `jspdf-autotable`
- **Backend**: Node.js, Express 4 (`server.ts` with Vite middleware in development), Multer, SSE streaming
- **Agent & Analytics Runtime**: Gemini Managed Agents Interactions API (`v1beta/interactions`), Python 3 (`pandas`, `numpy`, `matplotlib`, optional `scikit-learn`)

---

## Project Structure

```text
.
├── agent/                          # Files mounted into /.agents inside the remote sandbox
│   ├── AGENTS.md                   # Agent workflow, output contract, and edge-case rules
│   ├── agent.yaml                  # Agent definition and example prompts
│   ├── download_gcs.py             # Credential-free GCS downloader using gcs_manifest.json
│   ├── requirements.txt            # Core Python packages (pandas, numpy, matplotlib)
│   └── skills/
│       ├── data-explorer/SKILL.md  # Profiling instructions -> data/profile.json
│       ├── python-data/SKILL.md    # Pandas / scikit-learn analysis -> data/analysis/*.csv
│       ├── visualization/          # Matplotlib chart generator -> charts/*.png
│       │   ├── SKILL.md
│       │   └── scripts/make_chart.py
│       └── reporting/              # Report compiler -> data/report.json
│           ├── SKILL.md
│           └── scripts/build_report.py
├── server/
│   └── lib/
│       ├── agentClient.ts          # Gemini Managed Agents REST & SSE stream client
│       ├── gcsSources.ts           # gs:// URI validation and manifest serialization
│       ├── gcsSources.test.ts      # Unit tests for GCS URI parsing and allowlist rules
│       └── jsonExtractor.ts        # JSON block extraction helper
├── src/
│   ├── App.tsx                     # Main workspace UI, SSE client, chat/logs, dashboard, PDF export
│   ├── index.css                   # Tailwind CSS v4 theme and utilities
│   ├── main.tsx                    # React entry point
│   ├── types.ts                    # Shared TypeScript interfaces
│   └── lib/firestore.ts            # Optional client utilities
├── server.ts                       # Express server, API routes, tar extraction, Vite dev/prod serving
├── index.html                      # App shell
├── metadata.json                   # Applet metadata & capabilities
├── package.json                    # Scripts and dependencies
├── tsconfig.json                   # TypeScript configuration
└── vite.config.ts                  # Vite configuration
```

---

## Getting Started

### Prerequisites

- **Node.js** (v20+ recommended) and **npm**
- A **Gemini API key** (`GEMINI_API_KEY`) with access to the Gemini Interactions / Managed Agents API

### 1. Install Dependencies

```bash
npm install
```

### 2. Configure Environment Variables

Create a `.env` or `.env.local` file in the project root (note that `.env.local` takes precedence over `.env`):

```env
# Required: Gemini API key used by the server to call the Interactions API
GEMINI_API_KEY=your_gemini_api_key_here

# Optional: Comma-separated GCS buckets permitted for gs:// inputs and server-side proxy downloads
# GCS_ALLOWED_BUCKETS=my-public-or-service-bucket,another-bucket

# Optional: Public base URL of this server used by the sandbox GCS download proxy
# PUBLIC_SERVER_URL=https://your-server-domain.example.com

# Optional: Daily per-user quota limit in production
# DAILY_QUOTA_LIMIT=999999
```

### 3. Run in Development

Starts the Express server with Vite middleware on port `3000`:

```bash
npm run dev
```

Open `http://localhost:3000` in your browser.

### 4. Build & Run in Production

```bash
npm run build
npm start
```

- `npm run build` compiles the Vite frontend into `dist/` and bundles `server.ts` into `dist/server.cjs` via `esbuild`.
- `npm start` runs `node dist/server.cjs`.

---

## Available Scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Start Express + Vite development server (`tsx server.ts`) |
| `npm run build` | Build frontend assets and bundle `dist/server.cjs` |
| `npm start` | Start the production Node server (`dist/server.cjs`) |
| `npm run lint` | Run TypeScript type-checking (`tsc --noEmit`) |
| `npm test` | Run backend unit tests (`node --test server/lib/gcsSources.test.ts`) |
| `npm run clean` | Remove the `dist/` build directory |

---

## API Endpoints

| Method & Path | Description |
| --- | --- |
| `GET /api/health` | Health check (`{ status: "ok" }`) |
| `POST /api/upload` | Accepts multipart CSV upload (`file`, max 1 MB inline) and returns file metadata + UTF-8 content |
| `POST /api/analyze` | Starts or continues an SSE-streamed Managed Agents analysis (`question`, `files`, `datasetName`, `generationId`, `environmentId`) |
| `POST /api/cancel-show` / `POST /api/abort/:generationId` | Aborts an active analysis stream owned by the requesting `x-client-id` |
| `GET /api/download-file` | Streams an allowlisted `gsUri` object through the server using GCP metadata credentials (never exposed to the sandbox) |
| `GET /output/:runId/charts/:chartFile` | Serves generated PNG/JPEG chart artifacts extracted from the sandbox snapshot |
| `GET /api/quota` | Returns daily usage and quota limit |
| `POST /api/clear-files` | Resets session file state |

---

## How the Analysis Pipeline Works

1. **Staging**: `POST /api/analyze` packages `agent/` scripts/skills and inline CSVs (or a `gcs_manifest.json` for `gs://` URIs) into `/.agents` inside a remote sandbox environment.
2. **Explore**: The agent copies CSVs to `./workspace/data/`, installs Python dependencies (`pandas`, `numpy`, `matplotlib`), and writes `./workspace/data/profile.json`.
3. **Analyze**: The agent joins/cleans tables with Pandas (and `scikit-learn` when modeling is warranted) and writes intermediate results to `./workspace/data/analysis/*.csv`.
4. **Visualize**: `make_chart.py` renders headless Matplotlib charts to `./workspace/charts/*.png`.
5. **Report & Snapshot Retrieval**: `build_report.py` compiles `./workspace/data/report.json`. The Express server downloads the environment tar archive in memory, persists chart images under `output/<runId>/charts/`, and emits a `report_data` SSE event to render the dashboard.
6. **Automatic Cleanup**: Artifacts in `output/` older than 24 hours are automatically pruned on server startup and new analysis requests.
