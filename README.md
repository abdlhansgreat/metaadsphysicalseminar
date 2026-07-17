# Meta Ads Daily Automation

An agent-driven system that, **every morning**, reads yesterday's Meta ad spend and the leads
that landed in your Google Sheet, calculates **cost per lead (CPL)**, and proposes the day's
actions — launch a fresh ad from your Google Drive folder, and if CPL is over your threshold,
swap out the tired creative. It runs in **propose → approve** mode: it never spends or changes
anything until you reply `APPROVE`.

## How it actually runs (important)

Claude Code on the web is not a server that stays on. So the system is made of durable parts:

| Piece | What it is | Where it lives |
|-------|-----------|----------------|
| **Playbook** | The exact daily logic the agent follows | `PLAYBOOK.md` (in git) |
| **Config** | Your account, sheet, folder, CPL threshold, budget | `config.json` (in git) |
| **Daily Routine** | A scheduled trigger that opens a fresh session each morning and runs the playbook | Managed trigger (set up once) |
| **Logs** | A dated record of every day's numbers and decisions | `logs/` (in git) |

Each morning the Routine fires → a fresh agent reads `config.json` + `PLAYBOOK.md` → pulls
Meta + Sheet data → emails you a proposal → waits. You reply `APPROVE` → it executes and logs it.

## Data sources (all connected via MCP)

- **Meta Ads** — spend, leads, CPL; create/pause/adjust ads. (~30 ad accounts visible.)
- **Google Sheet** — the official daily lead count.
- **Google Drive folder** — the pool of ad creatives to launch from.
- **Gmail** — how the agent sends you the daily proposal and confirmations.

## Setup checklist

1. Fill in every `TODO_...` in [`config.json`](./config.json). See `docs/SETUP.md`.
2. Confirm the Facebook Page, ad set, and lead-form / landing URL for new ads.
3. Stage creatives in the Drive folder.
4. Approve the daily Routine (time set in `config.json` → `schedule`).
5. Watch the first few days in `logs/` and tune `cpl_rules`.

## Safety

- Nothing that spends money happens without `APPROVE`.
- Hard caps live in `config.json` → `safety` (max new ads/day, max spend change, never-touch list).
- Every action is logged and committed, so there is a full audit trail.

See `docs/SETUP.md` for the step-by-step, and `PLAYBOOK.md` for the exact daily algorithm.
