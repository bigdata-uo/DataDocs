# FEATURE_SHARED_INVENTORIES_05_DEPLOYMENT_AND_TESTING_PLAN

**Date:** 2025-10-22  
**Scope:** Migrations rollout, feature flag, test matrix, load/concurrency tests, rollback, monitoring, changelog

---

## 1. Rollout Plan

1. **Feature Flag** (optional): `VITE_FEATURE_SHARED_INVENTORIES=true` for gated UI surfaces.  
2. **Migrations:** Apply D1 migrations (local → staging → production).  
3. **Staging Verification:** End‑to‑end happy path (Personal↔Shared & Shared↔Shared).  
4. **Progressive Exposure:** Invite limited users/guilds; monitor logs.  
5. **Production Enable:** Remove flag once stable.

**Likely commands:**
```bash
# Local
npx wrangler d1 migrations apply outlands-lore-trader --local
npx wrangler dev

# Prod
npx wrangler d1 migrations apply outlands-lore-trader
npx wrangler deploy
```

---

## 2. Test Matrix

### 2.1 Unit (Frontend)
- Tabs render; role badges; reserved indicators.  
- Context selector drives matching API calls.  
- Modals validate form inputs.

### 2.2 Integration (API)
- Create shared → invite → accept → edit counts → match → initiate → confirm → complete.  
- Permission failures (403), availability failures (409).

### 2.3 Concurrency
- Two members initiate trades from same shared inventory; reservations prevent double‑spend.  
- Timeout cleanup releases reservations.

### 2.4 E2E Scenarios
- Personal ↔ Personal (regression)  
- Personal ↔ Shared  
- Shared ↔ Personal  
- Shared ↔ Shared

---

## 3. Monitoring

- `wrangler tail --format pretty` during rollout.  
- Log trade/reservation events (info level).  
- Track reservation counts per inventory to detect starvation.

---

## 4. Rollback

- `npx wrangler deployments list` → `npx wrangler rollback <id>`  
- Schema rollback scripts for newly added tables/columns.  
- Disable feature flag in UI if needed.

---

## 5. Changelog Entries (Template)

**2025-10-22 — Shared Inventories (Planned)**  
- Added shared inventory data model & endpoints.  
- Frontend tabs + trading context selector.  
- Reservation‑based concurrency model.  
- Security: membership checks & context validation.  
- Docs: user guide & security updates.

When shipped, add file references and commit hashes per repo standards.

---

## 6. Dependencies & Cross‑Refs

- Reuse **Inventory History** patterns (snapshots) if enabling shared snapshots.  
- Honor **TradingTab Orchestrator** contracts and hook boundaries.  
- Maintain CORS and timestamp handling per rules.

---

## Commit Checkpoint

```bash
git add .
git commit -m "docs(shared-inventories): stage 05 deployment & testing plan"
```

## Summary/Refresh Gate

- Summarize to ~2–3k tokens: rollout steps + test matrix.
