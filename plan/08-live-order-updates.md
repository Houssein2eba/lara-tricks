# Step 8: Live order updates

**Days:** 36-40 (Week 8)  
**Branch:** `feature/live-order-updates`

## Learning goal

Learn how Inertia props can be refreshed and distinguish polling from true
server-pushed real-time events.

## Day 36: Understand freshness and polling

- Start with a simple partial reload and a modest `usePoll` interval.
- Measure request frequency and decide whether this small demo needs push
  updates at all.

**Checkpoint:** You can explain the trade-off between polling latency and
server-sent updates before adding another service.

## Day 37: Configure broadcasts

- If polling is not sufficient, add Laravel broadcasting and Reverb using the
  framework's installer; review every generated change.
- Configure local environment values and the Vue Echo integration.
- If Reverb pushes Docker over your memory budget, keep the tested polling
  implementation and skip this optional service.

**Checkpoint:** If Reverb is enabled, a local test event is received by the
correct browser session. Otherwise, record why polling is sufficient for this
machine and continue with the tested polling implementation.

## Day 38: Publish a minimal status event

- Broadcast only after a committed order status change.
- Use a private channel and authorize it against the authenticated order owner.
- Do not broadcast full models or customer contact details.

**Checkpoint:** With Reverb enabled, unauthorized users cannot subscribe to
another customer's order channel. If skipped, write the authorization rule
that a future private channel must enforce.

## Day 39: Refresh the customer's order page

- Return only the order fields needed by the page through `Inertia::live()`.
- Render a status timeline and last-updated time.
- Handle connection loss without showing a false successful update.

**Checkpoint:** With Reverb enabled, a staff status change refreshes the
owner's page without a manual reload. Otherwise, the polling page refreshes
only the owner's order props at the chosen interval.

## Day 40: Compare, test, and decide

- Keep polling only as a documented fallback; do not run it alongside live
  props by default.
- Test event authorization and the returned order props.
- Record the additional Compose memory and operational cost of Reverb.

**Checkpoint:** The feature has ownership tests and a reasoned choice of push
updates versus polling.

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

On `feature/live-order-updates`, install the Laravel broadcasting scaffolding
only after the polling version works:

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
