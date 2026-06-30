---
name: project-documentation
category: productivity
description: Create and maintain project documentation — PRDs, Open Q&A, action logs, and session change records.
---

# Project Documentation

Class-level skill for creating and maintaining project documents during development sessions. Covers three document types commonly requested in the same session: PRD (Product Requirements Document), Open Q&A (FAQ / stakeholder Q&A), and Action Log (change record).

## When to Load

- User says "buat PRD" / "buat Open Q&A" / "buat FAQ" / "buat dokumentasi"
- User says "catat di action_log" / "catat semua perubahan" / "record changes"
- User asks to document the current session's work

---

## Document Types

### 1. PRD (Product Requirements Document)

A PRD defines what the product does, who it serves, and how it works. It is **not** a technical spec — focus on goals, users, features, and constraints.

**Mandatory sections:**

| Section | What to include |
|---------|----------------|
| Executive Summary | One-paragraph product positioning |
| Problem Statement | What pain points this solves |
| Target Users & Personas | 3-6 named personas with role, interface, and key goals |
| Product Vision | Concise one-paragraph vision statement |
| Core Capabilities (Epics) | 6-10 epics, each with bullet-point capabilities |
| Non-Functional Requirements | Offline-first, security, performance, audit, immutability |
| System Architecture | Brief stack diagram + component descriptions |
| Data Model Overview | Key entities and their relationships |
| API Overview | Endpoint categories, count, auth model |
| Security & Compliance | Auth, RBAC, data protection |
| Out of Scope | Explicit list of what V1/V2 does NOT include |
| Success Metrics | Quantitative targets |

**Sources to read before writing:** README.md, ROADMAP.md, docs/02_PRODUCT_SCOPE.md, docs/03_ARCHITECTURE.md, docs/08_FINANCE_RULES.md, docs/09_RBAC_SECURITY.md, docs/10_DATABASE_SCHEMA.md

### 2. Open Q&A (FAQ / Stakeholder Q&A)

An Open Q&A answers anticipated questions from stakeholders, users, and the team. Usually 20-30 questions grouped by topic.

**Grouping categories:**
- Produk & Visi (product vision, positioning)
- Arsitektur & Teknis (tech stack decisions)
- Keuangan (money handling, financial rules)
- Role & Keamanan (RBAC, security)
- Web Dashboard (web-specific questions)
- Sync & Data (sync strategy, offline behavior)
- Development (stack, progress, timeline)
- Miscellaneous (other questions)

**Each question format:** `Q{N}: {question}` with a clear, direct answer in Indonesian (if the project is Indonesian-focused).

### 3. Action Log (Session Change Record)

**This is REQUIRED for every session where files are changed.**

### 4. Obsidian Vault as AI Brain

**📁 Reference:** `references/obsidian-vault-ai-brain.md` — Full pattern for setting up an Obsidian vault with HERMES.md as persistent agent context across sessions. Used in FundManager V2. The user explicitly enforces this rule.

File location: `ACTION_LOG.md` in the project root.

**Structure:**

```markdown
# Action Log — {Project Name} {Task Name}

**Session ID:** `{session_id}`
**Date:** {date}
**Model:** {model}
**Provider:** {provider}

---

## {N}. {Action Group Title}

| Action | Detail |
|--------|--------|
| **{Action}** | {Description of what was done} |
```

**Rules:**
- Every section covers a logical group of related actions
- Include exact commands run, files changed, and results
- Bug fixes get their own section with: Laporan, Diagnosis, File, Fix, Status
- Testing gets its own section with pass/fail counts
- Running services end state is documented
- APK build history (if Android) gets its own table with build#, trigger, status, size
- End with a ringkasan table for quick reference

**When to update:**
- After every logical batch of changes (not after every single tool call)
- At minimum, update before the session ends
- When the user explicitly says "catat di action_log"

---

## Workflow

1. **Read existing project docs first** — README.md, ROADMAP.md, existing docs/*.md
2. **Create PRD** from project context — covers vision, users, features, constraints
3. **Create Open Q&A** — anticipate stakeholder questions
4. **Start Action Log** with session ID at the beginning of work
5. **Append to Action Log** as each batch of changes completes
6. **Update Open Q&A after feature implementation** — new features, schemas, or business rules warrant 8-10 new questions covering: workflow, security, offline handling, integration points
7. **Update Open Q&A after code review** — any finding that changes behavior (pagu enforcement, role checks) should be reflected in Q&A answers
6. **Update Open Q&A after feature implementation** — new features, schemas, or business rules warrant 8-10 new questions covering: workflow, security, offline handling, integration points
7. **Update Open Q&A after code review** — any finding that changes behavior (pagu enforcement, role checks) should be reflected in Q&A answers. Code review findings that answer "what happens if X" become Q&A entries.
8. **Create or update Obsidian vault index files** — if the project has an Obsidian vault (see `obsidian` skill), update `_index.md` in each affected folder after schema changes, new APIs, or workflow changes. Also update `HERMES.md` if the project status changes.
9. **Finalize Action Log** at session end with all changes, bug fixes, build history

### Project-specific conventions

For FundManager V2, see `references/fundmanager-conventions.md` for:
- Indonesian-language OPEN_QNA format
- ACTION_LOG entry templates (bug fix, code review, test results)
- When to add new Q&A categories

See `references/documentation-sync-workflow.md` for the 4-document sync rule — maintain all 4 simultaneously after every change batch.
- Commit tracking conventions

For Obsidian vault setup and maintenance, see `references/obsidian-project-vault.md`.

For code review documentation patterns, see `references/code-review-checklist.md`.

---

## Pitfalls

| Pitfall | Description |
|---------|-------------|
| **PRD too technical** | PRD is for stakeholders, not developers. Don't dive into implementation details. Focus on capabilities, not code. |
| **Q&A too shallow** | Anticipate real questions users will ask. If you don't know the answer, note it as "dokumen ini perlu diupdate ketika ada keputusan". |
| **Action Log missing session ID** | Always include the session ID at the top. The user may need to trace which session made which change. |
| **Action Log too vague** | Each entry should answer: what was changed, why, and what the result was. "Fixed bug" is not enough — what bug, what file, what was the root cause? |
| **Skipping Action Log update** | The user specifically enforces "catat semua di action_log setiap melakukan perubahan". Update it multiple times during the session, not just at the end. |
| **RBAC changes not in ACTION_LOG** | When modifying roles or permissions, document: Symptom → Diagnosis → Root Cause → Fix (with exact file paths) → Verification (test results + curl smoke test). See `fundmanager-rbac` skill for the full RBAC change audit pattern. |
| **Stale docs after schema change** | User directive: "Selalu cek dan update semua dokumen agar tidak ada yang miss saat debugging." After any DB migration, model change, or new config file, check ALL relevant docs (PRD.md, OPEN_QNA.md, skill reference files, ACTION_LOG.md) and update them. A schema change that isn't reflected in docs is a future debugging trap. |
| **Obsidian vault integration** | See `references/obsidian-vault-structure.md` for the recommended Obsidian vault layout within software projects. The vault structure extends this skill's document types (PRD, OPEN_QNA, ACTION_LOG) into a navigable wiki format with `_index.md` files per folder. |
| **Skipping research before acting** | User mandate: "Gunakan semua sumberdaya, jangan malas buat research." Before starting any task, research: web search for best practices, official docs, version-specific changes. Use web_search, web_extract, and load relevant skills. Never assume you already know — patterns change across versions. |
| **Q&A answers diverge from code after review** | After code review fixes (role checks added, pagu enforcement implemented, validation rules tightened), re-read the relevant Q&A entries and update them. If pagu enforcement went from unimplemented to 422-block for FIXED_PAGU, Q33 must reflect current behavior. Out-of-sync docs are worse than missing docs. |
| **Obsidian vault stale after backend changes** | New API routes, DB migrations, or workflow changes must update the corresponding _index.md in the Obsidian vault. The vault's HERMES.md instructs updating _index.md on every file change — this applies to backend files too. |
| **New files break web server (Laravel)** | When agent creates PHP files, config, or Blade views as root, PHP-FPM (www-data) cannot read them. See `references/laravel-permissions.md` for the complete checklist. Always run `chmod 755` on new directories and `chmod 644` on new PHP files after creating them. |
| **Android app freezes on login** | Ktor HttpClient has no default timeout. When server is unreachable, HTTP request hangs forever → `isLoading` stays `true` → UI frozen. See `references/android-ktor-timeout.md`. Always include `install(HttpTimeout)` with 15s request / 8s connect when creating HttpClient. |
| **Blade forms: money input + searchable dropdown** | See `references/laravel-alpinejs-form-patterns.md` for IDR formatting, searchable selects, auto-calculation, and Blade+Alpine.js syntax gotchas. |
| **Repo as GitHub Pages blog** | See `references/github-pages-blog.md` for enabling Pages via API, URL format, and Markdown-only strategy (no Jekyll config needed). |
