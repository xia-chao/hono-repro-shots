# `createPool()` leaks the slot and never settles the caller's promise when a task rejects

Issue: [honojs/hono#TBD](https://github.com/honojs/hono/issues/TBD)

Affected: `honojs/hono` `4.13.8` (`main`, `098e1191`), `bun` 1.4.3 on macOS.

`hono/utils/concurrent` is reachable through the `./utils/*` wildcard in `package.json`
(`"./utils/*" → ./dist/utils/*.js`), so `createPool()` is part of the public surface. Its
`run()` has two holes on the rejection path:

```ts
// src/utils/concurrent.ts:29
const run = async <T>(
  fn: () => T,
  promise?: Promise<T>,
  resolve?: (result: T) => void
): Promise<T> => {
  if (pool.size >= (concurrency as number)) {
    promise ||= new Promise<T>((r) => (resolve = r))  // ← only `resolve` is captured
    setTimeout(() => run(fn, promise, resolve))       // ← floating promise, never awaited
    return promise
  }
  const marker = {}
  pool.add(marker)
  const result = await fn()                           // ← a rejection jumps out here
  if (interval) {
    setTimeout(() => pool.delete(marker), interval)
  } else {
    pool.delete(marker)                               // ← ...so this never runs
  }
  if (resolve) {
    resolve(result)                                   // ← the only way `promise` settles
    return promise as Promise<T>
  } else {
    return result
  }
}
```

1. **The slot is never released.** There is no `try/finally` around `await fn()`, so a
   rejection skips `pool.delete(marker)`. The marker stays in the `Set` forever, and once
   `concurrency` tasks have rejected, `pool.size` is pinned at the limit — every later task
   is parked forever.
2. **The caller's promise never settles.** A parked task is handed the local `promise`,
   which can only ever be fulfilled by `resolve(result)`. The rejection path has no `reject`,
   so that promise stays pending: `await` hangs and `Promise.all()` never returns.

The minimal repro is two calls on a pool of one slot — the first throws, the second is a
plain function that never throws:

```ts
import { createPool } from 'hono/utils/concurrent'

const pool = createPool({ concurrency: 1 })

pool.run(async () => {
  throw new Error('task-1-boom')
}).catch((e) => console.log('task-1 rejected:', e.message))

// Never throws, and never runs: the slot from task 1 was never given back.
const probe = pool.run(() => 'PROBE-RAN')
probe.then((v) => console.log('task-2 resolved:', v))
```

```
[repro] task-1 -> rejected: task-1-boom
[repro] ---- waited 1500ms ----
[repro] task-2 (probe) : pending
[repro] RESULT         : HANG — the pool never runs another task
```

## Screenshots

**1. `01-breakpoint-empty-pool.webp`**

`src/utils/concurrent.ts:34` — task 1 entering `run()`, stopped inside `createPool`'s `run`,
with `module code` at `repro.mjs:22` (the first `pool.run` call).

```
pool.size         = 0
concurrency       = 1
[...pool].length  = 0
```

The pool is empty, so this task takes the normal branch (not the parked one).

**2. `02-breakpoint-marker-added.webp`**

`src/utils/concurrent.ts:41`

```ts
const result = await fn()
```

```
pool.size  = 1
```

`pool.add(marker)` has run, so the slot is now occupied. `Locals` shows `marker = {}` and
`result = undefined` — `fn` has not returned yet. The next step throws, and control leaves
the function body at this line, skipping the `pool.delete(marker)` at line 45.

**3. `03-breakpoint-slot-not-released.webp`**

`src/utils/concurrent.ts:34` again — but this is the **second** task. The call stack shows
`module code` at `repro.mjs:29` (the second `pool.run` call), and `Locals: run` shows
`fn = () => "PROBE-RAN"`.

```
pool.size         = 1
concurrency       = 1
[...pool].length  = 1
```

Task 1 has rejected, and `pool.size` is **still 1** — the slot never came back. So
`1 >= concurrency` holds and task 2 is dropped into the `setTimeout(() => run(...))` queue
instead of running.

**4. `04-debug-console-task-never-settles.webp`**

```
Debugger attached.
[repro] createPool({ concurrency: 1 })
[repro] task-1 -> rejected: task-1-boom
[repro] ---- waited 1500ms ----
[repro] task-2 (probe) : pending
[repro] RESULT         : HANG — the pool never runs another task
```

`task-1 -> rejected` is the trigger. `task-2 (probe)` is a function that cannot throw, yet
1.5 s later its promise is still `pending` — neither resolved nor rejected. The pool never
accepts another task again.

## Who hits this

`hono/ssg` builds a pool and wraps every static route in `pool.run`:

```ts
// src/helper/ssg/ssg.ts:243 and :265
await pool.run(() => app.fetch(forGetInfoURLRequest, { [SSG_CONTEXT]: true }))
let response = await pool.run(() => app.request(replacedUrlParam, requestInit, { ... }))
```

`toSSG` collects those into `Promise.all(getInfoPromises)` (`ssg.ts:452`), so a single
rejected `app.request`/`app.fetch` poisons the pool and the whole build hangs — no error, no
output, only a CI timeout.

For that to happen the `app.request` promise has to actually reject, which the default
`onError` prevents for `Error` instances. It does reject when a route or middleware throws a
**non-`Error`** value, because both `hono-base.ts:400` (`#handleError`) and `compose.ts:54`
only fall back to `onError` for `err instanceof Error` and otherwise re-`throw`; it also
rejects when a user supplies an `onError` that rethrows. The `createPool` defect above is
independent of all that — any `fn` that rejects triggers it.

## Reproducing

```
bun repros/repro-r7-001-pool.mjs
```

The breakpoints are on the source file only (`src/utils/concurrent.ts:34`, `:41`, `:45`),
driven with `ai-debug` against the Bun DAP adapter.
