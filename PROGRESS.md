# Sanctum Suite — Standardization Progress

Living document tracking the in-flight standardization of the Sanctum Suite. Updated as phases land. When starting a session ("pick up where we left off"), read this first.

**Last updated:** 2026-04-22

---

## The goal

Take the suite of independent apps the user built over 2026 — Consilium, Galatea, SanctumWriter/Pro, SanctumKanban, translachat, ACH (Sanctum Analyst), ProcessPulse — and standardize them around two shared HTTP services (**Sanctum Engine**, **Sanctum Forge**) so every app stops re-implementing Ollama/OpenRouter clients, document parsers, and model registries.

**The two laws** that enforce it (full text in [`SANCTUM_STANDARDS.md`](SANCTUM_STANDARDS.md)):
1. Never call models directly — every LLM call goes through Engine's `/task` or `/task/stream`.
2. Never parse documents directly — every file goes through Forge's `/import`.

---

## Where we are

| Phase | Status | What's done |
|---|---|---|
| **Pass 1** — audit + standards + one POC | ✅ Complete | Meta repo, standards doc, Engine extracted from ACH, Python client, ProcessPulse opt-in shim |
| **Pass 2.0** — validate Pass 1 with live smoke test | ✅ Complete | Engine docker-compose up, `/health`, `/task` generate_text + extract_json, `/task/stream` NDJSON, Python client install-from-GitHub + live call, all green |
| **Pass 2.1a** — dedupe ACH's Python client | ✅ Pushed to branch | `lafintiger/ACH#sanctum-engine-migration` — deletes ACH/engine/ (23 files), wrapper over published client, compose updated, Apache 2.0 license added, postgres port 5434 |
| **Pass 2.1b** — dedupe translachat's client | ✅ Pushed to branch | `lafintiger/translachat#sanctum-engine-dedupe` — 75 LOC → 45 LOC wrapper; user's WIP files (docker-compose, forge, frontend) untouched |
| **Pass 2.1c** — salvage llm-counsil's parallel-query | ✅ Pushed to main | Published client 0.1.2 → `run_tasks_parallel(task_specs)` ports llm-counsil's pattern; llm-counsil can be archived |
| **Pass 2.2** — extract sanctum-forge | ✅ Published | `SanctumSuite/sanctum-forge` subtree-split from `translachat/forge/`, history preserved. Python client package. FORGE_HOST_PORT env for port conflict with translachat's in-tree copy |
| **Pass 2.3** — TS client | ✅ Published | `SanctumSuite/sanctum-engine-client-ts` — strict TypeScript, zero runtime deps, ships via `prepare` script compile on install |
| **Pass 2.4** — Engine streaming + extensions | ✅ Pushed to main | `POST /task/stream` (NDJSON), `TaskRequest.messages[]` for multi-turn, `TaskRequest.runtime_api_key` for BYO-key apps. Both clients bumped to 0.3.0 with streaming + messages + runtime_api_key |
| **Pass 2.5a** — Consilium migration | ⚠️ POC pushed | `lafintiger/Consilium#sanctum-engine-migration` — `streamEngineChat` helper + compare-mode flow (1 of 9 call sites). 8 remaining (evaluate, consensus, 2×council rounds, 2×chat variant, synthesis) share the pattern. Also a healthcheck IPv4 fix hit this branch (orthogonal) |
| **Pass 2.5b** — Galatea migration | ✅ Pushed to branch | `lafintiger/galatea#sanctum-engine-migration` — OllamaService.chat_stream opt-in via `GALATEA_ENGINE_ENABLED`, direct-Ollama fallback on any Engine error. Voice/search handlers unchanged |
| **Pass 2.5c** — SanctumWriter + SanctumWriterPro | ⬜ Not started | Biggest remaining migration. Needs Engine for writing + Forge for doc imports (replace embedded Python docling). Not cloned locally yet |
| **Pass 2.6** — shared UI tokens (`@sanctum/ui-tokens`) | ⬜ Not started | Brand palette, Lucide baseline, shared Tailwind config |
| **Pass 2.7** — repo consolidation | ⬜ Not started | Transfer ACH / translachat / Consilium / Galatea / ProcessPulse / SanctumWriter / SanctumWriterPro / SanctumKanban to `SanctumSuite` org; archive lafintiger/theaihorizon mirrors |

---

## Feature branches awaiting review

All five default `engine_enabled=false`, so merging any of them is behavior-preserving until the user flips the flag in `.env`.

| App | Branch | Notable changes |
|---|---|---|
| [lafintiger/ACH](https://github.com/lafintiger/ACH/tree/sanctum-engine-migration) | `sanctum-engine-migration` | −23 files (`engine/`), +LICENSE, engine_client wrapper, compose reworked for external Engine, postgres port 5434 |
| [lafintiger/translachat](https://github.com/lafintiger/translachat/tree/sanctum-engine-dedupe) | `sanctum-engine-dedupe` | −30 LOC; in-tree `engine_client.py` becomes a thin wrapper; WIP files untouched |
| [lafintiger/processpulse](https://github.com/lafintiger/processpulse/tree/sanctum-engine-integration) | `sanctum-engine-integration` | +144 LOC; opt-in shim in `OllamaClient.generate` routes JSON-mode to Engine's extract_json |
| [lafintiger/Consilium](https://github.com/lafintiger/Consilium/tree/sanctum-engine-migration) | `sanctum-engine-migration` | −53 net LOC in `chat.js`; `streamEngineChat` helper covering compare-mode; also `frontend/Dockerfile` healthcheck IPv4 fix |
| [lafintiger/galatea](https://github.com/lafintiger/galatea/tree/sanctum-engine-migration) | `sanctum-engine-migration` | +71 LOC; `chat_stream` branches on flag; direct Ollama unchanged |

Recommended review order if validating incrementally:
1. **translachat** — smallest diff, highest-density test of the wrapper pattern (~5 min)
2. **ProcessPulse** — the original POC, never end-to-end tested (~10 min)
3. **Galatea** — validates streaming end-to-end (~15 min)
4. **Consilium** — TS client install + BYO-key passthrough + streaming (~15 min)
5. **ACH** — most comprehensive client API coverage (~20 min)

---

## Shared-service API surface (as of latest main)

### sanctum-engine

| Endpoint | Purpose |
|---|---|
| `GET /health` | Status, ollama connectivity, model list, 24h stats |
| `GET /models` | Registered models with capabilities, context limits |
| `GET /models/{name}/stats` | Per-model performance metrics |
| `POST /task` | Single-shot task (`generate_text`, `extract_json`, `embed`, `vision`, `translate`, `rerank`). Retry + validation. `messages[]` for multi-turn. `runtime_api_key` for BYO-key. |
| `POST /task/stream` | NDJSON stream — `{delta}`, `{done,meta}`, `{error}`. Only `generate_text` |
| `POST /task/embed` | Batch embeddings (Ollama only) |

### sanctum-forge

| Endpoint | Purpose |
|---|---|
| `GET /health` | Status + backend versions |
| `GET /formats` | Supported import/export formats |
| `POST /import` | File → `{markdown, blocks, metadata, stats}`. PDF, DOCX, HTML, txt, images (OCR via local vision models) |

### Clients (same shape in Python + TS, co-versioned)

| Function | What it does |
|---|---|
| `run_task` / `runTask` | Single-shot task; supports `on_complete` meta callback |
| `run_task_stream` / `runTaskStream` | NDJSON async iterator |
| `translate` | Translation convenience — capability-routed |
| `embed_texts` / `embedTexts` | Batch embeddings |
| `embed_query` / `embedQuery` | Single-string embedding |
| `run_tasks_parallel` / `runTasksParallel` | asyncio.gather-style multi-model compare |
| `engine_health` / `engineHealth` | Boolean reachability |

---

## Live stack (as of 2026-04-22)

Containers confirmed Up and healthy:

```
ach-sanctum-backend-1             0.0.0.0:8000
ach-postgres-1                    0.0.0.0:5434 (healthy)
sanctum-engine-sanctum-engine-1   0.0.0.0:8100
sanctum-engine-postgres-1         0.0.0.0:5433 (healthy)
translachat-translachat-1         0.0.0.0:8400
translachat-sanctum-forge-1       0.0.0.0:8200   [in-tree forge]
translachat-yjs-server-1          0.0.0.0:1234
translachat-postgres-1            0.0.0.0:5450 (healthy)
consilium-frontend                0.0.0.0:3800 (healthy)
consilium-backend                 0.0.0.0:3801 (healthy)
sanctum-kanban-app                0.0.0.0:3456
sanctum-kanban-db-dev             0.0.0.0:5432 (healthy)
```

**Port allocation across the suite** (to check when adding new services):

| Port | Claim |
|---|---|
| 1234 | translachat yjs-server |
| 3000 | Galatea perplexica-frontend (via profile) |
| 3001 | Galatea perplexica-backend (via profile) |
| 3125 | SanctumWriter (dev) |
| 3456 | SanctumKanban app |
| 3800 | Consilium frontend |
| 3801 | Consilium backend |
| 4000 | Galatea searxng |
| 5432 | SanctumKanban postgres |
| 5433 | sanctum-engine postgres |
| 5434 | ACH postgres |
| 5450 | translachat postgres |
| 5173 | Galatea frontend |
| 8000 | ACH backend |
| 8010 | Galatea backend |
| 8020 | Galatea vision |
| 8100 | **sanctum-engine** |
| 8200 | **sanctum-forge** (shared with translachat in-tree — use `FORGE_HOST_PORT=8201` override) |
| 8400 | translachat backend |
| 8880 | Galatea kokoro-tts |
| 8881 | Galatea chatterbox |
| 10200 | Galatea piper |
| 10300 | Galatea whisper |
| 11434 | host Ollama (Galatea's ollama container is profile-gated) |

---

## Known issues / technical debt

- **Consilium backend image build fails** on `npm ci --only=production` (exit 254). Not blocking — frontend can be rebuilt in isolation with `docker compose build frontend`. Needs a `rm package-lock.json && npm install` cleanup pass.
- **translachat has uncommitted WIP** on `docker-compose.yml`, `forge/importers.py`, `forge/requirements.txt`, `frontend/*`. Migration branches deliberately leave these untouched.
- **Galatea `docker-compose.yml`** has an uncommitted HF_TOKEN issue (the token value was pasted into the variable-name slot: `HF_TOKEN=${hf_rzl…:-}`). User needs to restore `HF_TOKEN=${HF_TOKEN:-}` and put the value in `.env`.
- **translachat's in-tree forge still runs on 8200**. sanctum-forge standalone uses `FORGE_HOST_PORT=8201` override to coexist. Real fix is migrating translachat to consume the published Forge client (Phase 2.2 follow-up).
- **ProcessPulse** had a stale `process-analyzer` compose project pointing to a deleted dir, causing its nginx frontend to crash-loop trying to proxy to a stopped backend. Cleaned up with `docker compose --project-name process-analyzer down`. Next `up` from the current repo creates fresh correctly-labeled containers.
- **Tokens** — user's GitHub PATs have been active through this work. They asked to revoke after work completes. See [`gitkeys.md`](d:/cowork/SynologyDrive/obsidian-next/015-sanctum/gitkeys.md) (git-ignored; lives outside the repo).

---

## What I'd tackle next when resuming

1. **Have the user validate one feature branch end-to-end** (translachat first — smallest). If it works, we have real confidence in the wrapper pattern.
2. **Finish Phase 2.5a Consilium** — apply `streamEngineChat` to the 8 remaining call sites.
3. **Phase 2.5c SanctumWriter/Pro** — clone both, audit LLM + docling call sites, migrate to Engine+Forge.
4. **Phase 2.6 `@sanctum/ui-tokens`** — brand palette package.
5. **Phase 2.7 repo transfers** — after everything else; move all 8 apps to the `SanctumSuite` org. GitHub redirects handle URL changes.

---

## Reference

- [`README.md`](README.md) — suite mission and app list
- [`SANCTUM_STANDARDS.md`](SANCTUM_STANDARDS.md) — canonical stack, config, license, deviations
- [`sanctum-engine` repo](https://github.com/SanctumSuite/sanctum-engine) — Engine service + Python client
- [`sanctum-forge` repo](https://github.com/SanctumSuite/sanctum-forge) — Forge service + Python client
- [`sanctum-engine-client-ts` repo](https://github.com/SanctumSuite/sanctum-engine-client-ts) — TypeScript client
