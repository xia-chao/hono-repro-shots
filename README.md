# hono-repro-shots

Standalone repro screenshots for Hono issues.

- [`formdata-rejected-cache`](formdata-rejected-cache) — [honojs/hono#5383](https://github.com/honojs/hono/issues/5383):
  a failed `c.req.formData()` leaves the request body unreadable for every other
  body method.
- [`lambda-xml-binary`](lambda-xml-binary) — [honojs/hono#5399](https://github.com/honojs/hono/issues/5399):
  a binary `+xml` response is decoded as text by the AWS Lambda adapter, corrupting
  the body that reaches the client.
