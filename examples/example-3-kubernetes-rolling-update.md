# Example 3: Kubernetes Deployment rolling update

*Verified against [kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) @ `1b4e48f` (2026-07-15) — `pkg/controller/deployment/rolling.go`, `pkg/controller/deployment/util/deployment_util.go`.*

This one stress-tests the skill's "delegate broad searches" path — the
answer isn't in one file, it's stitched from three functions across two
files in a controller most people never read.

**Prompt:**

> When I bump the image tag on a Deployment, does k8s wait for all new pods
> to be healthy before killing any old ones, or does it kill-and-create in
> lockstep?

**Response:**

Neither, exactly — it's bounded by two knobs (`maxSurge`, `maxUnavailable`),
and it's a repeating loop, not a single pass.

Say your Deployment has 10 replicas, default `maxSurge: 25%` and
`maxUnavailable: 25%`, and you bump the image tag.

Each reconcile of the controller (`rolling.go:30-62`) does two things in a
fixed order: first try to scale the *new* replica set up
(`reconcileNewReplicaSet`), and only if that made no change, try to scale the
*old* replica set(s) down (`reconcileOldReplicaSets`). It repeats this on
every sync — driven by the controller's watch/resync loop, not a single
atomic cutover — until the new replica set has fully replaced the old one.

The scale-up side is capped by `maxSurge`: total pods (old + new) can't
exceed `desired + maxSurge` (`deployment_util.go:817-836`). With your numbers,
`maxSurge` at 25% of 10 rounds up to 3 (`ResolveFenceposts` passes
`roundUp: true` for surge, `deployment_util.go:884`), so the controller can
create up to 3 new pods *before* removing any old ones — you briefly run 13
pods, not 10.

The scale-down side is capped by `maxUnavailable`, and here's the gotcha: it
rounds the *other* way. 25% of 10 rounds **down** to 2
(`roundUp: false` for unavailable, `:888`), and if both fenceposts round to
zero, `maxUnavailable` gets forced to 1 rather than 0 (`:893-898`, so a
rollout never fully stalls). The controller won't remove an old pod if doing
so would drop available capacity below `desired - maxUnavailable` — and it
also subtracts however many of the *new* pods aren't healthy yet
(`rolling.go:118-120`), specifically so a crash-looping new replica set can't
trick it into scaling down more old (working) pods than is safe.

So the real shape is: surge a few new pods in, wait for reconcile, retire as
many old pods as `maxUnavailable` (adjusted for new-pod health) allows,
repeat — not "all new healthy first" and not "1-for-1 lockstep."

**Confidence:** the scale-up/scale-down ordering, the surge/unavailable
formulas, and the rounding directions are verified directly from the three
functions cited, including the fencepost rounding, which is easy to get
backwards from memory (surge rounds up, unavailable rounds down — I checked
rather than assumed this, since it's a classic footgun). What I have *not*
traced is the exact resync interval or what specific watch events trigger a
reconcile — I know it's event/resync-driven, not that it's every N seconds
specifically.

**This rests on:** your Deployment uses the default `RollingUpdate` strategy
(not `Recreate`, which is a different code path entirely — kill-all-then-create-all),
and you haven't set `maxUnavailable: 0`, which removes the "scale down
unhealthy old pods early" allowance described above. Let me know if either
of those is off.
