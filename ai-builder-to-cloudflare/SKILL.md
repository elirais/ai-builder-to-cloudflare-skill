---
name: ai-builder-to-cloudflare
description: Migrate apps built on AI app builder platforms (Base44, Lovable, Bolt.new, V0, Replit) to Cloudflare Pages + Workers AI + D1 + R2. Use this skill whenever the user mentions migrating from Base44, Lovable, Bolt, V0, or Replit, wants to leave an AI app builder, escape vendor lock-in, move to Cloudflare, self-host an AI-generated app, or replace Supabase with Cloudflare services. Also trigger when the user has a codebase with @base44/sdk, @supabase/supabase-js, or Vercel-specific imports and wants more control, better performance, or lower costs.
---

# AI App Builder to Cloudflare Migration

Migrate any app built on an AI app builder platform to Cloudflare Pages + Workers AI + D1 + R2. Covers Base44, Lovable, Bolt.new, V0, and Replit — the five most common platforms people want to migrate from.

All these platforms generate React + Tailwind apps. The frontend is usually fine — the migration is really about **replacing the backend services** (database, auth, AI, file storage) with Cloudflare equivalents you own and control.

## Why Migrate

AI app builders trade control for speed. That tradeoff stops making sense when:
- You hit **scaling limits** (Base44's 100-user concurrent cap, Supabase free tier row limits)
- You're paying **platform fees** for infrastructure that costs pennies on Cloudflare
- The platform **injects tracking/branding** into your app (Base44 adds a 231KB badge script to every page load)
- You want **edge performance** — Cloudflare Pages serves from 300+ global PoPs vs a single origin server
- You need **AI at the edge** — Workers AI runs inference close to users instead of routing to a central API

Real-world results from a production migration: 48% faster TTFB, 71% smaller payload, AI inference at $0.0003/call.

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

**General rule:** Export every table as INSERT statements into `migrations/0002_seed.sql`. For tables with more than 500 rows, batch the inserts.

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

| Platform Feature | Cloudflare Replacement | Why It's Better |
|---|---|---|
| Database (Supabase PostgreSQL / Base44 entities) | **D1** (SQLite at edge) | Zero cold starts, globally replicated reads, no connection pooling needed |
| File storage (Supabase Storage / Base44 UploadFile) | **R2** (S3-compatible) | Zero egress fees, global distribution |
| Auth (Supabase Auth / Base44 Auth / NextAuth) | **Cloudflare Access** or D1 + JWT | No third-party dependency, edge-verified |
| AI / LLM calls (OpenAI API / Base44 InvokeLLM) | **Workers AI** | Runs on Cloudflare's GPU fleet, no API key management, $0.10/M tokens |
| Serverless functions (Edge Functions / API routes) | **Pages Functions** | Same V8 isolate model, file-based routing |
| Realtime (Supabase Realtime / .subscribe()) | **Durable Objects + WebSocket** | Complex — flag for manual work and explain what's involved |
| Email (SendGrid / Resend / Base44 SendEmail) | **MailChannels** or Email Workers | Free via Cloudflare |

Generate `wrangler.toml` with only the bindings the app actually needs:

```toml
name = "app-name"
pages_build_output_dir = "dist"
compatibility_date = "2025-06-01"

# Only include bindings for services the app uses:
[ai]
binding = "AI"

# [[d1_databases]]
# binding = "DB"
# database_name = "app-db"
# database_id = "<from: npx wrangler d1 create app-db>"

# [[r2_buckets]]
# binding = "BUCKET"
# bucket_name = "app-files"
```

See [cloudflare-services.md](references/cloudflare-services.md) for setup commands, code templates, and recommended models.

## Phase 3: Schema Migration

**Goal:** Convert the platform's database to D1 SQL.

The conversion depends on the source platform:

- **Base44** → MongoDB-style NoSQL entities → D1 SQL tables. See [schema-mapping.md](references/schema-mapping.md) for type mapping and query translation.
- **Supabase (Lovable/Bolt)** → PostgreSQL → D1 SQL. Most types map directly. Key differences: D1 doesn't support `uuid` type (use `TEXT`), `JSONB` → `TEXT` (use `json_extract()`), no `SERIAL` (use `INTEGER PRIMARY KEY AUTOINCREMENT`).
- **V0/Replit** → Varies by what they generated. Check for Prisma schemas, Drizzle configs, or raw SQL.

The schema mapping reference covers all conversion rules: [schema-mapping.md](references/schema-mapping.md).

Generate migration file: `migrations/0001_initial.sql`

For data export, use the platform's API or admin UI to pull records, then generate `INSERT` statements. For large datasets, batch in groups of 500.

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

## Phase 5: Build & Deploy

1. **Clean `package.json`** — remove platform SDKs and unused deps
2. **Audit UI components** — scaffold generates 30–50 shadcn/ui components; delete unused ones
3. **Add SPA fallback** — `public/_redirects`: `/* /index.html 200`
4. **Build:** `npm run build` — resolve any import errors before continuing
5. **Create the Pages project** (once, before first deploy):
   `npx wrangler pages project create <name> --production-branch=main`
6. **Create Cloudflare resources** — see [cloudflare-services.md](references/cloudflare-services.md) → "Deployment Gotchas" for activation steps, R2 opt-in, and the `--remote` flag for D1 migrations
7. **Deploy:** `npx wrangler pages deploy dist`
8. **Run migrations:** `npx wrangler d1 execute <db> --remote --file=migrations/0001_initial.sql`

## Phase 6: Verify

1. Test every feature manually
2. `grep -ri "base44\|supabase\|@vercel\|replit" src/ functions/` — should return nothing
3. Check bundle size — should drop significantly after removing unused deps and platform SDKs
4. Audit memory leaks: `URL.revokeObjectURL` for blob URLs, cleanup in `useEffect` returns

## Workers AI Notes

Hard-won knowledge from real migrations. Full code snippets in [cloudflare-services.md](references/cloudflare-services.md).

- **Response format varies by model.** Newer models (Gemma 4, Kimi K2.6) return OpenAI chat format; older ones return `response.response`. Always handle both — see cloudflare-services.md for the wrapper.
- **Disable thinking mode** with `chat_template_kwargs: { thinking: false }`. Without this, reasoning models (Gemma 4, Kimi K2.6) spend all tokens on chain-of-thought and return `content: null`. This is the #1 cause of "AI call returns nothing."
- **Set `max_tokens: 2048` minimum.** Default is often 256, which cuts off mid-JSON.
- **Vision input** uses a content array, not a plain string prompt — see cloudflare-services.md.
- **JSON from models** often comes wrapped in markdown fences or `<think>` tags. Strip before parsing — see cloudflare-services.md for the helper.

## Common Pitfalls

1. **CORS on external APIs** — third-party APIs (government data, payment processors) block browser requests. Proxy them through Pages Functions so the request comes from Cloudflare's edge, not the user's browser.
2. **Supabase `anon` key in client code** — Lovable/Bolt apps embed the Supabase anon key in frontend code. This is fine for Supabase (RLS enforces security), but when migrating to D1, you need server-side auth checks in your Pages Functions instead.
3. **SPA routing** — Cloudflare Pages needs `public/_redirects` with `/* /index.html 200` for client-side routing to work.
4. **Blob URL memory leaks** — `URL.createObjectURL()` leaks memory if you don't call `revokeObjectURL()` when done.
5. **PostgreSQL → SQLite gaps** — D1 is SQLite, not PostgreSQL. No `uuid` type, no `JSONB`, no `SERIAL`, no array columns. See schema-mapping.md for conversions.
6. **Hidden system fields** — Base44 entities have `id`, `created_date`, `updated_date`, `created_by` on every record but NOT in the schema definition. Supabase tables may have similar RLS-related columns. Always include them in D1 tables.
