# Pagoda Hardtop Watch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a weekly cloud routine that searches UK/EU classifieds and W113 specialist sources for standalone Pagoda hardtops, emails Glen a summary (new listings first, rest by distance from London), and persists state between runs — without ever connecting to Glen's Gmail account.

**Architecture:** A `RemoteTrigger` cloud routine (weekly cron) clones `G2thaCizzo/pagoda-hardtop-agent`, runs the prompt in `prompts/weekly_search.md` (WebSearch/WebFetch for listings, Bash for git), and pushes updated state back to the repo each run. Email goes out via a **Resend MCP connector** attached to the routine's `mcp_connections` — the routine calls the connector's `send-email` tool directly, no API key ever handled by the assistant or stored anywhere. (Two earlier choices were ruled out: SendGrid dropped its permanent free tier — now a 60-day trial only, then paid — and Brevo requires DKIM/DMARC domain verification Glen can't do without owning a domain. A raw-API-key-via-cloud-secret approach was also attempted and abandoned — no such secret-storage mechanism actually exists in the routine tooling. Resend's default `onboarding@resend.dev` sender is pre-authenticated and restricted to delivering only to the account's own signup email — exactly glendanielcooney@gmail.com, the one recipient needed here.)

**Tech Stack:** Claude Code `RemoteTrigger` cloud routine (claude-sonnet-5), Resend MCP connector (`mcp.resend.com`, `send-email` tool), git, JSON state file.

**Spec:** `docs/superpowers/specs/2026-09-09-hardtop-agent-design.md`

## Global Constraints

- No OAuth/direct connection to Glen's Gmail account, ever (explicit requirement).
- No credentials committed to the repo, written to any file, or handled by the assistant — email sends through the Resend MCP connector, which manages its own auth.
- Hardtop-only listings — full-car listings are discarded even if they mention a hardtop.
- A single source failing must not fail the whole run.
- If zero listings are found across all sources, the run must still send an email saying so (this is the "something's broken" signal), never skip silently.
- Recipient: `glendanielcooney@gmail.com`.
- Repo: `https://github.com/G2thaCizzo/pagoda-hardtop-agent` (private), already initialized with `CLAUDE.md`, the spec, `state/seen_listings.json` (`[]`), and `runs/.gitkeep`.
- Cloud environment: `env_01FAt9bWaCD17d2ri4J7MtLj` ("Default").
- Resend connector: `connector_uuid: b5ae2e69-79bd-4c96-8cad-3319171616a0`, `name: Resend`, `url: https://mcp.resend.com` — only its `send-email` tool is auto-allowed; every other tool on the connector is left on "Needs approval" or denied.
- Cron: `0 6 * * 1` (06:00 UTC Monday = 07:00 Europe/London during BST; drifts to 06:00 local once the UK reverts to GMT — cron is fixed UTC and does not auto-adjust for DST. Acceptable per spec; not a defect.).

---

### Task 1: Create a Resend account (done)

This was a manual account-setup task Glen did with guidance. Recorded here for the audit trail.

**Files:** none (external account setup only)

**Interfaces:**
- Consumes: nothing
- Produces: confirmation that the Resend account's signup email is exactly `glendanielcooney@gmail.com`. Nothing downstream needs an API key — Task 4 uses the connector instead (see Task 2).

- [x] **Step 1: Create a free Resend account**

Signed up at https://resend.com/signup using `glendanielcooney@gmail.com` — required, since Resend's default sender can only deliver to the account's own signup address.

---

### Task 2: Connect the Resend MCP connector (done)

The original Task 2 ("store the API key as a cloud-environment secret") turned out to target a mechanism that doesn't exist — there is no secrets/environment-variables page for cloud environments in the routine tooling. This replacement task is what actually works: a claude.ai connector, which manages its own auth and needs no key handling by the assistant at all.

**Files:** none (claude.ai connector configuration, outside this repo)

**Interfaces:**
- Consumes: nothing (Glen authorized the connector directly at claude.ai/customize/connectors)
- Produces: an active Resend connector — `connector_uuid: b5ae2e69-79bd-4c96-8cad-3319171616a0`, `name: Resend`, `url: https://mcp.resend.com` — with its `send-email` tool set to Allow. Task 4 attaches this via `mcp_connections`.

- [x] **Step 1: Connect the connector**

Connected at https://claude.ai/customize/connectors, scoped to "Sending access" (not full account access).

- [x] **Step 2: Set the one required tool permission**

In the connector's tool-permission list (Settings → Connectors → Resend), found "Send Email" under "Write/delete tools" and set it to **Allow**. Left every other tool (43 read-only + 59 other write/delete tools) on "Needs approval" or denied — a scheduled routine runs unattended, so anything left on "Needs approval" would simply hang forever waiting for a click that never comes; only the one tool actually needed is auto-allowed.

- [x] **Step 3: Confirm the connector is visible to routines**

Re-checked via the `schedule` skill's connector list after an overnight propagation delay — it initially showed "No available MCP connectors found" three times in a row despite being connected, then appeared correctly the next morning: `Resend (connector_uuid: b5ae2e69-79bd-4c96-8cad-3319171616a0, ...)`. Lesson for next time: this can take longer than one session to propagate — don't fall back to a less secure approach after just a few retries.

---

### Task 3: Verify the prompt file is placeholder-free and ready

Unlike SendGrid/Brevo, Resend needs no sender address substitution — `prompts/weekly_search.md` already has the real, final `onboarding@resend.dev` sender hard-coded in Section 7. This task is a final read-through check before that content becomes the routine's live instructions in Task 4, not an edit.

**Files:**
- Verify only: `Hard Top Agent/prompts/weekly_search.md`

**Interfaces:**
- Consumes: nothing new (the file is already in its final state)
- Produces: confirmed-final prompt text (string) — Task 4 copies this file's content verbatim into the routine's `events[].data.message.content`.

- [ ] **Step 1: Read the full file**

Read `prompts/weekly_search.md` end to end.

- [ ] **Step 2: Verify no placeholder text remains**

Confirm there is no `VERIFIED_SENDER_ADDRESS`, `SUBJECT_HERE`, `HTML_BODY_HERE`, or other bracketed/templated placeholder left unresolved anywhere — `SUBJECT_HERE` and `HTML_BODY_HERE` are intentional (the routine fills those in itself at runtime from the listings it finds, they are not setup placeholders). Confirm Section 7's `from` field reads `"Pagoda Hardtop Watch <onboarding@resend.dev>"` and the `to` field reads `["glendanielcooney@gmail.com"]`.

- [ ] **Step 3: No commit needed**

If Step 2 found nothing to fix, there's nothing to commit — proceed to Task 4. If it did find something, fix it, then:
```bash
cd "Hard Top Agent"
git add prompts/weekly_search.md
git commit -m "Fix outstanding placeholder in weekly prompt"
git push
```

---

### Task 4: Create the RemoteTrigger routine

**Files:** none (routine lives in claude.ai's routine store, not this repo)

**Interfaces:**
- Consumes: final prompt text from Task 3, repo URL, environment id, cron expression (all from Global Constraints)
- Produces: `trigger_id` (string) — Task 5 and Task 6 both consume this to run/inspect the routine.

- [ ] **Step 1: Load the RemoteTrigger tool**

Call `ToolSearch` with `select:RemoteTrigger` if not already loaded this session.

- [ ] **Step 2: Read the final prompt content**

Read `Hard Top Agent/prompts/weekly_search.md` in full (post-Task-3 edit) — this becomes the `message.content` string below verbatim.

- [ ] **Step 3: Create the routine**

Call `RemoteTrigger` with `action: "create"` and body:

```json
{
  "name": "pagoda-hardtop-watch",
  "cron_expression": "0 6 * * 1",
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "env_01FAt9bWaCD17d2ri4J7MtLj",
      "session_context": {
        "model": "claude-sonnet-5",
        "sources": [
          {"git_repository": {"url": "https://github.com/G2thaCizzo/pagoda-hardtop-agent"}}
        ],
        "allowed_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep", "WebSearch", "WebFetch"]
      },
      "events": [
        {"data": {
          "uuid": "<generate a fresh lowercase v4 uuid>",
          "session_id": "",
          "type": "user",
          "parent_tool_use_id": null,
          "message": {"content": "<full content of prompts/weekly_search.md from Step 2>", "role": "user"}
        }}
      ]
    }
  },
  "mcp_connections": [
    {"connector_uuid": "b5ae2e69-79bd-4c96-8cad-3319171616a0", "name": "Resend", "url": "https://mcp.resend.com"}
  ]
}
```

- [x] **Step 4: Record the trigger ID**

`trigger_id: trig_016X2UagUegJcG4Lid62r8Ez` — https://claude.ai/code/routines/trig_016X2UagUegJcG4Lid62r8Ez. Created 2026-09-10; `next_run_at` 2026-09-14T06:08:57Z (Monday, on schedule).

---

### Task 5: Manual test run and verification

**Files:**
- Read/verify: `Hard Top Agent/state/seen_listings.json`, `Hard Top Agent/runs/*.html` (after the run)

**Interfaces:**
- Consumes: `trigger_id` from Task 4
- Produces: a verified-working routine — Task 6 consumes this working state to test the dedup behavior.

- [x] **Step 1: Trigger a manual run**

Called `RemoteTrigger` `action: "run"` with `trigger_id: trig_016X2UagUegJcG4Lid62r8Ez` — session `cse_01Puxd5YnkvYRsQvQdwc9FVe`.

- [x] **Step 2: Watch the run log**

Reviewed via `get_run_log`. The `send-email` tool call to the Resend connector succeeded on the first try (`Email sent successfully! {"id":"6439057d-db3b-47b6-9e58-2293779bd723"}`) — no permission issues, confirming Task 2's connector setup works correctly.

**Finding:** the run also discovered that this cloud sandbox blocks all outbound WebFetch/curl to external domains (tested and confirmed against unrelated sites too, e.g. Wikipedia — a blanket egress restriction, not per-source). Only WebSearch works. Since every candidate listing needs page verification to confirm price/location/status, and none of that was possible, the routine correctly treated this as the zero-listings "something's broken" case per the spec, sent an honest email explaining it, left `state/seen_listings.json` untouched, and committed `runs/2026-09-10.html`. This is correct behavior for what happened, but means the original page-fetch-based design can't work in this sandbox — see the spec's Known Limitations for the reworked snippet-only approach, now reflected in `prompts/weekly_search.md`.

- [x] **Step 3: Confirm the email arrived**

Glen confirmed receipt of the "no listings found — network blocked" email.

- [x] **Step 4: Update the live routine with the reworked prompt**

`prompts/weekly_search.md` was rewritten after this run to extract listing details from WebSearch snippets directly (title/price/location/condition where the snippet shows them, no thumbnail/posted-date, `site:`-scoped queries per source) instead of relying on WebFetch. Pushed to the live routine via `RemoteTrigger` `action: "update"`.

- [x] **Step 5: Trigger a second manual run against the reworked prompt**

Called `RemoteTrigger` `action: "run"` with `trigger_id: trig_016X2UagUegJcG4Lid62r8Ez` — session `cse_01LDcpDDu3S3hkp3aQ67nkpk`. Result: the snippet-only approach works — searched all 13 sources via `site:`-scoped WebSearch, found 2 verifiable standalone-hardtop listings (both eBay UK; every other source returned only full cars, accessories, or category pages with no confirmable individual listing URL), sent the email successfully via the Resend connector, and committed.

**Second finding, fixed inline:** this run hit a real git issue — the sandbox checks out a **detached HEAD** each run rather than a checked-out `main` branch, and local `main` had gone stale after the first run (still pointing at an old commit even though the detached HEAD had moved forward via fetches). `git checkout main` failed as a result. The routine diagnosed this itself correctly (verified the old `main` was a safe fast-forward ancestor of the current detached HEAD, force-updated the branch pointer, checked it out, committed, and pushed successfully) — but this cost several extra tool calls it shouldn't need to repeat weekly. Hardened `prompts/weekly_search.md` Step 8 afterward to skip `git checkout main` entirely and push directly via `git push origin HEAD:main` from the detached HEAD — pushed to the live routine via `RemoteTrigger` `action: "update"`.

- [x] **Step 6: Confirm state and archive were committed**

```bash
cd "Hard Top Agent"
git fetch origin && git log --oneline origin/main -5
git pull --no-edit
```
Confirmed: `origin/main` advanced to `b05a9e4` ("Weekly run 2026-09-10: 2 new listings"), pulled cleanly. `state/seen_listings.json` now holds the 2 eBay UK listings (both with `price`/`location`/`currency` null — the snippets didn't carry that detail, exactly the tradeoff documented in the spec's Known Limitations) and `runs/2026-09-10.html` was overwritten with the second run's fuller email content.

---

### Task 6: Second manual run — confirm dedup works

**Files:** none (verification only)

**Interfaces:**
- Consumes: `trigger_id` from Task 4, the post-Task-5 state of `state/seen_listings.json`
- Produces: confidence the weekly cadence behaves correctly — consumed by Task 7 as the baseline it improves on.

- [x] **Step 1: Trigger a second manual run**

Called `RemoteTrigger` `action: "run"` with `trigger_id: trig_016X2UagUegJcG4Lid62r8Ez`, shortly after Task 5 — session `cse_012RidrpPPDUvBn7mD47nNPT`.

- [x] **Step 2: Confirm no false "New" repeats**

Confirmed via `state/seen_listings.json`: the 2 listings from Task 5 kept their original `first_seen: "2026-09-10"` unchanged (not re-flagged New) while 2 genuinely new listings (another eBay UK hardtop + an sl113.org forum post) were correctly appended — email reported "4 listings found, 2 new," matching.

- [x] **Step 3: Confirm the git fix worked cleanly**

This run's `git push origin HEAD:main` succeeded on the first try with no fast-forward workaround needed — confirms Task 5's hardening of Section 8 fixed the detached-HEAD issue for good.

- [x] **Step 4: Confirm the routine is left enabled on its real schedule**

`RemoteTrigger` `action: "get"` on the `trigger_id` — confirmed `enabled: true` and `cron_expression: "0 6 * * 1"`. The routine runs unattended every Monday from here.

---

### Task 7: Improve search query construction (post-launch refinement)

After Task 6, Glen reviewed an email preview and flagged two things: only 2 UK eBay listings were found despite 13 sources being searched, and no thumbnail images appeared. Investigated both directly (interactively, via this session's own WebSearch tool) before changing the prompt.

**Files:**
- Modify: `Hard Top Agent/prompts/weekly_search.md` (Section 2 — query construction and extraction instructions)

**Interfaces:**
- Consumes: Task 6's working routine as the baseline
- Produces: final prompt text (string) — pushed to the live routine the same way Task 4/5 did, via `RemoteTrigger` `action: "update"`.

- [x] **Step 1: Investigate the thumbnail gap**

Ran WebSearch directly (this session, not the routine) against several sources. Confirmed: WebSearch never returns image URLs in its response, in any form. Combined with WebFetch being blocked (Task 5's finding), there is no way to obtain a real thumbnail in this sandbox. `thumbnail: null` is a hard constraint, not a bug — documented plainly in the spec's Known Limitations rather than worked around.

- [x] **Step 2: Investigate the low-yield gap**

Ran comparative WebSearch queries: the routine's `site:domain.tld` + jargon approach (e.g. `site:kleinanzeigen.de W113 Hartschalendach Pagode`) vs. a natural-language query naming the domain as plain text plus a local "for sale" phrase (e.g. `Mercedes W113 hardtop kleinanzeigen.de`). The natural-language form consistently surfaced real standalone-hardtop listings with actual prices and locations that the `site:` form missed — confirmed on German, French, Dutch and Spanish sources (e.g. a €5,000 and a €3,500 hardtop on LeBonCoin, a €2,000/€2,500 hardtop on Kleinanzeigen, a €2,500 hardtop on Marktplaats). Root cause: WebSearch's synthesized summary (not just its `Links` array) carries this detail, and the routine's original instructions only pointed it at "the snippet" without directing it to that fuller summary text.

- [x] **Step 3: Rewrite Section 2's query and extraction instructions**

Updated `prompts/weekly_search.md`: dropped `site:` as the primary query form in favor of natural-language queries per source, added Milanuncios to the Spain row, and explicitly instructed reading WebSearch's full response (summary text included) rather than just link titles. Made the thumbnail constraint explicit and unconditional at the top of the section.

- [ ] **Step 4: Push the updated prompt to the live routine**

Read the final `prompts/weekly_search.md` content and call `RemoteTrigger` `action: "update"` with `trigger_id: trig_016X2UagUegJcG4Lid62r8Ez`, same shape as Task 4/5's updates (fresh UUID for the event).

- [ ] **Step 5: One more manual run to confirm the improvement**

Call `RemoteTrigger` `action: "run"` with the same `trigger_id`. Expected: listings from multiple European sources this time, not just eBay UK — if it's still UK-only after this change, that's worth a closer look rather than assuming the fix worked. `thumbnail` should remain `null` throughout (expected, not a regression).

---

## Self-review notes

- Spec coverage: sources list, extraction fields, GBP conversion, distance estimate, diff logic, email structure, error handling (single-source failure, zero-results case, state corruption), archiving, and the no-Gmail-OAuth constraint are all covered in `prompts/weekly_search.md` and enforced structurally by Task 1/2 (Resend connector, not Gmail).
- The cloud-environment-secret path assumed in an earlier version of this plan turned out not to exist — verified by finding no such field in the `RemoteTrigger` create schema and no working settings page, after three empty connector-list checks. Rather than fabricate a workaround, the plan was rewritten around the Resend MCP connector, which is the tooling's actual supported mechanism for this.
- Reused `trigger_id` and the connector's `connector_uuid`/`name`/`url` consistently across Tasks 2, 4, 5, 6.
