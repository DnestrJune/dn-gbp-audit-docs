# DECISIONS

Append-only. Newest first. Each entry is sourced from this repository's git history,
code comments or `CLAUDE.md` — nothing here is reconstructed from memory.

## 2026-08-20 — The outreach note is optional at send time, and fillable once afterwards
**Decided:** `POST /outreach` accepts `sentAt` with no `note`. The note stays write-once and
may be filled later by a request carrying `auditId` and `note` alone, guarded in SQL as
`SET outreach_note = ? WHERE id = ? AND outreach_sent_at IS NOT NULL AND outreach_note IS
NULL` (`recordOutreachNoteStatement`, `src/worker/outreach.ts`). An empty or blank note
reads as no note; a second note is 409 `outreach_note_already_recorded`; a note with no
send recorded in front of it is 409 `no_outreach_recorded`.
**Because:** requiring a note made the common case unrecordable. Forty reports go out and
two are answered — the thirty-eight silent ones are exactly the rows that need recording at
send time, when there is nothing yet to write. Without them the audit row carries no trace
of which report went out, and that link is the whole reason the field exists:
`contact_state = 'contacted'` says a venue was written to, it cannot say WHICH SET OF FACTS
was put in front of them.
**The row is still write-once, per field.** Two `UPDATE audits` statements now exist and
they are the only ones; each can fill a NULL and do nothing else. Every field goes NULL →
value exactly once and can never be rewritten. What changed is only that the two fields
need not be written in the same request — which is what the real sequence looks like, since
a reply arrives days after a send.
**Rejected:** refusing the note after the initial write and recording the reply elsewhere.
It has nowhere to go. A new column or table is out of scope and would be a second home for
one fact; `contact_state` is per venue, four values and no text, so it cannot hold a reply;
and a new audit row would claim a report was sent that never was. Taking that option would
lose the report-to-reply link, which is the thing being protected. Also rejected: making
the note freely editable, which is the same as not keeping history at all.
**Reversal condition:** a reply that itself needs revising — a correction, or a second
message in the same exchange. That is a log, not a field, and it needs a table with one row
per message rather than a looser guard on this one.

## 2026-08-20 — Record outreach in two places: state on the venue, the exchange on the run
**Decided:** two additive migrations. `nearby_venues.contact_state` +
`contact_state_at` (migrations/0012) hold ONE MUTABLE VALUE PER VENUE from a closed
vocabulary — `contacted | replied | declined | client`, with NULL meaning no contact has
been recorded. `audits.outreach_sent_at` + `outreach_note` (migrations/0013) hold ONE
WRITE-ONCE RECORD PER RUN: when the report went out and, in free text, what was sent and
what came back. `POST /outreach` (`src/worker/outreach.ts`, handler in
`src/worker/index.ts`) writes both in one batch behind the existing `isAuthorised` gate.
**Because:** the two facts answer different questions and neither can answer the other's.
"Who have I not written to yet" has one answer per venue, and held on `audits` it would
have N answers that every read has to reduce across — a query that gets it quietly wrong.
"Which report earned that reply" has one answer per run, and held on the venue it would be
overwritten by the next report. 19 of 70 audited venues have more than one audit today.
**The state set is four values and no more.** `replied` and `declined` do not overlap: the
boundary is whether anything is left to do. There is deliberately no "no reply" or "gone
cold" state — nobody can say how long silence lasts before it becomes one, and a state
whose boundary cannot be drawn returns a population nobody can describe. `contacted`
covers it; how long it has been is `contact_state_at` subtracted from today, which is a
figure rather than a verdict. It is not a state machine: any value may follow any other,
because a venue that declined in March and writes back in September has to be recordable.
**No state on the audit row.** A per-run state would be a second vocabulary answering the
same question one table over, and the only way to keep two in step is to derive one from
the other — at which point one is not a stored fact. The note carries the reply in the
operator's own words, which is more than an enum can hold, and the facts that went out are
already on the same row in `result_json`.
**They may disagree, and A wins.** `contact_state` is authoritative for "where does this
stand"; the audit's note is authoritative for "what happened on that run". The one
derivation is a FLOOR: recording a send sets `contact_state` to `contacted` only where it
is NULL (`WHERE ... AND contact_state IS NULL`), so it can fill an absence and can never
walk a `client` back. An explicit state in the same request wins over the floor, and
clearing a state is allowed even where an audit carries a send — an operator correcting the
column outranks a value inferred from a timestamp. Enforced in `handleOutreach`, in one
batch, with every refusal checked before the batch is applied.
**`audits` stays write-once per field.** This adds the only `UPDATE audits` in `src/`, and
it is guarded on `outreach_sent_at IS NULL`: every column the pipeline writes is still
unreachable by any update, and these two go NULL → value exactly once. A second attempt is
409, not a revision — the stored record is a true statement about a message that really was
sent. A venue re-contacted with a fresh report is a new audit row carrying its own record.
**Note cap: 4000 characters**, counted in code points after trimming. A pasted reply of
several paragraphs is around 1500; the bound exists so one request cannot make an audit row
large, and it sits far under any D1 value limit.
**Rejected:** an audit-lead join table. It is the correct model for anything that belongs
to an audit-lead PAIR — the 2026-08-19 entry below says so about `distance_m` — and it is
out of scope here. Nothing in this change needs it: the venue state is per venue by
design, and the outreach record is per audit, so neither fact is a pair. Also rejected: a
per-run state field; deriving the venue state from the audit rows on read; a fifth
`bounced` state; and any count or aggregate of outcomes, which is a decision to make
against real rows rather than against zero of them.
**ENGINE_VERSION: unchanged at 3.** The bump rule is that a stored record would render a
FALSE STATEMENT about the venue under the new code. Nothing here reaches the engine: no
input, no check, no metric, no field of `result_json`, and no report rendering. Every
stored record renders exactly the page it rendered before. A bump would take all 111 stored
audits dark to record a fact about our own outreach, which is the trade the rule exists to
refuse.
**Reversal condition:** the moment a question needs the state of a venue AS AT a particular
audit — "did the venues that were already clients get a different reply rate" — the state
belongs on the pair and this shape cannot answer it.

## 2026-08-19 — Stop the `distance_m` overwrite without moving distance off the lead row
**Decided:** `upsertNearbyStatements` now writes
`distance_m = COALESCE(distance_m, excluded.distance_m)` (`src/worker/leads.ts`), so the
first audit to see a lead sets its distance and later sightings only backfill a row that
has none.
**Because:** distance is measured from the seed of whichever audit ran the nearby search
(`src/adapters/placesNearby.ts`, `haversineMetres` from the seed), so a second audit's
figure is a distance from a different origin rather than a newer reading of the same one.
Taking it unconditionally destroyed the earlier value with nothing left to recover it
from. The column stays where it is: distance properly belongs to an audit-lead PAIR, not
to a lead, and one column on `nearby_venues` cannot hold one figure per audit.
**Rejected:** moving distance to a per-audit-lead table, which is the correct model and
was ruled out of scope for this change — it needs a table, a migration and a decision
about which distance the leads list sorts on.
**Reversal condition:** when distance ordering has to be correct across audits run from
different origins. First-writer-wins is right only while the list is read one
neighbourhood at a time.

## 2026-08-14 — Remove `establishment` from `categoryOnlyTokens` without bumping ENGINE_VERSION
**Decided:** dropped `establishment` from the category vocabulary, and left
`ENGINE_VERSION` at 2. Source: `f1db46e`, `a450738`, and the measurement written into
`src/engine/version.ts`.
**Because:** `establishment` is Google's universal Places base type, so
`foodServiceVerdict` answered `'yes'` for nearly every venue and the food-service gate on
`hoursImplausible` suppressed nothing. Blast radius was measured before the change: of 30
records then rendering, exactly two carried `hoursImplausible`, and at most one of those
could have changed meaning.
**Rejected:** bumping to 3, which would have taken all 30 rendering records dark to
retire a flag on one.
**Reversal condition:** a later measurement showing a materially larger share of
rendering records whose flag meaning moved.

## 2026-08-13 — Lower `DEFAULT_MAX_REVIEWS` from 200 to 90
**Decided:** the review fetch cap is 90. Source: `b37ca34` and the comment at
`src/config/audit.ts:9-30`.
**Because:** the cap is a spend ceiling, not a sample size — a venue with more than 90
reviews in the 180-day window is active enough that this tool has nothing useful to tell
its owner, so a run that hits the ceiling was not worth making. Measured before the
change: 5 of the 92 audits stored at the time carried `reviewsConsidered > 90`, and three
of those were already truncated at 200.
**Rejected:** keeping 200 to widen the window on busy venues; reviews are the dominant
per-audit cost and the extra spend buys reports that are not fit to send.
**Reversal condition:** the target segment shifting to venues at that review volume.

## 2026-08-08 — Invert the `actionLinkPresent` applicability gate, and hold ENGINE_VERSION at 2
**Decided:** the gate was inverted and the constant did not move. Source: `d6909b6`,
`95dda0e`, and the reasoning recorded in `src/engine/version.ts`.
**Because:** this is the case that fixed the bump rule as "render compatibility, not
arithmetic agreement". A record stored under 2 renders row for row after the change; what
is out of date is the fix it lists, which is stale rather than false about the venue.
**Rejected:** bumping to 3 — it would have darkened every stored record to retire a stale
suggestion in four of them.
**Reversal condition:** none. The rule it established is now the bump condition.

## 2026-08-08 — Bump ENGINE_VERSION to 2 for website classification
**Decided:** `websitePresent` reads `absent` when the field holds a social or
link-in-bio page, and the constant moved to 2. Source: `284a84c`, migration
`0007_website_classification.sql`, `src/engine/version.ts`.
**Because:** the same venue's completeness count and percentage differ before and after
with nothing about the venue having changed, and an Instagram link shown as "website
present" is untrue about the venue every time it is read.
**Rejected:** rendering version-1 records under the new rule, which would have kept
showing that untrue statement.
**Reversal condition:** none.

## 2026-07-29 — Peer group is matched on the searched type, not on strict `primaryType` equality
**Decided:** peers are the venues Google returns for the searched type. Source:
`b680aba` and the measurement table in `README.md` under "Searched-type matching is the
decision".
**Because:** strict `primaryType` equality was measured on all three acceptance venues
and moved the rating median by noise while collapsing one sample to the minimum.
**Rejected:** strict equality, as above; an equivalence set (`cafe` ≈ `coffee_shop`) was
deferred rather than rejected.
**Reversal condition:** real outreach showing owners disputing their peer list.

## 2026-07-29 — The frontend's Worker base URL has an empty default, not a fallback
**Decided:** `runtimeConfig.public.workerBase` defaults to `''` and an unset
`NUXT_PUBLIC_WORKER_BASE` throws on screen. Source: `7495279` and the comment at
`frontend/nuxt.config.ts:49-58`.
**Because:** a baked-in `workers.dev` fallback would let a misconfigured deploy look like
it works while pointing at somebody else's Worker.
**Rejected:** hardcoding the production Worker URL as the default.
**Reversal condition:** none.
