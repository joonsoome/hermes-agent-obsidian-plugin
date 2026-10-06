# AGENTS.md

This file provides guidance to opencode when working with code in this repository.

## What this is

An **Obsidian plugin** ("Hermes Agent", id `hermes-agent`) that chats with a locally
installed Hermes Agent over its gateway HTTP API (default `http://127.0.0.1:8642`,
auth via `Authorization: Bearer <API_SERVER_KEY>`). The plugin is a pure API client;
transport logic mirrors Hermes Desktop's own client.

Two transports, capability-detected via `GET /v1/capabilities`:
- **Runs** (`POST /v1/runs` + SSE `GET /v1/runs/{id}/events`) — tools/reasoning/usage events.
- **Chat Completions** (`POST /v1/chat/completions`, OpenAI-compatible streaming) — fallback.

## Commands

```bash
npm run build       # esbuild -> main.js (production); dev mode = node esbuild.config.mjs (watch)
npm test            # builds runtime modules to tests/.build via esbuild, then `node --test`
npm run typecheck   # tsc --noEmit
npm run lint        # eslint src/**/*.ts (eslint-plugin-obsidianmd + typescript-eslint)
```

Tests are Node built-in test runner files (`tests/*.test.mjs`) importing esbuild-bundled
ESM from `src/runtime/` (protocol, context, history, graph). View/main code is not unit-tested.

## Release flow

GitHub Actions (`.github/workflows/release.yml`) fires on bare semver tags (no `v`).
Bump version in all three of `manifest.json`, `package.json`, `versions.json`; workflow
verifies tag == manifest version and attaches `main.js`/`manifest.json`/`styles.css`.
Distribution is via GitHub releases / BRAT, not the Obsidian community store.

## Remotes & branch strategy

- `origin` = fork `https://github.com/joonsoome/hermes-agent-obsidian-plugin.git`
- `upstream` = `https://github.com/jsun2020/hermes-agent-obsidian-plugin.git`
- Upstream PRs must be based on `upstream/main`, never on the fork's `main`.
- Current work: Ignis (browser-hosted Obsidian) support per
  `docs/plans/2026-08-21-ignis-additive-transport-plan.md` — additive transport so desktop
  behavior is unchanged. Branches: `feature/restore-saved-conversation` (Delivery 1),
  `feature/ignis-additive-transport` (Delivery 2), working branch `agent/pr14-resolved`.

Key plan constraints (see the doc): keep default URL `127.0.0.1:8642`; keep Node
`http`/`https` requester on desktop (gateway has CORS disabled); keep SSE Runs on desktop;
browser `fetch` requester only when Node HTTP unavailable; poll `GET /v1/runs/{id}` instead
of SSE only for the browser path (Ignis buffers proxied SSE); English UI defaults.

## Architecture notes

- `src/main.ts` — plugin entry: commands, vault-as-workspace wiring, top-level Node imports
  (`fs`, `os`, `path`) which is why full mobile/browser safety is an open item.
- `src/runtime/gatewayClient.ts` — all gateway HTTP: requester selection (Node vs browser),
  Runs SSE vs polling, approvals, Ignis proxy handling (`window.__originalFetch`, same-origin
  `/hermes` relay convention, Cloudflare Access caveats).
- `src/runtime/protocol.ts` — capability detection, model/context-length parsing, profile names.
- `src/runtime/context.ts` — prompt/system-instructions building, working-folder resolution.
- `src/runtime/history.ts` — persisted chat history; `src/runtime/graph.ts` — SmartGraph.
- `src/view/HermesView.ts` — multi-tab sidebar chat UI; `src/view/HermesGraphView.ts` — graph view.
- `src/settings/types.ts` + `src/settings/HermesSettingTab.ts` — settings schema/UI.
- `tsconfig.json` is intentionally lax (`strict: false`, `noImplicitAny: false`).
- Agent workspace: run `instructions` carry the working folder (gateway has no `cwd` field);
  auto-approval answers tool requests via `POST /v1/runs/{id}/approval`. File access depends on
  the gateway-side Codex sandbox config (`~/.codex/config.toml`), not the plugin — README has details.
