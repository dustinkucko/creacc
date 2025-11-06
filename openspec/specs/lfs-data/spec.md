# LFS Data Capability


## Purpose

Git LFS protocol implementation with object storage backend for large binary files.

## Requirements

This capability implements Git LFS protocol for managing large binary files. All LFS operations MUST conform to the Git LFS specification and integrate seamlessly with S3-compatible object storage.

### Requirement: LFS Pointer Generation
The system SHALL generate Git LFS pointer files containing SHA-256 OID and size for tracked binary files.

#### Scenario: Generate pointer for image file
- **GIVEN** PNG file with 2MB size and SHA-256 hash
- **WHEN** generating LFS pointer
- **THEN** create text file with `version`, `oid sha256:`, and `size` fields

#### Scenario: Pointer replaces blob in Git
- **GIVEN** file tracked by LFS policy
- **WHEN** committing file
- **THEN** store pointer as Git blob, upload binary to object storage

### Requirement: LFS Pointer Parsing
The system SHALL parse LFS pointer files to extract object storage references.

#### Scenario: Parse valid pointer file
- **GIVEN** pointer text with version/oid/size fields
- **WHEN** parsing pointer
- **THEN** extract SHA-256 OID and size as integers

#### Scenario: Reject malformed pointer
- **GIVEN** pointer file with missing `oid sha256:` field
- **WHEN** parsing pointer
- **THEN** return validation error

### Requirement: Pre-Signed Upload URL Generation
The system SHALL issue time-limited pre-signed URLs for uploading LFS objects to S3-compatible storage.

#### Scenario: Generate upload URL for new object
- **GIVEN** repository ID, object OID, size, and MIME type
- **WHEN** requesting upload URL
- **THEN** return pre-signed S3 PUT URL valid for 15 minutes

#### Scenario: URL includes object metadata
- **GIVEN** PNG file upload request
- **WHEN** generating pre-signed URL
- **THEN** URL includes Content-Type and Content-Length headers

#### Scenario: URL expires after time limit
- **GIVEN** upload URL issued at T
- **WHEN** attempting upload at T+20 minutes
- **THEN** S3 rejects request with signature expired error

### Requirement: Pre-Signed Download URL Generation
The system SHALL issue time-limited pre-signed URLs for downloading LFS objects from storage.

#### Scenario: Generate download URL for existing object
- **GIVEN** repository ID and object OID
- **WHEN** requesting download URL
- **THEN** return pre-signed S3 GET URL valid for 60 minutes

#### Scenario: URL routes to correct object
- **GIVEN** LFS object at `s3://bucket/<repo_id>/lfs/objects/<oid>`
- **WHEN** generating download URL
- **THEN** URL points to exact object key

### Requirement: LFS File Locking
The system SHALL provide file locking mechanism for exclusive editing of binary assets (PSD, AI files).

#### Scenario: Create lock on file
- **GIVEN** user checking out `design.psd`
- **WHEN** requesting lock
- **THEN** create lock record with repo ID, file path, user ID, timestamp

#### Scenario: Prevent concurrent edits
- **GIVEN** lock held by User A on `design.psd`
- **WHEN** User B attempts to lock same file
- **THEN** reject with lock conflict error showing current lock owner

#### Scenario: Release lock on check-in
- **GIVEN** file lock held during check-out
- **WHEN** user completes check-in
- **THEN** automatically release file lock

### Requirement: LFS Tracking Policy
The system SHALL automatically track files by extension and size according to `.gitattributes` policy defined in PRD §6.

#### Scenario: Track by extension
- **GIVEN** file `photo.png`
- **WHEN** checking if LFS-tracked
- **THEN** match `.gitattributes` rule `*.png filter=lfs`

#### Scenario: Track by size threshold
- **GIVEN** unknown file type with size 12MB
- **WHEN** checking if LFS-tracked
- **THEN** apply auto-LFS rule for files >10MB

#### Scenario: Generate .gitattributes on repo init
- **GIVEN** new repository initialization
- **WHEN** creating repository
- **THEN** install `.gitattributes` with LFS tracking rules for extensions in PRD §6

### Requirement: Object Storage Integration
The system SHALL store LFS objects in S3-compatible storage with strong consistency and versioning.

#### Scenario: Upload object to S3
- **GIVEN** binary file and pre-signed upload URL
- **WHEN** client uploads via PUT request
- **THEN** object stored at `<repo_id>/lfs/objects/<sha256>` with versioning enabled

#### Scenario: Verify object integrity after upload
- **GIVEN** uploaded LFS object
- **WHEN** downloading object
- **THEN** SHA-256 hash matches pointer OID

#### Scenario: Handle versioned objects
- **GIVEN** LFS object with same OID uploaded twice
- **WHEN** S3 versioning is enabled
- **THEN** maintain all versions with unique version IDs
