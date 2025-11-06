# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Quick Start

**Creacc** is a Git-backed creative collaboration platform that enables creative teams to work with agents on projects without directly touching Git.

**Current Status:** Early design/documentation phase. No code implementation yet.

**Essential Reading:**
- `prd.md` — Product requirements, workflows, user personas, and detailed specifications
- `architecture.md` — Infrastructure design, storage strategy, and deployment model

## Key Concepts

These abstractions are fundamental to the domain model (see `prd.md` §1 for full definitions):

- **Repository** — Git repo containing creative project files (like the content of a book)
- **Project** — Repository + metadata (ULID, tags, manifests, leases) + workflow semantics (like a book with its library card)
- **Catalog** — Collection of projects with indexing for discovery and knowledge graph (like a library catalog)
- **Space** — Tenancy envelope grouping projects and catalogs (like a physical library)
- **Check Out / Check In** — Core workflow with leased file access and conflict resolution (see `prd.md` §13)
- **Lease** — Time-bounded exclusive lock on project scope (72h default, no auto-renew, admin eviction allowed)

## Architecture Quick Reference

For full details, see `architecture.md`. Key patterns:

- **Single-writer model** — One storage shard per writer VM; horizontal scaling via internal sharding within spaces
- **Git + LFS separation** — Git data on `/srv/git` (block storage or POSIX filesystem); LFS on S3-compatible object storage
- **Dual hashing** — All Git objects use both SHA-1 and SHA-256 OIDs (see `prd.md` §11)
- **Catalog format** — Append-only JSONL streams for knowledge graph ingestion (see `prd.md` §14)

## Critical Conventions

These are non-negotiable implementation requirements:

- **Unicode normalization** — All paths normalized to NFC; reject normalization-only variants
- **Case collisions** — Block sibling paths differing only in case; require explicit resolution
- **Git accuracy** — Canonical Git object formats; dual SHA-1/SHA-256 OIDs for all objects
- **ULID immutability** — `repository_id` is stable across renames/moves; validate format on ingestion
- **LFS policy** — Auto-track files >10MB + specific extensions (see `prd.md` §6 for list)
- **Explicit deletions** — Missing files in check-in scope flagged as deletions requiring confirmation
- **Lease lifecycle** — 72h duration, 2h pre-expiry reminder, no grace period
- **Catalog integrity** — `total_files` in release record must match file record count

## Development Principles

- **Simplicity over complexity** — Single-writer models over clustered filesystems
- **Git accuracy** — Canonical object hashes; strong POSIX semantics (atomic ops, locking, fsync)
- **Stateless where possible** — LFS servers are stateless; state lives in object storage or Git repos
- **Explicit over implicit** — Deletions, scope changes, conflicts require user confirmation
- **Auditability** — Log all check-out/check-in operations with who/when/what/resolutions

## Configuration Files

**`.yaml` (Project Configuration)** — See `prd.md` §5 for full spec
```yaml
repository_id: 01J9Z2Q3W8KFX8Q3R2A5V7YB1M  # ULID (immutable)
name: "My Creative Project"                # display name
created_at: "2025-10-14T13:00:00Z"         # set by server if absent
tags: ["campaign:Winter25"]                # optional
description: "Optional one-liner."         # optional
```

**`.gitattributes` (LFS Tracking)** — Server-installed; see `prd.md` §6 for full extension list

## Commands

Once the codebase has code, this section will document:
- How to build and run services locally
- How to run tests (unit, integration, Git object validation)
- How to deploy containers
- How to validate catalog integrity
- How to test check-out/check-in workflows

<!-- OPENSPEC:START -->
# OpenSpec Instructions

These instructions are for AI assistants working in this project.

Always open `@/openspec/AGENTS.md` when the request:
- Mentions planning or proposals (words like proposal, spec, change, plan)
- Introduces new capabilities, breaking changes, architecture shifts, or big performance/security work
- Sounds ambiguous and you need the authoritative spec before coding

Use `@/openspec/AGENTS.md` to learn:
- How to create and apply change proposals
- Spec format and conventions
- Project structure and guidelines

Keep this managed block so 'openspec update' can refresh the instructions.

<!-- OPENSPEC:END -->

## Task Master AI Instructions
**Import Task Master's development workflow commands and guidelines, treat as if import is in the main CLAUDE.md file.**
@./.taskmaster/CLAUDE.md
