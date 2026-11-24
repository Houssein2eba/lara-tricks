# Step 5: Cart and checkout

**Days:** 13-15  
**Branches:** `feature/cart`, then `feature/checkout`

## Goal

Build a useful customer flow while learning server-side validation, ownership,
and database transactions.

## Days 13-14: `feature/cart`

- Start from updated `main` and create `feature/cart`.
- Add, remove, and change item quantities from the Vue storefront.
- Keep the cart in the server session for this learning project; never trust a
  client-submitted price or total.
- Display a derived subtotal and disable unavailable items.
- Validate item IDs and quantity bounds on every server request.

**Checkpoint:** Refreshing the page preserves the cart, and tampering with
client-side prices cannot alter the server-calculated total.

## Day 15: `feature/checkout`

- Merge the cart branch, then branch from updated `main`.
- Add `orders` and `order_items` migrations with clear relationships.
- On checkout, re-fetch current catalog prices and create the order and item
  snapshots inside a database transaction.
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

Run the order feature tests and review transaction/price logic. Merge
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
database/migrations/*_create_orders_table.php
database/migrations/*_create_order_items_table.php
resources/js/pages/cart/Index.vue
tests/Feature/CheckoutTest.php
```

The cart is stored in the Laravel session. The browser sends item IDs and
quantities only; the server always looks up availability and price again.

Generate the starting models/migrations before editing them:

```powershell
docker compose exec laravel.test php artisan make:model Order -m
docker compose exec laravel.test php artisan make:model OrderItem -m
```

## Part A: add items to the session cart

Add routes inside the existing authenticated/verified group in `routes/web.php`:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::post('/cart/items/{menuItem}', [CartController::class, 'store'])
        ->name('cart.items.store');
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

Create a `show` action that loads the IDs from the session with
`MenuItem::whereIn('id', array_keys($cart))->where('is_available', true)`.
Return only `id`, `name`, `price_cents`, and the session quantity. Calculate
the subtotal from those current server-side prices.

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
