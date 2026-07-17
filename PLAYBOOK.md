# Daily Meta Ads Playbook

This is the **brain** of the automation. A scheduled Routine fires every morning, opens a
fresh Claude Code session on this repo, and runs the steps below. Follow them **in order**.
Do not improvise money-spending actions — this account runs in **propose → approve** mode.

> Ground rule: **You never create, pause, or change an ad, budget, or campaign unless the
> operator has written `APPROVE` for that specific proposal.** Reading data is always allowed.

---

## Step 0 — Load config
Read `config.json`. If any value is still `TODO_...`, stop and post a message listing the
missing fields instead of running. Everything below uses values from that file.

Set `TODAY` and `YESTERDAY` in the account timezone (`account.timezone`).

## Step 1 — Pull yesterday's performance from Meta
Use `ads_get_ad_entities` on `account.ad_account_id`:
- `level: "adset"` (and `level: "ad"` for the breakdown), `date_preset: "yesterday"`
- fields: `id`, `name`, `spend`, `cost_per_lead`, and the lead action count.
Record per-ad and per-ad-set: **spend, leads, CPL**.

## Step 2 — Count leads from the source of truth (the sheet)
Read `leads_source.sheet_file_id` with the Google Drive tool. Count rows where the
`date_column_header` equals `YESTERDAY` (parse using `date_format`). This sheet count is the
**official lead number**. Compute `CPL_actual = yesterday_spend / sheet_leads`
(if `sheet_leads` is 0, CPL is treated as "infinite / no leads").

## Step 3 — Judge CPL
Only judge if there is enough signal:
- `sheet_leads >= cpl_rules.min_leads_before_judging` **or**
  `yesterday_spend >= cpl_rules.min_spend_before_judging_inr`.
Otherwise mark **"not enough data, keep running"** and skip the breach action.

Breach = `CPL_actual > cpl_rules.target_cpl_inr` (and hard breach if above `hard_ceiling_cpl_inr`).

## Step 4 — Decide today's actions (do NOT execute yet)
Assemble a **proposal**:

1. **New ad for today** (`daily_new_ad.enabled`): pick the next creative from the Drive
   folder (`creatives.drive_folder_id`) using `selection_order`, skipping anything already in
   `logs/used_creatives.json`. Propose creating one new ad under `campaign.ad_set_id` with
   `daily_new_ad.daily_budget_inr`.
2. **If CPL breached** (`on_breach: swap_creative`): identify the highest-CPL ad above target
   with enough spend, and propose **pausing it** and launching a **fresh creative** in its place.
3. Respect `safety` caps (max new ads/day, max spend change, never-touch list).

## Step 5 — Send the proposal for approval
Write the proposal to `logs/YYYY-MM-DD.md` **and** send it via `approval.notify_channel`
(email to `approval.notify_email`) with a clear subject like
`[Meta Ads] 2026-07-17 — CPL ₹X, proposal inside`. The email must state exactly what will
happen and end with: *"Reply APPROVE to run this, or tell me what to change."*

**Then stop.** Do not create or modify anything on this run.

## Step 6 — Execute only after approval
On a run where the operator has replied `APPROVE` (or the previous day's proposal is marked
approved), execute the approved items:
- New ad: `ads_create_creative` → `ads_create_ad` under the ad set → keep **PAUSED**, then
  `ads_activate_entity` only if approval said "and turn it on".
- Swap: `ads_update_entity` to pause the fatigued ad, then create + activate the fresh one.
- Append the creative you used to `logs/used_creatives.json`.
- Write the outcome (IDs created, what changed) into today's log file and email a confirmation.

## Step 7 — Always leave a paper trail
Every run appends to `logs/YYYY-MM-DD.md`: the numbers pulled, the decision, the proposal,
and (if executed) the resulting entity IDs. Commit the log back to the repo so history is durable.

---

### Creative → Meta note
Meta needs the image/video in the ad account's library. Two supported paths:
- The creative file is reachable by a public `image_url`, **or**
- It has been uploaded to the ad account (an `image_hash` / `video_id` exists).
If neither is true for the chosen Drive file, the proposal must say so and ask the operator to
stage it, rather than failing silently.
