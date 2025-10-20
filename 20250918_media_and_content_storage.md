# Media and Content Storage

Purpose: Git LFS as source of truth for media manifests and derivatives; S3‑compatible object storage for large masters. Optimized for creators (easy upload/manage) and developers (scriptable, reproducible, CI‑friendly) without vendor lock‑in.

Core requirements
- Git versioning: text in Git; binaries via Git LFS pointers.
- Object storage backend: S3‑compatible, vendor‑agnostic; accessible via `rclone` and standard CLIs.
- Manifest‑driven mapping: per‑repo YAML manifest listing masters (object keys, size, sha256) and deterministic derivatives.
- Deterministic derivations: reproducible transforms for images/video/audio; derivation commands tracked in repo.
- Derivatives in Git LFS: commit publish‑ready outputs (e.g., webp, thumbnails, loops) to enable fast diffs and reviews.
- Locking for binaries: LFS file locking for PSD/AI and other exclusive‑edit assets.
- Creator UX: drag‑and‑drop or simple CLI for upload; bulk operations; easy previews.
- Developer UX: scriptable fetch/derive, CI verification, partial fetch, caching.
- Integrity & audit: checksums, size, and metadata tracked; CI validates manifest and outputs.
- Scalability & cost: lifecycle policies, cold storage options, CDN for delivery.

Workflow
1. Upload masters to object storage.
2. Update manifest with keys + hashes.
3. Fetch & derive locally or in CI (deterministic commands).
4. Commit derivatives to Git LFS; open PR for review.
5. CI validates manifest, derivations, and build; publish via CDN.

Tools
- `rclone`, ImageMagick/Sharp, ffmpeg, Git LFS (with locking), checksum validators.

Next step
- Provide a drop‑in starter kit: manifest template + rclone remote config + derive scripts.
