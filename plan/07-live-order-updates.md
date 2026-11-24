# Step 7: Live order updates

**Days:** 19-21  
**Branch:** `feature/live-order-updates`

## Goal

Learn how Inertia props can be refreshed and distinguish polling from true
server-pushed real-time events.

## Day 19: Inertia live props

- Configure Reverb and Echo following the Laravel installer output.
- Return the order through `Inertia::live()` and name the event that refreshes
  it.
- Return only the order props needed by the page; the live-prop update is a
  partial reload, not a full document reload.

**Checkpoint:** A status event from the staff dashboard updates the customer's
order page without a manual refresh.

## Day 20: Make updates safe and useful

- Ensure every tracking query checks order ownership.
- Display a clear status timeline and last-updated time.
- Handle temporary network errors and pause/resume behavior.
- Add tests for the tracking page's ownership and returned props.

**Checkpoint:** A user cannot poll or view another customer's order by changing
an ID.

## Day 21: Compare with polling

- Implement a polling fallback with Inertia's `usePoll` at a modest interval.
- Compare request frequency and update latency with the live-prop version.
- Keep polling only as a documented fallback; do not run it alongside live
  props by default.

**Checkpoint:** Document when polling is useful and why the primary
implementation uses event-driven live props.

## Deliverable

An order tracking page with efficient partial updates and access control.

## Branch closeout

Run tests and ensure polling stops appropriately. Merge
`feature/live-order-updates`.

## References

- [Inertia partial reloads](https://inertiajs.com/partial-reloads)
- [Inertia live props](https://inertiajs.com/docs/v3/data-props/live-props)
- [Laravel broadcasting](https://laravel.com/docs/13.x/broadcasting)
- [Laravel Reverb](https://laravel.com/docs/13.x/reverb)

## Tutorial file map

```text
app/Events/OrderStatusChanged.php
routes/channels.php
app/Http/Controllers/Orders/ShowController.php
resources/js/app.ts
resources/js/pages/orders/Show.vue
```

Inertia 3 **live props** refresh a prop when a Laravel broadcast event arrives.
Polling sends requests at a fixed interval; the live-prop implementation below
is event-driven. Use polling only as a measured fallback.

## Part A: start with partial polling

In `resources/js/pages/orders/Show.vue`:

```vue
<script setup lang="ts">
import { usePoll } from '@inertiajs/vue3';

const props = defineProps<{
    order: { id: number; status: string; updatedAt: string };
}>();

usePoll(10_000, {
    only: ['order'],
    preserveScroll: true,
});
</script>

<template>
    <main>
        <h1>Order #{{ props.order.id }}</h1>
        <p role="status">Status: {{ props.order.status }}</p>
        <time>{{ props.order.updatedAt }}</time>
    </main>
</template>
```

Observe the browser network panel. Only the `order` prop is requested every ten
seconds. Polling is an acceptable fallback, but it is not push-based.

## Part B: configure Reverb and Echo

Install the Laravel broadcasting scaffolding in a separate branch checkpoint:

```powershell
docker compose exec laravel.test php artisan install:broadcasting
```

Follow the generated Reverb instructions, add the service to Compose, and set
local-only `REVERB_*` values in `.env`. Never paste a production secret into
Vue or `VITE_*` variables.

Update `resources/js/app.ts` by importing the transport and configuring Echo:

```ts
import { echo } from '@inertiajs/vue3/echo';
import { configureEcho } from '@laravel/echo-vue';

configureEcho({ broadcaster: 'reverb' });

void createInertiaApp({
    live: echo(),
    // Keep the existing title, layout, withApp, and progress options here.
});
```

Do not replace the rest of the existing `createInertiaApp` configuration; add
the `live` option to it.

## Part C: define a private event

Create `app/Events/OrderStatusChanged.php`:

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderStatusChanged implements ShouldBroadcastNow
{
    use Dispatchable, SerializesModels;

    public function __construct(public Order $order) {}

    public function broadcastOn(): array
    {
        return [new PrivateChannel("orders.{$this->order->id}")];
    }

    public function broadcastAs(): string
    {
        return 'order.status.changed';
    }

    public function broadcastWith(): array
    {
        return [
            'orderId' => $this->order->id,
            'status' => $this->order->status->value,
        ];
    }
}
```

`ShouldBroadcastNow` is used here to keep the first local demo easy to observe.
Learn queued broadcasting later. In `routes/channels.php`, authorize the private
channel only for the order owner or staff:

```php
Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    $order = Order::find($orderId);

    return $order !== null
        && ($user->is_staff || $order->user_id === $user->id);
});
```

In `ShowController`, wrap the order prop:

```php
'order' => Inertia::live(
    fn () => $order->fresh()->load('items'),
    on: new OrderStatusChanged($order),
),
```

Dispatch `OrderStatusChanged` only after a valid status transition is saved.
Keep the event payload small; the live prop reloads the authorized server
response.

## Verify the live update

Open the customer tracking page in one browser session and update the order
from the staff page in another. Confirm the status changes without a manual
refresh. Test that another customer's account cannot subscribe to the private
channel or receive the order prop.
