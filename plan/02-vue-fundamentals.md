# Step 2: Vue fundamentals

**Days:** 4-6  
**Branch:** `learn/vue-fundamentals`

## Goal

Learn enough Vue 3 and TypeScript to build interactive screens without moving
business rules or secrets into the browser.

## Day 4: Single-file components

- Read the Vue guide sections on components and templates.
- Inspect an existing page under `resources/js/pages/` and a small component
  under `resources/js/components/`.
- Identify `<script setup>`, template, imports, props, and emitted events.
- Build a static menu-card component using a typed `MenuItem` prop.

**Checkpoint:** The component renders a name, description, formatted price, and
availability state from props only.

## Day 5: Reactivity and events

- Learn `ref`, `computed`, `v-if`, `v-for`, and event handling.
- Add a quantity selector and an add-to-cart event to the practice card.
- Keep price calculation derived from item price and quantity; do not store
  duplicate computed state.
- Handle unavailable items and invalid quantities in the UI.

**Checkpoint:** A small parent page can receive the event and update a visible
cart count.

## Day 6: Reusable UI and TypeScript

- Compose the practice card with existing project UI components and styling.
- Type the props, emitted event, and local state; avoid `any`.
- Add loading, empty, and error states to the practice page.
- Run `npm run check`, `npm run types:check`, and `npm run build` inside the
  container.

**Checkpoint:** Checks pass and the new UI behaves at narrow and wide viewport
sizes.

## Deliverable

A small, reusable, typed menu-card practice feature. It is not yet connected to
real database data; that separation makes Vue behavior easier to understand.

## Branch closeout

Review the diff, explain `ref` versus `computed`, run the checks above, and
merge `learn/vue-fundamentals` before beginning Step 3.

## References

- [Vue components](https://vuejs.org/guide/essentials/component-basics.html)
- [Vue reactivity fundamentals](https://vuejs.org/guide/essentials/reactivity-fundamentals.html)
- [Vue TypeScript with Composition API](https://vuejs.org/guide/typescript/composition-api.html)

## Tutorial files

Create these files on `learn/vue-fundamentals`:

```text
resources/js/types/menu.ts
resources/js/components/menu/MenuItemCard.vue
resources/js/pages/MenuPractice.vue
```

This is a practice page. Step 3 replaces its hard-coded sample data with
server-provided Inertia props.

### 1. Define the data type

In `resources/js/types/menu.ts`:

```ts
export interface MenuItem {
    id: number;
    name: string;
    description: string | null;
    priceCents: number;
    isAvailable: boolean;
}
```

Prices are integer cents so math does not depend on floating-point rounding.
Keep this interface aligned with the shape the Laravel controller sends.

### 2. Build a reusable card

In `resources/js/components/menu/MenuItemCard.vue`:

```vue
<script setup lang="ts">
import { computed } from 'vue';
import type { MenuItem } from '@/types/menu';

const props = defineProps<{ item: MenuItem }>();
const emit = defineEmits<{ add: [itemId: number] }>();

const formattedPrice = computed(() =>
    new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD',
    }).format(props.item.priceCents / 100),
);
</script>

<template>
    <article class="rounded-lg border p-4">
        <h2 class="font-semibold">{{ props.item.name }}</h2>
        <p v-if="props.item.description" class="text-sm text-muted-foreground">
            {{ props.item.description }}
        </p>
        <p class="mt-2">{{ formattedPrice }}</p>
        <p v-if="!props.item.isAvailable" role="status">
            Currently unavailable
        </p>
        <button
            type="button"
            :disabled="!props.item.isAvailable"
            class="mt-3 rounded border px-3 py-2 disabled:opacity-50"
            @click="emit('add', props.item.id)"
        >
            Add to cart
        </button>
    </article>
</template>
```

The child owns presentation and emits an item ID. It does not update the
database or calculate an order total.

### 3. Practice reactive state

In `resources/js/pages/MenuPractice.vue`:

```vue
<script setup lang="ts">
import { ref } from 'vue';
import MenuItemCard from '@/components/menu/MenuItemCard.vue';
import type { MenuItem } from '@/types/menu';

const items = ref<MenuItem[]>([
    {
        id: 1,
        name: 'Sample soup',
        description: 'A practice item, not a real menu entry.',
        priceCents: 850,
        isAvailable: true,
    },
]);

const cartCount = ref(0);

function addItem(itemId: number): void {
    const item = items.value.find((candidate) => candidate.id === itemId);

    if (item?.isAvailable) {
        cartCount.value += 1;
    }
}
</script>

<template>
    <main class="mx-auto max-w-3xl space-y-4 p-6">
        <h1 class="text-2xl font-bold">Vue practice menu</h1>
        <p role="status">Cart items: {{ cartCount }}</p>
        <MenuItemCard
            v-for="item in items"
            :key="item.id"
            :item="item"
            @add="addItem"
        />
    </main>
</template>
```

### 4. Run checks and reflect

Run the frontend commands inside the app container:

```powershell
docker compose exec laravel.test npm run check
docker compose exec laravel.test npm run types:check
docker compose exec laravel.test npm run build
```

Change one item's availability to `false`. Confirm the label appears and the
button is disabled. Then explain why `cartCount` is a `ref`, while
`formattedPrice` is a `computed` value.
