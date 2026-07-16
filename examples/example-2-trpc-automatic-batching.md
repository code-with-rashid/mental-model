# Example 2: tRPC httpBatchLink batching

*Verified against [trpc/trpc](https://github.com/trpc/trpc) @ `340811b` (2026-06-20) — `packages/client/src/links/httpBatchLink.ts`, `packages/client/src/internals/dataLoader.ts`.*

**Prompt:**

> Does `httpBatchLink` batch every query call into one request automatically,
> or do I need to opt in per call?

**Response:**

Automatic, with one sharp edge: batching is a *timing* window, not a
per-call setting — so which calls end up together depends on exactly when
they fire.

Picture a dashboard component that on mount calls `trpc.user.get.useQuery()`,
`trpc.orders.list.useQuery()`, and `trpc.notifications.count.useQuery()` — all
three synchronously, in the same render.

Each call goes through `dataLoader.load()` (`internals/dataLoader.ts:126-141`),
which pushes the request into a shared `pendingItems` array and arms a timer:
`dispatchTimer ??= setTimeout(dispatch)` (`:140`). That's a zero-delay
`setTimeout` — so it fires on the next tick, after the current synchronous
block finishes. All three of your dashboard's calls land in `pendingItems`
before that tick runs, so they all get swept into one `dispatch()` call
(`:90-134`), which groups them and sends a single batched HTTP request via
`batchLoader.fetch` (`httpBatchLink.ts:57-84`) — one network round trip for
all three.

The edge: this is a *tick* boundary, not a "component" or "request" boundary.
If one of those three queries were instead kicked off from inside a
`.then()` or an `await`ed call — i.e., after a tick has already passed — it
misses the first batch's timer and starts (or joins) a new one. There's no
per-call flag to force or prevent this; it's purely "were these `load()`
calls made before the pending timer fired."

Two more real constraints worth knowing before you assume everything always
batches together: `maxItems` and `maxURLLength` (both configurable on
`httpBatchLink`, default `Infinity`) can force a single tick's calls into
*multiple* HTTP requests if the batch would otherwise be too big
(`httpBatchLink.ts:32-49`, `groupItems` in `dataLoader.ts:44-79`). And per the
code comment at `dataLoader.ts:29-30`, cancelling one query in a batch only
cancels the network request if *every* item in that batch gets cancelled —
cancelling one of three still lets the batch (and the other two) go through.

**Confidence:** the batching window (`setTimeout(dispatch)`, same-tick
grouping) and the `maxItems`/`maxURLLength` splitting behavior are verified
directly from the two files cited. The cancellation-requires-all behavior is
from the source code comment, not from tracing the abort-signal plumbing
line by line — high confidence, not exhaustively verified.

**This rests on:** you're using `httpBatchLink` specifically (not
`httpLink`, which doesn't batch at all, or `httpBatchStreamLink`, which
streams responses back individually and may differ). Correct me if you're on
a different link.
