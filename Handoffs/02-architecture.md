# Architecture: Beer Mile Betting App

## Technology Stack
| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | TypeScript | Type safety for full-stack consistency |
| Framework | Next.js 14+ (App Router) | Full-stack framework with API routes, React frontend, built-in optimization |
| Frontend | React + TailwindCSS | Component-driven UI, rapid styling |
| Database | Neon PostgreSQL (serverless) | Relational model fits betting schema; serverless scales to zero for MVP; JSONB for flexible prop data |
| Session Management | Iron (or simple signed cookies) | Lightweight, no external session store needed for MVP |
| Real-time Updates | Server-Sent Events (SSE) or polling | SSE for live bet feed; polling for leaderboard updates (simpler than WebSocket for MVP) |
| Testing | Vitest + Jest | Fast unit tests, integration test support |

## Component Structure

### Backend (Next.js API Routes)
- `pages/api/events/[eventId]` - Event CRUD and status management
- `pages/api/events/[eventId]/props` - Prop listing
- `pages/api/events/[eventId]/bets` - Bet placement and listing
- `pages/api/events/[eventId]/bets/stream` - SSE endpoint for live bet feed
- `pages/api/events/[eventId]/results` - Admin result submission and resolution
- `pages/api/events/[eventId]/leaderboard` - Leaderboard calculation and retrieval
- `pages/api/auth/session` - Session creation and validation
- Middleware: `lib/middleware/auth.ts` - Role-based access control
- Middleware: `lib/middleware/validation.ts` - Input validation

### Frontend (React Components)
- `pages/` - Entry point and event join flow
- `components/BetCard.tsx` - Render individual prop with bet UI
- `components/BetFeed.tsx` - Live scrolling feed of bets placed (SSE consumer)
- `components/AdminDashboard.tsx` - Result submission UI (admin-only)
- `components/Leaderboard.tsx` - Ranking display with scores
- `lib/client/api.ts` - Client API helpers
- `lib/client/useSSE.ts` - Custom hook for SSE subscription

### Database Layer
- `lib/db.ts` - Neon PostgreSQL connection
- `lib/queries/` - Prepared queries for events, bets, props, users, resolution

### Business Logic
- `lib/scoring.ts` - Scoring and point calculation
- `lib/resolution.ts` - Auto-resolution logic for all bet types
- `lib/validation.ts` - Input validation rules

## API Design

### Authentication & Sessions

#### POST /api/auth/session
Join event as user (friend) or admin.

**Request Body:**
```json
{
  "eventId": "uuid",
  "username": "string (2-20 chars, alphanumeric + underscores)",
  "adminSecret": "string (optional, for admin role)"
}
```

**Response 201:**
```json
{
  "sessionId": "string (secure cookie set)",
  "userId": "uuid",
  "username": "string",
  "role": "admin|annie|friend",
  "eventId": "uuid"
}
```

**Errors:**
- 400: Invalid username format or missing required fields
- 404: Event not found
- 401: Invalid admin secret
- 409: Username already taken for this event

---

### Event Management

#### GET /api/events/:eventId
Fetch event details (public).

**Response 200:**
```json
{
  "id": "uuid",
  "name": "string",
  "status": "open|closed|resolved",
  "created_at": "ISO timestamp",
  "updated_at": "ISO timestamp",
  "final_time": "MM:SS or null",
  "final_answers": {
    "yes_no_prop_id": "yes|no",
    ...
  }
}
```

**Errors:**
- 404: Event not found

---

#### POST /api/events
Create a new event (admin only via hardcoded initialization).

**Request Body:**
```json
{
  "name": "string",
  "adminSecret": "string (to be shared with admin)"
}
```

**Response 201:**
```json
{
  "id": "uuid",
  "name": "string",
  "adminSecret": "string",
  "shareLink": "https://app.url/join/[eventId]",
  "status": "open",
  "created_at": "ISO timestamp"
}
```

---

### Prop Management

#### GET /api/events/:eventId/props
Fetch all props for event (public, includes weight and type info).

**Response 200:**
```json
{
  "props": [
    {
      "id": "uuid",
      "type": "time_range|exact_time|yes_no",
      "question": "string",
      "weight": 1.0,
      "data": {
        "range_start_seconds": 780,
        "range_end_seconds": 840
      }
    },
    {
      "id": "uuid",
      "type": "exact_time",
      "question": "What exact time?",
      "weight": 1.5
    },
    {
      "id": "uuid",
      "type": "yes_no",
      "question": "Will Annie throw up?",
      "weight": 1.0
    }
  ]
}
```

---

### Betting

#### POST /api/events/:eventId/bets
Place a bet (requires session).

**Request Body:**
```json
{
  "propId": "uuid",
  "bet_value": "string or number"
}
```

**Bet value formats by type:**
- `time_range`: `"yes"` or `"no"`
- `exact_time`: `"13:47"` (MM:SS format)
- `yes_no`: `"yes"` or `"no"`

**Response 201:**
```json
{
  "id": "uuid",
  "userId": "uuid",
  "propId": "uuid",
  "bet_value": "string",
  "resolved": false,
  "result": null,
  "points_earned": null,
  "created_at": "ISO timestamp",
  "username": "string"
}
```

**Errors:**
- 400: Invalid bet value for prop type; invalid MM:SS format for exact time
- 403: User already placed exact time guess for this event
- 409: Event is closed/resolved (bet window closed)

---

#### GET /api/events/:eventId/bets
Fetch all bets for event (public, includes usernames and bet values for live feed).

**Query Params:**
- `limit`: number (default 50)
- `offset`: number (default 0)
- `sinceId`: uuid (for polling updates)

**Response 200:**
```json
{
  "bets": [
    {
      "id": "uuid",
      "username": "string",
      "propId": "uuid",
      "bet_value": "string",
      "created_at": "ISO timestamp",
      "resolved": false
    }
  ],
  "total": 120
}
```

---

#### GET /api/events/:eventId/bets/stream
Server-Sent Events endpoint for live bet feed (public).

**Connection:** Opens SSE stream
**Message format:**
```json
{
  "type": "new_bet",
  "data": {
    "id": "uuid",
    "username": "string",
    "propId": "uuid",
    "bet_value": "string",
    "created_at": "ISO timestamp"
  }
}
```

or

```json
{
  "type": "resolved",
  "data": {
    "betId": "uuid",
    "points_earned": 10,
    "result": "correct"
  }
}
```

**Errors:**
- SSE stream auto-closes on disconnect (client reconnects)

---

### Results & Resolution

#### POST /api/events/:eventId/results
Submit final results and auto-resolve all bets (admin only).

**Request Body:**
```json
{
  "final_time": "13:47",
  "final_answers": {
    "yes_no_prop_id_1": "yes",
    "yes_no_prop_id_2": "no"
  }
}
```

**Response 200:**
```json
{
  "eventId": "uuid",
  "status": "resolved",
  "final_time": "13:47",
  "final_answers": {...},
  "resolution_summary": {
    "total_bets_resolved": 120,
    "exact_time_rankings": [
      {
        "bet_id": "uuid",
        "username": "string",
        "guess": "13:45",
        "difference_seconds": 2,
        "rank": 1,
        "points_earned": 15
      }
    ]
  }
}
```

**Errors:**
- 403: User is not admin
- 400: Invalid time format or missing required answers
- 409: Event already resolved

---

### Leaderboard

#### GET /api/events/:eventId/leaderboard
Fetch ranked leaderboard (public).

**Response 200:**
```json
{
  "leaderboard": [
    {
      "rank": 1,
      "userId": "uuid",
      "username": "string",
      "total_points": 45,
      "bets_correct": 8,
      "bets_total": 12
    },
    {
      "rank": 2,
      "userId": "uuid",
      "username": "string",
      "total_points": 42,
      "bets_correct": 7,
      "bets_total": 12
    }
  ],
  "eventId": "uuid",
  "status": "open|closed|resolved"
}
```

---

### Error Responses

| Status | Meaning | When |
|--------|---------|------|
| 400 | Bad Request | Invalid input (bad JSON, invalid time format, missing fields) |
| 401 | Unauthorized | Invalid or missing admin secret; session expired |
| 403 | Forbidden | User lacks permission (e.g., friend trying to resolve bets) |
| 404 | Not Found | Event, prop, or user not found |
| 409 | Conflict | Event already resolved; duplicate exact time guess; username taken |
| 500 | Server Error | Unexpected database or calculation error |

---

## Data Model (PostgreSQL)

### Events Table
```sql
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  admin_secret VARCHAR(255) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'closed', 'resolved')),
  final_time VARCHAR(10),
  final_answers JSONB,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_status ON events(status);
```

---

### Users Table
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  username VARCHAR(50) NOT NULL,
  role VARCHAR(20) NOT NULL CHECK (role IN ('admin', 'annie', 'friend')),
  total_points INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE(event_id, username)
);

CREATE INDEX idx_users_event ON users(event_id);
```

---

### Props Table
```sql
CREATE TABLE props (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  type VARCHAR(20) NOT NULL CHECK (type IN ('time_range', 'exact_time', 'yes_no')),
  question VARCHAR(255) NOT NULL,
  weight DECIMAL(3, 2) NOT NULL DEFAULT 1.0,
  data JSONB NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_props_event ON props(event_id);
CREATE INDEX idx_props_type ON props(type);
```

**JSONB data examples:**
- `time_range`: `{"range_start_seconds": 780, "range_end_seconds": 840}`
- `exact_time`: `{}` (empty, just the bet value matters)
- `yes_no`: `{}` (empty)

---

### Bets Table
```sql
CREATE TABLE bets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  prop_id UUID NOT NULL REFERENCES props(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  bet_value VARCHAR(255) NOT NULL,
  resolved BOOLEAN NOT NULL DEFAULT false,
  result VARCHAR(20) CHECK (result IN ('correct', 'incorrect', NULL)),
  points_earned INTEGER,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bets_event ON bets(event_id);
CREATE INDEX idx_bets_prop ON bets(prop_id);
CREATE INDEX idx_bets_user ON bets(user_id);
CREATE INDEX idx_bets_resolved ON bets(resolved);
CREATE UNIQUE INDEX idx_bets_exact_time_per_user ON bets(event_id, user_id)
  WHERE (SELECT type FROM props WHERE props.id = prop_id) = 'exact_time';
```

---

## Session & Authentication Flow

### Session Management Strategy

1. **Session Creation** (POST /api/auth/session)
   - User provides username and optional adminSecret
   - Backend creates user record (or returns existing)
   - Sets secure, HTTP-only cookie with encrypted session data
   - Cookie contains: `{userId, eventId, role, username}`

2. **Session Validation (Middleware)**
   - All API endpoints verify session cookie
   - Extract userId, role from cookie
   - Attach user context to request
   - Return 401 if invalid/expired

3. **Role-Based Authorization**
   - `POST /api/events/:eventId/results` checks `role === 'admin'`
   - Event creation (hardcoded initialization, not exposed via API)
   - All other endpoints allow `admin`, `annie`, or `friend`

4. **Admin Secret**
   - Shared with admin user only (e.g., `ADMIN_SECRET=xyz` env var)
   - Verified at session creation
   - NOT stored in database (compared against env var)
   - Single secret per event; anyone with secret becomes admin

---

## Real-Time Updates Strategy

### Live Bet Feed (SSE)
- Frontend connects to `GET /api/events/:eventId/bets/stream`
- Server sends `new_bet` event whenever a bet is placed
- Client renders new bet in live feed component
- Automatic reconnect on disconnect with exponential backoff

### Leaderboard Updates (Polling)
- Frontend polls `GET /api/events/:eventId/leaderboard` every 3-5 seconds (configurable)
- Low overhead for MVP scale (20-50 users)
- Alternative: SSE for leaderboard changes, but polling simpler for MVP

### Auto-Resolution Notifications (SSE)
- When admin resolves bets, server broadcasts `resolved` event to all connected clients
- Includes points earned, result for each bet
- Client updates UI to show resolved status and leaderboard immediately

---

## Scoring Algorithm

### Core Formula
```
points = bet_weight × outcome_multiplier
```

### Outcome Multiplier by Bet Type

#### 1. Time Range Bets
```
if final_time falls within range:
  outcome_multiplier = 1.0
else:
  outcome_multiplier = 0
```

**Example:**
- Bet: "12:47-13:02?" with weight 1.0
- User bet: "yes"
- Final time: 13:00
- Result: Correct (13:00 is within range)
- Points: 1.0 × 1.0 = 1 point

---

#### 2. Yes/No Proposition Bets
```
if user_answer matches final_answer:
  outcome_multiplier = 1.0
else:
  outcome_multiplier = 0
```

**Example:**
- Bet: "Will Annie throw up?" with weight 1.0
- User bet: "yes"
- Final answer: "no"
- Result: Incorrect
- Points: 0

---

#### 3. Exact Time Guess Bets (Closest Guess Ranking)
```
1. Calculate difference_seconds for all exact time guesses
2. Rank bets by difference_seconds (ascending; closest first)
3. Assign points based on rank:
   - Rank 1 (closest):     outcome_multiplier = 1.0   → points = 1.5 × 1.0 = 1.5 (if weight=1.5)
   - Rank 2 (2nd closest): outcome_multiplier = 0.8   → points = 1.5 × 0.8 = 1.2
   - Rank 3 (3rd closest): outcome_multiplier = 0.6   → points = 1.5 × 0.6 = 0.9
   - Rank 4+:              outcome_multiplier = 0.4   → points = 1.5 × 0.4 = 0.6
   - If exact match (0 seconds diff): outcome_multiplier = 1.5 → points = 1.5 × 1.5 = 2.25 (bonus)
4. Ties broken by creation_at (earliest bet wins in case of tie)
```

**Example:**
- Bet type: "Exact time" with weight 1.5
- Final time: 13:47
- Guesses:
  - User A: 13:47 (diff: 0s) → Rank 1, exact match → 1.5 × 1.5 = 2.25 points
  - User B: 13:45 (diff: 2s) → Rank 2 → 1.5 × 0.8 = 1.2 points
  - User C: 13:50 (diff: 3s) → Rank 3 → 1.5 × 0.6 = 0.9 points
  - User D: 14:00 (diff: 13s) → Rank 4+ → 1.5 × 0.4 = 0.6 points

---

### Leaderboard Ranking
1. Sort users by total_points (descending)
2. Break ties by earliest bet creation (fairness for tied users)
3. Display rank, username, total_points, bets_correct, bets_total

---

## Role-Based Access Control (RBAC)

### Matrix

| Action | Admin | Annie | Friend |
|--------|-------|-------|--------|
| Place bets | Yes | Yes | Yes |
| View own bets | Yes | Yes | Yes |
| View all bets (live feed) | Yes | Yes | Yes |
| Submit final results | **Only** | - | - |
| View leaderboard | Yes | Yes | Yes |
| Close betting window | Hardcoded | - | - |
| Create event | Hardcoded | - | - |

### Enforcement
- Middleware checks `role` from session cookie on protected endpoints
- `POST /api/events/:eventId/results` verifies `role === 'admin'`
- Other endpoints check session exists (any role allowed)

---

## Input Validation Rules

### Username
- Required, 2-20 characters
- Alphanumeric + underscores only
- Regex: `^[a-zA-Z0-9_]{2,20}$`
- Must be unique per event

### Time Formats
- MM:SS format only (e.g., "13:47")
- Regex: `^([0-9]{1,2}):([0-9]{2})$`
- Minutes: 0-99
- Seconds: 0-59
- No hours in MVP

### Bet Values
- `time_range`: "yes" or "no" (case-insensitive, trimmed)
- `exact_time`: Valid MM:SS format
- `yes_no`: "yes" or "no" (case-insensitive, trimmed)

### Admin Secret
- At least 8 characters
- Required only for admin role
- Compared against `process.env.ADMIN_SECRET`

### Final Answers (Resolution)
- JSONB object with `prop_id: answer` pairs
- Must include all `yes_no` props
- Answer format: "yes" or "no"

---

## Error Handling Strategy

### Input Validation
1. **Schema Validation**: Use `zod` or `joi` to validate request bodies
2. **Early Return**: Return 400 with descriptive error message
3. **Client Feedback**: Show user-friendly error messages in UI

### Database Errors
1. **Constraint Violations** (e.g., duplicate username): Catch and return 409 Conflict
2. **Foreign Key Errors**: Return 404 Not Found
3. **Transaction Failures**: Log to console; return 500 Server Error; ideally retry

### Authorization Errors
1. **Invalid Session**: Return 401 Unauthorized
2. **Insufficient Role**: Return 403 Forbidden
3. **Event State Violations** (e.g., betting closed): Return 409 Conflict

### Resolution Errors
1. **Invalid Time Format**: Return 400 with message
2. **Missing Answers**: Return 400 with missing prop list
3. **Already Resolved**: Return 409 Conflict

### Logging
- Use `console.log` for MVP (upgrade to Winston/Pino later)
- Log: timestamp, endpoint, error type, user context (userId, role)
- Do NOT log sensitive data (passwords, admin secrets)

---

## Architecture Decisions

### Decision 1: Server-Sent Events for Live Bet Feed
- **Context**: Real-time bet visibility was a new requirement for social/competitive feel
- **Options Considered**:
  1. WebSocket (full-duplex, overkill for MVP)
  2. Polling (simple, higher overhead)
  3. Server-Sent Events (simple, unidirectional, server-to-client)
- **Decision**: SSE
- **Trade-offs**:
  - Pro: Native browser support, works over HTTP/2, simpler than WebSocket
  - Con: Unidirectional only (but sufficient for this use case)

### Decision 2: Closest-Guess Ranking for Exact Time
- **Context**: Need to reward proximity, not just exact matches
- **Options Considered**:
  1. All-or-nothing (simple, unfair)
  2. Percentile scoring (complex to explain)
  3. Rank-based multipliers (intuitive, incentivizes trying)
- **Decision**: Rank-based multipliers (1.0, 0.8, 0.6, 0.4...) with exact-match bonus
- **Trade-offs**:
  - Pro: Encourages participation even if exact time unknown
  - Con: Slightly more complex scoring logic

### Decision 3: Polling for Leaderboard Updates (Not SSE)
- **Context**: Could use SSE for both bet feed and leaderboard
- **Options Considered**:
  1. SSE for everything (consistent, more connections)
  2. Polling for everything (simpler, more requests)
  3. SSE for bets + polling for leaderboard (hybrid)
- **Decision**: Hybrid (SSE for bets, polling for leaderboard)
- **Trade-offs**:
  - Pro: Bets are high-frequency, leaderboard is low-frequency; polling leaderboard saves connections
  - Con: Slightly inconsistent architecture; polling adds latency (3-5s)

### Decision 4: Neon PostgreSQL (Serverless) Over SQLite
- **Context**: Database choice for MVP
- **Options Considered**:
  1. SQLite (zero ops, local file)
  2. PostgreSQL (serverless, scales better)
- **Decision**: PostgreSQL (Neon)
- **Trade-offs**:
  - Pro: Scales beyond MVP; supports JSONB for flexible prop data; better for production later
  - Con: Adds slight latency; requires Neon account

### Decision 5: Session Cookies Over JWT
- **Context**: Auth strategy for stateless Next.js
- **Options Considered**:
  1. JWT tokens (standard, no server state)
  2. Secure cookies with iron (encrypted, simple)
  3. Database session store (complex, not needed for MVP)
- **Decision**: Secure cookies with iron
- **Trade-offs**:
  - Pro: Simple, secure by default, works out-of-box with Next.js
  - Con: Requires secure transport (HTTPS); ties session to origin

### Decision 6: Hardcoded Props at Backend (No Frontend Creation)
- **Context**: MVP scope constraint
- **Options Considered**:
  1. Admin UI for prop creation (flexible, out of scope)
  2. Hardcoded props (simple, meets MVP requirement)
  3. Seed script to load props from config file
- **Decision**: Hardcoded props in database seed
- **Trade-offs**:
  - Pro: Minimal backend work; props finalized before event
  - Con: Code change needed to modify props; Phase 2 will need admin UI

---

## Open Questions for Implementer

1. **Exact Time Tolerance**: Should the scoring penalize bets that are, e.g., 1 second off more than 30 seconds off? Currently uses rank-based multipliers. Alternative: Decay function based on seconds difference?

2. **Bet Window Management**: How should admin "close" betting? Via API endpoint, or implicit (admin resolves bets and betting is auto-closed)?

3. **Session Expiry**: What's the ideal session timeout? Suggestion: 24 hours (event is one-time, short-lived).

4. **Exact Time Tie Resolution**: If two users guess identical times, should they both be rank 1, or should creation_at decide? Currently: creation_at decides (FIFO fairness).

5. **Leaderboard Cache**: Should leaderboard be cached (computed once after resolution), or recalculated on each request? For MVP: compute on request (simpler, <50 users = fast).

6. **Event Visibility**: Can friends see other events, or only the one they joined? Currently: implicit (only the event in the URL). Phase 2: multiple events.

7. **Mobile Responsiveness**: Assumption is mobile-friendly React UI. TailwindCSS responsive classes will handle this automatically.

---

## Deployment Notes

### Environment Variables (Neon)
```
DATABASE_URL=postgresql://user:password@ep-xyz.neon.tech/beer-mile-db
ADMIN_SECRET=your-secret-key-here
NEXTAUTH_SECRET=generated-by-openssl-rand-hex-32
NODE_ENV=production
```

### Neon Setup
1. Create Neon project
2. Run migrations (provided as SQL files in `/migrations/`)
3. Copy `DATABASE_URL` to `.env.local`

### Next.js Deployment
1. Build: `npm run build`
2. Deploy to Vercel (native Next.js support) or self-hosted Node server
3. Ensure environment variables are set in deployment platform

---

## Testing Strategy (Out of Scope for Phase 1, but noted)

- **Unit Tests**: Scoring algorithm, validation rules
- **Integration Tests**: API routes + database
- **E2E Tests**: Selenium/Playwright for user flows (join event, place bet, resolve, view leaderboard)
