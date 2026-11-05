# Cloudflare Workers 403 + 503 despite correct R2 bindings and HMAC? Check `fetch(request, env, ctx)`

**Keywords:** Cloudflare Workers, `403 Forbidden`, `503 Service Unavailable`, `ExecutionContext`, `fetch(request, env, ctx)`, HMAC signature validation, R2 binding, Wrangler, `workerd`, `Number.isFinite(now)`, JavaScript default parameters.

> **Solved offline (November 5, 2026):** A Worker returned a generic **403** to a valid HMAC-signed diagnostic GET and **503** to a readiness GET, despite correct deployment metadata and R2 bindings. The cause was passing Cloudflare's `ExecutionContext` object into a function that expected a numeric clock. A one-function adapter fixed both symptoms in local `workerd` tests. **Hosted verification had not yet been performed at the time of writing.**

## Symptoms

A temporary, deliberately limited Worker test configuration appeared correct. Deployment metadata showed the expected lease variables, R2 binding, and signing-key binding. Still:

| Request | Before fix | After fix in local `workerd` |
| --- | --- | --- |
| Correctly HMAC-signed diagnostic `GET` | `403 BLOCKED` | `200`, expected diagnostic response |
| Unsigned request to a deliberately invalid route during an *enabled test lease* | `503 BLOCKED` | `403 BLOCKED` (expected invalid-route rejection) |
| Request while Worker is permanently disabled | `503 BLOCKED` | `503 BLOCKED` (safety behavior unchanged) |

The 403/503 statuses were application-generated fail-closed responses. They should **not** be assumed to represent a Cloudflare WAF block or an R2 outage in other applications.

## Root cause: the wrong third argument

Cloudflare's module Worker contract is:

```js
export default {
  async fetch(request, env, ctx) {
    // ctx is an ExecutionContext, NOT a timestamp.
  }
};
```

Our internal, testable handler used a third parameter as an injectable number:

```js
export async function bridgeFetch(request, env, now = Date.now()) {
  // Lease expiration and HMAC timestamp validation use now.
}
```

This export looked innocent, but was wrong:

```js
// BUG: Cloudflare injects ctx as now.
export default { fetch: bridgeFetch };
```

A JavaScript default parameter (`now = Date.now()`) runs only when the argument is **missing or `undefined`**. Cloudflare supplied an `ExecutionContext` object as argument 3, so `now` was that object—not a timestamp.

The logic contained defensive checks similar to:

```js
if (!Number.isFinite(now)) {
  return new Response("BLOCKED", { status: 503 });
}
```

Elsewhere, authenticated diagnostic validation rejected the nonnumeric `now` and returned a generic **403**. The HMAC was correct; timestamp validation was broken by argument forwarding.

## Minimal fix

Separate the **platform-facing** method signature from the **application-facing** one:

```js
export async function bridgeFetch(request, env, now = Date.now()) {
  // Existing lease, HMAC and storage logic is unchanged.
}

export default {
  fetch(request, env) {
    return bridgeFetch(request, env, Date.now());
  }
};
```

If you need `ctx.waitUntil` or another context method, accept `ctx` and use it **only as an execution context**:

```js
export default {
  fetch(request, env, ctx) {
    // ctx.waitUntil(promise) can be used here if appropriate.
    return bridgeFetch(request, env, Date.now());
  }
};
```

**Do not** remove authentication, disable lease expiry, weaken signature checks, change 403/503 to 200, or expose signing keys to work around this bug.

## Why ordinary tests missed it

Unit tests were calling the internal function directly:

```js
await bridgeFetch(request, mockEnv, fixedTimestamp);
```

That was valid, so tests passed. The missing test invoked the **actual default export** using Cloudflare's three-argument contract:

```js
import assert from "node:assert/strict";
import worker from "./worker.mjs";

const ctx = { waitUntil() {}, passThroughOnException() {} };
// Build a signed GET with a synthetic key, current timestamp,
// and local/mock R2 binding; never use a production secret.
const response = await worker.fetch(signedRequest, testEnv, ctx);
assert.equal(response.status, 200);
```

Also test that invalid requests fail closed, a disabled Worker still returns 503, and valid requests are rejected if the HMAC is wrong. Avoid R2 object operations in readiness tests.

## How we isolated it

1. Confirmed the **control plane** had the intended bindings and temporarily enabled lease. This alone did not establish which code handled a particular request.
2. Verified the script-content fingerprint matched across the short lease and rollback; the relevant diagnostic code was present.
3. Reproduced the **same signed 403 and unsigned 503** in local `workerd` using a synthetic key and simulated local R2 bucket.
4. Checked a **local-only** instrumentation wrapper: diagnostic key `FORMAT_VALID`, HMAC `MATCH`, R2 binding `READY`, GET body `NULL` and expected request headers `PRESENT`.
5. Compared Cloudflare's handler arguments with our internal function: the third argument was the mismatch.
6. Wrapped the handler to pass `Date.now()` explicitly. Local `workerd` then returned signed **200**, enabled invalid-route **403**, and disabled-state **503**. The offline regression and broader PR checks passed.

### Two separate local-testing gotchas

- Pinned Miniflare versions may change constructor schemas. Prefer supported Wrangler local commands and test APIs over copying old constructor examples.
- The pinned local `workerd` binary in this case supported compatibility dates only through **2026-10-19**, while the checked-in Worker targeted **2026-11-04**. An override for **local testing only** allowed reproduction; **the production compatibility date was not changed**. This limits exact hosted-runtime parity.

## Five-minute checklist

- Search for `export default { fetch: internalFunction }` or equivalent direct forwarding.
- Compare the full signatures: `fetch(request, env, ctx)` versus `internalFunction(request, env, now)`.
- Inspect `Number.isFinite(now)`, lease-expiration tests, and signed-request timestamp checks.
- Exercise the default export with a mock `ctx`, not just internal functions with a numeric timestamp.
- Run the same tests under local `workerd` with synthetic secrets and a simulated R2 binding.
- Keep fail-closed behavior, rollback, and independent disabled-state checks intact.

## Verification boundaries

**Established:** The third-argument mismatch reproduced the specific application-generated 403/503 failures; the minimal adapter corrected them under local `workerd`; the offline test suite passed.

**Not established:** That the fix succeeded in a fresh, hosted Cloudflare deployment when this guide was written. It would be incorrect to claim production verification, restored backup transfers, or a general Cloudflare outage fix.

## Official references

- [Cloudflare: Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)
- [Cloudflare: Context (`ctx`) API](https://developers.cloudflare.com/workers/runtime-apis/context/)
- [Cloudflare: Local development](https://developers.cloudflare.com/workers/local-development/)
- [Cloudflare: Testing Workers](https://developers.cloudflare.com/workers/testing/)

---

**Reusing this note:** Copy or adapt the explanation and examples to help other developers. Please avoid reposting private Worker domains, credentials, archive paths or raw diagnostic logs.
