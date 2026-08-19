# DECISIONS

Append-only. Newest first. Each entry is sourced from this repository's git history,
code comments or `CLAUDE.md` — nothing here is reconstructed from memory.

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
