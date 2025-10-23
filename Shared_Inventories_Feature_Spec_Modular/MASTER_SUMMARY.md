# MASTER_SUMMARY — Shared Inventories Feature (OutlandsLoreTrader)

**Date:** 2025-10-22  
**Owner:** Big Data (Discord: big_data)  
**Prepared by:** ChatGPT (aligned with `.cursor/rules/` and project context)  
**Delivery Format:** Modular plan for multi‑agent execution with context refresh gates.

---

## Purpose

Introduce **Shared Inventories** so that multiple users (e.g., guild officers) can share and manage a single lore inventory. 
Trading and matching must work across **Personal ↔ Personal**, **Personal ↔ Shared**, and **Shared ↔ Shared**, with safe concurrency.

This MASTER file is the index and coordinator for all sub‑documents. Feed documents **in order** to AI agents.
Each stage fits a ~200k–250k token context and ends with a **Commit Checkpoint** and **Summary/Refresh Gate**.

---

## File Index & Execution Order

1. **FEATURE_SHARED_INVENTORIES_01_OVERVIEW_AND_ARCHITECTURE.md**  
   *What & Why; UX overview; high-level architecture; concurrency philosophy; role matrix; glossary.*  
   **Context Gate:** After reading, summarize to ~2–3k tokens before proceeding.

2. **FEATURE_SHARED_INVENTORIES_02_DATABASE_AND_API_DESIGN.md**  
   *D1 schema, migrations, endpoint designs, request/response shapes, permissions & validation.*  
   **Context Gate:** Summarize key tables, columns, and endpoints to ~3–4k tokens.

3. **FEATURE_SHARED_INVENTORIES_03_FRONTEND_IMPLEMENTATION.md**  
   *UI/UX, component maps, hooks design, state order of operations, type contracts.*  
   **Context Gate:** Keep only the state order of operations, component contracts, and types (~3–4k tokens).

4. **FEATURE_SHARED_INVENTORIES_04_BACKEND_IMPLEMENTATION_AND_SECURITY.md**  
   *Cloudflare Worker routing, middleware, reservation lifecycle, error handling, logging, CORS, security.*  
   **Context Gate:** Keep reservation lifecycle & permission checks (~2–3k tokens).

5. **FEATURE_SHARED_INVENTORIES_05_DEPLOYMENT_AND_TESTING_PLAN.md**  
   *Migrations rollout, feature flags, test matrix, load/concurrency tests, rollback, monitoring, changelog entries.*  
   **Context Gate:** Keep only rollout steps and the test matrix (~2–3k tokens).

---

## Repository Path Guidance (Specific + General)

We balance **specificity** (so agents know where to look) with **flexibility** (so paths can be validated first).  
Agents must **confirm dependencies and verify actual paths** before writing code.

- **Frontend (general):** “Inventory hooks”, “Trading UI”, “Modals”, “Type definitions”.  
  **Likely paths:**  
  - `src/components/inventory/` (e.g., `MyInventoryCard.tsx`, hooks)  
  - `src/components/trading/` (e.g., `TradeMatchesSection.tsx`)  
  - `src/components/modals/` (new modals for shared inventories)  
  - `src/lib/` (clients & helpers)  
  - `src/data/` (static data & constants)  
- **Backend (general):** “API Worker”, “Auth Worker”, “Routes”, “Middleware”.  
  **Likely paths:**  
  - `cloudflare-workers/api/src/routes/` (e.g., `inventories.ts`, `trades.ts`)  
  - `cloudflare-workers/api/src/middleware/` (auth/CORS)  
  - `cloudflare-workers/discord-auth/src/` (unchanged)  
- **Migrations (general):** “D1 migrations”.  
  **Likely paths:**  
  - `supabase/d1-migrations/` (timestamped SQL files)

> Agents: If any path differs, **update the spec references in your working branch** to the correct locations before coding.

---

## Recent Change Highlights (for Context)

- **Inventory History Feature (2025-10-22)** — `inventory_snapshots` design and UI (`InventoryHistoryModal.tsx`), auto-snapshots on all inventory‑modifying operations.  
- **Multiple Copies Bug Fix (2025-10-21)** — Book completion script now prompts for new blank tome per copy; improved overhead messages.  
- **TradingTab Modular Refactor (2025-10-20)** — Extracted hooks & components; orchestrator pattern, type safety; D1‑only architecture.  
- **CORS & Summary Fixes (2025-10-21)** — PATCH allowed; purge refreshes summary; empty state UX improved.

These changes influence Shared Inventories by: reusing snapshot patterns, leveraging orchestrator + hooks, ensuring CORS/UX consistency.

---

## Security Posture (Summary)

- JWT-based auth; **user isolation** and **membership checks** for shared inventories.  
- Concurrency via **reservation records**; matching uses **available = count − reserved**.  
- No GitHub writes without explicit user permission.  
- Prepared statements only; input validation and clear error responses.

---

## Commit Rhythm & Context Refresh

Each file ends with:
- **Commit Checkpoint** — recommended `git` commands.  
- **Summary/Refresh Gate** — instructions to compress active context to 10% of prior stage before proceeding.

---

## Start Here

Proceed to: **FEATURE_SHARED_INVENTORIES_01_OVERVIEW_AND_ARCHITECTURE.md**
