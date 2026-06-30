# Obsidian Project Vault Setup — AI Context Engineering

## When to Create a Project Vault

Create an Obsidian vault (typically at `docs/obsidian/` in the project) when:
1. The project has 3+ documentation files (PRD, Q&A, Action Log, plans)
2. The user wants AI agents to have persistent context across sessions
3. The project spans multiple domains (backend, Android, DB, workflows)

## Vault Structure Template

```
docs/obsidian/
├── .obsidian/app.json              # Vault config (wikilinks)
├── HERMES.md                       # ROOT CONTEXT — agent reads first
├── 00 - Dashboard/_index.md        # Overview + stats + quick links
├── 01 - Product/_index.md          # PRD index
├── 02 - Architecture/_index.md     # System design
├── 03 - Backend/_index.md          # API routes + controllers
├── 04 - Android/_index.md          # Screen plans + data layer
├── 05 - Database/_index.md         # Schema + relationships
├── 06 - Workflows/_index.md        # Business flows
├── 07 - Sessions/_index.md         # Session logs
├── 08 - Open QNA/_index.md         # FAQ index
├── 09 - Resources/_index.md        # Skills + external refs
└── Templates/Note Template.md      # YAML frontmatter template
```

## HERMES.md Structure

The root context file is the agent's persistent brain. It should include:
- **Who I Am** — agent role description
- **What This Project Is** — 2-3 sentence summary
- **Vault Structure** — folder map
- **Current Status** — what's complete, pending, latest commit
- **How I Want You to Work** — language, wikilink convention, read _index.md first
- **Maintenance** — "Every time you create/delete a file, update _index.md"

## Key Principles

From research (Steph Ango, Michael Crist):
- Flat structure per domain (avoid nested folders)
- Wikilinks profusely — link concepts across domains
- YAML frontmatter: `created`, `status`, `tags`
- YYYY-MM-DD dates everywhere
- Update `_index.md` on every file change
- Set `OBSIDIAN_VAULT_PATH` in `.env`

## When to Update

| Trigger | Action |
|---------|--------|
| New API route added | Update `03 - Backend/_index.md` |
| New DB migration | Update `05 - Database/_index.md` |
| Workflow change | Update `06 - Workflows/_index.md` |
| Session ends | Update `07 - Sessions/_index.md` |
| Project status changes | Update `HERMES.md` |
