# hono-repro-shots

Standalone repro screenshots for Hono issues.

- [`formdata-rejected-cache`](formdata-rejected-cache) — [honojs/hono#5383](https://github.com/honojs/hono/issues/5383):
  a failed `c.req.formData()` leaves the request body unreadable for every other
  body method.
- [`lambda-xml-binary`](lambda-xml-binary) — [honojs/hono#5399](https://github.com/honojs/hono/issues/5399):
  a binary `+xml` response is decoded as text by the AWS Lambda adapter, corrupting
  the body that reaches the client.
- [`lambda-edge-xml-binary`](lambda-edge-xml-binary) — [honojs/hono#5411](https://github.com/honojs/hono/issues/5411):
  the same corruption in the Lambda@Edge adapter, whose own copy of `isContentTypeBinary`
  was never updated by [#4469](https://github.com/honojs/hono/pull/4469) — Office documents
  are affected too.
- [`concurrent-pool-slot-leak`](concurrent-pool-slot-leak) — [honojs/hono#5413](https://github.com/honojs/hono/issues/5413):
  `createPool()` never gives back the slot of a task that rejects, and the parked caller's
  promise never settles — `toSSG()` hangs on the first rejected route.
