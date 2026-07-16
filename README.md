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

### Recommended: as a plugin

Inside a Claude Code session, in any project:

```
/plugin marketplace add code-with-rashid/mental-model
/plugin install mental-model@mental-model
```

Then run `/reload-plugins` (or start a fresh `claude` session) — a plugin installed
mid-session doesn't retroactively load into that session's context, so the skill
won't trigger yet until you do. Confirm with `/plugin list`, or `claude plugin list`
from a regular terminal.

By default this installs at **user scope** (available in every project). To scope it
to just the current project instead, run the equivalent from a terminal:

```bash
claude plugin marketplace add code-with-rashid/mental-model --scope project
claude plugin install mental-model@mental-model --scope project
```

To remove it later: `/plugin uninstall mental-model@mental-model`.

<details>
<summary>Alternative: manual copy (no plugin system)</summary>

This is a folder with a `SKILL.md` — no build step, no dependencies. **Clone this
repo first**, then copy `skills/mental-model/` into a `.claude/skills/` directory.

Use this if you'd rather not use the plugin system, or want to read/vet the skill's
contents before trusting it in a project — a skill's instructions run with whatever
tool access you grant it, so reviewing it first is reasonable practice either way.

#### macOS / Linux (bash/zsh)

```bash
git clone https://github.com/code-with-rashid/mental-model.git ~/mental-model

# per-project (recommended)
mkdir -p /path/to/your-repo/.claude/skills
cp -r ~/mental-model/skills/mental-model /path/to/your-repo/.claude/skills/

# per-user (available in every project, instead of per-project)
mkdir -p ~/.claude/skills
cp -r ~/mental-model/skills/mental-model ~/.claude/skills/
```

#### Windows (PowerShell)

```powershell
git clone https://github.com/code-with-rashid/mental-model.git $HOME\mental-model

# per-project
New-Item -ItemType Directory -Force -Path "C:\path\to\your-repo\.claude\skills" | Out-Null
Copy-Item -Recurse -Force "$HOME\mental-model\skills\mental-model" "C:\path\to\your-repo\.claude\skills\"

# per-user
New-Item -ItemType Directory -Force -Path "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse -Force "$HOME\mental-model\skills\mental-model" "$HOME\.claude\skills\"
```

Only the copied `skills/mental-model` folder matters; the rest of the clone (this
README, the examples, the license) can be deleted afterward, or kept around so
`git pull` can fetch future updates before you re-copy it.

#### If you install it both ways

Don't — pick one. If a project ends up with both the plugin installed *and* a
manual copy under its own `.claude/skills/mental-model/`, the project-level copy
wins for the bare `/mental-model` command (project skills take priority over
plugin skills with the same name). The plugin is still reachable explicitly at
`/mental-model:mental-model`, but having both around is confusing for no benefit.
A project-scope `.claude/skills/` copy also only loads after you've accepted
Claude Code's workspace-trust dialog for that folder — `claude plugin list` will
call this out explicitly if it finds one that hasn't loaded yet.

</details>

### If natural-language phrases don't trigger it

Skill auto-invocation is Claude deciding your request matches a skill's
description — it's a judgment call, not a guarantee, and it competes with every
other skill you have installed. If "give me context to understand X" doesn't fire
it: confirm you reloaded after installing (see above), and if you have a lot of
other skills/plugins installed, run `/doctor` to check whether `mental-model`'s
description is being truncated from the context budget. Either way, typing
`/mental-model` explicitly always works regardless of auto-detection.

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
