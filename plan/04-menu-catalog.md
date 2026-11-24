# Step 4: Menu catalog and Eloquent

**Days:** 10-12  
**Branch:** `feature/menu-catalog`

## Goal

Learn migrations, Eloquent relationships, factories, seeders, and how to pass
only the catalog data a page needs.

## Day 10: Model the catalog

- Sketch a small single-store catalog: categories and menu items.
- Decide which values are required, whether an item is available, and how a
  price is represented.
- Store money as an integer number of minor units (for example cents), not a
  floating-point value.
- Create migrations and Eloquent models with an explicit category/item
  relationship.

**Checkpoint:** A fresh test database can migrate up and down cleanly.

## Day 11: Realistic sample data

- Create factories and a seeder for a few categories and menu items.
- Use deterministic, safe development data; do not seed real customer details.
- Add feature tests for category relationships and item availability.
- Seed locally and inspect the records with Tinker or the database client.

**Checkpoint:** Re-seeding produces a predictable demo catalog.

## Day 12: Catalog page

- Replace the practice array from Step 3 with Eloquent data.
- Select only fields needed by the storefront and order them consistently.
- Add an empty state and an unavailable-item state.
- Test that the page loads and unavailable items are clearly represented.

**Checkpoint:** Storefront data comes from the database, and the menu page is
covered by a feature test.

## Deliverable

A database-backed catalog that the customer-facing order flow can use.

## Branch closeout

Run migrations, seeders, tests, and frontend checks. Merge
`feature/menu-catalog`.

## References

- [Laravel migrations](https://laravel.com/docs/13.x/migrations)
- [Eloquent relationships](https://laravel.com/docs/13.x/eloquent-relationships)
- [Factories and seeders](https://laravel.com/docs/13.x/seeding)

## Tutorial files

Add these files:

```text
database/migrations/*_create_menu_categories_table.php
database/migrations/*_create_menu_items_table.php
app/Models/MenuCategory.php
app/Models/MenuItem.php
database/factories/MenuCategoryFactory.php
database/factories/MenuItemFactory.php
database/seeders/MenuCatalogSeeder.php
tests/Feature/MenuPageTest.php
```

Generate the starter files, then replace their contents with the examples and
your own explanations:

```powershell
docker compose exec laravel.test php artisan make:model MenuCategory -mf
docker compose exec laravel.test php artisan make:model MenuItem -mf
docker compose exec laravel.test php artisan make:seeder MenuCatalogSeeder
```

### 1. Create the tables

In the menu categories migration:

```php
Schema::create('menu_categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->timestamps();
});
```

In the menu items migration:

```php
Schema::create('menu_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')
        ->constrained('menu_categories')
        ->cascadeOnDelete();
    $table->string('name');
    $table->string('slug')->unique();
    $table->text('description')->nullable();
    $table->unsignedInteger('price_cents');
    $table->boolean('is_available')->default(true);
    $table->timestamps();
});
```

Run migrations only after both files are ready:

```powershell
docker compose exec laravel.test php artisan migrate
```

### 2. Define Eloquent relationships

In `app/Models/MenuCategory.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class MenuCategory extends Model
{
    use HasFactory;

    protected $fillable = ['name', 'slug'];

    public function items(): HasMany
    {
        return $this->hasMany(MenuItem::class, 'category_id');
    }
}
```

In `app/Models/MenuItem.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class MenuItem extends Model
{
    use HasFactory;

    protected $fillable = [
        'category_id',
        'name',
        'slug',
        'description',
        'price_cents',
        'is_available',
    ];

    protected function casts(): array
    {
        return ['is_available' => 'boolean'];
    }

    public function category(): BelongsTo
    {
        return $this->belongsTo(MenuCategory::class, 'category_id');
    }
}
```

### 3. Seed predictable local data

In `database/seeders/MenuCatalogSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\MenuCategory;
use App\Models\MenuItem;
use Illuminate\Database\Seeder;

class MenuCatalogSeeder extends Seeder
{
    public function run(): void
    {
        $category = MenuCategory::query()->updateOrCreate(
            ['slug' => 'lunch'],
            ['name' => 'Lunch'],
        );

        MenuItem::query()->updateOrCreate(
            ['slug' => 'garden-soup'],
            [
                'category_id' => $category->id,
                'name' => 'Garden soup',
                'description' => 'A sample seasonal soup.',
                'price_cents' => 850,
                'is_available' => true,
            ],
        );
    }
}
```

Call `$this->call(MenuCatalogSeeder::class);` from `DatabaseSeeder`. Then run
`docker compose exec laravel.test php artisan db:seed`.

### 4. Replace the sample controller data

Replace the Step 3 array with:

```php
$items = MenuItem::query()
    ->with('category:id,name')
    ->where('is_available', true)
    ->orderBy('name')
    ->get()
    ->map(fn (MenuItem $item): array => [
        'id' => $item->id,
        'name' => $item->name,
        'description' => $item->description,
        'priceCents' => $item->price_cents,
        'isAvailable' => $item->is_available,
        'category' => $item->category?->name,
    ]);
```

Add `category: string | null` to the TypeScript `MenuItem` interface. Keep the
Inertia prop shape explicit instead of returning whole Eloquent models.

### 5. Prove the page works

Create a feature test that uses a factory and asserts that `/menu` contains the
item. Also test that unavailable items are excluded. Do not rely only on a
manual browser check.
