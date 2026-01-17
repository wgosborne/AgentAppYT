# Testing: Beer Mile Betting App

## Test Strategy

This document outlines comprehensive testing for the Beer Mile Betting App MVP. The strategy covers:
- Unit tests for core business logic (scoring, validation)
- Integration tests for API endpoints and database interactions
- Validation tests for input sanitization and constraint enforcement
- Edge case tests for boundary conditions and race conditions
- Error handling tests for graceful failure modes

The testing approach is adversarial, expecting the code to handle invalid inputs, race conditions, and edge cases properly. All tests will be run against a test database with isolated data sets.

---

## Test Coverage

### Unit Tests

#### 1. Username Validation
- **Test Description**: Verify username format validation follows regex `^[a-zA-Z0-9_]{2,20}$`
- **Pre-conditions**: Validation utility loaded in memory
- **Test Steps**:
  1. Test valid usernames: "alice", "bob_123", "A1B2C3_XYZ"
  2. Test invalid usernames: "a" (too short), "a very long username that exceeds limit" (too long)
  3. Test invalid characters: "alice@domain", "bob-smith", "user.name", "user name"
  4. Test empty string and null
- **Expected Results**: Valid usernames pass; invalid usernames rejected with descriptive error
- **Priority**: High

#### 2. Time Format Validation (MM:SS)
- **Test Description**: Verify time format validation for MM:SS regex `^([0-9]{1,2}):([0-9]{2})$`
- **Pre-conditions**: Time validation utility loaded
- **Test Steps**:
  1. Test valid times: "13:47", "1:30", "00:00", "59:59", "99:59"
  2. Test invalid times: "13:60" (seconds out of range), "13:7" (single digit seconds), "13" (missing seconds)
  3. Test invalid formats: "13:47:00" (includes hours), "1347" (no colon), "13.47" (decimal)
  4. Test edge cases: "00:00", "99:59" (maximum MM)
  5. Test empty and null values
- **Expected Results**: Valid times pass; invalid times rejected with specific error message
- **Priority**: High

#### 3. Bet Value Validation by Type
- **Test Description**: Validate bet values match their prop type requirements
- **Pre-conditions**: Validation utility loaded with prop type definitions
- **Test Steps**:
  1. **time_range bets**: Test "yes", "no", "YES", "NO", "  yes  " (trimmed)
  2. **time_range invalid**: Test "maybe", "1", "true", null
  3. **exact_time bets**: Test "13:47", "1:30", "00:00"
  4. **exact_time invalid**: Test "13:60", "25:00", "abc:de"
  5. **yes_no bets**: Test "yes", "no", "YES", "NO"
  6. **yes_no invalid**: Test "maybe", "1", null
- **Expected Results**: Valid values pass; invalid values rejected with error indicating expected format
- **Priority**: High

#### 4. Admin Secret Validation
- **Test Description**: Verify admin secret is at least 8 characters
- **Pre-conditions**: Admin secret validation utility
- **Test Steps**:
  1. Test valid secrets: "12345678", "verylongsecretkey123"
  2. Test invalid secrets: "1234567" (too short), "", null
  3. Test secrets with special characters: "secret@123", "pass#word"
- **Expected Results**: Valid secrets pass; invalid secrets rejected
- **Priority**: Medium

#### 5. Scoring Algorithm - Time Range Bets
- **Test Description**: Verify scoring for time range bets (correct/incorrect)
- **Pre-conditions**: Scoring module loaded; sample time range props and bets defined
- **Test Steps**:
  1. Create time range prop: "12:47-13:02" (range_start_seconds=767, range_end_seconds=782)
  2. Create bets: user A bets "yes", user B bets "no"
  3. Set final_time to "13:00" (within range)
  4. Calculate points: user A should get 1.0 * 1.0 = 1 point; user B should get 0 points
  5. Set final_time to "13:30" (outside range)
  6. Calculate points: user A should get 0 points; user B should get 1.0 * 1.0 = 1 point
- **Expected Results**: Points calculated correctly based on range overlap
- **Priority**: High

#### 6. Scoring Algorithm - Yes/No Props
- **Test Description**: Verify scoring for yes/no proposition bets
- **Pre-conditions**: Scoring module loaded; sample yes/no props defined
- **Test Steps**:
  1. Create yes/no prop: "Will Annie throw up?" with weight 1.0
  2. Create bets: user A bets "yes", user B bets "no", user C bets "yes"
  3. Set final answer to "yes"
  4. Calculate points: users A and C get 1.0 points; user B gets 0 points
  5. Set final answer to "no"
  6. Calculate points: user B gets 1.0 points; users A and C get 0 points
- **Expected Results**: Binary outcome scoring works correctly
- **Priority**: High

#### 7. Scoring Algorithm - Exact Time Guess (Rank-Based)
- **Test Description**: Verify closest-guess ranking and multiplier assignment for exact time bets
- **Pre-conditions**: Scoring module loaded; exact_time prop with weight 1.5
- **Test Steps**:
  1. Create exact_time prop with weight 1.5
  2. Create guesses: A=13:47, B=13:45, C=13:50, D=14:00
  3. Set final_time to 13:47
  4. Calculate differences: A=0s, B=2s, C=3s, D=13s
  5. Rank and assign multipliers:
     - A: Rank 1, diff=0 (exact match) → multiplier=1.5 → points = 1.5 * 1.5 = 2.25
     - B: Rank 2, diff=2s → multiplier=0.8 → points = 1.5 * 0.8 = 1.2
     - C: Rank 3, diff=3s → multiplier=0.6 → points = 1.5 * 0.6 = 0.9
     - D: Rank 4+, diff=13s → multiplier=0.4 → points = 1.5 * 0.4 = 0.6
- **Expected Results**: Ranking calculated correctly; exact match receives bonus multiplier (1.5); other ranks receive correct multipliers
- **Priority**: High

#### 8. Scoring Algorithm - Exact Time Ties
- **Test Description**: Verify tie-breaking by creation_at (FIFO fairness)
- **Pre-conditions**: Scoring module; two users guess identical times
- **Test Steps**:
  1. Create two bets with same exact_time guess "13:47"
  2. Bet 1 created at 10:00:00
  3. Bet 2 created at 10:00:05
  4. Set final_time to 13:47
  5. Both are diff=0; check ranking
- **Expected Results**: Bet 1 ranks as Rank 1 (earliest); Bet 2 ranks as Rank 2 (later); tie broken by creation_at
- **Priority**: High

#### 9. Event Status Transitions
- **Test Description**: Verify valid event state transitions (open → closed → resolved)
- **Pre-conditions**: Event creation utility
- **Test Steps**:
  1. Create event with status="open"
  2. Verify status is "open"
  3. Admin closes betting (status="closed")
  4. Verify no new bets can be placed
  5. Admin resolves event (status="resolved")
  6. Verify bets are calculated and leaderboard is finalized
- **Expected Results**: State transitions are enforced; betting closed prevents new bets; resolution is idempotent
- **Priority**: High

#### 10. Leaderboard Ranking
- **Test Description**: Verify leaderboard sorts by total_points descending; ties broken by earliest bet
- **Pre-conditions**: Sample users with various point totals
- **Test Steps**:
  1. User A: 45 points, earliest bet at 10:00:00
  2. User B: 45 points, earliest bet at 10:00:05
  3. User C: 42 points
  4. Calculate leaderboard
  5. Verify rank 1 = A, rank 2 = B, rank 3 = C
- **Expected Results**: Leaderboard correctly sorted; tie-breaking works
- **Priority**: Medium

---

### Integration Tests

#### 1. User Registration via Session Creation
- **Test Description**: Verify user can join event with username and get session cookie
- **Pre-conditions**: Event created; database ready
- **Test Steps**:
  1. POST /api/auth/session with eventId, username="alice_test"
  2. Verify response status 201
  3. Verify sessionId returned
  4. Verify secure HTTP-only cookie set
  5. Verify user record created in database
  6. Verify role defaults to "friend"
- **Expected Results**: Session created; user record persisted; cookie secure
- **Priority**: High

#### 2. Admin Authentication via Admin Secret
- **Test Description**: Verify admin can join event with correct admin secret
- **Pre-conditions**: Event created with ADMIN_SECRET env var set
- **Test Steps**:
  1. POST /api/auth/session with eventId, username="admin_user", adminSecret="correct_secret"
  2. Verify response status 201
  3. Verify role="admin" in response
  4. Verify user role="admin" in database
- **Expected Results**: Admin secret verified; role set to admin
- **Priority**: High

#### 3. Annie Role Assignment
- **Test Description**: Verify user with username "annie" gets role="annie"
- **Pre-conditions**: Event created
- **Test Steps**:
  1. POST /api/auth/session with username="annie"
  2. Verify response includes role="annie"
  3. Verify database records role="annie"
- **Expected Results**: Annie identified and assigned correct role
- **Priority**: High

#### 4. Duplicate Username Rejection
- **Test Description**: Verify same username cannot be used twice in same event
- **Pre-conditions**: Event created; user "alice_test" already joined
- **Test Steps**:
  1. POST /api/auth/session with username="alice_test" (second time)
  2. Verify response status 409 Conflict
  3. Verify error message indicates duplicate username
  4. Verify only one user record exists in database
- **Expected Results**: Duplicate prevented; 409 error returned
- **Priority**: High

#### 5. Prop Listing
- **Test Description**: Verify GET /api/events/:eventId/props returns all hardcoded props with correct structure
- **Pre-conditions**: Event seeded with props; props table populated
- **Test Steps**:
  1. GET /api/events/:eventId/props
  2. Verify response status 200
  3. Verify props array includes all three types: time_range, exact_time, yes_no
  4. Verify each prop has: id, type, question, weight, data
  5. Verify time_range props have range_start_seconds and range_end_seconds in data
  6. Verify exact_time prop has weight >= 1.5
  7. Verify yes_no props have question text
- **Expected Results**: All props returned with correct structure and metadata
- **Priority**: High

#### 6. Place Bet - Time Range
- **Test Description**: Verify user can place time range bet (yes/no)
- **Pre-conditions**: User session exists; time_range prop exists
- **Test Steps**:
  1. POST /api/events/:eventId/bets with propId=time_range_prop, bet_value="yes"
  2. Verify response status 201
  3. Verify bet record created in database
  4. Verify bet_value="yes", resolved=false
  5. Repeat with bet_value="no"
- **Expected Results**: Bets created; database records show correct values
- **Priority**: High

#### 7. Place Bet - Exact Time
- **Test Description**: Verify user can place exact time guess
- **Pre-conditions**: User session exists; exact_time prop exists
- **Test Steps**:
  1. POST /api/events/:eventId/bets with propId=exact_time_prop, bet_value="13:47"
  2. Verify response status 201
  3. Verify bet_value="13:47" in database
  4. Try placing second exact_time bet as same user
  5. Verify response status 403 Forbidden (only one exact time guess per user)
- **Expected Results**: First bet created; second bet rejected
- **Priority**: High

#### 8. Place Bet - Yes/No Prop
- **Test Description**: Verify user can place yes/no proposition bet
- **Pre-conditions**: User session exists; yes_no prop exists
- **Test Steps**:
  1. POST /api/events/:eventId/bets with propId=yes_no_prop, bet_value="yes"
  2. Verify response status 201
  3. Repeat with bet_value="no"
  4. Verify user can bet on same yes_no prop multiple times (different answer is separate bet)
- **Expected Results**: Bets created for both yes and no answers
- **Priority**: High

#### 9. Fetch Bets (Live Feed)
- **Test Description**: Verify GET /api/events/:eventId/bets returns recent bets with pagination
- **Pre-conditions**: Event with multiple bets placed
- **Test Steps**:
  1. GET /api/events/:eventId/bets
  2. Verify response status 200
  3. Verify bets array returned with usernames and bet_values
  4. Verify pagination: default limit=50, offset=0
  5. GET /api/events/:eventId/bets?limit=10&offset=0
  6. Verify only 10 bets returned
  7. GET /api/events/:eventId/bets?limit=10&offset=10
  8. Verify second page of bets returned
- **Expected Results**: Pagination works; correct bets returned per page
- **Priority**: Medium

#### 10. Admin Can View All Bets
- **Test Description**: Verify admin can see all bet details including usernames
- **Pre-conditions**: Admin session; multiple bets placed
- **Test Steps**:
  1. Admin calls GET /api/events/:eventId/bets
  2. Verify all bets visible with usernames
  3. Non-admin calls same endpoint
  4. Verify same bets returned (no access restriction for viewing bets)
- **Expected Results**: All users can view all bets; no access restriction
- **Priority**: Medium

#### 11. SSE Live Bet Feed
- **Test Description**: Verify SSE endpoint streams new bets in real-time
- **Pre-conditions**: Event active; SSE client ready
- **Test Steps**:
  1. Client opens GET /api/events/:eventId/bets/stream (SSE connection)
  2. Verify connection opens (200 status)
  3. Another user places bet
  4. Verify SSE client receives new_bet event
  5. Verify event data includes: id, username, propId, bet_value, created_at
  6. Close connection and reopen
  7. Verify reconnection successful
- **Expected Results**: SSE stream active; new bets broadcast to all connected clients
- **Priority**: Medium

#### 12. Admin Submits Results
- **Test Description**: Verify admin can submit final time and prop answers to resolve bets
- **Pre-conditions**: Admin session; event open; bets placed
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_time="13:47", final_answers={...}
  2. Verify response status 200
  3. Verify event.status changed to "resolved"
  4. Verify event.final_time="13:47"
  5. Verify event.final_answers stored in database
  6. Verify all bets marked as resolved=true
  7. Verify points_earned calculated for each bet
- **Expected Results**: Results submitted; event resolved; bets auto-calculated
- **Priority**: High

#### 13. Admin-Only Result Submission
- **Test Description**: Verify non-admin cannot submit results
- **Pre-conditions**: Friend session; event open
- **Test Steps**:
  1. Friend calls POST /api/events/:eventId/results
  2. Verify response status 403 Forbidden
  3. Verify error message indicates admin-only action
  4. Verify event still in "open" status
- **Expected Results**: Non-admin denied; event unaffected
- **Priority**: High

#### 14. Leaderboard Retrieval
- **Test Description**: Verify leaderboard calculation and ranking
- **Pre-conditions**: Event resolved with bets calculated
- **Test Steps**:
  1. GET /api/events/:eventId/leaderboard
  2. Verify response status 200
  3. Verify leaderboard array sorted by total_points descending
  4. Verify each entry has: rank, userId, username, total_points, bets_correct, bets_total
  5. Verify rank numbering is sequential (1, 2, 3, ...)
  6. Verify total_points sum matches individual bet points
- **Expected Results**: Leaderboard correctly ranked and calculated
- **Priority**: High

#### 15. Leaderboard Before Resolution
- **Test Description**: Verify leaderboard shows all users but with 0 points before resolution
- **Pre-conditions**: Event open; bets placed but not resolved
- **Test Steps**:
  1. GET /api/events/:eventId/leaderboard
  2. Verify all users present
  3. Verify all total_points=0
  4. Verify bets_correct=0, bets_total=0
- **Expected Results**: Leaderboard shows users but no points until resolved
- **Priority**: Medium

#### 16. Time Range Bet Resolution
- **Test Description**: Verify time range bets resolve correctly (within vs outside range)
- **Pre-conditions**: Time range prop with range 12:47-13:02; bets placed
- **Test Steps**:
  1. User A bets "yes", User B bets "no"
  2. Submit final_time="13:00" (within range)
  3. Verify bet A resolved as correct (1 point); bet B resolved as incorrect (0 points)
  4. Submit final_time="13:30" (outside range)
  5. Verify bet A resolved as incorrect; bet B resolved as correct
- **Expected Results**: Range evaluation correct; points awarded properly
- **Priority**: High

#### 17. Yes/No Prop Resolution
- **Test Description**: Verify yes/no props resolve against final_answers
- **Pre-conditions**: Yes/no prop "Will Annie throw up?"; bets placed
- **Test Steps**:
  1. User A bets "yes", User B bets "no"
  2. Submit final_answers with answer="yes"
  3. Verify bet A correct (1 point); bet B incorrect (0 points)
  4. Test with answer="no"
  5. Verify bet A incorrect; bet B correct
- **Expected Results**: Yes/no evaluation correct; binary outcome handled
- **Priority**: High

#### 18. Exact Time Resolution with Ranking
- **Test Description**: Verify exact time bets ranked and scored by distance
- **Pre-conditions**: Exact time prop (weight 1.5); four users guess different times
- **Test Steps**:
  1. User A guesses "13:47", User B guesses "13:45", User C guesses "13:50", User D guesses "14:00"
  2. Submit final_time="13:47"
  3. Verify points: A=2.25 (rank 1, exact match), B=1.2 (rank 2), C=0.9 (rank 3), D=0.6 (rank 4+)
  4. Verify resolved field set to true for all
  5. Verify result field shows "correct" for A, B, C, D (all get some points)
- **Expected Results**: Ranking correct; multipliers applied correctly; all get ranked points
- **Priority**: High

#### 19. Multiple Bets Per User - Resolution
- **Test Description**: Verify user with multiple bets gets all points summed
- **Pre-conditions**: User A places 5 bets (mix of types)
- **Test Steps**:
  1. Bet 1: time_range "yes" (weight 1.0) - correct → 1 point
  2. Bet 2: time_range "no" (weight 1.0) - incorrect → 0 points
  3. Bet 3: exact_time "13:47" (weight 1.5) - rank 2 → 1.2 points
  4. Bet 4: yes_no "yes" (weight 1.0) - correct → 1 point
  5. Resolve and check total
  6. Verify total_points = 1 + 0 + 1.2 + 1 = 3.2
- **Expected Results**: Points summed across all bets; total accurate
- **Priority**: High

#### 20. Event State Prevents Betting After Closure
- **Test Description**: Verify bets cannot be placed after event closed/resolved
- **Pre-conditions**: Event status changed to "closed"
- **Test Steps**:
  1. Attempt POST /api/events/:eventId/bets with new bet
  2. Verify response status 409 Conflict
  3. Verify error message indicates event closed
  4. Change event status to "resolved"
  5. Attempt same bet placement
  6. Verify 409 Conflict again
- **Expected Results**: Betting blocked when event not open
- **Priority**: High

---

### Validation Tests

#### 1. Invalid Username - Too Short
- **Test Description**: Reject username with only 1 character
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session with username="a"
  2. Verify response status 400 Bad Request
- **Expected Results**: 400 error; user not created
- **Priority**: High

#### 2. Invalid Username - Too Long
- **Test Description**: Reject username exceeding 20 characters
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session with username="this_is_a_very_long_username_that_exceeds_twenty_characters"
  2. Verify response status 400
- **Expected Results**: 400 error; user not created
- **Priority**: High

#### 3. Invalid Username - Special Characters
- **Test Description**: Reject username with special characters
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. Test usernames: "alice@domain", "bob-smith", "user.name", "user name", "user!"
  2. Verify all return 400 Bad Request
- **Expected Results**: Only alphanumeric + underscore accepted
- **Priority**: High

#### 4. Invalid Time Format - Seconds Out of Range
- **Test Description**: Reject time with seconds > 59
- **Pre-conditions**: Exact time prop ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="13:60"
  2. Verify response status 400
  3. Test "13:75", "13:99"
  4. All should return 400
- **Expected Results**: Seconds validated to 0-59 range
- **Priority**: High

#### 5. Invalid Time Format - Missing Seconds
- **Test Description**: Reject time without seconds component
- **Pre-conditions**: Exact time prop ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="13"
  2. Verify response status 400
  3. Test "13:" (missing seconds value)
- **Expected Results**: 400 error; full MM:SS format required
- **Priority**: High

#### 6. Invalid Time Format - Hours Included
- **Test Description**: Reject time format with hours (HH:MM:SS)
- **Pre-conditions**: Exact time prop ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="1:13:47" (HH:MM:SS)
  2. Verify response status 400
- **Expected Results**: Only MM:SS format accepted
- **Priority**: Medium

#### 7. Invalid Bet Value - Wrong Type
- **Test Description**: Reject wrong bet value type for prop
- **Pre-conditions**: Various props ready
- **Test Steps**:
  1. Time range prop: POST with bet_value="1" (should be yes/no)
  2. Verify 400
  3. Exact time prop: POST with bet_value="yes" (should be MM:SS)
  4. Verify 400
  5. Yes/no prop: POST with bet_value="13:47" (should be yes/no)
  6. Verify 400
- **Expected Results**: Type validation enforced
- **Priority**: High

#### 8. Case Insensitivity - Yes/No Values
- **Test Description**: Accept "YES", "NO", "Yes", "No", "yEs" (case insensitive)
- **Pre-conditions**: Yes/no and time range props ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="YES"
  2. Verify 201 (accepted)
  3. Repeat with "NO", "Yes", "No", "yEs"
  4. All should return 201
- **Expected Results**: Case-insensitive yes/no handling
- **Priority**: Medium

#### 9. Whitespace Trimming - Yes/No Values
- **Test Description**: Accept "  yes  ", " no " (trimmed)
- **Pre-conditions**: Yes/no prop ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="  yes  "
  2. Verify 201
  3. Verify bet_value stored as "yes" (trimmed)
- **Expected Results**: Whitespace stripped; values normalized
- **Priority**: Medium

#### 10. Empty Bet Value Rejection
- **Test Description**: Reject empty or null bet_value
- **Pre-conditions**: Any prop ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value=""
  2. Verify 400
  3. POST with missing bet_value field
  4. Verify 400
  5. POST with bet_value=null
  6. Verify 400
- **Expected Results**: Empty/null values rejected
- **Priority**: High

#### 11. Missing Event ID
- **Test Description**: Reject requests with invalid/missing event ID
- **Pre-conditions**: N/A
- **Test Steps**:
  1. GET /api/events/invalid-uuid/props
  2. Verify 404 Not Found
  3. GET /api/events/00000000-0000-0000-0000-000000000000/props
  4. Verify 404 Not Found
- **Expected Results**: Invalid event IDs return 404
- **Priority**: High

#### 12. Missing Admin Secret for Admin Role
- **Test Description**: Reject admin attempt without providing secret
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session with username="admin_test" (no adminSecret field)
  2. Verify response has role="friend" (defaults to friend, not admin)
- **Expected Results**: Admin role requires secret; defaults to friend without it
- **Priority**: High

#### 13. Wrong Admin Secret
- **Test Description**: Reject admin with incorrect secret
- **Pre-conditions**: Event ready; ADMIN_SECRET="correct_secret"
- **Test Steps**:
  1. POST /api/auth/session with adminSecret="wrong_secret"
  2. Verify response status 401 Unauthorized
- **Expected Results**: 401 error; role not promoted to admin
- **Priority**: High

#### 14. SQL Injection Prevention - Username
- **Test Description**: Validate username input prevents SQL injection
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session with username="alice'; DROP TABLE users; --"
  2. Verify 400 Bad Request (invalid characters)
  3. Try username="alice\"; or 1=1; --"
  4. Verify 400 Bad Request
- **Expected Results**: Special characters rejected; table not dropped
- **Priority**: High

#### 15. SQL Injection Prevention - Bet Value
- **Test Description**: Validate bet value prevents SQL injection
- **Pre-conditions**: Time range prop ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="yes'; DELETE FROM bets; --"
  2. Verify 400 Bad Request (invalid value)
- **Expected Results**: SQL injection attempt rejected
- **Priority**: High

#### 16. XSS Prevention - Username Storage
- **Test Description**: Verify username with HTML/JS tags is stored safely and escaped on retrieval
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session with username="alice<script>alert('xss')</script>"
  2. Verify 400 Bad Request (invalid characters) or escaped storage
  3. GET /api/events/:eventId/bets
  4. Verify username is escaped/safe in JSON response
- **Expected Results**: XSS payload rejected or safely escaped
- **Priority**: High

#### 17. XSS Prevention - Bet Feed
- **Test Description**: Verify bet data in SSE stream is escaped
- **Pre-conditions**: SSE stream active
- **Test Steps**:
  1. Place bet with username containing HTML (if allowed)
  2. Check SSE message for proper escaping
  3. Verify browser doesn't execute any scripts
- **Expected Results**: SSE data is JSON; no script execution
- **Priority**: Medium

#### 18. Input Validation - Final Time Format
- **Test Description**: Verify admin-submitted final time follows MM:SS format
- **Pre-conditions**: Admin session; event ready
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_time="13:60"
  2. Verify 400 Bad Request
  3. Test final_time="13" (missing seconds)
  4. Verify 400
  5. Test final_time="1:13:47" (hours)
  6. Verify 400
- **Expected Results**: Final time validated same as bet times
- **Priority**: High

#### 19. Missing Required Final Answers
- **Test Description**: Verify all yes_no props require answers on resolution
- **Pre-conditions**: Event with 3 yes_no props; admin session
- **Test Steps**:
  1. Count yes_no props in event (assume 3)
  2. POST /api/events/:eventId/results with final_answers={prop1: "yes"} (missing 2)
  3. Verify 400 Bad Request
  4. Verify error lists missing prop IDs
- **Expected Results**: All yes_no props required
- **Priority**: High

#### 20. Invalid Final Answers - Wrong Values
- **Test Description**: Verify final answers only accept "yes" or "no"
- **Pre-conditions**: Admin session; event ready
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_answers={prop_id: "maybe"}
  2. Verify 400 Bad Request
  3. Test with "y", "n", "true", "false"
  4. All should return 400
- **Expected Results**: Only "yes" and "no" accepted
- **Priority**: High

---

### Edge Case Tests

#### 1. Time Range Boundary - Exact Start Time
- **Test Description**: Verify final time exactly at range start is considered inside range
- **Pre-conditions**: Time range prop 12:47-13:02
- **Test Steps**:
  1. User bets "yes"
  2. Submit final_time="12:47" (exactly at range_start_seconds)
  3. Verify bet resolved as correct
- **Expected Results**: Boundary inclusive on start
- **Priority**: Medium

#### 2. Time Range Boundary - Exact End Time
- **Test Description**: Verify final time exactly at range end is considered inside range
- **Pre-conditions**: Time range prop 12:47-13:02
- **Test Steps**:
  1. User bets "yes"
  2. Submit final_time="13:02" (exactly at range_end_seconds)
  3. Verify bet resolved as correct
- **Expected Results**: Boundary inclusive on end
- **Priority**: Medium

#### 3. Time Range Boundary - One Second Before Start
- **Test Description**: Verify final time one second before range start is outside
- **Pre-conditions**: Time range prop 12:47-13:02 (range_start_seconds=767)
- **Test Steps**:
  1. User bets "yes"
  2. Submit final_time="12:46" (766 seconds)
  3. Verify bet resolved as incorrect
- **Expected Results**: Boundary strictly enforced
- **Priority**: Medium

#### 4. Time Range Boundary - One Second After End
- **Test Description**: Verify final time one second after range end is outside
- **Pre-conditions**: Time range prop 12:47-13:02 (range_end_seconds=782)
- **Test Steps**:
  1. User bets "yes"
  2. Submit final_time="13:03" (783 seconds)
  3. Verify bet resolved as incorrect
- **Expected Results**: Boundary strictly enforced
- **Priority**: Medium

#### 5. Exact Time - Zero Second Difference (Exact Match)
- **Test Description**: Verify exact match (0 second difference) receives bonus multiplier
- **Pre-conditions**: Exact time prop (weight 1.5); user guesses exact time
- **Test Steps**:
  1. User guesses "13:47"
  2. Final time = "13:47"
  3. Difference = 0 seconds
  4. Verify multiplier = 1.5 (bonus)
  5. Verify points = 1.5 * 1.5 = 2.25
- **Expected Results**: Exact match receives full bonus
- **Priority**: High

#### 6. Exact Time - One Second Difference (Rank 1 vs 2)
- **Test Description**: Verify 1 second vs 2 second difference results in correct ranking
- **Pre-conditions**: Two users guess 1s and 2s off from actual
- **Test Steps**:
  1. User A guesses "13:47", User B guesses "13:46"
  2. Final time = "13:48"
  3. User A diff=1s, User B diff=2s
  4. Both should be Rank 1 or Rank 2 (A=Rank 1, B=Rank 2)
  5. Verify multipliers: A=1.0, B=0.8
- **Expected Results**: 1s difference prioritized over 2s
- **Priority**: High

#### 7. Many Exact Time Guesses - Overflow Rank 4+
- **Test Description**: Verify all guesses beyond rank 4 receive same multiplier (0.4)
- **Pre-conditions**: 10 users place exact time guesses with varying differences
- **Test Steps**:
  1. Create 10 guesses: 0s, 1s, 2s, 3s, 5s, 10s, 15s, 20s, 30s, 60s
  2. Verify ranks 1-3 get 1.0, 0.8, 0.6
  3. Verify ranks 4-10 all get 0.4 multiplier
- **Expected Results**: Multiplier consistent for Rank 4+
- **Priority**: Medium

#### 8. Zero Users Place Bets
- **Test Description**: Verify event with no bets resolves gracefully
- **Pre-conditions**: Event created; no users join or place bets
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_time and answers
  2. Verify response status 200
  3. Verify event resolved
  4. GET /api/events/:eventId/leaderboard
  5. Verify empty leaderboard
- **Expected Results**: Handles zero-bet scenario
- **Priority**: Medium

#### 9. One User Places All Bets
- **Test Description**: Verify single user placing multiple bets is scored correctly
- **Pre-conditions**: Single user in event
- **Test Steps**:
  1. User places 10 bets (mix of types)
  2. Resolve event
  3. Verify leaderboard shows user with combined points
  4. Verify total_points is sum of all bet points
- **Expected Results**: Single user scored correctly
- **Priority**: Medium

#### 10. Username Case Sensitivity
- **Test Description**: Verify usernames are case-sensitive (alice ≠ Alice)
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session with username="alice"
  2. Verify 201, user created
  3. POST /api/auth/session with username="Alice"
  4. Verify 201, different user created
  5. Verify two user records in database
- **Expected Results**: Usernames are case-sensitive
- **Priority**: Medium

#### 11. Very Long Event Name
- **Test Description**: Verify event name can be long (up to 255 chars)
- **Pre-conditions**: Event creation endpoint
- **Test Steps**:
  1. Create event with name = 255 character string
  2. Verify event created successfully
  3. Verify name stored and retrieved correctly
- **Expected Results**: Long names handled
- **Priority**: Low

#### 12. Very Large Number of Bets
- **Test Description**: Verify system handles 1000+ bets without performance degradation
- **Pre-conditions**: Event with many users (50+) placing many bets
- **Test Steps**:
  1. Place 1000 bets across 50 users
  2. GET /api/events/:eventId/bets with default pagination
  3. Verify response time < 1 second
  4. POST /api/events/:eventId/results
  5. Verify resolution completes within 500ms
- **Expected Results**: Performance acceptable; resolution fast
- **Priority**: Low

#### 13. Immediate Resolution After Bet Creation
- **Test Description**: Verify resolving immediately after betting works
- **Pre-conditions**: Event ready
- **Test Steps**:
  1. POST /api/auth/session
  2. POST /api/events/:eventId/bets (place bet)
  3. Immediately POST /api/events/:eventId/results
  4. Verify bet resolved correctly
- **Expected Results**: No race condition; all bets resolved
- **Priority**: Medium

#### 14. Overlapping Time Ranges
- **Test Description**: Verify user can place bets on overlapping ranges
- **Pre-conditions**: Two overlapping range props (e.g., 12:00-13:00 and 12:30-13:30)
- **Test Steps**:
  1. User places bets on both ranges
  2. Final time = 12:45 (in both ranges)
  3. User bets "yes" on both
  4. Verify both bets resolved as correct
- **Expected Results**: Overlaps allowed; each evaluated independently
- **Priority**: Medium

#### 15. Resolution Idempotency
- **Test Description**: Verify submitting results twice doesn't double-count or error
- **Pre-conditions**: Event open; bets placed
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_time="13:47", answers={...}
  2. Verify 200; event resolved
  3. Call same endpoint again with same data
  4. Verify 409 Conflict (already resolved) or 200 idempotent response
- **Expected Results**: Either blocked or idempotent
- **Priority**: High

#### 16. Floating Point Precision in Scoring
- **Test Description**: Verify scoring with decimal weights doesn't lose precision
- **Pre-conditions**: Bets with weights 1.5, exact time with multiplier 1.5
- **Test Steps**:
  1. Exact time bet: weight=1.5
  2. User rank 2: multiplier=0.8
  3. Points = 1.5 * 0.8 = 1.2
  4. User rank 3: multiplier=0.6
  5. Points = 1.5 * 0.6 = 0.9
  6. Verify no rounding errors; sums correct
- **Expected Results**: Floating point arithmetic correct
- **Priority**: Medium

#### 17. Maximum Minutes Value
- **Test Description**: Verify times with MM up to 99 are accepted
- **Pre-conditions**: Exact time prop
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="99:59"
  2. Verify 201 (accepted)
  3. Submit final_time="99:59"
  4. Verify accepted and processed
- **Expected Results**: Minutes 0-99 supported
- **Priority**: Low

#### 18. Duplicate Exact Time Guesses (Same Exact Time)
- **Test Description**: Verify multiple users can guess same exact time; all are ranked by creation_at
- **Pre-conditions**: Exact time prop; two users
- **Test Steps**:
  1. User A at 10:00:00 guesses "13:47"
  2. User B at 10:00:05 guesses "13:47"
  3. Final time = "13:47"
  4. Both have 0 second difference
  5. Verify User A Rank 1 (exact match bonus = 1.5)
  6. Verify User B Rank 2 (tie broken by creation_at, gets 0.8)
- **Expected Results**: Ties broken by creation_at; both ranked
- **Priority**: High

#### 19. Session Persistence Across Multiple Requests
- **Test Description**: Verify session cookie maintained across requests
- **Pre-conditions**: User session created
- **Test Steps**:
  1. POST /api/auth/session, receive cookie
  2. Using same cookie, POST /api/events/:eventId/bets
  3. Verify bet created under correct user
  4. GET /api/events/:eventId/bets
  5. Verify user appears in bets
  6. Session valid across 3+ API calls
- **Expected Results**: Session cookie works across requests
- **Priority**: High

#### 20. Empty Final Answers Object
- **Test Description**: Verify event with no yes_no props can be resolved with empty final_answers
- **Pre-conditions**: Event with only time_range and exact_time props (no yes_no props)
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_time="13:47", final_answers={}
  2. Verify 200; event resolved successfully
- **Expected Results**: Empty answers accepted when no yes_no props exist
- **Priority**: Medium

---

### Error Tests

#### 1. Database Connection Failure
- **Test Description**: Verify graceful handling of database unavailability
- **Pre-conditions**: Database connection interrupted
- **Test Steps**:
  1. Disconnect database
  2. POST /api/events/:eventId/bets
  3. Verify response status 500 Server Error
  4. Verify error message generic (no database internals exposed)
  5. Verify no user data revealed in error
- **Expected Results**: 500 error; no sensitive data leaked
- **Priority**: High

#### 2. Invalid JSON in Request Body
- **Test Description**: Verify malformed JSON is rejected
- **Pre-conditions**: API endpoint ready
- **Test Steps**:
  1. POST /api/events/:eventId/bets with body: `{invalid json}`
  2. Verify 400 Bad Request
- **Expected Results**: Malformed JSON caught and rejected
- **Priority**: High

#### 3. Invalid Content-Type Header
- **Test Description**: Verify request without JSON content-type is handled
- **Pre-conditions**: API endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/bets with Content-Type: text/plain
  2. Verify 400 Bad Request or correct parsing
- **Expected Results**: Invalid content-type rejected
- **Priority**: Medium

#### 4. Missing Authorization Header/Cookie
- **Test Description**: Verify request without session is rejected appropriately
- **Pre-conditions**: Protected endpoint (e.g., results submission)
- **Test Steps**:
  1. POST /api/events/:eventId/results without session cookie
  2. Verify 401 Unauthorized
- **Expected Results**: 401 error; must have session
- **Priority**: High

#### 5. Expired Session Cookie
- **Test Description**: Verify expired session is rejected
- **Pre-conditions**: Session created with 1-second expiry
- **Test Steps**:
  1. Create session
  2. Wait 2 seconds
  3. Use session cookie in request
  4. Verify 401 Unauthorized
- **Expected Results**: Expired session rejected
- **Priority**: High

#### 6. Tampered Session Cookie
- **Test Description**: Verify tampered/forged session is rejected
- **Pre-conditions**: Valid session cookie
- **Test Steps**:
  1. Modify last character of session cookie
  2. POST /api/events/:eventId/bets with modified cookie
  3. Verify 401 Unauthorized
- **Expected Results**: Tampered cookie detected and rejected
- **Priority**: High

#### 7. Foreign Key Constraint Violation
- **Test Description**: Verify invalid prop ID returns 404
- **Pre-conditions**: Bet placement endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/bets with propId="00000000-0000-0000-0000-000000000000"
  2. Verify 404 Not Found (prop doesn't exist)
- **Expected Results**: Foreign key error converted to 404
- **Priority**: Medium

#### 8. Race Condition - Multiple Simultaneous Bets
- **Test Description**: Verify duplicate exact time bet is caught even with simultaneous submissions
- **Pre-conditions**: Exact time prop ready; two concurrent requests
- **Test Steps**:
  1. Send two simultaneous POST /api/events/:eventId/bets (same user, same exact_time prop)
  2. Verify one returns 201; other returns 403 Forbidden
  3. Verify only one bet in database
- **Expected Results**: Race condition handled; only one exact time guess per user enforced
- **Priority**: High

#### 9. Concurrent Result Submissions
- **Test Description**: Verify second result submission is blocked/rejected
- **Pre-conditions**: Two concurrent admin requests to results endpoint
- **Test Steps**:
  1. Send two simultaneous POST /api/events/:eventId/results
  2. One should succeed (200)
  3. Other should return 409 Conflict (already resolved)
- **Expected Results**: First wins; second blocked
- **Priority**: High

#### 10. Result Submission with Missing Event
- **Test Description**: Verify resolving non-existent event returns 404
- **Pre-conditions**: Admin session
- **Test Steps**:
  1. POST /api/events/00000000-0000-0000-0000-000000000000/results
  2. Verify 404 Not Found
- **Expected Results**: 404 error
- **Priority**: Medium

#### 11. Leaderboard for Non-Existent Event
- **Test Description**: Verify leaderboard request for invalid event returns 404
- **Pre-conditions**: N/A
- **Test Steps**:
  1. GET /api/events/00000000-0000-0000-0000-000000000000/leaderboard
  2. Verify 404 Not Found
- **Expected Results**: 404 error
- **Priority**: Medium

#### 12. SSE Connection Interrupted
- **Test Description**: Verify SSE client reconnection handling
- **Pre-conditions**: SSE stream active
- **Test Steps**:
  1. Client opens GET /api/events/:eventId/bets/stream
  2. Stream receives events
  3. Client disconnects
  4. Client reconnects with same or new connection
  5. Verify new messages received after reconnection
- **Expected Results**: Reconnection works; no message loss expected in MVP
- **Priority**: Medium

#### 13. SSE Client Timeout
- **Test Description**: Verify server handles long-lived SSE connections
- **Pre-conditions**: SSE stream open for extended period
- **Test Steps**:
  1. Open SSE connection
  2. Keep open for 5+ minutes without events
  3. Verify server doesn't close connection prematurely
  4. Send heartbeat/keep-alive if implemented
- **Expected Results**: Connection maintained
- **Priority**: Low

#### 14. Null Bet Value
- **Test Description**: Verify null bet_value is rejected
- **Pre-conditions**: Bet placement endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value=null
  2. Verify 400 Bad Request
- **Expected Results**: Null caught and rejected
- **Priority**: High

#### 15. Undefined Bet Value
- **Test Description**: Verify missing bet_value field is rejected
- **Pre-conditions**: Bet placement endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/bets with propId only (no bet_value)
  2. Verify 400 Bad Request
- **Expected Results**: Missing field caught
- **Priority**: High

#### 16. Very Large Bet Value String
- **Test Description**: Verify oversized bet value is rejected
- **Pre-conditions**: Bet placement endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value = 10,000 character string
  2. Verify 400 Bad Request (or truncated to reasonable length)
- **Expected Results**: Oversized input rejected
- **Priority**: Low

#### 17. Special Characters in Final Time
- **Test Description**: Verify final_time rejects special characters
- **Pre-conditions**: Results endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/results with final_time="13:47&test"
  2. Verify 400 Bad Request
- **Expected Results**: Invalid characters rejected
- **Priority**: High

#### 18. Negative Minutes in Time
- **Test Description**: Verify negative minutes rejected
- **Pre-conditions**: Bet placement; results endpoint
- **Test Steps**:
  1. POST /api/events/:eventId/bets with bet_value="-1:30"
  2. Verify 400 Bad Request
- **Expected Results**: Negative values rejected
- **Priority**: Medium

#### 19. Division by Zero in Leaderboard Calculation
- **Test Description**: Verify leaderboard doesn't crash if edge case causes division
- **Pre-conditions**: Event with specific bet configuration
- **Test Steps**:
  1. Ensure event has users but no correct bets
  2. GET /api/events/:eventId/leaderboard
  3. Verify 200; all users shown with 0 points
- **Expected Results**: No crash; graceful handling
- **Priority**: Low

#### 20. Error Response Format Consistency
- **Test Description**: Verify all error responses follow consistent format
- **Pre-conditions**: Various endpoints ready
- **Test Steps**:
  1. Trigger 400 error (bad input)
  2. Verify response includes: status code, error message, timestamp
  3. Trigger 403 error (unauthorized)
  4. Verify same structure
  5. Trigger 404, 409, 500 errors
  6. All follow consistent format
- **Expected Results**: All errors consistently structured
- **Priority**: Medium

---

## Results Summary

**DESIGN PHASE - NOT YET EXECUTED**

- Total Test Cases Designed: 120
- Unit Tests: 10
- Integration Tests: 20
- Validation Tests: 20
- Edge Case Tests: 20
- Error Tests: 20

---

## Implementation Notes

### Test Execution Strategy
1. Start with unit tests (scoring, validation) - fastest feedback
2. Progress to integration tests (API endpoints) - verify contracts
3. Run validation and edge case tests concurrently
4. Error tests at the end to ensure graceful failures
5. Use test database with rollback/cleanup between test suites

### Test Data Setup
- Create seed fixture for event with all prop types
- Create seed fixture for sample users (admin, annie, friends)
- Create factories for bets with various values
- Use transaction rollback for isolation

### Performance Benchmarks
- Bet placement: < 1 second
- Resolution calculation: < 500ms
- Leaderboard retrieval: < 500ms
- SSE message latency: < 100ms

### Mocking Strategy
- Mock database queries for unit tests
- Use real database for integration tests
- Mock external APIs (if any) for error scenarios
- Mock time for session expiry tests

---

## Next Steps

1. **Await User Approval** - Review test cases and approve before implementation
2. **Set Up Test Infrastructure** - Create test database, fixtures, utilities
3. **Implement Test Suite** - Write tests in Vitest/Jest
4. **Execute Tests** - Run against implementation
5. **Report Results** - Update this document with pass/fail status
6. **Address Failures** - Debug and escalate to developer if needed
