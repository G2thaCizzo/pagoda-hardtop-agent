# Pagoda Hardtop Watch

## What this is
An automated weekly search for standalone Mercedes-Benz W113 "Pagoda" hardtops
(230SL/250SL/280SL-compatible) for sale across the UK and Europe, emailed to
Glen as a summary — new listings highlighted, the rest sorted by distance
from London. Full-car listings are explicitly out of scope; this is hardtop-
only.

This folder is also a standalone git repo
(`https://github.com/G2thaCizzo/pagoda-hardtop-agent`, private), because a
scheduled cloud agent (Claude Code "routine") clones it fresh every Monday to
read/write its state and pushes updates back. Treat commits to this repo as
coming from that automation as well as from manual edits.

## Key context
- Part of the broader [Pagoda](../CLAUDE.md) import project, but kept as its
  own repo/subfolder since it has its own automation lifecycle.
- Email delivery deliberately avoids any OAuth connection to Glen's Gmail
  account (explicit requirement) — uses a Resend send-only API key
  instead, with Gmail only ever as the recipient. Landed on Resend after
  two false starts: SendGrid dropped its permanent free tier (now a
  60-day trial, then paid), and Brevo requires DKIM/DMARC domain
  verification that isn't possible without owning a domain. Resend's
  default `onboarding@resend.dev` sender is pre-authenticated and can
  only deliver to the Resend account's own signup email — since that's
  glendanielcooney@gmail.com, the one recipient this needs, that
  restriction is actually a perfect fit, not a limitation.
- Design rationale, source list, and known limitations are in
  `docs/superpowers/specs/2026-09-09-hardtop-agent-design.md`.

## Constraints
- Follow workspace security rules: no credentials in files, HTTPS only, no
  sensitive data leakage.
- No direct Gmail account connection/OAuth — Resend or equivalent
  send-only mechanism only.
- Hardtop-only listings — discard full-car listings even if they mention a
  hardtop.

## Files in this folder
- `CLAUDE.md` — this file.
- `docs/superpowers/specs/2026-09-09-hardtop-agent-design.md` — full design
  spec (architecture, sources, error handling, known limitations).
- `state/seen_listings.json` — persisted history of every listing seen, used
  to detect "new" each week. Written by the cloud routine.
- `runs/` — archived HTML copy of each week's email.
