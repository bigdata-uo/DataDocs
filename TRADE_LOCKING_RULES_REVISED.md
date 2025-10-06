# Trade Visibility & Locking Rules - REVISED

**Date:** October 5, 2025  
**Status:** Updated with nuanced trade states  
**Key Change:** Multiple pending requests allowed, but only one active trade

---

## Core Principles (Revised)

### 1. **Pending Requests Are Non-Exclusive**
- ✅ User can receive multiple pending trade requests
- ✅ User remains visible in other people's match lists
- ✅ User's inventory is still current (they haven't accepted yet)
- ❌ User cannot initiate NEW requests while they have pending outbound request
- ✅ User can still RECEIVE requests while they have pending outbound request

### 2. **Active Trades Are Exclusive**
- ❌ User can only have ONE active trade at a time
- ✅ User remains visible but "dimmed" in match lists
- ❌ User cannot receive new trade requests (buttons disabled)
- ❌ User cannot initiate new trade requests
- ✅ Must complete or cancel active trade to unlock

### 3. **Visual States Matter**
- 🔓 **Available:** Normal brightness, "Request Trade" button enabled
- ⏳ **Pending (I initiated):** Normal brightness, "Cancel Trade" button, waiting for response
- 📬 **Pending (They initiated):** Normal brightness, "Accept/Reject" buttons
- 🔒 **Active (Trading):** Dimmed, "Trade in Progress" badge, buttons disabled
- 🟢 **My Active Trade:** Full brightness, "Complete/Cancel" buttons

---

## User States & Capabilities

### State A: No Trades (Free)
**User Status:** Not in any trades  
**Can Do:**
- ✅ Initiate trade requests (becomes "Pending Outbound")
- ✅ Receive trade requests (moves to "Pending Inbound")
- ✅ See all available users

**What Others See:**
- ✅ Full brightness
- ✅ "Request Trade" button enabled
- ✅ No badge

---

### State B: Pending Outbound (I Initiated)
**User Status:** Sent trade request, waiting for response  
**Can Do:**
- ❌ Cannot initiate MORE trade requests (locked on outbound)
- ✅ CAN still receive trade requests from others
- ✅ Can cancel my outbound request (unlocks me)
- ✅ See all available users (but cannot request from them)

**What Others See:**
- ✅ Full brightness (I'm still available to receive)
- ✅ "Request Trade" button enabled for them
- ⚠️ Small badge: "Has pending request" (optional, informational)

**Example:**
```
User A requests trade with User B
  ↓
User A: LOCKED from initiating new requests
        CAN receive requests from User C, D, E
        Can cancel request to User B
  ↓
User B: Sees notification from User A
User C: Can still request trade with User A
```

---

### State C: Pending Inbound (Received Request)
**User Status:** Has 1+ pending trade requests from others  
**Can Do:**
- ✅ Can initiate NEW trade requests (not locked)
- ✅ Can receive MORE trade requests
- ✅ Can accept ONE request (moves to "Active")
- ✅ Can reject requests (no state change)

**What Others See:**
- ✅ Full brightness
- ✅ "Request Trade" button enabled
- ⚠️ Small badge: "1 pending request" (optional)

**Example:**
```
User A requests trade with User B
User C requests trade with User B
User D requests trade with User B
  ↓
User B: Has 3 pending inbound requests
        Can still request trade with User E
        Can accept ONE trade (e.g., User C)
        Other 2 requests remain pending until B accepts one
```

---

### State D: Pending Both (Outbound + Inbound)
**User Status:** Sent request + received request(s)  
**Can Do:**
- ❌ Cannot initiate MORE requests (locked by outbound)
- ✅ Can receive MORE requests
- ✅ Can accept any inbound request (moves to "Active")
- ✅ Can cancel my outbound request (partial unlock)

**What Others See:**
- ✅ Full brightness
- ✅ "Request Trade" button enabled
- ⚠️ Badge: "Has pending request"

---

### State E: Active Trade (Accepted)
**User Status:** Accepted a trade, in progress  
**Can Do:**
- ❌ Cannot initiate new requests (LOCKED)
- ❌ Cannot receive new requests (LOCKED)
- ❌ Cannot accept other pending requests (LOCKED)
- ✅ Can complete active trade (unlocks)
- ✅ Can cancel active trade (unlocks)
- ⚠️ **Other pending requests remain queued** (not auto-rejected)

**What Others See:**
- 🔒 **Dimmed** card
- 🔒 "Trade in Progress" badge
- ❌ "Request Trade" button DISABLED (or removed)
- 💬 "Cannot receive requests while trading"

**Example:**
```
User B has 3 pending inbound requests (A, C, D)
User B accepts request from User C
  ↓
User B + User C: Both LOCKED, trade is ACTIVE
  ↓
Requests from A and D: Still pending, not rejected
  ↓
User E tries to request trade with User B:
  ❌ Button disabled, sees "Trade in Progress"
  ↓
User B completes trade with User C:
  ✅ Unlocked
  ✅ Requests from A and D still valid
  ✅ User E can now request
```

---

## Revised Database Schema

### active_trades Table - Status Values

```sql
-- Updated status constraint
ALTER TABLE active_trades
DROP CONSTRAINT IF EXISTS active_trades_status_check;

ALTER TABLE active_trades
ADD CONSTRAINT active_trades_status_check 
CHECK (status IN (
  'pending_target_acceptance',  -- Initiator locked, target can still receive others
  'active',                      -- Both locked, trade in progress
  'cancelled'                    -- Soft delete for audit trail
));

ALTER TABLE active_trades
ALTER COLUMN status SET DEFAULT 'pending_target_acceptance';
```

**Key Change:** `pending_target_acceptance` does NOT lock the target from receiving other requests

---

### Query Logic - Get My Proposed Trades (REVISED)

```sql
CREATE OR REPLACE FUNCTION get_my_proposed_trades(p_user_id UUID)
RETURNS TABLE (
  partner_id UUID,
  partner_username TEXT,
  partner_avatar_url TEXT,
  pages_i_offer INTEGER[],
  pages_i_request INTEGER[],
  match_score INTEGER,
  proposed_trade_id UUID,
  partner_status TEXT  -- NEW: 'available', 'pending', 'active'
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    CASE 
      WHEN pt.user_a_id = p_user_id THEN pt.user_b_id
      ELSE pt.user_a_id
    END as partner_id,
    
    CASE 
      WHEN pt.user_a_id = p_user_id THEN pt.user_b_username
      ELSE pt.user_a_username
    END as partner_username,
    
    CASE 
      WHEN pt.user_a_id = p_user_id THEN pb.avatar_url
      ELSE pa.avatar_url
    END as partner_avatar_url,
    
    CASE 
      WHEN pt.user_a_id = p_user_id THEN pt.user_a_offers
      ELSE pt.user_b_offers
    END as pages_i_offer,
    
    CASE 
      WHEN pt.user_a_id = p_user_id THEN pt.user_a_requests
      ELSE pt.user_b_requests
    END as pages_i_request,
    
    pt.match_score,
    pt.id as proposed_trade_id,
    
    -- Determine partner's status
    CASE
      -- Partner has ACTIVE trade (locked)
      WHEN EXISTS (
        SELECT 1 FROM active_trades at
        WHERE (at.initiator_user_id = CASE WHEN pt.user_a_id = p_user_id THEN pt.user_b_id ELSE pt.user_a_id END
           OR at.target_user_id = CASE WHEN pt.user_a_id = p_user_id THEN pt.user_b_id ELSE pt.user_a_id END)
          AND at.status = 'active'
      ) THEN 'active'
      
      -- Partner has PENDING request (either inbound or outbound)
      WHEN EXISTS (
        SELECT 1 FROM active_trades at
        WHERE (at.initiator_user_id = CASE WHEN pt.user_a_id = p_user_id THEN pt.user_b_id ELSE pt.user_a_id END
           OR at.target_user_id = CASE WHEN pt.user_a_id = p_user_id THEN pt.user_b_id ELSE pt.user_a_id END)
          AND at.status = 'pending_target_acceptance'
      ) THEN 'pending'
      
      -- Partner is available
      ELSE 'available'
    END as partner_status
    
  FROM proposed_trades pt
  JOIN profiles pa ON pt.user_a_id = pa.id
  JOIN profiles pb ON pt.user_b_id = pb.id
  
  WHERE (pt.user_a_id = p_user_id OR pt.user_b_id = p_user_id)
    AND pt.status = 'proposed'
    AND pt.expires_at > NOW()
    
    -- NO LONGER FILTER OUT USERS WITH PENDING TRADES
    -- Everyone is visible (may be dimmed in UI based on partner_status)
    
  ORDER BY 
    -- Sort my active trade to top
    CASE WHEN EXISTS (
      SELECT 1 FROM active_trades at
      WHERE (at.initiator_user_id = p_user_id OR at.target_user_id = p_user_id)
        AND (at.initiator_user_id = CASE WHEN pt.user_a_id = p_user_id THEN pt.user_b_id ELSE pt.user_a_id END
         OR at.target_user_id = CASE WHEN pt.user_a_id = p_user_id THEN pt.user_b_id ELSE pt.user_a_id END)
        AND at.status = 'active'
    ) THEN 0 ELSE 1 END,
    pt.match_score DESC;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Key Changes:**
- ✅ ALL users returned (no filtering)
- ✅ Returns `partner_status` field ('available', 'pending', 'active')
- ✅ UI decides how to display based on status

---

## UI Display Logic (REVISED)

### Trade Match Card - Visual States

#### 1. Available (No Trades)
```
┌─────────────────────────────────────┐
│ 👤 Username                         │
│ Score: 25 • Last seen: 2 hours ago  │
├─────────────────────────────────────┤
│ You want: 12 pages                  │
│ They want: 13 pages                 │
├─────────────────────────────────────┤
│ [Request Trade] 🤝 (ENABLED)        │
└─────────────────────────────────────┘
```
**CSS:** `opacity: 1`, normal colors

---

#### 2. Pending (Partner has pending request elsewhere)
```
┌─────────────────────────────────────┐
│ 👤 Username               ⏳ PENDING│
│ Score: 25 • Last seen: 2 hours ago  │
├─────────────────────────────────────┤
│ You want: 12 pages                  │
│ They want: 13 pages                 │
├─────────────────────────────────────┤
│ [Request Trade] 🤝 (ENABLED)        │
│ ℹ️ User has a pending request       │
└─────────────────────────────────────┘
```
**CSS:** `opacity: 1`, small info badge  
**Behavior:** Fully clickable, informational only

---

#### 3. Active (Partner in active trade)
```
┌─────────────────────────────────────┐
│ 👤 Username              🔒 TRADING │
│ Score: 25 • Last seen: 2 hours ago  │
├─────────────────────────────────────┤
│ You want: 12 pages                  │
│ They want: 13 pages                 │
├─────────────────────────────────────┤
│ [Request Trade] 🤝 (DISABLED)       │
│ 🔒 Currently in another trade       │
└─────────────────────────────────────┘
```
**CSS:** `opacity: 0.5`, grayed out  
**Behavior:** Button disabled, cannot interact

---

#### 4. My Pending Outbound (I requested from them)
```
┌─────────────────────────────────────┐
│ 👤 Username                  ⏳ SENT│
│ Waiting for their response...       │
├─────────────────────────────────────┤
│ You offered: 13 pages               │
│ You requested: 12 pages             │
├─────────────────────────────────────┤
│ [📋 Copy Discord Message]           │
│ [📜 Copy Pull/Sort Script]          │
│ [❌ Cancel Request]                 │
└─────────────────────────────────────┘
```
**CSS:** `opacity: 1`, highlighted border  
**Behavior:** Can cancel, can copy messages

---

#### 5. My Pending Inbound (They requested from me)
```
┌─────────────────────────────────────┐
│ 🔔 TRADE REQUEST                    │
│ 👤 Username wants to trade!         │
├─────────────────────────────────────┤
│ They're offering: 12 pages          │
│ You would give: 13 pages            │
├─────────────────────────────────────┤
│ [✅ Accept Trade] (PRIMARY)         │
│ [❌ Reject Trade] (SECONDARY)       │
└─────────────────────────────────────┘
```
**CSS:** `opacity: 1`, green highlight  
**Behavior:** PINNED TO TOP, can accept/reject

---

#### 6. My Active Trade
```
┌─────────────────────────────────────┐
│ 👤 Username                  ✅ ACTIVE│
│ Trade in progress                   │
├─────────────────────────────────────┤
│ You're offering: 13 pages           │
│ You're requesting: 12 pages         │
├─────────────────────────────────────┤
│ [📋 Copy Discord Message]           │
│ [📜 Copy Pull/Sort Script]          │
│ [✅ Complete Trade]                 │
│ [❌ Cancel Trade]                   │
└─────────────────────────────────────┘
```
**CSS:** `opacity: 1`, blue highlight  
**Behavior:** PINNED TO TOP, can complete/cancel

---

## Locking Rules Matrix (REVISED)

| My Status | Can Initiate New? | Can Receive New? | Others See Me As | Can They Request? |
|-----------|-------------------|------------------|------------------|-------------------|
| **No Trade** | ✅ YES | ✅ YES | 🟢 Available | ✅ YES |
| **Pending Outbound** | ❌ NO (locked) | ✅ YES | ⏳ Pending | ✅ YES |
| **Pending Inbound** | ✅ YES | ✅ YES | ⏳ Pending | ✅ YES |
| **Pending Both** | ❌ NO (locked) | ✅ YES | ⏳ Pending | ✅ YES |
| **Active Trade** | ❌ NO (locked) | ❌ NO (locked) | 🔒 Trading | ❌ NO |

---

## Button Availability Matrix (REVISED)

| Card Type | Request Trade | Accept | Reject | Cancel | Complete | Copy Discord | Copy Script |
|-----------|--------------|--------|--------|--------|----------|--------------|-------------|
| **Available** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Pending (Partner)** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Active (Partner)** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **My Pending Out** | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ |
| **My Pending In** | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **My Active** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |

---

## Client-Side Validation (REVISED)

### Pre-Request Check (Updated)
```typescript
const handleInitiateTrade = async (
  targetUserId: string,
  targetUsername: string,
  pagesOffered: number[],
  pagesRequested: number[],
  creaturesOnly: boolean = false
) => {
  try {
    // CHECK 1: Do I have a PENDING OUTBOUND request?
    const { data: myOutboundTrades } = await supabase
      .from('active_trades')
      .select('*')
      .eq('initiator_user_id', user.id)
      .eq('status', 'pending_target_acceptance')
    
    if (myOutboundTrades && myOutboundTrades.length > 0) {
      showToast('❌ You already have a pending trade request. Cancel it first.', '✗')
      return
    }

    // CHECK 2: Do I have an ACTIVE trade?
    const { data: amITrading } = await isUserCurrentlyTrading(user.id)
    if (amITrading) {
      showToast('❌ You already have an active trade', '✗')
      return
    }

    // CHECK 3: Does target have an ACTIVE trade? (Not just pending)
    const { data: targetActiveTrades } = await supabase
      .from('active_trades')
      .select('*')
      .or(`initiator_user_id.eq.${targetUserId},target_user_id.eq.${targetUserId}`)
      .eq('status', 'active')
    
    if (targetActiveTrades && targetActiveTrades.length > 0) {
      showToast(`❌ ${targetUsername} is currently in an active trade`, '✗')
      return
    }

    // Proceed with trade initiation
    await initiateTradeDirectly(targetUserId, pagesOffered, pagesRequested, creaturesOnly)
  } catch (error) {
    console.error('Error initiating trade:', error)
    showToast('❌ Error initiating trade', '✗')
  }
}
```

**Key Changes:**
- ✅ Check for MY pending outbound (not inbound)
- ✅ Check for target's ACTIVE trade (not pending)
- ✅ Allow requesting if target has pending trades

---

### Accept Trade Check (Updated)
```typescript
const handleAcceptTrade = async (tradeId: string) => {
  try {
    // CHECK 1: Do I already have an ACTIVE trade?
    const { data: myActiveTrade } = await supabase
      .from('active_trades')
      .select('*')
      .or(`initiator_user_id.eq.${user.id},target_user_id.eq.${user.id}`)
      .eq('status', 'active')
      .single()
    
    if (myActiveTrade) {
      showToast('❌ You already have an active trade. Complete or cancel it first.', '✗')
      return
    }

    // Proceed with acceptance
    await acceptTradeDirectly(tradeId)
    
    // Automatically reject OTHER pending inbound requests
    await rejectOtherPendingRequests(user.id, tradeId)
    
  } catch (error) {
    console.error('Error accepting trade:', error)
    showToast('❌ Error accepting trade', '✗')
  }
}
```

**New Behavior:**
- ✅ When accepting trade, OTHER pending inbound requests remain (not rejected)
- ✅ They become "stale" but user can still see them
- ✅ User can accept one after completing first trade

---

## Scenario Walkthroughs (REVISED)

### Scenario 1: Multiple Pending Requests

```
T+0: User A requests trade with User B
     ↓
     User A: LOCKED from initiating new requests
             CAN still receive requests
     User B: Sees "Accept/Reject" buttons
             CAN still receive other requests
             CAN still initiate requests

T+1: User C requests trade with User B
     ↓
     User C: LOCKED from initiating new requests
     User B: Now has 2 pending inbound requests
             Both shown in UI (Accept/Reject for each)

T+2: User D requests trade with User B
     ↓
     User D: LOCKED from initiating
     User B: Has 3 pending inbound requests

T+3: User E tries to request from User B
     ↓
     User E: Button ENABLED (User B not locked)
     ✅ Request goes through
     User B: Now has 4 pending requests

T+4: User B accepts request from User C
     ↓
     User B + User C: Both ACTIVE, both LOCKED
     Users A, D, E: Requests still pending (not rejected)
     Users A, D, E: See "User B is in a trade" (dimmed)

T+5: User B completes trade with User C
     ↓
     User B: UNLOCKED
     Users A, D, E: Requests still valid
     Users A, D, E: User B appears available again
     User B: Can now accept another pending request
```

---

### Scenario 2: Initiator Gets Counter-Offer

```
T+0: User A requests trade with User B
     ↓
     User A: LOCKED from new requests
             CAN receive requests

T+1: User C sees User A (shows "⏳ PENDING" badge)
     User C: Button still ENABLED
     User C: Requests trade with User A
     ↓
     User A: Has 1 outbound (to B), 1 inbound (from C)
             STILL LOCKED from new requests
             CAN accept C's request

T+2: User A accepts request from User C
     ↓
     User A: ACTIVE with User C
     User B: Still has pending request from A (not auto-cancelled)
     User B: Sees "User A is in a trade" (dimmed)

T+3: User A completes trade with User C
     ↓
     User A: UNLOCKED
     User B: Can now accept A's original request
```

---

## Summary of Changes

### What Changed
1. ✅ **Pending requests are non-exclusive**
   - Target can receive multiple requests
   - Target can initiate their own requests
   - Only initiator is locked

2. ✅ **All users always visible**
   - No hiding of cards
   - "Dimmed" visual state for active trades
   - Informational badges for pending

3. ✅ **Smart button states**
   - Disabled for users in active trades
   - Enabled for users with pending (just info badge)
   - Clear visual feedback

### What Stayed Same
1. ✅ **One active trade at a time** (exclusive)
2. ✅ **Symmetry enforcement** (database constraints)
3. ✅ **Two-phase commit** (request → accept → active)
4. ✅ **Real-time updates** (Supabase subscriptions)

---

## Implementation Priority (Updated)

### Phase 1: Fix Blocking Bug (15 min)
1. Fix `initiate_trade()` function column names
2. Test trade persistence

### Phase 2: Update Locking Logic (2 hours)
1. Update `get_my_proposed_trades()` to return ALL users with status
2. Add `partner_status` field to return type
3. Update UI to render cards based on status (dim/enable/disable)
4. Update pre-request validation (check outbound, not inbound)

### Phase 3: Multi-Request Support (1 hour)
1. Allow multiple inbound pending requests
2. Keep cards visible with status badges
3. Test accept/reject flow with multiple pending

### Phase 4: Real-Time Polish (1 hour)
1. Subscribe to active_trades changes
2. Update card status in real-time (dim when partner goes active)
3. Test concurrent requests

---

**Ready to implement this revised logic?**
