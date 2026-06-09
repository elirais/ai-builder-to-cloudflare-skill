---
name: ai-builder-to-cloudflare
description: A general, source-agnostic method for migrating an app off a hosted app-builder, no-code/low-code, or AI app-builder platform onto Cloudflare (Pages/Workers + D1 + R2 + Workers AI). Works for any source platform — acquire the code and data, analyze what the app does and the infrastructure it depends on, decide what to reuse vs rewrite, design and plan the Cloudflare target, then execute and verify. Use whenever the user wants to leave a platform, escape vendor lock-in, self-host an app, replace a hosted backend (Supabase, Firebase, etc.) with Cloudflare, or move a Base44 / Lovable / Bolt.new / V0 / Replit (or any other) app to Cloudflare. Also trigger when a codebase contains a proprietary platform SDK (e.g. @base44/sdk), @supabase/supabase-js, firebase, or Vercel-specific imports and the user wants more control, performance, or lower cost.
---

# Migrate Any App to Cloudflare

A repeatable method for moving an app off whatever platform it was built on — an AI app builder, a no-code/low-code tool, or any hosted backend — and onto Cloudflare infrastructure you own.

**This is a method, not a recipe.** There are countless source platforms and countless app shapes; don't assume you know the app before you've analyzed it. The flow is always the same:

> **Acquire → Analyze → Classify → Plan → Provision → Transform → Migrate data → Verify**

Named platforms (Base44, Lovable, Bolt, V0, Replit, Supabase, Firebase…) only change the *details* of the Analyze and Transform steps. When the source is one you recognize, a [known-platform playbook](#known-platform-playbooks) gives you a head start — but you still run the full method, because every app deviates from its platform's defaults.

## Operating principles

- **Analyze before you act.** Discover the actual app — its features, its real dependencies — rather than pattern-matching to a template.
- **Don't trust frozen facts.** Limits, pricing, model names, binding syntax, and adapter names change. Look them up live with the **Cloudflare Docs MCP** (`search_cloudflare_documentation`, `migrate_pages_to_workers_guide`).
- **Lean on Cloudflare's own skills** for depth — don't reimplement them:
  - `wrangler` — CLI, config, deploy
  - `workers-best-practices` — D1 batching, transactions, Sessions API, edge patterns
  - `durable-objects` — realtime / stateful / WebSocket
  - `cloudflare-email-service` — transactional email
  - `agents-sdk`, `sandbox-sdk`, `web-perf` — as the app requires
- **Plan, then get sign-off, then build.** Phase 4 produces a written plan; confirm it with the user before provisioning anything.

---

## Phase 1: Acquire the code and the data

You need both: the source code and a snapshot of the live data.

**Code** — try in order of fidelity:
1. **Platform connector / MCP**, if one is wired up — read files and schemas directly.
2. **Git sync** — most platforms push to a GitHub/GitLab repo; clone it.
3. **Export / download** — "Export project" or "Download code" → unzip locally.
4. **Last resort** — reconstruct from the running app (view source, bundle, network calls). Note the fidelity loss in the plan.

**Data** — the app's records live in the platform's backend; you need them to seed Cloudflare:
- Platform connector/MCP query, export API, database dump, or admin-UI CSV export — whatever the platform offers.
- Capture the schema *and* the rows. Note any binary/blob data stored inline (see Phase 7).

Snapshot both before changing anything.

---

## Phase 2: Analyze — understand the app

Produce **two inventories**. Build them from the code, and from clicking through the running app.

### A. Capability inventory — what the app *does*
List every user-facing feature and flow (e.g. "user signs up", "uploads an avatar", "AI summarizes a note", "live presence on a board"). This is your parity checklist for Phase 8 — the migration is done when every item still works.

### B. Infrastructure & dependency inventory — what it *runs on*
For an unknown platform, discover dependencies generically rather than guessing:
- **Manifest & lockfile** — `package.json`, requirements, etc. List every dependency and flag the vendor/SDK ones.
- **Imports** — grep for SDK and client imports across the source.
- **Config & env** — config files, `.env`, vendor directories (`supabase/`, `.vercel/`, `base44/`, `.replit`, `firebase.json`…), build config.
- **Runtime shape** — SPA? SSR/SSG? A server? Edge functions? (Determines the build/deploy path in Phase 6.)
- **Runtime network calls** — what the running app actually talks to.

For each, classify the capability behind it:

| Capability | What to look for |
|---|---|
| **Database / persistence** | ORM, query builder, `.from()/.find()/.query()` calls, schema files |
| **Auth** | login/signup, sessions, OAuth, JWT, RLS, identity SDK |
| **File / blob storage** | upload calls, storage buckets, CDN URLs, base64 columns |
| **AI / LLM** | model SDKs, `invokeLLM`/chat/completions, vision, embeddings |
| **Realtime** | subscriptions, WebSockets, presence, pub/sub |
| **Email / notifications** | mail SDKs, push, SMS |
| **Background work** | cron, queues, scheduled/async jobs |
| **Third-party APIs** | external HTTP calls (watch for CORS — must move server-side) |
| **Platform lock-in surface** | proprietary glue that only runs on the source platform (custom runtimes, virtual modules, injected agents, hosted-only services) |

The **lock-in surface** is the most important row — it's what *must* be replaced, and it's where each platform hides its surprises (e.g. build-time virtual modules with no files on disk, injected editor/analytics components, server primitives that don't exist off-platform).

---

## Phase 3: Classify — reuse vs rewrite

Bucket every part of the codebase:

- **Portable — keep as-is.** UI components, styling, routing, client-side business logic, pure functions. Usually the majority of the app.
- **Adaptable — keep the logic, swap the calls.** Code coupled to a vendor SDK but otherwise sound. Hide it behind a thin abstraction (Phase 6) so the components stop importing the SDK directly.
- **Locked — must replace.** Anything that only runs on the source platform: the lock-in surface from Phase 2, plus server code in a runtime Cloudflare doesn't offer.

Output a **reuse map**: for each module, the bucket and the action. This is what makes the migration estimable and keeps you from rewriting things that already work.

---

## Phase 4: Map to Cloudflare and write the plan

For each capability from Phase 2B, choose the best-match Cloudflare primitive *for this app's actual shape* — not by default. Starting candidates (confirm current fit + limits via the Docs MCP):

| Capability | Cloudflare candidate | Decide with the app in mind |
|---|---|---|
| Database | **D1** (SQLite); **KV** for simple key/value; **Durable Objects** for per-entity state | D1 for relational; KV for config/sessions; DO for coordination |
| File / blob storage | **R2** | move inline blobs out of the DB |
| Auth — internal/admin | **Cloudflare Access** (Zero Trust) | team gating only, *not* consumer signup |
| Auth — public B2C | **Auth.js / Lucia / Better Auth** + D1 adapter | edge-native consumer auth |
| AI / LLM | **Workers AI** | check current model catalog + pricing via docs |
| Server logic / APIs | **Pages Functions** or **Workers** | match the runtime shape from Phase 2 |
| Realtime | **Durable Objects** (or **PartyKit** over DO) | use the `durable-objects` skill |
| Email | **Email Workers / MailChannels** | use the `cloudflare-email-service` skill |
| Background work | **Cron Triggers** + **Queues** | |
| Static frontend | **Pages / Workers Assets** | |

Then **write the migration plan** and present it to the user before building:
- Target architecture (services + bindings)
- The reuse map (keep / adapt / replace)
- Data migration approach (schema conversion, row counts, blob handling)
- Work breakdown and sequence
- Risks & open decisions that need the user (auth model, realtime, anything with no clean 1:1 mapping)
- Which Cloudflare skills you'll lean on for each piece

Get sign-off. Don't provision until the plan is agreed.

---

## Phase 5: Provision infrastructure

Create the resources from the agreed plan (D1 databases, R2 buckets, KV namespaces, Workers AI binding, Queues…) and wire bindings in `wrangler` config. Defer exact commands, config format, and activation quirks (e.g. services that need a one-time dashboard opt-in) to the **`wrangler` skill** and the **Docs MCP**. See [cloudflare-services.md](references/cloudflare-services.md) → "Deployment Gotchas".

---

## Phase 6: Transform the code

1. **Introduce an abstraction layer.** Add a thin client (e.g. `src/api/client.js`) the app calls instead of the vendor SDK; back it with your own Pages Functions / Workers. This is what turns "adaptable" code into portable code. Template: [cloudflare-services.md](references/cloudflare-services.md).
2. **Remove the lock-in surface.** Delete platform scaffolding; replace virtual modules / injected components / hosted-only primitives with real files. (Known-platform specifics: [platform-patterns.md](references/platform-patterns.md).)
3. **Rewrite server logic** as Pages Functions or Workers, one per backend capability.
4. **Honor the runtime shape.** A static SPA builds to assets + a SPA fallback; an SSR/Next.js app uses the official Cloudflare adapter and edge runtime — don't force one into the other. Confirm the current adapter/path via the Docs MCP and `wrangler` skill.
5. **Move third-party calls server-side** to dodge CORS and hide keys.
6. **Watch edge consistency.** Where a write is immediately read back, use the **D1 Sessions API**; rewrite interactive transactions/RPCs as `db.batch()`. See `workers-best-practices`.

---

## Phase 7: Migrate the data

- **Convert the schema** to the target store. For SQL→D1 (SQLite) and NoSQL→D1 conversion rules: [schema-mapping.md](references/schema-mapping.md).
- **Move the rows.** Generate seed migrations as individual/small-batch INSERTs (a single giant multi-row INSERT exceeds D1's per-statement/parameter limits) or use D1's file/CSV import for large tables. Confirm limits via the Docs MCP.
- **Move blobs out of the DB.** Inline base64/binary columns blow past per-row caps — upload to R2, store only the key in D1.
- Preserve hidden system fields (ids, created/updated timestamps, ownership) even when they weren't in the visible schema.

---

## Phase 8: Verify

1. **Functional parity** — walk the Phase 2A capability inventory; every feature works.
2. **No leftover lock-in** — grep the source for the old platform's SDKs, clients, and URLs; should return nothing.
3. **Performance** — bundle size dropped after removing the platform SDK; use the `web-perf` skill if needed.
4. **Edge correctness** — read-after-write consistency, blob URLs cleaned up, no memory leaks.

---

## Known-platform playbooks

If the source is one of these, jump to its playbook for a head start on Phases 2, 3, and 6 — then return to the method (every app deviates from platform defaults):

- **Base44** — deep SDK surface in [base44-sdk.md](references/base44-sdk.md); transform specifics (virtual modules, app-entry rewrite) in [platform-patterns.md](references/platform-patterns.md).
- **Lovable / Bolt.new** (Supabase) — [platform-patterns.md](references/platform-patterns.md).
- **V0** (Next.js App Router) — [platform-patterns.md](references/platform-patterns.md); deploy via the official Cloudflare Next.js adapter.
- **Replit** — [platform-patterns.md](references/platform-patterns.md).
- **Any other platform** — no playbook needed; the method above is sufficient. Build the dependency inventory in Phase 2B and proceed.

## Reference index

- [cloudflare-services.md](references/cloudflare-services.md) — Cloudflare service code templates (D1 CRUD, R2, Workers AI, auth), the frontend API client, and deployment gotchas.
- [schema-mapping.md](references/schema-mapping.md) — database conversion rules to D1.
- [platform-patterns.md](references/platform-patterns.md) — known-platform shortcuts (delete lists, SDK→Cloudflare call maps, transform specifics).
- [base44-sdk.md](references/base44-sdk.md) — a worked example of fully mapping one platform's SDK surface.
