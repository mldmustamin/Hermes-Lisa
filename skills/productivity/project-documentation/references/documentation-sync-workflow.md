# Documentation Sync Workflow — FundManager V2

## User's Rule: Update ALL Docs After Every Change

> "Selalu cek dan update semua dokumen agar tidak ada yang miss saat debugging"

After every batch of changes (feature, bug fix, config change, 3+ file edits), update these 4 documents:

| Document | Location | Purpose |
|----------|----------|---------|
| **ACTION_LOG.md** | Project root | Chronological audit trail — symptom, diagnosis, fix, files changed, verification |
| **OPEN_QNA.md** | Project root | FAQ — answers to design decisions, architecture choices, known edge cases |
| **Obsidian Dashboard** | `docs/obsidian/00 - Dashboard/_index.md` | Current project state snapshot — progress %, stats, latest commit, next steps |
| **Obsidian Session Log** | `docs/obsidian/07 - Sessions/_index.md` | Session activities log — what was built, decisions made, sources consulted |

## ACTION_LOG Entry Format

```markdown
## NN. Title — What Changed

| Area | Before | After |
|------|--------|-------|
| Symptom | What was broken | What was the root cause |
| Diagnosis | How you found it | Steps to reproduce |
| Fix | Files changed | Verification method |
```

## OPEN_QNA Entry Format

```markdown
### QNN: Question title?

**Jawab:** Answer in Indonesian (for this project).

**Detail:** Technical details if needed.
```

## When to Update (Checklist)

- [ ] New feature added → ACTION_LOG + Q&A if design decision involved
- [ ] Bug fixed → ACTION_LOG with symptom/diagnosis/fix
- [ ] Config changed → ACTION_LOG
- [ ] API endpoint added → ACTION_LOG + OPEN_QNA if design choice
- [ ] Database schema changed → ACTION_LOG + Obsidian DB index
- [ ] Code review completed → ACTION_LOG entry
- [ ] Test suite run → Obsidian Dashboard + Session Log
- [ ] APK built → Obsidian Dashboard (build # and size)
- [ ] New research/source consulted → Session Log
