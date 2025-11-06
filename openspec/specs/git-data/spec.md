# Git Data Capability


## Purpose

Bare Git repository management with Git-accurate object creation and dual hashing.

## Requirements

This capability implements Git repository operations with strict adherence to Git object formats and dual-hash computation. All operations MUST be verifiable with native Git CLI tools (git fsck, git cat-file, etc.).

### Requirement: Repository Initialization
The system SHALL initialize bare Git repositories with correct structure and default branch configuration.

#### Scenario: Initialize bare repository
- **GIVEN** a space ID and repository name
- **WHEN** initializing repository
- **THEN** create bare repo at `/srv/git/<space_id>/<repo_id>` with objects/, refs/, HEAD

#### Scenario: Set default branch to main
- **GIVEN** newly initialized repository
- **WHEN** checking HEAD reference
- **THEN** HEAD points to `refs/heads/main`

#### Scenario: Validate with git fsck
- **GIVEN** repository initialized by system
- **WHEN** running `git fsck --full`
- **THEN** validation passes with no errors

### Requirement: Dual Hash Object Storage
The system SHALL store Git objects (blobs, trees, commits, tags) with both SHA-1 and SHA-256 hashes computed from canonical Git object formats.

#### Scenario: Store blob with dual hashes
- **GIVEN** file content "Hello World"
- **WHEN** storing as blob object
- **THEN** compute SHA-1 and SHA-256 from `blob 11\0Hello World` and store both OIDs

#### Scenario: Store tree with dual hashes
- **GIVEN** tree entries sorted bytewise with correct modes
- **WHEN** storing tree object
- **THEN** compute dual hashes from canonical `tree <size>\0<entries>` format

#### Scenario: Store commit with dual hashes
- **GIVEN** commit metadata (tree, parent, author, committer, message)
- **WHEN** storing commit object
- **THEN** compute dual hashes and persist root_tree_sha1/sha256

### Requirement: Reference Management
The system SHALL manage Git references (branches, tags) and reflogs with atomic updates.

#### Scenario: Update branch reference atomically
- **GIVEN** branch `main` at commit A
- **WHEN** updating to commit B with old OID = A
- **THEN** ref updated only if current value matches A (compare-and-swap)

#### Scenario: Record reflog entry
- **GIVEN** reference update from A to B
- **WHEN** updating ref
- **THEN** append reflog entry with committer, timestamp, old/new OID, message

#### Scenario: Reject stale reference update
- **GIVEN** branch `main` at commit C (not B)
- **WHEN** attempting update with old OID = B
- **THEN** reject update with stale reference error

### Requirement: Ephemeral Worktree Creation
The system SHALL create ephemeral worktrees for check-out operations with specified base commit and scope.

#### Scenario: Create full-scope worktree
- **GIVEN** repository and base commit SHA
- **WHEN** creating worktree with scope type=full
- **THEN** checkout all files from base commit to temporary directory

#### Scenario: Create sparse-scope worktree
- **GIVEN** repository, base commit, and path list
- **WHEN** creating worktree with scope type=sparse
- **THEN** checkout only specified paths using sparse-checkout

#### Scenario: Clean up worktree after use
- **GIVEN** ephemeral worktree at `/tmp/worktree-<lease-id>`
- **WHEN** lease is completed or expired
- **THEN** remove worktree directory and Git metadata

### Requirement: Git Object Format Validation
The system SHALL validate all Git objects against canonical format specifications before storage.

#### Scenario: Validate blob format
- **GIVEN** blob object bytes
- **WHEN** validating format
- **THEN** verify header is `blob <size>\0` followed by exact size bytes

#### Scenario: Validate tree entry modes
- **GIVEN** tree object with file entries
- **WHEN** validating modes
- **THEN** accept only 100644 (regular), 100755 (executable), 120000 (symlink), 040000 (tree)

#### Scenario: Validate tree entry sorting
- **GIVEN** tree with multiple entries
- **WHEN** validating entry order
- **THEN** verify entries sorted bytewise (directories treated as `name/`)

### Requirement: Storage Layer Integration
The system SHALL use storage abstraction for all repository I/O ensuring POSIX semantics and path normalization.

#### Scenario: Write object through storage adapter
- **GIVEN** computed blob object and dual OIDs
- **WHEN** storing object
- **THEN** use storage adapter write operation with normalized path

#### Scenario: Verify atomic storage operations
- **GIVEN** concurrent object writes
- **WHEN** storing objects via storage layer
- **THEN** each write is atomic (no partial writes visible)
