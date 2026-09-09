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
- **Email**: HTTP call to SendGrid's send API using an API key stored as a
  cloud-environment secret (never committed to the repo, never in the
  prompt). Single-sender verified address as "from". Recipient:
  glendanielcooney@gmail.com. This is a send-only credential with no
  relationship to Glen's Gmail account — Gmail is only ever the recipient.

### Run steps

1. Read `state/seen_listings.json`.
2. Search each source (see Sources below) for W113-compatible hardtop-only
   listings, using language-appropriate search terms per country.
3. For each candidate listing, extract: title, price + currency, town/
   country, listing URL, thumbnail image URL, condition notes, date posted
   if available. Discard anything that is clearly a full car, not a
   standalone hardtop.
4. Convert price to approximate GBP (note it's approximate).
5. Estimate straight-line distance from London (51.5074, -0.1278) to the
   listing's town.
6. Diff against `state/seen_listings.json` by listing URL: not present →
   "New"; present → "Seen before".
7. Build one HTML email: New listings first (full detail incl. thumbnail),
   then the rest sorted by distance ascending. Note any source that
   couldn't be checked this run.
8. Send via SendGrid.
9. Archive the same HTML to `runs/YYYY-MM-DD.html`.
10. Add any new listings to `state/seen_listings.json`; commit and push.
11. If zero listings found across all sources (e.g. total search failure),
    still send an email saying so rather than silently skipping — this is
    the signal something's broken.

## Sources (default set)

**General classifieds, one per country**: AutoScout24 (cross-EU), eBay
Kleinanzeigen (DE), Marktplaats (NL), LeBonCoin (FR), Subito (IT),
Coches.net / Wallapop (ES), willhaben (AT), StandVirtual (PT), Blocket
(SE), DBA (DK), Hasznaltauto (HU), eBay UK / Gumtree / PistonHeads (UK).

**W113 specialist**: sl113.org, Pagoda SL Group, Bring a Trailer, The MB
Market.

Search terms per country account for local phrasing (e.g. Hartschalendach/
Hardtop, toit rigide, capote rigida, kemény tető) plus "W113", "Pagoda",
"230SL", "250SL", "280SL".

## Known limitations

- **Best-effort coverage, not exhaustive.** This uses web search + page
  fetch, not a dedicated scraper per site. Sites that resist search
  indexing or block fetches (AutoScout24, Marktplaats and LeBonCoin can be
  aggressive about this) may be under-represented some weeks — the email
  will note when a source couldn't be checked.
- **Approximate GBP conversion and distance.** Good enough for at-a-glance
  triage, not exact.
- **Best-effort deduplication.** The same hardtop cross-posted to two
  sites may appear twice.

## Error handling

- A single source failing (blocked, timeout, no results) does not fail the
  whole run — it's noted in the email and the run continues.
- SendGrid send failure: retry once; if it still fails, the routine's run
  log will show the failure (visible via `RemoteTrigger` `get_run_log`) —
  no separate alerting for this v1.
- State file corruption/parse failure: treat as empty state for that run
  (everything shows as "New" once) rather than crashing the routine.

## Setup dependencies (one-time, outside this spec's automation)

1. GitHub repo created — done (`G2thaCizzo/pagoda-hardtop-agent`).
2. SendGrid account + single-sender verification — pending, Glen to do
   with guidance.
3. SendGrid API key added as a secret on the cloud environment used by the
   routine — pending.
4. The `RemoteTrigger` routine itself created (cron, prompt, repo source,
   env secret reference) — pending, part of the implementation plan.

## Testing

- After routine creation, trigger one manual run (`RemoteTrigger`
  `action: "run"`) before relying on the weekly cadence.
- Review the run log (`get_run_log`) for tool errors / permission denials.
- Confirm the email arrives, renders correctly (thumbnails load, links
  work), and correctly separates New vs by-distance sections.
- Confirm `state/seen_listings.json` and `runs/*.html` are committed and
  pushed after the run.
- Run a second manual trigger shortly after to confirm nothing from the
  first run gets re-flagged as "New".

## Files in this folder

- `CLAUDE.md` — project overview, constraints, file list.
- `docs/superpowers/specs/2026-09-09-hardtop-agent-design.md` — this file.
- `state/seen_listings.json` — persisted listing history (written by the
  cloud routine).
- `runs/` — archived copies of each week's email.
