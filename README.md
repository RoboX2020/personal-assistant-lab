# Personal Assistant Lab

An independent, single-owner assistant prototype built with public APIs. A mobile web interface and Telegram adapter share durable jobs, explicit memory, reminders and bounded read tools.

**Current release: v0.1, local-tested source prototype. Not deployed.**

## Works without an LLM key

- Owner-authenticated web chat and Telegram text webhook adapter.
- Explicit `/remember`, `/memory`, `/forget`, `/export` and `/delete-all CONFIRM` commands.
- Reminder preview, exact confirmation, cancellation and durable scheduling.
- Public GitHub repository reads.
- Task status, stop/resume switch, idempotent incoming events.
- A visibly simulated calendar, not a connected personal calendar.

## Implemented but needs live-provider testing

- Gemini chat and bounded public-GitHub/memory function calls.
- Public web research with Gemini Search grounding, source links and provider suggestions, in the web panel only.
- Telegram delivery: webhook authentication and owner filtering are locally tested; real bot delivery requires configuration and end-to-end verification.

## Not implemented

Official WhatsApp deployment, real calendar OAuth, private GitHub access, automatic messages to other people, purchases, browser control, arbitrary shell execution and multi-user tenancy.

This is not a reproduction of a proprietary product or a promise of product parity.

## Privacy and platform gates

Default `DATA_MODE=synthetic` and `MODEL_ROUTE=unpaid`: use only invented test facts and public data. Google's unpaid Gemini terms explicitly say not to submit sensitive, confidential or personal information. Do not connect a real calendar or private archive in this mode.

The model adapter refuses `DATA_MODE=real` unless `MODEL_ROUTE=paid-reviewed`. That setting is an operator attestation, not an automatic provider/privacy audit. Do not change it without actually reviewing the provider's data terms, billing and user permission. Billing is not enabled by this project.

WhatsApp is an inactive target channel. Meta restricts general-purpose AI assistants on its Business Platform except permitted markets. Eligibility for a US personal-assistant use case is not established. Unofficial clients are not included and are not a recommended policy bypass. See [public-source design](docs/design.md).

## Source distribution

This milestone publishes the complete 22-file source tree as a reviewed source archive plus a readable Node extractor. The archive contains the original folder layout, lockfile, tests, design and verification notes. No credentials, node_modules or runtime database are included.

After cloning this branch, run:

```sh
node 1-unpack-source.mjs
cd assistant-source
```

Alternatively extract `2-personal-assistant-lab-v0.1.tar.gz` into an empty directory. Do not run both into the same directory. The extractor refuses to overwrite existing source. Normal native source-tree commits are the next repository packaging improvement; this distribution does not activate root-level CI by itself.

## Run locally

Requires Node 22+, npm and a browser. No paid account is needed for command-only demo mode.

```sh
npm ci
cp .env.example .dev.vars
# Replace OWNER_WEB_TOKEN with a strong random value (at least 24 characters).
# Leave provider and Telegram secrets empty for the local command-only demo.
npm run db:local
npm run dev
```

Open the localhost URL Wrangler prints. Enter the owner token into the page, then try:

```text
/help
/remember The demo mascot is a fox.
/memory
/github octocat/Hello-World
/remind 2030-01-02T10:00:00-07:00 Review the demo
```

Review the reminder text and time, then send the exact `/confirm <id>` from its preview. A web reminder lands in the page's inbox, not a phone push notification. Refresh the reminder inbox to read it.

Use `npm run dev -- --test-scheduled` to expose Wrangler's local scheduled-event test endpoint; ordinary `wrangler dev` does not run a real remote minute cron.

## Tests

```sh
npm run check
npm test
npx playwright install chromium
npm run test:runtime
npm run test:schedule
npx wrangler deploy --dry-run
```

Runtime tests temporarily write `.dev.vars` with fake local-only secrets and remove it afterward. **Do not run them in a working directory containing real `.dev.vars`.** They use a separate temporary local database state. No provider key is used, no Telegram send occurs, and no real account is connected.

Utility coverage: parsing, invalid dates, offsets, token comparison, bounded queries and privacy route gate. Runtime coverage: owner auth, non-owner webhook rejection, duplicate events, memory correction/deletion, reminder confirmation/cancellation, stop/resume, command-only honesty, CSRF, scheduled dispatch and responsive layouts. Schedule tests verify due delivery, stop behavior and no resend after completion.

## Deploy only after you decide

1. Choose the channel and privacy mode. Default is public/synthetic data only.
2. Create a Cloudflare account/project if needed; confirm Free plan limits before continuing.
3. Run `npx wrangler d1 create assistant`, and paste the returned database ID into `wrangler.toml`.
4. Run `npm run db:remote` to apply the schema.
5. Add secrets using `npx wrangler secret put OWNER_WEB_TOKEN` and any optional provider/bot secrets. Never put secrets in source or chat.
6. Review the configured minute Cron Trigger and best-effort delivery expectations.
7. Run `npm run deploy` and test the deployment with synthetic data.

The Free Workers CPU cap is tight. The local runtime passing does not prove the remote handler stays within the free CPU budget. Measure actual remote CPU, latency and model failure behavior before calling this dependable. Longer tasks remain stored and are resumed by the dispatcher, but expired in-flight tasks become `outcome-unknown` rather than being blindly replayed.

## Telegram connection

Create your own bot with BotFather; configure `TELEGRAM_BOT_TOKEN`, numeric `TELEGRAM_OWNER_ID` and a random `TELEGRAM_WEBHOOK_SECRET` through server secrets. The owner must start the bot. Run `ASSISTANT_URL=https://your-deployed-host node scripts/telegram-webhook.mjs` with the token and secret in local environment variables. This configures Telegram `setWebhook` with the deployed HTTPS `/webhook/telegram` URL and the same `secret_token`. Use a local script/environment variable, not a token-bearing URL pasted into a conversation.

Only owner-to-bot private text messages are accepted. Group chats and non-owner messages are ignored. Telegram Search grounding is intentionally disabled until its provider-required presentation is implemented; use the web search panel.

## Data and operations

D1 tables: messages, jobs, memory, reminders, outbox, usage and settings. Never store credentials in these tables. Use server secrets for keys; real OAuth refresh-token encryption is a later feature.

A minute cron checks queued jobs and due reminders. It is not an exact alarm guarantee. Reminders are atomically claimed. After ambiguous delivery, the row becomes `outcome-unknown`, with no automatic resend. Reconcile provider state before manually retrying. Failed jobs appear under `/tasks`.

`/stop` stops new tools and reminder sends, but cannot retract a send already in flight. `/delete-all CONFIRM` deletes conversation records and reminders; it cannot delete copies retained by Telegram, model providers or operator backups.

No anonymous API access, no token persistence in browser storage, no HTML rendering of chat text, no arbitrary URL-fetch tool and no arbitrary code execution.

## Project structure

```text
src/worker.ts        web/Telegram entry points, commands, jobs and dispatch
src/providers.ts     bounded external API reads and model adapter
src/core.js          pure parsing and policy utilities
schema.sql           durable state
public/              responsive owner UI
scripts/             local setup aids
tests/               utility, runtime and scheduling tests
docs/design.md       public-source architecture and roadmap
docs/verification.md what was actually tested
```

## Roadmap

- v0.1: local chat shell, explicit memory and reminder/read commands.
- v0.2: live model/bot validation and free-host deployment metrics.
- v0.3: reviewed OAuth connections and read-only calendar/private repos.
- v0.4: individual approval-based write tools; permitted WhatsApp if eligible.
- Later: voice, documents, recurring projects and a test lab for agent permission controls.

This repository's publication visibility must match its owner's choice. Public source does not mean public access to the running assistant. New deployments must keep authenticated owner data separate from source.
