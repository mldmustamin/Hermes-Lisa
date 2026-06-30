# File Permission Debugging — Laravel Production

## Symptom
All PHP endpoints return HTTP 500. Health check (`/api/health`) returns 500. No error shown in browser.

## Root Cause Pattern

Laravel runs via PHP-FPM as user `www-data`. Every directory in the path from `/` to the PHP file must have execute (`x`) permission for `www-data`. When a single directory loses its execute bit, the error cascades.

## The Permission Cascade

Errors appear in this exact order — fix them one at a time:

### 1. Config Directory — No Execute
```
ERROR: "Path cannot be empty" in LoadConfiguration.php:97
ERROR: "Failed opening required config/budget.php"
```
**Check:** `namei -l /path/to/app/config/budget.php`
**Pattern:** `drw-r--r-- root root config/` (no execute bit)
**Fix:** `chmod 755 config/`

### 2. bootstrap/cache — Not Writable
```
ERROR: "bootstrap/cache directory must be present and writable"
ERROR: "file_put_contents(...): Failed to open stream: Permission denied"
```
**Check:** `ls -ld bootstrap/cache/`
**Fix:** `chmod -R 775 bootstrap/cache/ && chown -R www-data:www-data bootstrap/cache/`

### 3. storage/logs/laravel.log — Not Writable
```
ERROR: Monolog StreamHandler — can't open log file in append mode
ERROR: "Failed to open stream: Permission denied"
```
**Check:** `sudo -u www-data echo test >> storage/logs/laravel.log`
**Fix:** `chown www-data:www-data storage/logs/laravel.log && chmod 664 storage/logs/laravel.log`

### 4. storage/framework/views — Not Writable
```
ERROR: "file_put_contents(...): Failed to open stream" in compiled view
```
**Check:** `ls -ld storage/framework/views/`
**Fix:** `chmod -R 775 storage/framework/views/ && chown -R www-data:www-data storage/framework/views/`

### 5. New View Directory — No Execute
```
ERROR: "View [x.y.z] not found" but file exists on disk
```
**Check:** `namei -l resources/views/web/newdir/index.blade.php` — look for `drw-r--r--`
**Fix:** `find resources/views/web -type d -exec chmod 755 {} \;`

### 6. New Controller File — No Read
```
ERROR: "include(Controller.php): Failed to open stream: Permission denied"
```
**Fix:** `find app/Http/Controllers -name "*.php" -exec chmod 644 {} \;`

## Debugging Process

1. **Enable debug temporarily:**
   ```bash
   sed -i 's/APP_DEBUG=false/APP_DEBUG=true/' .env
   sed -i 's/APP_ENV=production/APP_ENV=local/' .env
   ```

2. **Check the FIRST error in laravel.log** (not the cascaded ones):
   ```bash
   grep "ERROR" storage/logs/laravel.log | tail -5
   ```

3. **Trace directory permissions with `namei`:**
   ```bash
   namei -l /path/to/app/public/index.php
   # Look for every directory with drw-r--r-- instead of drwxr-xr-x
   ```

4. **Test as www-data directly:**
   ```bash
   sudo -u www-data php -r "require '/path/to/app/vendor/autoload.php'; echo 'OK';"
   ```

5. **Enable PHP-FPM worker output** for future debugging:
   ```ini
   # /etc/php/8.2/fpm/pool.d/www.conf
   catch_workers_output = yes
   php_admin_value[error_log] = /var/log/fpm-php.www.log
   ```
   ```bash
   systemctl restart php8.2-fpm
   ```

## One-Shot Fix After Deployment

```bash
find /path/to/app -type d -exec chmod 755 {} \;
find /path/to/app -name "*.php" -exec chmod 644 {} \;
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
php artisan optimize:clear
```

## Pitfall: Cache After Permission Fix

`php artisan optimize` recreates cached files as root. After every `optimize`, re-apply:
```bash
chown -R www-data:www-data bootstrap/cache/
```
