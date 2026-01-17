# Implementation Plan: Beer Mile Betting App MVP

**Status**: Plan written, awaiting test case review

## Overview
Building a Next.js full-stack betting app for friends to wager on Annie's beer mile performance. Stack: Next.js 14 + TypeScript + Neon PostgreSQL + TailwindCSS.

---

## Phase 1: Project Setup & Foundation
1. Initialize Next.js 14 project with TypeScript
2. Install dependencies (React, TailwindCSS, Neon PostgreSQL driver, validation library, session management)
3. Configure database connection to Neon PostgreSQL
4. Create database schema (events, users, props, bets tables with indexes)
5. Set up environment variables (.env.local)

## Phase 2: Core Backend Infrastructure
1. Create database queries layer (`lib/db.ts`, `lib/queries/`)
2. Implement authentication middleware (session creation, validation, role-based access)
3. Set up input validation utilities (username, time formats, bet values)
4. Create error handling patterns (consistent error responses)
5. Seed initial event and props into database

## Phase 3: API Endpoints - Auth & Events
1. **POST /api/auth/session** - Join event with username/admin secret
2. **GET /api/events/:eventId** - Fetch event details
3. **POST /api/events** - Create event (hardcoded for admin setup)
4. **GET /api/events/:eventId/props** - List all props for event

## Phase 4: API Endpoints - Betting & Leaderboard
1. **POST /api/events/:eventId/bets** - Place bet
2. **GET /api/events/:eventId/bets** - Fetch bets (with pagination)
3. **GET /api/events/:eventId/leaderboard** - Ranked leaderboard
4. **GET /api/events/:eventId/bets/stream** - SSE endpoint for live feed

## Phase 5: API Endpoints - Resolution
1. Implement scoring engine (`lib/scoring.ts`) with all three bet types
2. Implement resolution logic (`lib/resolution.ts`)
3. **POST /api/events/:eventId/results** - Admin submits results and triggers auto-resolution

## Phase 6: Frontend UI Components
1. Create index page - Event join form
2. Create BetCard component - Display prop and place bet
3. Create BetFeed component - Live bet feed with SSE
4. Create AdminDashboard component - Result submission form
5. Create Leaderboard component - Ranked display
6. Create ResultsPage component - Post-resolution display

## Phase 7: Integration & Testing
1. Test API endpoints with curl/Postman
2. Test SSE live feed functionality
3. Test scoring algorithm with sample data
4. Manual end-to-end testing (join → place bet → resolve → check results)

---

## Technical Decisions
- **Session Auth**: Secure HTTP-only cookies with Iron encryption
- **Real-time**: SSE for live bet feed (simpler than WebSocket), polling for leaderboard
- **Validation**: Zod for schema validation (type-safe)
- **Scoring**: Rank-based multipliers for exact time bets (1.0, 0.8, 0.6, 0.4 with exact-match bonus)
- **Props**: Hardcoded seed script (no admin UI in MVP)

---

## Next Steps
1. Create comprehensive test cases for all betting mechanics
2. Test scoring algorithm with various time submissions
3. Test role-based access control
4. Test SSE live feed
5. Code implementation begins after test approval
