# Laravel File Permissions — Debugging Checklist

## The Problem

When an agent (running as root) creates PHP files, config files, or Blade views, they default to root ownership with restrictive permissions (600/640). When PHP-FPM runs as `www-data`, it cannot read these files, causing 500 errors, "View not found" errors, or "Path cannot be empty" errors.

This happens EVERY session where new files are created — the pattern repeats because each session starts fresh.

## Quick Fix

```bash
# Fix directories (need execute bit for traversal)
find /path/to/project -type d -exec chmod 755 {} \;

# Fix PHP files (need read for www-data)
find /path/to/project -name "*.php" -exec chmod 644 {} \;

# Fix writable directories (storage + cache)
chmod -R 775 /path/to/project/storage
chmod -R 775 /path/to/project/bootstrap/cache
chown -R www-data:www-data /path/to/project/storage
chown -R www-data:www-data /path/to/project/bootstrap/cache
```

## Symptom → Diagnosis Map

| Symptom | Likely Cause | Check |
|---------|-------------|-------|
| "Server Error" on all endpoints | Config directory no execute bit (`drw-r--r--` instead of `drwxr-xr-x`) | `namei -l path/to/config/file.php` |
| "View [x] not found" | Views directory no execute bit | `namei -l path/to/views/x.blade.php` |
| "Path cannot be empty" (LoadConfiguration) | Config directory no execute bit | Same as above |
| Health endpoint works, auth API fails | DB config readable by root only | Check `.env` ownership |
| "bootstrap/cache must be writable" | Cache dir not writable by www-data | `ls -la bootstrap/cache/` |
| Monolog StreamHandler error (log write fail) | `storage/logs/laravel.log` not writable | `ls -la storage/logs/laravel.log` |
| PHP file returns 500 but artisan tinker works | Individual PHP file 600, group can't read | `sudo -u www-data cat path/to/file.php` |
| "Failed opening required config/budget.php" | New config file 600, www-data can't read | `chmod 644 config/budget.php` |
| Blade view compiled but empty page | `storage/framework/views/` not writable | `chmod 775 storage/framework/views/` |

## Debugging Workflow

1. Enable debug mode temporarily:
   ```bash
   sed -i 's/APP_DEBUG=false/APP_DEBUG=true/' .env
   sed -i 's/APP_ENV=production/APP_ENV=local/' .env
   curl -s http://localhost/path | grep -o "…error message…"
   # Restore
   sed -i 's/APP_DEBUG=true/APP_DEBUG=false/' .env
   sed -i 's/APP_ENV=local/APP_ENV=production/' .env
   ```

2. Check directory traversal:
   ```bash
   namei -l /full/path/to/problem/file.php
   # Look for drw-r--r-- (644 permissions on a directory = NO execute bit)
   # All directories need drwxr-xr-x (755) for www-data to traverse into them
   ```

3. Check if www-data can read the file:
   ```bash
   sudo -u www-data cat /path/to/file.php
   ```

4. Enable PHP-FPM worker output capture (if error is invisible):
   ```bash
   # /etc/php/*/fpm/pool.d/www.conf
   ;catch_workers_output = yes  →  catch_workers_output = yes
   ;php_admin_value[error_log] = /var/log/fpm-php.www.log  → uncomment
   systemctl restart php*-fpm
   ```

## Prevention: After Creating ANY New Files

Run these commands after every batch of file creation by the agent:

```bash
# Fix all PHP files readable
find /path/to/project -newer /tmp/last_marker -name "*.php" -exec chmod 644 {} \;
# Fix all directories traversable  
find /path/to/project -newer /tmp/last_marker -type d -exec chmod 755 {} \;
# Fix writable dirs
chmod -R 775 /path/to/project/storage /path/to/project/bootstrap/cache
chown -R www-data:www-data /path/to/project/storage /path/to/project/bootstrap/cache
```
