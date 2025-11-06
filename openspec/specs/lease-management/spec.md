# Lease Management Capability


## Purpose

Time-bounded exclusive locks for check-out/check-in workflows with 72h default duration.

## Requirements

This capability enforces exclusive access control for check-out/check-in workflows. Leases MUST prevent overlapping access, expire automatically without grace periods, and support admin eviction.

### Requirement: Lease Creation
The system SHALL create leases with exclusive lock on project scope for specified duration (default 72 hours).

#### Scenario: Create full-scope lease
- **GIVEN** project ID, user ID, base commit
- **WHEN** creating lease with scope type=full
- **THEN** create lease record with 72h expiration, status=active

#### Scenario: Create sparse-scope lease
- **GIVEN** project ID, user ID, base commit, path list
- **WHEN** creating lease with scope type=sparse
- **THEN** create lease with explicit path scope, 72h expiration

#### Scenario: Generate unique lease ID
- **GIVEN** lease creation request
- **WHEN** creating lease
- **THEN** generate ULID for lease_id ensuring uniqueness

### Requirement: Lease Validation
The system SHALL validate lease ownership and status before allowing check-in operations.

#### Scenario: Validate active lease
- **GIVEN** active lease owned by User A
- **WHEN** User A attempts check-in
- **THEN** validation passes, return remaining time and scope

#### Scenario: Reject expired lease
- **GIVEN** lease created 73 hours ago
- **WHEN** attempting check-in
- **THEN** reject with lease expired error

#### Scenario: Reject wrong user
- **GIVEN** lease owned by User A
- **WHEN** User B attempts check-in
- **THEN** reject with ownership mismatch error

### Requirement: Lease Expiration
The system SHALL automatically expire leases after duration with no auto-renewal and no grace period.

#### Scenario: Expire lease at 72 hours
- **GIVEN** lease created at time T
- **WHEN** current time is T+72h+1second
- **THEN** lease status transitions to expired

#### Scenario: No auto-renewal
- **GIVEN** lease expiring in 1 hour
- **WHEN** expiration time reached
- **THEN** lease expires without automatic extension

#### Scenario: Batch expiration check
- **GIVEN** system cron job running every minute
- **WHEN** checking for expired leases
- **THEN** update all leases with expires_at < now to status=expired

### Requirement: Expiration Reminders
The system SHALL send reminder notifications 2 hours before lease expiration.

#### Scenario: Send 2-hour reminder
- **GIVEN** lease expiring at T+72h
- **WHEN** current time is T+70h
- **THEN** send notification to user with remaining time

#### Scenario: Reminder includes extension instructions
- **GIVEN** expiration reminder
- **WHEN** sending notification
- **THEN** include instructions to complete work or request early check-in

#### Scenario: One reminder per lease
- **GIVEN** reminder sent at T+70h
- **WHEN** lease still active at T+71h
- **THEN** do not send duplicate reminder

### Requirement: Admin Eviction
The system SHALL allow administrators to forcibly evict leases with recorded reason.

#### Scenario: Evict active lease
- **GIVEN** admin user and active lease
- **WHEN** issuing eviction with reason "Emergency maintenance"
- **THEN** set lease status=evicted, record admin ID, reason, timestamp

#### Scenario: Notify evicted user
- **GIVEN** lease eviction
- **WHEN** eviction completes
- **THEN** send notification to lease owner with reason

#### Scenario: Prevent check-in after eviction
- **GIVEN** evicted lease
- **WHEN** user attempts check-in
- **THEN** reject with lease evicted error

### Requirement: Scope Overlap Detection
The system SHALL prevent overlapping leases on same project scope to ensure exclusivity.

#### Scenario: Detect full-scope overlap
- **GIVEN** active full-scope lease on Project A
- **WHEN** requesting new lease on Project A
- **THEN** reject with active lease conflict error

#### Scenario: Allow non-overlapping sparse scopes
- **GIVEN** active lease on paths `/docs/*`
- **WHEN** requesting lease on paths `/src/*`
- **THEN** allow lease creation (non-overlapping scopes)

#### Scenario: Detect partial sparse overlap
- **GIVEN** active lease on `/docs/api.md`
- **WHEN** requesting lease including `/docs/api.md`
- **THEN** reject with scope overlap conflict

### Requirement: Lease Completion
The system SHALL mark leases as completed after successful check-in and release exclusivity.

#### Scenario: Complete lease on check-in
- **GIVEN** active lease and successful check-in
- **WHEN** check-in commits changes
- **THEN** set lease status=completed, record completion timestamp

#### Scenario: Release scope exclusivity
- **GIVEN** completed lease
- **WHEN** checking scope availability
- **THEN** scope is available for new leases

#### Scenario: Audit lease lifecycle
- **GIVEN** completed lease
- **WHEN** querying lease history
- **THEN** show created_at, expires_at, completed_at, status transitions
