# Obsidian Vault Template for Software Projects

**Pattern:** Structure Obsidian vault inside project repo for AI-agent-compatible documentation.

## Vault Location
Place at `docs/obsidian/` inside the project root. Version-controlled with git.

## Folder Structure
```
docs/obsidian/
├── .obsidian/app.json              # Vault config
├── HERMES.md                       # Root context file (like CLAUDE.md for AI agents)
├── 00 - Dashboard/_index.md        # Project overview + quick links + stats
├── 01 - Product/_index.md          # PRD, user personas, feature list
├── 02 - Architecture/_index.md     # System design, tech stack decisions
├── 03 - Backend/_index.md          # API routes, controllers, endpoints
├── 04 - Android/_index.md          # Screens, data layer, navigation
├── 05 - Database/_index.md         # Schema, migrations, ERD
├── 06 - Workflows/_index.md        # Business flows, approval chains
├── 07 - Sessions/_index.md         # Session logs, ACTION_LOG entries
├── 08 - Open QNA/_index.md         # FAQ, stakeholder questions
├── 09 - Resources/_index.md        # Skills, external references, config
└── Templates/Note Template.md      # YAML frontmatter template
```

## Key Files

### .obsidian/app.json
```json
{
  "promptDelete": false,
  "newLinkFormat": "shortest",
  "attachmentFolderPath": "Attachments",
  "alwaysUpdateLinks": true
}
```

### HERMES.md (Root Context)
```markdown
# HERMES.md — Project Name Context

## Who I Am
Hermes Agent — AI assistant for [Project Name].

## What This Project Is
[One paragraph describing the project]

## Vault Structure
[Brief description of each folder]

## Current Status
[Date + current sprint status]

## How I Want You to Work
- Read the relevant folder's `_index.md` before working on anything in it
- Update `_index.md` when creating or deleting files
- Keep responses concise and in [language]
- Use `[[wikilinks]]` when referencing other notes

## Maintenance
- Every time you create or delete a file, update the `_index.md` in that folder.
- After every session, update `07 - Sessions/` with session summary.
```

### Note Template
```markdown
---
created: YYYY-MM-DD
status: draft | active | complete | archived
tags: [tag1, tag2]
---

# Note Title
```

## Principles (from Steph Ango)
- Avoid splitting content into multiple vaults
- Avoid deep folder nesting
- Avoid non-standard Markdown
- Use `YYYY-MM-DD` dates everywhere
- Use wikilinks profusely
- Use YAML frontmatter for structured metadata

## Benefits for AI Agents
- Plain Markdown files — native format for LLMs
- Index files per folder serve as "maps" — agent reads _index.md instead of every file
- HERMES.md at root auto-loaded as session context
- Wikilinks create navigable knowledge graph
- Git-tracked — documentation stays with code

## Integration with [project-documentation](../productivity/project-documentation/SKILL.md)
This Obsidian vault structure extends the project-documentation skill:
- PRD.md → `01 - Product/`
- OPEN_QNA.md → `08 - Open QNA/`
- ACTION_LOG.md → `07 - Sessions/`
- Plans → referenced from `00 - Dashboard/`
