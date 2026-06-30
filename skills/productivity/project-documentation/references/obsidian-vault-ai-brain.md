# Obsidian Vault as AI Agent Brain

**Source:** [Michael Crist — How I Turned My Obsidian Vault Into Claude Code's Brain](https://michaelcrist.substack.com/p/context-engineering)

## Core Insight
AI agents (Claude Code, Hermes Agent) start fresh every session — like a brilliant employee with amnesia. Obsidian vaults solve this by providing persistent context via markdown files the agent reads at session start.

## Setup Pattern (used in FundManager V2)

### 1. Vault Location
Inside the project repo: `docs/obsidian/` — version-controlled alongside code.

### 2. HERMES.md (Root Context File)
Place at vault root. AI reads it on session start. Contains:
- Who the project serves
- What's being worked on (current status)
- Vault folder structure
- How the agent should work (conventions, language, linking)
- Maintenance instruction: "update _index.md when creating/deleting files"

### 3. Folder Structure
```
docs/obsidian/
├── HERMES.md                   ← Root context (like CLAUDE.md)
├── .obsidian/app.json          ← Vault config
├── 00 - Dashboard/             ← Overview + quick links + progress
│   └── _index.md
├── 01 - Product/               ← PRD, personas
│   └── _index.md
├── 02 - Architecture/          ← System design, tech stack
│   └── _index.md
├── 03 - Backend/               ← API routes, controllers
│   └── _index.md
├── 04 - Android/               ← Screens, data layer, sync
│   └── _index.md
├── 05 - Database/              ← Schema, migrations, ERD
│   └── _index.md
├── 06 - Workflows/             ← Business flows, sync patterns
│   └── _index.md
├── 07 - Sessions/              ← Session logs, ACTION_LOG refs
│   └── _index.md
├── 08 - Open QNA/              ← FAQ index
│   └── _index.md
├── 09 - Resources/             ← Skills, external refs, config
│   └── _index.md
└── Templates/                  ← Note templates
```

### 4. _index.md Pattern
Each folder has ONE `_index.md` that serves as a map for the AI:
- `created`, `status`, `tags` in YAML frontmatter
- Overview of what the folder contains
- Wiki links (`[[Note Name]]`) to key documents
- Quick stats relevant to the domain

### 5. Maintenance Rule (in HERMES.md)
```
## Maintenance
- Every time you create or delete a file, update the _index.md in that folder.
```

This makes the AI the librarian — keeping its own map current.

## Key Principles
- **Flat structure**: Avoid deep folder nesting. Max 1 level.
- **Wikilinks everywhere**: `[[Another Note]]` connects ideas
- **YYYY-MM-DD dates**: Standard format
- **YAML frontmatter**: `created`, `status`, `tags` on every note
- **One vault per project**: Don't split across multiple vaults
- **Version controlled**: The vault IS part of the project git repo
