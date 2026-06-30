# Laravel 11 — Common Production Bug Patterns

## 1. CSRF 419 in Web Tests

### Symptom

Web POST tests return HTTP 419 even with `$this->actingAs($user)`.

### Root Cause

Laravel 11 applies CSRF middleware to web routes by default. Test environment does not bypass it automatically. GET requests work (no CSRF needed), but POST/PUT/DELETE fail.

### Fix

Add to the test class `setUp()`:

```php
use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken;

protected function setUp(): void
{
    parent::setUp();
    $this->withoutMiddleware(VerifyCsrfToken::class);
}
```

Alternative: use `$this->withCsrfToken()` helper or include the CSRF token in POST data, but `withoutMiddleware` is simplest for API-level web route tests.

### Test structure pattern (from ApprovalQueueTest):

```php
class ApprovalQueueTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        $this->seed(RolePermissionSeeder::class);
        $this->withoutMiddleware(VerifyCsrfToken::class);
    }
    // ... tests using $this->actingAs($user)->post(...)
}
```

---

## 2. Route Model Binding Returns 500 Instead of 404 (APP_DEBUG=false)

### Symptom

Test expects 404 but gets 500 when requesting a non-existent UUID.

### Root Cause

Implicit route model binding with `getRouteKeyName()` returning `'uuid'` works in dev mode (`APP_DEBUG=true`) because Laravel's exception handler renders `ModelNotFoundException` as a readable page. In production (`APP_DEBUG=false`), the same exception may render as a generic 500 error depending on how the exception handler is configured.

### Fixed Pattern

Replace implicit binding with explicit lookup + null check:

```php
// ❌ BAD — implicit binding (500 on missing UUID in production)
public function show(Attachment $attachment, Request $request): mixed
{
    // Laravel throws ModelNotFoundException if not found
    // → 500 when APP_DEBUG=false
}

// ✅ GOOD — explicit lookup with 404
public function show(string $uuid, Request $request): JsonResponse
{
    $attachment = Attachment::where('uuid', $uuid)->first();
    if (! $attachment) {
        return response()->json(['error' => 'Not found.'], 404);
    }
    // ... proceed with $attachment
}
```

### When to apply

Any controller where:
1. Route uses implicit model binding (`{model}` in URL, `Model $model` in signature)
2. Model uses `getRouteKeyName()` returning `'uuid'` (not default `id`)
3. Production config has `APP_DEBUG=false`
4. The 500 error was observed instead of expected 404
