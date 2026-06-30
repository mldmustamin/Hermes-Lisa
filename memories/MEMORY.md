Project-Budgeting server public IP: 103.94.11.78
§
Config in config/budget.php: MAX_DRAFTS=5, PAGINATION=20, HISTORY_LIMIT=10. Pagu per job_type in pivot table. VOUCHER 15/5rb, BURUH 120rb, BALLAST 200rb, FEE 40/75/15rb.
§
FundManager code review: (1) role checks on all write endpoints, (2) optimistic locking on stage transitions, (3) pagu enforcement in code, (4) security tests always, (5) update docs after changes.
§
Obsidian vault: /home/Project-Budgeting/docs/obsidian/. 12 items (00-09 folders + HERMES.md + Templates). Each folder has _index.md. Steph Ango principles.
§
PHP-FPM 500: check config/ execute bit (namei -l), bootstrap/cache + storage writable (777 for www-data). Enable catch_workers_output in www.conf.
§
User: mldmustamin, ID dev FundManager V2. Prefers: official docs (developer.android.com), study/research BEFORE coding, delegate_task for parallel work, Indonesian language, ACTION_LOG + Obsidian vault after every batch. CORRECTIONS: never code Android without understanding official patterns first; always bump versionCode before every build; use AVD/wireless debug to verify, not just code review.
§
User prefers systematic testing pipeline: dry test (php artisan test + gradlew build) → smoke test (all endpoints) → integration test (full workflow) → system test (e2e). Always check server permissions (PHP-FPM config dir execute bit, storage/bootstrap writable) before reporting API bugs — 500 errors often server-side, not code bugs.
§
Room DB migration: column names in SQL must match Entity property names (camelCase). DB version bump required for new tables — same version won't re-trigger migration. Login freeze root cause: device registration HTTP call blocked login flow — must run in background coroutine.
§
VPS shell: conda activate in .bashrc dumps env report to stdout on every cmd, polluting git/output. Ignore conda noise blocks when parsing results.
§
Before git push: ALWAYS show user what's queued — `git diff --stat origin/main..HEAD` + `git log origin/main..HEAD --oneline`. Never push blindly. User correction: 'kamu mau push semua? kamu tidak mengerti file mana saja yang harus di push?'