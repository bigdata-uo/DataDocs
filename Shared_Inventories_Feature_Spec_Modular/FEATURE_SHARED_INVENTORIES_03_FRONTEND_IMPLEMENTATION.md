# FEATURE_SHARED_INVENTORIES_03_FRONTEND_IMPLEMENTATION

**Date:** 2025-10-22  
**Scope:** UI/UX, components, hooks, state order, types, examples

---

## 1. UX & Components

### 1.1 My Inventory Tabs
- Tabs: **Personal**, **Shared: <Name>** …  
- Tab header meta: member count, role, last updated.  
- Tab actions (permissioned): Invite, Manage Members, Rename, Leave, Delete.

**Likely files/paths (validate first):**
- `src/components/inventory/MyInventoryCard.tsx`
- `src/components/inventory/hooks/useInventoryState.ts`
- `src/components/modals/CreateSharedInventoryModal.tsx`
- `src/components/modals/InviteMembersModal.tsx`
- `src/components/modals/ManageMembersModal.tsx`

### 1.2 Trade Matches — “Currently Trading With”
- Selector in `TradeMatchesSection` to choose **trading context**:
  - Radio group: Personal / Shared: <Name> / …
- Drives matching queries and trade initiation.

**Likely files/paths:**
- `src/components/trading/TradeMatchesSection.tsx`
- `src/components/trading/hooks/useTradeMatching.ts`
- `src/components/trading/hooks/useTradeActions.ts`

### 1.3 Reserved Indicators
- In inventory UIs, show reserved quantity (subtracted from available).  
- Tooltip with trade id; reflect in “Books Done” modal & scripts.

---

## 2. Types & Contracts

**General path:** `src/lib/d1-types.ts` (validate location)

```ts
export type OwnerType = 'user' | 'shared'

export interface SharedInventory {
  id: string
  name: string
  guild_tag?: string
  description?: string
  owner_user_id: string
  created_at: string
  updated_at: string
}

export interface SharedInventoryMember {
  shared_inventory_id: string
  user_id: string
  role: 'owner'|'manager'|'member'
  status: 'active'|'invited'|'removed'
  created_at: string
  updated_at: string
}

export interface InventoryReservation {
  id: string
  owner_type: OwnerType
  owner_id: string
  active_trade_id: string
  canonical_id: number
  quantity: number
  created_at: string
  expires_at?: string
}
```

Matching input:
```ts
interface MatchingRequest {
  context_type: OwnerType
  context_id?: string
}
```

Trade creation input:
```ts
interface CreateTradeRequest {
  initiator_context_type: OwnerType
  initiator_context_id?: string
  target_context_type: OwnerType
  target_context_id: string
  initiator_offering_pages: number[]
  initiator_requesting_pages: number[]
}
```

---

## 3. State Management — Order of Operations

1. **Bootstrap session**
   - Fetch personal profile & inventories (+ reserved aggregate).  
   - Fetch `GET /shared-inventories/mine` → populate shared meta list.

2. **Active Tab & Trading Context**
   - Local UI state for **active tab** (viewing).  
   - Separate state for **trading context** (radio group). Defaults to Personal.

3. **Inventory Load per Context**
   - When tab changes or when trading context is set to Shared:  
     - Load `GET /shared-inventories/:id/pages` (with computed `available`).

4. **Matching**
   - On trading context change or inventory refresh → call `POST /matching` with context.  
   - Display matches; card renders contact surfaces (personal vs shared).

5. **Initiate Trade**
   - Call `POST /trades` with initiator context and page selections.  
   - On success: toast + deep link composed; reservations created on server.

6. **Confirm/Cancel**
   - Confirm trade (target): `POST /trades/:id/confirm` → reservations validated → apply deltas → success toasts.  
   - Cancel/Decline: release reservations; refresh inventories & matches.

7. **Error Paths**
   - If “insufficient available pages”, re-fetch and re-compute availability; show reserved sources.

---

## 4. Hooks & Components (General + Likely Paths)

- `useInventoryState.ts` (likely: `src/components/inventory/hooks/`):  
  - Extend to manage **tabs**, **shared memberships**, **inventory fetch for shared**, and **reserved** aggregates.
- `useTradeMatching.ts` (likely: `src/components/trading/hooks/`):  
  - Accept `tradingContext` and call `/matching` accordingly.
- `useTradeActions.ts` (likely: `src/components/trading/hooks/`):  
  - Include context fields in `/trades` calls.
- `MyInventoryCard.tsx` / `TradeMatchesSection.tsx`:  
  - UI controls for tab switching and context selection.

> Agents: Confirm exact file locations before editing.

---

## 5. UI Patterns (shadcn + Tailwind)

- **Tabs:** shadcn Tabs with Card skeleton; dark theme.  
- **Dialogs:** Create/Invite/Manage modals using `Dialog` with max‑w content.  
- **Toasts:** Success/error outcomes; short durations.  
- **Badges:** Roles and member count; semantic colors.

---

## 6. Error Handling & UX Consistency

- Always show **loading**, **error**, **empty** states.  
- Append `'Z'` to D1 timestamps when parsing.  
- Parse JSON text fields defensively.

---

## 7. Repository Path Guidance (Specific + General)

- **General:** Inventory hooks, Trading UI, Modals, Types.  
- **Likely paths:**  
  - `src/components/inventory/`, `src/components/trading/`, `src/components/modals/`  
  - `src/lib/` (API clients), `src/data/` (constants)  
- Validate paths & imports before code generation.

---

## Commit Checkpoint

```bash
git add src/components src/lib src/data
git commit -m "feat(shared-inventories): stage 03 frontend scaffolding (tabs, context selector, types)"
```

## Summary/Refresh Gate

- Summarize to ~3–4k tokens: state order, component contracts, types, and key UX flows.
