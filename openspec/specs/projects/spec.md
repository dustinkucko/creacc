# Projects Capability


## Purpose

Project lifecycle management including creation, initialization, metadata, and template application.

## Requirements

This capability manages the complete project lifecycle from creation through deletion. Projects MUST maintain stable ULIDs across all operations and integrate with Git, LFS, and metadata systems.

### Requirement: Project Creation
The system SHALL create projects with auto-generated Git repositories and metadata validation.

#### Scenario: Create project from YAML
- **GIVEN** valid `.yaml` with repository_id, name, tags
- **WHEN** creating project
- **THEN** initialize Git repo, validate YAML, persist project metadata

#### Scenario: Generate ULID if missing
- **GIVEN** YAML without repository_id field
- **WHEN** creating project
- **THEN** generate new ULID and add to metadata

#### Scenario: Set created_at timestamp
- **GIVEN** YAML without created_at field
- **WHEN** server processes creation
- **THEN** set created_at to current ISO 8601 timestamp

### Requirement: Media Upload with LFS
The system SHALL accept media uploads and apply LFS tracking policy for large files and binary types.

#### Scenario: Upload inspiration image
- **GIVEN** user uploads 5MB `inspiration.jpg`
- **WHEN** processing upload
- **THEN** store as LFS object, create pointer in Git

#### Scenario: Upload small text file
- **GIVEN** user uploads 2KB `notes.md`
- **WHEN** processing upload
- **THEN** store directly as Git blob (not LFS)

#### Scenario: Apply size-based LFS threshold
- **GIVEN** unknown file type with 15MB size
- **WHEN** processing upload
- **THEN** automatically track with LFS (>10MB threshold)

### Requirement: Template Application
The system SHALL apply project templates to structure deliverables (e.g., marketing campaign template).

#### Scenario: Apply marketing campaign template
- **GIVEN** project and marketing campaign template ID
- **WHEN** applying template
- **THEN** create directory structure: `/assets/`, `/copy/`, `/email-template.md`

#### Scenario: Template with placeholder files
- **GIVEN** template includes `email-template.md` with placeholders
- **WHEN** applying template
- **THEN** copy template files with placeholders intact for user editing

#### Scenario: Template validation
- **GIVEN** invalid template ID
- **WHEN** attempting to apply template
- **THEN** return error with available template list

### Requirement: Project Metadata Management
The system SHALL manage project metadata including ULID, name, tags, and description with validation.

#### Scenario: Update project name
- **GIVEN** project with name "Old Name"
- **WHEN** updating to "New Name"
- **THEN** update metadata, keep ULID unchanged

#### Scenario: ULID immutability
- **GIVEN** project with ULID `01J9Z2Q3W8KFX8Q3R2A5V7YB1M`
- **WHEN** attempting to modify repository_id
- **THEN** reject update with immutability error

#### Scenario: Add tags to project
- **GIVEN** project with tags `["campaign:Winter25"]`
- **WHEN** adding tag `"client:Creacc"`
- **THEN** update tags array to `["campaign:Winter25", "client:Creacc"]`

### Requirement: Automatic Git Initialization
The system SHALL automatically initialize Git repositories for uploaded projects lacking `.git/` directory.

#### Scenario: Upload directory without Git
- **GIVEN** uploaded folder with files but no `.git/`
- **WHEN** creating project
- **THEN** initialize bare repo, commit files to `main` branch

#### Scenario: Validate existing Git repository
- **GIVEN** uploaded project with `.git/` directory
- **WHEN** creating project
- **THEN** validate Git structure, reject if corrupted

#### Scenario: Install .gitattributes on init
- **GIVEN** newly initialized repository
- **WHEN** creating initial commit
- **THEN** include `.gitattributes` with LFS tracking rules

### Requirement: Committing Workflow Integration
The system SHALL create Git commits for project changes with proper author/committer metadata and LFS integration.

#### Scenario: Commit uploaded files
- **GIVEN** user uploads new files via check-in
- **WHEN** committing changes
- **THEN** create commit with user as author, server as committer, dual-hash all objects

#### Scenario: Commit includes LFS pointers
- **GIVEN** check-in contains large binary file
- **WHEN** creating commit
- **THEN** store LFS pointer in Git tree, upload binary to object storage

#### Scenario: Auto-tag release
- **GIVEN** successful check-in creating commit ABC123
- **WHEN** commit is created
- **THEN** create tag `release-YYYYMMDD-HHMMss-<short_sha>`

### Requirement: Project CRUD Operations
The system SHALL provide create, read, update, delete operations for projects with proper validation.

#### Scenario: List projects in space
- **GIVEN** space with 5 projects
- **WHEN** querying projects
- **THEN** return list with metadata (ID, name, tags, created_at)

#### Scenario: Get project by ID
- **GIVEN** project ID `01J9Z2Q3W8KFX8Q3R2A5V7YB1M`
- **WHEN** fetching project
- **THEN** return full metadata plus latest commit info

#### Scenario: Delete project
- **GIVEN** project to be deleted
- **WHEN** issuing delete request
- **THEN** remove Git repository, metadata, and LFS objects (or mark for cleanup)
