# Weekly Pagoda Hardtop search — cloud routine prompt

This is the literal text passed as `events[].data.message.content` when the
`pagoda-hardtop-watch` RemoteTrigger routine is created (Task 4 of the
implementation plan). It is versioned here so the logic is reviewable and
editable without digging through routine config. If you change this file,
update the live routine to match (via `RemoteTrigger` `action: "update"`).

---

You are running the weekly "Pagoda Hardtop Watch" search. You have a fresh
clone of this repo (`G2thaCizzo/pagoda-hardtop-agent`) checked out. You have
Bash, Read, Write, Edit, Glob, Grep, WebSearch and WebFetch. Work entirely
within this repo checkout. Do not ask the user anything — this run must
complete unattended and end with an email sent and the repo updated.

## 1. Load state

Read `state/seen_listings.json`. It is a JSON array of objects:
`{"url": str, "title": str, "price": number|null, "currency": str|null,
"location": str, "first_seen": "YYYY-MM-DD"}`. If the file is missing,
empty, or fails to parse, treat it as `[]` and continue (do not fail the
run).

## 2. Search sources

Find standalone Mercedes-Benz W113 "Pagoda" hardtops for sale — compatible
with the 230SL, 250SL and 280SL (the hardtop shape is shared across the
range). **Hardtop only, and the actual roof shell itself.** Discard:
- anything that is a full car for sale, even if the listing text mentions
  the hardtop is original/included — this search is not for cars.
- accessories and parts that are not the hardtop shell itself: covers,
  storage bags, carts/stands, trim (wood or chrome), hinges, seals,
  hardware kits, model/toy hardtops, or hardtop-shaped decor items. Only
  the actual roof.

**Known sandbox constraint — read before searching:** this environment's
outbound network access blocks WebFetch and any direct HTTP call
(confirmed: even fetching Wikipedia or Google directly fails here) — only
`WebSearch` works, because it's proxied through Anthropic's own
infrastructure rather than this sandbox's direct internet access. This
means you **cannot** open a listing page to confirm it's still active,
verify price, or read full condition details — you can only work from
what WebSearch's result snippets show you (title, URL, and a short text
excerpt). Do not attempt WebFetch or curl against classifieds sites —
it will fail every time; don't waste turns re-testing this.

For each source below, run a WebSearch using the `site:` operator shown
(this biases results toward that actual domain rather than generic SEO/
retailer noise) combined with "W113 hardtop" or "Pagoda hardtop" and the
local-language term. Try each source; if a source's search returns
nothing useful, move on and note it in the "sources not checked" list for
the email — do not let one source's failure stop the run.

| Country | Source(s) | Example WebSearch query |
|---|---|---|
| UK | eBay UK, Gumtree, PistonHeads classifieds | `site:ebay.co.uk W113 Pagoda hardtop`, `site:gumtree.com Mercedes Pagoda hardtop`, `site:pistonheads.com W113 hardtop` |
| Germany | eBay Kleinanzeigen | `site:kleinanzeigen.de W113 Hartschalendach Pagode` |
| France | LeBonCoin | `site:leboncoin.fr W113 Pagode toit rigide` |
| Netherlands | Marktplaats | `site:marktplaats.nl W113 Pagode hardtop kap` |
| Belgium | 2ememain, AutoScout24.be | `site:2ememain.be W113 hardtop`, `site:autoscout24.be W113 hardtop` |
| Spain | Coches.net, Wallapop | `site:coches.net W113 techo rígido Pagoda`, `site:wallapop.com W113 capota dura` |
| Italy | Subito.it | `site:subito.it W113 capote rigida Pagoda` |
| Austria | willhaben | `site:willhaben.at W113 Hardtop Pagode` |
| Portugal | StandVirtual.pt | `site:standvirtual.com W113 capota rígida hardtop` |
| Sweden | Blocket | `site:blocket.se W113 Pagoda hardtop` |
| Denmark | DBA.dk | `site:dba.dk W113 Pagoda hardtop` |
| Hungary | Hasznaltauto.hu | `site:hasznaltauto.hu W113 Pagoda kemény tető` |
| Cross-EU / specialist | AutoScout24.com, sl113.org classifieds/forum, Bring a Trailer, The MB Market | `site:autoscout24.com W113 hardtop`, `site:sl113.org hardtop for sale`, `site:bringatrailer.com W113 hardtop`, `site:thembmarket.com W113 hardtop` |

For every candidate result whose URL is actually on the target domain (not
a retailer/parts/SEO page unrelated to that specific source), extract from
the WebSearch snippet alone:
- `title` (from the search result)
- `price` and `currency` if visible in the snippet text (as shown — may be
  EUR, GBP, SEK, DKK, HUF etc.); `null` if not shown in the snippet
- `location` (town/region + country) if visible in the snippet; `null` if
  not shown
- `url` (the canonical listing URL)
- `thumbnail`: `null` — not available without fetching the page, do not
  guess an image URL
- `condition_notes`: whatever the snippet text says about condition, or
  `null` if the snippet doesn't mention it — do not invent detail beyond
  what the snippet actually shows
- `posted_date`: `null` — not reliably available from a snippet

Every listing is **unverified** — found via search, not confirmed still
for sale, since the page itself can't be opened. Say this plainly in the
email (see step 6) rather than implying these are live-checked.

## 3. Convert price to approximate GBP

Do one WebSearch for the current approximate EUR→GBP, SEK→GBP, DKK→GBP,
HUF→GBP rates (whichever currencies you actually encountered this run) and
apply them. Round to the nearest £10. Label this as "approx." in the email
— do not imply it's a live/precise conversion.

## 4. Estimate distance from London

London is fixed at 51.5074° N, -0.1278° E. For each listing with a known
location, use your own knowledge of that town's approximate coordinates
(or one WebSearch if you're not confident) and compute great-circle
distance in miles using the haversine formula. This is for rough sorting,
not precision — a nearest-10-mile estimate is fine. If `location` is
`null`, skip distance entirely for that listing (see step 6 for where it
goes in the email).

## 5. Diff against state

A listing is **New** if its `url` is not present in the state array loaded
in step 1. Otherwise it's **Seen before** — still include it in the email
body (in the distance-sorted section), just not flagged as new.

## 6. Build the email

One HTML email, structure:

1. Subject: `Pagoda Hardtop Watch — <today's date> — <N> new`
2. Short header line: date, total listings found, how many new, list any
   sources you couldn't check this run (e.g. "no usable results this run"),
   and a one-line note that all listings are found-via-search and
   unverified as still active (no page fetch is possible in this
   environment — see step 2).
3. **New listings** section first (if any): each as a block with title
   (linked to the listing URL), price (original + approx GBP, or "not
   shown" if the snippet didn't include one), location (or "not shown"),
   estimated distance from London (only if location is known), and
   condition notes (or omit the line entirely if null). No thumbnail —
   not available from a search snippet.
4. **All other listings** section: same block format, sorted by estimated
   distance ascending (listings with no known location go last, grouped
   under "distance unknown").
5. If literally zero listings were found across every source, still send
   the email — header should say so plainly (e.g. "No listings found this
   run — this may mean sources are blocked; check the run log"). This is
   the signal something is broken, so it must not be silently skipped.

## 7. Send the email

You have a connected Resend MCP tool available (attached to this routine
as a connector — do not use curl, do not look for any API key in the
environment; authentication is handled entirely by the connector). Call
its email-sending tool (action name `send-email`) with:

- `to`: `["glendanielcooney@gmail.com"]`
- `from`: `"Pagoda Hardtop Watch <onboarding@resend.dev>"`
- `subject`: SUBJECT_HERE
- `html`: HTML_BODY_HERE
- `text`: a short plain-text fallback with the same content (required —
  the tool rejects `html` without it)

`onboarding@resend.dev` is Resend's own pre-authenticated sending address
— do not change it, and do not attempt to verify a custom domain (not
needed here). Do not set `cc`, `bcc`, or `replyTo`. If the tool call
errors or fails, treat the run as failed at this step — still proceed to
step 8 (archive + commit) so the attempt is recorded, but do not silently
pretend it succeeded.

## 8. Archive and update state

- Write the same HTML body to `runs/<today's date, YYYY-MM-DD>.html`.
- Append any listings tagged New in step 5 to the state array (with
  `first_seen` = today's date). Leave existing entries unchanged. Write the
  full array back to `state/seen_listings.json`.
- `git add state/seen_listings.json runs/<today>.html && git commit -m
  "Weekly run <today's date>: <N> new listings" && git push`.

Use the commit trailer:
```
Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
```
