# Step 6: Order workflow and staff dashboard

**Days:** 26-30 (Week 6)  
**Branches:** `feature/order-workflow`, then `feature/staff-dashboard`

## Goal

Learn how to model allowed status changes and protect staff-only operations.

## Days 26-27: `feature/order-workflow`

- Day 26: add an order status enum and document a small explicit lifecycle:
  `pending -> accepted -> preparing -> ready -> completed`, with cancellation
  allowed only at defined stages.
- Day 27: centralize allowed transitions in a domain method/service and add
  tests for valid and invalid transitions.
- Never accept an arbitrary status string from the browser.

**Checkpoint:** A customer cannot mark their own order completed or jump it
directly from pending to ready.

## Days 28-30: `feature/staff-dashboard`

- Merge the workflow branch, then branch from updated `main`.
- Day 28: add the staff flag and policy/middleware; keep role assignment out of
  public registration.
- Day 29: build a staff queue with newest pending orders, item summaries, and
  only valid status actions.
- Day 30: test customer denial, staff access, order ownership, and invalid
  transitions; review queries for unnecessary data.
- Keep customer order history scoped to the authenticated user's orders.
- Use only a clearly documented local-only staff setup; never commit account
  credentials.

**Checkpoint:** Feature tests prove customers cannot access staff routes and
staff can update an order only through allowed transitions.

## Learning topics

- Eloquent scopes and relationships.
- Policies/authorization and ownership checks.
- State transitions, server validation, and feature tests.

## Deliverable

Customers see their own order history; staff can process orders through a
controlled workflow.

## Branch closeout

Run authorization, ownership, and transition tests, then merge
`feature/order-workflow` and `feature/staff-dashboard`.

## References

- [Laravel authorization](https://laravel.com/docs/13.x/authorization)
- [Laravel policies](https://laravel.com/docs/13.x/authorization#creating-policies)
- [Laravel enums and casts](https://laravel.com/docs/13.x/eloquent-mutators)

## Tutorial file map

```text
app/Enums/OrderStatus.php
app/Models/Order.php
app/Models/User.php
app/Policies/OrderPolicy.php
app/Http/Controllers/Staff/OrderStatusController.php
database/seeders/DatabaseSeeder.php  local-only staff setup
database/migrations/*_add_is_staff_to_users_table.php
database/migrations/*_add_status_to_orders_table.php  only if Step 5 did not add it
resources/js/pages/staff/Orders/Index.vue
tests/Feature/StaffOrderTest.php
```

Create the migration for the staff flag with:

```powershell
docker compose exec laravel.test php artisan make:migration add_is_staff_to_users_table --table=users
```

In `app/Models/Order.php`, import `App\Enums\OrderStatus` and
`Illuminate\Validation\ValidationException` for the snippets below. If Step 5
already created `orders.status`, do not add a duplicate status column; only
update its cast.

## Part A: express the state machine

Create `app/Enums/OrderStatus.php`:

```php
<?php

namespace App\Enums;

enum OrderStatus: string
{
    case Pending = 'pending';
    case Accepted = 'accepted';
    case Preparing = 'preparing';
    case Ready = 'ready';
    case Completed = 'completed';
    case Cancelled = 'cancelled';

    public function canTransitionTo(self $next): bool
    {
        return match ($this) {
            self::Pending => in_array(
                $next,
                [self::Accepted, self::Cancelled],
                true,
            ),
            self::Accepted => in_array(
                $next,
                [self::Preparing, self::Cancelled],
                true,
            ),
            self::Preparing => $next === self::Ready,
            self::Ready => $next === self::Completed,
            self::Completed, self::Cancelled => false,
        };
    }
}
```

In `Order::casts()`, cast the `status` column:

```php
protected function casts(): array
{
    return ['status' => OrderStatus::class];
}
```

Add a transition method to `Order`. It should reject invalid moves before
saving:

```php
public function transitionTo(OrderStatus $next): void
{
    if (! $this->status->canTransitionTo($next)) {
        throw ValidationException::withMessages([
            'status' => 'That order status transition is not allowed.',
        ]);
    }

    $this->status = $next;
    $this->save();
}
```

Test every allowed edge and at least one forbidden edge. Never accept a raw
status and call `$order->update($request->all())`.

## Part B: authorize staff operations

In the generated migration, add the imports and methods around this schema
change:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->boolean('is_staff')->default(false);
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('is_staff');
        });
    }
};
```

In the existing `User::casts()` method, add the boolean cast without removing
the current password, email, or two-factor casts:

```php
'is_staff' => 'boolean',
```

Do not add `is_staff` to the public registration fillable fields. Set it only
in a trusted local seeder, for example:

```php
$staff = User::factory()->create();
$staff->forceFill(['is_staff' => true])->save();
```

In `app/Policies/OrderPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Models\Order;
use App\Models\User;

class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->is_staff || $order->user_id === $user->id;
    }

    public function updateStatus(User $user, Order $order): bool
    {
        return $user->is_staff;
    }
}
```

Use the policy in the controller before updating an order:

```php
$this->authorize('updateStatus', $order);

$data = $request->validate([
    'status' => ['required', Rule::enum(OrderStatus::class)],
]);

$order->transitionTo(OrderStatus::from($data['status']));

return back();
```

The route should be under `auth` and `verified` middleware. Add a staff prefix
and route name, for example `staff.orders.status.update`.

## Build the staff page

The server should provide a small, explicit shape, not entire models:

```php
'orders' => Order::query()
    ->with('user:id,name', 'items:id,order_id,item_name,quantity')
    ->whereIn('status', [
        OrderStatus::Pending,
        OrderStatus::Accepted,
        OrderStatus::Preparing,
    ])
    ->latest()
    ->get()
    ->map(fn (Order $order): array => [
        'id' => $order->id,
        'customer' => $order->user->name,
        'status' => $order->status->value,
        'items' => $order->items->map(fn (OrderItem $item): array => [
            'name' => $item->item_name,
            'quantity' => $item->quantity,
        ]),
    ]),
```

In `resources/js/pages/staff/Orders/Index.vue`, type those props and post a
selected next status to the named route. Only show buttons that match the
current state, but remember that the policy and enum are the real security
checks.

## Acceptance tests

`tests/Feature/StaffOrderTest.php` should prove:

- A customer gets 403 from staff routes.
- A customer can view only their own order.
- A staff member can perform a valid transition.
- A staff member cannot skip a transition.

Do not use the test user's email/password as production credentials. Seed a
staff user only in local development and tests.
