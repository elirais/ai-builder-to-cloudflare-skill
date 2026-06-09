# Platform-Specific Migration Patterns

Each AI app builder platform has its own SDK and conventions. This reference maps the exact SDK calls to their Cloudflare replacements.

## Table of Contents

- [Base44](#base44)
- [Lovable / Bolt.new (Supabase)](#lovable--boltnew-supabase)
- [V0 (Vercel / Next.js)](#v0-vercel--nextjs)
- [Replit](#replit)

---

## Base44

**Lock-in level: Highest.** The `@base44/sdk` is proprietary — code literally cannot run without Base44's servers. Every entity operation, auth call, and integration goes through their API.

**Identifying markers:** `@base44/sdk` imports, `base44Client.js`, `entities.js` with Proxy-based entity access, `InvokeLLM`, `UploadFile`.

### Files to delete

| File | What it is |
|---|---|
| `src/api/base44Client.js` | SDK singleton — `createClient({ appId })` |
| `src/api/entities.js` | Entity proxy exports — `export const MyEntity = base44.entities.MyEntity` |
| `src/lib/app-params.js` | URL param reader (Base44-specific) |
| `src/components/VisualEditAgent.jsx` | Base44 in-browser editor |
| `src/components/NavigationTracker.jsx` | Base44 analytics tracker |
| `src/pages.config.js` | Base44 routing config (replace with react-router) |
| `base44/functions/` | Deno backend functions (migrate logic to Pages Functions) |

### SDK call replacements

| Base44 SDK | Cloudflare Replacement |
|---|---|
| `base44.entities.X.list(sort, limit, skip)` | `GET /api/entities/x?limit=&offset=&sort=` |
| `base44.entities.X.filter(query)` | `POST /api/entities/x/filter` with MongoDB-style query (translate to SQL) |
| `base44.entities.X.get(id)` | `GET /api/entities/x/:id` |
| `base44.entities.X.create(data)` | `POST /api/entities/x` |
| `base44.entities.X.update(id, data)` | `PUT /api/entities/x/:id` |
| `base44.entities.X.delete(id)` | `DELETE /api/entities/x/:id` |
| `base44.entities.X.bulkCreate([...])` | `POST /api/entities/x/bulk` |
| `base44.entities.X.subscribe(cb)` | Durable Objects + WebSocket (flag for manual work) |
| `integrations.Core.InvokeLLM({prompt, model})` | `POST /api/ai` → `env.AI.run()` |
| `integrations.Core.UploadFile({file})` | `POST /api/upload` → `env.BUCKET.put()` |
| `integrations.Core.SendEmail({to, subject, body})` | `POST /api/email` → MailChannels |
| `integrations.Core.GenerateImage({prompt})` | `POST /api/ai/image` → `env.AI.run('@cf/black-forest-labs/flux-2-klein-9b')` |
| `integrations.Core.ExtractDataFromUploadedFile({file_url})` | `POST /api/ai` with vision model |
| `base44.auth.me()` | `GET /api/auth/me` (Cloudflare Access JWT or custom) |
| `base44.auth.redirectToLogin()` | Cloudflare Access handles this automatically |
| `base44.auth.loginWithProvider('google')` | Cloudflare Access identity provider |
| `base44.analytics.track({eventName})` | `POST /api/analytics` → Workers Analytics Engine or KV |
| `base44.functions.invoke('name', data)` | `POST /api/functions/name` (rewrite Deno → Pages Function) |

### Base44 entity query translation

Base44 uses MongoDB-style queries. These need to become SQL WHERE clauses:

| Base44 Filter | SQL |
|---|---|
| `{ status: "active" }` | `WHERE status = 'active'` |
| `{ amount: { $gt: 100 } }` | `WHERE amount > 100` |
| `{ field: { $in: ["a","b"] } }` | `WHERE field IN ('a','b')` |
| `{ $or: [{a: 1}, {b: 2}] }` | `WHERE (a = 1) OR (b = 2)` |

Full query translation table: see schema-mapping.md.

### Virtual modules — Base44 only

`@base44/vite-plugin` generates `@/entities/*` and `@/integrations/*` as **virtual modules** at build time. These imports have no files on disk. Once you remove the plugin they will fail to resolve.

Check before assuming any import is a real file:
```bash
grep -r "vite-plugin" vite.config.* && find src/ -path "*/entities/*.js" -o -path "*/integrations/*.js"
```

Fix: create a real file for each virtual module that re-exports from your new `src/api/client.js`:
```javascript
// src/entities/Lesson.js
import { db } from '@/api/client';
export const Lesson = db.lessons;

// src/integrations/Core.js
import { uploadFile } from '@/api/client';
export const UploadFile = async ({ file }) => {
  const result = await uploadFile(file);
  return { file_url: result.file_url };
};
```

Repeat for every entity and integration the app imports. Lovable/Bolt/V0/Replit don't have this issue — their imports are real files.

### App entry point rewrite — Base44

Base44 injects `VisualEditAgent`, `NavigationTracker`, `AuthContext`, and `pagesConfig` into `App.jsx`. All of these come from Base44's virtual modules and break once the SDK is removed. Rewrite `App.jsx` cleanly — don't patch it:

```jsx
// src/App.jsx — clean replacement
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom'
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClientInstance } from '@/lib/query-client'
import { Toaster } from '@/components/ui/toaster'
import Layout from './Layout'
// import each page from src/pages/
import Home from './pages/Home'

function App() {
  return (
    <QueryClientProvider client={queryClientInstance}>
      <Router>
        <Routes>
          <Route path="/" element={<Home />} />
          {/* one Route per page in src/pages/ */}
        </Routes>
      </Router>
      <Toaster />
    </QueryClientProvider>
  )
}
export default App
```

Remove: `VisualEditAgent`, `NavigationTracker`, `AuthContext`, `pagesConfig`, `isLoadingPublicSettings`, `navigateToLogin`.

For Lovable/Bolt: same pattern but remove the Supabase `<SessionContextProvider>` wrapper. For V0: `app/layout.tsx` replaces `App.jsx` — remove `<VercelAnalytics />` and `<SpeedInsights />`.

---

## Lovable / Bolt.new (Supabase)

**Lock-in level: Medium.** Both platforms generate standard React + Supabase code. The code is portable, but tightly coupled to Supabase's client library and PostgreSQL features. Supabase is open-source, so you could self-host, but migrating to D1 gives you edge performance and zero egress costs.

**Identifying markers:** `@supabase/supabase-js` imports, `supabaseClient.ts`, `supabase.from('table')` calls, `supabase.auth.*`, `supabase.storage.*`, `supabase/functions/` directory.

### Files to delete or rewrite

| File | Action |
|---|---|
| `src/integrations/supabase/client.ts` | Delete — contains Supabase URL + anon key |
| `src/integrations/supabase/types.ts` | Delete — generated TypeScript types for Supabase schema |
| `src/integrations/supabase/` | Delete entire directory |
| `supabase/functions/` | Migrate logic to `functions/api/` (Supabase Edge Functions → Pages Functions) |
| `supabase/migrations/` | Convert PostgreSQL → SQLite syntax for D1 |
| `.env` / `.env.local` | Remove `SUPABASE_URL`, `SUPABASE_ANON_KEY` — no longer needed |

### SDK call replacements

| Supabase SDK | Cloudflare Replacement |
|---|---|
| `const { data, error } = await supabase.from('x').select('*')` | `GET /api/entities/x` → `env.DB.prepare('SELECT * FROM x')` |
| `.from('x').select('*').eq('col', val)` | `GET /api/entities/x?col=val` → `WHERE col = ?` |
| `.from('x').select('*').order('col', { ascending: false })` | `?sort=-col` → `ORDER BY col DESC` |
| `.from('x').select('*').range(0, 9)` | `?limit=10&offset=0` → `LIMIT 10 OFFSET 0` |
| `.from('x').insert(data).select()` | `POST /api/entities/x` → `INSERT INTO x` |
| `.from('x').update(data).eq('id', id)` | `PUT /api/entities/x/:id` → `UPDATE x SET ... WHERE id = ?` |
| `.from('x').delete().eq('id', id)` | `DELETE /api/entities/x/:id` → `DELETE FROM x WHERE id = ?` |
| `supabase.auth.signUp({email, password})` | `POST /api/auth/signup` (D1 + bcrypt + JWT) |
| `supabase.auth.signInWithPassword({email, password})` | `POST /api/auth/login` |
| `supabase.auth.signInWithOAuth({provider: 'google'})` | Cloudflare Access with Google identity provider |
| `supabase.auth.getUser()` | `GET /api/auth/me` (verify JWT) |
| `supabase.auth.signOut()` | `POST /api/auth/logout` (clear session cookie/KV) |
| `supabase.storage.from('bucket').upload(path, file)` | `POST /api/upload` → `env.BUCKET.put()` |
| `supabase.storage.from('bucket').getPublicUrl(path)` | `/files/:key` → `env.BUCKET.get()` |
| `supabase.channel('x').on('postgres_changes', cb).subscribe()` | Durable Objects + WebSocket (flag for manual work) |
| `supabase.functions.invoke('name', {body})` | `POST /api/functions/name` |

### PostgreSQL → SQLite gotchas

| PostgreSQL | D1 (SQLite) | Notes |
|---|---|---|
| `UUID` | `TEXT` | Generate with `crypto.randomUUID()` in JS |
| `SERIAL` / `BIGSERIAL` | `INTEGER PRIMARY KEY AUTOINCREMENT` | |
| `JSONB` | `TEXT` | Use `json_extract()` for queries |
| `TIMESTAMP WITH TIME ZONE` | `TEXT` | Store ISO 8601 strings |
| `BOOLEAN` | `INTEGER` | 0/1 instead of true/false |
| `TEXT[]` (arrays) | `TEXT` | Store as JSON array string |
| `NOW()` | `datetime('now')` | SQLite function |
| RLS policies | Application-level auth checks | Enforce in Pages Functions middleware |

---

## V0 (Vercel / Next.js)

**Lock-in level: Low-Medium.** V0 generates standard Next.js code. The main lock-in is Next.js-specific features (Server Components, Server Actions, `app/` directory routing) which don't exist in Vite/React. If the app is simple (mostly client components), migration is straightforward. If it uses heavy server-side features, expect more work.

**Identifying markers:** `next.config.js`, `app/` directory with `page.tsx` / `layout.tsx`, `'use server'` directives, `@vercel/*` imports, `vercel.json`.

### Migration strategy

V0 apps fall into two categories:

1. **Client-heavy apps** (most common from V0) — Convert to Vite SPA. Replace `app/` routing with react-router. Move API routes to `functions/api/`.
2. **Server-heavy apps** — Consider keeping Next.js and deploying with `@cloudflare/next-on-pages` instead of a full rewrite. This preserves Server Components while running on Cloudflare.

### Files to delete or rewrite

| File | Action |
|---|---|
| `next.config.js` | Delete (replace with `vite.config.js`) or keep if using next-on-pages |
| `vercel.json` | Delete |
| `.vercel/` | Delete |
| `app/api/` routes | Migrate to `functions/api/` (Pages Functions) |
| `middleware.ts` | Migrate to `functions/_middleware.js` |

### Pattern replacements

| Next.js / V0 | Cloudflare Replacement |
|---|---|
| `app/api/route.ts` with `export async function GET()` | `functions/api/route.js` with `export async function onRequestGet()` |
| `'use server'` actions | Pages Functions (POST endpoints) |
| `useRouter()` from `next/navigation` | `useNavigate()` from `react-router-dom` |
| `<Link href="...">` from `next/link` | `<Link to="...">` from `react-router-dom` |
| `<Image>` from `next/image` | Standard `<img>` with manual optimization |
| `getServerSideProps` | Pages Function that returns data + client fetch |

---

## Replit

**Lock-in level: High.** Replit-generated apps are tightly coupled to Replit's hosting, database, and deployment infrastructure. The code itself is standard (usually Express/Fastify + React), but it assumes Replit's environment for DB connections, secrets, and deployment.

**Identifying markers:** `.replit` config file, `replit.nix`, imports from `@replit/`, `REPL_*` environment variables, Replit Database imports.

### Migration strategy

1. Export the project from Replit (Git clone or download)
2. Replace Replit Database with D1
3. Replace `process.env.REPL_*` with Cloudflare environment bindings
4. Convert Express/Fastify backend to Pages Functions
5. Build frontend with Vite, deploy to Pages

### Pattern replacements

| Replit | Cloudflare Replacement |
|---|---|
| Replit Database (`@replit/database`) | D1 (`env.DB`) |
| `process.env.REPL_SLUG` | Not needed (Cloudflare handles routing) |
| Express routes (`app.get('/api/...')`) | Pages Functions (`functions/api/...`) |
| Replit Auth | Cloudflare Access or custom D1 + JWT |
| Replit Secrets | `wrangler secret put` or `wrangler.toml` vars |
