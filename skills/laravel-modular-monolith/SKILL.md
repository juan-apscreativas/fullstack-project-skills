---
name: laravel-modular-monolith
description: >
  This skill should be used when the user asks to "work on the Laravel backend", "create a PHP module", "configure Laravel Sail", "write a Pest test", "set up Scramble API docs", "implement a ModuleApi interface", "configure Reverb WebSockets", "design module boundaries", or "follow Laravel architecture conventions".
---

# Laravel Modular Monolith Expert

Expert in building maintainable Laravel 12 applications using a modular monolith architecture. Modules own their data, logic, and API surface. Communication between modules happens only through explicit contracts — never through direct class imports or shared database queries.

> **Documentation principle:** The project's CLAUDE.md, module CLAUDE.md files, ADRs, and Scramble output MUST be updated whenever this skill is applied. Code changes without documentation updates are incomplete.

## Core Principles

1. **Modules are the unit of organization** — not controllers, not services. A module = a bounded business context.
2. **Modules never import each other's internals** — only `ModuleApi` interfaces and `Shared/Events` contracts.
3. **No cross-module Eloquent relations** — save the foreign ID, query via ModuleApi.
4. **Business logic lives in services and models** — controllers are thin dispatchers.
5. **Result pattern for business errors** — exceptions are for infrastructure failures only.
6. **Sail is the only execution environment** — never run commands outside Docker.
7. **Scramble is the API contract** — keep Form Requests and Resources well-typed so Scramble generates accurate OpenAPI 3.1.0 output.
8. **Tests are non-negotiable** — every use case needs at least one Pest feature test.
9. **PSR-12 enforced by Pint** — `sail pint --test` must pass before every commit.
10. **Documentation in the same commit as code** — CLAUDE.md updates are not optional.

## Capabilities

- Designing module boundaries from a PRD or feature description
- Creating full module structure at Nivel 1 / 2 / 3
- Implementing Eloquent models with Value Objects and business invariants
- Writing Services that return `Result` types and coordinate domain events
- Defining ModuleApi interfaces and binding them in ServiceProviders
- Implementing domain events and cross-module listeners
- Writing Pest PHP tests (feature, unit, integration) using functional syntax
- Configuring Scramble for accurate OpenAPI documentation
- Setting up Reverb channels for WebSocket broadcasts
- Configuring Nightwatch for observability
- Managing database migrations scoped to modules
- Structuring seeders per environment (Local, Testing)
- Following Conventional Commits and branch naming conventions

## Requirements

- Laravel 12.x (latest stable)
- PHP 8.3+ (via Sail — never host PHP)
- PostgreSQL (always — no MySQL, no SQLite in production)
- Laravel Sail (Docker — mandatory dev environment)
- Pest PHP (`pestphp/pest`)
- Laravel Pint (included in Laravel — PSR-12 preset `laravel`)
- Scramble (`dedoc/scramble`) — OpenAPI 3.1.0
- Laravel Reverb — WebSocket server
- Laravel Echo — WebSocket client (frontend)
- Laravel Nightwatch — monitoring

## Patterns

### Module Structure (Nivel 2 — Standard)
```
app/Modules/[Module]/
├── CLAUDE.md
├── [Module]ServiceProvider.php
├── [Module]ModuleApi.php          # interface (only if consumed by other modules)
├── Models/
│   └── [Entity].php               # Eloquent + Value Objects + invariants
├── Services/
│   └── [Module]Service.php        # one public method = one use case
├── Http/
│   ├── Controllers/[Module]Controller.php
│   ├── Requests/[Action][Entity]Request.php
│   └── Resources/[Entity]Resource.php
├── Events/
│   └── [SomethingHappened].php
├── Listeners/
│   └── [HandleExternalEvent].php
├── Routes/
│   └── api.php
├── Database/
│   ├── Migrations/
│   ├── Factories/
│   └── Seeders/
│       ├── [Module]LocalSeeder.php
│       └── [Module]TestingSeeder.php
└── Tests/
    ├── Unit/
    └── Feature/
```

### ModuleApi Contract
```php
// app/Modules/Auth/AuthModuleApi.php
interface AuthModuleApi
{
    public function getUserById(int $id): ?UserDto;
    public function userExists(int $id): bool;
}

// app/Modules/Auth/AuthServiceProvider.php
$this->app->bind(AuthModuleApi::class, AuthService::class);

// app/Modules/Blog/Services/BlogService.php
public function __construct(private AuthModuleApi $auth) {}
// ✅ Injects interface — never the concrete AuthService
```

### Result Pattern
```php
// app/Shared/Base/Result.php
final class Result
{
    private function __construct(
        public readonly bool $success,
        public readonly mixed $value = null,
        public readonly ?string $error = null,
    ) {}

    public static function ok(mixed $value = null): self { ... }
    public static function fail(string $error): self { ... }
}

// In a service:
public function register(array $data): Result
{
    if (User::where('email', $data['email'])->exists()) {
        return Result::fail('email_taken');
    }
    $user = User::create($data);
    return Result::ok($user);
}

// In the controller:
$result = $this->service->register($request->validated());
if (!$result->success) {
    return response()->json(['error' => $result->error], 409);
}
return new UserResource($result->value);
```

### Pest Feature Test
```php
uses(RefreshDatabase::class);

it('creates an order for authenticated user', function () {
    $user = User::factory()->create();

    $response = actingAs($user)
        ->postJson('/api/orders', ['product_id' => 1, 'quantity' => 2]);

    $response->assertStatus(201);
    expect(Order::where('user_id', $user->id)->count())->toBe(1);
});
```

### Domain Event
```php
// Emitter (after completing the operation):
event(new OrderPlaced($order->id, $order->user_id, $order->total));

// Listener in another module registers in its own ServiceProvider:
$this->app['events']->listen(OrderPlaced::class, SendOrderConfirmationEmail::class);
```

### WebSocket Broadcast (Reverb)
```php
// Channel naming: [module].[resource].[id]
class OrderStatusUpdated implements ShouldBroadcast
{
    public function broadcastOn(): Channel
    {
        return new PrivateChannel("orders.{$this->order->user_id}");
    }
}
```

## Anti-Patterns

### ❌ Cross-Module Eloquent Relationship
```php
// WRONG — imports Auth model inside Blog module
public function user(): BelongsTo
{
    return $this->belongsTo(\App\Modules\Auth\Models\User::class);
}
```

### ❌ Logic in Controllers
```php
// WRONG — business rule in controller
public function store(Request $request): JsonResponse
{
    if (Order::where('user_id', auth()->id())->where('status', 'pending')->count() > 3) {
        return response()->json(['error' => 'Too many pending orders'], 422);
    }
    // ... more logic
}
```

### ❌ PHPUnit Syntax in Pest
```php
// WRONG — PHPUnit class-based test
class OrderTest extends TestCase
{
    public function test_order_creation(): void
    {
        $this->assertTrue(true);
    }
}
```

### ❌ Commands Outside Sail
```bash
# WRONG
php artisan migrate
composer require some/package
./vendor/bin/pest
```

### ❌ Silencing Nightwatch-Catchable Exceptions
```php
try {
    $this->externalService->charge($amount);
} catch (\Exception $e) {
    // WRONG — swallowed silently
}
```

## Related Skills

Works well with: `prd-analysis`, `backend-module-dev`, `api-contract-sync`, `project-orchestration`

## When to Use

This skill applies when working in the Laravel backend repository — creating modules, implementing services, writing tests, configuring Sail, managing database migrations, or designing inter-module communication patterns. For frontend work, use `nextjs-feature-based`. For cross-repo coordination, use `api-contract-sync`.

## Coexistence with Superpowers

This skill activates when working in the Laravel backend repository.
Superpowers has no equivalent — it is technology-agnostic and does not define Laravel architecture conventions.
This skill complements any Superpowers phase by applying Laravel Modular Monolith conventions.

If Superpowers is not installed, this skill works identically.
