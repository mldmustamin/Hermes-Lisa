# PHP-FPM Permission Fix — Web Dashboard 500 Error

## Symptom
- All web pages return 500 Internal Server Error
- Browser shows empty page or "Server Error"
- `php artisan` CLI works fine
- Health endpoint `curl localhost/api/health` returns 500
- Login redirects work but dashboard shows blank

## Root Cause Cascade (check in order)

### 1. Directory Execute Permission
PHP-FPM runs as `www-data`. Directories need EXECUTE bit (x) for traversal.
```
drw-r--r-- root root config/   ← NO x → PHP-FPM can't read config files
```

**Fix:**
```bash
find /path/to/project -type d -exec chmod 755 {} \;
```

### 2. bootstrap/cache Not Writable
Laravel needs `bootstrap/cache/` writable for compiled configs.
```
Error: "The bootstrap/cache directory must be present and writable"
```

**Fix:**
```bash
chmod -R 775 /path/to/project/bootstrap/cache
chown -R www-data:www-data /path/to/project/bootstrap/cache
```

### 3. storage/logs Not Writable
Monolog crashes when it can't write laravel.log → cascading 500.
```
Error: "StreamHandler: Failed to open stream: Permission denied"
```

**Fix:**
```bash
chmod -R 775 /path/to/project/storage
chown -R www-data:www-data /path/to/project/storage
touch /path/to/project/storage/logs/crash-reports.log  # if using custom channels
```

### 4. View Cache Directory
Compiled Blade views need writable storage.
```
Error: "View [web.budget.inbox] not found"
(But file exists at resources/views/web/budget/inbox.blade.php)
```

**Root cause:** `web/budget/` directory has no execute permission → Laravel can't find views.

**Fix:**
```bash
chmod 755 /path/to/project/resources/views/web/budget
chmod 644 /path/to/project/resources/views/web/budget/*.blade.php
php artisan view:clear
```

### 5. New Controller Files
Files created by root user are readable by www-data only if group/other has read.

**Fix:**
```bash
find /path/to/project/app/Http/Controllers -name "*.php" -exec chmod 644 {} \;
find /path/to/project/app/Models -name "*.php" -exec chmod 644 {} \;
```

## Quick Fix All-in-One
```bash
cd /path/to/project
find . -type d -exec chmod 755 {} \;
find . -name "*.php" -exec chmod 644 {} \;
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
php artisan optimize:clear
```

## Verification
After fix, ALL web pages should return proper titles:
```bash
curl -s http://localhost/ | grep -o "<title>[^<]*</title>"
# → <title>Funds Manager — Login</title> or <title>Redirecting...</title>
```
