# Beer Mile Betting App - Requirements Document

## Project Overview
A web app for friends to place bets on Annie's beer mile performance. Friends place prop bets, Annie completes the challenge, an admin submits the final results, and bets auto-resolve immediately. This is a one-time MVP focused on solid betting mechanics—no live commentary, no achievements, no complex features.

## User Types & Access

### User Roles
1. **Admin** (the app owner)
   - Can vote on bets like everyone else
   - Can submit final answers/times through frontend UI
   - Resolves all bets by entering final results
   - Full access to admin dashboard

2. **Annie** (the participant)
   - Can see all bets placed by friends
   - Can place bets like everyone else
   - No special admin powers

3. **Friends** (regular participants)
   - Click a shareable link to join
   - Choose a username
   - Can place bets on all available props
   - Can see bet status/results after resolution

### Access Control
- Link-based access (no authentication required initially, but username-based tracking)
- Admin has protected endpoints to submit final results
- Role-based authorization for admin-only actions

## Prop Bet Types (MVP)

### 1. Time Range Brackets
- **Format**: Binary yes/no bet on whether Annie finishes within a specific time range
- **Example bets**:
  - "12:47-13:02?" (yes/no)
  - "Under 13:00?" (yes/no)
  - "13:30-14:15?" (yes/no)
- **Multiple brackets**: App supports multiple time range options (e.g., 4-6 different brackets)
- **Backend-defined**: Hardcoded in backend, no frontend bet creation in MVP
- **Weight**: Standard weight (baseline scoring)

### 2. Exact Time Guess
- **Format**: User enters precise time to the second (MM:SS format)
- **Scoring**: Worth more points than range bets (higher weight)
- **Resolution**: Closest guess wins, or exact match gets bonus points
- **Constraint**: Only one exact time guess per user per event

### 3. Yes/No Proposition
- **Format**: Simple yes/no bet on an event (e.g., "Will Annie throw up?")
- **Backend-defined**: Hardcoded, no frontend creation
- **Weight**: Standard weight
- **Resolution**: Binary outcome based on admin submission

### 4. Custom Props (Phase 2+)
- Not included in MVP
- Reserve for future expansion

## Betting Workflow

### Bet Creation
- **Who**: Admin/backend developer (hardcoded)
- **When**: Before the event, during app setup
- **Props defined**: Time ranges, exact time guess option, yes/no props
- **No frontend creation**: MVP ships with predefined prop structure

### Bet Placement
- **Access**: Friends visit shareable link, choose username, place bets
- **Bet window**: Open until admin closes it (manual trigger, no auto-close in MVP)
- **Multiple bets**: Users can place multiple bets across different props
- **Constraints**: One exact time guess per user
- **Visibility**: Users can see their own bets; other bets not visible until resolution
- **No stake/wagers**: Bragging rights only

### Bet Weights (Scoring)
- **Time range brackets**: Weight = 1.0 (baseline)
- **Exact time guess**: Weight = 1.5 or 2.0 (higher value for difficulty)
- **Yes/No props**: Weight = 1.0
- **Scoring calculation**: Points awarded based on correct prediction × bet weight
- **Final leaderboard**: Ranked by total points earned

## Annie's Final Time Submission

### Input Method
- Admin enters final answers through frontend UI
- Answers include:
  - **Final time** (MM:SS format) - e.g., "13:47"
  - **Yes/No prop outcomes** - e.g., "Did she throw up? Yes"
  - Any other resolution data needed

### Verification
- Input is trusted (no external timing mechanism in MVP)
- Admin is responsible for accuracy

### Editability
- Time/answers are **final on submission**—no corrections/edits in MVP
- If a mistake occurs, admin must clear all bets and restart (manual workaround)

## Bet Resolution

### Auto-Resolution Process
1. Admin submits final answers through UI
2. System immediately evaluates all placed bets against final answers
3. All bets marked as resolved
4. Results calculated and displayed to all users

### Resolution Details
- **Time range bets**: Check if final time falls within range
- **Exact time guess**: Compare to final time (exact match or closest)
- **Yes/No props**: Match to submitted outcome

### After Resolution
- Display results page with:
  - Final time and all answers
  - Leaderboard ranked by points
  - Individual bet outcomes for each user
  - Bragging rights

## Data Model (High-Level)

### Core Entities
1. **Event**
   - id, event_name, created_at, status (open/closed/resolved)
   - final_time, final_answers

2. **Prop** (Bet Type)
   - id, event_id, prop_type (time_range|exact_time|yes_no)
   - prop_data (range_start, range_end, question, etc.)
   - weight (1.0, 1.5, 2.0, etc.)

3. **Bet** (Individual User Bet)
   - id, event_id, prop_id, user_name, bet_value (time, yes/no choice)
   - resolved, result (correct/incorrect), points_earned
   - created_at

4. **User** (Participant)
   - id, event_id, username, role (admin|annie|friend)
   - total_points, created_at

## Phase 1 Scope

### What's Included
- Event setup (hardcoded props)
- Bet placement UI for friends
- Admin dashboard to submit final results
- Auto-resolution on result submission
- Leaderboard with scoring
- Role-based access control

### What's NOT Included (Phase 2+)
- Live commentary/real-time updates
- Achievements/badges
- Custom prop creation UI
- User accounts/persistent login
- Bet editing/cancellation
- Analytics/stats
- Multiple events history
- Mobile app (web-only)

## Non-Functional Requirements

### Performance
- Page load under 2 seconds
- Bet placement under 1 second
- Resolution calculation under 500ms
- Support 20-50 concurrent users

### Scale
- MVP: Single event, <50 participants
- No database clustering needed
- Simple deployment (single server)

### Security
- Admin endpoint for result submission must be protected (simple auth or role check)
- Input validation on all bet submissions
- SQL injection prevention
- XSS protection

### Reliability
- No backup/failover needed for MVP
- Simple error handling and user feedback
- Graceful handling of duplicate submissions

## Edge Cases & Risks

1. **Exact time guess collisions**: Multiple users guess same time
   - Resolution: All correct guesses split points equally, or highest weight among tied bets

2. **Time range overlaps**: User bets on overlapping ranges
   - Resolution: User can bet on multiple ranges; each evaluated independently

3. **Admin submits wrong time**: Final time entered incorrectly
   - Workaround in MVP: Manual database correction or event reset

4. **Bet placement after event starts**: Timing window issues
   - Solution: Admin manually closes betting window before revealing results

5. **User with multiple accounts**: Same person places bets under different usernames
   - Risk: No prevention in MVP; accept this for MVP phase
   - Phase 2: Implement device/IP tracking or account linking

6. **Division by zero in scoring**: If no users get points
   - Resolution: Handle gracefully, show 0 points for all

7. **Timezone issues**: If times are submitted with timezone confusion
   - Assumption: All times are local/same timezone; specify in UI

## Open Questions

- Should exact time guess allow +/- tolerance (e.g., within 5 seconds)?
- Should there be a leaderboard reset between events, or cumulative scoring?
- What's the final weight/scoring formula? (e.g., points = weight × accuracy_multiplier)?
- Should users be able to see other users' bets on props before resolution?
- Do we need audit logs for admin actions?

## Constraints

- **No authentication framework**: MVP uses simple username/role in session
- **Single event**: Not designed for multi-event support yet
- **Hardcoded props**: All betting options defined at deploy time
- **Manual betting window**: No auto-close, admin must trigger
- **Local timezone**: Assume all times are in same timezone

## Technology Stack (To Be Determined by Architect)

- Frontend: [TBD]
- Backend: [TBD]
- Database: [TBD]
- Deployment: [TBD]
