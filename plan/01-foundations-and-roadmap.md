# Step 1: Foundations and roadmap

**Days:** 1-3  
**Branch:** `learn/laravel-foundations`  
**Time guide:** Aim for 60-90 minutes a day. Read a small section, implement it,
then explain the result in your own words.

## Assumptions and finish line

This plan turns the existing Laravel Vue starter kit into a small, single-store
ordering site. Customers browse a menu, place orders, and track status. Staff
manage incoming orders. The finished demo has email and in-app notifications,
near-real-time updates, and a carefully scoped AI menu/order assistant.

Payments, multi-vendor shops, inventory accounting, and production-grade AI
automation are out of scope for this first month. Orders are demo orders; no
real payment details are collected.

By Day 30, the learner should be able to trace a browser request through a
Laravel route/controller, Eloquent, Inertia props, and a Vue page; build and
test a feature; and explain the main trade-offs in the app.

## What this repository already has

- Laravel 13, PHP 8.3+, Inertia 3, Vue 3, and TypeScript.
- Laravel Fortify authentication, verification, passkeys, and two-factor
  authentication. Extend these rather than rebuilding login.
- A `Welcome.vue`, authenticated `Dashboard.vue`, settings pages, reusable UI
  components, and a toast notification composable.
- Vite Plus scripts for frontend build, checks, and TypeScript checks.
- Docker Compose with the `laravel.test` and MySQL services.

The plan is for learning and incremental changes, not a second starter-kit
installation. Keep existing authentication and settings working.

## Branch map: 15 topic branches

Create each branch when its step starts, merge it into `main` after its
acceptance checks pass, then create the next branch from updated `main`.
Several three-day steps intentionally have two small, separate topics. Do not
try to work on all branches at once.

1. `learn/laravel-foundations`
2. `learn/vue-fundamentals`
3. `learn/inertia-pages-props`
4. `feature/menu-catalog`
5. `feature/cart`
6. `feature/checkout`
7. `feature/order-workflow`
8. `feature/staff-dashboard`
9. `feature/live-order-updates`
10. `feature/email-notifications`
11. `feature/in-app-notifications`
12. `feature/ai-provider`
13. `feature/ai-order-assistant`
14. `test/e2e-hardening`
15. `chore/demo-release`

If Git is not initialized for this checkout, first confirm that the files are
backed up and no existing remote/history needs to be preserved. Initialize or
connect the repository only after that check. Never commit `.env`, API keys,
mail credentials, or other secrets.

## Daily learning loop

1. Write down one question before coding.
2. Read the relevant official documentation linked from each step.
3. Make one small change and inspect the request/data flow.
4. Add or run a check that proves the behavior.
5. Write a short note: what changed, what failed first, and why the final
   approach works.

## PowerShell project commands

Run these from the project root. Start the services before using `exec`:

```powershell
docker compose up -d
docker compose exec laravel.test php artisan migrate
docker compose exec laravel.test npm install
docker compose exec laravel.test npm run build
```

Use the port configured by `APP_PORT` in `.env`; otherwise the current Compose
file defaults to `http://localhost`. The `APP_URL` value alone does not publish
a Docker port.

## References

- [Laravel 13 documentation](https://laravel.com/docs/13.x)
- [Vue guide](https://vuejs.org/guide/)
- [Inertia documentation](https://inertiajs.com/)
- [Laravel testing](https://laravel.com/docs/13.x/testing)

## Tutorial: prepare the workspace

### Folder map

Learn what each existing directory is for before adding files:

```text
app/Http/Controllers/       server request handlers
app/Models/                Eloquent models
database/migrations/       versioned database schema
database/factories/        repeatable test data
database/seeders/          local demo data
routes/web.php             browser routes
resources/js/pages/        Inertia page components
resources/js/components/   reusable Vue components
resources/js/types/        shared TypeScript types
tests/Feature/             request and business-flow tests
plan/                      this learning course
```

Follow an existing file in the same folder before creating a new pattern. For
example, compare a new controller with
`app/Http/Controllers/Settings/ProfileController.php` and a new page with
`resources/js/pages/Dashboard.vue`.

### Day 1: boot and inspect

From PowerShell at the repository root:

```powershell
docker compose up -d --build
docker compose ps
docker compose exec laravel.test php artisan about
docker compose exec laravel.test php artisan route:list
```

In the output, identify the `laravel.test` and `mysql` services. Open the
configured app URL and note which route renders the page. Never run
`docker compose down --volumes` during this course unless you intentionally
want to erase the local database volume.

### Day 2: shorten Artisan commands

For the current PowerShell session, define:

```powershell
function art {
    docker compose exec laravel.test php artisan @args
}

art about
art route:list
```

To keep the helper in future PowerShell sessions, run `notepad $PROFILE`, paste
the function into the profile, save it, and reload with `. $PROFILE`. The
function only shortens Artisan commands; use `docker compose` for service
operations.

### Day 3: establish Git habits

Check `git status` before changing anything. If this checkout has no Git
history, confirm that with the repository owner before running `git init`.
Once a `main` branch exists, create the first course branch:

```powershell
git switch -c learn/laravel-foundations
```

For each feature, make a small commit after a working checkpoint. Before
committing, inspect `git diff` and `git status`; make sure `.env`, keys, and
local database files are not included.

## End-of-step exercise

Without looking at the code, explain:

1. What runs in a Laravel controller versus a Vue page?
2. Which directory stores a migration?
3. Why does this project run PHP and npm through `docker compose exec`?
4. What is the difference between stopping containers and deleting volumes?

**Pass condition:** the app starts, the route list is readable, Git status is
understood, and no secret or local database file is staged.
