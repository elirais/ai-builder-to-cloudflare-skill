# AI Builder App to Cloudflare Skill

A general method for migrating an app off **any** hosted platform — AI app builders, no-code/low-code tools, or hosted backends — and onto Cloudflare (Pages/Workers + D1 + R2 + Workers AI).

A repeatable method that works for a application source:

> **Acquire → Analyze → Classify → Plan → Provision → Transform → Migrate data → Verify**

## Install

```bash
npx skills add elirais/ai-builder-to-cloudflare-skill
```

Loads automatically when you ask Claude to migrate — nothing to configure.

## Use

```
> Migrate this app to Cloudflare
> I want to get off <platform>
> Replace our hosted backend with Cloudflare D1 and R2
```

The skill analyzes what the app does and what it runs on, decides what to reuse vs rewrite, designs the Cloudflare target, plans it with you, then executes and verifies.

## How it works

- **Source-agnostic.** Builds a real dependency inventory instead of assuming a template — so it handles platforms it's never seen.
- **Reuse-aware.** Buckets the codebase into keep / adapt / replace so you don't rewrite what already works.
- **Plans before it builds.** Produces a migration plan for sign-off before provisioning anything.
- **Defers to Cloudflare's own docs & skills** for current limits, pricing, models, and APIs — nothing stale baked in.
- **Known-platform playbooks** (Base44, Lovable, Bolt, V0, Replit, Supabase…) give a head start when the source is recognized — as accelerators, not the structure.

## License

MIT — [Leonid Rise](https://www.linkedin.com/in/leonid-rise/)
