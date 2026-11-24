# Step 3: Inertia pages, props, and navigation

**Days:** 7-9  
**Branch:** `learn/inertia-pages-props`

## Goal

Understand Inertia as the connection between Laravel routes/controllers and
Vue pages. A page prop is a server-provided snapshot; it is not automatically
live data.

## Day 7: Trace an existing request

- Trace `/` from `routes/web.php` to `Welcome.vue`.
- Trace the authenticated dashboard route and the app layout.
- Change a small server-provided page title or welcome prop, then observe it in
  Vue.
- Explain which code runs on the server and which runs in the browser.

**Checkpoint:** Draw the request path: URL -> route -> controller/page response
-> Inertia props -> Vue render.

## Day 8: Real server data

- Add a simple server endpoint/controller that returns a typed list of practice
  menu items as props.
- Render the list with the menu-card component from Step 2.
- Use Inertia links for navigation rather than full-page anchors for internal
  pages.
- Keep queries and authorization on the Laravel side.

**Checkpoint:** The menu page renders server data and navigation works without
losing the authenticated layout.

## Day 9: Forms and validation feedback

- Inspect existing profile form patterns and Inertia form helpers in this repo.
- Submit a small search or filter form to Laravel.
- Return validation errors and display them beside the relevant control.
- Run the frontend checks and Laravel feature tests.

**Checkpoint:** A failed form submission displays useful server validation
without a full page error.

## Deliverable

A server-rendered-through-Inertia menu practice page with navigation and one
validated form.

## Branch closeout

Review props for unnecessary or sensitive fields, run checks, and merge
`learn/inertia-pages-props`.

## References

- [Inertia server-side responses](https://inertiajs.com/responses)
- [Inertia forms](https://inertiajs.com/forms)
- [Inertia links](https://inertiajs.com/links)

## Tutorial files

Create:

```text
app/Http/Controllers/MenuController.php
resources/js/pages/menu/Index.vue
routes/web.php                 add the named menu route
```

Keep the sample array for this lesson. The next step moves the same shape to
Eloquent.

### 1. Add a route and controller

In `routes/web.php`, preserve the existing starter-kit routes and add:

```php
use App\Http\Controllers\MenuController;

Route::get('/menu', [MenuController::class, 'index'])->name('menu.index');
```

Create `app/Http/Controllers/MenuController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class MenuController extends Controller
{
    public function index(Request $request): Response
    {
        $validated = $request->validate([
            'q' => ['nullable', 'string', 'max:80'],
        ]);

        $query = mb_strtolower($validated['q'] ?? '');

        $items = collect([
            [
                'id' => 1,
                'name' => 'Sample soup',
                'description' => 'A demo menu item.',
                'priceCents' => 850,
                'isAvailable' => true,
            ],
            [
                'id' => 2,
                'name' => 'Sample salad',
                'description' => 'Another demo menu item.',
                'priceCents' => 1050,
                'isAvailable' => false,
            ],
        ])->filter(
            fn (array $item): bool => $query === ''
                || str_contains(mb_strtolower($item['name']), $query),
        )->values();

        return Inertia::render('menu/Index', [
            'items' => $items,
            'filters' => ['q' => $validated['q'] ?? ''],
        ]);
    }
}
```

The controller validates the query string and sends serializable data to one
Vue page. It does not accept a price or availability value from the browser.

### 2. Render props and submit a search

Create `resources/js/pages/menu/Index.vue`:

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { Head, Link, useForm } from '@inertiajs/vue3';
import MenuItemCard from '@/components/menu/MenuItemCard.vue';
import type { MenuItem } from '@/types/menu';

const props = defineProps<{
    items: MenuItem[];
    filters: { q: string };
}>();

const form = useForm({ q: props.filters.q });
const cartCount = ref(0);

function search(): void {
    form.get('/menu', { preserveState: true, replace: true });
}

function addItem(): void {
    cartCount.value += 1;
}
</script>

<template>
    <Head title="Menu" />
    <main class="mx-auto max-w-4xl space-y-6 p-6">
        <nav><Link href="/">Home</Link></nav>
        <h1 class="text-2xl font-bold">Menu</h1>
        <p>Practice cart count: {{ cartCount }}</p>
        <form class="flex gap-2" @submit.prevent="search">
            <label for="menu-search">Search menu</label>
            <input id="menu-search" v-model="form.q" maxlength="80" />
            <button type="submit" :disabled="form.processing">Search</button>
        </form>
        <p v-if="form.errors.q" role="alert">{{ form.errors.q }}</p>
        <p v-if="props.items.length === 0">No matching items.</p>
        <section class="grid gap-4 md:grid-cols-2">
            <MenuItemCard
                v-for="item in props.items"
                :key="item.id"
                :item="item"
                @add="addItem"
            />
        </section>
    </main>
</template>
```

The local `cartCount` is only a placeholder for this lesson. Step 5 moves cart
state and validation to Laravel.

### 3. Trace and verify

Open `/menu`, search for `soup`, and inspect the Inertia response in the
browser's network panel. Confirm that the `items` prop changes and the page
does not perform a full document navigation. Also run:

```powershell
docker compose exec laravel.test php artisan route:list --path=menu
docker compose exec laravel.test php artisan test
docker compose exec laravel.test npm run types:check
```
