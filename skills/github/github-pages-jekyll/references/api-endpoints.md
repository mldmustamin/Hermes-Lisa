# GitHub Pages API Quick Reference

## Enable Pages from /docs
```
POST /repos/{owner}/{repo}/pages
Body: {"source":{"branch":"main","path":"/docs"}}
```

## Set workflow build type
```
PUT /repos/{owner}/{repo}/pages
Body: {"build_type":"workflow"}
```

## Get Pages status
```
GET /repos/{owner}/{repo}/pages
```

## Get latest build
```
GET /repos/{owner}/{repo}/pages/builds/latest
```

## Get workflow runs
```
GET /repos/{owner}/{repo}/actions/runs?per_page=3
```

## Request a build (legacy only)
```
POST /repos/{owner}/{repo}/pages/builds
```

## All calls authenticated with:
```
Authorization: token ghp_...
Accept: application/vnd.github+json
```
