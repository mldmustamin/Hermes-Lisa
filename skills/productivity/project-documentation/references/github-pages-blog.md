# GitHub Pages — Repo as Blog

Enable GitHub Pages for `/docs` folder programmatically.

## Enable via API (no `gh` CLI needed)

```bash
GITHUB_TOKEN="<token>"
OWNER="<owner>"
REPO="<repo>"

curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/pages" \
  -d '{"source":{"branch":"main","path":"/docs"}}'
```

## Check build

```bash
curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/pages/builds/latest" \
  | python3 -m json.tool
```

Status: `building` → `built`. ~30s build time.

## URL

`https://<owner>.github.io/<repo>/`

## Strategy

- Single `docs/index.md` as blog homepage — wikilinks to all Obsidian `_index.md` pages
- No Jekyll config, no theme, no build step — GitHub renders markdown natively
- Obsidian vault folders become blog sections
- Pitfall: folder names with spaces need `%20` in URLs (e.g. `00%20-%20Dashboard`)
