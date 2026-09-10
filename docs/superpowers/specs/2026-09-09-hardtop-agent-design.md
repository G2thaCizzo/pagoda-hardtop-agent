# Pagoda Hardtop Watch — Design Spec

Date: 2026-09-09
Status: Approved, ready for implementation plan

## Purpose

Glen owns a Mercedes-Benz 230SL "Pagoda" (W113) and wants a spare/replacement
hardtop. Hardtops from the W113 range (230SL, 250SL, 280SL) are physically
interchangeable. This system searches UK and European classifieds/forums
weekly for hardtop-only listings (no full cars), and emails Glen a summary —
new listings highlighted, the rest sorted by distance from London.

## Non-goals

- Full car listings (explicitly excluded, even if a "parts car" mentions a
  hardtop)
- Automated purchasing or contacting sellers
- Perfect/complete coverage — see Known Limitations below

## Architecture

One repo, two roles:

- **Local project folder**: `pagoda\Hard Top Agent\` — the workspace's
  standard per-project home (this file, CLAUDE.md, etc.)
- **State store for the cloud routine**: the same repo, cloned fresh each
  week by the scheduled cloud agent, which reads/writes `state/` and
  `runs/`, then commits and pushes.

Repo: `https://github.com/G2thaCizzo/pagoda-hardtop-agent` (private)

```
pagoda-hardtop-agent/
  CLAUDE.md
  docs/superpowers/specs/2026-09-09-hardtop-agent-design.md   (this file)
  state/seen_listings.json   — every listing ever seen (URL, title, price,
                                currency, location, thumbnail, first-seen date)
  runs/YYYY-MM-DD.html       — archived copy of each week's email
```

## Weekly cloud routine

- **Trigger**: cron, Mondays 06:00 UTC (07:00 Europe/London)
- **Repo source**: the repo above
- **Tools**: WebSearch, WebFetch, Bash, Read, Write, Edit, Glob, Grep
- **Email**: sent via a connected Resend MCP connector (claude.ai
  connector, attached to the routine's `mcp_connections`) — no API key
  ever handled by the assistant, no raw HTTP call, no credential in any
  file or prompt. Sends from Resend's own pre-authenticated
  `onboarding@resend.dev` address — no custom domain, DKIM or DMARC setup
  needed, and no spam-filtering risk from an unauthenticated "from"
  address (the problem hit when trying both SendGrid and Brevo without
  owning a domain). This address can only deliver to the email used to
  sign up for Resend, which is exactly glendanielcooney@gmail.com — the
  one recipient this system needs. The connector's "Send Email" tool
  permission is set to Allow (auto-approved) since the routine runs
  unattended; every other tool on the connector is left on "Needs
  approval" or denied.

### Run steps

1. Read `state/seen_listings.json`.
2. Search each source (see Sources below) for W113-compatible hardtop-only
   listings, using `site:`-scoped WebSearch queries with language-
   appropriate terms per country. **WebFetch and raw HTTP calls to
   external domains are blocked in the cloud sandbox** (confirmed during
   testing — even Wikipedia/Google failed; this is a blanket egress
   restriction, not per-source) — only WebSearch works, so extraction is
   snippet-only (see step 3), not page-verified.
3. For each candidate listing, extract from the WebSearch snippet alone:
   title, price + currency (if shown), town/country (if shown), listing
   URL, condition notes (if the snippet mentions any). No thumbnail, no
   posted date — not available without fetching the page. Discard
   anything that is a full car (not a standalone hardtop) or a hardtop
   accessory (cover, stand, trim, hardware) rather than the roof itself.
   Every listing is unverified as still active, since its page can't be
   opened to check.
4. Convert price to approximate GBP where a price was found (note it's
   approximate).
5. Estimate straight-line distance from London (51.5074, -0.1278) to the
   listing's town, where a town was found.
6. Diff against `state/seen_listings.json` by listing URL: not present →
   "New"; present → "Seen before".
7. Build one HTML email: New listings first (title, price/location where
   known, distance where known, condition notes where known), then the
   rest sorted by distance ascending (unknown-distance listings grouped
   last), then a **Leads** section (see below). Note any source that
   returned nothing usable this run, and that all listings are unverified
   as still active.
8. Send via Resend.
9. Archive the same HTML to `runs/YYYY-MM-DD.html`.
10. Add any new listings to `state/seen_listings.json`; commit and push.
11. If zero listings found across all sources (e.g. total search failure),
    still send an email saying so rather than silently skipping — this is
    the signal something's broken.

## Sources (default set)

**General classifieds, one per country**: AutoScout24 (cross-EU), eBay
Kleinanzeigen (DE), Marktplaats (NL), LeBonCoin (FR), Subito (IT),
Coches.net / Wallapop / Milanuncios (ES), willhaben (AT), StandVirtual
(PT), Blocket (SE), DBA (DK), Hasznaltauto (HU), eBay UK / Gumtree /
PistonHeads (UK).

**W113 specialist**: sl113.org, Pagoda SL Group, Bring a Trailer, The MB
Market.

Search terms per country account for local phrasing plus "W113",
"Pagoda"/"Pagode", "230SL", "250SL", "280SL". **Query construction matters
more than translation accuracy**: testing (2026-09-10) found `site:`-
scoped queries return only thin title/URL lists, while a plain natural-
language query naming the domain plus a local "for sale" phrase (e.g.
"à vendre", "te koop", "zu verkaufen") gets WebSearch's own summarization
to surface real price and location pulled from the result pages —
sometimes listings the `site:` form missed entirely (confirmed: a €5,000
and a €3,500 standalone hardtop on LeBonCoin, a €2,000/€2,500 hardtop on
Kleinanzeigen, a €2,500 hardtop on Marktplaats). "Hardtop"/"hard top" as a
loanword outperformed translated jargon like "Hartschalendach" in testing
— kept as a fallback term, not primary.

## Leads (category-page fallback)

Confirmed necessary in testing (2026-09-10, second query-rewrite run):
even with the improved query construction above, WebSearch sometimes
clearly describes a specific real listing in its summary text but only
returns a category/search-results page as the URL for that source
(search engines tend to rank a site's category page above a specific
long-tail classified ad). Rather than discard this signal or fabricate an
individual URL, these are captured separately as **leads**:
`{description, category_url, source}`. They get their own "Possible leads
(no direct link found)" section in the email, visually distinct from
confirmed listings, and are **never** diffed against
`state/seen_listings.json` or tracked for New/Seen status — a category
page's URL is stable but what it currently points to isn't, so "seen
before" has no meaning for it. Every lead found is shown every run.

## Known limitations

- **Search-only, no page verification — this is a hard platform
  constraint, not a design choice.** The cloud routine's sandbox blocks
  all outbound web fetches (WebFetch, curl) to external domains; only
  WebSearch works, because it's proxied through Anthropic's own
  infrastructure. Discovered during the first test run (2026-09-10), which
  found curl failing against every domain tested, including unrelated
  ones like Wikipedia — confirming it's a blanket environment restriction,
  not something fixable per-source. Every listing in the email is
  therefore found-via-search and **unverified as still for sale** — no way
  to confirm the page is live, that the item hasn't sold, or to read full
  condition detail beyond what the search snippet shows.
- **No thumbnail images, ever — confirmed technically impossible in this
  setup, not an oversight.** WebSearch never returns image URLs (tested
  directly), and page-fetching to get one is blocked by the same sandbox
  restriction above. `thumbnail` is always `null`.
- **Price, location, condition and posted-date are inconsistently
  available.** Query construction (see Sources above) substantially
  affects this — natural-language queries surface real price/location far
  more often than `site:`-scoped ones did in initial testing — but
  coverage still depends on what WebSearch's summary happens to mention.
  Posted-dates aren't reliably available from search results at all and
  are always `null`.
- **Best-effort coverage, not exhaustive.** Sites that resist search
  indexing may be under-represented some weeks — the email notes when a
  source returned nothing usable.
- **Approximate GBP conversion and distance.** Good enough for at-a-glance
  triage, not exact — and distance is only computed when a location was
  found.
- **Best-effort deduplication.** The same hardtop cross-posted to two
  sites may appear twice.

## Error handling

- A single source failing (blocked, timeout, no results) does not fail the
  whole run — it's noted in the email and the run continues.
- Resend send failure: retry once; if it still fails, the routine's run
  log will show the failure (visible via `RemoteTrigger` `get_run_log`) —
  no separate alerting for this v1.
- State file corruption/parse failure: treat as empty state for that run
  (everything shows as "New" once) rather than crashing the routine.

## Setup dependencies (one-time, outside this spec's automation)

1. GitHub repo created — done (`G2thaCizzo/pagoda-hardtop-agent`).
2. Resend account (signed up with glendanielcooney@gmail.com) — done.
3. Resend connector connected at claude.ai/customize/connectors, "Send
   Email" tool permission set to Allow — done. (An earlier attempt used a
   raw API key stored as a cloud-environment secret, but that mechanism
   turned out not to exist — no such settings page/field is actually
   exposed by the routine tooling. The connector is the real, working
   equivalent: no key is ever handled by the assistant or stored in any
   file.)
4. The `RemoteTrigger` routine itself created (cron, prompt, repo source,
   Resend connector attached via `mcp_connections`) — done
   (`trig_016X2UagUegJcG4Lid62r8Ez`).

## Testing

- After routine creation, trigger one manual run (`RemoteTrigger`
  `action: "run"`) before relying on the weekly cadence.
- Review the run log (`get_run_log`) for tool errors / permission denials.
- Confirm the email arrives and correctly separates New vs by-distance
  sections.
- Confirm `state/seen_listings.json` and `runs/*.html` are committed and
  pushed after the run.
- Run a second manual trigger shortly after to confirm nothing from the
  first run gets re-flagged as "New".

**First test run (2026-09-10):** email send via the Resend connector
worked correctly (no permission issues, delivered successfully). However,
it surfaced the sandbox network-egress constraint described under Known
limitations — WebFetch/curl are blocked entirely, so the run correctly
found zero *verifiable* listings and sent the designed "something's
broken" email rather than fabricate data. The prompt was reworked
afterward to extract from WebSearch snippets directly (no page fetch) —
that revision still needs its own manual test run to confirm it actually
surfaces usable candidates within this constraint.

## Files in this folder

- `CLAUDE.md` — project overview, constraints, file list.
- `docs/superpowers/specs/2026-09-09-hardtop-agent-design.md` — this file.
- `state/seen_listings.json` — persisted listing history (written by the
  cloud routine).
- `runs/` — archived copies of each week's email.
