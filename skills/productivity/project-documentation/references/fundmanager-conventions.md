# FundManager V2 — Documentation Conventions

## Research-Before-Acting Directive

**User mandate:** "Gunakan semua sumberdaya yang kamu perlukan. jangan malas buat research."

Before writing any production code or major document update:

1. **Browse the web** — `web_search` + `web_extract` for best practices, official docs, version-specific changes
2. **Read existing project docs** — PRD.md, OPEN_QNA.md, docs/01-10, README.md, ROADMAP.md
3. **Inspect the codebase** — `search_files`, `codebase-inspection` skill for LOC/ratio
4. **Load relevant skills** — check skills_list for any skill that covers the domain
5. **Check reference files** — skills may have `references/` files with session-specific knowledge

**Pitfall:** "I already know this" is the most dangerous assumption. Even familiar stacks (Laravel, Room, Kotlin) have version-specific changes. A 2024 pattern may be deprecated in 2026.

Project-specific rules for OPEN_QNA.md and ACTION_LOG.md in the FundManager V2 repository.

## OPEN_QNA.md — Format & Triggers

### When to update
- After implementing a new feature or API (budget request, laporan pekerjaan, etc.)
- After schema changes (new tables, new models)
- After business rule changes (pagu enforcement, role permissions)
- When the user says "buat Open QNA" or "update Open QNA"

### Language
- **Indonesian** for all questions and answers
- Concise, direct answers — no paragraphs unless needed
- Each question: `Q{N}: {question}` with a clear answer

### Grouping
- Produk & Visi
- Arsitektur & Teknis
- Keuangan
- Role & Keamanan
- Web Dashboard
- Sync & Data
- Development
- **Budget Request Workflow (Baru)** — Q31+ for budget-specific questions
- Miscellaneous

### Example — Budget Request questions to always include after feature:
- Alur budget dari FE sampai rekonsiliasi (7 stage)
- Rejection cascade behavior
- Pagu enforcement rules (FIXED_PAGU, TICKET, MANAGER_APPROVAL)
- Offline mode handling
- Task assignment via SUPERVISOR
- Location history for OWNER approval
- Foto wajib SCM checklist

## ACTION_LOG.md — Format & Triggers

### Entry structure
```markdown
## {N}. {Action Group Title}

| Area | Detail |
|------|--------|
| **What** | Description |
| **Files** | Exact paths changed |
| **Verification** | Test results, commands run |
```

### Bug fix entries — extended format
```markdown
| Laporan | ... |
| Diagnosis | ... |
| Root Cause | ... |
| File diubah | ... |
| Fix | ... |
| Status | ... |
```

### Code review entries
Include severity table (CRITICAL/BUG/MISSING/QUALITY) with counts: found → fixed → noted.

### Testing entries
Final test count and assertion count. List new tests added.

### Commit tracking
Reference the session's commits indirectly through the action entries — no need for explicit commit SHAs in the log (those become stale).
