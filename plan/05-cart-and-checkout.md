# Step 5: Cart and checkout

**Days:** 21-25 (Week 5)  
**Branches:** `feature/cart`, then `feature/checkout`

## Goal

Build a useful customer flow while learning server-side validation, ownership,
and database transactions.

## Days 21-23: `feature/cart`

- Start from updated `main` and create `feature/cart`.
- Day 21: define the server-side session cart shape and add one item.
- Day 22: add remove/update quantity actions and clear unavailable items.
- Day 23: build the cart page and test boundary quantities and tampered input.
- Keep the cart in the server session for this learning project; never trust a
  client-submitted price or total.
- Display a derived subtotal and disable unavailable items.
- Validate item IDs and quantity bounds on every server request.

**Checkpoint:** Refreshing the page preserves the cart, and tampering with
client-side prices cannot alter the server-calculated total.

## Days 24-25: `feature/checkout`

- Merge the cart branch, then branch from updated `main`.
- Day 24: add `orders` and `order_items` migrations and Eloquent relationships.
- Day 25: implement checkout and test success, invalid cart, and server-priced
  snapshots.
- Re-fetch current catalog prices and create the order and item snapshots
  inside a database transaction.
- Store item name and unit price snapshots so later catalog edits do not rewrite
  old receipts.
- Require the authenticated customer and validate the checkout request.

**Checkpoint:** A feature test proves a valid cart creates one order and the
expected item rows; invalid or unavailable items create no partial order.

## Scope decisions

- Require login for placing orders; use the existing Fortify login/register
  flow.
- Do not collect card data or claim to process real payments.
- Keep delivery addresses and personal data to the minimum needed for the demo.

## Deliverable

An authenticated customer can build a cart and place a demo order safely.

## Branch closeout

Run the cart and checkout feature tests and review transaction/price logic. Merge
`feature/cart`, then `feature/checkout`.

## References

- [Laravel validation](https://laravel.com/docs/13.x/validation)
- [Laravel database transactions](https://laravel.com/docs/13.x/database#database-transactions)
- [Inertia forms](https://inertiajs.com/forms)

## Tutorial file map

```text
app/Http/Controllers/CartController.php
app/Http/Controllers/CheckoutController.php
app/Models/Order.php
app/Models/OrderItem.php
database/factories/OrderFactory.php
database/factories/OrderItemFactory.php
database/migrations/*_create_orders_table.php
database/migrations/*_create_order_items_table.php
resources/js/pages/cart/Index.vue
tests/Feature/CheckoutTest.php
```

The cart is stored in the Laravel session. The browser sends item IDs and
quantities only; the server always looks up availability and price again.

Generate the starting models/migrations before editing them:

```powershell
docker compose exec laravel.test php artisan make:model Order -mf
docker compose exec laravel.test php artisan make:model OrderItem -mf
```

## Part A: add items to the session cart

Add routes inside the existing authenticated/verified group in `routes/web.php`:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::post('/cart/items/{menuItem}', [CartController::class, 'store'])
        ->name('cart.items.store');
    Route::put('/cart/items/{menuItem}', [CartController::class, 'update'])
        ->name('cart.items.update');
    Route::delete('/cart/items/{menuItem}', [CartController::class, 'destroy'])
        ->name('cart.items.destroy');
    Route::get('/cart', [CartController::class, 'show'])->name('cart.show');
    Route::post('/checkout', [CheckoutController::class, 'store'])
        ->name('checkout.store');
});
```

Add the missing controller imports at the top of `routes/web.php`.

In `app/Http/Controllers/CartController.php`, implement the add action:

```php
public function store(Request $request, MenuItem $menuItem): RedirectResponse
{
    $data = $request->validate([
        'quantity' => ['required', 'integer', 'min:1', 'max:10'],
    ]);

    if (! $menuItem->is_available) {
        throw ValidationException::withMessages([
            'cart' => 'This item is not available.',
        ]);
    }

    $cart = $request->session()->get('cart', []);
    $current = (int) ($cart[$menuItem->id] ?? 0);
    $cart[$menuItem->id] = min(10, $current + $data['quantity']);
    $request->session()->put('cart', $cart);

    return back();
}
```

For `update`, validate the new quantity between 1 and 10 and reject unavailable
items. For `destroy`, remove the route-bound item ID from the session cart and
redirect back. The `show` action loads the session IDs from the database,
returns only the fields needed by the page, and includes availability so a
stale item can be removed. Calculate subtotal only from currently available
items and their current server-side prices.

The Vue cart page should use Inertia requests rather than changing the session
in the browser:

```ts
router.put(`/cart/items/${item.id}`, { quantity }, { preserveScroll: true });
router.delete(`/cart/items/${item.id}`, { preserveScroll: true });
```

In `resources/js/pages/menu/Index.vue`, replace the local-only add handler with
an Inertia request:

```ts
import { router } from '@inertiajs/vue3';

function addToCart(itemId: number): void {
    router.post(`/cart/items/${itemId}`, { quantity: 1 }, {
        preserveScroll: true,
    });
}
```

Bind it with `@add="addToCart"`. The server validates the request even if a
customer changes the browser payload.

## Part B: create an order atomically

The `orders` migration should include a user, status, and integer total:

```php
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('status')->default('pending')->index();
    $table->unsignedInteger('total_cents')->default(0);
    $table->timestamps();
});
```

The `order_items` migration stores a historical snapshot:

```php
Schema::create('order_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->foreignId('menu_item_id')->nullable()
        ->constrained()->nullOnDelete();
    $table->string('item_name');
    $table->unsignedInteger('unit_price_cents');
    $table->unsignedInteger('quantity');
    $table->unsignedInteger('line_total_cents');
    $table->timestamps();
});
```

In `app/Models/Order.php`, add relationships:

```php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function items(): HasMany
{
    return $this->hasMany(OrderItem::class);
}
```

In `app/Models/OrderItem.php`, add the inverse `order()` relationship. Add
explicit fillable fields to both models; never mass-assign `user_id` directly
from request data.

Keep the generated factories small and valid. For example, the order factory
should provide a customer, a pending status, and a zero total; the order-item
factory should provide an order, a snapshot name/price, and a positive
quantity. Tests can override these defaults with `for($customer)` and
`state([...])` rather than hand-editing the database.

In `CheckoutController::store`, validate the session cart against current
database rows, then create the order and snapshots in one transaction:

```php
$cart = $request->session()->get('cart', []);
$items = MenuItem::query()
    ->whereIn('id', array_keys($cart))
    ->where('is_available', true)
    ->get()
    ->keyBy('id');

if ($cart === [] || $items->count() !== count($cart)) {
    throw ValidationException::withMessages([
        'cart' => 'Review your cart; an item is unavailable.',
    ]);
}

$order = DB::transaction(function () use ($request, $cart, $items): Order {
    $order = $request->user()->orders()->create([
        'status' => 'pending',
        'total_cents' => 0,
    ]);
    $total = 0;

    foreach ($cart as $id => $quantity) {
        $item = $items->get($id);
        $lineTotal = $item->price_cents * (int) $quantity;
        $total += $lineTotal;

        $order->items()->create([
            'menu_item_id' => $item->id,
            'item_name' => $item->name,
            'unit_price_cents' => $item->price_cents,
            'quantity' => (int) $quantity,
            'line_total_cents' => $lineTotal,
        ]);
    }

    $order->update(['total_cents' => $total]);

    return $order;
});

$request->session()->forget('cart');

return to_route('orders.show', $order);
```

Add `orders(): HasMany` to `User`. Validate session quantities as integers
between 1 and 10 before this transaction. If a transaction fails, keep the
cart so the customer can retry.

## Tests to write

In `tests/Feature/CheckoutTest.php`, cover:

1. Guest checkout redirects to login.
2. A valid authenticated cart creates one order and the expected snapshots.
3. The submitted browser payload cannot choose the price or owner.
4. An unavailable/deleted item creates no order and leaves the cart intact.

Run the tests and migrations from PowerShell:

```powershell
docker compose exec laravel.test php artisan migrate
docker compose exec laravel.test php artisan test --filter=CheckoutTest
```
