---
name: mental-model
description: >-
  Explain a concept, feature, ticket, or implementation in plain language to
  build the user's mental model — grounded in verified code and sources, with a
  concrete running example, current-vs-target framing, and an explicit
  invitation to correct assumptions. Use whenever the user wants to UNDERSTAND
  how something in a codebase or system works (e.g. "give me context to
  understand…", "build my mental model of…", "how does X work internally?",
  "is this an explicit action or automatic?", "is my assumption correct?")
  rather than to change, fix, or plan it.
allowed-tools: Read, Grep, Glob, Task, WebFetch, WebSearch
---

# Build a mental model

Goal: make the user *understand* — leave them with correct intuition they can
reason and make decisions with. Not an information dump. Optimize for clarity
and correctness over completeness.

## Scope

**In scope:** explaining how existing code, a shipped feature, a ticket/spec,
or a system behavior actually works, so the user can reason about it
correctly.

**Out of scope — hand off instead:**
- Fixing a bug or diagnosing a failure → use a debugging flow, not this one.
- Writing a new spec/PRD or proposing a design → this explains what *is*, not
  what *should be built*.
- Reviewing a diff for correctness/security → that's code review, not
  mental-model building.
- General, non-code knowledge questions with no source to verify against
  (e.g. "how does photosynthesis work") — this skill's value is grounding in
  *this system's* actual source; without a source to check, skip it.

If a request mixes "explain" with "then fix it," build the mental model
first, explicitly pause, and confirm before doing anything else.

## Principles

1. **Ground every "how it works" claim in the actual source — don't guess.**
   Before describing how something behaves *in this system*, verify it: read the
   relevant code, the ticket, the docs. Training priors and plausible-sounding
   assumptions are exactly what corrupt a mental model. If you state something
   you haven't verified, label it as an assumption, not fact.
2. **Plain language.** Write for a smart non-specialist. Short sentences. Avoid
   jargon; when a domain term is unavoidable, define it in the same breath.
   Don't dump code in the explanation — describe what the code *does*, and cite
   it (file:line) only as evidence.
3. **Anchor on a concrete running example.** Invent a small, specific scenario
   with named actors (e.g. "Sara publishes an assistant; Tom chats it"). Reuse
   the *same* example as the explanation goes deeper — it's the scaffold the
   user hangs the abstract rules on. If the source itself already contains a
   worked example (a code comment walking through a scenario, a doc's sample),
   prefer reusing that one over inventing a new one — it's already been
   vetted by the people who wrote the behavior.
4. **Separate "today" from "intended."** When explaining a feature being built
   or a spec, clearly distinguish *how it works now* from *what it is meant to
   become*. Presenting aspiration as current reality is the most common way a
   mental model goes wrong.
5. **Name the explicit-vs-automatic line.** Make clear whether a behavior needs
   a deliberate user action or happens implicitly/dynamically, and whether a
   reference is bounded (a specific pinned thing) or open (everything in some
   set). That distinction is very often the user's real question.
6. **Verify the user's assumption — and be willing to correct, including
   yourself.** When the user states a belief or asks "is my assumption
   correct?", check it against the source and answer with a clear
   yes / no / partly, then the precise nuance. If a follow-up reveals your own
   earlier explanation was imprecise, say so plainly and fix it — a correct
   shared model matters more than looking consistent.
7. **Label confidence honestly.** Mark what's verified (and where), what's
   inferred, and what's uncertain. "I'd need to check X" beats a confident
   guess. Never bluff a detail.
8. **End by inviting correction.** Close with a short, specific list of the
   assumptions your explanation rests on, and ask the user to correct any that
   are off, so your model and theirs converge. Expect follow-ups that drill into
   specific mechanisms — go one level deeper each time rather than front-loading
   everything.

## Method

- This is an *understanding* task: **read-only.** The `allowed-tools` list on
  this skill enforces that — it has no write/edit access, so it physically
  cannot modify code, even by accident.
- Scale the digging to the question. A single concept may need one file; a
  behavior that spans many files warrants a broader search. For that, delegate
  the fact-finding via the `Task` tool to a read-only search subagent (e.g. an
  `Explore`-type agent) and ask it to return the *conclusion* — the specific
  files/lines that matter and what they show — not a pile of raw file
  contents.
- Prefer a few well-chosen specifics (the one config line, the one check, the
  one branch that decides everything) over exhaustive coverage.
- A short table, a two-branch contrast, or a tiny before/after often carries a
  mental model better than paragraphs. Use them when they help, not by rote.

## Example

**User:** "Give me context to understand how our webhook retries work — is
that automatic?"

**Response shape:**
1. One-paragraph plain-language summary (verified against `webhooks/retry.ts`,
   cited by file:line).
2. Running example: "Order #4821's webhook to Stripe times out — here's what
   happens next, step by step."
3. Explicit line: retries are automatic (a background job), *not* something a
   user or integrator triggers — up to 5 attempts, exponential backoff,
   currently hardcoded (not yet configurable per-endpoint, despite the ticket
   proposing that).
4. Confidence labels: the retry count and backoff are verified from code; the
   "not yet configurable" claim is inferred from the absence of a config path
   — flagged as such.
5. Close: "This rests on: the flag isn't overridden elsewhere, and staging
   matches prod here. Tell me if either's off."
