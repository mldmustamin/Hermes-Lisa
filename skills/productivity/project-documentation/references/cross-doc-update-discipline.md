# Cross-Documentation Update Discipline

**User directive:** "Selalu cek dan update semua dokumen agar tidak ada yang miss saat debugging."

## When to Check & Update

After ANY of these changes, verify ALL related documents:

| Change | Documents to Update |
|--------|-------------------|
| New DB migration | OPEN_QNA.md (Q44-Q46), Database/_index.md, Schema docs |
| New API endpoint | API Routes doc, Backend/_index.md, OPEN_QNA (Q41-Q43) |
| New config file | Resources/_index.md, OPEN_QNA, architecture docs |
| New model/entity | Database/_index.md, Schema docs |
| Role/permission change | RBAC docs, OPEN_QNA (Q17-Q25), plan file |
| Bug fix | ACTION_LOG.md, Sessions/_index.md, relevant QA |
| APK build | Dashboard/_index.md, Sessions/_index.md, ACTION_LOG |
| Code review | ACTION_LOG.md, Sessions/_index.md |
| New screen built | Android/_index.md, Dashboard/progress |
| Session end | Dashboard/progress, Sessions/_index.md, ACTION_LOG |

## Consequences of Skipping

A schema change that isn't reflected in docs becomes a **future debugging trap** — someone (or the AI itself) will reference stale docs and waste hours chasing ghosts.
