---
name: server-debugging
description: "Server-side debugging for web apps — PHP-FPM, nginx, Apache, permission issues, 500 errors."
version: 1.0.0
platforms: [linux]
metadata:
  hermes:
    tags: [server, debugging, php-fpm, nginx, permissions, 500-errors]
    related_skills: [systematic-debugging]
---

# Server Debugging

## Overview

Server-side 500 errors often look identical from the client but have different root causes. This skill covers systematic diagnosis of PHP web server issues (PHP-FPM + nginx/Apache).

## Quick Diagnostic Checklist

When ANY endpoint returns 500 (including health check):

1. **Is PHP-FPM running?** → `systemctl status php8.2-fpm`
2. **Can CLI work?** → `php artisan tinker --execute="echo 'ok';"` — if CLI works but web doesn't, it's a permission or PHP-FPM config issue.
3. **Check file permissions** → `namei -l /path/to/public/index.php` — every directory in the path needs execute (`x`) for the PHP-FPM user.
4. **Test as www-data** → `sudo -u www-data cat /path/to/config/file.php` — simulates what PHP-FPM sees.
5. **Enable FPM error logging** → See `references/php-fpm-permission-debugging.md`.
6. **Check disk space** → `df -h /` — Laravel needs writable storage.

## PHP-FPM Permission Debugging

When CLI works but web returns 500, the most common cause is that PHP-FPM runs as `www-data` but files are owned by `root` with restrictive permissions.

### The Execute Bit Trap

In Linux, accessing a file inside a directory requires **execute permission (`x`)** on EVERY directory in the path. A directory with `drw-r--r--` (no execute) blocks access regardless of file permissions.

### Diagnostic Workflow

```bash
# 1. Full permission trail
namei -l /home/project/backend/public/index.php

# 2. Test as PHP-FPM user
sudo -u www-data php -r "require '/path/to/vendor/autoload.php'; echo 'OK';"

# 3. Check FPM pool user
grep "^user" /etc/php/8.2/fpm/pool.d/www.conf

# 4. Enable worker output capture
sed -i 's/;catch_workers_output = yes/catch_workers_output = yes/' /etc/php/8.2/fpm/pool.d/www.conf
systemctl restart php8.2-fpm
```

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Path cannot be empty` in LoadConfiguration | Config dir lacks execute bit | `chmod 755 config/` |
| Monolog `StreamHandler::write() failed` | laravel.log not writable by www-data | `chown www-data storage/logs/laravel.log` |
| `bootstrap/cache directory must be present and writable` | Cache dir not writable | `chmod -R 775 bootstrap/cache` |
| `require(...file.php): Permission denied` | File not readable by www-data | `chmod 644 file.php` |

### Fix Command

```bash
find /path/to/project -type d -exec chmod 755 {} \;
find /path/to/project -name "*.php" -exec chmod 644 {} \;
chmod -R 775 /path/to/project/storage
chmod -R 775 /path/to/project/bootstrap/cache
```

## APP_DEBUG for Diagnosis

When stuck, temporarily enable debug mode to see the actual error:

```bash
sed -i 's/APP_DEBUG=false/APP_DEBUG=true/' .env
sed -i 's/APP_ENV=production/APP_ENV=local/' .env
systemctl restart php8.2-fpm
# Test the endpoint — now you'll see the real error
# RESTORE after debugging:
sed -i 's/APP_DEBUG=true/APP_DEBUG=false/' .env
sed -i 's/APP_ENV=local/APP_ENV=production/' .env
```

## Reference

See `references/php-fpm-permission-debugging.md` for the full diagnostic guide with example error traces.