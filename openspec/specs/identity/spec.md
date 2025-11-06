# Identity Capability

## Purpose

Provide canonical identity utilities including dual SHA-1/SHA-256 computation for Git objects, ULID generation/validation for stable project identifiers, Unicode NFC path normalization, case-collision detection, and YAML configuration validation.

## Requirements

This capability provides the foundational identity and validation primitives used across all Git operations. All hashing operations MUST produce Git-accurate results that validate with standard Git tools.

### Requirement: Dual OID Computation
The system SHALL compute both SHA-1 and SHA-256 object identifiers for all Git objects (blobs, trees, commits, tags) using canonical Git object formats.

#### Scenario: Compute dual OID for blob
- **GIVEN** a file with content "Hello World"
- **WHEN** computing OIDs using Git canonical format `blob <size>\0<content>`
- **THEN** return both SHA-1 (40 hex chars) and SHA-256 (64 hex chars) hashes

#### Scenario: Validate against git CLI
- **GIVEN** a Git object stored using dual OID
- **WHEN** validating with `git fsck`
- **THEN** validation passes with no errors

### Requirement: ULID Generation and Validation
The system SHALL generate and validate ULIDs (or UUID v7) for stable project/repository identifiers that remain constant across renames and moves.

#### Scenario: Generate valid ULID
- **GIVEN** an optional timestamp
- **WHEN** generating a ULID
- **THEN** return 26-character time-sortable string matching ULID specification

#### Scenario: Validate ULID format
- **GIVEN** a candidate identifier string
- **WHEN** validating ULID format
- **THEN** accept valid ULIDs and reject malformed strings

#### Scenario: ULID immutability across renames
- **GIVEN** a project with ULID `01J9Z2Q3W8KFX8Q3R2A5V7YB1M`
- **WHEN** project is renamed from "Project A" to "Project B"
- **THEN** ULID remains unchanged

### Requirement: Unicode Path Normalization
The system SHALL normalize all file paths to Unicode NFC (Normalization Form Canonical Composition) and reject paths that differ only in normalization form.

#### Scenario: Normalize path to NFC
- **GIVEN** a file path with decomposed Unicode characters
- **WHEN** normalizing the path
- **THEN** return NFC-normalized path and flag if changed

#### Scenario: Reject normalization-only variants
- **GIVEN** two paths differing only by normalization (é vs e + combining accent)
- **WHEN** attempting to add both to repository
- **THEN** reject the second path with normalization conflict error

### Requirement: Case Collision Detection
The system SHALL detect and block sibling file paths that differ only in case (e.g., `README.md` and `readme.md` in same directory).

#### Scenario: Detect case collision in tree
- **GIVEN** a tree containing `README.md`
- **WHEN** attempting to add `readme.md` to same tree
- **THEN** return collision error with conflicting path

#### Scenario: Case-insensitive path comparison
- **GIVEN** paths `docs/API.md` and `docs/api.md`
- **WHEN** checking for case collision
- **THEN** detect collision and require user resolution

### Requirement: YAML Configuration Validation
The system SHALL validate repository `.yaml` configuration files against the schema defined in PRD §5.

#### Scenario: Validate valid YAML
- **GIVEN** YAML with valid `repository_id`, `name`, `created_at`, `tags`, `description`
- **WHEN** validating configuration
- **THEN** parsing succeeds with no errors

#### Scenario: Reject invalid ULID in YAML
- **GIVEN** YAML with malformed `repository_id: "invalid-ulid"`
- **WHEN** validating configuration
- **THEN** return validation error for invalid ULID format

#### Scenario: Server-generated defaults
- **GIVEN** YAML without `created_at` field
- **WHEN** server processes configuration
- **THEN** set `created_at` to current ISO 8601 timestamp
