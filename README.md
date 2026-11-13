# Flow-Check

[![GitHub stars](https://img.shields.io/github/stars/olakunlevpn/olakunlevpn-flow-check?style=flat-square)](https://github.com/olakunlevpn/olakunlevpn-flow-check/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)

One skill that answers a single question with evidence: **does this feature actually connect, all the way through, with nothing dangling?**

It maps the user interaction flow and the server flow as two exhaustive lists, joins them into a seam table, then proves every link with a file path and a line number. Breaks get root-caused before they get fixed. Nothing passes on inspection-by-eye.

This is an **agent skill** that works with Claude Code, Cursor, Cline, Gemini CLI and 40+ other AI coding agents.

## Standalone

Flow-check depends on no other skill. The verification lenses, the data contract audit, the root-cause gates, the Evidence Ledger and the reasoning discipline are all defined inside `SKILL.md`. Drop the folder anywhere and it runs at full strength.

## What It Cannot Do

Stated plainly, because a verification tool that oversells itself is worse than none:

- **It proves connection and contract, not business correctness.** A flow that is perfectly wired and computes the wrong tax passes every link check. The Business Analyst lens reads the rules, but reading is not testing.
- **It is bounded by the scope you give it.** Contamination outside the named flow is not found.
- **It is a prompt, not CI.** It has no executable enforcement of its own rules; it raises the floor on a careful agent, it does not replace a pipeline.
- **No browser verification.** UI behavior is verified by reading Map A and by the contract audit, not by driving a real browser.

## The Problem

Most broken features are not broken in the backend or the frontend. They are broken at the seam:

- A button bound to an empty handler
- A route registered but never called by anything
- A handler that validates, returns 200, and never calls the service that does the work
- A response emitting `first_name` to a component reading `firstName`
- A field assigned on a model but absent from the migration
- A paginated `{data: []}` consumed as a bare array
- An error path nobody wired, so failure renders as a blank screen
- A job dispatched to a queue with no worker

Both halves look fine in isolation. Every file reviews clean. The flow is dead.

An agent asked "is this wired?" will typically read the controller, read the component, see plausible code in both, and say yes. It never wrote down the list of connections, so it never noticed the one that was missing. You cannot see the absence of something you never enumerated.

## What It Does

**Phase 0 — Scope.** Pin the feature's actors, entry points, exit points, source of truth, and what is explicitly out of scope.

**Phase 1 — Map, exhaustively, before verifying anything.** Three numbered lists: the user interaction flow (every step, element, trigger and state including loading, empty, error and forbidden), the server flow (route → middleware → auth → validation → controller → service → model → DB → side effects → response), and the seam — one row per client↔server interaction, which is where flows actually die.

**Phase 2 — Prove every link.** Thirteen mandatory checks per link, plus a field-by-field data contract audit. Each link gets exactly one status: `WIRED` with `file:line`, `BROKEN`, or `UNPROVEN`. `UNPROVEN` is a real status that blocks a clean pass — "I did not check it" never becomes "it is fine."

Phase 2 also runs two gates that must be *executed*, with raw output pasted into the run:

- **The Contamination Scan** — six grep categories over the flow path: unfinished work (`TODO`/`FIXME`/`HACK`), debug leftovers (`console.log`, `dd(`, `var_dump`, `debugger`), fake data (`lorem`, `placeholder`, `mock`, `test@`, `example.com`), stubbed returns, hardcoded values standing where computed ones belong, and disabled safety (`@ts-ignore`, `eslint-disable`, `.skip(`). Every hit is triaged individually as contamination or legitimate — never dismissed by category. **No pasted output means the line reports `UNPROVEN`, never "none".**
- **The Execution Gate** — typecheck, build, test suite, a live endpoint hit with the real response read, a database check confirming the write actually landed, and the async legs consumed by a real worker. A missing tool is a stated fact; an unrunnable one is `UNPROVEN`. A red result is `BROKEN` and goes to Phase 3. **A link proven by reading alone caps the whole run at 0.95** — `1.0` means the flow was watched working. (Browser and UI driving are deliberately out of scope; the gate covers the server flow, the build and the types.)

**Phase 3 — Fix with root-cause discipline.** Every break walks six gates (intent, observe, prove the cause at a specific line, blast radius, minimal change, verify fix and collateral) and gets an Evidence Ledger before a character is edited. Unattended, never blind. Below 70% confidence in the cause, it stops instead of guessing.

**Phase 4 — Seven lenses.** QA, Product, Tech Lead, Business Analyst, DevOps, Stakeholder, Integrator. These catch the flow where every link is individually wired and the whole is still wrong.

**Phase 5 — Confidence gate.** `1.0` is the only clean pass and the only result that may be called "wired". `0.95–0.99` is a conditional pass that must name every gap. Below `0.95` is a fail. Rounding up is banned.

The score is computed, not chosen. Before writing it, a **Leniency Check** answers five questions in the report: how many links carry `file:line`, how many commands were actually executed and pasted, whether the arithmetic closes, what was *not* checked (mandatory even at 1.0), and — named specifically — the most likely thing a skeptical reviewer would find that the run missed. "Nothing" is not an accepted answer. An agent grading its own work scores highest by checking least, so that joint is closed deliberately.

Every run ends in a structured report whose arithmetic must close: mapped links = wired + fixed + blocked + unproven.

## Two Modes

```
flow-check <feature>            standard — local evidence only
flow-check <feature> --deep     deep — no assumption survives
```

**Standard** runs the Execution Gate against whatever is on this machine: typecheck, build, tests, local endpoint, local database. It never leaves the machine and never asks for credentials. A link only provable remotely is `UNPROVEN`, and the score is capped.

**`--deep`** changes the contract in four ways:

1. **Execution stops being optional.** In standard mode a skipped runnable command lowers the score. In deep mode it fails the run. If a fact can be obtained by running something, it gets run — "the code says it returns 201" is not permitted when curling the endpoint was possible.
2. **It reaches the real environment when the flow warrants it** — infrastructure absent locally (queue, scheduler, object storage, webhook receiver), environment-dependent config, a symptom that only appears deployed, schema state that must be confirmed where it runs, or a seam crossing a proxy or gateway you cannot simulate. Convenience is not warrant; anything provable locally is proven locally.
3. **It asks for missing access rather than shrugging.** One consolidated Access Request naming each item, the link it unblocks, and the exact command that will run. Never drip-fed. Never more than the links require. Staging preferred; production only with a stated reason. A refusal is handled gracefully — those links become `UNPROVEN` with "access declined" and the run continues, capped.
4. **The command sweep widens** to live endpoint, live persistence, live config resolution, live migration state, live queue and scheduler, and live logs tailed while the flow executes.

Deep mode touches real systems, so it runs under safety rails: read-only by default on anything shared or production, explicit per-command confirmation for every write, destructive commands (`migrate:fresh`, `drop`, `truncate`, mass delete) banned outside local or disposable databases, synthetic test records named and cleaned up with the result reported, and credentials never echoed into the report, a log, a file, or a pasted transcript.

And the code stays the reference. If a live result contradicts the source, that is deployed drift — a stale build, a cached config, an unrun migration — and it is `BROKEN` until explained. It is never a reason to quietly revise the map to match.

## Installation

```bash
npx skills add olakunlevpn/olakunlevpn-flow-check
```

Or copy the folder into your agent's skills directory:

```bash
cp -r olakunlevpn-flow-check ~/.your-agent/skills/
```

## Use

```
/flow-check checkout
/flow-check checkout --deep
check the flow for user registration
is the invoice export actually wired?
map the notification flow and verify it --deep
```

It also auto-triggers when a feature is about to be called done, after frontend and backend meet for the first time, and on seam-shaped symptoms — the button does nothing, the list is always empty, the form submits but nothing saves, the data renders as `undefined`.

## What It Will Not Do

- Pass a link because the code around it looks right
- Batch-pass a group of links that "follow the same pattern"
- Audit only the happy path
- Patch a break before proving its cause
- Fix one link without re-verifying everything its change touches
- Report a contamination category as "none" without the command output behind it
- Reach 1.0 on reading alone, with nothing executed
- Edit or delete a test to turn the execution gate green
- Describe anything under 1.0 as wired

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
