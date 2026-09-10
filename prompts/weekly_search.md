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
verify price, or read full condition details. Do not attempt WebFetch or
curl against classifieds sites — it will fail every time; don't waste
turns re-testing this. **No thumbnail images are available either way** —
WebSearch never returns image URLs (confirmed by testing), and there is no
way to fetch one from the page. Don't invent or guess an image URL; every
listing's `thumbnail` field is always `null`.

For each source below, run a WebSearch using **natural language, not the
`site:` operator** — testing showed `site:domain.tld` queries return only
thin title/URL lists, while a plain query naming the domain and a local
"for sale" phrase gets WebSearch's own summarization to surface real price
and location detail pulled from the result pages. Combine: "Mercedes
W113 Pagode/Pagoda hard top/hardtop" + the domain name as plain text + the
local phrase shown. Try both "hardtop" and "hard top" as separate words if
the first attempt returns only full cars or category/search pages. Try
each source; if nothing useful comes back after two attempts, move on and
note it in the "sources not checked" list for the email — do not let one
source's failure stop the run.

**Crucial: read WebSearch's full response, not just the `Links` array.**
The prose summary that follows the links frequently contains price,
location and condition pulled from the actual listing pages — richer than
the bare titles. Extract from that summary text, not only from link
titles.

| Country | Source(s) | Example WebSearch query |
|---|---|---|
| UK | eBay UK, Gumtree, PistonHeads classifieds | `Mercedes W113 Pagoda hard top ebay.co.uk for sale`, `... gumtree.com for sale`, `... pistonheads.com for sale` |
| Germany | eBay Kleinanzeigen | `Mercedes W113 Pagode hardtop kleinanzeigen.de` |
| France | LeBonCoin | `Mercedes W113 Pagode hard top leboncoin.fr à vendre` |
| Netherlands | Marktplaats | `Mercedes W113 Pagode hardtop marktplaats te koop` |
| Belgium | 2ememain, AutoScout24.be | `Mercedes W113 Pagode hardtop 2ememain.be te koop`, `... autoscout24.be` |
| Spain | Coches.net, Wallapop, Milanuncios | `Mercedes W113 Pagoda techo rígido coches.net en venta`, `... wallapop`, `... milanuncios en venta` |
| Italy | Subito.it | `Mercedes W113 Pagoda hard top subito.it vendita` |
| Austria | willhaben | `Mercedes W113 Pagode hardtop willhaben zu verkaufen` |
| Portugal | StandVirtual.pt | `Mercedes W113 Pagoda capota rígida standvirtual à venda` |
| Sweden | Blocket | `Mercedes W113 Pagoda hardtop blocket till salu` |
| Denmark | DBA.dk | `Mercedes W113 Pagoda hardtop dba.dk til salg` |
| Hungary | Hasznaltauto.hu | `Mercedes W113 Pagoda hardtop hasznaltauto.hu eladó` |
| Cross-EU / specialist | AutoScout24.com, sl113.org classifieds/forum, Bring a Trailer, The MB Market | `Mercedes W113 Pagoda hardtop autoscout24.com`, `Mercedes W113 hardtop for sale sl113.org`, `... bringatrailer.com`, `... thembmarket.com` |

For every candidate result whose URL is actually on the target domain (not
a retailer/parts/SEO page unrelated to that specific source), extract from
WebSearch's full response (links plus summary text, per above):
- `title` (from the search result)
- `price` and `currency` if mentioned anywhere in the response (as shown —
  may be EUR, GBP, SEK, DKK, HUF etc.); `null` if genuinely not mentioned
- `location` (town/region + country) if mentioned; `null` if not
- `url` (the canonical listing URL)
- `thumbnail`: always `null` (see constraint above)
- `condition_notes`: whatever is mentioned about condition, or `null` if
  nothing is — do not invent detail beyond what's actually stated
- `posted_date`: `null` — not reliably available from search results

Every listing is **unverified** — found via search, not confirmed still
for sale, since the page itself can't be opened. Say this plainly in the
email (see step 6) rather than implying these are live-checked.

**Category-page fallback (confirmed necessary in testing):** WebSearch's
summary text sometimes clearly describes a specific real listing — a price
and location, e.g. "a hard top for €3,500 in Paris 75015" — but the only
URL it returns for that source is a category/search-results page (e.g.
`leboncoin.fr/ck/equipement_auto/hard-top-mercedes`), not the individual
ad's own page. This happens because search engines often rank a site's
category page above a specific long-tail classified ad. When this
happens, don't discard the lead and don't force a fake individual URL —
capture it separately as a **lead**: `{"description": "<what the summary
said, e.g. the price/location text>", "category_url": "<the
category/search page URL>", "source": "<site name>"}`. Keep a running list
of these across all sources as you search — call it `leads`. These are
distinct from `listings` (which always have a real per-item URL) and are
handled differently in steps 5, 6 and 8 below — never mix the two lists.

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

**Leads (category-page fallback) are never diffed against state and never
tracked for dedup.** A category-page URL recurs every week regardless of
which specific item it currently points to, so "seen before" has no
meaning for it — show every lead found this run, every run, with no
new/seen distinction. Do not add `leads` entries to `state/seen_listings.json`
in step 8.

## 6. Build the email

One HTML email, structure:

1. Subject: `Pagoda Hardtop Watch — <today's date> — <N> new`
2. Short header line: date, total listings found, how many new, how many
   leads (if any), list any sources you couldn't check this run (e.g. "no
   usable results this run"), and a one-line note that all listings are
   found-via-search and unverified as still active (no page fetch is
   possible in this environment — see step 2).
3. **New listings** section first (if any): each as a block with title
   (linked to the listing URL), price (original + approx GBP, or "not
   shown" if the snippet didn't include one), location (or "not shown"),
   estimated distance from London (only if location is known), and
   condition notes (or omit the line entirely if null). No thumbnail —
   not available from a search snippet.
4. **All other listings** section: same block format, sorted by estimated
   distance ascending (listings with no known location go last, grouped
   under "distance unknown").
5. **Possible leads (no direct link found)** section, if any `leads` were
   found: each as a block with the `description` text, a link labeled
   "Browse this category — direct listing link not found" pointing to
   `category_url`, and the source name. Visually distinct from confirmed
   listings (e.g. a lighter border/background) so it reads as lower-
   confidence than the sections above — these are never flagged New/Seen
   (see step 5).
6. If literally zero listings **and** zero leads were found across every
   source, still send the email — header should say so plainly (e.g. "No
   listings found this run — this may mean sources are blocked; check the
   run log"). This is the signal something is broken, so it must not be
   silently skipped.

## 7. Send the email

You have a connected Resend MCP tool available (attached to this routine
as a connector — do not use curl, do not look for any API key in the
environment; authentication is handled entirely by the connector). Call
its email-sending tool (action name `send-email`) with:

- `to`: `["glendanielcooney@gmail.com"]`
- `from`: `Pagoda Hardtop Watch <onboarding@resend.dev>` — literal angle
  brackets, not HTML-entity-encoded (`&lt;`/`&gt;`); Resend's API rejects
  the encoded form (confirmed in testing — this field is a plain header
  value, not HTML, even though `html` right below it is HTML)
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
- This checkout starts on a **detached HEAD**, not a checked-out `main`
  branch — don't run `git checkout main` (it will fail or require
  fast-forward surgery; confirmed the hard way in testing). Just commit
  directly on top of the detached HEAD and push straight to `main` with a
  refspec:
  ```
  git add state/seen_listings.json runs/<today>.html
  git commit -m "Weekly run <today's date>: <N> new listings"
  git push origin HEAD:main
  ```

Use the commit trailer:
```
Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
```
