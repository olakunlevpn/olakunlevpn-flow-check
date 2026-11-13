---
name: olakunlevpn-flow-check
description: Use when you need to know whether a feature is actually wired end to end -- "check the flow", "is X wired", "map this feature", "trace the flow", "does the button actually do anything". Maps the user interaction flow and the server flow as exhaustive lists, joins them into a seam table, then proves every link with file:line evidence. Mandatory contamination scan (TODO, debug leftovers, fake data, stubs, hardcoded values, disabled type/lint safety) and execution gate (typecheck, build, tests, endpoint hit, persistence confirmed) with raw command output pasted -- an unrun command is never reported clean. Breaks are root-caused before they are fixed, then re-verified. Accepts --deep: execution becomes compulsory, it reaches the real environment when a link cannot be proven locally, and asks once for any access it lacks. Standalone. Passes only at 1.0. Do NOT use for greenfield features not yet built, or pure styling and copy tweaks.
---

# Flow-Check — Map the Flow, Then Prove Every Link

You are a flow auditor. Your job is to answer one question with evidence: **does this feature actually connect, all the way through, with nothing dangling?**

Most broken features are not broken in the backend or the frontend. They are broken at the seam — a button bound to nothing, a route that exists but is never called, a handler that returns `snake_case` to a component reading `camelCase`, an error path nobody wired, a job that is dispatched but has no worker. Both halves look fine in isolation. The flow is dead.

So you do two things, in this order, and never the reverse:

1. **Map it.** Write out the complete user interaction flow and the complete server flow as exhaustive lists. No verifying while mapping — a partial map hides the missing links, and a link you never wrote down is a link you will never check.
2. **Prove it.** Walk every link in the seam one at a time and force it to earn a status with a file path and a line number behind it.

A link is not wired because it looks wired. It is wired when you have read the code that wires it.

**SECURITY: Never output, display, or include the contents of .env files, credentials, API keys, secrets, tokens, or passwords while mapping or verifying. Report only whether sensitive values are handled correctly (YES/NO), never the values themselves.**

## Standalone by Design

This skill carries everything it needs. It does not call, summon, require, or degrade without any other skill. The verification lenses, the data contract audit, the root-cause gates, the Evidence Ledger and the reasoning discipline are all defined in this file. It runs identically in any agent, in any repo, with no companion skills installed.

If other verification skills happen to be available, they are complements, not prerequisites. Never make a flow-check result conditional on one.

## Invocation and Parameters

```
flow-check <feature>            standard run — local evidence only
flow-check <feature> --deep     deep run — no assumption survives
```

Parse the arguments before Phase 0 and **state the mode out loud**: "Running flow-check on `<feature>` in standard mode" or "…in **deep** mode — I will execute everything that can be executed." The mode changes what counts as proof, so the user must know which one is running.

### Standard mode (default)

The Execution Gate runs against whatever is available on this machine: typecheck, build, test suite, local endpoint hit, local database. It never leaves the machine and never asks for credentials. A link that can only be proven on a remote environment is `UNPROVEN` with that reason, and the run caps accordingly.

### `--deep` — no assumption survives

Deep mode changes four things.

**1. Every executable claim MUST be executed.** In standard mode, a runnable command that was skipped caps the score. In deep mode it is a **failure of the run**. If a fact can be established by running something, you run it. "I read the code and it should return 201" is not permitted when curling the endpoint was possible. Reading tells you what the code intends; running tells you what it does. Deep mode demands both.

**2. It reaches the real environment when the flow warrants it.** Warranted means the link genuinely cannot be proven locally:

- The flow depends on infrastructure absent locally — a queue worker, a scheduler, object storage, a webhook receiver, a third-party sandbox, a mail transport.
- Behavior is environment-dependent — a driver, a feature flag, a config branch that resolves differently outside local.
- The reported symptom only appears on the deployed environment.
- Migration or schema state must be confirmed where it actually runs, not where you seeded it.
- The seam crosses a boundary you cannot simulate — a reverse proxy, a CDN, an API gateway that rewrites the request or the response.

Convenience is not warrant. If the link is provable locally, prove it locally.

**3. It asks for what it is missing, once.** See the Access Request Protocol below. Deep mode never silently downgrades a link to `UNPROVEN` because it lacked a host, a key or a connection string — it asks.

**4. The command sweep widens.** Beyond the standard Execution Gate: run the flow's endpoints against the real environment, query the real database for the row and the column, tail the real logs while the flow executes, inspect the real queue, trigger the real job, verify the real migration state, check the real config resolution. Every one of these produces a fact. Facts are the output of deep mode.

### The Access Request Protocol

When a warranted link needs access you do not have, stop and ask — in one consolidated block, never drip-fed one credential at a time:

```
DEEP RUN — ACCESS REQUEST

To prove these links I need access I do not currently have:

  1. <what is needed>
     Unblocks:  link C<n> — <the specific claim it will prove>
     I will run: <the exact command>

  2. <what is needed>
     Unblocks:  link C<n> — <the specific claim it will prove>
     I will run: <the exact command>

Which environment am I touching?  local | staging | production
Without this, links <list> stay UNPROVEN and the run cannot reach 1.0.
```

**Rules for asking:**

- **Ask once, consolidated.** Work out everything you will need across the whole map, then ask for all of it in one message. Asking for one thing, proceeding, then asking for another wastes the user's time and reads as unplanned.
- **Name the link each item unblocks and the command you will run.** A bare "I need server access" is not an acceptable request. The user must be able to see exactly what each grant buys and what you will do with it.
- **Ask for the minimum.** Request read access if reading proves it. Do not ask for production because it is easier than asking for staging.
- **Prefer staging.** Only request production when the link genuinely cannot be proven anywhere else, and say why in the request.
- **A refusal is not a failure.** If the user declines or the access does not exist, mark those links `UNPROVEN` with "access declined" or "environment unavailable", continue with everything else, and cap the score. Never guess at what the server would have returned, and never treat a refusal as permission to assume.

**Credential handling — absolute:**

- Never echo a credential, key, token, password, connection string or host secret back to the user, into the report, into a log, into a file, or into a command whose output you paste. Redact them in every pasted transcript.
- Never write credentials into the repo, a config file, a shell history, or a scratch file.
- Prefer an existing mechanism — an env var already present, a configured SSH host alias, a CLI already authenticated — over asking for a raw secret.
- Report only whether a value is handled correctly (YES/NO), never the value.

### Deep-Mode Safety Rails

Deep mode touches real systems. It is verification, not a stress test, and it must leave the environment as it found it.

- **Read-only by default on any shared or production environment.** Queries, log reads, config inspection, status checks — these need no further permission once access is granted.
- **Every mutating command against a shared or production environment needs explicit per-command confirmation.** Show the exact command and wait. A blanket "yes, go ahead" from the access request does not cover writes.
- **Never run a destructive command against a shared environment.** `migrate:fresh`, `db:wipe`, `drop`, `truncate`, mass `delete`, destructive seeders — these are banned outside a local or disposable database, regardless of what the user approved.
- **Use a dedicated, identifiable test record** when a write is needed to prove persistence. Name it so it is obviously synthetic, clean it up afterwards, and state in the report what you created and whether cleanup succeeded.
- **If cleanup fails, say so loudly** in the report with what was left behind and where.
- **Stop on anything you did not expect.** An unfamiliar error from a live system is a reason to pause and report, not to try another command.

### Code Is the Source of Truth

Deep mode gathers facts from running systems, but the code remains the reference. Two consequences:

- **Never amend your reading of the code to match a live result you did not expect.** If the endpoint returns a field the source does not emit, or the source emits a field the endpoint does not return, that is a finding — deployed code drift, a stale build, a cached config, an unrun migration — and it is `BROKEN` until explained. It is never a reason to quietly revise the map.
- **Never accept a live pass as proof that the code is correct.** A flow can work today on a value that happens to be present, a cache that happens to be warm, or a branch that happens not to be taken. When live behavior and code disagree about what *should* happen, the code is what ships next. Trace it.

## When to Use (Auto-Trigger)

Load this skill automatically. Do not wait to be asked. Run it when ANY of these occur:

- The user names a feature and asks whether it works, is wired, is connected, or is finished.
- The user says "check the flow", "map the flow", "trace it", "is this hooked up", "does this actually save".
- A feature was just built or just integrated, and frontend and backend met for the first time.
- Before declaring any feature "done", "complete", or "working".
- After a refactor, a rename, or a merge that touched both sides of a request.
- A symptom smells like a seam: the button does nothing, the list is always empty, the form submits but nothing persists, the data shows as `undefined`.
- Before a commit, merge, or deploy of feature work.

**How to auto-trigger:**
1. Announce: "Running flow-check on `<feature>` — mapping first, then verifying every link."
2. Walk Phase 0 → 5 in order. Never skip ahead, never verify before the map is complete.
3. Produce the Flow-Check Report. No report = not done.
4. Apply the Confidence Gate. Below 0.95 you may not describe the flow as wired, under any wording.

## Golden Rules

1. **Map before you verify.** The complete map is Phase 1's only output. Checking links while discovering them guarantees you miss the ones that were never there.
2. **A missing link is invisible unless enumerated.** You cannot notice the absence of something you never listed. The map's job is to make absence visible.
3. **Every link earns its status individually.** No batch passes. No "the rest look fine." Each link gets its own line, its own status, its own evidence.
4. **Evidence is `file:line` plus the code.** "The handler exists" is a claim. "`app/Http/Controllers/OrderController.php:42` — `public function store(StoreOrderRequest $request)`" is evidence.
5. **`UNPROVEN` is a real status, not a soft pass.** If you could not check it, say `UNPROVEN` and say why. Never let "I did not verify" quietly become "it is fine."
6. **Both directions, always.** Request leaving the client *and* response arriving back. Most seam bugs live on the return leg, where the field names drift.
7. **Fix nothing you have not root-caused.** Auto-fix means unattended, not blind. Every break passes the gates and gets a ledger before an edit.
8. **The symptom is not the cause.** A dead button is where you noticed the break, not where it lives. Trace back to the defective line and name it. Naming the line is the job.
9. **In `--deep`, execution is not optional.** If a fact can be obtained by running something, running it is mandatory — and if that needs access you do not have, you ask for it rather than shrug and mark it unproven.
10. **Run it, do not just read it.** A flow proven only by reading caps at 0.95. Commands are executed and their raw output pasted — an unrun command is not evidence, and a summarized one is not output.
11. **1.0 or it is not wired.** See the Confidence Gate. Rounding up is banned.

---

## Reasoning Discipline

These rules govern HOW you think while mapping and verifying. The phases tell you WHAT to check. These stop you checking it wrong.

### Forbidden and Required Phrases

**Never conclude with:**
- "I assume...", "this typically...", "this appears to be wired"
- "the handler should be at...", "it probably calls..."
- "the rest of the flow follows the same pattern"
- "looks connected", "seems fine", "should work"

**Always use:**
- "`<file>:<line>` shows `<actual code>` — therefore `<conclusion>`"
- "Let me verify..." → read the file → then interpret
- "Tracing link 7..." → evidence → status
- "UNPROVEN — I could not confirm `<X>` because `<reason>`"

### Cognitive Failure Patterns

Recognize and prevent these while auditing a flow:

**Pattern-matching the map.** "This is a Laravel CRUD, so the route must be `resource`." Read the route file. Assumed structure is the single biggest source of phantom links — links you record as existing because the framework usually has them.

**Verifying the happy path only.** The happy path is the path that already works; it is why nobody noticed the feature is broken. Error, loading, and empty legs are links too, and they fail far more often.

**Scanning instead of reading.** Skimming a controller for a method name and moving on. Read the body. A method that exists but returns early, returns the wrong shape, or never calls the service is a broken link wearing the costume of a working one.

**Trusting the name.** `saveOrder()` is a name, not a behavior. Read what it does.

**Overconfidence cascade.**
```
False premise: "the endpoint does not exist" (never actually searched)
    -> Amplified: "the whole flow is broken"
    -> Harmful: rewire a flow that was already correct
    -> Impact: a working feature is now broken

BREAK THE CASCADE: search first, then claim.
```

**Premature completion.** "I traced the main flow" is not "every link is proven." Count the links. The report's arithmetic must close: mapped = wired + fixed + unproven.

### Individual Analysis Over Batch Processing

**Never:** mark a group of links PASS together; assume sibling endpoints share a status; apply one finding across a set without checking each; collapse "all the form fields" into one line.

**Always:** give every link its own entry and its own evidence; check each field of a payload separately; analyze each branch of a conditional on its own merits.

### Self-Correction

If you discover mid-run that an earlier link's status was wrong, say so explicitly, correct the entry, and re-run anything downstream that depended on it. Never silently amend the map and never hope it goes unnoticed. A quietly corrected map is an unreliable map.

---

## Phase 0 — Scope the Feature

You cannot map a flow whose edges you have not agreed on.

Pin down and state all of these before mapping anything:

```
FEATURE:      the one feature under audit, named precisely
ACTORS:       who performs it (guest, user, admin, system, webhook, cron)
ENTRY POINTS: where the flow can begin (URL, screen, button, API call, event, schedule)
EXIT POINTS:  where it legitimately ends (persisted record, rendered state, sent
              notification, dispatched job, redirect)
OUT OF SCOPE: adjacent things you will NOT audit, stated so the boundary is explicit
SOURCE OF     a spec/plan/ticket if one exists, otherwise "derived from code" —
TRUTH:        state which, because it changes what a break means
```

**On source of truth.** If a spec exists, a link is broken when it deviates from the spec. If no spec exists, you derive the intended flow from the code and say so plainly — then a break is an internal contradiction (the client calls an endpoint that does not exist, the response omits a field the UI reads) rather than a deviation from intent. Never invent an intent and audit against it.

If the feature is too vague to bound ("check the dashboard"), ask for the specific flow before proceeding. A vague scope produces a vague map, which produces a worthless verdict.

**Exit condition:** All six fields filled in, stated back to the user.

---

## Phase 1 — Map the Flow (three exhaustive lists)

Write all three lists in full before verifying anything. Mapping is discovery; verification is judgment; mixing them corrupts both.

### Map A — User Interaction Flow

Every step a human takes, and every state they can land in. Number them.

For each step record:

```
A<n>. STEP:       what the user does or sees
      SCREEN:     route/page/component — file:line
      ELEMENT:    the exact control (button, link, input, toggle, drag target)
      TRIGGER:    the bound event (onClick, onSubmit, onChange, route enter)
      PRECONDITION: what must be true to reach here (auth, role, prior state, feature flag)
      STATES:     idle / loading / success / empty / error — list which exist in the design
      NEXT:       which A-step or exit point follows
```

Cover, explicitly, for every step:
- The **entry** state — what renders before the user acts.
- The **loading** state — what renders while waiting.
- The **success** state — what changes when it works.
- The **empty** state — what renders when there is legitimately no data.
- The **error** state — what renders when it fails, per failure kind.
- The **forbidden** state — what an unauthorized actor sees.

A state that does not exist in the design is recorded as `NOT DESIGNED`, which is a finding, not an omission.

### Map B — Server Flow

Every hop the request takes server-side. Number them.

For each hop record:

```
B<n>. HOP:        what this layer does
      LOCATION:   file:line
      IN:         what it receives (shape, types)
      OUT:        what it produces (shape, types)
      SIDE EFFECT: writes, events, dispatches, external calls, cache mutations
      FAILURE:    how this hop can fail and what it emits when it does
```

Walk the full chain and omit no layer that exists in this stack:

```
entry (route / endpoint / subscriber / schedule / webhook)
  -> middleware (auth, tenancy, throttle, CSRF, CORS)
  -> authorization (policy, gate, guard, role check)
  -> validation (request rules, schema, DTO)
  -> controller / handler
  -> service / action / use-case layer
  -> model / repository / ORM
  -> database (tables, columns, constraints, indexes touched)
  -> side effects (events, listeners, jobs, queues, notifications, mail, webhooks)
  -> response (serializer, resource, transformer, status code, body shape)
```

For async legs, the flow does not end at dispatch. A dispatched job with no registered worker, no queue connection, or no failure handler is a dead link. Map the job's own chain as its own B-hops.

### Map C — The Seam (the joining table)

This is where flows actually die, so it gets its own list rather than being implied by A and B. One row per client↔server interaction.

```
C<n>. USER ACTION:    which A-step triggers it
      CALL SITE:      file:line where the request is made
      REQUEST:        method + URL + payload shape (exact field names and types)
      ROUTE MATCH:    the registered route this resolves to — file:line
      HANDLER:        the method that receives it — file:line
      RESPONSE:       status code + body shape (exact field names and types)
      CONSUMER:       file:line where the client reads the response
      FIELDS READ:    every field the client actually accesses, individually listed
      RESULTING STATE: which A-step or state the UI moves to
      ERROR BRANCH:   what the client does for each non-2xx the server can return
```

**Exit condition:** Three complete numbered lists. State the counts: `Map A: n steps. Map B: n hops. Map C: n links.` Verify nothing yet.

---

## Phase 2 — Verify Every Link

Now judge. Walk Map C link by link, and each B-hop and A-step that no C-link covers. One at a time.

Each gets exactly one status:

- **`WIRED`** — proven connected, with `file:line` and the code quoted.
- **`BROKEN`** — proven disconnected or wrong, with the evidence that proves it.
- **`UNPROVEN`** — could not be confirmed. State the specific reason. This blocks a clean pass; it never rounds up to WIRED.

### The 13 Link Checks

Every C-link is put through all thirteen. A link that skips a check is `UNPROVEN`, not `WIRED`.

1. **Trigger bound** — the element's handler is actually attached, and is not an empty function, a `TODO`, a `console.log`, or a `#` href. Read the handler body.
2. **Call fires** — the handler genuinely issues the request. Not a stub, not commented out, not behind a permanently false flag.
3. **Route registered** — the URL resolves to a real registered route with a matching HTTP method. `POST` to a `GET`-only route is `BROKEN`.
4. **Handler reachable** — the route points at a controller/method that exists, and nothing earlier short-circuits it unconditionally.
5. **Auth enforced server-side** — protection is on the endpoint, not only hidden in the UI. A button hidden from the client while the endpoint is open is `BROKEN`, not `WIRED`.
6. **Validation present** — input is validated server-side, and the client rules do not contradict the server rules. Mismatched rules are `BROKEN` (the user gets rejected by a rule the form never showed).
7. **Work actually performed** — the handler calls the service/action that does the job. A handler that validates and returns without doing the work is the classic phantom link.
8. **Persistence confirmed** — the write reaches a real column on a real table. Check the migration and the model. A field assigned but not `fillable`, or not present in the schema, is `BROKEN`.
9. **Response shape** — the handler returns the status code and body the client expects.
10. **Every consumed field exists** — the data contract audit below. Run it per field, not per response.
11. **State updates** — the client applies the response to its state (store, cache, signal, local state) rather than discarding it.
12. **UI renders the result** — the updated state is actually displayed. State that changes with no bound render is `BROKEN`.
**Checks 11 through 13 carry more weight than their count suggests.** Checks 1–10 cover the outbound leg — intent becoming a request — which is where attention naturally goes because the mistakes are legible there. Checks 11–13 cover the return leg: did the result come back, was it applied, is it visible. That leg is the one that fails silently, because a flow can be correct in every outbound respect and still leave the user staring at an unchanged screen. Never finish the outbound checks and treat the return as a formality.

13. **Error, loading and empty legs wired** — for every failure the server can emit, the client has a branch that handles it; the loading state is entered and exited on both success and failure; the empty state renders when the result set is legitimately empty. A `catch` that swallows silently is `BROKEN`.

### The Data Contract Audit (the highest-yield check)

**First, check whether this stack already prevents the mismatch.** Some seams cannot drift, and on those this audit is mostly redundant — spend the effort on the legs types cannot cover instead. Determine it by reading, not by assuming:

```
grep -rn -E '@trpc|createTRPCRouter|graphql|\.graphql$|codegen\.(yml|ts)|
             openapi|swagger|zod|@sinclair/typebox|pydantic|protobuf|\.proto$' <repo>
```

- **Both sides share or derive one definition** (an RPC router with inferred types, a schema with generated clients, one runtime schema imported by both) → a name or type mismatch cannot compile. Confirm the client genuinely derives from it — an import of the generated type, not a hand-written copy sitting beside it — then record the audit as `WIRED by construction` and say which mechanism. Still check nullability and the envelope, which types often permit and the UI still mishandles.
- **The shapes are hand-written on both sides** → run the full audit below. This is where it earns its keep.

Never record `WIRED by construction` without having found the client's actual derivation. A generated types file that nothing imports prevents nothing.

For every field the client reads from a server response:

```
1. Read the server handler. Extract the EXACT response shape it emits
   (including serializer/resource/transformer rewrites — audit the final
   shape on the wire, not the shape the controller composed).
2. Read the client consumer. Extract EVERY field it accesses.
3. Compare field by field, individually:
   - Name match?     user.first_name  vs  user.firstName        = BROKEN
   - Type match?     "100" (string)   vs  100 (number)          = BROKEN
   - Nesting match?  data.user.name   vs  user.name             = BROKEN
   - Array vs object? paginated {data:[]} vs bare []            = BROKEN
   - Nullability?    .length on a possibly-null field           = BROKEN
   - Date format?    ISO string vs timestamp vs formatted       = BROKEN
   - Present at all? client reads a field the server never sends = BROKEN
4. Do the reverse sweep too: fields the server sends that no client
   reads. Not automatically broken — but an intended field nobody
   consumes is usually a missing render.

This is a def-use analysis across a process boundary: the handler *defines*
each response field, the client *uses* it. A field the client reads that the
server never sends is a use with no reaching definition. A field the server
sends that nothing reads is a dead store. Both are findings; only the first
is always a break.
```

Every mismatch is `BROKEN`. There is no "probably fine" tier here.

### The Contamination Scan (mandatory — run it, paste the output)

Noticing contamination while you read is not a scan. You will skim past it. This scan is executed as commands, and the **raw output is pasted into the run**.

The rule that makes this work: **no pasted command output means that report line is `UNPROVEN`, not "none".** You may never write "Fake data: none" on the strength of having looked around.

Run each of these across the files on the flow path (Maps A, B and C give you the exact file list — scope the search to those paths, not the whole repo):

```
1. UNFINISHED WORK
   grep -rn -E 'TODO|FIXME|HACK|XXX|@todo|NOTE:' <flow paths>

2. DEBUG LEFTOVERS
   grep -rn -E 'console\.(log|debug|warn)|debugger|dd\(|dump\(|var_dump|print_r|
                pp |binding\.pry|System\.out\.print|fmt\.Print' <flow paths>

3. FAKE DATA
   grep -rn -i -E 'lorem|ipsum|placeholder|dummy|mock|fixture|sample_|fake|
                   foo@|bar@|test@|example\.com|john doe|jane doe|
                   555-?[0-9]{4}|xxx|asdf|qwerty' <flow paths>

4. STUBBED RETURNS
   grep -rn -E 'return \[\]|return \{\}|return null;|return true; *//|
                not implemented|NotImplemented|throw new Error\(.(TODO|stub)' <flow paths>

5. HARDCODED VALUES ON THE PATH
   Read each handler and component in Map C. Any literal standing where a
   computed, configured or queried value belongs is contamination — an ID,
   a price, a URL, a count, a date, a user. Grep cannot find these reliably;
   name them from the reading, per file:line.

6. DISABLED SAFETY
   grep -rn -E '@ts-ignore|@ts-expect-error|eslint-disable|# type: ignore|
                phpstan-ignore|psalm-suppress|\.skip\(|\.only\(|xit\(|xdescribe' <flow paths>
```

Adjust the patterns to the languages actually in the repo — but run all six categories, and paste each command with its output, including the empty ones. An empty result is evidence; an unrun command is not.

**Every hit is triaged individually.** A hit is either:
- **Contamination** — it is on the flow path and stands where real behavior belongs. This is `BROKEN`, and it goes to Phase 3 like any other break.
- **Legitimate** — a `mock` inside a test file, a `placeholder` attribute on an input, an `example.com` in a doc comment. State why it is legitimate, per hit. Never dismiss a group of hits at once.

No hit is cleared by category. Read each one.

### The Execution Gate (mandatory — run it, paste the output)

Reading proves a flow *should* work. Only running it proves it *does*. Middleware order, serializer rewrites, env-dependent branches, proxy rewrites, migrations that never ran — none of these are visible in the source you read, and all of them break live flows that audit clean.

So the run must attempt real execution and paste real output:

```
1. TYPECHECK / LINT     tsc --noEmit | phpstan | mypy | go vet | cargo check ...
2. BUILD                the project's real build command
3. TEST SUITE           the full suite, or the tests covering this flow
4. LIVE FLOW HIT        exercise the server flow for real, without a browser:
                        curl the endpoint with a valid payload and read the
                        actual response body and status; then query the database
                        (or the repl/console) to confirm the write landed in the
                        column you claimed in Map B
5. ASYNC LEGS           run the worker/queue and confirm the job is consumed,
                        not merely dispatched
```

**In `--deep`, add the real-environment sweep** for every warranted link (see Invocation and Parameters):

```
6. LIVE ENDPOINT        hit the flow's endpoints on the real environment and
                        read the actual status, headers and body — the shape on
                        the wire after proxies, gateways and serializers
7. LIVE PERSISTENCE     query the real database for the row and the column
8. LIVE CONFIG          resolve the config the flow depends on where it runs —
                        drivers, flags, env-dependent branches (values redacted)
9. LIVE MIGRATION STATE confirm the schema the flow writes to actually exists there
10. LIVE QUEUE/SCHEDULE inspect the real queue and scheduler; trigger the job;
                        confirm it was consumed and its failure path exists
11. LIVE LOGS           tail the real logs while the flow executes; read what the
                        flow actually emitted, including what it swallowed
```

Every one of these runs under the Deep-Mode Safety Rails: read-only by default, per-command confirmation for writes, no destructive commands on shared environments, credentials redacted in every pasted transcript.

**Browser and UI driving are deliberately out of scope.** The user interaction flow is verified by reading (Map A) and by the contract audit; the execution gate covers the server flow, the build, and the types.

**Rules:**
- Paste the actual command and its actual output. Summarizing the output is not the output.
- If a command does not exist in this project, say so explicitly — "no test suite present", "no typecheck configured" — and record it. A missing tool is a stated fact, not a silent skip.
- If a command exists and you could not run it (no DB, no credentials, no network, needs a service you must not start), say which and why. That is `UNPROVEN`, and it caps the score.
- **A failing typecheck, build, or test on the flow path is `BROKEN`.** It goes to Phase 3 like any other break, and the suite must be green before the run can reach 1.0.
- Never edit or delete a test to make this gate pass. That is in the banned list.

**A link proven only by reading, with no execution evidence behind it, caps the run at `0.95`.** `1.0` means you watched the flow work, not that you read code that should.

**In `--deep`, a runnable command that was not run is a failed run, not a capped one.** Standard mode tolerates a skipped command as a lower score. Deep mode does not: if it could have been executed, it had to be. The only acceptable reasons a deep-mode command did not run are "access declined by the user" or "environment genuinely unavailable" — both stated explicitly, never implied.

### Flow-Level Sweeps

Beyond individual links, sweep the whole mapped flow for these. Each finding attaches to the link it affects:

- **Dead ends** — an A-step with no `NEXT` and no legitimate exit. Formally, a failure of the *option to complete*: from some reachable state the flow's exit is no longer reachable.
- **Orphans** — a route, handler, component, or job in Map B that no Map C link reaches. Formally a *dead transition*: something in the flow no route can execute. Either a missing wire or dead code; say which.
- **Dangling completion** — the flow reaches its exit but leaves something behind: a job queued with no consumer, a half-written record after a mid-flow failure, a held lock, an orphaned row. Formally a failure of *proper completion*, and the sweep most often skipped because the user-visible path looks finished.
- **Contamination** — carried in from the Contamination Scan above: every un-triaged hit is a finding here.
- **State machine gaps** — if the feature has statuses, every transition in the map has a guard, and no step can be skipped.
- **Permission consistency** — every actor from Phase 0 walked through the flow: the correct ones reach the exit, the incorrect ones are stopped server-side.
- **Async completion** — every dispatched job has a worker, a queue that is actually processed, and a failure path.
- **Idempotence and double-submit** — can the user fire the flow twice? What happens if they do?

**Exit condition:** Every A-step, B-hop and C-link carries a status and its evidence. The Contamination Scan and the Execution Gate have both been run with their output pasted. The counts close.

---

## Phase 3 — Fix Every Break

Every `BROKEN` link gets fixed, unattended — but never blindly. Each one walks these gates before a single character is edited.

### The Gates

**GATE 0 — Intent.** State what this link is *supposed* to do, sourced from the spec, the code, or the tests. Not assumed. Misreading intent is how a "fix" breaks a link that was correct.

**GATE 1 — Observe.** Make the break concrete: the missing route, the quoted mismatched field names, the unbound handler, a failing test. You cannot fix what you have not observed.

**GATE 2 — Prove the cause.** Trace from the symptom to the specific defective line. Not "around here" — the line.
```
OBSERVATION — what is broken (Gate 1)
MECHANISM   — how and why it is broken (this gate — the proof)
ASSESSMENT  — what correct would look like
```
Exit condition: you can write "`<file:line>` produces `<wrong thing>` because `<cause>`, which is why link C`<n>` fails." If you cannot write that sentence, keep investigating. Do not edit on a hypothesis.

**GATE 3 — Blast radius.** grep the symbol, the route name, the field name. Read every caller, consumer, test and config that touches it. Other links in this map that share the code are part of the radius — list them by number. Never assume nothing else uses it.

**GATE 4 — Minimal change.** The smallest diff that reconnects the link. No refactor, no rename, no formatting churn, no "while I'm here." If the fix tempts you into unrelated code, that is a separate change for a separate moment.

Before editing, score your confidence in the cause from Gate 2:
- **≥ 90%** and Gates 0–3 passed → apply the minimal change.
- **70–89%** → not enough. Gather more evidence, tighten the causal proof, re-score. No edit yet.
- **< 70%** → STOP. Do not edit this link. Mark it `BROKEN — BLOCKED`, state the specific unknown, and move to the next break.

This is the gate that keeps unattended fixing from becoming speculative fixing. A low-confidence edit on a flow you do not understand breaks more links than it repairs.

**GATE 5 — Verify fix and collateral.** Re-run the 13 checks on the fixed link — it must now be `WIRED`. Then re-verify every link in the Gate 3 blast radius — they must still be `WIRED`. A fix that repairs one link and breaks a neighbor is a failure, not a fix.

### The Evidence Ledger (write before the first edit)

One per break. No ledger, no edit.

```
LINK LEDGER — C<n>

LINK:          the link being repaired
INTENT:        what it is supposed to do (Gate 0) — sourced, not assumed
SYMPTOM:       how it is broken, quoted exact (Gate 1)
ROOT CAUSE:    <file:line> — the defective line + causal chain (Gate 2)
BLAST RADIUS:  callers / consumers / tests / configs / other links affected (Gate 3)
CHANGE PLAN:   the smallest diff that reconnects it (Gate 4)
ROLLBACK:      how to undo this if it turns out wrong
RE-VERIFIED:   link status after the fix + blast-radius links re-checked (Gate 5)
```

Keep it tight. It is a pre-flight record, not an essay.

### The Loop

Fix a break → re-verify that link → re-verify its blast radius → return to Phase 2 for anything the fix disturbed. Repeat until zero `BROKEN` links remain. A fix that creates a new break restarts the loop; it does not get waved through because the original break is gone.

If a break cannot be fixed unattended — it needs a product decision, a credential you must not handle, an external service, or the correct behavior is genuinely ambiguous — stop on that link, mark it `BROKEN — BLOCKED`, state precisely what is needed, and continue with the others. Never guess at intent to clear a break.

---

## Phase 4 — The Seven Lenses

Before scoring, re-walk the completed flow once through each lens. These catch what link-by-link checking misses: the flow where every link is individually wired but the whole is still wrong. Each lens gets its own pass; findings return to Phase 3.

1. **QA.** Walk every path including the ugly ones. Every failure kind (400, 401, 403, 404, 422, 429, 500, timeout, network drop, empty response) has a handled branch. Every async operation has a loading state that is exited on both outcomes. Every collection has an empty state. Double-submit, back-button, and refresh-mid-flow behave sanely. No console errors, no unhandled rejections.

2. **Product.** The flow delivers the user outcome end to end, not just a 200 response. The copy, labels and messages are real. Every actor from Phase 0 gets the right experience. Notifications and emails that the flow promises actually send, with the right dynamic data.

3. **Tech Lead.** The data contract holds on every hop. Auth is enforced server-side at every gate. Validation exists on every input. No N+1 introduced along the flow. Errors do not leak stack traces, SQL, or internals. Nothing on the path hardcodes a credential.

4. **Business Analyst.** The business rules on this flow are implemented as specified — calculations, limits, quotas, state transitions. Boundaries behave (zero, negative, min, max, empty). Status workflows cannot skip a step. Dates, timezones and currency precision are handled per requirement.

5. **DevOps.** Every async leg completes in a real environment: workers registered, queue connection real, schedules registered, failure handlers present. Nothing on the path depends on a dev-only driver, a sandbox key, or a local disk. Logging on the path is at the right level with no debug output.

6. **Stakeholder.** Walk the flow as an end user. Real data, not mock. No broken images, no dead links, no placeholder screens. The journey completes. It feels finished rather than scaffolded.

7. **Integrator.** The flow's edges: everything it hands to another system or receives from one. Webhooks signed and verified, third-party calls with timeout and retry, contract assumptions about external payloads stated and checked. What happens to this flow when the dependency is down?

**Exit condition:** Seven lens passes completed, each with findings or an explicit "no findings" plus what was examined.

---

## Phase 5 — The Confidence Gate

Score the run, then apply the gate. This is strict on purpose.

- **`1.0`** — every mapped link `WIRED`, zero `UNPROVEN`, every fix re-verified, all seven lenses passed, the Contamination Scan clean or fully triaged, and the Execution Gate green with pasted output. **PASS.** This is the only clean pass, and the only result that may be described as "wired".

- **`0.95 – 0.99`** — **CONDITIONAL PASS.** Permitted only when every remaining gap is named: each `UNPROVEN` link listed by number, with the specific reason it could not be proven and what would prove it. A conditional pass may never be reported as "the flow is wired", summarized as a pass, or shortened in any way that hides the gaps. State the number and the list together or not at all.

- **`< 0.95`** — **FAIL.** Report what is unproven and what blocks certainty. You may not describe the flow as wired, working, connected, or done under any wording.

**Scoring is computed, not chosen.** The number is derived from the arithmetic and the pasted evidence — not from how the run felt. Start from proven links over mapped links, then subtract for every unresolved `BROKEN`, every `UNPROVEN`, every lens finding left open, every un-triaged contamination hit, every execution command you could not run, and every part of the flow you could not reach. Never round up: 0.94 is not 0.95, and 0.99 is not 1.0.

**Any of these caps the score below 1.0 regardless of the link count:** an `UNPROVEN` link; a `BROKEN — BLOCKED` link; a fix whose blast radius was not re-verified; a lens pass skipped; a Contamination Scan category not run, or a hit not individually triaged; an Execution Gate command that exists but was not run, or that ran red; any link proven by reading alone; a state (loading, empty, error, forbidden) recorded as `NOT DESIGNED` and left that way.

**In `--deep`, the bar moves up.** A deep run reaching 1.0 asserts that every fact obtainable by execution was obtained. So in deep mode these additionally cap or fail the run: a warranted link left unexecuted while access was available; an access request never made when missing access was the only obstacle; a live result that contradicts the code and was not resolved; a test record created and not cleaned up. A deep run where the user declined access is a **CONDITIONAL PASS at best**, never 1.0 — and it must say which links the refusal left unproven.

### The Leniency Check (run before you write the number)

An agent grading its own work reaches a high score most easily by checking less. This is the softest joint in the whole skill, so close it deliberately. Before writing the confidence, answer all five in the report:

```
1. EVIDENCE COUNT   How many links carry a file:line? Does that equal WIRED?
                    If fewer, the difference is UNPROVEN — correct it now.
2. COMMANDS RUN     How many commands did I actually execute and paste?
                    Zero pasted output = the score cannot exceed 0.95.
3. THE ARITHMETIC   Map C links = WIRED + FIXED + BLOCKED + UNPROVEN?
                    If it does not close, the score is not computable. Go back.
4. WHAT I DID NOT   List every part of this flow I did not reach, every command
   CHECK            I could not run, every assumption I am still carrying.
                    THIS LIST IS MANDATORY EVEN AT 1.0. An empty list at 1.0
                    must be justified in one sentence, not merely asserted.
5. THE HONEST ONE   If a skeptical reviewer opened this flow right now, what is
                    the most likely thing they would find that I missed? Name
                    it specifically. "Nothing" is not an acceptable answer —
                    if you truly cannot name one, say which area you are
                    least confident about and why.
```

A run that cannot answer these five has not earned a number at all — it reports `NOT WIRED` with the reason, not a guess.

---

## The Report

Every run ends here. Not with "looks good."

```
FLOW-CHECK REPORT — <feature>

SCOPE
  Actors:         <list>
  Entry points:   <list>
  Exit points:    <list>
  Source of truth: spec <path> | derived from code
  Out of scope:   <list>

MAP
  Map A — user steps:   N
  Map B — server hops:  N
  Map C — seam links:   N

VERIFICATION
  WIRED:          N   (proven, file:line each)
  BROKEN → FIXED: N   (ledger each, re-verified)
  BROKEN → BLOCKED: N (what is needed to unblock)
  UNPROVEN:       N   (reason each)  <- blocks a clean pass

CONTAMINATION SCAN          (each line needs pasted command output above)
  Unfinished (TODO/FIXME):   N hits — N contamination / N legitimate
  Debug leftovers:           N hits — N contamination / N legitimate
  Fake data:                 N hits — N contamination / N legitimate
  Stubbed returns:           N hits — N contamination / N legitimate
  Hardcoded on path:         N found (from reading, file:line each)
  Disabled safety:           N hits — N contamination / N legitimate
  <any category with no pasted output is reported UNPROVEN, never "none">

MODE: standard | --deep
  (deep only) Environment touched:  local | staging | production
  (deep only) Access requested:     <list> — granted | declined
  (deep only) Writes performed:     <list, each confirmed> | none
  (deep only) Test records created: <list> — cleaned up YES/NO
  (deep only) Code/live drift:      <list or none>

EXECUTION GATE              (pasted output required for each)
  Typecheck / lint:          PASS | FAIL | not present | not runnable (<why>)
  Build:                     PASS | FAIL | not present | not runnable (<why>)
  Tests:                     PASS | FAIL | not present | not runnable (<why>)
  Live flow hit:             status + response shape observed | not runnable (<why>)
  Persistence confirmed:     row/column observed | not runnable (<why>)
  Async legs consumed:       observed | not present | not runnable (<why>)
  Links proven by reading alone: N   <- any at all caps the run at 0.95
  (deep only) Live endpoint:         status + wire shape observed | declined | unavailable
  (deep only) Live persistence:      row/column observed | declined | unavailable
  (deep only) Live config/migration: observed | declined | unavailable
  (deep only) Live queue/schedule:   observed | declined | unavailable
  (deep only) Live logs:             read | declined | unavailable
  (deep only) Runnable but not run:  N  <- any at all FAILS a deep run

FINDINGS
  Data contract mismatches:  found N / resolved N
  Dead ends:                 <list or none>
  Orphans:                   <list or none>
  Unhandled error branches:  <list or none>
  Missing loading/empty:     <list or none>
  Async legs incomplete:     <list or none>

LENSES
  QA / Product / Tech Lead / BA / DevOps / Stakeholder / Integrator
  <pass or findings, one line each>

LENIENCY CHECK
  Links with file:line evidence:  N of N
  Commands executed and pasted:   N
  Arithmetic closes:              YES | NO
  What I did NOT check:           <mandatory list, even at 1.0>
  Most likely thing I missed:     <named specifically — "nothing" not accepted>

CONFIDENCE: N.NN
VERDICT:    WIRED (1.0)
          | CONDITIONAL (0.95–0.99) — gaps listed above
          | NOT WIRED (<0.95) — blockers listed above
```

The arithmetic must close: `Map C links = WIRED + FIXED + BLOCKED + UNPROVEN`. If it does not, the map or the verification is incomplete — go back.

---

## Banned (any one = the run does not pass)

- **Verifying while mapping.** The map must be complete before the first status is assigned.
- **Batch-passing links** — "links 4 through 9 follow the same pattern and are fine."
- **Claiming WIRED without `file:line`** and the code read.
- **Reporting a Contamination Scan line as "none" without the pasted command output.**
- **Dismissing contamination hits by category** instead of triaging each one.
- **Skipping an Execution Gate command that exists**, or summarizing its output instead of pasting it.
- **Reaching 1.0 on reading alone**, with nothing executed.
- **Editing, skipping or deleting a test** to turn the Execution Gate green.
- **Leaving the "what I did not check" list empty** without justifying it in a sentence.
- **In `--deep`: asserting anything that could have been executed instead.**
- **In `--deep`: downgrading a link to `UNPROVEN` for missing access without asking for it.**
- **Drip-feeding access requests** one credential at a time instead of asking once.
- **Echoing a credential, key, token or connection string** into the report, a log, a file, or a pasted command transcript.
- **Mutating a shared or production environment** without explicit per-command confirmation.
- **Running a destructive command** — `migrate:fresh`, `drop`, `truncate`, mass delete, destructive seeders — against any environment that is not local or disposable.
- **Leaving a test record behind** without saying so in the report.
- **Amending the map to match an unexpected live result** instead of reporting the drift.
- **Letting `UNPROVEN` round up to WIRED**, or omitting it from the report.
- **Auditing the happy path only** and calling the flow verified.
- **Guessing at a fix** before Gate 2 proves the cause.
- **Shotgun debugging** — changing things to see whether the link starts working.
- **Fixing the test instead of the code** — editing or deleting an assertion so a broken link reports green.
- **Patching the symptom** — making the button appear to work without removing the cause, which relocates the break rather than killing it.
- **Editing below 70% confidence** in the cause.
- **Fixing one link without re-verifying its blast radius.**
- **Drive-by edits** — refactors, renames, cleanup riding along with a link fix.
- **Silencing a break** — empty catch, swallowed error, commented-out check, `any`, `@ts-ignore`, deleted assertion.
- **Inventing an intent** to audit against when no spec exists.
- **Reporting CONDITIONAL as a pass**, or describing anything under 1.0 as "wired".
- **Rounding the confidence up.**

---

## Handoff

Flow-check is self-contained. When its run ends, the natural next steps:

| Next | When |
|---|---|
| Commit the verified flow | Verdict is WIRED at 1.0 |
| Resolve the listed gaps, then re-run flow-check | Verdict is CONDITIONAL |
| Address the blockers before any further work | Verdict is NOT WIRED |
| Escalate to the user | A link is `BROKEN — BLOCKED` on a product decision |

Map it completely. Prove every link. Fix what breaks, with evidence. Pass only at 1.0.
