# Base44 SDK — Complete API Surface

Reference for the `@base44/sdk` npm package. Always use the latest version — do NOT pin to a specific version.

## Installation & Setup

```javascript
import { createClient, getAccessToken } from '@base44/sdk';

const base44 = createClient({
  appId: 'YOUR_APP_ID',          // required
  serverUrl: 'https://base44.app', // default
  requiresAuth: false,            // set true to force login redirect
});
```

## 8 Modules

### 1. Entities (CRUD + Realtime)

Dynamic proxy — any property name becomes an entity handler:

```javascript
const items = await base44.entities.MyEntity.list(sort, limit, skip, fields);
const filtered = await base44.entities.MyEntity.filter(query, sort, limit, skip);
const one = await base44.entities.MyEntity.get(id);
const created = await base44.entities.MyEntity.create(data);
const updated = await base44.entities.MyEntity.update(id, data);
await base44.entities.MyEntity.delete(id);
await base44.entities.MyEntity.deleteMany(query);
const many = await base44.entities.MyEntity.bulkCreate([...]);
const updated = await base44.entities.MyEntity.bulkUpdate([{id, ...data}]);
await base44.entities.MyEntity.updateMany(query, { $set: {}, $inc: {} });
const result = await base44.entities.MyEntity.importEntities(csvFile);

// Realtime (WebSocket)
const unsubscribe = base44.entities.MyEntity.subscribe(callback);
```

**Filter operators (MongoDB-style):**
`$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin, $exists, $regex, $all, $size, $not`
Logical: `$and, $or, $nor`

**System fields on every record:** `id, created_date, updated_date, created_by, created_by_id, is_sample`

### 2. Integrations

```javascript
// Core (7 built-in endpoints):
await base44.integrations.Core.InvokeLLM({
  prompt: '...',
  model: 'gpt_5_mini',  // or gemini_3_flash, gpt_5, claude_sonnet_4_6, etc.
  response_json_schema: { ... },  // forces structured JSON output
  file_urls: ['...'],
  add_context_from_internet: false,
});

await base44.integrations.Core.UploadFile({ file });           // → { file_url }
await base44.integrations.Core.UploadPrivateFile({ file });    // → { file_uri }
await base44.integrations.Core.CreateFileSignedUrl({ file_uri, expires_in: 300 });
await base44.integrations.Core.SendEmail({ to, subject, body, from_name });
await base44.integrations.Core.GenerateImage({ prompt });      // → { url }
await base44.integrations.Core.ExtractDataFromUploadedFile({ file_url, json_schema });

// Custom integrations (OpenAPI-based):
await base44.integrations.custom.call(slug, operationId, { payload, pathParams, queryParams });
```

### 3. Auth

```javascript
const user = await base44.auth.me();
await base44.auth.updateMe(data);
base44.auth.redirectToLogin(nextUrl);
base44.auth.loginWithProvider('google'); // google, microsoft, facebook, apple, sso
await base44.auth.loginViaEmailPassword(email, password, turnstileToken);
base44.auth.logout(redirectUrl);
const isAuth = await base44.auth.isAuthenticated();
await base44.auth.inviteUser(email, role);
await base44.auth.register({ email, password });
await base44.auth.verifyOtp({ email, otpCode });
await base44.auth.resetPasswordRequest(email);
await base44.auth.resetPassword({ resetToken, newPassword });
await base44.auth.changePassword({ userId, currentPassword, newPassword });
```

### 4. Functions (Serverless Backend)

```javascript
const result = await base44.functions.invoke('functionName', data);
const response = await base44.functions.fetch('/path', init); // raw fetch, supports streaming
```

Server-side (Deno):
```typescript
import { createClientFromRequest } from 'npm:@base44/sdk';
const base44 = createClientFromRequest(req);
const user = await base44.auth.me();
const data = await base44.asServiceRole.entities.X.filter({});
```

### 5. Agents (AI Chat)

```javascript
const conversations = await base44.agents.getConversations();
const conv = await base44.agents.createConversation({ agent_name: '...' });
await base44.agents.addMessage(conv, message);
const unsub = base44.agents.subscribeToConversation(convId, onUpdate);
const whatsappUrl = await base44.agents.getWhatsAppConnectURL(agentName);
const telegramUrl = await base44.agents.getTelegramConnectURL(agentName);
```

### 6. Analytics

```javascript
base44.analytics.track({ eventName: 'page_view', properties: { page: '/home' } });
```

### 7. App Logs

```javascript
await base44.appLogs.logUserInApp(pageName);
await base44.appLogs.fetchLogs(params);
```

### 8. Connectors (OAuth)

```javascript
const url = await base44.connectors.connectAppUser(connectorId);
await base44.connectors.disconnectAppUser(connectorId);
// Service role:
const { accessToken } = await base44.asServiceRole.connectors.getConnection(type);
```

## Typical Base44 File Structure

Every Base44 app follows this pattern:
```
src/
├── api/
│   ├── base44Client.js      ← SDK singleton (delete in migration)
│   ├── entities.js           ← entity proxy exports (delete)
│   └── integrations.js       ← integration re-exports (rewrite)
├── lib/
│   ├── app-params.js         ← URL param reader (delete)
│   └── utils.js              ← cn() helper for tailwind (keep if used)
├── components/
│   ├── ui/                   ← shadcn components (audit for usage)
│   ├── VisualEditAgent.jsx   ← Base44 editor (delete)
│   └── NavigationTracker.jsx ← Base44 analytics (delete)
├── pages/                    ← app pages (transform SDK calls)
├── pages.config.js           ← Base44 routing (delete, use react-router)
└── ...
base44/
└── functions/                ← Deno backend functions (migrate to Pages Functions)
```
