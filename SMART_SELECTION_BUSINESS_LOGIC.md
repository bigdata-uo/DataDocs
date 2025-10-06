# Smart Selection Business Logic Specification

**Created:** October 6, 2025  
**Purpose:** Define comprehensive business rules for optimal lore page trading algorithm  
**Status:** Draft - Pending Review

## Executive Summary

The Smart Selection system determines optimal lore page trades between users by analyzing complete inventory intersections, applying book completion priorities, and balancing creature vs non-creature page weights to create fair, efficient exchanges capped at 120 pages per trade.

## Core Business Rules

### 1. Trade Discovery Phase

**Objective:** Identify ALL possible trade opportunities for a session user across the entire user base.

**Process:**
1. **Query session user's complete inventory** with `is_need=true` and `is_have=true` flags
2. **Query ALL other users' inventories** (excluding session user) with need/have flags
3. **Calculate intersections** for each potential trading partner:
   - `session_user.need ∩ other_user.have` = Pages session user would receive
   - `other_user.need ∩ session_user.have` = Pages session user would offer
4. **Filter valid partners:** Only include users with mutual trade opportunities (both directions > 0)
5. **No artificial limits** applied during discovery phase

**Key Principle:** Discover the complete universe of trading possibilities before optimization.

### 2. Trade Optimization Phase

**Objective:** Create balanced, priority-weighted trades capped at practical limits.

#### 2.1 Volume Limits
- **Hard Cap:** Maximum 120 unique pages per trade direction
- **Rationale:** UO bag limit of 125 items, leaving 5 slots for other items
- **Application:** Applied after all other optimizations complete

#### 2.2 Book Completion Priority System

**Data Requirements:**
- `book_completion` table must be current for all trading partners
- Completion percentage calculated as: `(owned_pages / total_pages) * 100`

**Priority Scoring:**
- **90-100% complete:** Priority Score = 100 (Highest - complete the book!)
- **75-89% complete:** Priority Score = 75 (High - close to completion)
- **50-74% complete:** Priority Score = 50 (Medium)
- **25-49% complete:** Priority Score = 25 (Low)
- **0-24% complete:** Priority Score = 10 (Lowest)

**Application:**
- Each page receives the priority score of its book
- Higher priority pages selected first within their category

#### 2.3 Page Type Weighting System

**Creature Books:** (15 pages each)
- **Weight:** 1.0 (standard)
- **Books:** All books starting with "Creatures of "
- **Philosophy:** Creature pages are common, less valuable

**Non-Creature Books:** (10 pages each)
- **Weight:** 1.5 (premium)
- **Books:** History, Tales, Outlands Mythology series
- **Philosophy:** Non-creature pages are rare, more valuable

**Balancing Rules:**
1. **Prefer non-creature trades** when both sides have options
2. **Equitable creature distribution** - if creatures must be traded, balance them across both sides
3. **Last resort creatures** - only trade creatures when no non-creature alternatives exist

### 3. Trade Selection Algorithm

**Step-by-Step Process:**

#### Step 1: Collect Complete Trade Universe
```typescript
for each user in database (excluding session_user) {
  pagesUserCanReceive = intersect(session_user.need, other_user.have)
  pagesUserCanOffer = intersect(other_user.need, session_user.have)
  
  if (both arrays > 0) {
    add to potential_trades[]
  }
}
```

#### Step 2: Calculate Book Completion Data
```typescript
for each user in potential_trades {
  calculate_book_completion_percentages(user.id)
  update_completion_priority_scores()
}
```

#### Step 3: Apply Priority Weighting
```typescript
for each potential_trade {
  for each page in trade {
    page.priority_score = book_completion_priority[page.book]
    page.type_weight = is_creature_book(page.book) ? 1.0 : 1.5
    page.final_score = page.priority_score * page.type_weight
  }
}
```

#### Step 4: Optimal Page Selection
```typescript
for each potential_trade {
  // Sort pages by final_score (descending)
  user_receives = sort_by_score(pagesUserCanReceive).slice(0, 120)
  user_offers = sort_by_score(pagesUserCanOffer).slice(0, 120)
  
  // Balance the trade to equal page counts
  balanced_count = min(user_receives.length, user_offers.length)
  final_trade = {
    receive: user_receives.slice(0, balanced_count),
    offer: user_offers.slice(0, balanced_count)
  }
}
```

#### Step 5: Trade Ranking
```typescript
// Rank trades by total value
for each optimized_trade {
  trade.total_score = sum(receive_pages.final_scores) + sum(offer_pages.final_scores)
  trade.page_count = trade.receive.length
  trade.completion_books = count_books_that_would_complete()
}

// Sort by: completion_books DESC, total_score DESC, page_count DESC
return sorted_trades
```

## Business Logic Implementation

### 4. Exception Handling

#### 4.1 Data Integrity
- **Missing book_completion data:** Use default priority score of 25
- **Invalid canonical_ids:** Skip and log error
- **Corrupted is_need/is_have flags:** Recalculate from count thresholds

#### 4.2 Edge Cases
- **Single page trades:** Allowed, no minimum trade size
- **Asymmetric inventories:** Use smaller side for balanced trade
- **Zero mutual interest:** Return empty trade array
- **All creatures vs all non-creatures:** Apply weighting but allow trade

#### 4.3 Performance Considerations
- **Cache book completion data** for session duration
- **Limit concurrent trade calculations** to prevent database overload
- **Timeout protection** for large inventory intersections

### 5. Quality Assurance Rules

#### 5.1 Trade Validation
- **Verify mutual benefit:** Both sides must receive pages they need
- **Verify page availability:** All offered pages must be in user's have array
- **Verify count limits:** Total pages ≤ 120 per direction
- **Verify book completion accuracy:** Recalculate completion percentages

#### 5.2 User Experience
- **Clear trade rationale:** Show why specific pages were selected
- **Priority transparency:** Display completion percentages and weights
- **Alternative options:** Show other available trades if user declines
- **Script safety:** Generated UO scripts must not exceed bag limits

## Algorithm Replacement Strategy

### 6. Implementation Plan

#### 6.1 New Function Development
- **Function Name:** `calculateOptimalTrades()`
- **Input:** `sessionUserId: string, options?: TradeOptions`
- **Output:** `OptimalTrade[]` sorted by value
- **Location:** New file `src/lib/optimal-trading.ts`

#### 6.2 Integration Points
- **Replace:** `calculateTradeMatch()` in TradingTab.tsx
- **Update:** Smart selection UI logic
- **Maintain:** Existing trade initiation and completion flows
- **Preserve:** Two-phase commit system and database functions

#### 6.3 Testing Strategy
- **Unit tests:** Each algorithm step independently
- **Integration tests:** Full trade flow with test data
- **Performance tests:** Large inventory stress testing
- **User acceptance:** A/B test with existing algorithm

### 7. Success Metrics

#### 7.1 Trade Quality
- **Completion rate:** Percentage of initiated trades that complete
- **User satisfaction:** Feedback on trade fairness and value
- **Book completions:** Number of books completed per trade
- **Trade frequency:** Average trades per active user

#### 7.2 System Performance
- **Response time:** < 2 seconds for trade calculation
- **Database efficiency:** Minimize query count and complexity
- **Memory usage:** Handle large inventories without memory leaks
- **Error rate:** < 1% of trade calculations fail

## Conclusion

This Smart Selection business logic creates a sophisticated, fair, and efficient trading system that prioritizes user goals (book completion) while respecting practical constraints (bag limits, page rarity). The algorithm ensures complete trade discovery followed by intelligent optimization, replacing the current truncation-based approach with a value-maximizing system.

**Next Steps:**
1. Review and approve this business logic specification
2. Develop `calculateOptimalTrades()` function per this specification
3. Create comprehensive test suite
4. Implement gradual rollout with fallback to existing system
5. Monitor performance and user feedback for further optimization

---

**Document Status:** Ready for review and approval before implementation begins.