# Pagoda Hardtop Watch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a weekly cloud routine that searches UK/EU classifieds and W113 specialist sources for standalone Pagoda hardtops, emails Glen a summary (new listings first, rest by distance from London), and persists state between runs — without ever connecting to Glen's Gmail account.

**Architecture:** A `RemoteTrigger` cloud routine (weekly cron) clones `G2thaCizzo/pagoda-hardtop-agent`, runs the prompt in `prompts/weekly_search.md` (WebSearch/WebFetch for listings, Bash for git and the Resend HTTP call), and pushes updated state back to the repo each run. Email goes out via a Resend send-only API key stored as a cloud-environment secret — no OAuth link to any real mailbox. (Two earlier choices were ruled out: SendGrid dropped its permanent free tier — now a 60-day trial only, then paid — and Brevo requires DKIM/DMARC domain verification Glen can't do without owning a domain. Resend's default `onboarding@resend.dev` sender is pre-authenticated and restricted to delivering only to the account's own signup email — exactly glendanielcooney@gmail.com, the one recipient needed here.)

**Tech Stack:** Claude Code `RemoteTrigger` cloud routine (claude-sonnet-5), Resend email API (HTTP/curl), git, JSON state file.

**Spec:** `docs/superpowers/specs/2026-09-09-hardtop-agent-design.md`

## Global Constraints

- No OAuth/direct connection to Glen's Gmail account, ever (explicit requirement).
- No credentials committed to the repo or written to any file — the Resend key lives only as a cloud-environment secret, read via `$RESEND_API_KEY`.
- Hardtop-only listings — full-car listings are discarded even if they mention a hardtop.
- A single source failing must not fail the whole run.
- If zero listings are found across all sources, the run must still send an email saying so (this is the "something's broken" signal), never skip silently.
- Recipient: `glendanielcooney@gmail.com`.
- Repo: `https://github.com/G2thaCizzo/pagoda-hardtop-agent` (private), already initialized with `CLAUDE.md`, the spec, `state/seen_listings.json` (`[]`), and `runs/.gitkeep`.
- Cloud environment: `env_01FAt9bWaCD17d2ri4J7MtLj` ("Default").
- Cron: `0 6 * * 1` (06:00 UTC Monday = 07:00 Europe/London during BST; drifts to 06:00 local once the UK reverts to GMT — cron is fixed UTC and does not auto-adjust for DST. Acceptable per spec; not a defect.).

---

### Task 1: Create a Resend account and API key

This is a manual account-setup task Glen does with guidance — there is no code to write, but there is a concrete, checkable deliverable: an account signed up with the right email, and an API key. Unlike SendGrid/Brevo, no separate sender-verification step is needed — Resend's default sending address is pre-authenticated.

**Files:** none (external account setup only)

**Interfaces:**
- Consumes: nothing
- Produces: confirmation that the Resend account's signup email is exactly `glendanielcooney@gmail.com`, and a Resend API key (string, held only in Resend's dashboard at this point — not pasted anywhere yet). Task 2 consumes the key; Task 3 needs no address substitution this time (the sender is fixed as `onboarding@resend.dev`, already correct in `prompts/weekly_search.md`).

- [ ] **Step 1: Create a free Resend account**

Go to https://resend.com/signup and sign up using **exactly** `glendanielcooney@gmail.com` — this must match, since Resend's default sender can only deliver to the account's own signup address. Free plan (3,000 emails/month) is far more than needed for one email/week.

- [ ] **Step 2: Create an API key**

In the Resend dashboard: API Keys → "Create API Key". Name it `pagoda-hardtop-watch`. Permission: "Sending access" only if that option is offered (not full account access). Copy the generated key immediately (Resend shows it once).

**Do not paste the key into chat or save it to any file in this repo or workspace.** It goes directly into the cloud environment's secret store in Task 2.

- [ ] **Step 3: Confirm deliverable**

Glen confirms in chat: "account created with the right email, key created" (without pasting the key). This unblocks Task 2. (No Task 3 file edit is needed this time — skip straight to Task 2, then Task 4.)

---

### Task 2: Store the Resend API key as a cloud-environment secret

**Files:** none (cloud environment configuration, outside this repo)

**Interfaces:**
- Consumes: the API key from Task 1 (never handled by the assistant directly — Glen enters it himself)
- Produces: `RESEND_API_KEY` readable as an environment variable inside any cloud routine run using environment `env_01FAt9bWaCD17d2ri4J7MtLj`. Task 4's routine and Task 5/6's manual runs both consume this.

- [ ] **Step 1: Open the environment's secret settings**

Go to https://claude.ai/code/environments, open the "Default" environment (`env_01FAt9bWaCD17d2ri4J7MtLj`), find its secrets/environment-variables section.

- [ ] **Step 2: Add the secret**

Add a secret named exactly `RESEND_API_KEY` with the value from Task 1 Step 2. Save.

- [ ] **Step 3: Confirm deliverable**

Glen confirms in chat: "secret added" (without pasting the value). If this environment turns out not to support custom secrets, stop and report back — that changes Task 4's design (would need a different secret-injection path, e.g. a connector) and should not be silently worked around.

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
  }
}
```

- [ ] **Step 4: Record the trigger ID**

Note the `trigger_id` returned. Confirm the routine appears via `RemoteTrigger` `action: "list"`.

---

### Task 5: Manual test run and verification

**Files:**
- Read/verify: `Hard Top Agent/state/seen_listings.json`, `Hard Top Agent/runs/*.html` (after the run)

**Interfaces:**
- Consumes: `trigger_id` from Task 4
- Produces: a verified-working routine — Task 6 consumes this working state to test the dedup behavior.

- [ ] **Step 1: Trigger a manual run**

Call `RemoteTrigger` `action: "run"` with the `trigger_id` from Task 4.

- [ ] **Step 2: Watch the run log**

Poll `RemoteTrigger` `action: "list_runs"` for the new session, then `action: "get_run_log"` on it. Expected: no permission denials, no unhandled tool errors; the log shows sources being searched, an email send attempt, and a git commit/push.

Note: the first run should log an HTTP 200 from the Resend call. If it doesn't, stop here — check Task 1/2 (account signup email matches the recipient exactly, secret name/value) before re-running, rather than re-running blindly.

- [ ] **Step 3: Confirm the email arrived**

Glen checks glendanielcooney@gmail.com for "Pagoda Hardtop Watch — ... — N new". Confirm: thumbnails render, listing links open correctly, the New-listings section (if any) appears before the by-distance section, and any "source not checked" notes read sensibly.

- [ ] **Step 4: Confirm state and archive were committed**

```bash
cd "Hard Top Agent"
git pull
```
Confirm `state/seen_listings.json` is no longer `[]` (assuming at least one listing was found) and a new `runs/<date>.html` file exists matching the email content.

---

### Task 6: Second manual run — confirm dedup works

**Files:** none (verification only)

**Interfaces:**
- Consumes: `trigger_id` from Task 4, the post-Task-5 state of `state/seen_listings.json`
- Produces: confidence the weekly cadence will behave correctly going forward — final task, nothing downstream consumes this.

- [ ] **Step 1: Trigger a second manual run**

Call `RemoteTrigger` `action: "run"` with the same `trigger_id`, shortly after Task 5 completes.

- [ ] **Step 2: Confirm no false "New" repeats**

Check the resulting email/log: listings present in `state/seen_listings.json` from the first run should now appear in the "all other listings" (by-distance) section, not re-flagged as New. Only genuinely new listings found since the first run should be flagged New.

- [ ] **Step 3: Confirm the routine is left enabled on its real schedule**

`RemoteTrigger` `action: "get"` on the `trigger_id` — confirm `enabled: true` and `cron_expression: "0 6 * * 1"`. The routine now runs unattended every Monday.

---

## Self-review notes

- Spec coverage: sources list, extraction fields, GBP conversion, distance estimate, diff logic, email structure, error handling (single-source failure, zero-results case, state corruption), archiving, and the no-Gmail-OAuth constraint are all covered in `prompts/weekly_search.md` and enforced structurally by Task 1/2 (Resend, not Gmail).
- The one open risk flagged rather than assumed away: Task 2 Step 2 explicitly stops and reports back if the "Default" cloud environment doesn't support custom secrets, instead of guessing at a workaround.
- Reused `trigger_id` consistently across Tasks 4, 5, 6. `VERIFIED_SENDER_ADDRESS` no longer applies (Resend's sender is fixed, not per-account) — Task 3 was rewritten as a verification-only step rather than left as a stale reference to a resolved-elsewhere placeholder.
