# Brand Repository

## Requirements

- Brand repository works in tandem with creatives tools
  - Integration with social media platforms and Shopify stores/themes
- Scope includes assets, copy, guidelines, templates, prompts
- Media files will exist in object storage (S3-compatible).
- Repository will contain Git LFS pointers and/or manifests that reference the media files.

## Naming Conventions and Consistency:
- Use file/folder slugs (kebab-case or snake_case) derived from names.
  - i.e. Midnight Black T-Shirt.md → midnight-black-t-shirt.md or midnight_black_t-shirt.md
- Folders of campaigns and creatives organized by date
  - i.e. YYYY-MM-DDTHH:MM:SS±hh:mm (ISO 8601 contained in metadata)
- ULID for unique, sortable identifiers
- SKU product identifiers that enable relationships with assets.
- Templates: Reusable patterns for pages.
- Automation: Simple scripts + CI workflows to validate manifests and lint Markdown.

## Example Naming Conventions

Derive slugs from names
- Files: topic_or_sku_slug.ext → midnight_black_teaser.mp4
- Campaigns: YYYYMMDD_slug → 20250918_midnight_black_launch
