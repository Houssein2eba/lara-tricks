# Step 1: Workspace, Laravel foundations, and the learning method

**Days:** 1-5 (Week 1)  
**Branch:** `learn/laravel-foundations`  
**Time guide:** Plan for 60-120 focused minutes per day. If a checkpoint takes
longer, extend the schedule instead of skipping the test or review.

## Assumptions and finish line

This plan turns the existing Laravel Vue starter kit into a small, single-store
ordering site. Customers browse a menu, place demo orders, and track status;
staff process orders. Build and understand this core before adding email,
in-app notifications, live updates, or an optional AI menu helper.

Do not collect payment details, build a multi-vendor marketplace, or automate
order changes with AI. These features expand the domain without helping a
beginner master the core Laravel/Inertia/Vue request and data flow.

This is a 50-study-day course, not a promise of mastery in one month. By the
end, aim to independently trace, implement, test, and explain a vertical
feature from route to database to Vue UI. Mastery comes from repeating that
cycle on new features after the course.

## What this repository already has

- Laravel 13, PHP 8.3+, Inertia 3, Vue 3, and TypeScript.
- Laravel Fortify authentication, verification, passkeys, and two-factor
  authentication. Extend these rather than rebuilding login.
- A `Welcome.vue`, authenticated `Dashboard.vue`, settings pages, reusable UI
  components, and a toast notification composable.
- Vite Plus scripts for frontend build, checks, and TypeScript checks.
- Docker Compose with the `laravel.test` and MySQL services.
- PHPUnit-based feature tests; follow the existing test style rather than
  introducing Pest or another test framework.

The plan is for learning and incremental changes, not a second starter-kit
installation. Keep existing authentication and settings working.

## Branch map: 14 topic branches

Use one branch per reviewable topic, not per study day. Merge a branch into
`main` only after its tests and acceptance check pass. For a second branch in a
module, start from the updated `main`. The AI branch is optional and comes
after the core ordering flow works.

1. `learn/laravel-foundations`
2. `learn/vue-fundamentals`
3. `learn/inertia-workflow`
4. `feature/menu-catalog`
5. `feature/cart`
6. `feature/checkout`
7. `feature/order-workflow`
8. `feature/staff-dashboard`
9. `feature/email-notifications`
10. `feature/in-app-notifications`
11. `feature/live-order-updates`
12. `feature/ai-menu-assistant` (optional)
13. `test/core-flow-hardening`
14. `chore/demo-release`

This checkout currently has no Git repository. Before the first branch, make a
backup and confirm there is no remote or history to preserve. Then inspect
`.gitignore`, initialize Git, and verify that `.env`, `vendor`, `node_modules`,
and generated build output are not staged. Never commit credentials or local
database data.

## 10-week roadmap

| Week | Days | Focus | Exit checkpoint |
|---|---:|---|---|
| 1 | 1-5 | Workspace, PHP/Laravel request flow, tests, Git | Explain one request and run one focused test |
| 2 | 6-10 | Vue 3, TypeScript, component contracts | Build a typed interactive component |
| 3 | 11-15 | Inertia pages, props, navigation, forms | Submit a form and display server validation |
| 4 | 16-20 | Migrations, Eloquent, factories, catalog | Test a database-backed menu |
| 5 | 21-25 | Session cart and transactional checkout | Create a server-priced order safely |
| 6 | 26-30 | Order state, staff authorization and queue | Prove ownership and transition rules |
| 7 | 31-35 | Mailpit, queued email and in-app notices | Test delivery behavior without real mail |
| 8 | 36-40 | Polling first; optional Reverb/Live Props | Explain and test the chosen update strategy |
| 9 | 41-45 | Optional server-side AI menu helper | Fake-provider tests; skip if core is unfinished |
| 10 | 46-50 | Hardening, documentation and demo | A reproducible demo and retrospective |

## Daily learning loop

1. Recall yesterday's idea without looking at the notes; write down one
   question you still have.
2. Read only the relevant official documentation linked from the step.
3. Write the smallest failing test or reproduce the behavior manually.
4. Implement one small change; trace the data through Laravel, Inertia, and Vue.
5. Run the focused test/check, inspect the diff, and explain the result aloud.
6. End each week by revisiting an earlier feature and improving one thing
   without copying a tutorial.

## PowerShell project commands

Run these from the project root. This Compose file builds from Sail files under
`vendor/`, so a fresh clone must have Composer dependencies before the first
image build. If that path is missing, bootstrap dependencies without Composer
scripts on Windows, then run normal Composer inside the container:

```powershell
if (-not (Test-Path .\.env)) {
    Copy-Item .\.env.example .\.env
}
if (-not (Test-Path .\vendor\laravel\sail\runtimes\8.5\Dockerfile)) {
    composer install --no-scripts
}
docker compose up -d --build
docker compose exec laravel.test composer install
# Run only once for a fresh .env with an empty APP_KEY.
docker compose exec laravel.test php artisan key:generate
docker compose exec laravel.test php artisan migrate
docker compose exec laravel.test npm install
docker compose exec laravel.test npm run build
docker compose exec laravel.test composer ci:check
```

`--no-scripts` avoids running Laravel's npm-related Composer hooks on Windows
while the container does not exist yet. After startup, regular Composer and
npm commands run inside Linux. Run `key:generate` only when the copied `.env`
has no `APP_KEY`; do not rotate an existing key on every startup. Wait for
MySQL to become healthy before running migrations.

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

### Day 1: boot and record a baseline

From PowerShell at the repository root:

```powershell
if (-not (Test-Path .\.env)) {
    Copy-Item .\.env.example .\.env
}
if (-not (Test-Path .\vendor\laravel\sail\runtimes\8.5\Dockerfile)) {
    composer install --no-scripts
}
docker compose up -d --build
docker compose ps
docker compose exec laravel.test composer install
# Run only once for a fresh .env with an empty APP_KEY.
docker compose exec laravel.test php artisan key:generate
docker compose exec laravel.test php artisan migrate
docker compose exec laravel.test php artisan about
docker compose exec laravel.test php artisan route:list
docker compose exec laravel.test npm install
docker compose exec laravel.test composer ci:check
```

In the output, identify the `laravel.test` and `mysql` services. Open the
configured app URL and note which route renders the page. Never run
`docker compose down --volumes` during this course unless you intentionally
want to erase the local database volume. Record the first failing command as a
baseline; do not hide or weaken existing checks to make the plan pass.

### Day 2: trace a request

Trace `/` and the authenticated dashboard through `routes/web.php`, their
controller or route closure, the Inertia page name, and the Vue page. Use:

```powershell
docker compose exec laravel.test php artisan route:list
docker compose exec laravel.test php artisan about
```

Draw the request path in your own words. Identify which code runs in PHP,
which runs in the browser, and what Inertia transports between them.

### Day 3: PHP and Laravel basics

Read a small PHP 8.3 language section on arrays, types, functions, and classes.
Then inspect an existing controller and explain its request, validation,
response type, and imports. Avoid changing authentication or starter-kit
scaffolding.

### Day 4: write and run a focused feature test

Read `tests/Feature/DashboardTest.php`. Add one assertion to an existing
appropriate test or write a small PHPUnit feature test for a harmless route.
Run only that test first, then the whole existing suite. Keep the current
`Tests\TestCase` style.

### Day 5: initialize safe Git habits

After confirming a backup and no existing history to preserve, inspect
`.gitignore`. If this directory has no Git history, initialize it with
`git init -b main`; otherwise keep the existing repository and branch
conventions. Check `git status`, stage only intended files, inspect
`git diff --cached`, and create the first course branch:

```powershell
git switch -c learn/laravel-foundations
```

Commit only the reviewed baseline/course changes. Never stage `.env`, API keys,
mail credentials, `vendor`, `node_modules`, or generated build artifacts.

## End-of-step exercise

Without looking at the code, explain:

1. What runs in a Laravel controller versus a Vue page?
2. Which directory stores a migration?
3. Why do application PHP/Node commands run inside Compose, and why can a
   no-scripts Composer bootstrap be needed before the first image build?
4. What is the difference between stopping containers and deleting volumes?

**Pass condition:** the app and baseline checks are understood; you can explain
one request path and one test; Git is safe to use; no secret is staged.
