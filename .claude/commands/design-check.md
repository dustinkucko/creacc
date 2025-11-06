# Design Consistency Check

You are reviewing the design documents for Creacc to ensure consistency, completeness, and flag potential issues before implementation begins.

## Your Task

1. **Read both design documents:**
   - `prd.md` — Product requirements and specifications
   - `architecture.md` — Infrastructure and system design

2. **Check for inconsistencies:**
   - Do the architecture choices support all PRD requirements?
   - Are there contradictions in terminology, data models, or workflows?
   - Are there gaps where PRD defines a feature but architecture doesn't address it?
   - Are there ambiguities that would block M1 implementation?

3. **Validate critical paths:**
   - Check Out / Check In workflow (PRD §13) — does architecture support lease enforcement, conflict resolution, and catalog updates?
   - Dual SHA-1/SHA-256 hashing (PRD §11) — does architecture address storage and indexing requirements?
   - LFS policy (PRD §6) — does architecture handle the 10MB threshold and extension-based tracking?
   - Catalog format (PRD §14) — does architecture provide append-only JSONL with integrity guarantees?

4. **Flag implementation risks:**
   - Dependencies on unspecified components
   - Performance assumptions that need validation
   - Security considerations that need detail
   - Scalability bottlenecks

5. **Output format:**
   ```markdown
   ## Consistency Check Results

   ### ✅ Aligned
   - [Brief list of major areas that are well-aligned]

   ### ⚠️  Potential Issues
   - [Specific concerns with references to PRD/architecture sections]

   ### ❓ Questions for Clarification
   - [Ambiguities that should be resolved before M1]

   ### 💡 Suggestions
   - [Optional improvements or areas to elaborate]
   ```

Be concise but specific. Reference section numbers (e.g., "PRD §13", "architecture.md line 45") so issues can be quickly located.
