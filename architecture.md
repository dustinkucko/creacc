# Git + LFS Server High-Level System Architecture

### Mini‑TOC
- [§ Overview](#overview)
- [§ Architecture Summary](#architecture-summary)
- [§ Key Principles & System Design](#key-principles--system-design)
- [§ Components](#components)
- [§ Primary File/Block Storage](#primary-fileblock-storage)
- [§ VM / Instance Setup](#vm--instance-setup)
- [§ Monitoring & Backups](#monitoring--backups)
- [§ Deployment Notes](#deployment-notes)
- [§ Scalability & Growth](#scalability--growth)
- [§ Primary Storage Semantics & Leader‑Replicated Layout](#primary-storage-semantics--leaderreplicated-layout)
- [§ TODO: Runbook](#todo--runbook-one-page)
- [§ Implementation Checklist](#implementation-checklist-terraform-level)
- [§ Appendix: Provider Notes](#appendix-provider-notes)

## Overview

A minimal, consistent, performant, and scalable architecture for hosting Git repositories with Git LFS. It emphasizes simplicity, strong POSIX semantics, and clean separation of concerns using a single‑writer, active/passive model: Git data on file/block storage; stateless LFS on object storage, with DR via snapshots.
- Focus on simplicity, correctness, and modular scaling
- See PRD: [§1 Abstractions & Definitions](./prd.md#abstractions--definitions), [§6 Git & LFS Behavior](./prd.md#6-git--lfs-behavior), and [§13 Check‑Out / Check‑In Workflow](./prd.md#13-check-out--check-in-workflow) for terminology and flow context

---

## Architecture Summary

- **Git Repositories:** Stored on primary storage mounted at `/srv/git` (cloud block volume or validated POSIX network filesystem)
- **Filesystem:** Local POSIX mount for Git data: either a `ZFS` filesystem on a block device or a validated network filesystem (e.g., TigrisFS)
- **Git Access:** Users interact via HTTPS (Smart HTTP); SSH optional
- **LFS Plane:** Stateless LFS servers behind HTTP/S endpoints
- **Object Storage:** S3-compatible storage for large binary files
- **Deployment Model:** A user‑facing space may be internally sharded across multiple storage shards (block volumes or filesystem namespaces); each shard is served by a single‑writer instance (plus standby)
- **Containerization:** OCI containers on VMs (e.g., Fly.io microVMs); default is two separate containers (Git and LFS) on the same VM
- **Spaces & Owners:** An owner (person or organization) is the top-level space; an owner can have multiple spaces; each space belongs to exactly one owner; internal sharding within a space is transparent to users → See PRD: [§1 Abstractions & Definitions](./prd.md#abstractions--definitions) for terminology

---

### Project ↔ Repository Mapping
- Projects carry workflow metadata (leases, catalogs, tags) and reference a single canonical repository via a shared ULID; the repository record owns storage bindings, refs, and head hashes.
- Architectural services reach Git data by resolving project → repository, keeping workflow concerns and Git semantics decoupled while preserving identity stability.
- Git submodules follow native Git behavior: each linked repository lives as a normal repository record, and check-in validation ensures `.gitmodules` entries point at known ULIDs with matching SHA-1/SHA-256 commits.

### Single‑writer model & failover (at a glance)
- One storage shard (block volume or filesystem namespace) per single‑writer VM; readers are served via mirrors
- Failover by attaching/mounting the same shard on a standby; see [§ Primary Storage Semantics & Leader‑Replicated Layout](#primary-storage-semantics--leaderreplicated-layout) for details

## Key Principles & System Design

- Simplicity first with minimal operations — avoid clustered filesystems unless necessary
- Strong POSIX semantics (atomic ops, locking) on the primary storage mounted to the VM using either:
  - Cloud‑managed block volume with a local filesystem
    - Benefits: strong consistency, durable, encrypted, resizable; slightly higher latency than on‑host NVMe but operationally safer; predictable IOPS/throughput, high I/O performance, built-in snapshot/replication features (with `ZFS`), and operational simplicity
  - Validated POSIX network filesystem (e.g., TigrisFS) that meets Git requirements
    - Must demonstrate comparable semantics and predictable performance
- Stateless LFS backed by object storage
- Scale horizontally by adding single‑writer instances via internal sharding within a user‑facing space; scale vertically via volume size/IOPS
- Snapshot‑oriented backups and disaster recovery
- Durability via snapshots with optional replication

---

## Components

### 1. Git Service (Container)

- The Git service (bare Git) runs on a single VM/instance
- The Git server handles all repository I/O locally; users never interact with the storage directly
- Runs Git over HTTPS using Smart HTTP
- **Co-located on the same VM as the LFS container by default** (separate containers, shared host network or bridge)
- Mounts primary storage at `/srv/git` (a cloud block volume with `ZFS`, or a validated POSIX network filesystem such as TigrisFS)
- Handles all push/pull operations for assigned repositories
- Exposes 443 for HTTPS; optional 80 for HTTP → HTTPS redirect

### 2. LFS Service (Container)

- Stateless LFS server that issues pre-signed URLs for upload/download
- **Co-located on the same VM as the Git container by default** to minimize hops and simplify ops
- Connects to object storage (S3-compatible) via credentials
- Optionally fronted by CDN (e.g., Cloudflare) for global caching of downloads
- Exposes 443 for HTTPS; optional 80 for HTTP → HTTPS redirect

### 3. Object Storage

- Stores all large files referenced by Git LFS
- Strong consistency for new objects (S3-compatible)
- Versioning enabled for durability and rollback
- Examples: Tigris, AWS S3, Cloudflare R2, DigitalOcean Spaces, MinIO

### 4. Primary File/Block Storage

#### TigrisFS

- See Appendix: [TigrisFS (Network Filesystem)](#tigrisfs-network-filesystem)

#### Cloud-Managed Block Storage (Network-Attached)

- Cloud‑managed volumes: Amazon EBS, GCP PD, Azure MD, DigitalOcean Volumes
  - These are network‑attached block devices exposed by the provider that appear as local disks to the VM
    - NVMe-oF, iSCSI, or proprietary — not Linux NBD
- Provide strong consistency and high I/O performance for Git data
- Each writer node mounts one or more block devices
- Each Git server mounts a cloud block volume dedicated to an internal shard of a user‑facing space
- The block device is formatted with a local `ZFS` filesystem
  - `ZFS` for built-in snapshot/replication features, data integrity, compression, and performance tuning
    - Assess whether snapshot/replication features can be handled by the cloud provider
- The filesystem is mounted at a stable path such as `/srv/git`
  - All Git repositories and related metadata reside under this mountpoint
- Use cloud volumes for repositories; reserve ephemeral local disks (e.g., NVMe) for caches and temporary data only
- Attachment model: single writer per volume; failover via detach/attach to a standby. If the provider offers multi‑attach, restrict to read‑only mounts (if used at all) and never RW multi‑attach; prefer Git mirrors for reads

##### DigitalOcean Volumes

- See Appendix: [DigitalOcean Volumes Block Storage](#digitalocean-volumes-block-storage)

---

## VM / Instance Setup

- Single‑writer VM per internal shard of a user‑facing space. Git & LFS run as two containers on the same VM by default
- Attach or mount primary storage at a stable path (e.g., `/srv/git`): block volumes → create a filesystem and mount; network filesystems → mount the namespace and validate POSIX semantics
- Mount persistence: use `/etc/fstab` or a systemd mount unit. Enable `noatime`; tune reserved space as appropriate; enable TRIM/`discard` if SSD‑backed
- Separation by shard: use one storage shard per internal shard to isolate capacity, performance, and snapshot schedules; shards together form the user‑facing space
- Snapshots/Backups: schedule storage snapshots (volume or filesystem) for fast rollback and DR
- Networking: expose 443 for HTTPS; allow egress to the object store endpoint; prefer private endpoints/VPC gateways when available
- IAM/Secrets: bind an instance role or scoped credentials for the LFS server to access the bucket
- Time sync: ensure NTP/chrony is enabled to avoid TLS and signature skew
- Compute: run a primary VM and optionally a warm standby in the same region; keep the standby detached until promotion
- Security rules: allow inbound HTTPS (443); enable SSH (22) only if required and tightly scoped to admins
- Network throughput: size NIC/bandwidth to saturate expected Git I/O throughput

---

## Monitoring & Backups

- Metrics: Disk I/O, HTTP latency, upload timeouts, and object storage latency
- Snapshots: Periodic storage snapshots for Git data (volume or filesystem); versioning on LFS buckets
- Logs: Centralized logging for Git/LFS operations
- Alerts: Trigger on >80% disk capacity or inode usage; watch for failed uploads and elevated HTTP error rates
- Snapshot verification: Track snapshot completion and schedule periodic restore tests

### Backup & Disaster Recovery
- Use automated snapshots of the primary storage (volume or filesystem) at regular intervals (daily, weekly, monthly)
- Optionally, replicate snapshots to another region or cloud for DR
- Maintain object storage backups (e.g., tarballs of bare repositories uploaded to S3-compatible storage) for long-term retention
- Snapshot cadence and retention: daily (7d), weekly (4w), monthly (6m); perform periodic restore tests
- Optional: run `git gc` during low-traffic windows for repo maintenance

---

## Deployment Notes

- Early validation:
  - Block I/O latency
  - Git locking behavior
  - LFS multipart upload throughput

- Start in a single region (e.g., U.S.) for latency and simplicity
- Replication via storage‑level mechanisms (`ZFS` send/recv) or provider snapshot scheduling
- Failover and recovery: keep a warm standby in the same AZ for fast attach; use snapshots to restore across AZ/region when needed
- Container orchestration: optional lightweight orchestration (e.g., Nomad) for consistency

---

## Scalability & Growth

- Horizontal scaling: keep one user‑facing space, and add writers by sharding it internally; each shard maps to one storage shard (volume or filesystem namespace) and one single‑writer VM; route within the space via DNS/LB or application routing
- Vertical scaling: increase block volume size, IOPS, or throughput; prefer online resize where supported
- Read scaling: serve clones/fetches from mirrors or cached clones; optionally use a CDN to accelerate LFS downloads
- Rebalancing: move repositories between internal shards of the same space with snapshot/copy and a brief push pause; update internal routing/config after cutover; the user‑facing space stays unchanged
- Monitoring‑driven growth: thresholds (e.g., >80% disk, IOPS/latency, snapshot lag) trigger capacity planning or adding shards (new volumes and single‑writer VMs) within the space
  - Avoid multi‑writer clustered filesystems; sharding preserves single‑writer‑per‑volume semantics

---

## Primary Storage Semantics & Leader‑Replicated Layout

Single‑attach rule: A cloud block volume is attached to one VM at a time for read‑write (RW) use, including NBD‑style devices; this underpins the single‑writer model
For network‑backed filesystems (e.g., TigrisFS, ZeroFS via NBD), enforce the same constraint by policy: only one RW mount is active at a time; failover unmounts the writer and mounts the standby

### High Availability (HA) Model
- Each storage shard (volume or filesystem namespace) is active on one primary server at a time (single‑writer)
- On failure, either detach/attach the volume to the standby, or unmount/demote the writer and mount the standby for network filesystems
- The standby resumes service using the same configuration and namespace

**How this works with our design**
- Each internal shard of a user‑facing space maps to one storage shard (block volume or filesystem namespace) and one single‑writer VM (with its own standby)
- All writes for a repository route to the shard’s writer over HTTPS
- Read mirrors do not mount the primary storage; they sync over the network using Git (e.g., `--mirror`) and serve clones/fetches independently
- A warm standby VM in the same AZ stays ready (it does not mount the live storage), receiving continuous replication (e.g., snapshots or stream replication); on failover, it attaches/restores the replicated storage and becomes the new writer

**Scaling by internal sharding**
- Keep one user‑facing space; add writers by creating additional internal shards (each shard = one volume + one single‑writer VM)
- To add capacity, split repository ownership across shards (e.g., by namespace or repo ranges), and update internal routing/LB to direct writes appropriately
- Rebalance by moving repositories between shards with a brief push pause and snapshot/copy; update routing/config after cutover

**What we intentionally avoid**
- **Multi‑attach RW** of the **same filesystem** using multiple VMs; even where providers support "multi‑attach" semantics, safe concurrent writes still require a clustered filesystem; we keep the design simpler with **one writer per volume**

**Summary**
- One storage shard (volume or filesystem namespace) → one single‑writer VM (plus standby) → many readers via mirrors
- A user‑facing space can contain multiple internal shards to scale writes; each shard preserves single‑attach semantics
- Replication via snapshots or `ZFS` send/recv; standby nodes remain detached until promotion, then attach/mount the replicated storage and take over

---

## TODO:  Runbook (One-Page)

- Roles and responsibilities (Git service, LFS service, object storage, primary storage, monitoring)
- Container lifecycle management (restarts, image updates), configuration templating for multiple spaces, and automated snapshot verification
- Container network design (port mapping, internal service discovery, TLS termination)
- Ports and service exposure summary
- Health checks (Git push/pull latency, LFS upload/download status)
- Snapshot cadence (block and object storage)
- Failover checklist (promote standby, attach volume, DNS update)
- Security & access notes (TLS, API credentials, IAM policies)
- Metrics & alerts (disk usage thresholds, failed uploads, HTTP errors)

## Implementation Checklist (Terraform-Level)

1. **Define variables and providers** (region, instance size, volume config, backup schedule)
2. **Create compute instance** (VM for Git server with networking and security rules)
3. **Provision primary storage**
  - For block volumes: create cloud volume
  - For network filesystem: provision and authorize mounts (e.g., TigrisFS)
4. **Attach block volume** to the instance (**block volumes only**)
5. **Cloud-init or provisioner script**
  - For block volumes: format as preferred FS and mount at `/srv/git`
  - For network filesystems: mount the namespace at `/srv/git` and validate POSIX semantics
  - Persist mounts in `/etc/fstab` or systemd mount units
6. **Install and configure Git service** to store data under `/srv/git`
7. **Configure snapshot schedules** (native backup plans or Terraform resources for snapshots)
8. **Optional** nightly backup of repos to object storage
9. **Implement observability** (volume metrics, alerting, snapshot status)
10. **Document failover procedure** (detach/attach steps or restore workflow)

---

## Appendix: Provider Notes

### TigrisFS (Network Filesystem)
- Use as the primary filesystem only if it meets POSIX requirements (atomic rename, hard links, correct locking, fsync)
- Mount each internal shard at `/srv/git` on the writer VM; preserve single‑writer semantics (no concurrent RW mounts)
- Failover: unmount/demote the writer; mount the same namespace on the standby and resume; verify no split‑brain
- Validate latency/throughput for metadata‑heavy Git workloads before production; prefer Tigris object storage for the LFS plane when using TigrisFS
- Preflight: run atomic rename, file locking, and fsync checks; similar alternatives include ZeroFS via NBD layered over S3‑compatible storage

### DigitalOcean Volumes Block Storage
- Volumes are network-based block devices built with Ceph
- One Volume per internal shard; attach to the shard’s writer Droplet in the same region
- Format with `ZFS` and mount at `/srv/git/<space>`; persist with `/etc/fstab`
- Use DigitalOcean Spaces (S3‑compatible) for LFS; co‑locate in the same region
- Failover: detach the Volume from the failed writer and attach to the standby; alternatively, restore from snapshot to a new Volume
- Treat Volumes as single‑attach RW; monitor IOPS/throughput and CPU/RAM; scale by adding shards/Volumes across writer Droplets
