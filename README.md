# AI Diff Review Service

An asynchronous REST API that reviews git diffs and returns structured code-quality and security findings. Reviews run either through a **deterministic rules engine** or through **Google Gemini**, and results can be streamed live. A built-in web dashboard lets you try it from the browser.

![Dashboard preview](public/dashboard-live.png)

**Live demo:** https://ai-diff-review-c8pe.onrender.com
The demo runs on a free Render instance, so the first request after a quiet period can take a while while it wakes up. The API needs a bearer token (see [Authentication](#authentication)); ask the repo owner if you want to try the API against the hosted demo.

---

## Features

- **Two review providers**
  - `mock`: a comment-aware rules engine that scans only the *added* lines of a diff (see [Built-in rules](#built-in-rules)).
  - `llm`: Google Gemini (`gemini-3.5-flash`) with a JSON schema, so findings come back in the same structured shape as the rules engine.
- **Asynchronous jobs.** `POST` a diff, get a `jobId` back immediately (`202`), then poll or stream the result. A FIFO queue runs at most 4 jobs at a time.
- **Live streaming (SSE).** Status changes and findings arrive as Server-Sent Events. Connecting to a finished job replays the same event sequence.
- **Idempotency.** Send an `Idempotency-Key` header and a repeated request with the same body returns the same job. The same key with a different body returns `409`.
- **Caching.** A byte-identical diff and options pair resolves instantly with `usage.cacheHit: true`.
- **Chunking.** Diffs above 64 KiB are split on file boundaries into chunks of at most 64 KiB.
- **Rate limiting.** Submissions are limited to 30 per minute per IP (`429` after that).
- **Web dashboard** served at `GET /` to paste a diff, choose a provider and watch findings arrive.

## Built-in rules

| Rule | Finding | Severity | Category |
| :--- | :--- | :--- | :--- |
| `MOCK-001` | `eval` usage | critical | security |
| `MOCK-002` | Hardcoded credential | critical | security |
| `MOCK-003` | SQL string concatenation | high | security |
| `MOCK-004` | Swallowed exception (empty `catch`) | high | correctness |
| `MOCK-005` | Loose null comparison | medium | correctness |
| `MOCK-006` | Deep clone via JSON | medium | performance |
| `MOCK-007` | `console.log` left in | low | style |
| `MOCK-008` | Unresolved marker (TODO and similar) | low | style |
| `MOCK-INJ` | Prompt-injection content | critical | security |

Matches inside comments are ignored, and strict comparisons (`===`, `!==`) are not flagged as loose null checks. The severities above come from the rule definitions in `src/rules-engine.ts`.

## Getting started

**Requirements:** Node.js 20+ and npm 10+.

```bash
git clone https://github.com/malsaaaa/ai-diff-review.git
cd ai-diff-review
npm install
```

Create a `.env` file in the project root:

```env
PORT=3000
AUTH_BEARER_TOKEN=choose-a-long-random-string
GEMINI_API_KEY=your-google-gemini-api-key
```

- `AUTH_BEARER_TOKEN` is the secret that clients must send. **Always set your own value.** If it is missing, the server falls back to a built-in default token that is public in the source code.
- `GEMINI_API_KEY` is only needed for the `llm` provider. Without it, `llm` jobs fail with a clear error and the `mock` provider keeps working.

```bash
npm run dev      # development, restarts on changes
npm run build    # compile TypeScript to dist/
npm start        # run the compiled server
```

Open http://localhost:3000 for the dashboard.

## Authentication

Every `/v1/*` route requires a bearer token:

```
Authorization: Bearer <your AUTH_BEARER_TOKEN>
```

`GET /`, `GET /health` and `GET /spec` are public.

## API reference

| Method | Route | Auth | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/` | no | Web dashboard |
| `GET` | `/health` | no | Service status, version and uptime |
| `GET` | `/spec` | no | Machine-readable limits and providers |
| `POST` | `/v1/reviews` | yes | Submit a unified diff for review (`202` with a `jobId`) |
| `GET` | `/v1/reviews/:jobId` | yes | Job status, findings and usage |
| `GET` | `/v1/reviews/:jobId/stream` | yes | SSE stream of `status`, `finding` and `done` events |

### Submit a review

```bash
curl -X POST http://localhost:3000/v1/reviews \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $AUTH_BEARER_TOKEN" \
  -H "Idempotency-Key: my-first-review" \
  -d '{
    "diff": "diff --git a/src/db.ts b/src/db.ts\n+++ b/src/db.ts\n@@ -1,2 +1,3 @@\n+const x = 1;\n+eval(x);",
    "options": { "provider": "mock", "maxFindings": 100 }
  }'
```

`options` is optional: `provider` defaults to `mock`, and `maxFindings` is the cap on returned findings.

### A finding

```json
{
  "id": "MOCK-001:src/db.ts:3",
  "ruleId": "MOCK-001",
  "path": "src/db.ts",
  "line": 3,
  "severity": "critical",
  "category": "security",
  "title": "eval usage",
  "evidence": "eval(x);"
}
```

A job's `status` moves through `queued`, `running`, then `done` or `failed`. The `usage` object reports `inputBytes`, `chunks` and `cacheHit`.

### Errors

| Status | Code | Meaning |
| :--- | :--- | :--- |
| `400` | | Malformed JSON body |
| `401` | | Missing or wrong bearer token |
| `404` | `not_found` | Unknown job |
| `409` | `idempotency_conflict` | Same `Idempotency-Key` used with a different body |
| `413` | | Payload larger than 1 MiB |
| `422` | `invalid_diff` | Body is not a valid unified diff |
| `429` | `rate_limited` | More than 30 submissions per minute |

### Limits

| Limit | Value |
| :--- | :--- |
| Request body | 1 MiB |
| Chunk size | 64 KiB |
| Concurrent jobs | 4 |
| Submissions | 30 per minute per IP |

## Generating a diff to review

```bash
git diff                 # unstaged changes
git diff --cached        # staged changes
git diff main            # changes compared to main
git diff > changes.diff  # save to a file
```

Paste the full output, including the header lines, into the dashboard or send it as the `diff` field.

## Architecture

- **Express 5** handles HTTP, with a small custom middleware layer for auth, rate limiting and error handling.
- **Store** (`src/store.ts`) keeps jobs, findings, idempotency keys and the result cache in memory and writes them to `data-store.json` with debounced background writes.
- **Queue** (`src/queue.ts`) is a custom FIFO scheduler capped at 4 running jobs. It splits large diffs into chunks and emits the SSE events.
- **Providers** are `src/rules-engine.ts` (regex rules over parsed hunks) and `src/llm-provider.ts` (plain `fetch` calls to the Gemini API with a response schema).
- **Diff parser** (`src/diff-parser.ts`) reconstructs file paths and line numbers from the hunk headers so each finding points at the right line.

### Design decisions

- **A JSON file instead of SQLite.** An AI assistant suggested `better-sqlite3`. It was rejected because native build dependencies often fail on Windows development machines. A file-backed JSON store gives enough persistence for this service without native compilation.
- **Add a database later.** For multiple instances, use PostgreSQL or Redis rather than a local file.

### Next steps

- Request tracing with correlation IDs
- A persistent database for distributed use
- Separate workers for CPU-bound rule scanning and network-bound LLM calls
- Horizontal auto-scaling

## Tests

The integration suite starts from the HTTP layer and covers auth, rate limiting, error handling, caching, idempotency, chunking and SSE streaming with replay:

```bash
npx tsx src/test/integration.ts
```

## Project structure

```
ai-diff-review/
├── src/
│   ├── index.ts           # Express app, routes, middleware
│   ├── queue.ts           # Job queue and chunking
│   ├── rules-engine.ts    # Deterministic "mock" provider
│   ├── llm-provider.ts    # Gemini provider
│   ├── diff-parser.ts     # Unified diff parser
│   ├── store.ts           # In-memory store with JSON persistence
│   ├── types.ts           # Shared types
│   └── test/integration.ts
├── public/                # Web dashboard (HTML, CSS, JS)
└── package.json
```

## Built with

Node.js, TypeScript, Express 5, Google Gemini API. The code was developed with the help of an AI coding assistant (Gemini 3.5 Flash via Antigravity).

## License

Copyright (c) 2026 malsaaaa. All rights reserved.

This repository is public for viewing and portfolio purposes. No license is granted to copy, modify, distribute or use the code without the author's written permission.
