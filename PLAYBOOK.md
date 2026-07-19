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

## Step 1 — Pull yesterday's leads + CPL from META (source of truth)
Use `ads_get_ad_entities` on `account.ad_account_id`, scoped to `campaign.campaign_id`,
`level: "adset"`, `time_range` = yesterday, `time_increment: 1`, fields:
`id`, `name`, `spend`, `results`, `cost_per_result`.
- `results` = **Website leads** (same number Ads Manager shows), `cost_per_result` = **CPL**.
- This is LIVE and reliable. Record spend, leads (results), and CPL per ad set. Repeat at
  `level: "ad"` for the per-ad breakdown when you need to name a specific ad.

## Step 2 — (Sheet is NOT used for counts)
Do **NOT** count leads from the Google Sheet — the Drive export is cached/stale and has produced
false "no leads" alarms. The sheet only holds contact details; Meta's Website leads (Step 1) is
the official count. If you ever open the sheet, treat its freshness with suspicion.

## Step 3 — Judge CPL
For each ad set: CPL = `cost_per_result` (or spend/results). Only judge with enough signal:
- `results >= cpl_rules.min_leads_before_judging` **or** `spend >= cpl_rules.min_spend_before_judging_inr`.
Otherwise **"not enough data, keep running."**
Breach = `CPL > cpl_rules.target_cpl_inr` (hard breach above `hard_ceiling_cpl_inr`).

## Step 3.5 — RED-FLAG CHECKS (do this FIRST, alert IMMEDIATELY) 🚨
Before anything else, evaluate every condition in `config.red_flag_alerts.conditions`. If ANY is
true, send a **separate 🚨 RED FLAG WhatsApp message** to the group RIGHT AWAY (and email it) —
do NOT wait and do NOT bury it inside the daily report. The most important one:
- **leads_stopped_while_spending:** base this on **META Website leads (results), NOT the sheet.**
  If the lead campaign spent > ~₹300 in a full day but Meta reports **0 Website leads** that day,
  the form/landing-page pixel is likely broken — flag CRITICAL with the spend and the last day that
  had leads. NEVER raise this from the Google Sheet being empty (its Drive export is cached and gave
  a false alarm on 17-18 Jul). Re-confirm in the group as soon as Meta shows leads again.
Only after red-flag checks are handled do you proceed to the normal report/optimisation.

## Step 4 — Decide today's actions (do NOT execute yet)
Assemble a **proposal**:

1. **New ad for today** (`daily_new_ad.enabled`): for each target ad set in
   `campaign.target_ad_sets`, pick the next unused image from **that ad set's own
   `creatives_folder_id`** (its city folder), using `selection_order` and skipping anything in
   `logs/used_creatives.json`. Propose creating one new ad **inside that same ad set** (it shares
   the ad set's existing daily budget — no budget is added).
   - **CITY-LOCK (hard rule):** a Lucknow ad set may use ONLY Lucknow-folder images; a Chandigarh
     ad set ONLY Chandigarh-folder images. Never use an image from the other city's folder.
   - If a city's folder is empty (e.g. Chandigarh has no graphics yet), skip that city and say so
     in the proposal — do not substitute another city's image.
2. **If CPL breached** (`on_breach: swap_creative`): identify the highest-CPL ad above target
   with enough spend, and propose **pausing it** and launching a **fresh creative from the same
   city's folder** in its place.
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

### Creative → Meta bridge (PROVEN 2026-07-17)
Meta CANNOT fetch Google Drive links, and the Meta connector has NO image-upload tool. So to
launch a Drive graphic, host it at a public URL Meta can fetch:
1. `download_file_content` the Drive image, decode the base64 **to a local file with a shell
   command** (do NOT pull 2 MB of base64 through the model context).
2. `git add` the file under `creatives/` and push to the public repo
   (`image_hosting.public_repo`, branch `image_hosting.branch`).
3. Use the raw URL `image_hosting.raw_url_pattern` as `image_url` in `ads_create_creative`.
   Meta fetches raw.githubusercontent.com fine.
4. `ads_create_ad` with the returned `creative_id` under the target ad set, then
   `ads_activate_entity` to make it live. Record file→creative_id→ad_id in
   `logs/used_creatives.json`.

**Page ad-limit fallback:** after activating, check the ad's `effective_status`. If it comes back
`WITH_ISSUES` because the `pages.primary_page_id` hit its ad limit, recreate the SAME creative with
`pages.fallback_page_id` (Bihar Wedding Photography) and relaunch. The fallback page has no linked
Instagram accessible to this account, so fallback ads are **Facebook-only** (omit `instagram_user_id`).
Leave the blocked primary-page ad paused. Note the page switch in the report and the log.

### Reporting scope + FORMAT (send this every day)
Report per ad set for EVERY campaign in `monitoring.report_campaigns`. Use exactly this layout:

```
📊 Meta Ads — Daily Report — <date> (IST)

LEAD CAMPAIGN — Lucknow Workshop
Ad set        Spent     Leads   CPL      vs ₹60
Lucknow|UP    ₹—        —       ₹—       🟢/🟡/🔴
Chandigarh|PB ₹—        —       ₹—       🟢/🟡/🔴
Total         ₹—        —       ₹— (blended)

MESSAGING CAMPAIGN — New Leads
Spent: ₹—   Conversations started: —   Cost/conversation: ₹—

ACTION
- <either> ✅ Done automatically: <what creatives were pushed / where>
- <or>     ⚠️ Recommend (needs your APPROVE): <pause / budget change>
- <or>     — Nothing needed today.
```

- Lead campaign → CPL = Meta spend ÷ sheet leads (UTM attribution), per ad set.
- Messaging campaign (`120248325986010412`) → read STRAIGHT FROM META (the dashboard numbers):
  query `ads_get_ad_entities` with fields `spend`, `results`, `cost_per_result`. `results` =
  "Messaging conversations started", `cost_per_result` = cost per conversation. **Never** use
  `cost_per_lead` or derive conversations from spend — that is wrong (it gave ₹407/2 instead of
  the true ₹81.37/10). What the report shows must equal what Ads Manager shows.

### Autonomy (per `approval.rules`) — the daily optimisation job
Each day, judge WHICH ad set and WHICH creative is performing (spend, leads/conversations, CPL,
CTR trend) and optimise. Then always inform the operator in the report.
- **DO ON YOUR OWN, then report:** launch/add creatives; pause a weak ad or creative; pause
  (switch OFF) a weak ad set; any optimisation that does NOT raise spend.
- **TAKE OPERATOR'S CONSIDERATION FIRST (never do unasked):** increasing any budget (2x/3x etc.);
  creating a new campaign; anything that raises total spend. Put these under "Recommend — needs APPROVE".
- **SAFETY:** before switching OFF the last active ad set in a campaign (stops all delivery for that
  city), confirm first.
- **LOCATION IS LOCKED — never edit geo targeting.** Lucknow = Uttar Pradesh only; Chandigarh =
  Punjab only.
