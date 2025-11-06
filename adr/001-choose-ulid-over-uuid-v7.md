# ADR 001: Choose ULID Over UUID v7 for Project Identifiers

**Status:** Accepted — 2025-11-05

---

## Context

Creacc requires globally unique, time-sortable identifiers for projects, spaces, leases, and other entities. These identifiers must be:

- Immutable across the entity lifecycle
- Time-sortable for chronological ordering
- Compact and efficient for storage and transmission
- Human-readable for support, debugging, and user communication
- URL-safe for API endpoints
- Git-friendly for commit messages and branch names

Two primary candidates were evaluated:
1. **ULID** (Universally Unique Lexicographically Sortable Identifier)
2. **UUID v7** (Time-Ordered UUID, RFC 9562)

---

## Technical Specifications

**ULID (Universally Unique Lexicographically Sortable Identifier)**
- **Format**: 26 characters, base32-encoded (Crockford's base32)
- **Structure**: 48-bit timestamp (millisecond precision) + 80-bit random
- **Example**: `01ARZ3NDEKTSV4RRFFQ69G5FAV`
- **Encoding**: Case-insensitive, URL-safe, human-readable
- **Sort order**: Lexicographic string sorting equals chronological sorting
- **Timestamp range**: ~10,000 years from Unix epoch
- **Collision resistance**: 2^80 random bits per millisecond

**UUID v7 (Time-Ordered UUID)**
- **Format**: 36 characters (32 hex + 4 hyphens), or 32 hex without hyphens
- **Structure**: 48-bit timestamp (millisecond precision) + 12-bit random sub-millisecond sequence + 62-bit random
- **Example**: `018e8c3a-6d9f-7000-8000-123456789abc`
- **Encoding**: Hexadecimal, requires parsing for sorting
- **Sort order**: Binary sorting equals chronological sorting
- **Timestamp range**: ~10,000 years from Unix epoch (same as ULID)
- **Collision resistance**: 2^74 random bits (12-bit counter + 62-bit random)

---

## Pros and Cons for Creacc

### ULID Advantages ✅

1. **Human-Readable**: Easier to read, copy, and communicate (`01ARZ...` vs `018e8c3a-...`)
2. **URL-Safe by Default**: No special characters, works in URLs without encoding
3. **Shorter String Representation**: 26 chars vs 36 chars (28% more compact)
4. **Case-Insensitive**: Reduces human error when transcribing
5. **Simpler Database Indexing**: Native string comparison works for time-ordering
6. **Git-Friendly**: No hyphens, easier to select/copy in terminals
7. **Existing PRD Choice**: PRD already specifies ULID (consistency matters)
8. **Better Collision Resistance**: 80 random bits vs 74 in UUID v7

### ULID Disadvantages ⚠️

1. **Less Standardized**: RFC draft vs UUID v7's RFC 9562 (approved standard)
2. **Smaller Ecosystem**: Fewer native database/language implementations
3. **No Sub-Millisecond Ordering**: Can't guarantee order within same millisecond
4. **Custom Implementation**: May need to implement Crockford base32 encoding

### UUID v7 Advantages ✅

1. **Official RFC Standard**: RFC 9562 approved (July 2024)
2. **Native Database Support**: PostgreSQL, MySQL have UUID types
3. **Wider Ecosystem**: More libraries and tools support UUIDs
4. **Sub-Millisecond Ordering**: 12-bit counter provides ordering within same ms
5. **Backward Compatible**: Fits existing UUID infrastructure
6. **Standard Binary Format**: 128-bit binary representation is well-defined

### UUID v7 Disadvantages ⚠️

1. **Less Human-Readable**: Hexadecimal with hyphens is harder to transcribe
2. **Longer Representation**: 36 characters (28% larger than ULID)
3. **URL-Unfriendly**: Requires hyphen handling in some contexts
4. **Case-Sensitive Hex**: Can cause confusion with mixed case
5. **Hyphen Handling**: Different systems format with/without hyphens
6. **Less Collision Resistance**: 74 random bits vs 80 in ULID

---

## Performance Characteristics

| Metric | ULID | UUID v7 |
|--------|------|---------|
| **String Length** | 26 chars | 36 chars (with hyphens) |
| **Binary Size** | 128 bits | 128 bits |
| **Parse Complexity** | Base32 decode | Hex decode + hyphen stripping |
| **Sort Performance** | String comparison | Binary comparison (best), String comparison (works) |
| **Index Size** | Smaller (shorter strings) | Larger (longer strings if stored as text) |
| **Generation Speed** | Fast (timestamp + random) | Fast (timestamp + counter + random) |

---

## Creacc-Specific Considerations

**For Your Use Case:**
- **Immutable Project Identifiers**: Both work equally well
- **Time-Sortability Required**: Both provide millisecond-precision time-ordering
- **Git Integration**: ULID's hyphen-free format is slightly better
- **Database Schema**: SQLite/PostgreSQL both can handle either (text or binary)
- **API URLs**: ULID's shorter, URL-safe format is cleaner (`/projects/01ARZ3NDEKTSV4RRFFQ69G5FAV`)
- **Human Interaction**: Creative teams may need to reference project IDs in communication - ULID is easier
- **Audit Logs**: ULID's readability helps with debugging and support

**From Your Architecture:**
```yaml
repository_id: 01J9Z2Q3W8KFX8Q3R2A5V7YB1M  # ULID format already in PRD
```

---

## Recommendation: Use ULID 🎯

**Rationale:**

1. **Consistency with PRD**: PRD already specifies ULID format. Changing now would require updating all documentation.

2. **Better for Creative Users**: Creative teams (non-technical users) will interact with these IDs. ULID's readability is a significant UX advantage.

3. **Git-Friendly**: Hyphen-free format works better in commit messages, branch names, and terminal operations.

4. **URL Cleanliness**: `/projects/01ARZ3NDEKTSV4RRFFQ69G5FAV` is cleaner than `/projects/018e8c3a-6d9f-7000-8000-123456789abc`.

5. **Strong Enough Standard**: While UUID v7 is RFC-approved, ULID has proven production use (GitHub, MongoDB, etc.) and sufficient library support.

6. **Implementation Simplicity**:
   - TypeScript: `ulid` npm package (well-maintained)
   - PostgreSQL: Store as TEXT or BYTEA (no special UUID type needed)
   - Git: Direct string usage without escaping

7. **Future-Proof**: If you ever need UUID compatibility, you can add a secondary `uuid` field, but primary `repository_id` stays ULID.

---

## Implementation

### TypeScript Generation and Validation
```typescript
// src/identity/ulid.ts
import { ulid } from 'ulid';

export function generateUlid(): string {
  return ulid(); // 01ARZ3NDEKTSV4RRFFQ69G5FAV
}

export function validateUlidFormat(id: string): boolean {
  return /^[0123456789ABCDEFGHJKMNPQRSTVWXYZ]{26}$/.test(id);
}

export function extractTimestamp(id: string): Date {
  const timestampChars = id.slice(0, 10);
  const timestamp = decodeTime(timestampChars);
  return new Date(timestamp);
}
```

### Database Schema (PostgreSQL)
```sql
CREATE TABLE projects (
  project_id TEXT PRIMARY KEY
    CHECK (project_id ~ '^[0-9A-Z]{26}$'),
  space_id TEXT NOT NULL
    CHECK (space_id ~ '^[0-9A-Z]{26}$'),
  name TEXT NOT NULL,
  repository_id TEXT NOT NULL
    CHECK (repository_id ~ '^[0-9A-Z]{26}$'),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  tags JSONB DEFAULT '[]',
  description TEXT
);

CREATE INDEX idx_projects_space_created
  ON projects (space_id, created_at DESC);
```

### Database Schema (SQLite)
```sql
CREATE TABLE projects (
  project_id TEXT PRIMARY KEY
    CHECK (length(project_id) = 26 AND project_id GLOB '[0-9A-Z]*'),
  space_id TEXT NOT NULL
    CHECK (length(space_id) = 26),
  name TEXT NOT NULL,
  repository_id TEXT NOT NULL
    CHECK (length(repository_id) = 26),
  created_at TEXT NOT NULL
    DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
  tags TEXT DEFAULT '[]',
  description TEXT
);

CREATE INDEX idx_projects_space_created
  ON projects (space_id, created_at DESC);
```

### API Response Example
```json
{
  "project_id": "01J9Z2Q3W8KFX8Q3R2A5V7YB1M",
  "space_id": "01J9Z2PXRQJ8V6H3N2K5W7YB1N",
  "name": "Winter 2025 Campaign",
  "repository_id": "01J9Z2Q3W8KFX8Q3R2A5V7YB1M",
  "created_at": "2025-10-14T13:00:00Z",
  "tags": ["campaign:Winter25"],
  "description": "Marketing materials for winter product launch"
}
```

---

## Consequences

### Positive ✅

- **Improved UX**: Shorter, more readable IDs for creative team communication
- **Reduced API Payload Size**: 28% smaller identifiers reduce bandwidth usage
- **Simpler Git Integration**: No hyphen handling in branch names or commit messages
- **Strong Collision Resistance**: 2^80 random bits provides ample safety margin
- **Consistent Documentation**: Aligns with existing PRD specifications

### Negative ⚠️

- **Custom Validation Logic**: Cannot rely on native UUID validation in some contexts
- **Smaller Library Ecosystem**: Fewer off-the-shelf tools compared to UUID
- **Perceived "Non-Standard" Status**: Some developers may prefer RFC-approved UUID v7

### Mitigations

- Use well-maintained `ulid` npm package (70k+ weekly downloads, active maintenance)
- Implement comprehensive validation in identity module (see `openspec/specs/identity/spec.md`)
- Document ULID format clearly in API documentation and developer guides
- Provide helper functions for timestamp extraction and validation

---

## References

- [ULID Specification](https://github.com/ulid/spec) — Canonical ULID spec
- [RFC 9562: UUID Version 7](https://www.rfc-editor.org/rfc/rfc9562.html) — UUID v7 standard
- [ulid npm package](https://www.npmjs.com/package/ulid) — TypeScript implementation
- Creacc PRD (`prd.md`) § 1.5 — Repository identity and immutability requirements
- Creacc PRD (`prd.md`) § 5 — Project configuration with ULID examples
- OpenSpec `identity/spec.md` — ULID generation and validation requirements

---

## Notes

- This decision applies to **all** entity identifiers: projects, spaces, leases, repositories, audit entries
- If future requirements demand UUID compatibility (e.g., third-party integrations), we can add secondary UUID fields without changing primary `*_id` columns
- ULID's time-sortability enables efficient range queries: `WHERE project_id >= '01ARZ...' AND project_id < '01ASA...'`
- The decision aligns with Creacc's design principle: "Simplicity over complexity, explicit over implicit"
