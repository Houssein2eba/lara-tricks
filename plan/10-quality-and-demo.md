# Step 10: Quality, accessibility, and demo release

**Days:** 28-30  
**Branches:** `test/e2e-hardening`, then `chore/demo-release`

## Goal

Make the learning project reliable enough to demonstrate and explain. Do not add
new major features in the final three days.

## Day 28: `test/e2e-hardening`

- Start from updated `main`.
- Review the complete customer flow: browse -> cart -> place order -> track
  status -> receive notification.
- Add/finish Laravel feature tests for validation, ownership, staff authorization,
  order transitions, and notifications.
- Add a browser-level smoke test if the repo's existing test setup supports it;
  otherwise write a repeatable manual test checklist.
- Test the AI provider failure path without making live API calls.

**Checkpoint:** A fresh test database can run migrations and the critical
feature tests pass.

## Day 29: Quality and accessibility

- Check keyboard navigation, visible focus, labels, contrast, loading/error
  states, and mobile layout.
- Run frontend `check`, TypeScript checks, production asset build, Laravel
  tests, Pint, and PHPStan using the project's available scripts.
- Inspect browser console, Laravel logs, queue/notification behavior, and
  Compose health.
- Fix only defects in the planned app; record unrelated ideas as future work.

**Checkpoint:** No known blocker remains in the core order flow; checks and
their exact commands are recorded.

## Day 30: `chore/demo-release`

- Merge the test branch, then create the release/demo branch from updated
  `main`.
- Update the README with setup, environment variables, ports, seeded demo
  accounts (never passwords/secrets), and how to start/stop the app.
- Confirm `.env`, AI keys, and real customer data are not tracked.
- Prepare a short demo script and a list of known limits: no real payments,
  polling latency, free AI quota, and local-only email.
- Tag or record a demo commit only if the repository has a remote/release
  convention.

**Checkpoint:** Another developer can follow the README and reproduce the demo
without private credentials.

## Final acceptance checklist

- Customer can browse items, maintain a cart, and place a demo order.
- Staff-only workflow updates statuses with authorization and valid transitions.
- Customer sees their own updated order and notifications only.
- Local email is inspectable in Mailpit; no production mail is sent.
- AI helper is optional, rate-limited, server-keyed, and read-only.
- Fresh migrations, seed data, tests, frontend checks, and build pass.
- The main branch contains all merged topic branches and remains runnable.

## Branch closeout

Merge `test/e2e-hardening`, then `chore/demo-release`. Preserve the 15 topic
branches in history; do not leave incomplete work merged to `main`.

## Tutorial: final checks by file area

### Test file map

```text
tests/Feature/MenuPageTest.php
tests/Feature/CheckoutTest.php
tests/Feature/StaffOrderTest.php
tests/Feature/OrderNotificationsTest.php
tests/Feature/AiChatTest.php
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

Adapt the factory and route names to the implementation. Add a matching
positive test proving the owner can view their own order.

### Example checkout test

In `tests/Feature/CheckoutTest.php`:

```php
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('creates an order using the server menu price', function () {
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
});
```

The hostile price fields demonstrate that the server calculates the total from
database values. If the project uses PHPUnit class syntax rather than Pest,
write the same Arrange/Act/Assert steps in the existing test style.

## Run the project's checks

Run commands in PowerShell from the project root:

```powershell
docker compose exec laravel.test php artisan test
docker compose exec laravel.test ./vendor/bin/pint --test
docker compose exec laravel.test ./vendor/bin/phpstan analyse
docker compose exec laravel.test npm run check
docker compose exec laravel.test npm run types:check
docker compose exec laravel.test npm run build
```

If a command fails, record the first actionable error, fix the cause, and rerun
that command before continuing. Do not hide a failing build by weakening the
check.

## Write the README tutorial

The final README should document the actual Compose workflow:

```powershell
docker compose up -d --build
docker compose exec laravel.test php artisan migrate --seed
docker compose exec laravel.test npm install
docker compose exec laravel.test npm run build
docker compose down
```

Also document the app URL (`http://localhost` unless `APP_PORT` is set), local
Mailpit URL, required AI environment variable names without values, how to stop
services, and that demo payments are not real. Never include a real password,
API key, or `.env` contents.
