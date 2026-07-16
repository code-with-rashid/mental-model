# Example 1: Next.js ISR revalidation

*Verified against [vercel/next.js](https://github.com/vercel/next.js) @ `50925bb` (2026-07-15) — `packages/next/src/server/lib/incremental-cache/index.ts`, `packages/next/src/server/response-cache/index.ts`.*

**Prompt:**

> I'm using ISR with `revalidate: 60` on a product page. If nobody hits the
> page for an hour, does Next.js revalidate it in the background
> automatically, or does the first visitor after the hour get a stale page
> while it revalidates?

**Response:**

Short answer: both, depending on how stale it's gotten — and there's a
second, fully explicit path that doesn't wait on any timer at all.

Take your product page, cached with `revalidate: 60`. Call the visitor who
hits it at minute 61 "Priya."

When Priya's request comes in, the incremental cache checks the cached
entry's age against `revalidate` (`incremental-cache/index.ts:551`,
`:603-604`). It's stale, so two things happen essentially at once: Priya gets
the *existing* cached page immediately — no waiting — and, in the background,
Next.js kicks off a regeneration for that route (`response-cache/index.ts:356-368`,
which resolves the cached response to the requester before falling through to
`this.revalidate(...)`). The next visitor after that background job finishes
gets the fresh version. Priya never sees the update — she got the stale-but-fast
copy.

That's the *automatic* path: nobody triggered it, no code ran anything, it's
purely "was the timer up when a request happened to arrive."

There's a second case worth naming because it's stricter: if the entry is
past its `expire` window (not just `revalidate` — these are different knobs;
`expire` matters if you're using `cacheLife({ expire })`), the cache signals
`isStale: -1`, and the code explicitly skips serving the stale copy
(`response-cache/index.ts:354-362`, comment: *"we must NOT early-resolve with
the stale value — instead we fall through to a blocking revalidation"*). In
that case the visitor waits for the regenerate, same as an uncached request.

Separately — and this is the *explicit* action — you (or your code) can call
`revalidatePath()` / `revalidateTag()` from a server action or route handler.
That sets `isOnDemandRevalidate`, which bypasses the stale-serve entirely and
forces a blocking revalidation on the next matching request
(`response-cache/index.ts:359`). This is the one part of the flow that's a
deliberate action rather than something time triggers.

One thing flagged as *emerging*, not the stable default: the same file has a
newer code path gated behind an `appShells` flag for how prefetch requests
interact with an in-flight background revalidation
(`response-cache/index.ts:375-388`). If you're not intentionally on that
flag, ignore it — the description above is the path you're on.

**Confidence:** the `revalidate` vs `expire` staleness logic and the
stale-serve-then-background-revalidate behavior are verified directly from
the two files cited above. The behavior of `revalidatePath`/`revalidateTag`
itself (what sets `isOnDemandRevalidate` upstream) I traced to the flag being
read, not to the route handler call site — reasonably confident, not fully
walked end to end.

**This rests on:** you're using the default `revalidate: 60`, not a custom
cache handler that overrides this logic, and you're not on the `appShells`
flag. Tell me if any of those don't hold.
