# Cloudflare Services Reference

Quick reference for the Cloudflare services that replace AI app builder backends. Includes setup commands, code templates, and recommended models.

## Workers AI (replaces InvokeLLM, OpenAI API, GenerateImage, ExtractDataFromUploadedFile)

**Setup (wrangler.toml):**
```toml
[ai]
binding = "AI"
```

**Text generation:**
```javascript
const response = await env.AI.run('@cf/google/gemma-4-26b-a4b-it', {
  messages: [
    { role: 'system', content: 'You are helpful.' },
    { role: 'user', content: prompt },
  ],
  max_tokens: 2048,
  chat_template_kwargs: { thinking: false },
});
// Response: { choices: [{ message: { content: "..." } }] }
const text = response.choices?.[0]?.message?.content;
```

**Vision (image analysis):**
```javascript
const response = await env.AI.run('@cf/google/gemma-4-26b-a4b-it', {
  messages: [{
    role: 'user',
    content: [
      { type: 'image_url', image_url: { url: `data:image/jpeg;base64,${base64}` } },
      { type: 'text', text: 'Describe this image.' },
    ],
  }],
  max_tokens: 2048,
  chat_template_kwargs: { thinking: false },
});
```

**Structured JSON output:** Include a JSON schema description in the system prompt and ask the model to respond with JSON only. Parse with `JSON.parse()` and handle markdown fences.

**Image generation (replaces GenerateImage):**
```javascript
const response = await env.AI.run('@cf/black-forest-labs/flux-2-klein-9b', {
  multipart: { body: formStream, contentType: formContentType },
});
```

**Recommended models:**
| Use Case | Model | Cost |
|---|---|---|
| Text/JSON generation | `@cf/google/gemma-4-26b-a4b-it` | $0.10/M input |
| Vision/OCR | `@cf/google/gemma-4-26b-a4b-it` | $0.10/M input |
| Complex reasoning | `@cf/moonshotai/kimi-k2.6` | $0.95/M input |
| Image generation | `@cf/black-forest-labs/flux-2-klein-9b` | Per-image pricing |

## D1 Database (replaces Supabase PostgreSQL / Base44 Entities / Replit DB)

**Setup (wrangler.toml):**
```toml
[[d1_databases]]
binding = "DB"
database_name = "my-app-db"
database_id = "<from wrangler d1 create>"
```

**Create database:**
```bash
npx wrangler d1 create my-app-db
```

**Run migrations:**
```bash
npx wrangler d1 execute my-app-db --file=migrations/0001_initial.sql
```

**CRUD in Pages Functions:**
```javascript
// List
const { results } = await env.DB.prepare('SELECT * FROM items LIMIT ?').bind(50).all();

// Get by ID
const item = await env.DB.prepare('SELECT * FROM items WHERE id = ?').bind(id).first();

// Create
await env.DB.prepare('INSERT INTO items (id, name) VALUES (?, ?)')
  .bind(crypto.randomUUID(), name).run();

// Update
await env.DB.prepare('UPDATE items SET name = ?, updated_date = datetime("now") WHERE id = ?')
  .bind(name, id).run();

// Delete
await env.DB.prepare('DELETE FROM items WHERE id = ?').bind(id).run();

// Batch (transaction)
await env.DB.batch([
  env.DB.prepare('INSERT INTO items (id, name) VALUES (?, ?)').bind(id1, name1),
  env.DB.prepare('INSERT INTO items (id, name) VALUES (?, ?)').bind(id2, name2),
]);
```

## R2 Object Storage (replaces Supabase Storage / Base44 UploadFile / S3)

**Setup (wrangler.toml):**
```toml
[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-app-files"
```

**Create bucket:**
```bash
npx wrangler r2 bucket create my-app-files
```

**Upload:**
```javascript
await env.BUCKET.put(key, file.stream(), {
  httpMetadata: { contentType: file.type },
});
```

**Download / serve:**
```javascript
const object = await env.BUCKET.get(key);
if (!object) return new Response('Not found', { status: 404 });
return new Response(object.body, {
  headers: { 'Content-Type': object.httpMetadata?.contentType || 'application/octet-stream' },
});
```

**Presigned URLs (replaces CreateFileSignedUrl):**
Requires R2 API token and aws4fetch. See Cloudflare docs for presigned URL generation.

## KV (replaces session storage, simple caches)

**Setup (wrangler.toml):**
```toml
[[kv_namespaces]]
binding = "KV"
id = "<from wrangler kv namespace create>"
```

```javascript
await env.KV.put(key, value, { expirationTtl: 3600 });
const value = await env.KV.get(key);
await env.KV.delete(key);
```

## Pages Functions (replaces Supabase Edge Functions / Base44 Functions / Next.js API Routes / Express)

Pages Functions live in the `functions/` directory. Each file maps to a URL path:

```
functions/
├── api/
│   ├── ai.js              → POST /api/ai
│   ├── upload.js           → POST /api/upload
│   ├── entities/
│   │   └── [name].js       → /api/entities/:name (dynamic route)
│   └── auth/
│       └── me.js           → GET /api/auth/me
```

**Handler signature:**
```javascript
export async function onRequestPost({ request, env, params }) {
  // request: standard Request object
  // env: bindings (AI, DB, BUCKET, KV, etc.)
  // params: dynamic route params (e.g., params.name)
}

export async function onRequestGet({ request, env, params }) { ... }
```

**CORS middleware** (`functions/api/_middleware.js`) — needed for local dev and if your frontend is on a different domain:

```javascript
export async function onRequest({ request, next }) {
  if (request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      },
    });
  }
  const response = await next();
  response.headers.set('Access-Control-Allow-Origin', '*');
  return response;
}
```

## Cloudflare Access (replaces Auth)

For apps that need user authentication, Cloudflare Access provides:
- Login page with multiple identity providers (Google, Microsoft, GitHub, SAML, OIDC)
- JWT verification in Workers/Functions
- No user management code needed

**Verify JWT in a Page Function:**
```javascript
export async function onRequest({ request, env }) {
  const jwt = request.headers.get('Cf-Access-Jwt-Assertion');
  if (!jwt) return new Response('Unauthorized', { status: 401 });
  // Verify and decode JWT to get user identity
  const identity = await verifyAccessJWT(jwt, env.ACCESS_AUD);
  return Response.json({ email: identity.email });
}
```

For simpler apps without Cloudflare Access, implement email/password auth with D1 + bcrypt + JWT.

## Ready-to-Use Pages Function Templates

### AI Endpoint (`functions/api/ai.js`)

Handles text generation and vision. Drop-in replacement for InvokeLLM or OpenAI API calls:

```javascript
export async function onRequestPost({ request, env }) {
  const { prompt, image, system } = await request.json();
  if (!prompt) return Response.json({ error: 'No prompt' }, { status: 400 });

  const messages = [];
  if (system) messages.push({ role: 'system', content: system });

  if (image) {
    const imageUrl = image.startsWith('data:') ? image : `data:image/jpeg;base64,${image}`;
    messages.push({ role: 'user', content: [
      { type: 'image_url', image_url: { url: imageUrl } },
      { type: 'text', text: prompt },
    ]});
  } else {
    messages.push({ role: 'user', content: prompt });
  }

  const response = await env.AI.run('@cf/google/gemma-4-26b-a4b-it', {
    messages,
    max_tokens: 2048,
    chat_template_kwargs: { thinking: false },
  });

  const content = response.choices?.[0]?.message?.content ?? response.response;
  return Response.json({ response: content });
}
```

### Entity CRUD (`functions/api/entities/[name].js`)

Generic D1 CRUD handler — works for any table. **Important:** The table name comes from the URL, so validate it against an allowlist to prevent SQL injection:

```javascript
// Define allowed tables (update this list for your app)
const TABLES = ['violation_reports', 'users', 'settings'];

function validateTable(name) {
  if (!TABLES.includes(name)) {
    return new Response(JSON.stringify({ error: 'Unknown table' }), {
      status: 400, headers: { 'Content-Type': 'application/json' },
    });
  }
  return null;
}

export async function onRequestGet({ params, request, env }) {
  const { name } = params;
  const err = validateTable(name);
  if (err) return err;
  const url = new URL(request.url);
  const limit = parseInt(url.searchParams.get('limit') || '50');
  const offset = parseInt(url.searchParams.get('offset') || '0');
  const { results } = await env.DB.prepare(
    `SELECT * FROM ${name} ORDER BY created_date DESC LIMIT ? OFFSET ?`
  ).bind(limit, offset).all();
  return Response.json(results);
}

// Define allowed columns per table to prevent column injection
const TABLE_COLUMNS = {
  violation_reports: ['photo_url', 'license_plate', 'car_color', 'violation_type'],
  users: ['name', 'email', 'role'],
  settings: ['key', 'value'],
};

export async function onRequestPost({ params, request, env }) {
  const { name } = params;
  const err = validateTable(name);
  if (err) return err;
  const data = await request.json();
  const allowed = TABLE_COLUMNS[name] || [];
  const filtered = Object.fromEntries(
    Object.entries(data).filter(([k]) => allowed.includes(k))
  );
  const id = crypto.randomUUID();
  const now = new Date().toISOString();
  const fields = { id, created_date: now, updated_date: now, ...filtered };
  const keys = Object.keys(fields);
  const placeholders = keys.map(() => '?').join(', ');
  await env.DB.prepare(
    `INSERT INTO ${name} (${keys.join(', ')}) VALUES (${placeholders})`
  ).bind(...Object.values(fields)).run();
  return Response.json(fields, { status: 201 });
}
```

### File Upload (`functions/api/upload.js`)

R2 upload handler — replaces Supabase Storage or Base44 UploadFile:

```javascript
export async function onRequestPost({ request, env }) {
  const formData = await request.formData();
  const file = formData.get('file');
  if (!file) return Response.json({ error: 'No file' }, { status: 400 });
  // Sanitize filename: strip path traversal, keep only the basename
  const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
  const key = `${Date.now()}_${safeName}`;
  await env.BUCKET.put(key, file.stream(), {
    httpMetadata: { contentType: file.type },
  });
  return Response.json({ url: `/files/${key}`, file_url: `/files/${key}` });
}
```
