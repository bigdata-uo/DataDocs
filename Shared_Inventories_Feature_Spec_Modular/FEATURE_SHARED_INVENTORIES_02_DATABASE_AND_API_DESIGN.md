# FEATURE_SHARED_INVENTORIES_02_DATABASE_AND_API_DESIGN

**Date:** 2025-10-22  
**Scope:** D1 schema, migrations, endpoints, permissions, validation, examples

---

## 1. D1 Schema Additions

> Use snake_case, JSON columns as TEXT, timestamps in UTC TEXT, prepared statements only.

### 1.1 New Tables

**`shared_inventories`**
```sql
CREATE TABLE IF NOT EXISTS shared_inventories (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  guild_tag TEXT,
  description TEXT,
  owner_user_id TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);
```

**`shared_inventory_members`**
```sql
CREATE TABLE IF NOT EXISTS shared_inventory_members (
  shared_inventory_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('owner','manager','member')),
  status TEXT NOT NULL CHECK (status IN ('active','invited','removed')),
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (shared_inventory_id, user_id)
);
```

**`shared_inventory_pages`**
```sql
CREATE TABLE IF NOT EXISTS shared_inventory_pages (
  shared_inventory_id TEXT NOT NULL,
  canonical_id INTEGER NOT NULL,
  count INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (shared_inventory_id, canonical_id)
);
```

**`inventory_reservations`**
```sql
CREATE TABLE IF NOT EXISTS inventory_reservations (
  id TEXT PRIMARY KEY,
  owner_type TEXT NOT NULL CHECK (owner_type IN ('user','shared')),
  owner_id TEXT NOT NULL,
  active_trade_id TEXT NOT NULL,
  canonical_id INTEGER NOT NULL,
  quantity INTEGER NOT NULL,
  created_at TEXT DEFAULT (datetime('now')),
  expires_at TEXT
);
CREATE INDEX IF NOT EXISTS idx_res_owner ON inventory_reservations(owner_type, owner_id);
CREATE INDEX IF NOT EXISTS idx_res_trade ON inventory_reservations(active_trade_id);
CREATE INDEX IF NOT EXISTS idx_res_owner_page ON inventory_reservations(owner_type, owner_id, canonical_id);
```

**`shared_inventory_invitations` (optional)**
```sql
CREATE TABLE IF NOT EXISTS shared_inventory_invitations (
  id TEXT PRIMARY KEY,
  shared_inventory_id TEXT NOT NULL,
  invited_user_id TEXT NOT NULL,
  invited_by_user_id TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('manager','member')),
  status TEXT NOT NULL CHECK (status IN ('pending','accepted','declined','expired')),
  token TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);
CREATE UNIQUE INDEX IF NOT EXISTS idx_invite_token ON shared_inventory_invitations(token);
```

### 1.2 Table Alterations

**`active_trades`** — add context columns
```sql
ALTER TABLE active_trades ADD COLUMN initiator_context_type TEXT NOT NULL DEFAULT 'user';
ALTER TABLE active_trades ADD COLUMN initiator_context_id TEXT;
ALTER TABLE active_trades ADD COLUMN target_context_type TEXT NOT NULL DEFAULT 'user';
ALTER TABLE active_trades ADD COLUMN target_context_id TEXT;
```

**(Optional) `inventory_snapshots`** — polymorphic owner
```sql
ALTER TABLE inventory_snapshots ADD COLUMN owner_type TEXT DEFAULT 'user';
ALTER TABLE inventory_snapshots ADD COLUMN owner_id TEXT;
```

> Ensure app code continues to read legacy rows that lack new columns by applying defensive defaults.

---

## 2. Endpoints (Cloudflare Worker API)

All responses: `{ data?: T, error?: string }`  
Errors use 4xx/5xx with explicit messages; include CORS headers.

### 2.1 Shared Inventory CRUD

**POST** `/shared-inventories`  
Body: `{ name: string, guild_tag?: string, description?: string }`  
- Auth required (JWT). Creates record; creator becomes `owner` in members.

**GET** `/shared-inventories/mine`  
- Returns shared inventories where caller is `status='active'`, with role & member_count.

**PATCH** `/shared-inventories/:id`  
- Owner/Manager: rename, update description/guild_tag.

**DELETE** `/shared-inventories/:id`  
- Owner: only when **no active trades/reservations** exist for this inventory.

### 2.2 Memberships & Invitations

**POST** `/shared-inventories/:id/invitations`  
Body: `{ invited_user_id: string, role: 'manager'|'member' }` → returns invite `token`.

**POST** `/shared-inventories/invitations/:token/accept`  
- Activates membership; sets `status='active'`.

**GET** `/shared-inventories/:id/members`  
- List members `{ user_id, role, status, joined_at }`.

**PATCH** `/shared-inventories/:id/members/:userId`  
- Owner/Manager: change role, remove (status='removed').

**POST** `/shared-inventories/:id/leave`  
- Member leaves; Owner must transfer first or delete.

### 2.3 Inventory Data

**GET** `/shared-inventories/:id/pages`  
- Returns pages with `available` computed as `count - reserved` (aggregate from `inventory_reservations`).

**PATCH** `/shared-inventories/:id/pages`  
- Bulk updates `{ updates: [{ canonical_id, delta }] }`; ensure counts never drop below 0.  
- Optionally `{ set: [...] }` for full replace (restricted).

**POST** `/shared-inventories/:id/snapshots` (optional)  
**GET** `/shared-inventories/:id/snapshots` (optional)

### 2.4 Matching & Trades

**POST** `/matching`  
Body: `{ context_type: 'user'|'shared', context_id?: string }`  
- Selects correct inventory source & subtracts reservations.

**POST** `/trades`  
Body includes contexts and pages:
```json
{
  "initiator_context_type":"shared",
  "initiator_context_id":"<shared-id>",
  "target_context_type":"user",
  "target_context_id":"<profile-id>",
  "initiator_offering_pages":[1,2,3],
  "initiator_requesting_pages":[4,5,6]
}
```
- Server validates membership/permissions and **creates reservations** for initiator offers.

**POST** `/trades/:id/confirm`  
- Target validates availability and creates reservations for their offers, then completes and applies deltas.

**POST** `/trades/:id/cancel` | **POST** `/trades/:id/decline`  
- Releases reservations.

**GET** `/trades/:id`  
- Participant‑only access (user or any active member of the shared inventory).

---

## 3. Permissions & Validation

- **Auth:** JWT required; decode with middleware, attach `userId`.  
- **Shared access:** require `(SELECT 1 FROM shared_inventory_members WHERE shared_inventory_id=? AND user_id=? AND status='active')`.  
- **Edit rights:** owner/manager; optionally allow members if shared setting permits.  
- **Trade initiation:** caller must belong to initiator context.  
- **Trade confirmation:** caller must belong to target context.

---

## 4. Example SQL Patterns

**Reserve pages (per page):**
```sql
INSERT INTO inventory_reservations (id, owner_type, owner_id, active_trade_id, canonical_id, quantity, created_at)
VALUES (?, 'shared', ?, ?, ?, ?, datetime('now'));
```

**Compute availability (shared):**
```sql
SELECT p.canonical_id,
       p.count - IFNULL(SUM(r.quantity), 0) AS available
FROM shared_inventory_pages p
LEFT JOIN inventory_reservations r
  ON r.owner_type='shared'
 AND r.owner_id = p.shared_inventory_id
 AND r.canonical_id = p.canonical_id
GROUP BY p.shared_inventory_id, p.canonical_id;
```

**Apply deltas on completion (pseudo):**
- decrement from each initiator offering owner
- decrement from each target offering owner
- increment to receiving owners
- delete reservations for `active_trade_id`

---

## 5. Repository Path Guidance (Specific + General)

- **General:** API Worker routes and middleware.  
- **Likely paths:** `cloudflare-workers/api/src/routes/`, `cloudflare-workers/api/src/middleware/`  
  - New: `shared-inventories.ts`  
  - Updates: `inventories.ts`, `trades.ts`, `matching.ts`  
- **Migrations:** `supabase/d1-migrations/` with timestamped filenames.

> Agents must **confirm actual file paths** and update imports accordingly before implementation.

---

## Commit Checkpoint

```bash
git add supabase/d1-migrations/*.sql cloudflare-workers/api/src/routes/* cloudflare-workers/api/src/middleware/*
git commit -m "feat(shared-inventories): stage 02 D1 schema & API surfaces"
```

## Summary/Refresh Gate

- Summarize to ~3–4k tokens: table list, new columns, endpoint list, validation rules, and key SQL patterns.
