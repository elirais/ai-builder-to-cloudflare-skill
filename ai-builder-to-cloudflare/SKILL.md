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

Every platform generates boilerplate files that need to go. Common ones:

**Base44:** `src/api/base44Client.js`, `src/api/entities.js`, `base44/` directory, `src/components/VisualEditAgent.jsx`, `src/components/NavigationTracker.jsx`, `src/pages.config.js`, `src/lib/app-params.js`

**Lovable/Bolt (Supabase):** `src/integrations/supabase/client.ts`, `src/integrations/supabase/types.ts`, `supabase/` directory (Edge Functions — migrate the logic, not the files)

**V0:** `.vercel/`, `vercel.json`, Next.js-specific API routes in `app/api/` (rewrite as Pages Functions)

**Replit:** `.replit`, `replit.nix`, Replit-specific DB imports

#### Watch out: virtual modules vs real files

Some platforms generate SDK imports that **don't exist as real files** — they're resolved at build time by a Vite or webpack plugin. If you delete the plugin and the import path still doesn't exist on disk, the build will fail with "Cannot resolve module."

Check for this before assuming an import is a real file:
```bash
# Does this module actually exist on disk?
find src/ -name "Lesson.*" -o -name "entities.*" -o -name "base44Client.*"

# Or grep for the Vite plugin that generates them
grep -r "vite-plugin\|virtualModule\|resolve.*virtual" vite.config.*
```

**Base44** is the main offender: `@base44/vite-plugin` generates all `@/entities/*` and `@/integrations/*` as virtual modules. They have no files on disk. Fix: create real `src/entities/Lesson.js` etc. that import from your new `src/api/client.js`.

**Lovable/Bolt** use real files for Supabase (`src/integrations/supabase/client.ts` exists). No virtual module issue.

**V0/Next.js** uses real files throughout. No virtual module issue.

#### Rewrite the app entry point

Every platform injects routing + auth scaffolding into `src/App.jsx` (or `src/main.tsx`, `app/layout.tsx`) that breaks once the platform SDK is removed. Don't try to patch it — rewrite it cleanly.

The pattern for a React + react-router app (Base44, Lovable, Bolt):

```jsx
// src/App.jsx — clean version after platform scaffolding removed
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom'
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClientInstance } from '@/lib/query-client'
import { Toaster } from '@/components/ui/toaster'
import Layout from './Layout'
import Home from './pages/Home'
import Dashboard from './pages/Dashboard'
// ... other pages

function App() {
  return (
    <QueryClientProvider client={queryClientInstance}>
      <Router>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/Dashboard" element={<Layout><Dashboard /></Layout>} />
          {/* add remaining pages */}
        </Routes>
      </Router>
      <Toaster />
    </QueryClientProvider>
  )
}
export default App
```

Remove: `VisualEditAgent`, `NavigationTracker`, `AuthContext`, `pagesConfig`, any `isLoadingPublicSettings` / `navigateToLogin` logic — these are all platform-specific and have no equivalent on Cloudflare.

### 4b. Create the API client

Replace platform SDK calls with `fetch()` calls to your own Pages Functions:

```javascript
// src/api/client.js — thin wrapper around your Pages Functions

export async function query(endpoint, options = {}) {
  const res = await fetch(`/api/${endpoint}`, {
    method: options.method || 'GET',
    headers: options.body ? { 'Content-Type': 'application/json' } : {},
    body: options.body ? JSON.stringify(options.body) : undefined,
  });
  if (!res.ok) throw new Error(`API ${endpoint}: ${res.status}`);
  return res.json();
}

export const db = {
  list:   (table) => query(`entities/${table}`),
  get:    (table, id) => query(`entities/${table}/${id}`),
  create: (table, data) => query(`entities/${table}`, { method: 'POST', body: data }),
  update: (table, id, data) => query(`entities/${table}/${id}`, { method: 'PUT', body: data }),
  delete: (table, id) => query(`entities/${table}/${id}`, { method: 'DELETE' }),
};

export const ai = {
  run: (prompt, opts) => query('ai', { method: 'POST', body: { prompt, ...opts } }),
};

export const files = {
  upload: async (file) => {
    const form = new FormData();
    form.append('file', file);
    const res = await fetch('/api/upload', { method: 'POST', body: form });
    if (!res.ok) throw new Error(`Upload: ${res.status}`);
    return res.json();
  },
};
```

### 4c. Transform SDK calls in components

Read [platform-patterns.md](references/platform-patterns.md) for the exact find-and-replace patterns for each platform. The general pattern:

| Platform Call | Cloudflare Replacement |
|---|---|
| `Entity.list()` / `supabase.from('x').select()` | `db.list('x')` |
| `Entity.create(data)` / `supabase.from('x').insert(data)` | `db.create('x', data)` |
| `InvokeLLM({prompt})` / `openai.chat.completions.create()` | `ai.run(prompt)` |
| `UploadFile({file})` / `supabase.storage.upload()` | `files.upload(file)` |
| `auth.me()` / `supabase.auth.getUser()` | `fetch('/api/auth/me')` |

### 4d. Generate Pages Functions

Create a Pages Function for each backend service. Use the code templates in [cloudflare-services.md](references/cloudflare-services.md) — it has ready-to-use handlers for Workers AI, D1 CRUD, R2 uploads, and auth.

## Phase 5: Build & Deploy

1. **Clean `package.json`** — remove platform SDKs (`@base44/sdk`, `@supabase/supabase-js`, `@vercel/*`, etc.) and unused deps
2. **Audit UI components** — AI builders scaffold 30-50 shadcn/ui components; most are unused. Delete what's not imported.
3. **Add SPA fallback** — create `public/_redirects` with `/* /index.html 200`
4. **Build:** `npm run build` — fix any import errors before proceeding
5. **Create the Pages project** (required before first deploy — this is separate from deploying):
   ```bash
   npx wrangler pages project create <project-name> --production-branch=main
   ```
6. **Activate Cloudflare services that need manual opt-in:**
   - **R2** is not enabled by default. If your app uses file uploads/storage, go to [Cloudflare Dashboard → R2](https://dash.cloudflare.com/?to=/:account/r2) and click **Enable R2** before running `wrangler r2 bucket create`. Skipping this causes error 10042 and blocks deployment if the R2 binding is in `wrangler.toml`. If R2 isn't ready yet, comment out the `[[r2_buckets]]` binding and deploy without it — images will load from their original URLs until you're ready.
   - **Workers AI** — available by default, no activation needed.
   - **D1** — available by default, no activation needed.
7. **Create D1 database and R2 bucket:**
   ```bash
   npx wrangler d1 create <db-name>         # copy the database_id into wrangler.toml
   npx wrangler r2 bucket create <bucket-name>  # only after R2 is activated
   ```
8. **Deploy:** `npx wrangler pages deploy dist`
9. **Run migrations:**
   ```bash
   npx wrangler d1 execute <db-name> --remote --file=migrations/0001_initial.sql
   npx wrangler d1 execute <db-name> --remote --file=migrations/0002_seed.sql
   ```

## Phase 6: Verify

1. Test every feature manually
2. `grep -ri "base44\|supabase\|@vercel\|replit" src/ functions/` — should return nothing
3. Check bundle size — should drop significantly after removing unused deps and platform SDKs
4. Audit memory leaks: `URL.revokeObjectURL` for blob URLs, cleanup in `useEffect` returns

## Workers AI Notes

Hard-won knowledge from real migrations — these will save hours of debugging:

- **Response format varies by model.** Newer models (Gemma 4, Kimi K2.6) return OpenAI chat format: `response.choices[0].message.content`. Older models return `response.response`. Always handle both:
  ```javascript
  const content = response.choices?.[0]?.message?.content ?? response.response;
  ```

- **Disable thinking mode** with `chat_template_kwargs: { thinking: false }`. Without this, models with built-in reasoning (Gemma 4, Kimi K2.6) spend all tokens on chain-of-thought and return `content: null`. This is the #1 cause of "my AI call returns nothing" bugs.

- **Set `max_tokens: 2048` minimum.** The default is often 256, which cuts off mid-JSON-object. 2048 gives enough room for structured responses while keeping costs negligible ($0.0006 at Gemma 4 rates).

- **Vision input format:** Use content array, not a plain string:
  ```javascript
  { role: 'user', content: [
    { type: 'image_url', image_url: { url: `data:image/jpeg;base64,${b64}` } },
    { type: 'text', text: 'Describe this image.' },
  ]}
  ```

- **Model selection** — check Cloudflare's model catalog for the latest options. As of writing, good starting points are:
  - **Vision/OCR/multilingual:** `@cf/google/gemma-4-26b-a4b-it` ($0.10/M input) — MoE architecture, only 4B active params so it's fast and cheap
  - **Complex reasoning fallback:** `@cf/moonshotai/kimi-k2.6` ($0.95/M input) — use when the primary model isn't accurate enough
  - **Budget vision:** `@cf/meta/llama-3.2-11b-vision-instruct` ($0.049/M input)

- **JSON parsing:** Models sometimes wrap JSON in markdown fences or thinking tags. Always strip before parsing:
  ```javascript
  function parseJSON(text) {
    if (typeof text !== 'string') return text;
    let s = text.replace(/```(?:json)?\s*/g, '').replace(/```\s*/g, '');
    s = s.replace(/<think>[\s\S]*?<\/think>/g, '').trim();
    return JSON.parse(s);
  }
  ```

## Common Pitfalls

1. **CORS on external APIs** — third-party APIs (government data, payment processors) block browser requests. Proxy them through Pages Functions so the request comes from Cloudflare's edge, not the user's browser.
2. **Supabase `anon` key in client code** — Lovable/Bolt apps embed the Supabase anon key in frontend code. This is fine for Supabase (RLS enforces security), but when migrating to D1, you need server-side auth checks in your Pages Functions instead.
3. **SPA routing** — Cloudflare Pages needs `public/_redirects` with `/* /index.html 200` for client-side routing to work.
4. **Blob URL memory leaks** — `URL.createObjectURL()` leaks memory if you don't call `revokeObjectURL()` when done.
5. **PostgreSQL → SQLite gaps** — D1 is SQLite, not PostgreSQL. No `uuid` type, no `JSONB`, no `SERIAL`, no array columns. See schema-mapping.md for conversions.
6. **Hidden system fields** — Base44 entities have `id`, `created_date`, `updated_date`, `created_by` on every record but NOT in the schema definition. Supabase tables may have similar RLS-related columns. Always include them in D1 tables.
