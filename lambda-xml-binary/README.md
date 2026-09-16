# A binary `+xml` response is corrupted by the AWS Lambda adapter

Issue: [honojs/hono#5399](https://github.com/honojs/hono/issues/5399)

Affected: `honojs/hono` `4.13.8` (`main`, `098e1191`), `bun` 1.4.3 on macOS.

`defaultIsContentTypeBinary()` decides whether a response body is sent as base64 or as
plain text. It treats any media type that contains `/xml` or `+xml` at the end as text,
so a response declared as `application/vnd.apple.installer+xml` is decoded with
`Response.text()`. That content type describes an Apple installer package, which is a
`xar` archive — the header starts with the bytes `xar!` (`78 61 72 21`) followed by
binary integers and a zlib stream. Decoding those bytes as UTF-8 replaces every
non-UTF-8 byte with U+FFFD, so the body that reaches the client is corrupted.

```ts
const app = new Hono()

app.get('/app.pkg', (c) =>
  c.body(pkgBytes, 200, { 'content-type': 'application/vnd.apple.installer+xml' })
)

export const handler = handle(app)
```

## Screenshots

**1. `01-breakpoint-content-type-check.webp`**

`src/adapter/aws-lambda/handler.ts:671`

```ts
return !/^text\/(?:plain|html|css|javascript|csv)|(?:\/|\+)(?:json|xml)\s*(?:;|$)/.test(
  contentType
)
```

Stopped inside `defaultIsContentTypeBinary`, with `createResult` next on the call stack.

```
contentType                                = "application/vnd.apple.installer+xml"
contentType.endsWith('+xml')               = true
/(?:\/|\+)(?:json|xml)\s*(?:;|$)/.test(…)  = true
```

The second alternative of the pattern matches, so the function returns `false` and the
media type is classified as text.

**2. `02-breakpoint-encoding-decision.webp`**

`src/adapter/aws-lambda/handler.ts:356`

```ts
let isBase64Encoded = contentType && isContentTypeBinary(contentType) ? true : false
```

The return value from the previous frame is used directly.

```
contentType                       = "application/vnd.apple.installer+xml"
isContentTypeBinary(contentType)  = false
options                           = { isContentTypeBinary: undefined }
```

`options.isContentTypeBinary` is `undefined`, so the default function is what produced
the `false` — no user-supplied override is involved.

**3. `03-breakpoint-reads-body-as-text.webp`**

`src/adapter/aws-lambda/handler.ts:363`

```ts
const body = isBase64Encoded ? encodeBase64(await res.arrayBuffer()) : await res.text()
```

```
isBase64Encoded                           = false
res.headers.get('content-type')           = "application/vnd.apple.installer+xml"
isBase64Encoded ? 'base64' : 'res.text()' = "res.text()"
```

The `false` branch is taken, so the body is read through `Response.text()` and decoded
as UTF-8.

**4. `04-debug-console-bytes-corrupted.webp`**

The adapter's return value compared with the bytes the app sent:

```
--- AWS Lambda adapter: binary response through Hono ---
content-type      : application/vnd.apple.installer+xml
isBase64Encoded   : false
bytes sent by app : 78617221001c0001000000000000006400000000000000c800000001fffe8081 (32 bytes)
bytes in response : 78617221001c000100000000000000efbfbd00000001efbfbdefbfbdefbfbdefbfbd (42 bytes)
RESULT            : response body is corrupted
```

The last four bytes of the payload (`ff fe 80 81`) are not valid UTF-8. Each invalid
byte becomes U+FFFD, encoded as `ef bf bd`, so 32 bytes turn into 42 and the downloaded
file cannot be opened. Nothing is logged and no error is raised.

## Reproducing

`probes/round6/debug-src.ts` in the report workspace builds a 20-byte `xar` header,
passes it through `handle(app)` with a minimal `APIGatewayProxyEventV2`, and compares
the returned body with the original bytes.

```
bun run probes/round6/debug-src.ts
```

## Related

- [#4468](https://github.com/honojs/hono/issues/4468) — the same failure for Office files
  (`xml` appears in the media type, so the body was classified as text and the download
  was broken). [#4469](https://github.com/honojs/hono/pull/4469) fixed that case by
  rewriting this pattern.
- [#4250](https://github.com/honojs/hono/pull/4250) added `isContentTypeBinary` so a user
  can override the classification.
