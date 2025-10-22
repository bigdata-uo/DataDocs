# UO Outlands Guild Officer - Project Overview

**Project Name:** UO Outlands Guild Officer  
**Initial Guild:** [L%C] - Loot Crusade  
**Status:** Planning Phase  
**Created:** October 21, 2025  
**Version:** 1.0

---

## 📋 Project Description

A web-based guild management platform for UO Outlands guilds to track boss tokens, manage Omni Boss progression, and optimize dungeon farming based on weekly bonuses. 

**MVP Scope:** Boss Token Tracker + Omni Boss Progress Calculator for [L%C] guild.

**Future Vision:** Multi-guild platform with Discord authentication, treasury management, auction systems, and inter-guild communication.

---

## 🎯 Project Goals

### Primary Goals (MVP)
1. **Track Boss Tokens** - Maintain an inventory of all boss tokens the guild has collected
2. **Calculate Omni Progress** - Show what tokens are needed to complete the next Tome of Heroism
3. **Priority Recommendations** - Suggest which dungeons to farm based on needed tokens

### Secondary Goals (Future)
1. **Weekly Bonus Integration** - Track and optimize farming based on active bonuses
2. **Multi-Guild Support** - Allow multiple guilds to use the platform
3. **Discord Integration** - Login via Discord, sync member rosters
4. **Guild Communication** - Cross-guild messaging and coordination
5. **Treasury Management** - Track guild finances and gold contributions
6. **Member Management** - Sync in-game names with Discord identities
7. **Auction System** - Inter-guild trading and auctions

---

## 🛠️ Tech Stack

### Frontend
- **Framework:** React 18 + TypeScript
- **Build Tool:** Vite
- **UI Library:** shadcn/ui + Radix UI primitives
- **Styling:** Tailwind CSS (utility-first, dark theme default)
- **Icons:** Lucide React
- **Hosting:** Cloudflare Pages

### Backend
- **Runtime:** Cloudflare Workers (serverless edge functions)
- **Database:** Cloudflare D1 (SQLite-based)
- **API Style:** RESTful JSON API
- **Language:** TypeScript

### Authentication (Future)
- **OAuth:** Discord OAuth 2.0
- **Tokens:** JWT for session management
- **MVP:** Simple token-based auth (single guild)

### Development Tools
- **Package Manager:** npm
- **CLI:** Wrangler (Cloudflare Workers CLI)
- **Version Control:** Git
- **Code Editor:** Cursor IDE
- **Shell:** PowerShell (Windows)

### Why This Stack?
- ✅ **Free Tier Friendly** - Cloudflare generous limits
- ✅ **Global Distribution** - Edge deployment for all users
- ✅ **Developer Experience** - Existing rules/patterns from OutlandsLoreTrades project
- ✅ **Type Safety** - TypeScript end-to-end
- ✅ **Performance** - Serverless + SQLite at the edge
- ✅ **Scalability** - Built to handle multiple guilds

---

## 📊 Data Model

### Design Principles
1. **Multi-tenant from Day 1** - Schema supports multiple guilds
2. **L%C as First User** - Initially populated with only [L%C] data
3. **Scalable** - Easy to add more guilds without schema changes
4. **Normalized** - Proper relationships, avoid data duplication
5. **Flexible** - JSON columns where appropriate for game data

### Core Tables

#### `guilds`
```sql
CREATE TABLE guilds (
  id TEXT PRIMARY KEY,                -- UUID
  name TEXT NOT NULL,                 -- "Loot Crusade"
  tag TEXT NOT NULL,                  -- "L%C"
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  
  -- Customization (future)
  logo_url TEXT,
  banner_url TEXT,
  primary_color TEXT,
  secondary_color TEXT,
  
  -- Settings
  is_active INTEGER DEFAULT 1 CHECK (is_active IN (0, 1))
);
```

#### `dungeons`
Game metadata - seeded from UO Outlands knowledge base.

```sql
CREATE TABLE dungeons (
  id TEXT PRIMARY KEY,                -- "shadowspire-cathedral"
  name TEXT NOT NULL,                 -- "Shadowspire Cathedral"
  type TEXT NOT NULL,                 -- "outlands-specific", "classic", "special"
  
  -- Boss Information
  has_boss INTEGER DEFAULT 0 CHECK (has_boss IN (0, 1)),
  boss_name TEXT,                     -- "The Lich Lord"
  boss_token_id TEXT,                 -- FK to tokens table
  
  -- Mini-Boss Information
  has_miniboss INTEGER DEFAULT 0 CHECK (has_miniboss IN (0, 1)),
  miniboss_name TEXT,
  miniboss_token_id TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (boss_token_id) REFERENCES tokens(id),
  FOREIGN KEY (miniboss_token_id) REFERENCES tokens(id)
);
```

#### `tokens`
Game metadata - all unique boss/mini-boss tokens in the game.

```sql
CREATE TABLE tokens (
  id TEXT PRIMARY KEY,                -- "shadowspire-boss-token"
  name TEXT NOT NULL,                 -- "Shadowspire Boss Token"
  type TEXT NOT NULL,                 -- "boss" or "miniboss"
  source_type TEXT NOT NULL,          -- "dungeon", "ocean", "wilderness"
  source_dungeon_id TEXT,             -- FK to dungeons (nullable for ocean/wilderness)
  
  -- Metadata
  rarity TEXT,                        -- "common", "rare", "epic" (if relevant)
  description TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (source_dungeon_id) REFERENCES dungeons(id)
);
```

#### `guild_token_inventory`
Guild's current token holdings.

```sql
CREATE TABLE guild_token_inventory (
  id TEXT PRIMARY KEY,                -- UUID
  guild_id TEXT NOT NULL,
  token_id TEXT NOT NULL,
  quantity INTEGER DEFAULT 0,
  
  last_updated TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (guild_id) REFERENCES guilds(id) ON DELETE CASCADE,
  FOREIGN KEY (token_id) REFERENCES tokens(id),
  UNIQUE(guild_id, token_id)          -- One row per guild per token
);
```

#### `tomes`
Tracks Tome of Heroism progress for each guild.

```sql
CREATE TABLE tomes (
  id TEXT PRIMARY KEY,                -- UUID
  guild_id TEXT NOT NULL,
  
  -- Token Progress (JSON arrays of token IDs)
  boss_tokens_added TEXT DEFAULT '[]',      -- ["token-id-1", "token-id-2", ...]
  miniboss_tokens_added TEXT DEFAULT '[]',  -- ["token-id-3", "token-id-4", ...]
  
  -- Status
  is_completed INTEGER DEFAULT 0 CHECK (is_completed IN (0, 1)),
  completed_at TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (guild_id) REFERENCES guilds(id) ON DELETE CASCADE
);
```

#### `omni_encounters`
Historical record of Omni Boss fights.

```sql
CREATE TABLE omni_encounters (
  id TEXT PRIMARY KEY,                -- UUID
  guild_id TEXT NOT NULL,
  tome_id TEXT,                       -- Which tome was used
  
  -- Boss Details
  boss_type TEXT NOT NULL,            -- "Abyssal Daemon", "Astral Daemon", "Storm Daemon"
  summon_date TEXT NOT NULL,
  
  -- Fight Stats
  total_damage_dealt INTEGER,
  players_participated INTEGER,
  mastery_links_earned INTEGER,
  gold_loot_received INTEGER,
  
  -- Notes
  notes TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (guild_id) REFERENCES guilds(id) ON DELETE CASCADE,
  FOREIGN KEY (tome_id) REFERENCES tomes(id)
);
```

#### `weekly_bonuses` (Future)
Tracks active weekly bonuses.

```sql
CREATE TABLE weekly_bonuses (
  id TEXT PRIMARY KEY,                -- UUID
  
  -- Bonus Details
  week_start_date TEXT NOT NULL,
  region_or_dungeon TEXT NOT NULL,    -- "Global", "Shadowspire Cathedral", etc.
  bonus_type TEXT NOT NULL,           -- "Faster Respawn", "Gold Loot", "Experience"
  
  -- Timing
  expires_at TEXT NOT NULL,
  
  -- Cost (if activated by guild)
  society_cost INTEGER,
  activated_by_guild_id TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (activated_by_guild_id) REFERENCES guilds(id)
);
```

#### `members` (Future - Multi-Guild Phase)
```sql
CREATE TABLE members (
  id TEXT PRIMARY KEY,                -- UUID (user ID from Discord eventually)
  
  -- Identity
  discord_id TEXT UNIQUE,
  discord_username TEXT,
  in_game_name TEXT,
  
  -- Guild Membership
  guild_id TEXT NOT NULL,
  role TEXT DEFAULT 'member',         -- "officer", "member"
  
  -- Stats (optional tracking)
  gold_contributed INTEGER DEFAULT 0,
  omni_damage_contributed INTEGER DEFAULT 0,
  
  joined_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (guild_id) REFERENCES guilds(id) ON DELETE CASCADE
);
```

### Relationships
- **One Guild → Many Tokens** (via `guild_token_inventory`)
- **One Guild → Many Tomes** (multiple Tomes in progress/completed)
- **One Guild → Many Omni Encounters** (history of fights)
- **One Guild → Many Members** (future)
- **One Dungeon → One Boss Token** (if has boss)
- **One Dungeon → One Mini-Boss Token** (if has mini-boss)

---

## 🎨 MVP Features

### 1. Boss Token Tracker

**User Stories:**
- **Officer:** "I need to see and update which boss tokens we have, so I can keep inventory current."
- **Viewer:** "I need to see which boss tokens we have, so I know what we're still collecting."

**Functionality:**
- Display all 20+ unique tokens (10 boss, 10+ mini-boss)
- Show current quantity for each token
- Visual indicators:
  - ✅ Green: Guild has this token
  - ⚠️ Yellow: Needed for current Tome
  - ❌ Red: Don't have, needed
- Filter/search tokens by dungeon, type
- **Officer Only:** Add/remove tokens, "Add Token" button, quantity adjustment buttons

**UI Components:**
- Token inventory table/grid (visible to all)
- Token detail cards (visible to all)
- Add/edit token modal (officer only - conditionally rendered)
- Filter sidebar (visible to all)
- "Add Token" button (officer only - NOT rendered for viewers)

### 2. Omni Boss Progress Calculator

**User Stories:**
- **Officer:** "I want to see and manage our Tome progress, so I can coordinate farming and complete Tomes."
- **Viewer:** "I want to see our Tome progress, so I know how close we are to summoning an Omni Boss."

**Functionality:**
- Show current Tome progress (X/10 boss tokens, Y/10 mini-boss tokens)
- List which specific tokens are still needed
- Calculate how many tokens away from completion
- Visual progress bars
- Suggest priority dungeons to farm
- **Officer Only:** "Complete Tome" button, "Start New Tome" button

**UI Components:**
- Tome progress card (visible to all)
- Needed tokens list (visible to all)
- Priority dungeon recommendations (visible to all)
- Complete Tome button (officer only - NOT rendered for viewers)
- Start New Tome button (officer only - NOT rendered for viewers)

### 3. Priority Dungeon Recommendations

**User Story (All Roles):** "I want to know which dungeons to farm, so I can contribute efficiently."

**Functionality:**
- Analyze current token inventory
- Cross-reference with Tome requirements
- Suggest top 5 priority dungeons
- Show which token(s) each dungeon drops
- Show reasoning for priority
- (Future) Factor in weekly bonuses

**UI Components:**
- Priority dungeons card (visible to all)
- Dungeon detail view (visible to all)
- Reasoning display (visible to all)
- Bounty suggestion (future, visible to all)

---

## 🔐 Authentication & Permissions Strategy

### MVP (Phase 1)

#### Authentication
**Simple Token-Based Auth:**
- Single guild ([L%C]) hardcoded
- Two access levels: **Officer** (editor) and **Viewer** (read-only)
- Simple password/token for officers
- No password for viewers (public read access)
- Stored in environment variable or simple D1 table

**Why:**
- Fastest to implement
- No OAuth complexity for MVP
- Proves concept before scaling

#### Permission Levels

**Officer (Editor Role):**
- Can view all data
- Can add/remove tokens from inventory
- Can create/complete Tomes
- Can input weekly bonuses (Phase 2)
- Can access settings/configuration
- **Sees:** Edit buttons, "Add Token" buttons, "Complete Tome" buttons, config menus

**Viewer (Read-Only Role):**
- Can view all data (token inventory, Tome progress, recommendations)
- **Cannot** edit anything
- **Does NOT see:** Edit buttons, "Add" buttons, config menus, officer-only UI elements
- **Seamless experience:** UI looks complete, no indication of missing features
- No "locked" icons or disabled buttons

#### UI Design Philosophy
- **Conditional Rendering:** Officer-only elements are NOT rendered at all for viewers (not just disabled)
- **Identical Layout:** Both roles see the same page structure and information displays
- **No Permission Indicators:** Viewers shouldn't see "You don't have access" messages
- **Officer Badge (Optional):** Small "Officer" badge in header for officers (but viewers don't see "Viewer" badge)

### Future (Phase 3)
**Discord OAuth Integration:**
- Users login via Discord
- Sync Discord roles with guild roles
- Map Discord IDs to in-game names
- JWT tokens for session management
- Role mapping: Discord "Officer" role → Officer permission, everyone else → Viewer

**Migration Path:**
- Current code uses abstract helper functions:
  - `getCurrentGuildId()` - returns guild ID
  - `getUserRole()` - returns "officer" or "viewer"
- Replace helper implementations when migrating
- No changes to business logic or UI components
- Just swap auth layer

**Design Considerations:**
```typescript
// MVP Implementation
async function getCurrentGuildId(request: Request): Promise<string> {
  // Hardcoded for L%C
  return "loot-crusade-guild-id";
}

async function getUserRole(request: Request): Promise<'officer' | 'viewer'> {
  const authHeader = request.headers.get('Authorization');
  if (authHeader === `Bearer ${OFFICER_TOKEN}`) {
    return 'officer';
  }
  return 'viewer'; // Default to viewer for public access
}

// Future Implementation (Phase 3)
async function getCurrentGuildId(request: Request): Promise<string> {
  const jwt = extractJWT(request);
  const user = await verifyJWT(jwt);
  return user.guildId;
}

async function getUserRole(request: Request): Promise<'officer' | 'viewer'> {
  const jwt = extractJWT(request);
  const user = await verifyJWT(jwt);
  // Check Discord role or database role
  return user.role; // 'officer' or 'viewer'
}
```

---

## 📐 Application Architecture

### Frontend Architecture

```
src/
├── components/
│   ├── ui/                     # shadcn/ui components
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── dialog.tsx
│   │   ├── table.tsx
│   │   └── ...
│   │
│   ├── layout/                 # Layout components
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx
│   │   └── Footer.tsx
│   │
│   ├── tokens/                 # Token-related components
│   │   ├── TokenInventoryTable.tsx
│   │   ├── TokenCard.tsx
│   │   ├── AddTokenModal.tsx
│   │   └── TokenFilters.tsx
│   │
│   ├── tomes/                  # Tome-related components
│   │   ├── TomeProgressCard.tsx
│   │   ├── NeededTokensList.tsx
│   │   └── CompleteTomeModal.tsx
│   │
│   └── dungeons/               # Dungeon-related components
│       ├── PriorityDungeonsList.tsx
│       ├── DungeonCard.tsx
│       └── DungeonDetailModal.tsx
│
├── lib/
│   ├── api-client.ts           # D1 API client (similar to OutlandsLoreTrades)
│   ├── types.ts                # TypeScript interfaces
│   └── utils.ts                # Helper functions
│
├── data/
│   └── constants.ts            # Static game data if needed
│
├── App.tsx                     # Main app component
├── main.tsx                    # Entry point
└── index.css                   # Global styles (Tailwind)
```

### Backend Architecture

```
cloudflare-workers/
├── api/
│   ├── src/
│   │   ├── index.ts            # Main entry point, routing
│   │   ├── middleware/
│   │   │   └── auth.ts         # Authentication helpers
│   │   │
│   │   ├── routes/
│   │   │   ├── tokens.ts       # Token CRUD endpoints
│   │   │   ├── tomes.ts        # Tome management endpoints
│   │   │   ├── dungeons.ts     # Dungeon data endpoints
│   │   │   ├── inventory.ts    # Guild inventory endpoints
│   │   │   └── encounters.ts   # Omni encounter tracking
│   │   │
│   │   └── utils/
│   │       ├── db-helpers.ts   # D1 query helpers
│   │       └── responses.ts    # Standard response formatters
│   │
│   ├── wrangler.toml           # Cloudflare configuration
│   ├── package.json
│   └── tsconfig.json
│
└── migrations/                 # D1 database migrations
    ├── 0001_initial_schema.sql
    ├── 0002_seed_dungeons.sql
    ├── 0003_seed_tokens.sql
    └── ...
```

### API Endpoints (MVP)

#### Tokens
- `GET /api/tokens` - List all tokens (game metadata)
- `GET /api/tokens/:id` - Get single token details

#### Inventory
- `GET /api/inventory` - Get guild's current token inventory
- `POST /api/inventory` - Add tokens to inventory
- `PUT /api/inventory/:tokenId` - Update token quantity
- `DELETE /api/inventory/:tokenId` - Remove tokens from inventory

#### Tomes
- `GET /api/tomes` - List guild's tomes (current + completed)
- `GET /api/tomes/:id` - Get specific tome details
- `POST /api/tomes` - Create new tome
- `PUT /api/tomes/:id/add-token` - Add token to tome
- `PUT /api/tomes/:id/complete` - Mark tome as completed

#### Dungeons
- `GET /api/dungeons` - List all dungeons
- `GET /api/dungeons/:id` - Get dungeon details
- `GET /api/dungeons/priority` - Get priority recommendations

#### Encounters (Future)
- `GET /api/encounters` - List Omni encounters
- `POST /api/encounters` - Record new encounter
- `GET /api/encounters/stats` - Guild stats and trends

---

## 🚀 Development Phases

### Phase 1: MVP (Current Focus)
**Goal:** Working boss token tracker + Omni progress calculator for [L%C]

**Tasks:**
1. ✅ Create UO Outlands knowledge base
2. ✅ Design data model
3. ⬜ Set up project structure
4. ⬜ Initialize Cloudflare Workers + D1
5. ⬜ Create database migrations
6. ⬜ Seed dungeon and token data
7. ⬜ Build API endpoints (tokens, inventory, tomes)
8. ⬜ Build React frontend (token tracker)
9. ⬜ Build Tome progress calculator
10. ⬜ Build priority dungeon recommendations
11. ⬜ Simple authentication
12. ⬜ Deploy to Cloudflare Pages + Workers
13. ⬜ Test with [L%C] guild data

**Success Criteria:**
- Officers can view and update token inventory
- System calculates Tome progress accurately
- Recommends correct priority dungeons
- Deployed and accessible to [L%C] members

### Phase 2: Weekly Bonus Integration
**Goal:** Optimize farming recommendations with weekly bonuses

**Tasks:**
1. ⬜ Add weekly bonus tracking UI
2. ⬜ Officers can input current bonuses
3. ⬜ System factors bonuses into priority recommendations
4. ⬜ Display bonus expiration timers
5. ⬜ Show society costs for activating bonuses

**Success Criteria:**
- Weekly bonuses displayed prominently
- Priority recommendations consider active bonuses
- Officers can easily update bonus status

### Phase 3: Multi-Guild Support
**Goal:** Support multiple guilds on the platform

**Tasks:**
1. ⬜ Implement Discord OAuth
2. ⬜ Guild registration flow
3. ⬜ Guild selection/switching UI
4. ⬜ Per-guild customization (logo, colors)
5. ⬜ Guild settings page
6. ⬜ Member management (officers, members)

**Success Criteria:**
- Multiple guilds can use the platform independently
- Proper data isolation between guilds
- Discord login works seamlessly

### Phase 4: Advanced Features
**Goal:** Treasury, auctions, communication

**Tasks:**
1. ⬜ Guild treasury tracking
2. ⬜ Gold contribution tracking
3. ⬜ Member performance stats
4. ⬜ Inter-guild messaging
5. ⬜ Auction system
6. ⬜ Guild roster sync (Discord ↔ in-game names)

**Success Criteria:**
- Guilds can manage finances
- Members can see their contributions
- Cross-guild coordination enabled

---

## 🎨 Design System

### Color Palette

#### [L%C] Theme (Default)
- **Primary:** Custom L%C brand color (TBD)
- **Background:** Dark theme (`bg-background`)
- **Cards:** `bg-card` with subtle borders
- **Text:** `text-foreground` / `text-muted-foreground`
- **Accent:** Discord blue `#5865F2` for Discord actions (future)

#### Token Status Colors
- **Have Token:** Green `text-green-500`
- **Need Token:** Red `text-red-500`
- **In Progress:** Yellow `text-yellow-500`
- **Completed:** Blue `text-blue-500`

### Typography
- **Headers:** Use `CardTitle` component
- **Body:** `text-sm` or `text-base`
- **Font:** System fonts via Tailwind defaults

### Component Patterns
- **Cards** for all major content sections
- **Tables** for token/dungeon lists
- **Modals** for add/edit actions
- **Badges** for status indicators
- **Progress Bars** for Tome progress

---

## 📏 Development Guidelines

### Code Style
- **TypeScript Strict Mode:** Enabled
- **No `any` types:** Use proper types or `unknown`
- **Consistent Naming:**
  - Components: PascalCase
  - Functions: camelCase
  - Constants: UPPER_SNAKE_CASE
  - DB Tables: snake_case

### Git Workflow
- **Branches:** Feature branches off `main`
- **Commits:** Conventional commits (`feat:`, `fix:`, `docs:`)
- **Pre-Commit:** Always run `npm run build`
- **No Direct Push to Main:** Use PRs (even solo dev for history)

### Testing Strategy
- **Manual Testing:** Primary method for MVP
- **Real Data:** Test with actual L%C token data
- **Multiple Browsers:** Chrome, Firefox, Edge
- **Responsive:** Test mobile and desktop views

### Performance Goals
- **First Load:** < 2s
- **API Response:** < 200ms (edge workers)
- **Interactive:** < 100ms for UI actions

---

## 🔧 Environment Setup

### Required Tools
- Node.js 18+ (LTS)
- npm (comes with Node)
- Wrangler CLI (`npm install -g wrangler`)
- Git
- Cursor IDE (or VS Code)

### Environment Variables

#### Frontend (.env)
```bash
VITE_API_URL=http://localhost:8787/api    # Local dev
# VITE_API_URL=https://api.your-domain.workers.dev/api  # Production
```

#### Backend (wrangler.toml secrets)
```bash
# Set via: npx wrangler secret put SECRET_NAME

# MVP
AUTH_TOKEN=<simple-auth-token>            # For officer access

# Future
DISCORD_CLIENT_ID=<discord-app-id>
DISCORD_CLIENT_SECRET=<discord-secret>
DISCORD_REDIRECT_URI=<callback-url>
JWT_SECRET=<random-secret>
```

### Local Development Commands

```powershell
# Frontend
npm install
npm run dev                    # Vite dev server on :5173

# Backend
cd cloudflare-workers/api
npm install
npx wrangler dev              # Workers dev server on :8787

# Database (D1)
npx wrangler d1 migrations apply outlands-guild-officer --local  # Apply migrations locally
npx wrangler d1 migrations apply outlands-guild-officer          # Apply to production
npx wrangler d1 execute outlands-guild-officer --local --command "SELECT * FROM guilds"
```

### Deployment Commands

```powershell
# Backend (Workers)
cd cloudflare-workers/api
npx wrangler deploy

# Frontend (Pages)
npm run build
npx wrangler pages deploy dist --project-name outlands-guild-officer
```

---

## 📊 Success Metrics (MVP)

### Technical
- ✅ All API endpoints respond < 200ms
- ✅ No TypeScript errors in build
- ✅ No console errors in browser
- ✅ Mobile responsive (tested on iPhone & Android)
- ✅ Works on Chrome, Firefox, Edge

### Functional
- ✅ Officers can add/remove tokens from inventory
- ✅ Tome progress calculates correctly (10+10 unique tokens)
- ✅ Priority dungeons update when inventory changes
- ✅ Data persists across sessions (D1 storage works)

### User Experience
- ✅ [L%C] officers can use without tutorial
- ✅ Clear visual feedback for all actions
- ✅ Error messages are helpful
- ✅ Loading states prevent confusion
- ✅ Dark theme is pleasant to use

---

## 🚧 Known Limitations (MVP)

### Intentional Scope Cuts
- ❌ No Discord authentication (coming Phase 3)
- ❌ No multi-guild support (coming Phase 3)
- ❌ No weekly bonus tracking (coming Phase 2)
- ❌ No member management (coming Phase 4)
- ❌ No treasury tracking (coming Phase 4)
- ❌ Manual token entry only (no game API integration)

### Technical Debt (Address Later)
- Simple authentication (no proper user accounts)
- Hardcoded guild ID in codebase
- No automated testing
- No CI/CD pipeline
- No monitoring/logging infrastructure

---

## 📚 Related Documentation

- **Rules:** `.cursor/rules/` (bio.rules.md, FRONTEND_RULES.md, BACKEND_RULES.md)
- **Game Knowledge:** `.cursor/rules/outlands-context.md`
- **Backlog:** `.cursor/rules/backlog.md`
- **Changelog:** `.cursor/rules/changelog.md`

---

## 🎯 Next Steps

1. **User Approval** - Review this overview, provide feedback
2. **Define Business Requirements** - Detailed feature specs
3. **Set up Repository** - Initialize git, create base structure
4. **Create Migrations** - Database schema SQL files
5. **Seed Data** - Populate dungeons and tokens tables
6. **Build MVP** - Start with API, then frontend
7. **Deploy** - Cloudflare Pages + Workers
8. **Test with [L%C]** - Real-world usage and feedback

---

**This document is the source of truth for the project. Update it as the project evolves.**


