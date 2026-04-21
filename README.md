# Sanctum Suite

**Privacy-first, local-first AI productivity suite.** Every app runs on your own hardware. Your documents, conversations, voice, and analytical work never leave your machine unless you explicitly opt in to a cloud model.

---

## Why this exists

Cloud AI is fast and capable — and it's also surveillance, lock-in, latency you can't control, and a subscription that stops working when the vendor's incentives change. The Sanctum Suite is a bet that the same tools can be built on **local models** (via Ollama, LM Studio, or any OpenAI-compatible endpoint you host), with cloud models as an optional runtime you reach for when you want to, not when you're forced to.

The suite is a set of focused apps (writing, chat, voice, analysis, project work, model comparison) that share a common foundation: a local model service and a document conversion service. You can run one app or all of them; they work independently and compose through stable HTTP contracts.

---

## What's in the suite

### End-user apps

| App | Purpose | Repo |
|---|---|---|
| **SanctumWriter** | Local-first markdown editor with AI writing companion (Ollama / LM Studio) | [`sanctum-writer`](https://github.com/SanctumSuite/sanctum-writer) |
| **SanctumWriterPro** | AI markdown editor with frontier models (OpenRouter: GPT, Claude, Gemini, Grok, …) | [`sanctum-writer-pro`](https://github.com/SanctumSuite/sanctum-writer-pro) |
| **Consilium** | Council of AIs — query multiple models simultaneously, compare responses side-by-side | [`consilium`](https://github.com/SanctumSuite/consilium) |
| **Galatea** | Local voice + vision AI companion — privacy-first, runs entirely on your machine | [`galatea`](https://github.com/SanctumSuite/galatea) |
| **translachat** | Bilingual chat + collaborative document workspace (Yjs CRDT, bilingual block storage) | [`translachat`](https://github.com/SanctumSuite/translachat) |
| **Sanctum Analyst (ACH)** | Local-first intelligence analysis using Structured Analytic Techniques (ACH matrix, premortem, red hat, devils advocate, starbursting, argument mapping, …) | [`sanctum-analyst`](https://github.com/SanctumSuite/sanctum-analyst) |
| **ProcessPulse** | AI writing-process analysis for educators — analyzes student thinking, not just output | [`processpulse`](https://github.com/SanctumSuite/processpulse) |
| **SanctumKanban** | Self-hosted kanban for small teams — real-time, drag-drop, heat-map overview | [`sanctum-kanban`](https://github.com/SanctumSuite/sanctum-kanban) |

### Shared services

| Service | Purpose | Repo |
|---|---|---|
| **Sanctum Engine** | Local LLM service layer. All model calls go through here: task-oriented API (`extract_json` / `generate_text` / `embed` / `vision` / `translate` / `rerank`), capability-based model routing, context budgeting + chunking, JSON output validation + auto-repair, retry with temperature decay, Postgres-backed model registry and task log. Runtimes: Ollama, OpenRouter. | [`sanctum-engine`](https://github.com/SanctumSuite/sanctum-engine) |
| **Sanctum Forge** | Document conversion service. File formats in → markdown + block-structured JSON out. Handles PDF (pymupdf4llm), DOCX (mammoth), HTML (html2text), plain text, images (OCR via local vision models). | [`sanctum-forge`](https://github.com/SanctumSuite/sanctum-forge) |

### Meta

| | | |
|---|---|---|
| **sanctum-suite** | This repo. Mission, standards, roadmap, architecture. | [`sanctum-suite`](https://github.com/SanctumSuite/sanctum-suite) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          End-user apps                              │
│                                                                     │
│   Writer    WriterPro    Consilium    Galatea    translachat        │
│   Analyst   ProcessPulse  Kanban                                    │
│                                                                     │
└──────────────┬──────────────────────────────┬───────────────────────┘
               │                              │
               │  HTTP                        │  HTTP
               │                              │
       ┌───────▼────────┐           ┌─────────▼────────┐
       │ Sanctum Engine │           │  Sanctum Forge   │
       │   (port 8100)  │           │   (port 8200)    │
       │                │           │                  │
       │  /task         │           │  /import         │
       │  /models       │           │  /export         │
       │  /health       │           │  /health         │
       └───────┬────────┘           └──────────────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼───┐ ┌────▼───┐ ┌────▼─────────┐
│ Ollama│ │OpenRtr │ │OpenAI-compat │
└───────┘ └────────┘ └──────────────┘
    (local)   (cloud)    (your own server, e.g. LM Studio, vLLM)
```

**The two rules every app follows:**

1. **Never call models directly.** Every LLM invocation goes through Sanctum Engine's `/task` endpoint. That's where retry, validation, context budgeting, and fallback live.
2. **Never parse documents directly.** Every file import/export goes through Sanctum Forge. Apps store the returned markdown as canonical.

These two rules are what make the suite a suite and not a pile of independent apps. They also make every app instantly better as Engine and Forge improve — a new OpenRouter runtime, a better JSON repair, a new document format lands once and lifts all the apps.

---

## Installation

Each app ships a `docker-compose.yml` with its dependencies. To run the full suite locally:

```bash
# Clone the apps you want
git clone https://github.com/SanctumSuite/sanctum-engine
git clone https://github.com/SanctumSuite/sanctum-forge
git clone https://github.com/SanctumSuite/translachat   # or any other app

# Engine and Forge run once; apps talk to them
cd sanctum-engine && docker compose up -d
cd ../sanctum-forge && docker compose up -d
cd ../translachat && docker compose up -d
```

Each app's README has its own quickstart with the env vars it expects. Apps point at Engine and Forge via `ENGINE_URL` / `FORGE_URL` env vars (defaults to `http://localhost:8100` / `http://localhost:8200`).

---

## Contributing

- Architecture and stack conventions: [`SANCTUM_STANDARDS.md`](SANCTUM_STANDARDS.md)
- Every app ships a `CLAUDE.md` with per-repo conventions.
- Engine API contract: [`sanctum-engine/docs/PHASE0_SPEC.md`](https://github.com/SanctumSuite/sanctum-engine/blob/main/docs/PHASE0_SPEC.md)
- Forge API contract: [`sanctum-forge/docs/FORGE_SPEC.md`](https://github.com/SanctumSuite/sanctum-forge/blob/main/docs/FORGE_SPEC.md)

---

## License

Apache 2.0 across the suite — see [`LICENSE`](LICENSE).
