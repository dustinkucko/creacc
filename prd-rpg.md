# PRD (Product Requirements Document) ― RPG (Repository Planning Graph)

**Product Name:** Creacc ― Creative Collaboration Platform
**Status:** Draft v1

---

<overview>

## Product Statement

Creatives need to collaborate with agents on projects, current tools are limited in supporting this workflow:

- **Shared Workspace** — agents and humans work together in the same structured environment
- **Version History** — changes to copy and assets lack clear provenance and rollback
- **Context Persistence** — each agent session may start fresh and require memory of project evolution
- **File Organization** — assets scattered across folders with no relationship to current project context

**Core insight:** Git repositories provide perfect version control and context persistence. We will provide users with a simple UX that abstracts away the complexity.

## Target Users

### Creative (Primary User)
- **Goal:** Find inspiration, start a project, collaborate with an agent to iterate on creative work
- **Needs:** Simple upload, clear way to organize, agent can access and edit files
- **Workflow:** Upload inspiration (quote/image) → agent helps develop concept → iterate together
- **Success:** Can create and version a creative project without touching Git

### Developer (Secondary User)
- **Goal:** Extend platform with automation and integrations
- **Needs:** APIs, Git-accurate operations, scriptable workflows
- **Success:** Can clone projects, run scripts, publish outputs via standard Git tools

## Success Metrics

**Workflow Effectiveness:** A user can create, iterate, and version a creative project seamlessly
- **Target:** User uploads inspiration → agent helps expand → multiple iterations → versioned history
- **Measurement:** Successful start with using an image to get a finished project with a marketing email

**Git Accuracy:** 100% correct Git operations
- **Target:** All Git objects validate with `git fsck`; dual SHA-1/SHA-256 hashing works
- **Measurement:** Automated validation suite

</overview>

---

<functional-decomposition>

## Capability Tree

1. **Project Development** - Full lifecycle management of creative projects
  - **Initialization:** Create projects with media, notes, and templates
    - Upload inspiration media: User uploads images, videos, or other files that sparked the creative idea
    - Add project note/prompt: User describes their vision or idea in a single note
    - Apply project template: Optionally structure project using a template (e.g., marketing campaign, game asset)
    - Auto-generate Git repository: System creates versioned workspace with metadata files automatically
  - **Lease Management:** Handle check-out/check-in with exclusive access
  - **Conflict Resolution:** Manage stale bases and merge conflicts during check-in

2. **Version History** - Automatic Git versioning of text, metadata, and binaries
  - Store text content: Version notes, prompts, copy with readable diffs
  - Store metadata: Version .yaml config and .md notes that provide agent context
  - Store large binaries: Track images, videos, design files efficiently without bloating repo
  - Generate commit history: Provide automatic versioning when changes occur
  - Restore previous versions: Enable rollback and exploration of project history

3. **Programmatic Access** - Enable agents and developers to interact with projects
  - API access to files: Read/write project files and folders via HTTP API
  - Access project metadata: Read/write notes, config, tags programmatically
  - Commit changes with attribution: Agents and scripts can make versioned changes with clear authorship
  - Git protocol support: Standard clone, pull, push for developer workflows
  - Structured context for agents: Metadata files (.yaml, .md) are easily parseable for agent understanding

4. **Standards & Validation** - Ensure all operations are reproducible and cross-platform compatible
  - **Identity:** Stable ULID/UUID for projects across renames/moves
  - **Validation:** dual OID correctness, Git object format, verify project metadata and assets
  - Cross-platform compatibility: Handle path/case/Unicode normalization for macOS, Windows, Linux
  - Templates: Define recipes for deliverables (e.g., marketing campaign = image + email copy)
  - Automated checks: Git hooks or similar ensure project completeness before commit
  - Project comparison: Enable combining and comparing projects at metadata level

5. **Platform Operations** - Ensure projects are durable, backed up, and shareable across spaces
  - Internal mirroring: Sync projects between spaces so that each has its own (prevent issue with access loss)
  - Automated backups: Regular snapshots and disaster recovery
  - Export capabilities: Hand off projects to external teams or systems when ready
  - Platform health monitoring and integrity validation


</functional-decomposition>

---

<structural-decomposition>

## Module Structure

```
creacc/
├── src/
│   ├── projects/            # Project lifecycle (Project Development)
│   ├── git-data/            # Git data operations (Version History)
│   ├── lfs-data/            # LFS data operations (Version History)
│   ├── lease-management/    # Lease Management (Project Development)
│   ├── conflict-resolution/ # Conflict Resolution (Project Development)
│   ├── identity/            # Identity (Standards & Validation)
│   ├── validation/          # Validation (Standards & Validation)
│   ├── api/                 # Programmatic Access
│   ├── storage/             # Storage abstraction layers
│   └── shared/              # Shared utilities, types, config, logging (Platform Operations)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
└── scripts/                 # Deployment, migrations, maintenance
```

## Module Definitions

### Module: projects
- **Maps to capability:** Project Development
- **Responsibility:** Project lifecycle management - creation, initialization, metadata, templates
- **File structure:**
  ```
  projects/
  ├── create.ts             # Project creation and initialization
  ├── metadata.ts           # Project metadata management (ULID, tags, etc.)
  ├── templates.ts          # Template application and management
  ├── lifecycle.ts          # Project CRUD operations
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `createProject(spaceId, name, options)` → Project
  - `uploadMedia(projectId, files)` → MediaRefs
  - `applyTemplate(projectId, templateId)` → ProjectStructure
  - `updateMetadata(projectId, metadata)` → Project

### Module: git-data
- **Maps to capability:** Version History
- **Responsibility:** Bare Git repository management with Git-accurate object creation
- **File structure:**
  ```
  git-data/
  ├── repository.ts         # Repository initialization and lifecycle
  ├── objects.ts            # Blob/tree/commit storage with dual hashing
  ├── refs.ts               # Reference and reflog management
  ├── worktree.ts           # Ephemeral worktree creation
  ├── validation.ts         # Git object format validation
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `initRepository(spaceId, repoName, repoConfig)` → Repository
  - `storeObject(type, content)` → { sha1, sha256 }
  - `updateRef(refName, oldOid, newOid, committer)` → RefUpdate
  - `createWorktree(repoPath, baseCommit, scope)` → WorktreePath

### Module: lfs-data
- **Maps to capability:** Version History
- **Responsibility:** Git LFS protocol implementation with object storage backend
- **File structure:**
  ```
  lfs-data/
  ├── pointer.ts            # LFS pointer generation and parsing
  ├── upload.ts             # Pre-signed upload URL generation
  ├── download.ts           # Pre-signed download URL generation
  ├── locking.ts            # LFS file locking
  ├── policy.ts             # LFS tracking rules (.gitattributes)
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `generatePointer(filePath, bytes, sha256, size)` → PointerContent
  - `issueUploadUrl(repoId, oid, size, mime)` → { url, expiresAt }
  - `issueDownloadUrl(repoId, oid)` → { url, expiresAt }
  - `createLock(repoId, filePath, userId)` → LockId

### Module: lease-management
- **Maps to capability:** Platform Operations: Lease Management
- **Responsibility:** Time-bounded exclusive locks for check-out/check-in workflows
- **File structure:**
  ```
  lease-management/
  ├── lease.ts              # Lease creation and validation
  ├── expiration.ts         # Expiration logic and reminders
  ├── eviction.ts           # Admin eviction controls
  ├── scope.ts              # Scope overlap detection
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `createLease(repoId, userId, baseCommit, scope, duration)` → Lease
  - `validateLease(leaseId, userId)` → { valid, remainingTime, scope }
  - `evictLease(leaseId, adminId, reason)` → EvictionResult
  - `checkExpiration()` → ExpiredLeaseIds[]

### Module: conflict-resolution
- **Maps to capability:** Platform Operations: Conflict Resolution
- **Responsibility:** Detect and resolve conflicts during check-in
- **File structure:**
  ```
  conflict-resolution/
  ├── detection.ts          # Stale base detection
  ├── rebase.ts             # Auto-rebase strategy
  ├── lfs-conflicts.ts      # LFS pointer conflict handling
  ├── deletions.ts          # Deletion confirmation logic
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `detectStaleBase(baseCommit, currentHead)` → { conflicted, divergence }
  - `autoRebase(baseCommit, changes, currentHead)` → { commit, conflicts }
  - `resolveLfsConflict(filePath, ours, theirs, choice)` → Resolution
  - `confirmDeletions(scope, uploaded, base)` → { confirmed, pending }

### Module: identity
- **Maps to capability:** Standards & Validation
- **Responsibility:** Git object integrity, path normalization, immutable identifiers
- **File structure:**
  ```
  identity/
  ├── hashing.ts            # Dual SHA-1/SHA-256 OID computation
  ├── ulid.ts               # ULID/UUID generation and validation
  ├── normalization.ts      # Unicode NFC path normalization
  ├── case-collision.ts     # Case-insensitive sibling detection
  ├── yaml-validation.ts    # Repo YAML validation
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `computeDualOid(type, size, content)` → { sha1, sha256 }
  - `generateUlid(timestamp?)` → UlidString
  - `normalizePath(path)` → { normalized, changed }
  - `detectCaseCollision(tree, newPath)` → { collided, conflictingPath }
  - `validateYaml(yamlContent)` → { valid, errors, parsed }

### Module: validation
- **Maps to capability:** Standards & Validation
- **Responsibility:** Cross-platform compatibility, template schemas, project validation

### Module: api
- **Maps to capability:** Programmatic Access
- **Responsibility:** HTTP/S endpoints for all user-facing operations
- **File structure:**
  ```
  api/
  ├── routes/
  │   ├── projects.ts       # Project CRUD
  │   ├── leases.ts         # Lease check-out/check-in
  │   └── health.ts         # Health checks
  ├── middleware/
  │   ├── auth.ts           # Authentication and authorization
  │   ├── validation.ts     # Request validation
  │   └── error.ts          # Error handling
  ├── server.ts             # HTTP server setup
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `startServer(port, config)` → HttpServer
  - `stopServer(server)` → Promise<void>

### Module: storage
- **Maps to:** Project Management (capability), primary storage abstraction (infrastructure)
- **Responsibility:** Abstract storage layer for Git data (block volume, network FS, local)
- **File structure:**
  ```
  storage/
  ├── interface.ts          # Storage interface definition
  ├── block-volume.ts       # Cloud block volume implementation (ZFS)
  ├── network-fs.ts         # Network filesystem implementation (TigrisFS)
  ├── local.ts              # Local filesystem (dev/test)
  └── index.ts              # Public exports
  ```
- **Exports:**
  - `StorageAdapter` interface
  - `createStorageAdapter(config)` → StorageAdapter

### Module: shared
- **Maps to:** Cross-cutting utilities (not a capability)
- **Responsibility:** Shared types, configuration, logging, error handling
- **File structure:**
  ```
  shared/
  ├── types.ts              # Common TypeScript types
  ├── config.ts             # Configuration loading
  ├── logger.ts             # Structured logging
  ├── errors.ts             # Custom error classes
  └── index.ts              # Public exports
  ```
- **Exports:**
  - Common types (Repository, Lease, Commit, etc.)
  - `loadConfig()` → Config
  - `logger` → Logger instance
  - Error classes (ValidationError, ConflictError, etc.)

</structural-decomposition>

---

<dependency-graph>

## Dependency Chain

### Foundation Layer (no dependencies)
- **shared**: provides common types, config, logging, error handling for all modules
- **identity**: provides ULID/UUID generation, hashing, normalization used across all modules
- **validation**: provides request validation and error handling utilities
- **storage**: abstracts primary storage layer; other modules use this interface

### Data Layer
- **git-data**: depends on [shared, identity, validation, storage]
  - Uses shared types and errors
  - Uses identity for dual OID computation and path normalization
  - Uses validation for dual OID validation
  - Uses storage adapter for reading/writing Git data
- **lfs-data**: depends on [shared, identity, validation, storage]
  - Uses shared types and config for object storage credentials
  - Uses identity for dual OID computation
  - Uses validation for dual OID validation
  - Independent of git-data

### Logic Layer
- **lease-management**: depends on [shared, identity, validation, git-data]
  - Uses shared types (Lease, User, Repo)
  - Uses identity for ULID/UUID generation (lease IDs)
  - Uses validation for lease scope and expiration checks
  - Uses git-data to validate base commit exists
- **conflict-resolution**: depends on [shared, identity, validation, git-data]
  - Uses shared types (Commit, Conflict)
  - Uses identity for path normalization during conflict detection
  - Uses validation for conflict resolution strategies
  - Uses git-data for worktree operations and rebase

### Presentation Layer
- **api**: depends on [shared, git-data, lfs-data, lease-management, conflict-resolution]
  - Uses all core modules to implement HTTP endpoints
  - Orchestrates workflows (check-out/check-in) by calling multiple modules
  - Uses shared for error handling and logging

</dependency-graph>

---

<implementation-roadmap>

## Development Phases

### Phase 1: Foundation
- Shared utilities, identity/validation, storage abstraction
- Git operations with dual OID support
- LFS storage with object routing
- Cross-platform compatibility

### Phase 2: Core Features
- Project lifecycle (create, upload, commit)
- API endpoints (HTTP + Git protocol)
- Basic version history operations
- Lease management (check-out/check-in)
- Template system
- Internal mirroring and backups

</implementation-roadmap>

---

<test-strategy>

## Test Strategy

- **Unit tests**: Pure functions, validation logic, isolated modules (60%)
- **Integration tests**: Git operations against native Git CLI, module interactions (30%)
- **E2E tests**: Full user workflows via API (10%)
- **Coverage target**: 80% minimum, 95% on critical paths (Git operations, version history)

</test-strategy>

---

<architecture>

## System Components

### Git Service Container
- Runs bare Git server exposing Smart HTTP protocol
- Mounts primary storage at `/srv/git`
- Handles all repository I/O locally (single-writer model)
- Co-located with LFS container on same VM by default

### LFS Service Container
- Stateless LFS server issuing pre-signed URLs
- Connects to S3-compatible object storage
- Co-located with Git container on same VM
- Optionally fronted by CDN for global caching

### Primary Storage
- **Option A:** Validated POSIX network filesystem (e.g., TigrisFS) with single-writer semantics
- **Option B:** Cloud block volume (e.g., DigitalOcean Volumes) is attached and then mounted with ZFS
- Single-attach model: one writer VM per storage shard
- Failover via detach/attach to standby VM

### Object Storage
- S3-compatible storage (Tigris, AWS S3, Cloudflare R2, DigitalOcean Spaces)
- Stores all LFS objects with versioning enabled
- Strong consistency for new objects

### Database
- Stores project metadata, lease records, audit logs
- Options: Supabase, PostgreSQL (recommended), SQLite (dev/test)
- Single source of truth for non-Git data

## Data Models

### Project & Repository Relationship
- Projects retain workflow, while their canonical Git repositories remain the source of truth for all versioned state.
- A shared ULID ties each project to exactly one repository record so renames or mirror promotion never break identity.
- Higher-level constructs (leases) resolve through the project layer for the corresponding repository.

### Project Metadata Record
```typescript
interface Project {
  project_id: string;       // ULID/UUID (primary key, same as repository_id)
  space_id: string;         // ULID/UUID
  name: string;             // user-facing display name
  repository_id: string;    // ULID/UUID (points to canonical repository, same as project_id)
  created_at: string;       // ISO 8601
  tags: string[];
  description?: string;
  // Note: project_id === repository_id (same value)
}
```

### Repository Record
```typescript
interface Repository {
  repository_id: string;    // ULID/UUID (same as project_id)
  space_id: string;         // ULID/UUID
  storage_path: string;     // repository storage location: /srv/git/<space_id>/<repo_id>
  default_branch: string;   // e.g., "main"
  head_commit_sha1: string;
  head_commit_sha256: string;
  created_at: string;       // ISO 8601
}
```

### Mirror Relationship
```typescript
interface MirrorRelationship {
  mirror_id: string;              // ULID/UUID
  source_repo_id: string;         // source repository (ULID/UUID)
  mirror_repo_id: string;         // mirror repository (ULID/UUID)
  source_space_id: string;        // space of source repository
  mirror_space_id: string;        // space of mirror repository
  sync_status: 'active' | 'paused' | 'failed';
  last_synced_at?: string;        // ISO 8601
  created_at: string;             // ISO 8601
}
```

### Repository Relationship
```typescript
interface RepositoryRelationship {
  relationship_id: string;           // ULID/UUID
  source_repo_id: string;            // ULID/UUID (from repository)
  target_repo_id: string;            // ULID/UUID (to repository)
  relationship_type: 'submodule' | 'shared-assets' | 'related-campaign' | 'inspiration' | 'fork' | 'variant';
  metadata?: {
    description?: string;
    tags?: string[];
    submodule_path?: string;         // For submodule relationships: path in .gitmodules
    pinned_commit_sha1?: string;     // For submodule relationships: pinned commit
    pinned_commit_sha256?: string;   // For submodule relationships: pinned commit
    created_by?: string;
  };
  created_at: string;                // ISO 8601
}
```

### Submodule Compatibility
- Git submodules supported by treating each linked repository as its own `Repository` (and optional `Project`) record.
- System validates `.gitmodules` during check-in to ensure every referenced repository ULID exists and its pinned commit hashes (SHA-1/SHA-256) remain consistent.
  - System rejects check-in updates when `.gitmodules` references an unknown repository ULID or the pinned SHA-1/SHA-256 pair drift from the recorded commit
- Submodule relationships automatically captured in `RepositoryRelationship` table with `relationship_type: 'submodule'` once `.gitmodules` validation passes.
- No additional workflow is introduced for submodules; existing lease and catalog flows simply operate per repository.

### Lease Record
```typescript
interface Lease {
  lease_id: string;           // ULID/UUID
  project_id: string;         // ULID/UUID
  user_id: string;
  base_commit_sha1: string;
  base_commit_sha256: string;
  scope_type: 'full' | 'sparse';
  scope_paths: string[];      // Empty for full scope
  created_at: string;         // ISO 8601
  expires_at: string;         // ISO 8601
  status: 'active' | 'expired' | 'evicted' | 'completed';
}
```

### Audit Log
```typescript
interface AuditEntry {
  entry_id: string;           // ULID/UUID
  lease_id: string;
  user_id: string;
  action: 'check-out' | 'check-in' | 'eviction';
  timestamp: string;          // ISO 8601
  base_commit?: string;
  new_commit?: string;
  scope: { type: string; paths: string[] };
  changes: {
    adds: string[];
    mods: string[];
    renames: { from: string; to: string }[];
    deletes: string[];
  };
  conflict_resolutions?: {
    file: string;
    strategy: 'ours' | 'theirs' | 'keep-both';
  }[];
}
```

### Git Object Metadata (Dual OID Index)
```typescript
interface ObjectMetadata {
  sha1: string;       // 40 hex chars
  sha256: string;     // 64 hex chars
  type: 'blob' | 'tree' | 'commit' | 'tag';
  size: number;       // bytes
  stored_at: string;  // ISO 8601
}
```

## Technology Stack

### Backend
- **Language:** TypeScript with Bun runtime
- **Runtime:** Bun with native Git bindings (or isomorphic-git)
- **Framework:** TanStack Start with Nitro + H3 + Vite — lightweight HTTP server
- **Database:** PostgreSQL 15+ (production), SQLite (dev/test)
- **Object Storage:** S3-compatible (Tigris or DigitalOcean Spaces)
- **Primary Storage:** TigrisFS (validated POSIX network filesystem) or ZFS on cloud block volume (DigitalOcean Volumes)

### Infrastructure
- **Containers:** OCI containers (Podman)
- **Orchestration:** Fly.io microVMs or lightweight Kubernetes (K3s)
- **CDN:** Cloudflare (optional, for LFS downloads)
- **Monitoring:** Prometheus + Grafana, structured logging (Pino or Winston)

### Development
- **Build:** TypeScript compiler, esbuild (bundling)
- **Testing:** Vitest (unit/integration), Playwright (E2E API tests)
- **Linting:** Biome
- **Git Operations:** native Git or isomorphic-git (pure JS)
- **Hashing:** Node.js crypto module (SHA-1, SHA-256)
- **ULID/UUID:** `ulid` or `uuid` package

## Key Design Decisions

### Decision: TypeScript + Bun
- Fast development velocity for web APIs and Git operations
- Rich ecosystem and type safety for complex data models
- I/O-bound workload suits Bun single-threaded model
  - Git I/O is typically storage-bound; Bun single-threaded model acceptable for I/O-heavy workload

### Decision: Single-writer storage model
- One VM per storage shard (simplest operational model)
- TigrisFS (network filesystem) or cloud block volumes with strong consistency
- Avoids clustered filesystem complexity

### Decision: LFS server for file routing/delivery
- LFS servers route users to correct files based on their requests
- Can be stateless, proxying to object storage
- Handles authentication and access control to ensure users get appropriate file versions

### Decision: Dual SHA-1 + SHA-256 OIDs
- Git ecosystem is migrating to SHA-256 (Git 2.42+ supports it)
- Storing both hashes now future-proofs the platform
- Enables interop with both SHA-1 and SHA-256 Git clients

### Decision: Leases for exclusive access
- Leases provide exclusive access to edit assets
- When a lease expires, exclusivity is lost and normal Git workflow resumes (multiple users can check in)

</architecture>

---

<risks>

## Key Risks

**Technical**
- Git object hashing accuracy (mitigation: use native Git CLI, extensive testing against canonical behavior)
- Storage performance (mitigation: early benchmarks, simple single-writer architecture)

**Implementation**
- Scope creep and over-complexity (mitigation: focus on Phase 1 simplicity, defer Phase 2 features)

</risks>

---

<appendix>

## References

### Papers and Specifications
- **Git Internals:** https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain
- **Git LFS Specification:** https://github.com/git-lfs/git-lfs/blob/main/docs/spec.md
- **SHA-256 Transition in Git:** https://git-scm.com/docs/hash-function-transition
- **ULID Specification:** https://github.com/ulid/spec
- **Unicode Normalization (UAX #15):** https://unicode.org/reports/tr15/

### Tools and Libraries
- **isomorphic-git:** https://github.com/isomorphic-git/isomorphic-git (pure JS Git)
- **Vitest:** https://vitest.dev/ (test framework)

### Comparable Systems
- **GitHub:** Git hosting with web UI and conflict resolution
- **GitLab:** Self-hosted Git with LFS support
- **Perforce Helix Core:** Version control for large binary assets (creative industry standard)

## Glossary

- **ULID:** Universally Unique Lexicographically Sortable Identifier (26 chars, time-sortable)
- **UUID v7:** Universally Unique Identifier version 7 (RFC 4122, time-based)
- **OID:** Object Identifier (SHA-1 or SHA-256 hash of Git object)
- **Dual OID:** Both SHA-1 and SHA-256 hashes for same Git object
- **LFS pointer:** Small text file containing SHA-256 OID and size of large binary
- **NFC:** Unicode Normalization Form C (canonical composition)
- **Check-out:** Create lease and download project files
- **Check-in:** Upload changes, resolve conflicts, create commit, close lease
- **Lease:** Time-bounded exclusive lock on project scope for check-out/check-in
- **Sparse scope:** Lease covering only specific file paths (vs. full project)
- **Auto-rebase:** Automatically rebase check-in changes onto current HEAD

## Open Questions

### Phase 1 Scope
- Should Phase 1 include sparse checkout, or defer to Phase 2? **Decision: Defer to Phase 2 (user answered)**
- Should Phase 1 include auto-rebase, or require clean base? **Decision: Include auto-rebase (user wants workflow efficiency)**
- Should Phase 1 include catalog/JSONL generation? **Decision: Defer to Phase 2 (user answered)**

### Technical Decisions
- Git library for commands: isomorphic-git (pure JS, portable) over NodeGit (native?, faster)
  - Do we need a library?
- Cloud provider for database: Supabase (managed) over self-hosted PostgreSQL (or SQLite)
  - Do we need a database?
- Cloud provider for initial deployment: DigitalOcean (simpler) over AWS (more features)
  - Do we need a traditional cloud provider?

### UX/Design
- What should conflict resolution UI look like for LFS pointer conflicts?
- Should we show Git commit history to users, or abstract it away?
- How should we handle lease expiration UX (email, in-app notification, both)?

</appendix>

---

<task-master-integration>

# How Task Master Uses This PRD

When you run `task-master parse-prd prd-rpg.md`, the parser:

1. **Extracts capabilities** → Main tasks
   - Each `### Capability:` becomes a top-level task
   - Example: "Git Core Operations" → Task 1

2. **Extracts features** → Subtasks
   - Each `#### Feature:` becomes a subtask under its capability
   - Example: "Repository initialization" → Task 1.1

3. **Parses dependencies** → Task dependencies
   - Dependencies in the dependency graph section map to task.dependencies
   - Example: "git-core: Depends on [shared, identity, storage]" means Task "git-core" depends on Tasks "shared", "identity", "storage"

4. **Orders by phases** → Task priorities
   - Phase 0 tasks = highest priority (foundation layer)
   - Phase 1 tasks = medium priority (data layer)
   - Phase 2+ tasks = lower priority, properly sequenced

5. **Uses test strategy** → Test generation context
   - Feeds critical test scenarios to Surgical Test Generator during implementation
   - Coverage requirements inform test completeness checks

**Result:** A dependency-aware task graph that can be executed in topological order.

## Why RPG Structure Matters

Traditional flat PRDs lead to:
- ❌ Unclear task dependencies
- ❌ Arbitrary task ordering
- ❌ Circular dependencies discovered late
- ❌ Poorly scoped tasks

RPG-structured PRDs provide:
- ✅ Explicit dependency chains
- ✅ Topological execution order
- ✅ Clear module boundaries
- ✅ Validated task graph before implementation

## Tips for Best Results

1. **Spend time on dependency graph** - This is the most valuable section for Task Master parsing
2. **Keep features atomic** - Each feature should be independently testable
3. **Progressive refinement** - After parsing, use `task-master expand` to break down complex tasks
4. **Use research mode** - `task-master parse-prd --research` leverages AI for better task generation

</task-master-integration>
