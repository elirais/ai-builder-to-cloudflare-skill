# Schema Mapping → D1 SQL

Covers Base44 entities (NoSQL) and Supabase/PostgreSQL (used by Lovable and Bolt.new).

## Base44 Entity Type Mapping

| Base44 Field Type | D1 SQL Type | Notes |
|---|---|---|
| `string` | `TEXT` | Default string type |
| `string` + `enum: [...]` | `TEXT CHECK(col IN ('a','b','c'))` | Enum constraint |
| `string` + `format: "email"` | `TEXT` | Add CHECK if desired |
| `string` + `format: "date-time"` | `TEXT` | ISO 8601 string |
| `string` + `format: "date"` | `TEXT` | ISO 8601 date |
| `string` + `format: "time"` | `TEXT` | Time string |
| `string` + `maxLength: N` | `TEXT` | D1 doesn't enforce length; document in comment |
| `number` | `REAL` | Floating point |
| `integer` | `INTEGER` | Whole numbers |
| `boolean` | `INTEGER` | 0 = false, 1 = true |
| `object` (with properties) | `TEXT` | Store as JSON string |
| `object` (freeform, no properties) | `TEXT` | Arbitrary JSON — store as-is |
| `array` of strings | `TEXT` | JSON array: `'["a","b","c"]'` |
| `array` of objects | `TEXT` | JSON array of objects |

## System Fields (always add)

Every Base44 entity has these hidden system fields that are NOT in the schema but exist on every record:

```sql
id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(12)))),
created_date TEXT DEFAULT (datetime('now')),
updated_date TEXT DEFAULT (datetime('now')),
created_by TEXT,
created_by_id TEXT
```

## Foreign Keys

Base44 has NO formal FK system. References are plain string fields containing an entity ID.

**Convention:** If a field ends in `_id` or matches another entity name + `_id`, add a REFERENCES clause:
```sql
user_id TEXT REFERENCES users(id),
```

**Warning:** Some apps use custom FK targets (e.g., `unique_claim_id` instead of `id`). Check the schema description field for hints. When unsure, omit the REFERENCES and use a plain TEXT column.

## Row-Level Security (RLS)

Base44 RLS is a declarative JSON system with template variables. In Cloudflare, implement as middleware in Pages Functions:

| Base44 RLS Pattern | Cloudflare Implementation |
|---|---|
| `{ created_by: "{{user.email}}" }` | `WHERE created_by = ?` (bind logged-in user email) |
| `{ user_condition: { role: "admin" } }` | Check JWT role claim in middleware |
| `{ $or: [{created_by: "{{user.email}}"}, {user_condition: {role: "admin"}}] }` | Combined check |
| `{ data_condition: { is_public: true } }` | `WHERE is_public = 1 OR created_by = ?` |
| `true` | No restriction |

## MongoDB → SQL Query Translation

| MongoDB Operator | SQL Equivalent | Example |
|---|---|---|
| `{ field: value }` | `WHERE field = ?` | `{ status: "active" }` → `WHERE status = 'active'` |
| `{ field: { $gt: n } }` | `WHERE field > ?` | `{ amount: { $gt: 100 } }` → `WHERE amount > 100` |
| `{ field: { $gte: n } }` | `WHERE field >= ?` | |
| `{ field: { $lt: n } }` | `WHERE field < ?` | |
| `{ field: { $lte: n } }` | `WHERE field <= ?` | |
| `{ field: { $ne: v } }` | `WHERE field != ?` | |
| `{ field: { $in: [...] } }` | `WHERE field IN (?, ?, ?)` | |
| `{ field: { $nin: [...] } }` | `WHERE field NOT IN (...)` | |
| `{ field: { $exists: true } }` | `WHERE field IS NOT NULL` | |
| `{ field: { $regex: "..." } }` | `WHERE field LIKE '%...%'` | Limited in SQLite |
| `{ $or: [...] }` | `WHERE (...) OR (...)` | |
| `{ $and: [...] }` | `WHERE (...) AND (...)` | |

**Unsupported in D1 (need application-level handling):**
- `$all` — array containment (use `json_each()` in SQLite)
- `$size` — array length (use `json_array_length()`)
- `$push` / `$pull` — array mutation (read, modify in JS, write back)
- `$inc` — atomic increment → `SET field = field + ?` (works in SQL)
- `$set` — partial update → standard `UPDATE SET` (works in SQL)

## Example Migration

**Base44 schema:**
```json
{
  "name": "ViolationReport",
  "properties": {
    "photo_url": { "type": "string" },
    "license_plate": { "type": "string" },
    "car_color": { "type": "string" },
    "violation_type": { "type": "string", "enum": ["sidewalk", "double", "crosswalk", "red_zone"] }
  },
  "required": ["photo_url"]
}
```

**D1 migration SQL:**
```sql
CREATE TABLE IF NOT EXISTS violation_reports (
  id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(12)))),
  created_date TEXT DEFAULT (datetime('now')),
  updated_date TEXT DEFAULT (datetime('now')),
  created_by TEXT,
  photo_url TEXT NOT NULL,
  license_plate TEXT,
  car_color TEXT,
  violation_type TEXT CHECK(violation_type IN ('sidewalk', 'double', 'crosswalk', 'red_zone'))
);
```

## Complex Fields — Decision Guide

When encountering arrays of objects or deeply nested structures:

1. **If the field is only ever read/written as a whole** (e.g., order items embedded in an order) → keep as JSON TEXT column
2. **If individual items need to be queried/filtered** (e.g., find all orders containing product X) → normalize to a child table
3. **If the structure is freeform/schemaless** (e.g., webhook payload, metadata) → must be JSON TEXT column

For JSON columns, use SQLite JSON functions for queries:
```sql
-- Query into JSON array
SELECT * FROM orders WHERE EXISTS (
  SELECT 1 FROM json_each(items) WHERE json_extract(value, '$.product_id') = ?
);

-- Get array length
SELECT *, json_array_length(tags) as tag_count FROM posts;
```

## Supabase (PostgreSQL) → D1 Mapping

For Lovable and Bolt.new apps that use Supabase as their backend.

### Type Mapping

| PostgreSQL Type | D1 SQL Type | Notes |
|---|---|---|
| `UUID` | `TEXT` | Generate with `crypto.randomUUID()` in JS |
| `SERIAL` / `BIGSERIAL` | `INTEGER PRIMARY KEY AUTOINCREMENT` | Only for primary keys |
| `INTEGER` | `INTEGER` | Direct mapping |
| `BIGINT` | `INTEGER` | SQLite integers are 64-bit |
| `REAL` / `DOUBLE PRECISION` | `REAL` | Direct mapping |
| `NUMERIC` / `DECIMAL` | `REAL` | Loses arbitrary precision |
| `TEXT` / `VARCHAR(n)` | `TEXT` | D1 doesn't enforce length |
| `BOOLEAN` | `INTEGER` | 0 = false, 1 = true |
| `JSONB` / `JSON` | `TEXT` | Use `json_extract()` for queries |
| `TIMESTAMP WITH TIME ZONE` | `TEXT` | Store as ISO 8601 string |
| `TIMESTAMP` | `TEXT` | Store as ISO 8601 string |
| `DATE` | `TEXT` | Store as `YYYY-MM-DD` |
| `TIME` | `TEXT` | Store as `HH:MM:SS` |
| `TEXT[]` (array) | `TEXT` | Store as JSON array: `'["a","b"]'` |
| `BYTEA` | `BLOB` | Binary data |

### Supabase-Specific Conversions

**RLS policies → Application-level checks:**

Supabase enforces security through Row Level Security in PostgreSQL. D1 has no RLS — enforce in Pages Functions middleware:

```javascript
// Supabase RLS: auth.uid() = user_id
// Cloudflare equivalent: check in Pages Function
export async function onRequestGet({ request, env }) {
  const userId = await getUserFromJWT(request);
  const { results } = await env.DB.prepare(
    'SELECT * FROM items WHERE user_id = ?'
  ).bind(userId).all();
  return Response.json(results);
}
```

**Supabase `gen_random_uuid()` → JS `crypto.randomUUID()`:**

```sql
-- Supabase migration:
CREATE TABLE items (id UUID DEFAULT gen_random_uuid() PRIMARY KEY);

-- D1 equivalent:
CREATE TABLE items (id TEXT PRIMARY KEY);
-- Generate ID in JS: crypto.randomUUID()
```

**Supabase triggers → Application logic:**

Supabase uses PostgreSQL triggers for `updated_at`. In D1, set `updated_date` in your Pages Function:
```javascript
await env.DB.prepare(
  "UPDATE items SET name = ?, updated_date = datetime('now') WHERE id = ?"
).bind(name, id).run();
```

### Converting Supabase Migrations

Lovable and Bolt apps store migrations in `supabase/migrations/`. Convert each file:

1. Replace `UUID` → `TEXT`
2. Replace `SERIAL` → `INTEGER PRIMARY KEY AUTOINCREMENT`  
3. Replace `BOOLEAN` → `INTEGER`
4. Replace `JSONB` → `TEXT`
5. Replace `TIMESTAMP WITH TIME ZONE` → `TEXT`
6. Replace `NOW()` → `datetime('now')`
7. Replace `gen_random_uuid()` → remove (generate in JS)
8. Remove all `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and `CREATE POLICY` statements
9. Remove `GRANT` and `REVOKE` statements
