# FEATURE_SHARED_INVENTORIES_01_OVERVIEW_AND_ARCHITECTURE

**Date:** 2025-10-22  
**Scope:** What/Why, UX overview, high-level architecture, roles, concurrency philosophy, glossary  
**Project:** OutlandsLoreTrader (React + TS + Vite + Tailwind + shadcn/ui; Cloudflare Workers + D1; Discord OAuth)  

---

## 1. Problem & Objectives

### Problem
Guilds often maintain one or more **shared lore tome inventories** curated by multiple officers. The current model is **per‑user** only, with trades **user ↔ user**. We must support **Shared Inventories** so multiple users can view, edit, match, and trade with a single pooled inventory safely.

### Objectives
1. Add **Shared Inventory tabs** to “My Inventory” allowing users to switch between **Personal** and **Shared** contexts.  
2. Support trades across all combinations: **Personal ↔ Personal**, **Personal ↔ Shared**, **Shared ↔ Shared**.  
3. Solve **concurrency** with safe **reservations** so pages are not double‑spent.  
4. Maintain UX parity: all existing inventory features must work in Shared mode.  
5. Provide **membership management** (owner/manager/member) and **invite/accept** flows.

---

## 2. UX Overview

### 2.1 Tabs in My Inventory
- Tabs: **Personal**, **Shared: <Name>** (one per shared membership).  
- Tab header shows **member count**, **role**, **last updated**.  
- Tab actions (permissioned): **Invite**, **Manage Members**, **Rename**, **Leave**, **Delete** (owner).

### 2.2 “Currently Trading With” Selector
- New control in **TradeMatchesSection**:  
  **Currently trading with**: `( ) Personal` `(|) Shared: <Name>` `(|) …`  
- Changes **trading context** used for matching and trade initiation (doesn’t have to switch visible tab).

### 2.3 Contact Surfaces in Trade Cards
- If **target is Shared**, show **primary contacts** (owner/managers) and **Copy Group Message**.  
- If **both Shared**, show **both rosters** and both group message options.

### 2.4 Reservations Visibility
- Show **Reserved** indicator for pages reserved by active trades; subtract from availability in UI and “Books Done” flows.

---

## 3. High‑Level Architecture

### Frontend (Specific + General)
- **General:** Inventory hooks, Trading UI, Type models, Modals.  
- **Likely paths:**  
  - `src/components/inventory/` (tabs, counts, reserved indicators)  
  - `src/components/trading/TradeMatchesSection.tsx` (context selector)  
  - `src/components/modals/` (create/invite/manage shared inv)  
  - `src/lib/` (API clients), `src/data/` (constants)  

### Backend (Specific + General)
- **General:** API Worker routes, Middleware, Matching & Trades controllers.  
- **Likely paths:**  
  - `cloudflare-workers/api/src/routes/` (new `shared-inventories.ts`, updates to `inventories.ts`, `trades.ts`, `matching.ts`)  
  - `cloudflare-workers/api/src/middleware/` (auth, CORS)  

### Database
- **New tables:** `shared_inventories`, `shared_inventory_members`, `shared_inventory_pages`, `inventory_reservations`, (`shared_inventory_invitations` optional).  
- **Altered:** `active_trades` (+context columns), optionally `inventory_snapshots` (+owner polymorphism).

---

## 4. Roles & Permissions (Summary)

| Role | Personal | Shared Owner | Shared Manager | Shared Member |
|---|---:|---:|---:|---:|
| View | ✅ | ✅ | ✅ | ✅ |
| Edit counts | ✅ | ✅ | ✅ | (configurable) |
| Invite | — | ✅ | ✅ | ❌ |
| Remove/change roles | — | ✅ | ✅ (no Owner) | ❌ |
| Transfer ownership | — | ✅ | ❌ | ❌ |
| Initiate/Confirm trades | ✅ | ✅ | ✅ | ✅ |

> Fine‑grained edit permission for Members can be a per‑shared setting.

---

## 5. Concurrency Philosophy

- **Reservation‑first:** On initiate/confirm, **reserve** page quantities tied to `active_trade_id`.  
- **Matching uses available:** `available = count − sum(reserved)` by canonical page id.  
- **Release on end states:** completion, cancel, decline, timeout.  
- **Optional lease:** `expires_at` with scheduled cleanup worker.

This avoids global locks and scales across Personal/Shared contexts.

---

## 6. Glossary

- **Personal Inventory:** pages in `user_inventories` for a single profile.  
- **Shared Inventory:** pooled pages in `shared_inventory_pages` managed by multiple members.  
- **Context:** the owner (user or shared) on behalf of which a trade is being made.  
- **Reservation:** a temporary hold against available quantity for a page while a trade is active.

---

## 7. Dependencies & Prior Art

- **Inventory Snapshots (2025‑10‑22):** pattern for history/restore — extendable to shared.  
- **TradingTab Orchestrator (2025‑10‑20):** hooks separation; easy to extend with a trading context state.  
- **CORS/Empty State Fixes (2025‑10‑21):** keep parity for new flows.

---

## 8. Risks & Mitigations

- **Path drift vs. spec:** Require agents to **confirm real paths** before execution; update references if needed.  
- **Double‑spend:** Handled via reservations.  
- **Membership churn mid‑trade:** Validate membership on each mutating action; block if revoked.  
- **Over‑complex UX:** Keep tabs + a single selector; reuse shadcn patterns.  
- **Data migration coupling:** New tables; minimal changes to existing tables; feature‑flag rollout.

---

## Commit Checkpoint

```bash
git add .
git commit -m "docs(shared-inventories): stage 01 overview & architecture"
```

## Summary/Refresh Gate

- Summarize this file to ~2–3k tokens (objectives, roles, concurrency, UX, high‑level arch).  
- Drop detailed prose; keep only what downstream agents need to recall.
