# Beer Mile Betting App

A lightweight web app for friends to place prop bets on Annie's beer mile performance. When Annie finishes, an admin submits results and the app auto-calculates a leaderboard ranked by points earned.

## Current Phase

**Architecture & Planning** - Architecture designed, test strategy documented. Ready for implementation phase to begin.

## Tech Stack

- **Language**: TypeScript
- **Framework**: Next.js 14+ (App Router) with React
- **Frontend**: React + TailwindCSS
- **Database**: Neon PostgreSQL (serverless)
- **Session Management**: Iron (signed HTTP-only cookies)
- **Real-time**: Server-Sent Events (SSE) for live bet feed, polling for leaderboard
- **Testing**: Vitest + Jest (planned, not yet implemented)
- **Validation**: Zod (schema validation)

## Core Entities

- **Event**: Beer mile instance (status: open/closed/resolved, final_time, final_answers)
- **User**: Participant (username, role: admin/annie/friend, total_points)
- **Prop**: Betting option with type (time_range, exact_time, yes_no), weight, and metadata
- **Bet**: Individual user prediction (bet_value, resolved, result, points_earned)

## Betting Mechanics

### Prop Types
1. **Time Range** - Binary yes/no on whether Annie finishes within MM:SS bracket (e.g., "12:47-13:02?")
2. **Exact Time** - User enters precise MM:SS guess; closest guesses win highest points (weight 1.5)
3. **Yes/No Proposition** - Binary outcome (e.g., "Will Annie throw up?")

### Scoring Formula
```
points = bet_weight × outcome_multiplier
```

**Outcome Multipliers:**
- **Time Range**: 1.0 (correct) or 0 (incorrect)
- **Yes/No**: 1.0 (correct) or 0 (incorrect)
- **Exact Time**: Rank-based (Rank 1: 1.0, Rank 2: 0.8, Rank 3: 0.6, Rank 4+: 0.4; exact match bonus: 1.5)

Leaderboard ranked by total_points (descending); ties broken by earliest bet creation.

## API Endpoints

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| POST | /api/auth/session | Join event as user/admin | None (creates session) |
| GET | /api/events/:eventId | Fetch event details | None |
| POST | /api/events | Create event | Hardcoded (admin setup only) |
| GET | /api/events/:eventId/props | List all props | None |
| POST | /api/events/:eventId/bets | Place bet | Session required |
| GET | /api/events/:eventId/bets | Fetch bets (pagination) | None |
| GET | /api/events/:eventId/bets/stream | SSE live bet feed | None |
| POST | /api/events/:eventId/results | Admin resolves bets | Admin only |
| GET | /api/events/:eventId/leaderboard | Ranked results | None |

## Key Decisions

- **Session Cookies over JWT**: Secure, simple, works out-of-box; requires HTTPS
- **SSE for Live Feed**: Unidirectional, simple, native browser support (vs WebSocket overkill)
- **Polling for Leaderboard**: Low-frequency updates; polling simpler than SSE for MVP
- **Hardcoded Props**: Props defined at deploy time (no admin creation UI in Phase 1)
- **PostgreSQL over SQLite**: Scales beyond MVP; JSONB for flexible prop data; serverless scales to zero
- **Rank-Based Scoring for Exact Time**: Encourages participation even if exact time unknown; intuitive
- **Single Event MVP**: No multi-event history; Phase 2 will expand

## Constraints

- **No persistent auth**: Username-based tracking; no accounts/login required
- **Link-based access**: Share event URL; anyone can join with unique username
- **Manual bet window**: Admin must manually close betting (no auto-close)
- **No bet editing**: Bets final on submission
- **Local timezone**: All times assumed same timezone
- **One exact time guess per user**: Enforced at submission
- **Admin secret**: Shared via env var; compares at session creation

## Database Schema

**Events**: id, name, admin_secret, status, final_time, final_answers (JSONB), created_at, updated_at

**Users**: id, event_id, username, role, total_points, created_at, updated_at (unique: event_id + username)

**Props**: id, event_id, type, question, weight, data (JSONB), created_at (indexes: event_id, type)

**Bets**: id, event_id, prop_id, user_id, bet_value, resolved, result, points_earned, created_at, updated_at (indexes: event_id, prop_id, user_id, resolved; unique on exact_time per user per event)

## How to Run (Planned Implementation)

```bash
# Clone and setup
git clone <repo>
cd fun-agent-project
npm install

# Configure environment (.env.local)
DATABASE_URL=postgresql://[credentials]
ADMIN_SECRET=your-secret-here
NEXTAUTH_SECRET=$(openssl rand -hex 32)
NODE_ENV=development

# Setup database
npm run db:migrate  # Run SQL migrations to create tables

# Development server
npm run dev         # Starts http://localhost:3000

# Testing (when implemented)
npm test            # Run Vitest suite
npm run test:e2e    # End-to-end tests

# Production build
npm run build
npm start
```

## Current Status

- [x] Requirements documented (01-requirements.md)
- [x] Architecture designed (02-architecture.md)
- [x] Implementation plan created (03-implementer.md)
- [x] Comprehensive test strategy defined (05-test.md - 120 test cases)
- [ ] Next.js project scaffolding
- [ ] Database setup (Neon PostgreSQL)
- [ ] API routes implementation (auth, events, props, bets, results, leaderboard)
- [ ] Frontend components (join form, bet cards, admin dashboard, leaderboard, SSE feed)
- [ ] Test suite execution
- [ ] End-to-end testing
- [ ] Deployment to Vercel

## Open Questions / Notes

1. **Exact Time Tolerance**: Current design uses rank-based multipliers. Could explore decay function based on seconds difference.
2. **Betting Window**: Admin manually closes betting; consider API endpoint to auto-close vs implicit closure on resolution.
3. **Session Timeout**: Suggested 24 hours (one-time event, short-lived).
4. **Leaderboard Cache**: Currently computed on each request (simpler for MVP, <50 users = fast).
5. **Event Visibility**: MVP shows only one event; Phase 2 will support multiple events per user.
6. **Mobile Responsiveness**: TailwindCSS responsive classes will handle automatically.

## Key Files

- **Requirements**: `/Handoffs/01-requirements.md` - User roles, prop types, betting workflow, MVP scope
- **Architecture**: `/Handoffs/02-architecture.md` - Tech stack, API design, database schema, scoring algorithm
- **Implementation Plan**: `/Handoffs/03-implementer.md` - 7 phases of development
- **Test Strategy**: `/Handoffs/05-test.md` - 120 comprehensive test cases (unit, integration, validation, edge, error)

## Deployment Notes

- **Vercel**: Native Next.js support; environment variables configured in platform
- **Database**: Create Neon project, run migrations, set DATABASE_URL in env
- **Admin Secret**: Must be set as environment variable before first admin user joins
- **Health Check**: Monitor `/api/health` (if implemented) or check SSE endpoint
