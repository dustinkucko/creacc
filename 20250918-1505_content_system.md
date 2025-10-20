# Content System Requirements

- **Human-friendly**: easy for non-technical users to read/write/edit.
- **Agent-friendly**: easy for AI and automation to read/write/edit.
  - MCP integration for tool-calls.
- **Markdown format** for both agents and humans to understand.
  - Support for front-matter metadata (YAML, TOML, JSON).
- **Versioned**: tracked in Git for history, diffs, collaboration.
  - Markdown is primary source of truth, tracked in Git.
- **Open Source**: no vendor lock-in, minimal dependencies.
- **Media References**: must support linking any media files like images/videos for:
  - Social media creatives, website content, marketing materials.
- Google Drive (current) → S3-compatible object storage (future)
- **Front-End Hosting**: Cloudflare Workers
- **Validation**: ensure all media references are valid (no broken links) and there are no orphaned assets.

## Nice-to-Haves

- Support for using Static Site Generators with HTML/Liquid templates.
- Lightweight UI for non-technical users.
- Incremental builds / fast previews.

## Potential Solution

A lean, agent-first, git-based setup that is portable and Markdown-centric

### Minimal Git-Based Markdown

### Media Handling

- Media binaries are NOT committed.
- Manifest file or files for references (Google Drive or S3, IDs and URLs).
- Markdown docs reference assets by **key**

### Front-End Integration

- MDX, remark, GFM are all options for Markdown rendering
  - evaluate what is best for an end user to edit through a UI in a future state (may be simple enough if it is designed well)
  - evaluate what is best for an agent / developer to edit (programmatically, cli, ide, or other)
- Resolve URLs

### Automation

- Edit Markdown and manifest
- Validate links to ensure all asset keys exist
