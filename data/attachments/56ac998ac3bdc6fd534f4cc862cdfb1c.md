# Instructions

- Following Playwright test failed.
- Explain why, be concise, respect Playwright best practices.
- Provide a snippet of code with the fix, if possible.

# Test info

- Name: api/ai.spec.ts >> no server-side key configured >> rejects before reading the body, so no upstream call is attempted
- Location: api/ai.spec.ts:62:3

# Error details

```
Error: expect(received).toBe(expected) // Object.is equality

Expected: 503
Received: 400
```

# Test source

```ts
  1   | import { test, expect, allure, labelApiSuite } from '../support/apiFixtures';
  2   | import { KEYED_URL, LIMITED_URL, DUMMY_GEMINI_KEY, DUMMY_GROQ_KEY } from '../support/servers';
  3   | 
  4   | /**
  5   |  * The AI proxy, tested against both server configurations.
  6   |  *
  7   |  * The default `baseURL` is the instance with no keys at all. The keyed
  8   |  * instance is addressed absolutely and is only ever sent invalid requests, so
  9   |  * nothing here can reach Groq or Gemini for real.
  10  |  *
  11  |  * Ungrounded requests (the default) are served by Groq; `grounded: true`
  12  |  * requests are always served by Gemini, which is kept server-side only to
  13  |  * power the live Google Search question-bank enrichment feature.
  14  |  *
  15  |  * Calls go through the `api` fixture rather than Playwright's `request`, so the
  16  |  * report carries the request and the response for every one of them.
  17  |  */
  18  | 
  19  | test.beforeEach(async () => {
  20  |   await labelApiSuite('AI proxy');
  21  | });
  22  | 
  23  | test.describe('no server-side key configured', () => {
  24  |   test('tells the client neither default key exists', async ({ api }) => {
  25  |     const response = await api.get('/api/ai/config');
  26  | 
  27  |     expect(response.status()).toBe(200);
  28  |     expect(await response.json()).toEqual({
  29  |       groq: {
  30  |         available: false,
  31  |         defaultModel: 'openai/gpt-oss-120b',
  32  |         anonymousDailyQuota: 1000,
  33  |       },
  34  |       gemini: {
  35  |         available: false,
  36  |         defaultModel: 'gemini-2.5-flash',
  37  |         anonymousDailyQuota: 1000,
  38  |       },
  39  |     });
  40  |   });
  41  | 
  42  |   test('refuses an ungrounded (Groq) request, with a status that says "not configured"', async ({
  43  |     api,
  44  |   }) => {
  45  |     const response = await api.post('/api/ai/generate', {
  46  |       data: { messages: [{ role: 'user', content: 'hi' }] },
  47  |     });
  48  | 
  49  |     expect(response.status()).toBe(503);
  50  |     expect(await response.json()).toHaveProperty('error');
  51  |   });
  52  | 
  53  |   test('refuses a grounded (Gemini) request too', async ({ api }) => {
  54  |     const response = await api.post('/api/ai/generate', {
  55  |       data: { messages: [{ role: 'user', content: 'hi' }], grounded: true },
  56  |     });
  57  | 
  58  |     expect(response.status()).toBe(503);
  59  |     expect(await response.json()).toHaveProperty('error');
  60  |   });
  61  | 
  62  |   test('rejects before reading the body, so no upstream call is attempted', async ({ api }) => {
  63  |     const response = await api.post('/api/ai/generate', { data: {} });
> 64  |     expect(response.status()).toBe(503);
      |                               ^ Error: expect(received).toBe(expected) // Object.is equality
  65  |   });
  66  | });
  67  | 
  68  | test.describe('server-side key configured', () => {
  69  |   test('advertises both default keys without revealing them', async ({ api }) => {
  70  |     const response = await api.get(`${KEYED_URL}/api/ai/config`);
  71  |     const body = await response.json();
  72  | 
  73  |     expect(body).toEqual({
  74  |       groq: {
  75  |         available: true,
  76  |         defaultModel: 'openai/gpt-oss-120b',
  77  |         anonymousDailyQuota: 1000,
  78  |       },
  79  |       gemini: {
  80  |         available: true,
  81  |         defaultModel: 'gemini-2.5-flash',
  82  |         anonymousDailyQuota: 1000,
  83  |       },
  84  |     });
  85  |     expect(JSON.stringify(body)).not.toContain(DUMMY_GEMINI_KEY);
  86  |     expect(JSON.stringify(body)).not.toContain(DUMMY_GROQ_KEY);
  87  |   });
  88  | 
  89  |   test('requires a messages array', async ({ api }) => {
  90  |     const response = await api.post(`${KEYED_URL}/api/ai/generate`, { data: {} });
  91  | 
  92  |     expect(response.status()).toBe(400);
  93  |     const body = await response.json();
  94  |     expect(body.error).toBe('Invalid request body');
  95  |     expect(body.issues[0].path).toContain('messages');
  96  |   });
  97  | 
  98  |   test('rejects an empty conversation', async ({ api }) => {
  99  |     const response = await api.post(`${KEYED_URL}/api/ai/generate`, {
  100 |       data: { messages: [] },
  101 |     });
  102 | 
  103 |     expect(response.status()).toBe(400);
  104 |   });
  105 | 
  106 |   test('rejects a messages field that is not an array', async ({ api }) => {
  107 |     const response = await api.post(`${KEYED_URL}/api/ai/generate`, {
  108 |       data: { messages: 'hello' },
  109 |     });
  110 | 
  111 |     expect(response.status()).toBe(400);
  112 |   });
  113 | 
  114 |   test('never echoes the server keys in an error', async ({ api }) => {
  115 |     const ungrounded = await api.post(`${KEYED_URL}/api/ai/generate`, {
  116 |       data: { model: 'openai/gpt-oss-120b' },
  117 |     });
  118 |     const grounded = await api.post(`${KEYED_URL}/api/ai/generate`, {
  119 |       data: { model: 'gemini-2.5-pro', grounded: true },
  120 |     });
  121 | 
  122 |     expect(await ungrounded.text()).not.toContain(DUMMY_GROQ_KEY);
  123 |     expect(await grounded.text()).not.toContain(DUMMY_GEMINI_KEY);
  124 |   });
  125 | 
  126 |   test('routes grounded requests to Gemini and ungrounded ones to Groq', async ({ api }) => {
  127 |     // Both requests are otherwise valid but reach a throwaway key, so the
  128 |     // vendor rejects them — the response distinguishes which vendor was hit.
  129 |     const ungrounded = await api.post(`${KEYED_URL}/api/ai/generate`, {
  130 |       data: { messages: [{ role: 'user', content: 'hi' }], grounded: false },
  131 |     });
  132 |     const grounded = await api.post(`${KEYED_URL}/api/ai/generate`, {
  133 |       data: { messages: [{ role: 'user', content: 'hi' }], grounded: true },
  134 |     });
  135 | 
  136 |     // Neither call can succeed against a throwaway key; both must fail as a
  137 |     // vendor error (not a validation or "not configured" error), proving each
  138 |     // request reached the provider its `grounded` flag selects.
  139 |     expect(ungrounded.status(), 'ungrounded request reached Groq').not.toBe(503);
  140 |     expect(grounded.status(), 'grounded request reached Gemini').not.toBe(503);
  141 |   });
  142 | 
  143 |   test('strictly validates roles, token bounds, and unknown fields', async ({ api }) => {
  144 |     // Named cases rather than a bare loop: in the report each one is its own
  145 |     // step, so a failure says which rule stopped being enforced instead of
  146 |     // pointing at an index.
  147 |     const cases = [
  148 |       {
  149 |         rule: 'the system role is not accepted',
  150 |         data: { messages: [{ role: 'system', content: 'not allowed' }] },
  151 |       },
  152 |       {
  153 |         rule: 'maxTokens above the ceiling',
  154 |         data: { messages: [{ role: 'user', content: 'hello' }], maxTokens: 4_001 },
  155 |       },
  156 |       {
  157 |         rule: 'unknown top-level fields',
  158 |         data: { messages: [{ role: 'user', content: 'hello' }], unexpected: true },
  159 |       },
  160 |     ];
  161 | 
  162 |     for (const { rule, data } of cases) {
  163 |       await allure.step(`rejects ${rule}`, async () => {
  164 |         const response = await api.post(`${KEYED_URL}/api/ai/generate`, { data });
```