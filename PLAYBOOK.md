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

## Step 1 — Pull yesterday's spend from Meta (per ad set AND per ad)
Use `ads_get_ad_entities` on `account.ad_account_id`, scoped to `campaign.campaign_id`:
- `level: "ad"` and `level: "adset"`, `date_preset: "yesterday"`
- fields: `id`, `name`, `adset_id`, `spend`.
Record **spend per ad set** and **spend per ad**. (Meta's own `cost_per_lead` can differ from
your sheet's truth — spend is what we take from Meta; leads come from the sheet.)

## Step 2 — Count leads from the source of truth (the sheet)
Read `leads_source.sheet_file_id` with the Google Drive tool. The sheet is UTM-tagged, so:
- Keep only rows where `attribution.campaign_id_column` (**UTM Campaign**) ==
  `attribution.only_count_campaign_id`.
- Convert each row's `date_column_header` (**Timestamp**, ISO-8601 UTC) into
  `leads_source.convert_to_timezone` and keep rows whose local date == `YESTERDAY`.
- Bucket the leads by `attribution.adset_id_column` (**UTM Term** = ad set) and
  `attribution.ad_id_column` (**UTM Content** = ad).
This gives **leads per ad set** and **leads per ad** — the official numbers.

## Step 3 — Compute CPL per ad and per ad set, then judge
For each ad (and ad set): `CPL = yesterday_spend / sheet_leads` (0 leads → treat as "no leads /
infinite CPL"). Only judge an entity if it has enough signal:
- `sheet_leads >= cpl_rules.min_leads_before_judging` **or**
  `spend >= cpl_rules.min_spend_before_judging_inr`.
Otherwise mark it **"not enough data, keep running."**

Breach = `CPL > cpl_rules.target_cpl_inr` (hard breach above `hard_ceiling_cpl_inr`). Because the
sheet carries the ad ID, you can name the **exact ad** whose CPL crossed the line.

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

### Autonomy (per `approval.rules`)
- Adding / pushing new creatives into a target ad set is AUTONOMOUS — do it, then report it under "Done automatically".
- Pausing ads, changing budgets, or creating campaigns REQUIRES an explicit APPROVE (list under "Recommend").
- **LOCATION IS LOCKED — never edit geo targeting.** Lucknow ad set = Uttar Pradesh only;
  Chandigarh ad set = Punjab only. The agent only ever adds creatives; it must never touch location.
