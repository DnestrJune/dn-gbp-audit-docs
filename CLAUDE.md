# CLAUDE.md

## Response style

Report outcomes, not process. Assume I read diffs.

Do NOT write:

- what you are about to do, before doing it
- narration while working ("now updating X", "next I'll check Y")
- step-by-step recaps of what you just did
- restatements of my instructions
- reasoning for decisions I didn't question
- summaries of files you read or explored

After finishing work, reply with:

- one line per file changed
- test count and tsc status
- judgement calls, capped at five

## Judgement calls

Report a judgement call only when it meets one of these:

- it contradicts something I explicitly specified
- reversing it later would be expensive
- it changes behaviour I would not expect from reading my instructions

Two or three sentences each. If more than five qualify, report the five that
matter most and say how many you dropped.

Everything else — naming, file placement, test structure, refactors that
don't change behaviour — just decide and move on. Do not report it.

## Display strings

The engine emits ids and values. It never emits composed prose, so the report
needs connective text that has to be written somewhere — "4 of 6 items
present", "Peer median 4.7 — …", section headings.

**Any string added for display may name, count or connect. It may never assert
anything about the venue that the engine did not.**

That rules out, permanently: a word that grades ("good", "poor", "weak",
"score", "grade", "quality"), a number the engine did not compute, a threshold
the engine does not hold, a comparison with nothing behind it, and any claim
about a field no source exposes. If a sentence needs a fact the engine does not
carry, the fix is to stop writing the sentence — not to source the fact
elsewhere.

Two consequences worth stating because they look like exceptions and are not:

- **Reputation strings label, they do not characterise.** "600 / reviews on
  Google" is a name and a count. "600 reviews — a strong base" is a claim the
  engine has no basis for and never will, because there is nothing to grade a
  review count against.
- **Wording may vary by context; the fact may not.** The same check reads
  "Website link / Not filled in" in the completeness table and "No website" in
  the share view. Different registers, one observation. A context-specific
  string that says something the neutral one does not is a bug.

**The peer-group comparison must say "Nearby venues Google also lists as
`<type>`", never "nearby `<type>`s".** The benchmark sample matches a place's
full type list, not its primary type, so a `coffee_shop` search returns
bistros and italian restaurants that also serve coffee. "Nearby coffee shops"
is a claim the sample cannot support — a reader who recognises a mismatched
venue in the list is right to distrust the whole number. The peer list is
shown in full for the same reason: the median has to be derived from
something the reader can check, not asserted over a sample they cannot see.

The rule holds for both locales, and for anything added to the engine's own
`src/strings/{en,ro}.ts` as well as the frontend's glue table.

## Check states

**The frontend renders the engine's check states and never invents another
one.** `CheckState` is the whole vocabulary — today `present | partial | absent
| na` — and the frontend's job is to draw each one, not to add a fourth reading
of its own.

So: no state inferred from a value the frontend happens to have, no "almost" or
"needs attention" between two real states, and no mark that nothing can emit. A
square with no state behind it is worse than a missing square — it implies the
audit measured a gradation it never looked for. If a new state is wanted, it is
an engine change first: the state, its counting rule, and its words in both
locales.

## Local scripts

`scripts/live-audit.ts` and `scripts/record-fixtures.ts` use `fetch`, and
**Node does not honour `HTTPS_PROXY` on its own** the way curl does. Behind a
proxying egress gateway the symptom is actively misleading: curl succeeds
while Node gets `403 Host not in allowlist` or `503 DNS resolution failure`
against the same host, which reads as a blocked host rather than a client
that never used the proxy.

Run them with:

```sh
NODE_USE_ENV_PROXY=1 NODE_EXTRA_CA_CERTS=/root/.ccr/ca-bundle.crt \
  node scripts/live-audit.ts "venue, city"
```

`live-audit.ts` writes every raw payload to `.live-run/` on every run; `VERBOSE=1`
only controls whether they are printed to the terminal as well. Do not add dumps back
to the script to inspect a payload — read `.live-run/`.

Check this before concluding a host is blocked: confirm with curl first, and
only believe Node's verdict once it is going through the proxy.

This affects local scripts only. The Worker runs on Cloudflare with no proxy
in front of it.

## Final status block

End every reply that ran code, tests or a deploy with:

```
STATUS: OK | PARTIAL | FAILED
Tests: N passed / M failed
Types: clean | N errors
Deploy: deployed <url> | not attempted | failed
Watching: none | <mechanism> on <target> — requested at <where you asked>
Blocked: <one line each, or "none">
Needs me: <one line each, or "nothing">
```

It is the last thing in the reply, after the judgement calls, always fenced.

OK only when everything requested actually succeeded. PARTIAL when the task
finished but something was skipped, degraded, or silently unavailable.
FAILED when the core task did not complete.

Blocked = anything that stopped you and that you could not fix yourself:
network, credentials, permissions, missing access, upstream errors. Quote the
exact error string. A missing data source is Blocked even if the code around
it works.

Never report OK when a required external call did not happen.

Watching = anything left running after this reply that keeps acting without me
present: a PR subscription (`subscribe_pr_activity`), a scheduled check-in
(`send_later`, `create_trigger`), a background agent left polling. The field
has no blank state — write `none` or name the mechanism and target, and when
you name one, point at the sentence in this conversation that asked for it. If
you cannot point at one, you did not have permission to arm it: cancel it
(`unsubscribe_pr_activity`, `delete_trigger`, stop the agent) before writing
this line, not after I ask why it's still running. This line exists because
"I'll watch that" is a decision I make, not a default you reach for while
finishing unrelated work — filling it in is the check, not a reminder to run
one.

## Session start

Run scripts/preflight.sh once at the start of any session that will run code, hit an
external API, or deploy. Report only its final status line unless a check failed.

The container starts without node_modules, and there are two package trees. Run
`npm ci && (cd frontend && npm ci)` before the first test run or any wrangler command.
Do this at the start of the task, not at the end — a deploy step that fails on missing
dependencies wastes the whole session's work.

A blocked host may be a transient gateway failure. Re-run scripts/preflight.sh once
before concluding. If it is still blocked, that is Blocked — do not diagnose it
further and do not work around it.

Allowlist changes I make take effect in new sessions only, never the current one.

## Commits

Commit after each numbered item in a task, separately, with the item number in the
message. Do not run the test suite between items — run it once at the end.

If a session is interrupted, the completed items must already be committed.

## Project state

docs/STATE.md must be true of main at all times. If a change you make would make any
line in it false, update that line in the same commit as the change. Do not defer it to
the end of the task and do not open a separate PR for it.

Recording a defect in the Known defects list is part of finding it, not a follow-up.

## Commands

Typecheck: `npx tsc --noEmit`
Tests: `npx vitest run --reporter=dot`
Frontend typecheck: `cd frontend && npm run typecheck`
Frontend test typecheck: `cd frontend && npm run typecheck:test`
Frontend tests: `cd frontend && npx vitest run --reporter=dot`
Frontend build: `cd frontend && npm run generate`
Deploy: `npm run deploy`, never bare `wrangler deploy` — the bare command loses the commit stamp and `/health` then reports `unknown`.

Run the typecheck before the test suite. It is far cheaper and catches most breakage.

Both frontend typecheck commands, not just the first. `npm run typecheck` is `nuxi
typecheck`, which compiles the program Nuxt generates and covers app code only —
`frontend/test` sits outside it, and vitest transpiles without checking. `typecheck:test`
is the one that compiles the fixtures, and the fixtures are what implement the worker's
wire shapes.

The root `Tests` and `Frontend tests` runs are not the same suite twice: the root
`vitest run` uses the root `vitest.config.ts`, whose `include` is `test/**/*.test.ts`, and
never sees `frontend/test`. The frontend has its own config and its own tests under
`frontend/test/**/*.spec.ts` — skip the second run and a frontend that renders wrong
strings still passes.

`Frontend build` is `nuxi generate` against an empty `NUXT_PUBLIC_WORKER_BASE`, checking
that the bundle compiles, not that it points anywhere. `npm run generate` runs it at the
default log level into `frontend/nuxt-generate.log` (gitignored by the root `*.log`) and
prints only the `WARN` and `ERROR` lines; a failing build dumps the whole log and exits
non-zero. `generate:loud` is the plain command when you want the build output itself.

The default log level is the point of it. `--logLevel=silent` would cut 70 lines to 21 on
its own, but it also swallows Vite's warn-level lines — an `import 'node:fs'` reaching the
client bundle warns at that level and exits 0, which is a build that passes here and
breaks in the browser.

Never run the full suite with the default reporter. Its output is several hundred lines
and lands in context in full. On failures, re-run only the failing file with the default
reporter.

## Exploration

Delegate codebase discovery to the Explore subagent before implementing anything that
touches more than two files.

Ask it for file paths, symbol names and line ranges. Never ask it to return file
contents. The point is that the files it reads do not enter this conversation.

Read a file directly only when you are about to edit it.

## Verification limits

*.pages.dev is reachable from the sandbox and a deployed frontend can be verified from
here. Verified 2026-08-06 against dn-gbp-audit.pages.dev.

**Chromium through the egress proxy resets on every host at TLS 1.3.** Launch it with
`--ssl-version-max=tls1.2` and it works, with certificate verification left on and the
proxy CA trusted — nothing needs to be disabled. Without the flag every navigation
fails as `net::ERR_CONNECTION_RESET`, on allowlisted hosts, which reads as a blocked
host rather than a TLS negotiation the gateway will not complete. curl is unaffected.
Verified 2026-08-06 against dn-gbp-audit.pages.dev and the Worker `/health`.

The deployed Worker on *.workers.dev may be unreachable. If it is, run against the
adapters locally and say so; do not report a Worker check that did not happen.

## Compaction

When compacting, always preserve: the list of files modified so far, any command that
failed with its exact error string, and any judgement call already reported.
