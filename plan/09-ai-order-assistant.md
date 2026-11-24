# Step 9: Optional AI menu assistant

**Days:** 41-45 (Week 9; optional extension)  
**Branch:** `feature/ai-menu-assistant`

## Why this is optional

Finish and test the core ordering site before adding AI. AI introduces provider
cost, privacy, availability, and prompt-injection concerns; it is not required
to learn Laravel, Inertia, or Vue.

For the first version, keep the integration in Laravel's HTTP client. Do not
add a Node microservice or Vercel AI SDK to the core course: they create a
second server stack and obscure the Laravel request flow. If learning Vercel AI
SDK is still a goal, build it later as a separate experiment after this
server-side version is understood.

Use an OpenAI-compatible provider such as OpenRouter only if you have access to
a currently available free model. Free model IDs, quotas, and availability
change; never promise that a free tier will remain usable. Do not send customer
or order data to the model.

## Goal

Build a rate-limited assistant that can answer short questions using only
currently available menu data. It cannot place, cancel, or change orders.

## Day 41: Decide whether it adds value

- Write three menu questions the existing menu UI does not answer well.
- Decide what public menu fields the assistant may use.
- If the core flow or tests are still failing, skip this optional branch and
  continue to Step 10.

**Checkpoint:** A short scope statement describes what the assistant can and
cannot do.

## Day 42: Configure a server-side provider

- Keep the API key only in the untracked local `.env`.
- Add the provider URL, key, and selected model to `config/services.php`.
- Verify that no key is placed in a `VITE_*` variable, page prop, or browser
  request.

**Checkpoint:** Laravel can read the configuration, and the key is absent from
source control and frontend output.

## Day 43: Add a bounded Laravel endpoint

- Require authentication and rate-limit requests.
- Validate the question length and send only available menu fields to the
  provider.
- Set a short timeout and return a clear error for missing credentials,
  provider rejection, empty output, or connection failure.
- Do not log prompts, API keys, or full provider responses.

**Checkpoint:** A fake-provider test covers a successful answer and at least
one failure without making a real API call.

## Day 44: Add the Vue form

- Build a small Inertia form with processing, answer, and error states.
- Render the response as plain text; do not treat model output as trusted HTML.
- Tell the user to confirm prices and availability on the menu page.

**Checkpoint:** Keyboard users can submit the form and understand its loading
and failure states.

## Day 45: Review safety and decide whether to keep it

- Test that request limits work and the endpoint cannot access order data.
- Check that unavailable menu items are not sent to the provider.
- Compare provider latency and reliability with the cost of maintaining the
  feature.
- Remove the branch from the release scope if it creates more risk than value.

**Checkpoint:** The feature is optional, bounded, and covered by fake-provider
tests; no test consumes a live API quota.

## Tutorial file map

```text
app/Http/Controllers/AiMenuAssistantController.php
config/services.php                 add provider configuration
routes/web.php                      add authenticated, rate-limited route
resources/js/pages/ai/Index.vue
tests/Feature/AiMenuAssistantTest.php
.env                                local only; never commit
```

## Part A: configure the provider

Add these local-only values to `.env`, choosing a model that is currently
available to your account:

```dotenv
OPENROUTER_API_KEY=replace-locally
OPENROUTER_MODEL=choose-a-current-model-id
```

Inside the array returned by `config/services.php`, add:

```php
'openrouter' => [
    'base_url' => env('OPENROUTER_BASE_URL', 'https://openrouter.ai/api/v1'),
    'api_key' => env('OPENROUTER_API_KEY'),
    'model' => env('OPENROUTER_MODEL'),
],
```

Never expose the key through `VITE_*` configuration or share it with Vue.

## Part B: validate, limit, and call from Laravel

In `routes/web.php`, import the controller and add both routes inside the
existing `auth` and `verified` group:

```php
Route::get('/ai/menu-assistant', [AiMenuAssistantController::class, 'index'])
    ->name('ai.index');

Route::post('/ai/menu-assistant', [AiMenuAssistantController::class, 'ask'])
    ->middleware('throttle:5,1')
    ->name('ai.ask');
```

Generate `app/Http/Controllers/AiMenuAssistantController.php` with Artisan:

```powershell
docker compose exec laravel.test php artisan make:controller AiMenuAssistantController
```

Add these imports at the top of the controller:

```php
use App\Models\MenuItem;
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;
use Inertia\Inertia;
use Inertia\Response;
```

Render the page with the one-request session answer:

```php
public function index(Request $request): Response
{
    return Inertia::render('ai/Index', [
        'answer' => $request->session()->get('aiAnswer'),
    ]);
}
```

In the controller, validate input, build context from current public menu data,
and call the provider. Keep the system instructions on the server:

```php
public function ask(Request $request): RedirectResponse
{
$data = $request->validate([
    'question' => ['required', 'string', 'max:500'],
]);

$apiKey = config('services.openrouter.api_key');
$model = config('services.openrouter.model');

if (! is_string($apiKey) || $apiKey === '' || ! is_string($model) || $model === '') {
    return back()->withErrors([
        'question' => 'The menu assistant is not configured.',
    ]);
}

$menu = MenuItem::query()
    ->where('is_available', true)
    ->orderBy('name')
    ->limit(50)
    ->get(['name', 'description', 'price_cents'])
    ->map(fn (MenuItem $item): array => [
        'name' => $item->name,
        'description' => $item->description,
        'price_cents' => $item->price_cents,
    ])
    ->all();

try {
    $response = Http::withToken($apiKey)
        ->acceptJson()
        ->timeout(15)
        ->post(rtrim(config('services.openrouter.base_url'), '/').'/chat/completions', [
            'model' => $model,
            'messages' => [
                [
                    'role' => 'system',
                    'content' => 'Answer briefly using only the supplied menu. '
                        .'Do not invent prices or claim to place orders.',
                ],
                [
                    'role' => 'user',
                    'content' => json_encode($menu, JSON_THROW_ON_ERROR)
                        ."\nQuestion: ".$data['question'],
                ],
            ],
            'max_tokens' => 200,
        ]);
} catch (ConnectionException $exception) {
    report($exception);

    return back()->withErrors([
        'question' => 'The menu assistant is temporarily unavailable.',
    ]);
}

if ($response->failed()) {
    Log::warning('Menu assistant provider request failed', [
        'status' => $response->status(),
    ]);

    return back()->withErrors([
        'question' => 'The menu assistant is temporarily unavailable.',
    ]);
}

$answer = $response->json('choices.0.message.content');

if (! is_string($answer) || trim($answer) === '') {
    return back()->withErrors([
        'question' => 'The menu assistant did not return an answer.',
    ]);
}

$request->session()->flash('aiAnswer', $answer);

return to_route('ai.index');
}
```
Confirm the provider's current response schema before using a different model
API. Never add automatic retries without considering duplicate provider cost.

## Part C: build a small accessible Vue page

Create `resources/js/pages/ai/Index.vue` using the project's existing form and
layout patterns. Define the `answer: string | null` prop and use
`useForm({ question: '' })` to post to the named route. Render the answer as
ordinary text and show validation/provider errors beside the form. Disable
submission while processing and do not use `v-html`.

The essential form behavior looks like this:

```vue
<script setup lang="ts">
import { useForm } from '@inertiajs/vue3';

const props = defineProps<{ answer: string | null }>();
const form = useForm({ question: '' });

function submit(): void {
    form.post('/ai/menu-assistant', {
        preserveScroll: true,
        onSuccess: () => form.reset('question'),
    });
}
</script>

<template>
    <form @submit.prevent="submit">
        <label for="question">Ask about the menu</label>
        <textarea
            id="question"
            v-model="form.question"
            maxlength="500"
            required
        />
        <p v-if="form.errors.question" role="alert">
            {{ form.errors.question }}
        </p>
        <button type="submit" :disabled="form.processing">
            {{ form.processing ? 'Thinking...' : 'Ask' }}
        </button>
        <p v-if="props.answer" role="status">{{ props.answer }}</p>
    </form>
</template>
```

The assistant is not authoritative. Link back to the menu and make the user
confirm current availability and prices there.

## Tests

In `tests/Feature/AiMenuAssistantTest.php`, follow the repository's PHPUnit
test-class style and use `Http::fake()`:

- Authenticated user receives a response from a fake provider.
- Unauthenticated requests are redirected to login.
- Invalid questions fail validation without an HTTP request.
- Provider failure produces a safe error and never exposes provider details.
- The test suite never calls OpenRouter or consumes a live quota.

For the successful response case, fake the provider before sending the
authenticated request:

```php
Http::fake([
    'https://openrouter.ai/api/v1/chat/completions' => Http::response([
        'choices' => [
            ['message' => ['content' => 'The soup is currently available.']],
        ],
    ]),
]);

config([
    'services.openrouter.api_key' => 'test-key',
    'services.openrouter.model' => 'test-model',
]);

$this->actingAs(User::factory()->create())
    ->post(route('ai.ask'), ['question' => 'Is the soup available?'])
    ->assertRedirect(route('ai.index'))
    ->assertSessionHas('aiAnswer', 'The soup is currently available.');
```

Add imports and the `RefreshDatabase` trait in the PHPUnit test class. Add
separate tests for validation and provider failure; use `Http::assertNothingSent()`
to prove invalid input never reaches the provider.

## Branch closeout

Review the changed diff, run the focused PHPUnit test and project checks, and
merge `feature/ai-menu-assistant` only if the optional feature is worth keeping.

## References

- [Laravel HTTP client](https://laravel.com/docs/13.x/http-client)
- [Laravel rate limiting](https://laravel.com/docs/13.x/rate-limiting)
- [OpenRouter quickstart](https://openrouter.ai/docs/quickstart)
- [OpenRouter FAQ](https://openrouter.ai/docs/faq)
