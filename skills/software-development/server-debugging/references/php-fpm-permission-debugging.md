# PHP-FPM Permission Debugging — Full Guide

## The Core Principle

PHP-FPM runs as `www-data` (or configured user). When files are created by `root` (via CLI tools like `php artisan`), the PHP-FPM worker may not be able to read them. **Always test with `sudo -u www-data` when debugging FPM issues.**

## The Execute Bit Trap

Linux directory permissions:
- `r` (read) — list files in directory
- `w` (write) — create/delete files in directory  
- `x` (execute) — **access** files inside the directory

A directory with `drw-r--r--` (644) blocks ALL access to files inside it, regardless of individual file permissions. This is the most common hidden cause of "500 Server Error" on Laravel apps after deployment or after `chmod` commands that only target files (`find -type f -exec chmod 640 {} \;`).

## Real-World Error Trace (This Session)

```
PHP Fatal error: Uncaught ValueError: Path cannot be empty
at LoadConfiguration.php:97
  -> require $path  (where $path is empty because glob couldn't read config dir)

PHP Fatal error: Monolog StreamHandler::write() failed
  -> laravel.log not writable by www-data

Error: bootstrap/cache directory must be present and writable
  -> Cache dir not accessible/writable
  
Error: require(config/budget.php): Failed to open stream: Permission denied
  -> File permission 600 (owner-only)
```

## Step-by-Step Diagnosis

### 1. Verify PHP-FPM is running
```bash
systemctl status php8.2-fpm
ss -tlnp | grep php
```

### 2. Check the full permission trail
```bash
namei -l /home/Project-Budgeting/backend/public/index.php
```
Look for any directory WITHOUT `x` (execute) permission.

### 3. Test file access as www-data
```bash
sudo -u www-data cat /home/Project-Budgeting/backend/config/budget.php
sudo -u www-data php -r "require '/home/Project-Budgeting/backend/vendor/autoload.php'; echo 'OK';"
```

### 4. Check PHP-FPM pool configuration
```bash
grep -E "^user|^group|^listen|catch_workers_output" /etc/php/8.2/fpm/pool.d/www.conf
```

### 5. Enable FPM error logging
Edit `/etc/php/8.2/fpm/pool.d/www.conf`:
```ini
catch_workers_output = yes
php_admin_value[error_log] = /var/log/fpm-php.www.log
```
Then:
```bash
touch /var/log/fpm-php.www.log
chown www-data:www-data /var/log/fpm-php.www.log
systemctl restart php8.2-fpm
```

### 6. Trigger a request and check the log
```bash
curl -s http://localhost/api/health
tail -20 /var/log/fpm-php.www.log
```

### 7. Temporarily enable debug mode for more detail
```bash
cd /path/to/project
sed -i 's/APP_DEBUG=false/APP_DEBUG=true/' .env
sed -i 's/APP_ENV=production/APP_ENV=local/' .env
systemctl restart php8.2-fpm
curl -s http://localhost/api/health  # Now shows full error with stack trace
# RESTORE:
sed -i 's/APP_DEBUG=true/APP_DEBUG=false/' .env
sed -i 's/APP_ENV=local/APP_ENV=production/' .env
```

## The Fix

```bash
# Make ALL directories traversable
find /path/to/project -type d -exec chmod 755 {} \;

# Make ALL PHP files readable
find /path/to/project -name "*.php" -exec chmod 644 {} \;

# Make storage and cache writable
chmod -R 775 /path/to/project/storage
chmod -R 775 /path/to/project/bootstrap/cache

# Optional: set group ownership to www-data
chown -R root:www-data /path/to/project
```

## Quick Test After Fix
```bash
curl -s http://localhost/api/health
# Should return: {"status":"ok"}
```
