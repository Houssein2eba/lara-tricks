# Step 8: Email and in-app notifications

**Days:** 22-24  
**Branches:** `feature/email-notifications`, then
`feature/in-app-notifications`

## Goal

Notify a customer when an order is accepted or its status changes, using a local
email catcher and a persistent in-app notification list.

## Day 22: `feature/email-notifications`

- Add a local Mailpit service to Compose and configure Laravel's mail settings
  for its SMTP host/port.
- Create a Laravel notification or mailable for an order status change.
- Send a local email for one meaningful transition; avoid sending an email for
  every poll.
- Use Laravel's mail fake in tests and Mailpit to inspect rendered messages.

**Checkpoint:** The test asserts an email is queued/sent, and the local message
can be previewed without contacting a real recipient.

## Day 23: Queue and delivery behavior

- Learn notification channels, queueing, and failure handling.
- Queue email if the current queue setup is understood and working; otherwise
  keep this demo synchronous and record queueing as follow-up work.
- Ensure emails contain only the recipient's own order details.
- Test that unrelated users receive no notification.

**Checkpoint:** Delivery failures are visible in logs/failed jobs; they are not
silently treated as success.

## Day 24: `feature/in-app-notifications`

- Branch from updated `main`.
- Add Laravel's database notification table and an authenticated notification
  list/read action.
- Add a Vue notification bell/list and use the existing toast library for brief
  foreground feedback.
- Optionally explore the browser Notification API only after explicit user
  permission; never request permission on page load.

**Checkpoint:** A user sees and can mark their own order notification as read;
tests verify notification ownership.

## Deliverable

Local email and in-app notifications for selected order events.

## Branch closeout

Check Mailpit settings are local-only and no real credentials are committed.
Merge `feature/email-notifications`, then `feature/in-app-notifications`.

## References

- [Laravel notifications](https://laravel.com/docs/13.x/notifications)
- [Laravel mail](https://laravel.com/docs/13.x/mail)
- [Mailpit](https://mailpit.axllent.org/)
- [MDN Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API)

## Tutorial file map

```text
compose.yaml
app/Notifications/OrderStatusUpdated.php
app/Http/Controllers/Notifications/IndexController.php
app/Http/Controllers/Notifications/ReadController.php
resources/js/pages/notifications/Index.vue
tests/Feature/OrderNotificationsTest.php
```

## Part A: inspect email locally with Mailpit

Add a local-only service to `compose.yaml`:

```yaml
mailpit:
    image: axllent/mailpit:latest
    ports:
        - '${MAILPIT_WEB_PORT:-8025}:8025'
    networks:
        - sail
```

The Laravel container can reach SMTP at `mailpit:1025` on the Compose network.
Add these values to the local, untracked `.env`:

```dotenv
MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_SCHEME=null
MAIL_FROM_ADDRESS="orders@example.test"
MAIL_FROM_NAME="${APP_NAME}"
```

Recreate the service and open `http://localhost:8025`. Do not use a real SMTP
provider or real recipient address in the tutorial environment.

## Part B: send one status notification

Generate the class:

```powershell
docker compose exec laravel.test php artisan make:notification OrderStatusUpdated
```

In `app/Notifications/OrderStatusUpdated.php`, begin with:

```php
public function __construct(public Order $order) {}

public function via(object $notifiable): array
{
    return ['mail', 'database'];
}

public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->subject("Order #{$this->order->id} updated")
        ->line("Your order is now {$this->order->status->value}.")
        ->action('View order', route('orders.show', $this->order));
}

public function toDatabase(object $notifiable): array
{
    return [
        'order_id' => $this->order->id,
        'status' => $this->order->status->value,
    ];
}
```

Add the imports for `Order`, `MailMessage`, and `Notification`. Notify the
customer only after a valid transition:

```php
$order->user->notify(new OrderStatusUpdated($order));
```

During development, use a test fake and assert the recipient:

```php
Notification::fake();

// Perform the accepted status transition as staff.

Notification::assertSentTo(
    $customer,
    OrderStatusUpdated::class,
);
```

## Part C: display and mark in-app notifications

Generate the database notification migration with Laravel's
`make:notifications-table` command, then migrate. Query notifications through
the signed-in user, never from a global `Notification` query:

```php
$notifications = $request->user()
    ->notifications()
    ->latest()
    ->limit(20)
    ->get();
```

For a read action, resolve the notification through
`$request->user()->notifications()` before calling `markAsRead()`. This prevents
one user from marking another user's notification as read.

In Vue, show a concise list and use the existing `vue-sonner` toast for an
immediate foreground message:

```ts
import { toast } from 'vue-sonner';

toast.success('Your order status changed.');
```

Browser notifications are an optional enhancement. Ask permission only after a
user clicks an explicit "Enable desktop notifications" control.
