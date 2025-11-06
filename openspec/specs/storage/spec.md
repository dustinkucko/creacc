# Storage Capability


## Purpose

Abstract storage layer for Git data supporting block volumes, network filesystems, and local storage.

## Requirements

This capability abstracts storage operations to support multiple backends (local filesystem, cloud block volumes, network filesystems) while ensuring POSIX semantics and path normalization. All storage implementations MUST provide atomic operations and proper file locking.

### Requirement: Storage Interface Definition
The system SHALL provide a StorageAdapter interface abstracting read/write operations for Git repositories across different storage backends.

#### Scenario: Define storage adapter interface
- **GIVEN** need to support multiple storage backends
- **WHEN** defining StorageAdapter interface
- **THEN** provide methods for read, write, delete, exists, list operations

#### Scenario: Swap storage implementations
- **GIVEN** an application using local storage adapter
- **WHEN** switching to block volume adapter
- **THEN** application continues working without code changes

### Requirement: Local Filesystem Implementation
The system SHALL provide a local filesystem implementation with atomic writes and proper file locking for development and testing.

#### Scenario: Atomic file write
- **GIVEN** a file being written to local storage
- **WHEN** write operation is interrupted
- **THEN** file is either fully written or not present (no partial writes)

#### Scenario: File locking for concurrent access
- **GIVEN** multiple processes accessing same file
- **WHEN** one process holds a write lock
- **THEN** other processes wait or receive lock contention error

### Requirement: Cloud Block Volume Support
The system SHALL support cloud block volumes (DigitalOcean Volumes, AWS EBS) formatted with ZFS for production deployments.

#### Scenario: Mount block volume at /srv/git
- **GIVEN** a cloud block volume attached to VM
- **WHEN** mounting at /srv/git
- **THEN** volume is accessible with strong POSIX semantics

#### Scenario: Single-attach enforcement
- **GIVEN** a block volume mounted on one VM
- **WHEN** attempting to attach to second VM
- **THEN** operation is rejected or first mount is detached

### Requirement: Network Filesystem Support
The system SHALL support validated POSIX network filesystems (TigrisFS) that meet Git requirements for atomic operations and locking.

#### Scenario: Validate TigrisFS POSIX compliance
- **GIVEN** TigrisFS mounted filesystem
- **WHEN** testing atomic rename, hard links, fsync
- **THEN** all operations behave correctly per POSIX spec

#### Scenario: Single-writer enforcement for network FS
- **GIVEN** network filesystem mounted on primary VM
- **WHEN** secondary VM attempts RW mount
- **THEN** mount is rejected or coordinated to prevent split-brain

### Requirement: Path Normalization Integration
The system SHALL integrate with identity module for Unicode NFC path normalization on all storage operations.

#### Scenario: Normalize paths before storage
- **GIVEN** a file path with non-NFC Unicode characters
- **WHEN** storing file via storage adapter
- **THEN** path is normalized to NFC before filesystem operation

#### Scenario: Reject denormalized path variants
- **GIVEN** storage containing `café.txt` (NFC)
- **WHEN** attempting to store `café.txt` (NFD - decomposed)
- **THEN** operation rejected with normalization conflict error

### Requirement: Configuration-Based Adapter Selection
The system SHALL select storage adapter based on configuration allowing deployment-specific storage strategies.

#### Scenario: Load local adapter in development
- **GIVEN** configuration specifies `storage.type: local`
- **WHEN** initializing storage adapter
- **THEN** local filesystem adapter is instantiated

#### Scenario: Load block volume adapter in production
- **GIVEN** configuration specifies `storage.type: block-volume`
- **WHEN** initializing storage adapter
- **THEN** ZFS block volume adapter is instantiated with production settings
