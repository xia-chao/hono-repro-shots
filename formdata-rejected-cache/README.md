# A failed `c.req.formData()` leaves the whole request body unreadable

Issue: [honojs/hono#5383](https://github.com/honojs/hono/issues/5383)

Affected: `honojs/hono` `4.13.7` (`main`, `edd138ee`) — reproduced on Node.js 26
(`@hono/node-server`), Bun 1.4.3 and Deno 2.9.6.

`c.req.formData()` consumes the request stream and caches only its result. When the
media type is not a form, the cached entry is a **rejected promise**, and every later
representation of the same body (`text()`, `json()`, `arrayBuffer()`, `bytes()`,
`blob()`) is handed that same rejection.

The shape that surfaces it is trying one parser and falling back to another:

```ts
let body
try {
  body = await c.req.formData() // JSON request, so this throws
} catch {
  body = await c.req.json() // throws as well
}
```

## Screenshots

**1. `01-breakpoint-first-parse-caches-promise.png`**

`src/request.ts:243` — `return (bodyCache[key] = raw[key]())`

The first call, `c.req.formData()` on a `Content-Type: application/json` request.
`bodyCache` is still empty and the line stores `raw.formData()`'s promise in it
unconditionally, even though this parse is about to reject.

Watch: `formData requested | cached: []`

**2. `02-breakpoint-second-parse-reuses-rejected-promise.webp`**

`src/request.ts:226` — `return (bodyCache[anyCachedKey as keyof Body] as Promise<BodyInit>).then(...)`

The second call comes from the `catch` fallback: `c.req.json()` asks for `text`. The
direct cache lookup misses, so it drops into the fallback loop — and the only entry
there is the already-rejected `formData` promise.

```
key          = "text"        // what was asked for
anyCachedKey = "formData"    // what is in the cache
bodyCache    = { formData: Promise }

bodyCache.formData
  = Promise { result: TypeError [ERR_FORMDATA_PARSE_ERROR]:
              Can't decode form data from body because of incorrect MIME type/boundary,
              status: "rejected" }
```

Call stack: `#cachedBody:226 ← json:259 ← <anonymous> debug-src.ts:27`

**3. `03-debug-console-json-rejected.webp`**

The fallback rejects with the first parser's error, so the request ends as a 500.

```
Debugger attached.
FORM_DATA_REJECTED: TypeError
JSON_REJECTED: TypeError | Can't decode form data from body because of incorrect MIME type/boundary
STATUS: 500
RESULT: {"ok":false,"error":"Can't decode form data from body because of incorrect MIME type/boundary"}
Debugger detached.
```

## Why `parseBody()` does not have this problem

`c.req.parseBody()` reads the bytes first and parses them afterwards, so the raw body
survives a failed parse (`src/utils/body.ts`, `parseFormData()`). `c.req.formData()`
goes straight to `raw.formData()`, which consumes the stream and keeps only the result.

Related: #4806 / #4807 fixed the same class of problem for `parseBody()`.
