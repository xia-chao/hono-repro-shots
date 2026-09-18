# A binary `+xml` response is corrupted by the Lambda@Edge adapter

Issue: [honojs/hono#5411](https://github.com/honojs/hono/issues/5411)

Affected: `honojs/hono` `4.13.8` (`main`, `098e1191`), `bun` 1.4.3 on macOS — the
`lambda-edge` adapter (Lambda@Edge, `origin-response`).

`isContentTypeBinary()` decides whether a response body is sent as base64 or as plain
text. The Lambda@Edge adapter has **its own copy** of that function, and it was never
updated when [#4469](https://github.com/honojs/hono/pull/4469) rewrote the same function
in the `aws-lambda` adapter. It still runs the pre-#4469 pattern:

```ts
// src/adapter/lambda-edge/handler.ts:214
export const isContentTypeBinary = (contentType: string): boolean => {
  return !/^(text\/(plain|html|css|javascript|csv).*|application\/(.*json|.*xml).*|image\/svg\+xml.*)$/.test(
    contentType
  )
}
```

The `application\/(.*json|.*xml)` alternative matches whenever `json` or `xml` appears
**anywhere** after `application/`, so `application/vnd.apple.installer+xml` is classified
as text. That media type is an Apple installer package — a `xar` archive whose header
starts with the bytes `xar!` (`78 61 72 21`), followed by binary integers and a zlib
stream. It is not XML, and decoding it as UTF-8 replaces every non-UTF-8 byte with
U+FFFD, so the body that reaches the client is corrupted.

```ts
const app = new Hono()

app.get('/Installer.pkg', (c) => {
  c.header('Content-Type', 'application/vnd.apple.installer+xml')
  return c.body(pkgBytes)
})

export const handler = handle(app)
```

## How far the two adapters have drifted apart

The same function exists twice, and only one of the two copies has been maintained:

| `content-type` | `aws-lambda` (`098e1191`) | `lambda-edge` (`098e1191`) |
| --- | --- | --- |
| `application/vnd.apple.installer+xml` | text — same bug ([#5403](https://github.com/honojs/hono/issues/5403)) | text — **corrupts the body** |
| `application/vnd.mozilla.xul+xml` | text — same bug | text — **corrupts the body** |
| `…wordprocessingml.document` (Office) | binary — fixed by [#4469](https://github.com/honojs/hono/pull/4469) | text — **corrupts the body** |
| `…spreadsheetml.sheet` (Office) | binary — fixed by #4469 | text — **corrupts the body** |
| `…presentationml.presentation` (Office) | binary — fixed by #4469 | text — **corrupts the body** |
| `text/xml` | text | binary — needlessly base64 encoded |

Office documents are the clearest signal: `#4469` fixed them for `aws-lambda`, and they
are still broken on CloudFront because this second copy was never touched. `#4469` only
changed `src/adapter/aws-lambda/handler.ts` and its test — the sibling file was left
behind.

## Screenshots

**1. `01-breakpoint-content-type-check.webp`**

`src/adapter/lambda-edge/handler.ts:215` — stopped inside `isContentTypeBinary`, with
`createResult` next on the call stack.

```
contentType                       = "application/vnd.apple.installer+xml"
contentType.endsWith('+xml')      = true
isContentTypeBinary(contentType)  = false
```

The media type ends in `+xml` and is a `xar` archive, yet the function returns `false`
("not binary").

**2. `02-breakpoint-encoding-decision.webp`**

`src/adapter/lambda-edge/handler.ts:154`

```ts
const body = isBase64Encoded ? encodeBase64(await res.arrayBuffer()) : await res.text()
```

```
isBase64Encoded                   = false
contentEncoding                   = null
res.headers.get('content-type')   = "application/vnd.apple.installer+xml"
```

`isBase64Encoded` is `false`, so the `: await res.text()` branch is taken.

**3. `03-breakpoint-body-read-as-text.webp`**

`src/adapter/lambda-edge/handler.ts:156` — after `res.text()` has run.

```
body.length                                        = 17
body.includes('\uFFFD')                            = true
[...body].map(c => c.charCodeAt(0).toString(16).padStart(2, '0')).join(' ')
    = "78 61 72 21 1a 07 01 00 fffd fffd fffd fffd fffd fffd 28 fffd fffd"
```

The payload sent by the app was `78 61 72 21 1a 07 01 00 80 81 82 ff fe c3 28 a0 a1`. The
eight non-UTF-8 bytes are already `fffd` in the local variable.

**4. `04-debug-console-bytes-corrupted.webp`**

```
Debugger attached.
[repro] content-type  : application/vnd.apple.installer+xml
[repro] origin payload: 17 bytes 786172211a070100808182fffec328a0a1
[repro] bodyEncoding  : text(default)
[repro] delivered     : 33 bytes 786172211a070100efbfbdefbfbdefbfbdefbfbdefbfbdefbfbd28efbfbdefbfbd
[repro] lossless      : false
[repro] U+FFFD in body: true
[repro] RESULT: CORRUPTED (non-UTF8 bytes replaced by U+FFFD)
Debugger detached.
```

`bodyEncoding` is not `base64`, and the 17 bytes the app sent arrive as 33 bytes. Each
invalid byte became U+FFFD (`ef bf bd`), so the package cannot be opened.

## Reproducing

`repros/repro-r7-002-lambda-edge.mjs` builds a 17-byte `xar` header, passes it through
`handle(app)` with a minimal CloudFront `origin-response` event, and compares the returned
body with the original bytes.

```
bun repros/repro-r7-002-lambda-edge.mjs
```

The breakpoints are set on the source files only
(`src/adapter/lambda-edge/handler.ts:215`, `:154`), driven with `ai-debug` against the Bun
DAP adapter.

## Related

- [#5403](https://github.com/honojs/hono/issues/5403) — the same `+xml` corruption in the
  `aws-lambda` adapter.
- [#4468](https://github.com/honojs/hono/issues/4468) — the original report for Office
  files; [#4469](https://github.com/honojs/hono/pull/4469) fixed it, but only for
  `aws-lambda`.
- [#4250](https://github.com/honojs/hono/pull/4250) added `isContentTypeBinary` as an
  overridable hook.
