# Project Context

## Purpose
Creacc is a Git‑backed creative collaboration platform that lets creative teams and AI agents work on projects without directly touching Git. It unifies brand copy (Markdown/HTML) and media assets, preserves authoritative Git history, and emits an append‑only catalog for knowledge‑graph ingestion. Initial focus is Spaces → Projects → Check‑Out/Check‑In with leases, conflict resolution, and auditability.

## Tech Stack
- Backend & Infrastructure
  - Core VCS: Git (Smart HTTP); ephemeral worktrees for compute; one commit per check‑in
  - Large files: Git LFS (locking supported); S3‑compatible object storage for binaries
  - Hashing/identity: Dual SHA‑1 and SHA‑256 OIDs; ULIDs for `repository_id`
  - Storage layout: `/srv/git` on POSIX filesystem (ZFS on a cloud block volume or validated network FS)
  - Runtime/deployment: OCI containers on VMs; Git and LFS services co‑located on the same VM by default; single region to start
  - Scaling: Single‑writer per storage shard; horizontal scale via internal sharding within a Space; mirrors for read scaling
  - Catalog: Append‑only JSONL streams (`repos.jsonl`, `projects.jsonl`, `projects/<id>.jsonl`)
  - Observability/ops: Volume snapshots (ZFS or provider), object storage versioning, centralized logs, alerts on capacity/latency
  - Media tooling: ImageMagick/Sharp/ffmpeg; `rclone` for storage sync; Git LFS CLI; checksum/hash validators
- Frontend Stack
  - Language: TypeScript (strict, modern ECMAScript, ESM)
  - Framework: React
  - Runtime: Bun; default runtime for scripts, dev, and tests
  - Package Manager: Bun; commit only its lockfile
  - Build Tool: Vite (ESM dev server with HMR)
  - Test Runner: Vitest (unit tests)
  - Lint/Format: Biome
  - Styling: Tailwind CSS
  - UI Kit: shadcn/ui (Radix primitives + Tailwind)
  - Libraries: TanStack Query/Router/Table as needed; modular components and code organization

## Project Conventions

### Code Style
- Normalize text files with `.editorconfig`; enforce UTF‑8 and LF line endings
  - Prefer explicit formats and formatters per language (e.g., Biome for TS/JS/JSON, gofumpt for Go, shfmt for shell)
  - Biome is the authoritative linter/formatter for TS/JS/JSON; do not add ESLint/Prettier in parallel
  - Keep functions small and pure where practical; prefer composition over inheritance
  - Naming: kebab‑case for directories/files (except language‑specific norms), snake_case for config keys, PascalCase for types
  - 100‑column soft wrap; avoid trailing whitespace; one logical concept per module/file
  - TypeScript: `strict` mode on; ES modules; modern ECMAScript targets; modular component architecture (small, focused components)
  - Frontend: Vite dev server with ESM; TanStack Query for data‑fetching/cache, TanStack Router for routing; colocate component, style, and test files
  - Package manager policy: commit exactly one lockfile — `bun.lockb`; do not commit multiple lockfiles
  - Scripts: prefer portable scripts across JS runtimes; `bunx` may be used for local CLIs when Bun is chosen
  - Tailwind CSS: utility‑first classes; prefer class‑based styling over CSS‑in‑JS; use CSS variables for theme tokens; co‑locate styles with components
  - shadcn/ui: manage components via CLI; keep generated components under `src/components/ui`; adapt tokens via Tailwind config; follow Radix a11y patterns

#### Recommended package.json scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome lint .",
    "format": "biome format --write .",
    "check": "biome check .",
    "ui": "bunx shadcn-ui@latest"
  }
}
```

### Architecture Patterns
- Single‑writer model: one storage shard per writer VM; mirrors for reads
- Git accuracy: canonical object formats; dual SHA‑1/SHA‑256 OIDs recorded and verifiable
- Stateless LFS plane: issues pre‑signed URLs; no mutable state on the LFS service
- Ephemeral worktrees for compute/API tasks; all authoritative history recorded in bare repos
- Append‑only catalog writers with integrity checks; consumers build the knowledge graph from JSONL

### Testing Strategy
- Unit: ULID validation, NFC path normalization, case‑collision detection, LFS policy enforcement, dual‑hash calculation
- Integration: end‑to‑end check‑out/check‑in with leases, auto‑rebase on moved HEAD, conflict resolution (text and LFS pointers), explicit deletions
- Catalog integrity: `release` record followed by exactly `total_files` file records; root tree OIDs match `git rev-parse HEAD^{tree}` (both SHA‑1 and SHA‑256)
- Storage/ops: snapshot/restore smoke tests for `/srv/git`; object storage upload/download, multipart performance, and presigned URL expiry
- Determinism: verify declared derivative commands produce the same outputs bit‑for‑bit or within specified tolerances
 - Frontend: Vitest recommended for unit tests; E2E via Playwright can be evaluated later

### Git Workflow — End‑User Projects
- Branching: trunk‑based (`main`) with server‑managed check‑ins; one commit per check‑in
- Leases: check‑out acquires a lease over a sparse path set; default 72h; admin eviction allowed
- Rebase on check‑in if HEAD moved; resolve conflicts via agent‑assisted UI (text merge + LFS pointer choice: ours/theirs/keep both → auto‑rename)
- LFS policy: auto‑track files >10 MB and known extensions; `.gitattributes` installed/managed by the server
- Deletions: missing files in scope are treated as explicit deletions and require confirmation
- Commit metadata: include repository ULID, lease id, and scope in the message footer

Commit footer template (user check‑ins):
```
Repository-ULID: 01J9Z2Q3W8KFX8Q3R2A5V7YB1M
Lease-ID: <ulid-or-uuid>
Scope: /path/a,/path/b
Hashes: sha1=<oid>, sha256=<oid>
```

### Git Workflow — This Repository (Creacc)
- Branching: trunk‑based (`main`) with short‑lived topic branches; prefer PRs for all changes
- Branch names: `feat/<slug>`, `fix/<slug>`, `docs/<slug>`, `chore/<slug>`, `refactor/<slug>`
- Protected main: require review + passing checks; squash‑merge preferred
- Conventional Commits for messages; keep subjects ≤ 50 chars; wrap body at 72 chars

Commit message template:
```
<type>(<scope>): <short summary>

[Why]
Explain the motivation for this change.

[What]
Summarize the key implementation details.

[Notes]
Edge cases, trade‑offs, or follow‑ups.

Refs: openspec/<change-id>  # optional, e.g., add-check-in-ui
Task: <task-id>             # optional, e.g., 1.2
BREAKING CHANGE: <description>
```

Where `type` ∈ { feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert } and `scope` is optional (e.g., `web`, `api`, `lfs`, `docs`, `openspec`).

## Domain Context
- Repository: canonical Git repository for a block of creative work
- Project: repository + metadata (ULID, tags, manifests, leases) + workflow semantics
- Catalog: collection of projects with indexing and append‑only JSONL for knowledge‑graph ingestion
- Space: tenancy envelope grouping projects and catalogs; internally sharded for scale while appearing unified to users
- Check‑Out/Check‑In: core workflow with leases over sparse paths; agent/human edits; server commits and updates catalog

## Important Constraints
- Unicode normalization: normalize all paths to NFC; reject names that differ only by normalization
- Case collisions: block sibling paths differing only in case; require explicit resolution
- Git accuracy: canonical object formats; maintain dual SHA‑1/SHA‑256 OIDs for all Git objects
- ULID immutability: `repository_id` is stable across renames/moves; validate on ingestion
- LFS policy: auto‑track >10 MB files and specific extensions (see PRD §6 list)
- Explicit deletions: missing files in check‑in scope are flagged and must be confirmed
- Lease lifecycle: 72h duration, 2h pre‑expiry reminder, no grace period; admin eviction allowed
- Catalog integrity: `total_files` in each release record must equal the number of file records appended for that commit

## External Dependencies
- S3‑compatible object storage for LFS binaries (optionally fronted by a CDN such as Cloudflare for downloads)
- Cloud block storage or validated POSIX network filesystem for `/srv/git` (e.g., ZFS on a cloud volume)
- DNS/LB for routing within a Space to the correct single‑writer shard and mirrors
- Build/derive toolchain for media (ImageMagick/Sharp, ffmpeg) and `rclone` for storage syncing
- Observability stack (logs/metrics/alerts) and snapshot tooling for backups/DR
