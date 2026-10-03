# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Status

Freshly scaffolded `create-next-app` project (Next.js 16 App Router, React 19, Tailwind CSS 4, TypeScript strict). No support-agent logic, API routes, or tests exist yet — `app/page.tsx` and `app/layout.tsx` are still the starter template.

## Commands

- `npm run dev` — dev server at http://localhost:3000
- `npm run build` / `npm run start` — production build / serve
- `npm run lint` — ESLint (flat config in `eslint.config.mjs`, extends `eslint-config-next` core-web-vitals + typescript)
- No test runner is configured.

## Notes

- Routing is App Router only (`app/`); path alias `@/*` maps to the repo root.
- Tailwind v4 is wired via `@tailwindcss/postcss` (`postcss.config.mjs`) and `app/globals.css`; there is no `tailwind.config` file.
- `eslint-config-next` in `package.json` is `^14` while `next` is `16.3.8` — a version mismatch to be aware of if lint behaves oddly.
- Per `AGENTS.md`, Next.js here has breaking changes from older versions: consult `node_modules/next/dist/docs/` before writing Next-specific code.

# Project brief

## What this is

An ecommerce support agent delivered as a web chat, on a fully mocked backend. A customer writes about their order; the agent identifies them, answers, and for refunds takes action under hard rules enforced in code. This is a portfolio piece to demonstrate Claude architecture skills. Reviewers will read the code, README, logs and eval results, so architecture decisions and clarity matter more than UI polish.

## Core principle

The model proposes, code disposes. Anything involving identity, money, ownership, or data exposure is decided by code from session state, never only by the prompt. If a rule could be broken by a persuasive customer message, it belongs in code.

## Stack (decided, don't change without asking)

- TypeScript everywhere, Next.js (App Router): UI and API routes in one app. Keep the UI thin.
- Anthropic TypeScript SDK, Messages API with a hand-written agent loop. Do NOT use the Agent SDK. Check the current SDK docs for exact API usage instead of relying on memory.
- Single agent (no multi-agent, no subagents).
- Default model claude-sonnet-5-5, configurable via AGENT_MODEL.
- In-memory session store behind an interface, keyed by httpOnly session cookie. Nothing is persisted. New session = new customer. Store must survive Next dev hot reloads.
- Mocked in-memory backend with fake data. No real customer data, payments, email, or SMS.
- Zod for tool input validation (single source of truth for tool input schemas).
- Vitest for unit tests (offline, no API key needed). Real API calls only in manual chat and in the eval harness.

## Supported intents

1. Order status (payment status and shipping status of an order)
2. Order tracking (carrier, tracking number, latest event, estimated delivery)
3. Refund (full refund only)

Plus identity verification and handoff to a human.

Adding a new intent must be cheap: one intent definition plus its tools, with no changes to the agent loop, guardrail pipeline, session handling or logging.

## Architecture in brief

- Agent loop: calls the Messages API, handles stop_reason, runs every tool call through the guardrail pipeline, always returns a tool_result for every tool_use block (is_error on failures), caps iterations per customer message (default 6).
- Intent registry: declarative definition per intent (tools, prompt fragment, whether it needs a verified customer). It composes the system prompt and the tool list. No separate classifier: the model's tool choice is the classification.
- Guardrail pipeline: ordered rules that run before each tool executes; each rule returns allow, deny (structured error), or force-handoff.
- Tools: small, single-purpose, strict input validation, structured errors, and an explicit output allowlist (the model only sees fields it needs; raw records never reach it).
- Session store, mock backend (customers, orders, shipments, payment gateway, tickets, dev inbox, clock), structured logger, web UI (chat, confirm buttons, handoff banner, "Talk to a human" button, dev panel).

## Hard rules (enforced in code)

1. Tool inputs are validated against strict schemas.
2. A handed-off session accepts nothing; after handoff no model calls are made, only a fixed message with the ticket ID.
3. All tools except verification and escalation need a verified customer; they are also hidden from the model until verification.
4. Customer identity comes only from the session. No tool has a customer-id parameter.
5. The order must belong to the verified customer, otherwise the response is the same "order not found" error as for a nonexistent order. The log records the real reason.
6. Tools return only allowlisted fields. Internal fields never reach the model.
7. Verification limits (configurable): code expires in 5 minutes, 3 attempts per code, 3 codes per session. Exhaustion forces handoff.
8. Refund eligibility: payment status paid, not already refunded, placed within a configurable window (default 30 days).
9. Refund auto-approval limit (configurable, default 100 EUR): above it, force handoff before asking for confirmation, no payment call.
10. State-changing actions need a UI button confirmation bound to the exact action. A typed "yes" never counts. A confirmation is consumed once and expires in 5 minutes. It stays valid across transient gateway retries.
11. Gateway failures: transient (timeout) returns a retryable error; two consecutive transient failures force handoff; a permanent decline forces handoff.
12. Idempotency: one refund per order; the idempotency key is reused on retries.
13. Max 6 model iterations per customer message; max user message length 2000 characters.
14. The refund amount is computed server-side from the order, never from model input.

## Conversation rules

- Always identify the customer before any order lookup, even if email/phone and order number arrive in the same message.
- Verification: customer gives email or phone, a code is "sent" to the dev inbox if the account exists, customer types the code. The model never sees the code; it passes the customer's typed digits to a verify tool and the server compares.
- Never reveal whether an account exists (identical responses and identical limit behavior either way).
- The model sees only the customer's first name after verification, no other PII.
- If something is ambiguous (e.g. missing order number), the agent asks instead of guessing.
- Handoff can be triggered by code (forced), by the agent (escalation tool), or by the customer (UI button, no model call). After handoff the session is locked.
- Code-forced handoff reasons are not available to the model.

## Conventions

- Tool result envelope: success = `{ ok: true, data }`, error = `{ ok: false, error: { code, message, retryable, ...details } }`. Known error codes: INVALID_INPUT, SESSION_LOCKED, NOT_VERIFIED, INVALID_CODE, CODE_EXPIRED, NO_PENDING_VERIFICATION, ORDER_NOT_FOUND, NOT_ELIGIBLE, CONFIRMATION_REQUIRED, GATEWAY_TRANSIENT, HANDOFF_FORCED, INTERNAL_ERROR.
- Order numbers look like ORD-123456. Customer-facing statuses: payment = paid | awaiting_payment | payment_failed | refunded; shipping = preparing | in_delivery | delivered. Internal backend statuses can be more granular and are mapped in code.
- All tunable limits live in one config module with env overrides.
- Tool descriptions and system prompt fragments live in clearly separated, easy-to-diff places (they are rewritten and measured later). Write honest, reasonable first versions; never weaken them on purpose.
- Mock data: obviously fake (emails @example.com), a single seed source, dates relative to a Clock abstraction (so "too old" stays too old and tests can advance time). Every internal-only field in an order holds a unique recognizable marker ("canary") value so tests can detect leaks to the model.
- Logging: structured events (model call, tool call, guardrail decision, state transition, ticket created) with session and turn ids, to console and a JSONL file in logs/. Emails and phones are redacted. Verification codes are never logged or put in model-visible text.
- Secrets only in .env, never committed.

## Working agreement

- Work on ONE feature at a time, the one I paste. Do not build ahead.
- Before coding, show a short plan (approach, main modules, tests) and wait for my go-ahead. If something in the spec is genuinely missing or contradictory, ask; do not invent business rules.
- Concrete file structure and mock data details are your call, but keep the code simple and readable; this will be reviewed.
- Each feature includes unit tests for its rules (offline) and a manual scenario list I can try in the chat.
- When a feature is finished: run typecheck, lint and tests; give a short summary and the manual scenarios; update "Current state" below; commit; stop and wait.
