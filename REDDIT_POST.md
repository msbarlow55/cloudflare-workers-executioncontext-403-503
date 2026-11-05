# Reddit draft — r/CloudFlare

**Suggested title:** PSA: Cloudflare Worker returned HMAC 403 + readiness 503 despite correct R2 bindings — ExecutionContext was passed as a timestamp

**Body:**

Sharing a debugging trap that cost us hours in case it saves someone else the same effort.

We had a Cloudflare Worker with a temporary test lease, an R2 binding, and an HMAC-signed diagnostic GET. Deployment metadata showed the correct bindings and signing key, and the HMAC was correct. But:

- Valid signed diagnostic GET → **403 BLOCKED**
- Unsigned readiness/invalid-route GET during enabled test lease → **503 BLOCKED**
- Permanently disabled Worker → **503 BLOCKED** (expected)

We investigated propagation, Worker versions, R2 binding injection, bodyless GET normalization, and signing. Local `workerd` reproduced the same 403/503 responses. A local-only inspector independently confirmed the R2 binding was ready, the HMAC matched, the signing key was present, and the GET had no body.

**The actual bug was the fetch handler's third argument.**

Cloudflare calls a module Worker as `fetch(request, env, ctx)`, where `ctx` is an **ExecutionContext**. Our internal function expected a numeric timestamp as its third argument:

```js
async function bridgeFetch(request, env, now = Date.now()) {
  // Lease expiry + signed timestamp checks use now.
}

// BUG: Cloudflare's ctx object is forwarded as now.
export default { fetch: bridgeFetch };
```

Because Cloudflare *supplied* the third argument, the `Date.now()` default never ran. `Number.isFinite(now)` failed. The readiness path failed closed with 503; the signed diagnostic failed timestamp validation with 403.

**Fix: explicitly adapt the platform handler.**

```js
export default {
  fetch(request, env) {
    return bridgeFetch(request, env, Date.now());
  }
};
```

We added a regression that invokes the **default exported handler** with a simulated `ctx` object (instead of only testing the internal function with a numeric clock). After the fix, local `workerd` returned **200** for a valid signed diagnostic, **403** for an intentionally invalid route in enabled test mode, and **503** for disabled mode. All offline PR checks passed.

**Important caveat:** We had not run a new hosted deployment verification when writing this. The bug was reproduced and corrected **offline**; I'm not claiming the hosted fix is production-verified. These codes were **our Worker's own fail-closed responses**, not a universal explanation for every Cloudflare 403 or 503.

**Takeaway:** If a Worker behaves as if its lease is expired or a valid signature is wrong even though the environment is correct, inspect `export default { fetch: someFunction }` and compare that function's third argument with `ctx`.

Full troubleshooting guide and regression test pattern:
https://github.com/msbarlow55/cloudflare-workers-executioncontext-403-503

Official docs:
https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/
https://developers.cloudflare.com/workers/runtime-apis/context/

Hope this saves someone else a long debugging session.

---

*This file is a draft to copy to Reddit. It is not a claim that a Reddit post has been published.*
