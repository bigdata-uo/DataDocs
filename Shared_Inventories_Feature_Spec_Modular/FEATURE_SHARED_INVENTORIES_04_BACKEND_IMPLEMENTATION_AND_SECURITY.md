# FEATURE_SHARED_INVENTORIES_04_BACKEND_IMPLEMENTATION_AND_SECURITY

**Date:** 2025-10-22  
**Scope:** Worker routing, middleware, reservation lifecycle, error handling, logging, CORS, security

---

## 1. Routing & Middleware

- Add new route module (likely): `cloudflare-workers/api/src/routes/shared-inventories.ts`  
- Update existing routes: `inventories.ts`, `trades.ts`, `matching.ts`  
- Ensure **auth middleware** attaches `userId` and enforces CORS including `PATCH`.

**Validation helpers (general):**
- `assertActiveMember(sharedId, userId)` → boolean/throw 403
- `assertRole(sharedId, userId, roles)` → owner/manager checks
- `assertContextOwnership(type, id, userId)` → user is self or member of shared

---

## 2. Reservation Lifecycle (Critical Path)

1. **Initiate Trade (initiator)**  
   - Validate initiator context (user/self or shared membership).  
   - Check available pages for initiator; **create reservations** per canonical id & qty.  
   - Create `active_trades` with context columns set.

2. **Confirm Trade (target)**  
   - Validate target context; check availability; **create reservations** for target.  
   - Apply deltas (decrement from each offering owner, increment to the other side).  
   - Delete reservations for the trade; move to `completed_trades`.

3. **Cancel/Decline**  
   - Delete reservations for `active_trade_id` and set status accordingly.

4. **Timeout/Lease Expiry (optional)**  
   - Scheduled Worker: purge reservations where `expires_at < now()`; notify parties?

---

## 3. Security Controls

- **Prepared statements only**; never interpolate.  
- **User isolation** on all queries.  
- **Membership checks** on shared routes.  
- **Context validation** on trades & matching.  
- **Input validation**: JSON parse guards & schema checks.  
- **CORS**: include `PATCH` (already fixed previously).  
- **Logging**: structured logs in DEV only (`env.ENVIRONMENT === 'development'`).  
- **Rate limits** (optional future).

---

## 4. Errors & Responses

- `401 Unauthorized` missing/invalid JWT.  
- `403 Forbidden` not a member / insufficient role.  
- `409 Conflict` insufficient available pages (reserved elsewhere).  
- `404 Not Found` resources.  
- `422 Unprocessable Entity` invalid payload.  
- `500` generic server errors with safe message.

---

## 5. Example Pseudocode (Workers)

**Initiate trade** (simplified):
```ts
// Validate initiator context
if (!canActForContext(initiator_context_type, initiator_context_id, userId)) return 403

// Compute availability and verify
for (const page of initiator_offering_pages) assertAvailable(initiator_context, page)

// Create trade id, insert active_trades
// Create reservations for initiator pages (one row per canonical id)
// Return trade payload + deep link
```

**Confirm trade** (simplified):
```ts
if (!canActForContext(target_context_type, target_context_id, userId)) return 403
for (const page of target_offering_pages) assertAvailable(target_context, page)

// Apply inventory deltas (batched)
// Delete reservations for this trade
// Insert into completed_trades; set status
```

---

## 6. Repository Path Guidance (Specific + General)

- **General:** API Worker routes, middleware.  
- **Likely paths:** `cloudflare-workers/api/src/routes/`, `cloudflare-workers/api/src/middleware/`  
- New: `shared-inventories.ts`  
- Updates: `trades.ts`, `matching.ts`, `inventories.ts`

> Agents: Confirm structure/imports before coding.

---

## Commit Checkpoint

```bash
git add cloudflare-workers/api/src/routes cloudflare-workers/api/src/middleware
git commit -m "feat(shared-inventories): stage 04 backend routing & reservation lifecycle"
```

## Summary/Refresh Gate

- Summarize to ~2–3k tokens: reservation lifecycle & key security validations.
