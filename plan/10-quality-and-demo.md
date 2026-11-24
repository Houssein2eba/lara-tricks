# Step 10: Quality, accessibility, and demo release

**Days:** 46-50 (Week 10)  
**Branches:** `test/core-flow-hardening`, then `chore/demo-release`

## Goal

Make the learning project reliable enough to demonstrate and explain. Do not add
new major features in the final three days.

## Day 46: `test/core-flow-hardening`

- Start from updated `main`.
- Review the complete customer flow: browse -> cart -> place order -> track
  status -> receive notification.
- Add/finish Laravel feature tests for validation, ownership, staff authorization,
  order transitions, and notifications.
- Test the optional AI provider failure path with `Http::fake()` only, if that
  optional feature was kept.

**Checkpoint:** A fresh test database can run migrations and the critical
feature tests pass.

## Day 47: Quality and accessibility

- Check keyboard navigation, visible focus, labels, contrast, loading/error
  states, and mobile layout.
- Inspect the browser console, Laravel logs, queue/notification behavior, and
  Compose health.
- Fix only defects in the planned app; record unrelated ideas as future work.

**Checkpoint:** No known blocker remains in the core order flow; checks and
their exact commands are recorded.

## Day 48: Verify configuration and recovery

- Run all automated checks, including PHP tests, Pint, PHPStan, frontend checks,
  TypeScript checks, and production asset build.
- Re-test a migration from a clean test database; do not delete the local
  development volume to do this.
- Confirm optional services can be stopped without breaking the core app.

**Checkpoint:** The core app works with optional AI/realtime services stopped,
and failures are visible rather than success-shaped.

## Day 49: `chore/demo-release`

- Merge the test branch, then create the release/demo branch from updated
  `main`.
- Update the README with setup, environment variables, ports, demo data, and
  start/stop instructions. Never include demo passwords or secrets.
- Confirm `.env`, AI keys, and real customer data are not tracked.

**Checkpoint:** A new developer can reproduce the core demo from the README.

## Day 50: Demo and retrospective

- Walk through browse -> cart -> checkout -> staff status change -> customer
  order history.
- Explain one database decision, one authorization rule, one Inertia flow, and
  one Vue component contract without reading notes.
- Record three follow-up improvements and one thing you would redesign.
- Tag a demo commit only if the repository has an established remote/release
  convention.

**Checkpoint:** The course ends with a working demo and a concrete next
learning project, not a claim that one month guarantees mastery.

## Final acceptance checklist

- Customer can browse items, maintain a cart, and place a demo order.
- Staff-only workflow updates statuses with authorization and valid transitions.
- Customer sees their own updated order and notifications only.
- Local email is inspectable in Mailpit; no production mail is sent.
- If retained, the optional AI helper is rate-limited, server-keyed, and cannot
  access or mutate orders.
- Fresh migrations, seed data, tests, frontend checks, and build pass.
- The main branch contains all merged topic branches and remains runnable.

## Branch closeout

Merge `test/core-flow-hardening`, then `chore/demo-release`. Keep each branch
small and reviewable; do not merge incomplete work to `main`.

## Tutorial: final checks by file area

### Test file map

```text
tests/Feature/MenuPageTest.php
tests/Feature/CheckoutTest.php
tests/Feature/StaffOrderTest.php
tests/Feature/OrderNotificationsTest.php
tests/Feature/AiMenuAssistantTest.php       optional
resources/js/pages/               storefront, customer orders, staff queue
resources/js/components/          menu cards, cart, notification bell, AI UI
README.md                          setup and demo instructions
```

Use factories and `RefreshDatabase` for database-backed feature tests. Test
observable behavior and authorization, not private helper implementation.

### Example ownership test

In `tests/Feature/StaffOrderTest.php`, the important pattern is to make two
different users and assert that one cannot see the other's order:

```php
public function test_customer_cannot_view_another_customers_order(): void
{
    $owner = User::factory()->create();
    $otherCustomer = User::factory()->create();
    $order = Order::factory()->for($owner)->create();

    $this->actingAs($otherCustomer)
        ->get(route('orders.show', $order))
        ->assertForbidden();
}
```

Place this method in a PHPUnit class extending the repository's
`Tests\TestCase`, with `RefreshDatabase` and the required imports. Adapt the
factory and route names to the implementation. Add a matching positive test
proving the owner can view their own order.

### Example checkout test

In `tests/Feature/CheckoutTest.php`, use the existing PHPUnit class style:

```php
<?php

namespace Tests\Feature;

use App\Models\MenuCategory;
use App\Models\MenuItem;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class CheckoutTest extends TestCase
{
    use RefreshDatabase;

    public function test_checkout_uses_the_server_menu_price(): void
    {
        $customer = User::factory()->create();
        $category = MenuCategory::factory()->create();
        $item = MenuItem::factory()->for($category)->create([
            'price_cents' => 850,
            'is_available' => true,
        ]);

        $this->actingAs($customer)
            ->withSession(['cart' => [$item->id => 2]])
            ->post(route('checkout.store'), [
                'total_cents' => 1,
                'price_cents' => 1,
            ])
            ->assertRedirect();

        $this->assertDatabaseHas('orders', [
            'user_id' => $customer->id,
            'total_cents' => 1700,
        ]);
    }
}
```

The hostile price fields demonstrate that the server calculates the total from
database values. Keep the Arrange/Act/Assert steps in the repository's existing
PHPUnit test-class style.

## Run the project's checks

Run the repository's defined checks from PowerShell in the project root:

```powershell
docker compose exec laravel.test composer ci:check
docker compose exec laravel.test npm run build
```

If a command fails, record the first actionable error, fix the cause, and rerun
that command before continuing. Do not hide a failing build by weakening the
check.

## Write the README tutorial

The final README should document the actual Compose workflow:

```powershell
if (-not (Test-Path .\.env)) {
    Copy-Item .\.env.example .\.env
}
if (-not (Test-Path .\vendor\laravel\sail\runtimes\8.5\Dockerfile)) {
    composer install --no-scripts
}
docker compose up -d --build
docker compose exec laravel.test composer install
# Run only once for a fresh .env with an empty APP_KEY.
docker compose exec laravel.test php artisan key:generate
docker compose exec laravel.test php artisan migrate --seed
docker compose exec laravel.test npm install
docker compose exec laravel.test npm run build
docker compose exec laravel.test composer ci:check
docker compose down
```

Also document the app URL (`http://localhost` unless `APP_PORT` is set), local
Mailpit URL, optional extension services and required variable names without
values, how to stop services, and that demo payments are not real. Never
include a real password, API key, or `.env` contents.
