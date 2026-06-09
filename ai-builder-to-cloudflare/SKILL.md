---
name: ai-builder-to-cloudflare
description: A general, source-agnostic method for migrating an app off any hosted platform — AI app builders, no-code/low-code tools, or hosted backends — onto Cloudflare (Pages/Workers + D1 + R2 + Workers AI). Works for any source platform. Use whenever the user wants to leave a platform, escape vendor lock-in, self-host an app, replace a hosted backend (Supabase, Firebase, etc.) with Cloudflare, or move a Base44 / Lovable / Bolt.new / V0 / Replit (or any other) app to Cloudflare. Also trigger when a codebase contains a proprietary platform SDK, @supabase/supabase-js, firebase, or Vercel-specific imports and the user wants more control, performance, or lower cost.
---

# Migrate Any App to Cloudflare

A repeatable method for moving an app off whatever platform it was built on and onto Cloudflare infrastructure you own.

**This is a method, not a recipe.** There are countless source platforms and app shapes — don't assume you know the app before you've analyzed it.

> **Acquire → Analyze → Classify → Plan → Provision → Transform → Migrate data → Verify**

Named platforms (Base44, Lovable, Bolt, V0, Replit, Firebase…) only change the *details* of the Analyze and Transform steps. The method handles a platform you've never seen just as well as a familiar one.

## Operating principles

- **Analyze before you act.** Discover the actual app first.
- **Don't trust frozen facts.** Limits, pricing, model names, binding syntax change. Look them up live:
  - **Cloudflare Docs MCP** (`search_cloudflare_documentation`, `migrate_pages_to_workers_guide`) — current limits, APIs, model catalog, pricing.
  - **Official Cloudflare npm packages** — `wrangler`, `@cloudflare/workers-types`, platform adapters — as the source of code patterns, not any template in this skill.
- **Lean on Cloudflare's own skills** for depth:
  - `wrangler` — CLI, config, deploy
  - `workers-best-practices` — D1 batching, transactions, Sessions API, edge patterns
  - `durable-objects` — realtime / stateful / WebSocket
  - `cloudflare-email-service` — transactional email
  - `agents-sdk`, `sandbox-sdk`, `web-perf` — as the app requires
- **Plan, then get sign-off, then build.** Phase 4 produces a written plan; confirm before provisioning anything.

---

## Phase 1: Acquire the code and the data

You need both before touching anything.

**Code** — try in order:
1. Platform MCP/connector — read files and schemas directly if wired up.
2. Git sync — most platforms push to GitHub/GitLab; clone it.
3. Export/download — "Export project" or "Download code" zip.
4. Last resort — reconstruct from the running app. Note fidelity loss.

**Data** — the app's records live in the platform's backend:
- Platform MCP query, export API, database dump, or admin CSV export.
- Capture schema *and* rows. Flag any binary/blob data stored inline (relevant in Phase 7).

---

## Phase 2: Analyze — understand the app

Produce two inventories from the code and the running app.

### A. Capability inventory — what the app *does*
List every user-facing feature and flow. This becomes the parity checklist for Phase 8.

### B. Infrastructure & dependency inventory — what it *runs on*

Discover generically — don't assume:
- **Manifest & lockfile** — list every dependency, flag vendor/SDK ones.
- **Imports** — grep for SDK and client imports.
- **Config & env** — `.env`, vendor directories, build config.
- **Runtime shape** — SPA? SSR/SSG? Server? Edge functions? (Determines the deploy path in Phase 6.)
- **Network calls** — what the running app actually talks to.

Classify each dependency by capability:

| Capability | What to look for |
|---|---|
| Database / persistence | ORM, query builder, schema files, `.from()/.find()/.query()` calls |
| Auth | Login/signup, sessions, OAuth, JWT, identity SDK |
| File / blob storage | Upload calls, storage buckets, CDN URLs, base64 columns |
| AI / LLM | Model SDKs, chat/completions, vision, embeddings |
| Realtime | Subscriptions, WebSockets, presence, pub/sub |
| Email / notifications | Mail SDKs, push, SMS |
| Background work | Cron, queues, scheduled/async jobs |
| Third-party APIs | External HTTP calls (CORS — must move server-side) |
| **Lock-in surface** | **Proprietary glue that only runs on the source platform** — custom runtimes, virtual modules, injected agents, hosted-only services |

The **lock-in surface** row is the most important — it's what *must* be replaced and where every platform hides its surprises.

---

## Phase 3: Classify — reuse vs rewrite

Bucket every part of the codebase:

- **Keep as-is** — UI components, styling, routing, client-side business logic, pure functions. Usually the majority.
- **Adapt** — logic coupled to a vendor SDK but otherwise sound. Swap the calls, keep the logic.
- **Replace** — the lock-in surface; anything that only runs on the source platform.

Produce a **reuse map**: for each module, the bucket and the action. This is what makes the migration estimable and prevents rewriting things that already work.

---

## Phase 4: Map to Cloudflare and write the plan

For each capability from Phase 2B, choose the best-match Cloudflare primitive for *this app's actual shape* — not by default. Query the Docs MCP for current capabilities, limits, and pricing before deciding.

| Capability | Cloudflare candidate |
|---|---|
| Relational database | **D1** (SQLite at the edge) |
| Simple key/value | **KV** |
| Per-entity coordination | **Durable Objects** |
| File / blob storage | **R2** |
| Auth — internal/admin | **Cloudflare Access** (Zero Trust — not for consumer signup) |
| Auth — public B2C | **Auth.js / Lucia / Better Auth** with a D1 adapter |
| AI / LLM | **Workers AI** — query Docs MCP for current models + pricing |
| Server logic / APIs | **Pages Functions** or **Workers** |
| Realtime | **Durable Objects** / PartyKit — use the `durable-objects` skill |
| Email | **Email Workers** — use the `cloudflare-email-service` skill |
| Background work | **Cron Triggers** + **Queues** |
| Static frontend | **Pages / Workers Assets** |

Then **write the migration plan** and present it before building:
- Target architecture and bindings
- Reuse map (keep / adapt / replace)
- Data migration approach
- Work breakdown and sequence
- Risks and decisions that need the user
- Which Cloudflare skills cover each piece

---

## Phase 5: Provision infrastructure

Create the agreed resources and wire bindings in `wrangler` config. Use the **`wrangler` skill** for current CLI commands and config format.

**First-time deploy gotchas** — these are not well-surfaced in the docs:
- Create the Pages project before first deploy: `wrangler pages project create <name> --production-branch=main` — `wrangler pages deploy` will fail with "Project not found" without this.
- **R2 requires manual activation** in the Cloudflare Dashboard before any bucket can be created or any R2 binding deployed. If R2 isn't activated yet, comment it out of `wrangler` config, deploy without it, add it back after activation.
- Always use `--remote` with `wrangler d1 execute` — the default runs against a local in-memory DB, not production.

---

## Phase 6: Transform the code

1. **Introduce an abstraction layer.** Replace direct SDK imports with a thin client (`src/api/client.js` or similar) that calls your own Pages Functions / Workers. This converts "adapt" code to portable code without touching every component. Use the official `@cloudflare/workers-types` package and Cloudflare's Workers/Pages docs for code patterns.
2. **Remove the lock-in surface.** Delete platform scaffolding, replace virtual modules or injected components with real files, rewrite hosted-only primitives as Workers/Pages Functions.
3. **Rewrite server logic** as Pages Functions or Workers, one per capability.
4. **Honor the runtime shape:**
   - Static SPA → build to assets, add `public/_redirects: /* /index.html 200`.
   - SSR / Next.js → use the official Cloudflare Next.js adapter (`@opennextjs/cloudflare`), mark server code `export const runtime = 'edge'`. Confirm current adapter via Docs MCP — don't force it into an SPA deploy.
5. **Move third-party calls server-side** to avoid CORS and hide credentials.
6. **Edge consistency:**
   - Write followed immediately by a dependent read → use the **D1 Sessions API** to pass a bookmark. See `workers-best-practices`.
   - Interactive transactions / RPCs → rewrite as `db.batch()` with statements prepared upfront.

---

## Phase 7: Migrate the data

- **Convert the schema** to the target store. For SQL-to-D1 and NoSQL-to-SQL conversion, query the Docs MCP for current D1 type support and constraints.
- **Seed the rows.** Generate individual or small-batch INSERTs — a single large multi-row INSERT can exceed D1's per-statement size and bound-parameter limits. For large tables use D1's file/CSV import. Confirm current limits via the Docs MCP.
- **Move blobs out of the DB.** Inline base64/binary columns exceed D1's per-row cap — upload to R2, store only the object key in D1.
- Preserve hidden system fields (IDs, timestamps, ownership) even when they weren't in the visible schema.

---

## Phase 8: Verify

1. **Functional parity** — walk every item in the Phase 2A capability inventory.
2. **No leftover lock-in** — grep for old platform SDK imports and URLs; should return nothing.
3. **Performance** — bundle size dropped after removing the platform SDK. Use the `web-perf` skill if needed.
4. **Edge correctness** — read-after-write consistency, no blob URL leaks, no memory leaks.
