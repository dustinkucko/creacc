# Conflict Resolution Capability


## Purpose

Detect and resolve conflicts during check-in with auto-rebase and user-assisted resolution.

## Requirements

This capability handles merge conflicts during check-in operations. The system MUST preserve data integrity while providing clear user choices for conflict resolution including auto-rebase and manual resolution strategies.

### Requirement: Stale Base Detection
The system SHALL detect when check-in base commit differs from current HEAD indicating concurrent changes.

#### Scenario: Detect stale base
- **GIVEN** lease with base commit A, HEAD now at commit C
- **WHEN** checking stale base on check-in
- **THEN** detect stale base, return divergence info (commits B,C)

#### Scenario: Clean base allows fast-forward
- **GIVEN** lease with base commit A, HEAD still at A
- **WHEN** checking base staleness
- **THEN** base is clean, allow direct commit

#### Scenario: Calculate divergence
- **GIVEN** base commit A, HEAD at C via B
- **WHEN** detecting staleness
- **THEN** return divergence count=2 commits (B, C)

### Requirement: Automatic Rebase
The system SHALL automatically rebase check-in changes onto current HEAD when base is stale.

#### Scenario: Auto-rebase non-conflicting changes
- **GIVEN** stale base, user changed file X, HEAD changed file Y
- **WHEN** performing auto-rebase
- **THEN** replay user changes onto HEAD, create new commit

#### Scenario: Preserve authorship during rebase
- **GIVEN** auto-rebase creating new commit
- **WHEN** setting commit metadata
- **THEN** user is author, server is committer

#### Scenario: Update lease base after rebase
- **GIVEN** successful rebase from base A to HEAD C
- **WHEN** rebase completes
- **THEN** update lease base_commit to C

### Requirement: LFS Pointer Conflict Resolution
The system SHALL handle LFS pointer conflicts when same binary file modified in both branches.

#### Scenario: Detect LFS pointer conflict
- **GIVEN** base has `image.png` pointer OID=X, HEAD has OID=Y, user has OID=Z
- **WHEN** rebasing user changes
- **THEN** detect three-way conflict on LFS pointer

#### Scenario: Present conflict resolution UI
- **GIVEN** LFS pointer conflict
- **WHEN** presenting to user
- **THEN** show options: ours (Z), theirs (Y), keep-both (rename)

#### Scenario: Resolve with "ours" strategy
- **GIVEN** user selects "ours"
- **WHEN** resolving conflict
- **THEN** use user's version (OID=Z), continue rebase

#### Scenario: Resolve with "keep-both" strategy
- **GIVEN** user selects "keep-both"
- **WHEN** resolving conflict
- **THEN** rename one file (`image.png` → `image-conflict.png`), keep both versions

### Requirement: Deletion Confirmation
The system SHALL flag missing files within check-in scope as deletions requiring explicit user confirmation.

#### Scenario: Detect missing file in scope
- **GIVEN** lease scope includes `/docs/api.md`
- **WHEN** user uploads changes without `api.md`
- **THEN** flag as pending deletion requiring confirmation

#### Scenario: Confirm intentional deletion
- **GIVEN** flagged deletion of `api.md`
- **WHEN** user confirms deletion
- **THEN** include file removal in commit

#### Scenario: Reject unconfirmed deletions
- **GIVEN** flagged deletions not confirmed
- **WHEN** attempting check-in
- **THEN** reject with unconfirmed deletions error

#### Scenario: Preserve out-of-scope files
- **GIVEN** sparse lease scope `/docs/*`
- **WHEN** user uploads changes
- **THEN** do not flag deletions for files outside `/docs/*`

### Requirement: Path Normalization During Conflict Detection
The system SHALL normalize paths to NFC before comparing for conflicts to prevent false positives.

#### Scenario: Normalize before conflict check
- **GIVEN** user changed `café.txt` (NFD), HEAD changed `café.txt` (NFC)
- **WHEN** detecting conflicts
- **THEN** normalize both to NFC, recognize as same file

#### Scenario: Reject normalization-only changes
- **GIVEN** file path differs only in normalization form
- **WHEN** attempting check-in
- **THEN** reject with normalization conflict error

### Requirement: Conflict Resolution Audit
The system SHALL record all conflict resolutions in audit log with chosen strategy and affected files.

#### Scenario: Log LFS conflict resolution
- **GIVEN** LFS pointer conflict resolved with "ours"
- **WHEN** completing check-in
- **THEN** record audit entry with file=`image.png`, strategy=ours

#### Scenario: Log deletion confirmations
- **GIVEN** confirmed deletion of `old-file.md`
- **WHEN** completing check-in
- **THEN** record audit entry with deleted file and confirmation timestamp

#### Scenario: Include resolutions in commit message
- **GIVEN** successful check-in after conflict resolution
- **WHEN** creating commit
- **THEN** append conflict resolution summary to commit message
