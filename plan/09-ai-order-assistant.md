# Step 9: AI provider and order assistant

**Days:** 25-27  
**Branches:** `feature/ai-provider` (Days 25-26), then
`feature/ai-order-assistant` (Day 27)

## Goal

Build one narrow AI feature that helps a customer understand the menu and their
own order. AI must not be trusted to change an order or bypass authorization.

## Day 25: Provider and architecture spike

- Read the AI SDK Vue UI documentation and OpenRouter's current free-model
  documentation.
- Use Vercel AI SDK's Vue package for the chat UI. Keep all provider credentials
  server-side; never put an API key in a `VITE_` variable or browser bundle.
- Since this app's server is PHP, choose and document a small server-side
  adapter before implementation. The recommended learning path is a private
  Node service using the AI SDK, with Laravel authenticating/authorizing and
  proxying the request. Do not expose the Node service or API key directly to
  the browser.
- Use OpenRouter's OpenAI-compatible API and select an available `:free` model
  dynamically from its current catalog. Free models and quotas change; its
  FAQ currently states a default limit of 50 free-model requests per day
  without purchased credits. Verify the live limit before each session; do not
  build production availability assumptions around a free tier.

**Checkpoint:** A local-only spike returns one assistant answer. No key is in
source control, Vite env, page props, or browser network headers.

## Day 26: `feature/ai-provider`

- Continue the provider branch and create a server-side provider adapter.
- Store the key in an untracked `.env` entry and pass it only to the server
  container.
- Add timeouts, input length limits, server-side rate limiting, and clear
  provider error handling.
- Log request metadata without logging secrets or unnecessary customer
  messages.

**Checkpoint:** A missing key, timeout, rate limit, or provider error produces a
useful safe response, never a fake successful answer.

## Day 27: `feature/ai-order-assistant`

- Merge `feature/ai-provider`, then create `feature/ai-order-assistant` from
  updated `main`.
- Add a Vue chat panel with loading, streaming/response, empty, and error states.
- Limit the assistant to menu questions and, for an authenticated owner, a
  read-only lookup of that user's order status.
- Validate any tool arguments on the server. Re-check user ownership before
  every order lookup.
- State that the answer may be wrong; link to the authoritative order screen.
- Do not allow the model to cancel, submit, refund, or modify an order.

**Checkpoint:** Tests prove the AI endpoint is authenticated/rate-limited as
designed and cannot read another user's order.

## Deliverable

A small, clearly labeled menu/order helper that degrades gracefully when the
free provider is unavailable.

## Branch closeout

Review privacy, cost limits, provider key handling, and authorization. Merge
`feature/ai-provider`, then `feature/ai-order-assistant`.

## References

- [AI SDK UI overview and Vue support](https://ai-sdk.dev/docs/ai-sdk-ui/overview)
- [AI SDK chatbot UI](https://ai-sdk.dev/docs/ai-sdk-ui/chatbot)
- [AI SDK OpenAI-compatible providers](https://ai-sdk.dev/providers/openai-compatible-providers)
- [OpenRouter quickstart](https://openrouter.ai/docs/quickstart)
- [OpenRouter free-model rate limits and privacy FAQ](https://openrouter.ai/docs/faq)

## Tutorial file map

The recommended capstone keeps the AI key outside the browser:

```text
ai-service/package.json
ai-service/package-lock.json
ai-service/Dockerfile
ai-service/src/server.ts
app/Http/Controllers/AiChatController.php
config/services.php             add the ai endpoint/token config
routes/web.php
resources/js/components/ai/OrderAssistant.vue
compose.yaml                    add an optional internal ai service
.env                         local only; never commit
```

This is intentionally an advanced optional service: the application server is
PHP, while the Vercel AI SDK is TypeScript. Build and secure the narrow provider
spike before adding tools or order lookups.

## Part A: create the private Node service

Create the folders from PowerShell:

```powershell
New-Item -ItemType Directory -Force ai-service\src
docker compose exec laravel.test npm init -y --prefix ai-service
docker compose exec laravel.test npm install --prefix ai-service ai @ai-sdk/openai-compatible hono @hono/node-server zod
docker compose exec laravel.test npm install --prefix ai-service --save-dev typescript tsx @types/node
docker compose exec laravel.test npm install ai @ai-sdk/vue
```

This writes `ai-service/package.json` and `package-lock.json`; commit both so
the image build is repeatable. Merge these entries into the generated package
file, preserving its installed dependencies:

```json
"type": "module",
"scripts": {
    "start": "tsx src/server.ts"
}
```

In `ai-service/Dockerfile`:

```dockerfile
FROM node:24-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY src ./src
CMD ["npm", "run", "start"]
```

Keep the AI service on the Compose network only; do not publish its port to the
host. Add local `.env` entries (values are placeholders, not real secrets):

```dotenv
OPENROUTER_API_KEY=replace-locally
OPENROUTER_MODEL=choose-an-available-model-id
AI_SERVICE_TOKEN=replace-with-a-long-random-local-token
AI_SERVICE_URL=http://ai:3000
```

In `compose.yaml`, add the private, opt-in service and pass the server-only
values. Add the two `AI_SERVICE_*` mappings to the existing `laravel.test`
environment so Laravel can proxy authenticated requests:

```yaml
services:
    laravel.test:
        environment:
            AI_SERVICE_URL: '${AI_SERVICE_URL:-http://ai:3000}'
            AI_SERVICE_TOKEN: '${AI_SERVICE_TOKEN:-}'
    ai:
        build:
            context: './ai-service'
        profiles: ['ai']
        expose:
            - '3000'
        environment:
            OPENROUTER_API_KEY: '${OPENROUTER_API_KEY:-}'
            OPENROUTER_MODEL: '${OPENROUTER_MODEL:-}'
            AI_SERVICE_TOKEN: '${AI_SERVICE_TOKEN:-}'
        networks:
            - sail
```

The `ai` profile keeps the optional service from starting during ordinary app
work. Start it only when studying this step:

```powershell
docker compose --profile ai up -d --build
```

Stop the optional service when you finish to release its memory:

```powershell
docker compose stop ai
```

Select an available `:free` model from the live OpenRouter catalog. Free model
names, availability, and request quotas can change. Do not put these entries in
`VITE_*` variables or commit them.

## Part B: create a small AI SDK endpoint

In `ai-service/src/server.ts`, the important server-side shape is:

```ts
import { serve } from '@hono/node-server';
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';
import {
    convertToModelMessages,
    createUIMessageStreamResponse,
    streamText,
    toUIMessageStream,
    type UIMessage,
} from 'ai';
import { Hono } from 'hono';

const app = new Hono();
const serviceToken = process.env.AI_SERVICE_TOKEN;
const apiKey = process.env.OPENROUTER_API_KEY;
const modelId = process.env.OPENROUTER_MODEL;

if (!serviceToken || !apiKey || !modelId) {
    throw new Error('AI service environment is incomplete.');
}

const provider = createOpenAICompatible({
    name: 'openrouter',
    baseURL: 'https://openrouter.ai/api/v1',
    apiKey,
});

app.post('/chat', async (context) => {
    if (context.req.header('X-AI-Service-Token') !== serviceToken) {
        return context.json({ message: 'Forbidden' }, 403);
    }

    const body = await context.req.json<{ messages: UIMessage[] }>();
    if (!Array.isArray(body.messages) || body.messages.length > 12) {
        return context.json({ message: 'Invalid conversation.' }, 422);
    }

    const result = streamText({
        model: provider(modelId),
        system: 'Answer menu questions briefly. Never change an order.',
        messages: await convertToModelMessages(body.messages),
        maxOutputTokens: 300,
    });

    return createUIMessageStreamResponse({
        stream: toUIMessageStream({ stream: result.stream }),
    });
});

serve({ fetch: app.fetch, port: 3000 });
```

Before using this sample, verify package APIs against the installed AI SDK
version. Add schema validation for every message and request size; the short
example is not a substitute for that validation.

## Part C: protect the browser-to-model path

The Vue page should send chat requests to an authenticated Laravel route, not
directly to OpenRouter or the private AI service. In
`routes/web.php`, add an authenticated, rate-limited endpoint:

```php
Route::post('/ai/chat', AiChatController::class)
    ->middleware(['auth', 'verified', 'throttle:10,1'])
    ->name('ai.chat');
```

Add this entry inside the existing returned array in `config/services.php`:

```php
'ai' => [
    'url' => env('AI_SERVICE_URL', 'http://ai:3000'),
    'token' => env('AI_SERVICE_TOKEN'),
],
```

In `AiChatController`, validate request size, authorize the customer, and
forward to the private service. The controller must stream the response rather
than return the provider key or buffer a long response:

```php
$data = $request->validate([
    'messages' => ['required', 'array', 'min:1', 'max:12'],
]);

$upstream = Http::withOptions(['stream' => true])
    ->withHeaders([
        'X-AI-Service-Token' => config('services.ai.token'),
    ])
    ->timeout(30)
    ->post(rtrim(config('services.ai.url'), '/').'/chat', $data);

if ($upstream->failed()) {
    return response()->json(['message' => 'AI service unavailable.'], 502);
}

$body = $upstream->toPsrResponse()->getBody();

return response()->stream(function () use ($body): void {
    while (! $body->eof()) {
        echo $body->read(1024);
        if (ob_get_level() > 0) {
            ob_flush();
        }
        flush();
    }
}, $upstream->status(), [
    'Content-Type' => $upstream->header('Content-Type', 'text/event-stream'),
    'Cache-Control' => 'no-cache',
]);
```

Add imports and handle timeouts/provider failures with explicit server logging.
Validate each message's shape and maximum text length before forwarding. Never
trust a user ID, order ID, or tool result supplied by the model.

In `resources/js/components/ai/OrderAssistant.vue`, the Vue SDK UI begins like
this:

```vue
<script setup lang="ts">
import { useChat } from '@ai-sdk/vue';
import { DefaultChatTransport } from 'ai';
import { ref } from 'vue';

const input = ref('');
const { messages, sendMessage, status } = useChat({
    transport: new DefaultChatTransport({
        api: '/ai/chat',
        fetch: (input, init) => {
            const cookie = document.cookie
                .split('; ')
                .find((part) => part.startsWith('XSRF-TOKEN='));
            const headers = new Headers(init?.headers);

            if (cookie) {
                headers.set(
                    'X-XSRF-TOKEN',
                    decodeURIComponent(cookie.slice('XSRF-TOKEN='.length)),
                );
            }

            return fetch(input, {
                ...init,
                headers,
                credentials: 'same-origin',
            });
        },
    }),
});

function submit(): void {
    const text = input.value.trim();
    if (text === '' || status.value !== 'ready') {
        return;
    }

    sendMessage({ text });
    input.value = '';
}
</script>
```

Render only text parts from the assistant and show loading/error states.
Confirm the current Vue SDK's ref/template behavior against its installed
version before finishing the component.

## Safe tool design

Start with menu descriptions. If adding a read-only order-status tool, have
Laravel verify `order.user_id === request.user().id` on every invocation and
return only status and timestamps. Do not let the model cancel, submit, refund,
or modify orders. Test the API with a fake provider response; do not spend free
API quota in the normal test suite.
