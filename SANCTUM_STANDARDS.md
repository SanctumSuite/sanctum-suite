# Sanctum Standards

Canonical tech standards for every repo in the Sanctum Suite. Derived from `translachat` and `sanctum-analyst` (ACH), which are currently the most disciplined codebases. Deviations are allowed when justified (see end of doc), but every deviation shortens the suite's shared surface — default to conformance.

This document is the contract. When a standard conflicts with an older file in a repo, the standard wins — the file gets migrated.

---

## 1. The two laws

These are non-negotiable. Every app in the suite follows them.

### 1.1 Never call models directly

All LLM invocations go through **Sanctum Engine's `/task` endpoint**. No `httpx.post("http://localhost:11434/api/chat", …)` in app code. No direct OpenRouter calls. No SDK imports.

Apps consume Engine via the Python client package `sanctum-engine-client`:

```python
from sanctum_engine_client import engine_client

result, latency_ms = await engine_client.run_task(
    task_type="generate_text",
    model_preference="reasoning",   # or "fast" / "translation" / "vision" / "embedding"
    system_prompt="…",
    user_prompt="…",
    max_retries=2,
)
```

The signature is stable across the suite. When Engine grows a new task type (e.g., `rerank`), every app becomes able to use it via a client update, no per-app refactor.

Node/TypeScript apps call Engine over HTTP directly until `@sanctum/engine-client` ships (D6).

### 1.2 Never parse documents directly

All file imports/exports go through **Sanctum Forge**. No `mammoth`, `pymupdf4llm`, `docling`, `python-docx`, or `pdfplumber` imports in app code. Apps accept a file, stream it to Forge's `/import`, and store the returned markdown + blocks as canonical.

```python
from sanctum_forge_client import forge_client

result = await forge_client.import_file(filename, mime, raw)
# result.markdown, result.blocks, result.metadata, result.stats
```

A new format (e.g., EPUB) lands in Forge once and every app gets it.

---

## 2. Stack

| Layer | Choice | Reference |
|---|---|---|
| **Backend runtime** | Python 3.11 + FastAPI + Uvicorn + httpx (async) | `translachat/backend`, `sanctum-analyst/backend`, `sanctum-engine` |
| **ORM** | SQLAlchemy 2 (async mode) + Alembic for migrations | `sanctum-engine`, `translachat/backend` |
| **DB — multi-user apps** | PostgreSQL 16 + pgvector (Docker) | translachat, sanctum-analyst |
| **DB — single-user apps** | localStorage (Zustand persist) + filesystem for content; LanceDB for vector indexes if needed | sanctum-writer (current state) |
| **Frontend** | React 19 + TypeScript + Vite 6 + Tailwind 4 | translachat, sanctum-analyst |
| **State (frontend)** | Zustand v5 with `persist` middleware, single store per app (named `use<App>`) | `translachat/frontend/src/state/store.ts` |
| **Styling** | Tailwind 4 utilities + shared brand colors (see §6). No shadcn / Radix. Lucide for icons. | translachat |
| **Editor** | Monaco (document pane) + CodeMirror 6 (markdown editor) | translachat (Monaco), sanctum-writer (CodeMirror) |
| **CRDT sync** (where collab is a feature) | Yjs + `y-websocket` | translachat |
| **Voice** | Whisper (STT) + Piper (TTS) via the Wyoming protocol | galatea |
| **Package mgr** | `pip` + `requirements.txt` (Python); `npm` + `package-lock.json` tracked (frontend) | all |
| **Container** | Docker + docker-compose, multi-stage builds where the frontend ships with the backend image | translachat's Dockerfile is the reference |

Deviations that Pass 1 does not force to the standard:
- **Consilium**: Node/Express backend. Leave as-is until it migrates.
- **SanctumKanban**: Next.js 15 + Prisma. Leave as-is.
- **SanctumWriter / SanctumWriterPro**: Next.js 14 + embedded Python for docling. D6 work replaces docling with Forge.

These exist; they'll converge over time.

---

## 3. TypeScript

Every TypeScript app uses the same strict baseline. Source: [`translachat/frontend/tsconfig.json`](https://github.com/SanctumSuite/translachat/blob/main/frontend/tsconfig.json).

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "useDefineForClassFields": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

`npx tsc --noEmit` is the first CI gate before any test runs. No PR merges with type errors.

---

## 4. Config

### Python (backends)

`pydantic-settings` with a per-app env prefix. Source: [`translachat/backend/app/config.py`](https://github.com/SanctumSuite/translachat/blob/main/backend/app/config.py).

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="TRANSLACHAT_",   # change per app: GALATEA_, ACH_, CONSILIUM_, etc.
        env_file=".env",
        extra="ignore",
    )

    # Engine (always present, always opt-out-able for dev)
    engine_url: str = "http://localhost:8100"
    engine_enabled: bool = True

    # Forge (present when app handles documents)
    forge_url: str = "http://localhost:8200"
    forge_enabled: bool = True

    # … app-specific settings

settings = Settings()
```

No hard-coded endpoints anywhere in app code. No `process.env.OLLAMA_URL || "http://localhost:11434"` scattered across files.

### TypeScript (frontends)

Vite `import.meta.env.VITE_*`. Frontends never speak to models or Forge directly — they hit their own backend, which handles Engine/Forge.

---

## 5. Repo layout

Target layout. New repos match this from day one; existing repos converge over time.

```
<app-name>/
├── docs/
│   ├── PRD.md
│   ├── ROADMAP.md
│   └── <spec>.md            # any service/protocol spec (e.g. FORGE_SPEC, PHASE0_SPEC)
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── api/             # HTTP endpoints
│   │   ├── services/        # engine_client, forge_client, business logic
│   │   ├── models/          # SQLAlchemy
│   │   └── schemas/         # Pydantic
│   ├── tests/
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── lib/             # api.ts, ws.ts, utils
│   │   ├── pages/
│   │   ├── components/
│   │   └── state/           # Zustand stores
│   ├── index.html
│   ├── package.json
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   └── vite.config.ts
├── shared/                  # optional: cross-language protocol shapes
├── docker/                  # optional: init-db.sql etc.
├── docker-compose.yml
├── .github/workflows/       # CI
├── .env.example
├── CLAUDE.md
├── README.md
└── LICENSE                  # Apache 2.0
```

Backend-only services (Engine, Forge) drop `frontend/`.

---

## 6. Brand + UI

### Colors

Shared brand palette defined once in `tailwind.config.base.ts` at the suite level and extended per-app. TODO in D3 follow-up: pin the final palette — for Pass 1, translachat's current `index.css` custom dark-theme colors are the provisional baseline.

### Icons

[Lucide](https://lucide.dev/) for all icons. No Heroicons, no Tabler, no inline SVGs (except for one-off decorative art).

### Components

No shadcn/ui, no Radix, no MUI. Custom Tailwind utility components live in `frontend/src/components/ui/` per app. When a component gets reused across 2+ apps, promote it to a shared package (D6+).

### State

Zustand v5, single store named `use<App>` (e.g., `useRoom`, `useEditor`). `persist` middleware for anything that should survive a reload. No Redux, no Jotai, no Recoil.

---

## 7. LLM patterns

### Capability names

Apps request models by *capability*, not by name. Engine resolves capability → concrete model via its registry. Capabilities defined in `sanctum-engine`:

| Capability | Meaning | Typical model class |
|---|---|---|
| `fast` | Low-latency, small-context extraction or classification | 3B–8B instruct |
| `reasoning` | Multi-step analysis, complex prompts | 14B–32B instruct |
| `translation` | Text-to-text translation | TranslateGemma, Aya-Expanse, Qwen |
| `vision` | Image → text | llava, qwen-vl |
| `embedding` | Text → vector | nomic-embed-text |
| `ocr` | Image → text with OCR prompting | DeepSeek-OCR, vision models |
| `rerank` | Query + candidates → ranked | rerank-capable model |

Apps should almost never pass `model=` explicitly — doing so opts out of Engine's capability routing.

### Timeouts

- Connect timeout: 10s default.
- Read timeout: 120s default; bump to 180s+ for translation or long-context tasks.
- Never `None` / infinite.

### Retries

Engine handles retries internally (temperature decay, format reminders for JSON). Apps set `max_retries` in the TaskRequest, usually `2` or `3`. Don't wrap Engine calls in additional retry logic — that's what Engine is for.

---

## 8. Testing

- **Backend**: `pytest tests/ -v`. Test at the `services/` layer for business logic; use `pytest-httpx` to mock Engine for unit tests, spin up a real Engine via docker-compose for integration tests.
- **Frontend**: `npx tsc --noEmit` is the required first gate. Vitest tests added as the app stabilizes; no hard coverage floor yet.
- **Integration**: every app's README has a "smoke test" section (curl against `/health`, one round-trip through the primary user flow).
- **Every new API endpoint gets a curl example** in its PR description.

---

## 9. Commit & git hygiene

- Commit messages explain **why**, not just **what**.
- Never commit `.env`, credentials, `node_modules`, or large binaries.
- `package-lock.json` is tracked; `package-lock.json` churn gets its own dep-bump commit.
- Scope commits: feature work, refactors, and dependency bumps go in separate commits.
- Squash-merge PRs; PR title becomes the merge commit title.

---

## 10. CLAUDE.md

Every app repo ships one. 120–250 lines. Source template: [`translachat/CLAUDE.md`](https://github.com/SanctumSuite/translachat/blob/main/CLAUDE.md). Minimum sections:

- **Name + 1-sentence purpose**
- **Key documents** (PRD, roadmap, specs)
- **Architecture** (layers and how they talk)
- **Key principles** (the app-specific version of The Two Laws plus any app-specific rules)
- **Stack** (exact versions)
- **Development practices**: verification after changes, context management, code patterns, git hygiene, testing

---

## 11. License

**Apache 2.0** across every suite repo. Including Engine and Forge (so they can be used by third parties), including apps (so the suite is genuinely open). Use [`LICENSE`](https://www.apache.org/licenses/LICENSE-2.0.txt) verbatim; `NOTICE` file per repo if the repo includes third-party code with attribution requirements.

SanctumWriter's current Polyform Noncommercial 1.0.0 license gets replaced with Apache 2.0 as part of the D3 rollout. If there's commercial protection you want that Apache doesn't offer, raise it before the change lands.

---

## 12. Deviations

A deviation is any repo-local override of a rule above. Record it in the repo's `CLAUDE.md` under a "Deviations from Sanctum Standards" section, with the reason and a target date to converge. Examples we already expect in Pass 1:

- **Consilium**: Node/Express backend. Reason: existing, working, not blocking other work. Target: D6+.
- **SanctumKanban**: Next.js + Prisma. Reason: existing. Target: D6+.
- **SanctumWriter**: CodeMirror instead of Monaco. Reason: different product shape (writing editor vs document viewer). Target: permanent deviation.
- **SanctumWriter**: Embedded Python docling. Reason: will migrate to Forge. Target: D6.

Deviations that are NOT allowed (no target date, just fix):
- Direct LLM calls bypassing Engine.
- Direct document parsing bypassing Forge.
- Hard-coded endpoints in code.
- TypeScript without `strict: true`.
- Commits that include `.env` or secrets.
