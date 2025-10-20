# PRD — Creative Library (Git‑based Knowledge Graph)

**Doc status:** Draft v1  
**Owners:** Dustin Kucko (PM), Engineering (Agents)
**Last updated:** 2025‑10‑14

---

## 1) Problem & Vision
Creative teams need a simple, server‑centric way to catalog project files, track editions over time, and build a knowledge graph—without requiring users to know Git. The app hosts repos, enforces a clean check‑out/check‑in workflow, and emits a query‑friendly catalog that mirrors Git object identity, enabling reliable cross‑repo reference.

**Vision:** A UI‑first library where projects feel like folders and “editions” feel like snapshots; Git runs behind the scenes; the catalog serves as the API for discovery, provenance, and graph building.

---

## 2) Goals / Non‑Goals
**Goals**
- Treat each creative project as a first‑class entity, Git or not.
- Auto‑initialize non‑Git projects into repos; server manages all Git ops.
- Support check‑out/check‑in with exclusive leases and sparse paths (explicit selection in UI).
- Store Git‑accurate SHA‑1 and SHA‑256 OIDs for blobs/trees/commits; editions map 1:1 to commits.
- Use Git LFS for binaries/large files; keep text/code in normal Git for readable diffs.
- Generate a compact, append‑friendly catalog suitable for a knowledge graph.

**Non‑Goals (v1)**
- Branching workflows and multi‑branch history visualization.
- Per‑directory tree OID storage (derivable later).
- Complex role/ACL models beyond admin eviction & basic project ownership.
- Real‑time collaborative editing.

---

## 3) Primary Users & Personas
- **Creative pro:** Non‑technical; uploads/downloads folders or files; expects simple conflict help.
- **Power user/Engineer:** May inspect OIDs, mirrors, and use API; expects source‑of‑truth integrity.
- **Librarian/Admin:** Manages projects, leases, and evictions; audits history; curates catalog.

---

## 4) High‑Level Architecture (v1)
- **Server‑centric Git host** (bare repos for storage, working trees for compute as needed).
- **Web app** for check‑out/check‑in, sparse path selection, conflict resolution, and audit.
- **Catalog emitter** writing JSONL shards per project (edition metadata + file manifests combined).
- **Knowledge graph** consumers ingest catalog; projects are nodes; editions/files become nodes/edges.

---

## 5) Project Model & Discovery
- **Location:** Projects reside under `/projects/<project>/` (top‑level structure may evolve; consider future `library.yaml` for multi‑library spaces).
- **Marker:** A project is any directory under `/projects/` containing a `project.yaml`.
- **Immutable ID:** `project_id` is a **ULID** stored in `project.yaml` (sortable by time; stable across renames/moves).
- **User‑provided `project.yaml`:** Accepted if valid; server generates if missing on first upload.

`project.yaml` (minimal v1)
```yaml
project_id: 01J9Z2Q3W8KFX8Q3R2A5V7YB1M  # ULID (immutable)
name: "My Creative Project"            # display name (optional)
created_at: "2025-10-14T13:00:00Z"     # set by server if absent
tags: ["campaign:FW25", "client:Acme"] # optional
description: "Optional one-liner."     # optional
```

---

## 6) Git & LFS Behavior
- **Auto‑init:** If a project lacks `.git/`, server initializes a worktree repo (default branch `main`).
- **Server authority:** Client `.git/` ignored; server’s history is canonical.
- **Default branch scope:** Indexing & editions based on **default branch HEAD** only.
- **LFS policy:**
  - Track binaries/large types via `.gitattributes` (images/video/audio/archives/3D).
  - **Size‑based auto‑LFS:** enabled at **10 MB** for unknown types.
  - Text/code remain normal Git for diffs.
- **.gitattributes:** Installed by server in each project; ensures correct behavior in downstream clones.

**Extensions tracked by LFS (v1):** `png, jpg, jpeg, webp, gif, tiff, psd, ai, svgz, mp4, mov, mkv, wav, aiff, flac, mp3, ogg, zip, 7z, rar, tar, gz, bz2, blend, fbx, obj, glb, usdz`.

---

## 7) Hashing & Identity (Git‑accurate)
- **Dual OIDs:** Store both **SHA‑1** and **SHA‑256** OIDs for blobs, trees, and commits.
- **Canonical bytes:** Use Git object formats: `"blob <size>\0<content>"`, `"tree <size>\0<entries>"`, `"commit <size>\0<content>"`.
- **Trees:** Correct modes (`100644/100755/120000/040000`), bytewise sort, raw child IDs (20B for SHA‑1 tree; 32B for SHA‑256 tree).
- **Root tree:** Persist `root_tree_sha1` and `root_tree_sha256` in each edition for fast integrity checks.
- **Case & Unicode:** Reject sibling case collisions; **normalize paths to NFC** for identity and comparisons.

---

## 8) Remotes & Mirrors
- **Canonical remote:** `origin` (HTTPS) recorded as canonical.
- **Mirrors:** Additional remotes (e.g., GitHub) recorded in the same `repos.jsonl` stream with `is_canonical=false` and `order_index ≥ 1`.
- **Fields:** `remote_name, remote_url, remote_host, remote_path, is_canonical, order_index, default_branch, head_commit_sha1/sha256, uses_lfs, repo_type`.

---

## 9) Check‑Out / Check‑In Workflow
**Leases**
- **Default duration:** 72h (customizable per checkout). No auto‑renew.
- **Reminder:** Notify user **2h before expiry**. No grace period.
- **Admin eviction:** Allowed.

**Check‑out**
- Scope options: **Full project** or **Sparse** with **explicit paths only** (UI selection).
- Server issues a lease: `{project_id, base_commit, scope_type, pathset[]}` and a download.

**Check‑in**
- User uploads full project, sparse subset, or individual files.
- Within **scope**, upload is an **authoritative snapshot**: adds/changes applied; missing entries flagged as **deletions** for explicit confirmation.
- Outside scope: untouched.
- If HEAD moved: **auto‑rebase** on current HEAD; conflicts handled in an **agent‑assisted UI** (LFS pointer choice: ours/theirs/keep both → auto‑rename).
- On success: commit created, HEAD advanced, lease closed, **auto‑tag** `edition-<YYYYMMDD>-<HHMMss>-<shortsha>`.

**Audit**
- Persist `{who, when, base_commit, new_commit, scope, adds, mods, renames, deletes, resolutions}`.

---

## 10) Catalog Layout (v1)
**Goal:** Append‑friendly, per‑project shards that combine edition metadata + file manifests.

- **Repos stream:** `catalog/repos.jsonl` (one object per remote; `origin` first).
- **Projects directory:** `catalog/projects.jsonl` (light directory: `project_id, name, created_at, tags`).
- **Per‑project shard:** `catalog/projects/<project_id>.jsonl` — **mixed record types**:
  - `{"type":"edition", ...}` one row per commit (includes root tree OIDs, counts, scope info).
  - `{"type":"file", ...}` one row per file in that edition (blob OIDs, mode, size, mime, LFS flag).
- **Rollover:** Start `.../<project_id>.part2.jsonl` when a shard exceeds ~**256 MB**.
- **Integrity:** `total_files` in the edition must match the number of following `file` rows for that commit.

---

## 11) API Surface (initial)
**Projects**
- `POST /projects` — create from `project.yaml` or server‑generated; returns `project_id`.
- `GET /projects/:id` — metadata + latest edition.

**Checkout**
- `POST /projects/:id/checkout` — body `{scope: {type: "full"|"sparse", paths:[]}, duration_hours?}` → returns lease + download token.
- `GET /leases/:lease_id` — lease status.

**Checkin**
- `POST /leases/:lease_id/checkin` — multipart (files or zip) + JSON ops (confirm deletions/renames). Returns new edition metadata.

**Catalog**
- `GET /catalog/repos.jsonl` — stream.
- `GET /catalog/projects.jsonl` — stream.
- `GET /catalog/projects/:id.jsonl` — stream (supports HTTP range for tailing).

---

## 12) Security, Privacy, Compliance
- Ignore client `.git/` to prevent history injection.
- Validate ULID, NFC path normalization, and case‑collision rules on ingestion.
- Size/type enforcement for LFS on server.
- Admin eviction for stuck leases; all actions audited.

---

## 13) Telemetry & Observability
- Metrics: check‑out counts, average lease length, check‑in success rate, conflict rate, average catalog append latency, LFS bytes stored.
- Logs: per‑check‑in audit and conflict resolutions.

---

## 14) Edge Cases & Handling
- **Case collisions:** Block; require user resolution (choose/rename/keep both).
- **Unicode variants:** Normalize to NFC; reject names differing only by normalization.
- **Sparse uploads adding out‑of‑scope paths:** Warn; require scope extension.
- **Deletes:** Never silent; require explicit confirmation.
- **Multiple remotes:** `origin` canonical; mirrors appended with ordering.

---

## 15) Open Questions / Future Work
- **Multi‑library spaces:** Introduce top‑level `library.yaml`, namespace isolation.
- **Per‑directory tree OIDs:** Store `edition_trees/*.jsonl` for faster subtree diffing.
- **Submodule‑backed catalog sharding:** Convert `catalog/projects/<id>` into submodules for scale/ACLs.
- **Advanced conflict tooling:** Visual diff previews for common creative formats; binary diff heuristics.
- **Background validation:** Periodic rehash to detect bit rot; verify root tree vs HEAD^{tree}.
- **Richer metadata:** Optional sidecar ingestion (`*.meta.json`, EXIF → normalized fields).

---

## 16) Milestones
**M1 — Foundations (2–3 sprints)**
- Project detection via `project.yaml` (ULID validation + server generation).
- Auto‑init Git repo; install `.gitattributes` per policy.
- Basic checkout/lease; full‑project scope only.
- Check‑in with auto‑rebase; agent UI for conflicts (text + LFS pointer choice).
- Catalog writers: `repos.jsonl`, `projects.jsonl`, `projects/<id>.jsonl` (edition + files).
- Auto‑tag editions.

**M2 — Sparse & Polish (2 sprints)**
- Sparse path selection (explicit paths), sparse check‑in with deletion confirmations.
- Case/Unicode enforcement; admin eviction; 2h pre‑expiry reminders.
- Catalog rollover; integrity verifier.

**M3 — Mirrors & API hardening (2 sprints)**
- Mirror recording; refined repos stream; pagination/range streaming endpoints.
- Basic analytics (metrics + audit export).

---

## 17) Acceptance Criteria (samples)
- Uploading a new valid `project.yaml` auto‑creates a project; ULID remains stable through renames.
- Checking in a changed PNG (>10 MB) stores content in LFS; edition captures both blob SHA‑1 and SHA‑256.
- Checking in with stale base commit triggers auto‑rebase; conflict UI resolves an LFS pointer by choosing ours/theirs; resulting edition is committed and tagged.
- Catalog stream for the project includes an `edition` record followed by exactly `total_files` `file` records; `root_tree_sha*` matches `git rev-parse HEAD^{tree}`.

