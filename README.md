# HLC-Chat

Fast, minimal, multi-user chat interface for your OpenAI-compatible LLM server.

The feel of Llama Swap's chat, without the weight: no model registry, no proxy layer, nothing between you and your model. Sign in through Authentik, pick a model, chat.

## Features (v1)

- **Blazing fast** — a thin server-side relay adds a few milliseconds per request (≈2 ms vs ≈2 s of model inference) and streams tokens as soon as the server emits them
- **Live model switch** — a compact switch above the input, list served live from the LLM server's `/v1/models`; switch models mid-conversation. No registry, no import, no per-model config
- **Authentik SSO** — direct OIDC (authorization code + PKCE), no local passwords; the session is validated per request against the issuer's JWKS
- **Private per-user history** — your conversations are yours; isolation is a data-layer rule, not a UI nicety
- **Streaming with Stop** — SSE relay; interrupted responses are kept and flagged, never lost

## Status

🚧 **Phase 0 complete — the design is locked.**
The full handoff payload (understanding lock, assumptions, open questions, decision log D1–D12, confirmed design) is in [`docs/brainstorm-handoff.md`](docs/brainstorm-handoff.md).

Proposed implementation slices:

| # | Slice | State |
|---|-------|-------|
| 1 | Design, handoff payload, config template (this PR) | ✅ |
| 2 | Foundation — SvelteKit scaffold, Dockerfile, CI (svelte-check, lint, tests) | planned |
| 3 | Auth — Authentik OIDC flow, session hook, JWKS validation | planned |
| 4 | Data layer + conversations API + chat list page | planned |
| 5 | Chat streaming, model switch, composer | planned |

## Requirements

- An **OpenAI-compatible LLM server** — llama.cpp (`llama-server`), vLLM, LM Studio, or similar
- An **Authentik** instance with an OIDC application for HLC-Chat
- **Docker** for deployment, or **Node.js ≥ 22** for development

## Configuration

All configuration is via environment variables — see [`.env.example`](.env.example). Only server-side secrets live there; nothing is ever exposed to the browser.

| Variable | Required | Description |
|----------|----------|-------------|
| `PUBLIC_URL` | yes | Public URL of the app, used for the OIDC `redirect_uri` (e.g. `https://chat.example.com`) |
| `LLM_URL` | yes | Base URL of your OpenAI-compatible server; the app calls `<LLM_URL>/v1/models` and `<LLM_URL>/v1/chat/completions` |
| `AUTHENTIK_ISSUER` | yes | Issuer of your Authentik application (e.g. `https://auth.example.com/application/o/hlc-chat/`) |
| `AUTHENTIK_CLIENT_ID` | yes | Client ID of the Authentik OIDC application |
| `AUTHENTIK_CLIENT_SECRET` | no | Only for confidential OAuth clients; omit for public clients (PKCE) |
| `SYSTEM_PROMPT` | no | Global default system prompt applied to all chats (empty by default) |
| `LLM_TIMEOUT_MS` | no | Connect timeout to the LLM in milliseconds (default `10000`) |
| `DB_PATH` | no | SQLite file path (default `./data/hlc-chat.db`; volume-mount in Docker) |

## Design decisions

The full decision log with alternatives and rationale lives in [the handoff doc](docs/brainstorm-handoff.md). The short version:

- **SvelteKit, one process, one Docker image** — smallest possible payloads, built-in streaming, one thing to deploy
- **SQLite** (better-sqlite3 + Drizzle) — one volume, zero extra services, plenty for a per-user chat store
- **Direct OIDC in the app** — no reverse-proxy auth hop; the id_token lives in an httpOnly cookie and is validated per request against a cached JWKS
- **Thin relay** — the app does exactly four jobs: auth, model list, stream relay, history. Nothing else.

## Non-goals (v1)

Multiple LLM backends · shared/team conversations · per-model parameter UI · model management · admin console · i18n · native mobile apps · per-user rate limiting (accepted risk: trusted network + SSO)
