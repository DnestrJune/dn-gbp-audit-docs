# OPERATIONS

Reference for doing things correctly: bootstrapping the Worker, deploying the
frontend, the CORS configuration, and the `GET /leads` contract in full. This
goes stale only when the setup changes — it is not a record of what was
decided (`docs/DECISIONS.md`) or of where the project stands right now
(`docs/STATE.md`).

## Bootstrapping the Worker

`AUDIT_KEY`, `GOOGLE_PLACES_KEY` and `APIFY_TOKEN` are set with `wrangler
secret put` and appear nowhere in the repo or in `wrangler.toml`.

```sh
wrangler d1 create dn-gbp-audit     # put the printed id into wrangler.toml
wrangler d1 migrations apply dn-gbp-audit --remote
wrangler secret put AUDIT_KEY
wrangler secret put GOOGLE_PLACES_KEY
wrangler secret put APIFY_TOKEN
npm run deploy
```

`npm run deploy`, never bare `wrangler deploy` — the bare command loses the
`--define COMMIT_SHA` stamp `package.json`'s `deploy` script passes, and
`GET /health` then reports `commit: "unknown"`.

Deployed at `https://dn-gbp-audit.dnestrjune.workers.dev`.

### curl examples

```sh
curl -X POST https://dn-gbp-audit.dnestrjune.workers.dev/resolve \
  -H "x-audit-key: $AUDIT_KEY" -H 'content-type: application/json' \
  -d '{"query":"Mugshot coffee Bucuresti"}'

curl -X POST https://dn-gbp-audit.dnestrjune.workers.dev/audit \
  -H "x-audit-key: $AUDIT_KEY" -H 'content-type: application/json' \
  -d '{"placeId":"ChIJkzjG3Pf_sUAR2bIJbUU04oo","radiusM":1200}'

curl -H "x-audit-key: $AUDIT_KEY" \
  'https://dn-gbp-audit.dnestrjune.workers.dev/leads?has_website=false&audited=false'

curl -H "x-audit-key: $AUDIT_KEY" \
  https://dn-gbp-audit.dnestrjune.workers.dev/leads/stats
```

## Deploying the frontend

Cloudflare Pages, a separate project from the Worker, named `dn-gbp-audit` —
Cloudflare derives the production hostname from the project name, so a
project created under a different name fails every browser request on CORS
while curl keeps working (see "ALLOWED_ORIGIN" below).

| setting | value |
| --- | --- |
| root directory | `frontend` |
| build command | `npm install && npx nuxi generate` |
| output directory | `dist` — **not** `.output/public` |
| `NODE_VERSION` | `22` |
| `NUXT_PUBLIC_WORKER_BASE` | `https://dn-gbp-audit.dnestrjune.workers.dev` |

`dist` rather than `.output/public` because the Nuxt preset is
`cloudflare-pages-static`.

### `wrangler pages deploy` fails from a build environment

The Cloudflare API token available in a build environment carries Workers and
D1 permissions and no Pages permission, so `wrangler pages deploy` returns
`Authentication error [code: 10000]` before uploading anything.

That is only a CLI limitation, not a blocker: the repo is connected to the
`dn-gbp-audit` Pages project through the dashboard integration, so a push
builds and deploys a branch preview on its own, with a `*.pages.dev` preview
URL posted on the pull request. Use the dashboard integration as the deploy
path from a build environment; do not try to route around the missing Pages
permission by running `wrangler pages deploy` there.

## `ALLOWED_ORIGIN`

`src/worker/cors.ts`. Comma-separated in `wrangler.toml [vars]`. Entries are
either exact origins or a single-label wildcard:

```
ALLOWED_ORIGIN = "https://dn-gbp-audit.pages.dev, https://*.dn-gbp-audit.pages.dev"
```

The wildcard exists because Cloudflare Pages mints a fresh origin for every
deployment and the hash changes on every push — an exact list has to be
edited after each one, and when it is not the failure is a silent CORS block
that reads as a broken app rather than a stale config.

**The wildcard is bounded by the namespace, not by the syntax.** Cloudflare
issues `*.dn-gbp-audit.pages.dev` to deployments of this project and to
nothing else, so a hostname in it cannot be obtained without the account. The
same pattern over a namespace anyone can register in would be worthless. The
match requires the scheme, forbids a port, and allows exactly one label; it
is a suffix rule and never a prefix one, because
`https://evil-dn-gbp-audit.pages.dev` is a different site under a domain
anyone can buy. All of `https://dn-gbp-audit.pages.dev.attacker.com`,
`http://…`, `…:8443`, `https://a.b.dn-gbp-audit.pages.dev` and `null` are
refused.

Three states, not two:

| `ALLOWED_ORIGIN` | behaviour |
| --- | --- |
| unset or blank | `*`. The pre-configuration state; the boundary is the shared key, not the origin, so failing closed on a missing var would trade a real outage for an imaginary breach. |
| set, with usable rules | those origins, echoed |
| set, but no rule survives parsing | **every origin refused.** A typo must not read as "allow everything" — that is the widest possible interpretation of a mistake. |

`ALLOWED_ORIGIN` must match the Pages project name — see the frontend deploy
section above.

## `GET /leads`

`src/worker/leads.ts`, `src/config/audit.ts`. All parameters optional;
absent means no clause, never a substituted default.

```
GET /leads?min_reviews=&max_reviews=&min_rating=&max_rating=
          &has_website=&website_type=&primary_type=
          &contact_state=&audited=&include_closed=&sort=&limit=
GET /leads/stats?include_closed=
```

| parameter | meaning |
| --- | --- |
| `min_reviews`, `max_reviews` | review-count bounds. See "min_reviews counts null as 0" below. |
| `min_rating`, `max_rating` | rating bounds. See "rating bound excludes unrated venues" below. |
| `has_website` | `true`/`1`/`yes` or `false`/`0`/`no`; anything else reads as absent, never as `false`. |
| `website_type` | `site`, `social` or `none`. See "website_type NULL semantics" below. |
| `primary_type` | exact match against the stored `primary_type`. |
| `contact_state` | one of the recorded outreach states, or `not_contacted` for rows with none recorded (`contact_state IS NULL`). |
| `audited` | whether an `audits` row exists for the venue. |
| `include_closed` | default `false`. See "include_closed" below. |
| `sort` | `distance` (default), `reviews_asc` or `reviews_desc`. An unrecognised value is a 400, not a silent fallback — a list ordered by something other than what was asked for looks exactly like a correct answer. |
| `limit` | default `LEAD_DEFAULT_LIMIT` (500), clamped to `LEAD_MAX_LIMIT` (1000). |

### `LEAD_MIN_REVIEWS` / `LEAD_MAX_REVIEWS`, and why

Defined in `src/config/audit.ts`: `LEAD_MIN_REVIEWS = 0`, `LEAD_MAX_REVIEWS =
800`. Both are guesses to recalibrate after real outreach, and neither
affects what is stored — they are **not** defaults substituted onto
`GET /leads` when a bound is omitted (that was tried and reverted: it meant a
caller asking for every un-audited lead was silently handed the subset under
800 reviews, with no sign a ceiling had been applied). `GET /leads/stats`
uses them to report `withinDefaultThresholds`, the count of stored rows the
current guess would keep, so the guess can be judged.

Above the ceiling a venue is established, self-sustaining and usually already
has an agency — the pitch competes with an incumbent rather than filling a
gap. There is no lower bound: a very low review count means either a venue
that just opened (budget, no incumbent — the best segment) or a dormant
listing (no budget — the worst), and `searchNearby` exposes nothing that
separates them; filtering there would discard the best leads with the worst.

### `website_type` NULL semantics

`site`, `social` or `none` sits **beside** `has_website` rather than
replacing it. The flag answers whether the field is filled; the column
answers what is in it — `has_website=1` with `website_type=social` is a venue
whose website field holds an Instagram or Linktree page, which `has_website`
alone can never select.

It is **not backfilled**: only the boolean was ever stored for a nearby venue
discovered before the classifier existed, so there is no URL on disk to
classify. The upsert refreshes it on every sighting, so rows classify
themselves as they reappear. A `NULL` means "not seen since the column
existed" and satisfies none of the three filter values — a query for any one
of `site`, `social` or `none` excludes it. Never read a `NULL` row as "no
website".

### `min_reviews` counts a null review count as 0

Places omits `userRatingCount` for an unrated venue, so a null
`user_rating_count` means zero reviews — a real observation, and exactly the
"just opened" case the floor of 0 exists to keep. The SQL is
`COALESCE(n.user_rating_count, 0) >= ?` / `<= ?`, so a bound applied without
this would silently drop every unrated venue out of a bounded result.

### The rating bound excludes unrated venues

Opposite of the review-count case: `NULL` means unrated, and an unrated venue
cannot satisfy "at least 4.0" — there is no rating to compare, so the SQL is
a direct `n.rating >= ?` / `<= ?` with no `COALESCE`. Omitting the bound
keeps unrated venues in the result; supplying one excludes them.

### `include_closed`, and the closed-listing argument

Non-operational venues are hidden by default (`business_status IS NULL OR
business_status = 'operational'`), and `include_closed=true` shows them. The
default is right for the common case — most closed listings are dead rows.

The exception is why the switch exists: a venue Google marks closed is
*either* a dead listing *or* an operating business whose profile is telling
every searcher it shut down, losing essentially all discovery traffic,
usually without the owner knowing. That second case is the most valuable
lead the table can produce and the most urgent finding an audit has — it is
why `listingClosed` is both a gate and a share fact in the engine. Nothing in
a nearby search separates the two; only a human looking at the venue can.
`include_closed` widens the population and bypasses nothing else — every
other threshold still applies, and `business_status` is on each row.

`GET /leads/stats` takes the same flag and moves its buckets with it, so the
counts never describe a different population than the rows. `storedTotal`
and `closed` are unfiltered either way.

## Benchmark exclusions

`buildBenchmark` (`src/adapters/placesNearby.ts:519-557`) excludes a nearby venue from the sample for five reasons: `self` (the audited venue, at line 533), `notOperational` (537), `noRating` (541), `noReviewCount` (545), and `fewReviews` (556, below `MIN_PEER_REVIEWS` — the only one of the five kept in the peer list, marked `inSample: false`, rather than dropped outright).
