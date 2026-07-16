# Response shape (illustrative template, not a real trace)

The three other files in this folder (`example-1`, `example-2`, `example-3`)
are real transcripts, verified against the actual source of Next.js, tRPC,
and Kubernetes. This one is a synthetic, invented scenario kept only to show
the shape a response should take at a glance, before you go read a full
verified transcript.

**User:** "Give me context to understand how our webhook retries work — is
that automatic?"

**Response shape:**
1. One-paragraph plain-language summary, grounded in cited source (`file:line`).
2. A concrete running example, reused as the explanation goes deeper.
3. An explicit call-out of what's automatic vs. what requires a deliberate action.
4. Confidence labels — what's verified from source vs. inferred vs. uncertain.
5. A closing list of the assumptions the explanation rests on, inviting correction.

For the real thing, start with `example-1-nextjs-isr-revalidation.md`.
