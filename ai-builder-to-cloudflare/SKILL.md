---
name: ai-builder-to-cloudflare
description: Migrate apps built on AI app builder platforms (Base44, Lovable, Bolt.new, V0, Replit) to Cloudflare Pages + Workers AI + D1 + R2. Use this skill whenever the user mentions migrating from Base44, Lovable, Bolt, V0, or Replit, wants to leave an AI app builder, escape vendor lock-in, move to Cloudflare, self-host an AI-generated app, or replace Supabase with Cloudflare services. Also trigger when the user has a codebase with @base44/sdk, @supabase/supabase-js, or Vercel-specific imports and wants more control, better performance, or lower costs.
---

# AI App Builder to Cloudflare Migration

Migrate any app built on an AI app builder platform to Cloudflare Pages + Workers AI + D1 + R2. Covers Base44, Lovable, Bolt.new, V0, and Replit — the five most common platforms people want to migrate from.

Most of these platforms generate a **React + Tailwind SPA** (Base44, Lovable, Bolt — Vite builds to a static `dist/`). The frontend is usually fine — the migration is really about **replacing the backend services** (database, auth, AI, file storage) with Cloudflare equivalents you own and control.

**Exception: V0 generates Next.js App Router** (React Server Components, Server Actions, route handlers) — that's a different build and deploy path, covered in Phase 5. Don't assume SPA for V0.

## How to Use This Skill

This skill is the **method** — the ordered sequence of steps to move an app off a builder platform and onto Cloudflare. It deliberately does **not** hardcode limits, pricing, model IDs, API signatures, or feature availability, because those change. For anything that can go stale, look it up live:

- **Cloudflare Docs MCP** (`search_cloudflare_documentation`) — the source of truth for D1/R2 limits, Workers AI models and pricing, binding syntax, and current APIs. Query it before relying on any number. Use `migrate_pages_to_workers_guide` when a Pages→Workers question comes up.
- **Official Cloudflare skills** — invoke these for the deep, current detail on each service. Don't reimplement what they cover:
  - `wrangler` — CLI commands, `wrangler.toml`/`wrangler.jsonc` config, deploy flags
  - `workers-best-practices` — D1 batching, transactions, the Sessions API, edge patterns
  - `durable-objects` — anything realtime / stateful / WebSocket
  - `cloudflare-email-service` — transactional email
  - `agents-sdk`, `sandbox-sdk`, `web-perf` — as the app needs them

When this skill says "check the docs" or "use the X skill," do it — don't guess from memory.

## Why Migrate

AI app builders trade control for speed. That tradeoff stops making sense when you hit the platform's scaling limits, you're paying platform fees for infrastructure that's cheap to self-own, the platform injects tracking/branding into your bundle, or you want edge performance and edge AI you control. (For current Cloudflare limits and pricing to put in a before/after comparison, query the Docs MCP — don't quote stale figures.)

## Step 0: Identify the Platform

Before anything else, figure out which platform generated the app. Grep the codebase:

```bash
# Base44
grep -r "@base44/sdk\|base44Client\|base44\.entities\|base44\.integrations" src/ --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" -l

# Lovable / Bolt.new (both use Supabase)
grep -r "@supabase/supabase-js\|supabaseClient\|supabase\.from\|supabase\.auth\|supabase\.storage" src/ --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" -l

# V0 / Next.js
grep -r "next/\|@vercel/\|getServerSideProps\|getStaticProps\|app/api/\|server actions" src/ app/ --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" -l

# Replit
grep -r "@replit/\|replit\.dev\|REPL_" src/ --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" -l
```

Then read the platform-specific reference for that platform's SDK patterns and how they map to Cloudflare: [platform-patterns.md](references/platform-patterns.md).

## Step 1: Get the Code & Data

Before touching anything, get the source code and a copy of the live data. Both are needed for the migration.

### Get the source code

Every platform has a way to export code. Try in this order:

**Via MCP (fastest — if the platform has a Claude MCP server connected):**
```
Ask Claude: "Export the source code for app <id> via the Base44/Lovable/etc. MCP"
```
Claude can read files and entity schemas directly through the MCP connection — no manual download needed.

**Via GitHub (most common):**
Most AI builders sync your project to a GitHub repo automatically. Check:
- Base44: Settings → GitHub → connect or view repo
- Lovable: Project → Export → GitHub (or the repo is auto-created under your account)
- Bolt.new: Download as zip or push to GitHub via the UI
- V0: Already Next.js — clone the repo or download from the dashboard

Then clone it:
```bash
git clone https://github.com/your-username/your-app-repo
```

**Via platform zip export:**
All platforms have a "Download code" or "Export project" button. Download and unzip locally.

### Export the live data

Don't skip this — the app's data is in the platform's backend, and you need it to seed D1.

**Base44 (via MCP — preferred):**
If the Base44 MCP is connected, Claude can query entities directly:
```
Ask Claude: "Query all records from the Lesson, Material, etc. entities in app <id> and save as seed SQL"
```

**Base44 (manual export):**
Use the Base44 admin panel → Data → Export, or call the API:
```bash
curl "https://api.base44.com/api/apps/<app-id>/entities/<entity-name>" \
  -H "Authorization: Bearer <token>" | jq '.'
```

**Supabase (Lovable/Bolt):**
```bash
# Export each table
supabase db dump --data-only -f seed.sql
# Or via dashboard: Table Editor → Export → CSV per table
```

**General rule:** Export every table as INSERT statements into `migrations/0002_seed.sql`. D1 caps the size and bound-parameter count of a single statement, so don't pack hundreds of rows into one multi-row INSERT — generate individual (or small-batch) INSERTs, and for large tables prefer D1's CSV/file import. Check the current per-statement limits with the Docs MCP before generating the seed file.

## Phase 1: Discovery & Inventory

**Goal:** Catalog every backend service the app uses.

Scan the codebase and produce a checklist:

- [ ] **Database calls** — CRUD operations, queries, filters, joins
- [ ] **Auth** — login, signup, OAuth providers, session management
- [ ] **File uploads** — images, documents, user-generated content
- [ ] **AI/LLM calls** — text generation, vision, structured output
- [ ] **Email** — transactional email sends
- [ ] **Realtime** — WebSocket subscriptions, live updates
- [ ] **External APIs** — third-party integrations called from the frontend (CORS issues ahead)
- [ ] **Backend functions** — server-side code (Deno functions, Edge Functions, API routes)
- [ ] **Analytics/tracking** — event logging, page views

For Base44 apps specifically, also check for: `InvokeLLM`, `UploadFile`, `SendEmail`, `GenerateImage`, `ExtractDataFromUploadedFile`, `.subscribe()`, custom integrations. See [base44-sdk.md](references/base44-sdk.md) for the full API surface.

## Phase 2: Cloudflare Infrastructure

**Goal:** Set up the Cloudflare services that replace the platform's backend.

Every AI builder's backend maps to the same set of Cloudflare services:

| Platform Feature | Cloudflare Replacement | Notes |
|---|---|---|
| Database (Supabase PostgreSQL / Base44 entities) | **D1** (SQLite at edge) | Globally replicated reads, no connection pooling |
| File storage (Supabase Storage / Base44 UploadFile) | **R2** (S3-compatible) | Zero egress fees |
| Auth — **internal / admin tools** | **Cloudflare Access** (Zero Trust) | Only for team/admin gating, NOT public consumers |
| Auth — **public B2C** (Supabase Auth / NextAuth) | **Auth.js, Lucia, or Better Auth** + D1 adapter | Cloudflare Access can't do consumer signup; use an edge-native auth library with its official D1 adapter |
| AI / LLM calls (OpenAI API / Base44 InvokeLLM) | **Workers AI** | No API-key management; query Docs MCP for current models + pricing |
| Serverless functions (Edge Functions / API routes) | **Pages Functions** | Same V8 isolate model, file-based routing |
| Realtime (Supabase Realtime / .subscribe()) | **Durable Objects** (or **PartyKit**, which wraps DO for WebSockets) | Use the `durable-objects` skill; flag as manual work |
| Email (Resend / SendGrid / Base44 SendEmail) | **Email Workers / MailChannels** | Use the `cloudflare-email-service` skill |

Generate `wrangler.toml` with only the bindings the app actually needs — one binding per service the app uses (`[ai]`, `[[d1_databases]]`, `[[r2_buckets]]`, etc.), and nothing it doesn't. Set `compatibility_date` to today's date. For exact binding syntax and the latest config format (`wrangler.jsonc`), use the **`wrangler` skill** rather than copying a fixed template.

See [cloudflare-services.md](references/cloudflare-services.md) for the binding examples and Pages Function code templates.

## Phase 3: Schema Migration

**Goal:** Convert the platform's database to D1 SQL.

The conversion depends on the source platform:

- **Base44** → MongoDB-style NoSQL entities → D1 SQL tables. See [schema-mapping.md](references/schema-mapping.md) for type mapping and query translation.
- **Supabase (Lovable/Bolt)** → PostgreSQL → D1 SQL. Most types map directly. Key differences: D1 doesn't support `uuid` type (use `TEXT`), `JSONB` → `TEXT` (use `json_extract()`), no `SERIAL` (use `INTEGER PRIMARY KEY AUTOINCREMENT`).
- **V0/Replit** → Varies by what they generated. Check for Prisma schemas, Drizzle configs, or raw SQL.

The schema mapping reference covers all conversion rules: [schema-mapping.md](references/schema-mapping.md).

Generate migration file: `migrations/0001_initial.sql`

Before generating it, scan the source schema for two things D1 handles differently:

- **Heavy columns** — AI builders often store uploaded images/avatars as base64 in a `TEXT`/`BYTEA` column. D1 has a hard per-row size cap and a modest per-database cap, so a blind copy will blow past them. Split that data out: upload the blobs to **R2** and store only the object key in D1.
- **Transactions / RPCs** — D1 has no interactive `BEGIN…COMMIT`; multi-step logic (Supabase RPCs, Prisma `$transaction`) must be rewritten as `db.batch()` with all statements prepared upfront, intermediate reads pulled out. See the `workers-best-practices` skill.

For the seed file (`migrations/0002_seed.sql`): generate individual/small-batch INSERTs, not one giant multi-row statement — see the per-statement limits note in Step 1. Confirm the current D1 limits with the Docs MCP.

## Phase 4: Code Transform

**Goal:** Replace all platform SDK calls with Cloudflare service calls.

### 4a. Delete platform scaffolding

Every platform generates boilerplate files that need to go. The full per-platform delete list is in [platform-patterns.md](references/platform-patterns.md).

Two things to check before deleting:

- **Virtual modules (Base44 only):** `@base44/vite-plugin` generates `@/entities/*` and `@/integrations/*` at build time — these imports have no files on disk. Creating real wrapper files is required. See platform-patterns.md → "Virtual modules".
- **App entry point:** `App.jsx` (or `app/layout.tsx`) contains platform-specific routing and auth scaffolding that breaks once the SDK is removed. Rewrite it cleanly rather than patching. See platform-patterns.md → "App entry point rewrite".

### 4b. Create the frontend API client

Create `src/api/client.js` — the single file that replaces the entire platform SDK on the frontend. Full template in [cloudflare-services.md](references/cloudflare-services.md) → "Frontend API Client".

Then create thin entity files so existing import paths keep working:
- `src/entities/X.js` → re-exports from `db.x` in the client
- `src/integrations/Core.js` → re-exports `uploadFile`, `invokeAI`, etc.

### 4c. Update SDK calls in components

Replace platform SDK calls with the new client. Per-platform find-and-replace table: [platform-patterns.md](references/platform-patterns.md).

### 4d. Generate Pages Functions

One function per backend service. Templates for D1 CRUD, R2 upload, Workers AI, auth: [cloudflare-services.md](references/cloudflare-services.md).

**Read-your-own-writes:** D1 reads can hit a replica that hasn't caught up to a just-committed write (e.g. sign up → immediately redirect → "user not found"). Where a write is followed by a read that depends on it, use the **D1 Sessions API** to pass a bookmark between them. See the `workers-best-practices` skill.

## Phase 5: Build & Deploy

The build/deploy path depends on what the frontend is.

**SPA path (Base44 / Lovable / Bolt — Vite → static `dist/`):**

1. **Clean `package.json`** — remove platform SDKs and unused deps
2. **Audit UI components** — scaffold generates 30–50 shadcn/ui components; delete unused ones
3. **Add SPA fallback** — `public/_redirects`: `/* /index.html 200`
4. **Build:** `npm run build` — resolve any import errors before continuing
5. **Create the Pages project** (once, before first deploy):
   `npx wrangler pages project create <name> --production-branch=main`
6. **Create Cloudflare resources** — see [cloudflare-services.md](references/cloudflare-services.md) → "Deployment Gotchas" for activation steps, R2 opt-in, and the `--remote` flag for D1 migrations
7. **Deploy:** `npx wrangler pages deploy dist`
8. **Run migrations:** `npx wrangler d1 execute <db> --remote --file=migrations/0001_initial.sql`

**Next.js path (V0):** Don't rewrite Server Actions/route handlers into Pages Functions. Deploy with the official Workers Next.js adapter (`@opennextjs/cloudflare`, formerly `@cloudflare/next-on-pages`) and mark server code for the edge runtime (`export const runtime = 'edge'`). Confirm the current adapter and steps with the Docs MCP (`migrate_pages_to_workers_guide`) and the `wrangler` skill before starting — this path changes faster than the SPA one.

## Phase 6: Verify

1. Test every feature manually
2. `grep -ri "base44\|supabase\|@vercel\|replit" src/ functions/` — should return nothing
3. Check bundle size — should drop significantly after removing unused deps and platform SDKs
4. Audit memory leaks: `URL.revokeObjectURL` for blob URLs, cleanup in `useEffect` returns

## Workers AI Notes

Behavioral gotchas that hold regardless of which model you pick. Full code snippets in [cloudflare-services.md](references/cloudflare-services.md). For the current model catalog and pricing, query the Docs MCP — model names below are illustrative, not recommendations.

- **Response format varies by model.** Some return OpenAI chat format (`choices[0].message.content`), others return `response.response`. Always handle both — see cloudflare-services.md for the wrapper.
- **Disable thinking mode** with `chat_template_kwargs: { thinking: false }`. Reasoning models otherwise spend all tokens on chain-of-thought and return `content: null`. This is the #1 cause of "AI call returns nothing."
- **Raise `max_tokens`.** The default is low and cuts off mid-JSON. Set a generous ceiling for structured responses.
- **Vision input** uses a content array, not a plain string prompt — see cloudflare-services.md.
- **JSON from models** often comes wrapped in markdown fences or `<think>` tags. Strip before parsing — see cloudflare-services.md for the helper.

## Common Pitfalls

1. **CORS on external APIs** — third-party APIs (government data, payment processors) block browser requests. Proxy them through Pages Functions so the request comes from Cloudflare's edge, not the user's browser.
2. **Supabase `anon` key in client code** — Lovable/Bolt apps embed the Supabase anon key in frontend code. This is fine for Supabase (RLS enforces security), but when migrating to D1, you need server-side auth checks in your Pages Functions instead.
3. **SPA routing** — Cloudflare Pages needs `public/_redirects` with `/* /index.html 200` for client-side routing to work.
4. **Blob URL memory leaks** — `URL.createObjectURL()` leaks memory if you don't call `revokeObjectURL()` when done.
5. **PostgreSQL → SQLite gaps** — D1 is SQLite, not PostgreSQL. No `uuid` type, no `JSONB`, no `SERIAL`, no array columns. See schema-mapping.md for conversions.
6. **Hidden system fields** — Base44 entities have `id`, `created_date`, `updated_date`, `created_by` on every record but NOT in the schema definition. Supabase tables may have similar RLS-related columns. Always include them in D1 tables.
