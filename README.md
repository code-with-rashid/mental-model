# mental-model

> A Claude Code skill that stops "explain how this works" answers from quietly mixing guesses with facts — it grounds every claim in your actual code, keeps "how it works today" separate from "how it's supposed to work," and tells you which parts it's not sure about.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## The problem

Ask an AI assistant "how does our webhook retry logic work?" and you'll often get an answer that sounds authoritative but silently blends three things: what the code actually does, what the ticket says it's *supposed* to do, and a plausible-sounding guess filling the gap. You walk away with a mental model that's wrong in a way you can't detect until it costs you — a bug you built on top of a misunderstood assumption.

## What this does

- Explains a concept, feature, ticket, or piece of implementation in plain language, so you leave with correct intuition — not a wall of text.
- Verifies every "how it works" claim against actual source (code, ticket, docs) before stating it, and labels anything unverified as an assumption.
- Anchors the explanation in one concrete, named example that gets reused as the explanation goes deeper.
- Explicitly separates "how it works today" from "what it's meant to become" — the single most common way a mental model goes wrong.
- Ends by listing the assumptions the explanation rests on and asks you to correct any that are off.
- Is read-only by design (enforced via `allowed-tools`, not just instructions) — it will not modify code, even if the explanation surfaces something that looks like a bug.

**What it explicitly does NOT do:**
- Fix bugs or debug failures — it explains, it doesn't change anything.
- Write specs, PRDs, or design proposals.
- Review a diff for correctness or security.
- Answer general knowledge questions with no source to check against.

## Install

Copy the skill into your project or personal skills directory:

```bash
# Project-level (shared with your team via git)
cp -r mental-model .claude/skills/

# Or personal (available across all your projects)
cp -r mental-model ~/.claude/skills/
```

Claude Code picks it up automatically — no restart needed for project-level skills in most setups; personal skills apply on the next session.

## Example

Tested against real, unmodified source from three widely-used open source
projects — not a synthetic demo:

- [Next.js: does ISR revalidate automatically, or does a request trigger it?](examples/example-1-nextjs-isr-revalidation.md) — verified against `vercel/next.js`
- [tRPC: does `httpBatchLink` batch every call, or do you opt in?](examples/example-2-trpc-automatic-batching.md) — verified against `trpc/trpc`
- [Kubernetes: does a Deployment wait for new pods healthy before killing old ones?](examples/example-3-kubernetes-rolling-update.md) — verified against `kubernetes/kubernetes`

Each one cites the exact file/line it's grounded in, names what's automatic
vs. what needs an explicit action, and flags what's verified vs. inferred.
See [`examples/example-0-response-shape-template.md`](examples/example-0-response-shape-template.md)
for the shape at a glance.

## Why I built this

I use this daily on my own TypeScript/Next.js/Expo work — mostly when picking up a ticket or an unfamiliar part of a codebase and needing to actually understand it before touching anything, not get a confident-sounding summary that turns out half-guessed.

## License

MIT
