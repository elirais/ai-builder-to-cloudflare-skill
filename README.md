# AI Builder to Cloudflare — Claude Code Skill

Migrate apps from **Base44**, **Lovable**, **Bolt.new**, **V0**, or **Replit** to Cloudflare Pages + Workers AI + D1 + R2.

One skill. Five platforms. Full migration — schema, SDK calls, infrastructure, deploy.

## Install

```bash
npx skills add elirais/ai-builder-to-cloudflare-skill
```

Once installed, the skill loads automatically when you ask Claude to migrate — nothing else to configure.

## Use

```
> Migrate this Lovable app to Cloudflare
> I want to get off Base44
> Replace Supabase with Cloudflare D1 and R2
```

The skill auto-detects your platform from imports and walks through 6 phases: detect → discover → infrastructure → schema → code transform → deploy.

## What's Inside

- **5 platform SDKs** — exact migration patterns for Base44, Supabase (Lovable/Bolt), Next.js (V0), Replit
- **Schema conversion** — MongoDB + PostgreSQL → D1 SQLite
- **Workers AI gotchas** — thinking mode, vision format, response parsing
- **Ready-to-use templates** — Pages Functions for AI, CRUD, uploads, auth, CORS

## License

MIT — [Leonid Rise](https://www.linkedin.com/in/leonid-rise/)
