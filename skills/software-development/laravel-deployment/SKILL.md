---
name: laravel-deployment
category: software-development
description: Deploy Laravel 11 applications to production — install, configure DB/Redis, run migrations, build assets, optimize, test, and smoke-check.
---

# Laravel Production Deployment

Class-level skill for deploying a Laravel application to a production/staging environment. Covers the full pipeline from infrastructure setup through smoke testing.

## When to Load

- User says "deploy Laravel" / "set up production" / "build server" / "setup staging"
- User says "server error after deploy" / "migration failed" / "500 error in production"
- User asks to set up PostgreSQL + Redis for a Laravel backend

## Prerequisites

- PHP 8.2+ with extensions: `pdo_pgsql`, `pgsql`, `redis`, `mbstring`, `xml`, `curl`, `gd`, `bcmath`, `intl`
- Composer, Node.js, npm
- PostgreSQL 14+ and Redis (install if missing)
- Laravel 11 project with `composer.json`, `.env.example`, `artisan`

---

## Step 1 — Provision Infrastructure

```bash
# Install PostgreSQL + Redis if missing
apt-get install -y postgresql postgresql-client redis-server

# Install PHP extensions
apt-get install -y php8.2-pgsql php8.2-redis

# Start services
pg_ctlcluster 14 main start
redis-server --daemonize yes
```

**Verify:**
```bash
pg_isready                    # "accepting connections"
redis-cli ping                # "PONG"
php -m | grep -iE 'pgsql|redis'  # "pdo_pgsql", "pgsql", "redis"
```

---

## Step 2 — Create Database & User

```bash
su - postgres -c "psql -c \"CREATE USER myapp WITH PASSWORD 'secret';\""
su - postgres -c "psql -c \"CREATE DATABASE myapp_production OWNER myapp;\""
su - postgres -c "psql -c \"GRANT ALL PRIVILEGES ON DATABASE myapp_production TO myapp;\""
```

---

## Step 3 — Configure `.env`

Copy from `.env.example`, then set:

```env
APP_ENV=production
APP_DEBUG=false
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=myapp_production
DB_USERNAME=myapp
DB_PASSWORD=secret
QUEUE_CONNECTION=redis
CACHE_STORE=redis
SESSION_DRIVER=redis
```

```bash
php artisan key:generate --force
```

**Check storage directories exist:**
```bash
mkdir -p storage/framework/views storage/framework/cache/data storage/framework/sessions
chmod -R 775 storage bootstrap/cache
```

---

## Step 4 — Install Dependencies

```bash
composer install --no-interaction --prefer-dist
npm install
```

> **Pitfall:** `composer install` may fail on `post-autoload-dump` if `APP_KEY` is not set. Generate key first, then run `php artisan package:discover` manually.

---

## Step 5 — Fix Migration Ordering (Common Pitfall)

Laravel runs migrations in timestamp order. When multiple migrations share the same timestamp **and** have cross-table FK dependencies, migrations fail because the referenced table hasn't been created yet.

**Symptoms:** `SQLSTATE[42P01]: Undefined table: 7 ERROR: relation "X" does not exist (Connection: pgsql...`

**Root cause:** Files like `2026_06_27_142227_create_transactions_table.php` and `2026_06_27_142227_create_attachments_table.php` sort alphabetically. If `attachments` has a FK to `transactions` but runs first, it fails.

**Fix:** Rename migration files to establish correct dependency order:

```bash
# Audit FK dependencies in each migration file first
# grep for 'foreign' or 'constrained' in each migration to build the dependency graph

# Example fix pattern — rename to bump dependent migrations later
mv 2026_06_27_142227_create_attachments_table.php  2026_06_27_142230_create_attachments_table.php
mv 2026_06_27_142227_create_transactions_table.php  2026_06_27_142229_create_transactions_table.php
```

**Dependency order rule:** tables with FK references must run AFTER the tables they reference. Build the graph first, then assign timestamps.

```bash
# After fixing, reset DB and retry
su - postgres -c "psql -c \"DROP DATABASE myapp_production;\""
su - postgres -c "psql -c \"CREATE DATABASE myapp_production OWNER myapp;\""
php artisan migrate --force
```

---

## Step 6 — Migrate & Seed

```bash
php artisan migrate --force
php artisan db:seed --class=StagingSeeder --force
php artisan storage:link --force
```

---

## Step 7 — Build Frontend Assets

```bash
npx tailwindcss -i ./resources/css/app.css -o ./public/css/app.css --minify
npx vite build
```

> **Pitfall on Vite:** `npx vite build` may hang or start a dev server if the environment is unusual. Run with a timeout or in background mode. If it hangs, retry.

---

## Step 8 — Production Optimization

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

> **Pitfall:** Ensure storage directories exist BEFORE running config:cache. Missing `storage/framework/views/` causes: `InvalidArgumentException: Please provide a valid cache path.`

---

## Step 9 — Queue Worker (Horizon)

```bash
# Start (background process)
php artisan horizon

# Verify
php artisan horizon:status   # Should say "Horizon is running."

# Production: supervise with systemd or supervisord
```

---

## Step 10 — Run Tests

```bash
php artisan test
```

**Pitfall — CSRF in web tests (Laravel 11):** POST requests to web routes return HTTP 419 because CSRF middleware is active by default in tests. Fix:

```php
// In setUp() of the test class:
$this->withoutMiddleware(
    \Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class
);
```

**Pitfall — 500 instead of 404 from route model binding (APP_DEBUG=false):** When `APP_DEBUG=false`, implicit route model binding may throw an unhandled exception instead of returning 404. Fix: use explicit lookup with null check:

```php
public function show(string $uuid, Request $request): JsonResponse
{
    $model = MyModel::where('uuid', $uuid)->first();
    if (! $model) {
        return response()->json(['error' => 'Not found.'], 404);
    }
    // ...
}
```

---

## Step 11 — Production Web Server (Nginx + PHP-FPM)

For production, replace `php artisan serve` (dev only) with Nginx + PHP-FPM.

### 11a — Install & start PHP-FPM

```bash
apt-get install -y php8.2-fpm
systemctl start php8.2-fpm
systemctl enable php8.2-fpm
ls /run/php/php8.2-fpm.sock   # verify socket exists
```

### 11b — Configure Nginx site

Create `/etc/nginx/sites-available/myapp`:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    server_name _;
    
    root /path/to/backend/public;
    index index.php index.html;
    
    charset utf-8;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
    
    access_log /var/log/nginx/myapp-access.log;
    error_log  /var/log/nginx/myapp-error.log error;
    
    sendfile off;
    client_max_body_size 100m;
    
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 11c — Enable & restart

```bash
rm -f /etc/nginx/sites-enabled/default
ln -sf /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
nginx -t                       # verify config
systemctl restart nginx
```

### 11d — Cloud firewall check

> **Cloud firewall pitfall:** Most VPS/cloud providers have an **external firewall** (separate from the server's UFW/iptables) that only opens ports **80** and **443** by default. Before configuring a non-standard port, verify which ports the cloud firewall actually allows by running curl from an external machine:
>
> ```bash
> for port in 80 443 8080 9090 8011; do
>   result=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 3 http://<PUBLIC_IP>:$port/health 2>&1)
>   echo "Port $port: $result"
> done
> ```
>
> If port 80 works but 8080/9090 show HTTP 000, the cloud firewall is blocking them. You must either:
> - Use port 80/443 only, or
> - Open the port in the cloud provider's firewall panel (not just UFW on the server)

### 11e — Cockpit conflict on port 9090

Many Ubuntu VPS have **Cockpit** (web-based server management) pre-installed and listening on port **9090** via `cockpit.socket` (systemd socket-activated). Before using port 9090 for your app, check:

```bash
ss -tlnp | grep 9090
systemctl status cockpit.socket
```

If Cockpit is using it and you need the port, disable it:

```bash
systemctl stop cockpit.socket
systemctl stop cockpit.service
systemctl disable cockpit.socket
systemctl disable cockpit.service
systemctl mask cockpit.service
systemctl mask cockpit.socket
```

> **Note:** This is permanent. Cockpit will not survive reboot.

### 11f — Local port forwarding with iptables DNAT (optional)

If cloud firewall only opens port 80 but you need the app behind a different internal port, use iptables DNAT:

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 16661
```

Make Nginx listen on the target port:

```nginx
listen 16661;
listen [::]:16661;
```

> **Limitation:** PREROUTING only applies to packets from external hosts, not locally-generated packets. For local testing, use `curl http://localhost:16661` directly.

---

## Step 12 — Smoke Test

### Via Nginx (production)

```bash
# Health check
curl -s -o /dev/null -w "HTTP %{http_code}" http://localhost/up

# Login (auto-detects email or employee_id)
curl -s -X POST http://localhost/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -d '{"login":"admin@test.com","password":"admin"}' | python3 -m json.tool

# Authenticated endpoints
curl -s http://localhost/api/v1/auth/me \
  -H 'Authorization: Bearer <token>' \
  -H 'Accept: application/json'

# Unauthorized access check
curl -s http://localhost/api/v1/projects \
  -H 'Accept: application/json' | grep -q "Unauthenticated" && echo "Auth blocked OK"

# Web dashboard page
curl -s -o /dev/null -w "%{http_code}" http://localhost/login
```

### Via artisan serve (dev/staging)

```bash
php artisan serve --port=8080 --host=0.0.0.0 &
# Then use http://localhost:8080/... for all curl commands above
```

---

## Step 13 — Post-Deploy: Audit RBAC Authorization

After deployment, inline role checks in controllers may be too restrictive or too permissive — users report "logic app kacau" when roles can't perform actions they should, or can perform actions they shouldn't.

**Audit pattern — for every controller that has an `authorize*()` method:**

```bash
# Find all inline role checks
grep -rn 'hasRole' app/Http/Controllers/ --include='*.php'
```

For each match, verify:
1. Does the allowed role list match the business requirements document (PRD / product scope)?
2. Are there roles that SHOULD be allowed but AREN'T? (too restrictive — e.g. FINANCE_MANAGER blocked from project creation)
3. Are there roles that SHOULDN'T be allowed but ARE? (too permissive — e.g. FIELD_ENGINEER can create projects)
4. Does the mobile app have corresponding UI gating for the same action?

**Android cross-check — role-based UI gating:**

When a backend endpoint is role-restricted, the Android app must also hide the triggering UI element for unauthorized roles. Pattern:

```kotlin
// 1. Add canDoX field to UiState data class
data class ProjectListUiState(
    val canCreateProject: Boolean = false,
    // ...
)

// 2. Observe session roles in ViewModel
private fun observeCanCreateProject() {
    viewModelScope.launch {
        sessionManager.activeSession.collectLatest { session ->
            val canCreate = session?.roles?.any { 
                it in listOf("OWNER", "ADMIN", "FINANCE_MANAGER", "SUPERVISOR") 
            } ?: false
            _uiState.update { it.copy(canCreateProject = canCreate) }
        }
    }
}

// 3. Conditional rendering in Screen composable
if (uiState.canCreateProject) {
    Button(onClick = { showAddDialog = true }) { Text("Add") }
}
```

**Smoke test — verify each role boundary:**

```bash
# Finance Manager should be able to create projects
curl -s -X POST :8080/api/v1/projects -H 'Authorization: Bearer <fm_token>' \
  -d '{"name":"test"}' -w "\nHTTP: %{http_code}"
# Expected: 201

# Field Engineer should NOT be able to create projects
curl -s -X POST :8080/api/v1/projects -H 'Authorization: Bearer <fe_token>' \
  -d '{"name":"test"}' -w "\nHTTP: %{http_code}"
# Expected: 422 with role error message

# Field Engineer SHOULD be able to create transactions
curl -s -X POST :8080/api/v1/transactions -H 'Authorization: Bearer <fe_token>' \
  -d '{"type":"FUND_IN","project_uuid":"...","date":"...","reported_amount":500,"real_amount":500}' \
  -w "\nHTTP: %{http_code}"
# Expected: 201
```

> **Pitfall — RolePermissionSeeder without permissions:** If the seeder only creates `Role` records via `Role::findOrCreate()` without assigning any Spatie permissions, all authorization relies on inline `$user->hasRole()` checks. This is fine for RBAC but means every controller method must independently enforce its own role check — there's no centralized permission gate. Audit every sensitive endpoint.

---

## Step 14 — Post-Deploy: Verify Mobile-Required Endpoints

After deploying, the mobile app may call backend endpoints that were never defined. This causes silent 404s that look like the app is broken.

**Compare Android API calls vs backend routes:**

```bash
# List URLs the mobile app hits:
grep -roh 'ApiConfig\.baseUrl/[a-z-]*' app/src/main/java/ | sort -u

# List backend routes (strip leading /):
cd backend && php artisan route:list --path=api/v1 | grep -oP 'api/v1/\S+' | sort -u

# Spot-check any endpoint that exists in Android but NOT in the backend
```

**Common missing endpoint — change-password:**

```bash
# Test from the server:
curl -s -X POST http://localhost/api/v1/auth/change-password \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{"password":"newpass","password_confirmation":"newpass"}' \
  -w "\nHTTP: %{http_code}"
```

If it returns 404, add the endpoint:

```php
// routes/api.php — in the auth:sanctum group:
Route::post('/change-password', [AuthController::class, 'changePassword']);

// AuthController.php:
public function changePassword(Request $request): JsonResponse
{
    $request->validate([
        'password' => 'required|string|min:6',
        'password_confirmation' => 'required|string|same:password',
    ]);
    $user = $request->user();
    $user->password = Hash::make($request->password);
    $user->password_change_required = false;
    $user->save();
    return response()->json(['message' => 'Password changed successfully.']);
}
```

**Refresh caches after any route change:**

```bash
php artisan config:clear && php artisan route:clear && php artisan view:clear
php artisan config:cache && php artisan route:cache && php artisan view:cache
systemctl restart nginx
```

---

## Pitfalls Summary

| Issue | Symptom | Fix |
|-------|---------|-----|
| Missing storage dirs | `InvalidArgumentException: cache path` | `mkdir -p storage/framework/views storage/framework/cache/data storage/framework/sessions` |
| Migration FK order | `SQLSTATE[42P01]: relation "X" does not exist` | Rename files to correct timestamp order |
| CSRF in web tests | HTTP 419 on POST | `$this->withoutMiddleware(VerifyCsrfToken::class)` in `setUp()` |
| Model binding 500 | 500 instead of 404 | Explicit `where()->first()` with null check |
| `composer install` fails | `package:discover` error | Generate `APP_KEY` first, then run `php artisan package:discover` manually |
| Cloud firewall blocks port | HTTP 000 on non-80/443 ports | Use port 80 or open port in cloud provider panel |
| Cockpit occupies port 9090 | Port 9090 already in use | Disable cockpit.socket/systemd mask |
| iptables DNAT not working locally | PREROUTING rule works from external but not local | Use curl to localhost:target_port directly for local testing |
| Vite build hangs | No output / process hangs | Run in background with timeout or retry |
| Mobile endpoint mismatch | Android calls endpoint, gets 404 | `grep` Android `ApiConfig.baseUrl` calls, compare with `php artisan route:list`, add missing routes |
| Change-password missing | User login forces password change, submit fails silently | Add `POST /api/v1/auth/change-password` route + controller method |
| RBAC: inline role check mismatch | "Logic app kacau" — user can't do allowed action or can do forbidden action | Audit every `hasRole()` call in controllers against business requirements doc; fix allowed role lists; cross-check Android UI gating |
| RBAC: Android missing UI gating | Unauthorized user sees restricted buttons (e.g. "Create Project" for FIELD_ENGINEER) | Add `canDoX` to UiState, observe session roles in ViewModel, conditional rendering in Screen |
| File permissions: config dir no execute | All endpoints 500, health check fails, `Path cannot be empty` in LoadConfiguration | Config directory lost execute bit (e.g. `drw-r--r--`). Fix: `chmod 755 config/`. Debug: `namei -l path/to/file` to trace permissions. See `references/file-permission-debugging.md` |
| File permissions: bootstrap/cache not writable | 500 with `bootstrap/cache directory must be present and writable` | `chmod -R 777 bootstrap/cache` |
| PHP-FPM silent 500 (no error output) | 500 with no message in laravel.log or nginx error log | Enable `catch_workers_output = yes` in `/etc/php/8.2/fpm/pool.d/www.conf` and set `php_admin_value[error_log]`. Restart PHP-FPM, check `/var/log/fpm-php.www.log`. Temporarily set `APP_DEBUG=true` in `.env`. See `references/file-permission-debugging.md` |

---

## Verification Checklist

- [ ] PostgreSQL accepting connections, database exists
- [ ] Redis responding to PING
- [ ] All PHP extensions loaded (`php -m`)
- [ ] `.env` configured with correct `DB_CONNECTION`, redis settings
- [ ] `APP_KEY` generated
- [ ] All 19+ migrations pass without FK errors
- [ ] Seeders run (staging users exist)
- [ ] Config/route/view cached
- [ ] Tailwind + Vite build succeeds
- [ ] Horizon running (`horizon:status`)
- [ ] Full test suite passes (`php artisan test`)
- [ ] Nginx config valid (`nginx -t`)
- [ ] Nginx running
- [ ] Smoke test: health check returns 200
- [ ] Smoke test: login works (email + employee_id)
- [ ] Smoke test: auth/me returns user
- [ ] Smoke test: unauthenticated blocked
- [ ] Smoke test: web login page returns 200
- [ ] Mobile-required endpoints verified (`grep Android URL` vs `php artisan route:list`)
- [ ] change-password endpoint tested (if Android app requires it)
- [ ] RBAC smoke test: each role can/cannot perform expected actions (see `references/post-deploy-rbac-smoke-test.md`)
- [ ] Route cache refreshed after any route addition
- [ ] RBAC: Android app has role-based UI gating for all restricted actions
- [ ] RBAC: smoke tested — each role can do what it should, blocked from what it shouldn't
- [ ] Route cache refreshed after any route addition
