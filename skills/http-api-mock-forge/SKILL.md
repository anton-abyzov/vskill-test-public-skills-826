---
name: http-api-mock-forge
version: 1.2.0
description: Stand up deterministic HTTP API mocks for tests and local dev using MSW, Nock, or Prism — request matching, stateful handlers, and contract fidelity.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - testing
  - http
  - mocking
  - api
  - msw
---
# http-api-mock-forge

Replace real upstream HTTP calls with deterministic mocks so tests are fast,
offline, and repeatable — without losing contract fidelity.

## Pick the right tool

| Tool          | Best for                                  | Layer it intercepts        |
|---------------|-------------------------------------------|----------------------------|
| **MSW**       | browser + node app code, shared handlers  | `fetch`/XHR via service worker / node interceptor |
| **Nock**      | node-only, low-level HTTP assertions      | node `http`/`https` module |
| **Prism**     | mock straight from an OpenAPI spec         | standalone HTTP server     |
| **WireMock**  | language-agnostic, JVM-friendly services  | standalone HTTP server     |

Rule of thumb: if you have an OpenAPI document, generate the mock from it
(Prism/WireMock) so the mock can never drift from the contract. Otherwise use
MSW for app-level tests.

## MSW handler example

```ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('https://api.acme.example/v1/users/:id', ({ params }) => {
    if (params.id === '404') {
      return new HttpResponse(null, { status: 404 });
    }
    return HttpResponse.json({ id: params.id, name: 'Ada' });
  }),

  http.post('https://api.acme.example/v1/users', async ({ request }) => {
    const body = await request.json();
    if (!body.name) {
      return HttpResponse.json({ error: 'name required' }, { status: 422 });
    }
    return HttpResponse.json({ id: 'usr_001', ...body }, { status: 201 });
  }),
);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Setting `onUnhandledRequest: 'error'` is key: it fails the test the moment code
hits an un-mocked endpoint, catching accidental real network calls.

## Stateful mocks

For flows like "create then fetch", keep state inside the handler closure and
reset it per test:

```ts
let store = new Map();
afterEach(() => { store = new Map(); });

http.post('/v1/items', async ({ request }) => {
  const item = { id: `itm_${store.size + 1}`, ...(await request.json()) };
  store.set(item.id, item);
  return HttpResponse.json(item, { status: 201 });
});
http.get('/v1/items/:id', ({ params }) =>
  store.has(params.id)
    ? HttpResponse.json(store.get(params.id))
    : new HttpResponse(null, { status: 404 }));
```

## Fidelity checklist

- Match **method + path + query + relevant headers**, not just the path.
- Mirror real **status codes and error bodies**, including 4xx/5xx, so error
  handling is actually exercised.
- Reproduce **pagination, rate-limit headers, and latency** where the code
  depends on them (use a small delay helper for timeouts/retries).
- Keep fixtures close to real payloads; trim only irrelevant fields.

## Anti-patterns

- Mocking your own code instead of the network boundary — mock at the HTTP edge.
- Over-broad matchers (`*`) that hide real request bugs.
- Letting mocks drift from the contract. Regenerate from OpenAPI when the spec
  changes, or add a contract test that runs against the real sandbox in CI.

## Agent guidance

- Default to MSW for app tests; Prism when an OpenAPI spec exists.
- Always assert on the request the code sent, not only on the response.
- Fail on unhandled requests so silent real calls cannot slip through.
