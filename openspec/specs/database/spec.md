# Database Capability


## Purpose

Persistent storage for project metadata, lease records, and audit logs with SQLite (dev) and PostgreSQL (production) support.

## Requirements

This capability provides persistent storage with support for both SQLite (development) and PostgreSQL (production). Schema migrations MUST be reversible and maintain data integrity across all operations.

### Requirement: Schema Design
The system SHALL define database schemas for Project, Repository, Lease, AuditEntry, and related tables per PRD data models.

#### Scenario: Project table schema
- **GIVEN** Project data model from PRD
- **WHEN** creating projects table
- **THEN** include columns: project_id (PK), space_id, name, repository_id, created_at, tags (JSONB), description

#### Scenario: Lease table schema
- **GIVEN** Lease data model from PRD
- **WHEN** creating leases table
- **THEN** include columns: lease_id (PK), project_id (FK), user_id, base_commit_sha1, base_commit_sha256, scope_type, scope_paths (JSONB), created_at, expires_at, status

#### Scenario: Dual OID columns
- **GIVEN** any table storing Git commits
- **WHEN** defining schema
- **THEN** include both sha1 (40 chars) and sha256 (64 chars) columns

### Requirement: Migration Management
The system SHALL provide migration files for schema evolution with up/down scripts.

#### Scenario: Create initial migration
- **GIVEN** new database
- **WHEN** running migrations
- **THEN** execute `001_initial_schema.sql` creating all tables

#### Scenario: Apply migration incrementally
- **GIVEN** database at version N
- **WHEN** running migrations
- **THEN** apply only migrations > N in order

#### Scenario: Rollback migration
- **GIVEN** applied migration M
- **WHEN** rolling back
- **THEN** execute down script for M, revert schema changes

### Requirement: SQLite Development Support
The system SHALL support SQLite as lightweight database for local development and testing.

#### Scenario: Initialize SQLite database
- **GIVEN** development environment
- **WHEN** starting application with SQLite config
- **THEN** create database file at configured path

#### Scenario: SQLite JSON support
- **GIVEN** tags stored as JSONB
- **WHEN** using SQLite
- **THEN** store as JSON text, provide query functions for array operations

#### Scenario: In-memory SQLite for tests
- **GIVEN** unit test suite
- **WHEN** running tests
- **THEN** use `:memory:` SQLite database for fast isolated tests

### Requirement: PostgreSQL Production Support
The system SHALL support PostgreSQL 15+ for production deployments with full JSONB and constraint features.

#### Scenario: Connect to PostgreSQL
- **GIVEN** production configuration with Postgres connection string
- **WHEN** starting application
- **THEN** establish connection pool with configured limits

#### Scenario: Use JSONB for structured fields
- **GIVEN** tags and scope_paths columns
- **WHEN** storing data in PostgreSQL
- **THEN** use native JSONB type with indexing support

#### Scenario: Enforce foreign key constraints
- **GIVEN** lease referencing project_id
- **WHEN** inserting lease
- **THEN** PostgreSQL enforces FK constraint, rejects invalid project_id

### Requirement: Data Access Layer
The system SHALL provide DAO (Data Access Object) implementations for CRUD operations on all entities.

#### Scenario: Create project record
- **GIVEN** Project data object
- **WHEN** calling DAO.createProject()
- **THEN** insert record, return created entity with generated ID

#### Scenario: Query leases by status
- **GIVEN** multiple leases with different statuses
- **WHEN** calling DAO.findLeasesByStatus('active')
- **THEN** return only active leases

#### Scenario: Update lease status
- **GIVEN** lease ID and new status
- **WHEN** calling DAO.updateLeaseStatus(id, 'expired')
- **THEN** update lease record, return updated entity

### Requirement: Transaction Support
The system SHALL provide transaction boundaries for operations requiring atomicity (e.g., check-in + lease completion).

#### Scenario: Atomic check-in workflow
- **GIVEN** successful check-in creating commit
- **WHEN** executing transaction
- **THEN** update lease status=completed AND create audit entry atomically

#### Scenario: Rollback on error
- **GIVEN** transaction updating lease and creating audit log
- **WHEN** audit log insert fails
- **THEN** rollback lease update, database remains consistent

#### Scenario: Isolation level configuration
- **GIVEN** concurrent lease operations
- **WHEN** configuring transaction isolation
- **THEN** use READ COMMITTED for PostgreSQL, default for SQLite

### Requirement: Database-Agnostic Abstraction
The system SHALL abstract database-specific features allowing swap between SQLite and PostgreSQL without code changes.

#### Scenario: JSON query abstraction
- **GIVEN** query for leases with specific scope path
- **WHEN** executing query
- **THEN** use appropriate JSON operators for SQLite vs PostgreSQL

#### Scenario: Switch database via configuration
- **GIVEN** application configured for SQLite
- **WHEN** changing config to PostgreSQL
- **THEN** application works without code modification

#### Scenario: Feature detection
- **GIVEN** database-specific features (JSONB indexes)
- **WHEN** initializing DAO
- **THEN** detect database type, enable features only if supported
