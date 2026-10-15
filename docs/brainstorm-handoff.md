# Brainstorming-epa Handoff Payload → enterprise-project-analyzer

**Date:** 2026-07-21
**Project:** HLC-Chat — fast, minimal, multi-user LLM chat UI (Llama Swap–inspired, without the weight)
**Repo state at handoff:** fresh empty git repo (no commits), remote `https://github.com/platteXDlol/HLC-Chat.git`. No prior specs, ADRs, or decision logs existed.

---

## understanding_summary

- **WHAT** — A self-hosted, minimal multi-user chat web app (SvelteKit, single process) for **one** OpenAI-compatible LLM server. Llama Swap-style single-column chat: streaming responses, markdown/code rendering, and a **live model switch above the input** (list pulled from `/v1/models`, tap to switch mid-conversation, no model registry, no per-model config).
- **WHY** — The user wants Llama Swap's chat *feel* without Llama Swap's *weight*: no model registry, no proxy layer, nothing that slows it down — with Authentik SSO and per-user isolation instead of built-in accounts.
- **WHO** — The user and a small team (assumed <50 people), all signed in through Authentik. No local passwords, ever.
- **CONSTRAINTS** — One LLM backend (URL is a server-side env var, never exposed to the browser); direct OIDC against Authentik with JWKS-validated session cookie; SvelteKit + SQLite in a single Docker deploy.
- **NON-GOALS (v1)** — multiple LLM backends, shared/team chats, per-model temperature/system-prompt UI, model import/management, admin console, i18n, mobile app.

## assumptions

- **A1:** Authentik is already deployed and reachable from the app.
- **A2:** TLS is terminated at the user's edge (Authentik or the proxy in front); the app itself runs plain HTTP behind it.
- **A3:** Performance targets: app adds <10 ms overhead to LLM requests; streamed first token lands as soon as the server emits it; chat page loads <1 s on the LAN.
- **A4:** Scale: tens of concurrent users, single node, no high-availability needs.
- **A5:** All chat traffic goes through the app (thin relay to the LLM server) — per-user history and private isolation fall out naturally.
- **A7:** A *global* default system prompt via env var (one for everyone, no per-chat UI), **empty by default** — proposed default, pending OQ2 confirmation.
- **Accepted risk (no v1 mitigation):** no per-user rate limiting on `/api/chat` — trusted LAN + SSO.

## open_questions

- **OQ1:** Is Authentik already up and running? (affects only setup, not design)
- **OQ2:** OK with a global default system prompt via env var (empty by default)? (assumption A7 rests on this)

## decision_log

| # | Decision | Alternatives considered & rejected | Why |
|---|----------|-----------------------------------|-----|
| D1 | One OpenAI-compatible server as the LLM backend | multi-backend gateway; hosted APIs (key-based); Llama Swap's own API | no proxy layer, least code, fastest path; Llama Swap coupling explicitly rejected by user |
| D2 | SvelteKit (single full-stack process) | Next.js (heavier runtime/bundle); FastAPI + tiny static frontend (more owned glue code) | smallest JS payloads, built-in streaming, one deployable |
| D3 | Model switch: compact switch above the chat input, live list from `/v1/models`, instant mid-conversation switch, no registry | header dropdown with search; pinned model tabs; Llama Swap's model-registry model | no registry to manage; server decides what exists; fastest switching |
| D4 | Multi-user = private per-user chat history, server-side (one table + `user_id` scoping) | ephemeral localStorage-only; shared team chats | expected UX for a chat app at ~zero cost; team chat = significant extra surface (non-goal v1) |
| D5 | Auth = direct OIDC in the app: authorization code + PKCE, `id_token` in httpOnly/Secure/SameSite=Lax cookie, per-request JWKS-cache validation | reverse-proxy forward-auth (Traefik/Caddy + Authentik proxy) | fewer hops, fewer moving parts; JWKS validation is sub-millisecond |
| D6 | Understanding Lock confirmed by user | — | explicit user confirmation ("Confirmed — lock it") |
| D7 | Architecture Approach A — thin relay + local state (auth + `/v1/models` proxy + SSE stream relay + per-user SQLite history; nothing else) | B: app as OpenAI-compatible gateway + UI (more security surface, YAGNI — kept as cheap v2); C: client calls LLM server directly (exposes key/URL in browser, breaks isolation — rejected on D1/A5 grounds) | exactly the brief's scope; B remains a cheap v2 because the relay layer already exists |
| D8 | Design sections 1–4 accepted as presented, no revisions: (1) architecture & components, (2) data flow, (3) error handling & edge cases, (4) testing strategy | — | per-section user confirmation |
| D9 | Persistence rules: user message persisted only when the LLM stream starts (HTTP 200 + first byte); mid-stream break → partial assistant text kept, flagged `incomplete`; user Stop → partial kept, flagged `stopped`; connection failure → nothing persisted, message stays in composer | persist user message before LLM call (leaves dangling messages on LLM failure) | no dangling/inconsistent chat state when the LLM is down |
| D10 | No per-user rate limiting in v1 (accepted, documented risk) | per-user throttle on `/api/chat` | trusted LAN + SSO; keep surface minimal (see assumptions) |
| D11 | Storage = SQLite via better-sqlite3 + Drizzle, single Docker volume | PostgreSQL/external DB (operates a second service); raw SQL (no type safety) | file-based, zero extra services, still type-safe |
| D12 | SvelteKit retained after user question "why not Rust/C++/C?" — answered: app is I/O-bound (JWT verify µs, SQLite sub-ms, SSE relay); language diff is ~2 ms on a ~2000 ms LLM response; UI is JavaScript regardless of backend; Rust/Go become relevant only at multi-backend-gateway scale | Rust (Axum) backend; C++/C backend | "blazing fast" is achieved by architecture (thin relay, streaming, tiny bundle), not backend language |

## target_skill

`enterprise-project-analyzer`

## mode

`step-by-step`

## context_snapshot

- **Repo:** `/opt/projects/HLC-Chat` — fresh empty git repo (no commits), remote → `https://github.com/platteXDlol/HLC-Chat.git`. No AGENTS.md/README/specs/ADRs existed; no prior brainstorming output.
- **Confirmed design (sections 1–4, all user-accepted):**

### Architecture & components
- Browser (SvelteKit, SSR shell + SPA-ish chat) → HTTPS (TLS at user's edge, not the app) → **one SvelteKit process, one Docker image**.
- Server layers: auth (OIDC flow + session cookie, JWKS-validated); pages `/` (chat list) and `/chat/[id]` (conversation view); API endpoints (all session-gated): `GET /api/models` (proxies LLM `/v1/models`, 30 s in-memory cache), `GET/POST /api/conversations`, `GET/PATCH/DELETE /api/conversations/:id`, `POST /api/chat` (SSE stream relay); LLM client `fetch(LLM_URL)` — env var, `stream:true`, never client-side; SQLite (better-sqlite3 + Drizzle, one volume).
- DB: `conversations(id, user_id, title, model, created_at, updated_at)`, `messages(id, conversation_id, role, content, model, created_at)`.
- Client components: `ChatList` (collapsible sidebar), `ConversationView` (message list + streaming), `Message` (markdown + code highlighting), `ModelSwitch` (live pill above input), `Composer` (auto-growing textarea, Enter to send, Stop button while streaming), `AuthGate` (redirects to OIDC flow).
- No state library, no ORM beyond Drizzle, no queue, no websocket. ~10 ms app overhead in the LLM path (A3).

### Data flow
- **Auth:** unauthenticated visit → `/auth/signin` → PKCE + state → Authentik `/application/o/authorize/` (`scope: openid profile email`, `redirect_uri: /auth/callback`) → user logs in in Authentik → app exchanges code server-side → `id_token` (JWT) in httpOnly, Secure, SameSite=Lax cookie. Per request: verify vs cached JWKS (fetched once, refreshed on new `kid`), check `exp` (30 s leeway) + `aud`; fail → clear cookie, re-redirect (silent, user already in Authentik). No refresh-token juggling in v1. Identity = `sub` + `email`; every query scoped `WHERE user_id = ?`.
- **Chat:** client `POST /api/chat { conversation_id, model, content }` → first message auto-creates conversation (title = first words) → server checks session + ownership (else 404, never leaks existence) → calls `LLM_URL/v1/chat/completions` (`stream:true`, full history + global system prompt env var) → **pipes SSE bytes straight through** (no buffering, no per-token DB writes) → client renders tokens live, Stop aborts → on clean end, assistant message persisted with model stamp, `updated_at` bumped.

### Error handling & edge cases
- No first byte within 10 s / connection refused → SSE `error` event → dismissible banner; nothing persisted; message stays in composer.
- Mid-stream break → partial kept, flagged `incomplete`; composer refills with last user message.
- Stop button → partial kept, flagged `stopped`.
- LLM 400 (bad model / context overflow) → exact error as chat error bubble; context overflow offers "New chat" action.
- Model vanishes from `/v1/models` → fall back to first available + toast; old messages keep historical stamps.
- `/v1/models` fetch fails → last known list (server 30 s cache + browser cache) with warning icon; chat keeps working.
- Model switch disabled while streaming; takes effect from next message.
- XSS via model output → markdown renderer escapes all HTML; code highlighting lazy-loaded, runs once on block completion.
- Concurrent tabs → last-write-wins (documented). Long conversations → full history, no truncation (sliding window = v2).
- Delete → hard delete, cascades. Empty model list → chat disabled with clear message.

### Testing strategy
1. **Unit (Vitest):** JWT validation (valid/expired/bad sig/unknown kid/wrong aud); ownership scoping (every data-layer function must reject foreign ids, parameterized); SSE relay vs mock upstream (clean end, mid-break, 400 passthrough, connect timeout); model-list cache (TTL, fallback).
2. **Integration (Vitest + real SQLite temp file + mock LLM fixture):** full round-trip (create → send → stream → both messages persisted with model stamps); first-message auto-create + title; unauthenticated `/api/*` → 401/redirect; `/auth/callback` exchange (mocked Authentik).
3. **E2E (Playwright, 5 smoke flows):** login → chat list; send → tokens stream → survive reload; model switch changes next message's model; other user's conversation URL → 404; Stop mid-stream.
4. **Manual (once per deploy):** real Authentik + real LLM server (OIDC is environment-coupled).
- Not testing (YAGNI): visual regression, load testing, exhaustive markdown matrix.
- CI (GitHub Actions): `svelte-check` + lint + Vitest + Playwright on every PR.

---
*Payload produced by `brainstorming-epa` (agent-kit first-party fork, MIT). Handoff target: `enterprise-project-analyzer`, mode `step-by-step`. Do not paraphrase fields when ingesting.*
