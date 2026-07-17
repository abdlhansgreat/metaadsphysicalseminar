# Setup Guide

Fill these five things and the system is ready. Everything goes into `config.json`.

## 1. Which ad account
You have ~30 Meta ad accounts connected. Pick the ONE this automation manages and put its
numeric ID in `account.ad_account_id` (and a friendly name in `ad_account_name`).
Ask the agent "list my ad accounts" any time to see them again.

## 2. The campaign / ad set new ads go under
- `campaign.ad_set_id` — new daily ads are created inside this ad set (so they inherit its
  targeting, optimization, and audience). If you don't have one yet, tell the agent your
  targeting and it will propose creating a Leads campaign + ad set first.
- `campaign.page_id` — the Facebook Page the ads publish from.
- `campaign.link_url` — the landing page or lead-form URL the ad clicks to.
- `campaign.instagram_user_id` — optional, for Instagram placements.

## 3. The lead sheet
- `leads_source.sheet_file_id` — the Google Sheet's file ID (the long string in its URL:
  `docs.google.com/spreadsheets/d/<THIS_PART>/edit`).
- `date_column_header` + `date_format` — so the agent can count leads per day.
- Optional `campaign_or_source_column_header` if the sheet mixes leads from several campaigns
  and you want to count only this one.

## 4. The creatives folder
- `creatives.drive_folder_id` — the Google Drive folder ID where you upload ad images/videos.
  The agent launches the next unused file each day.

## 5. Money + threshold
- `cpl_rules.target_cpl_inr` — your CPL threshold. Above this = "swap creative" is proposed.
- `cpl_rules.hard_ceiling_cpl_inr` — the red line.
- `daily_new_ad.daily_budget_inr` — budget for each new daily ad.
- `safety.*` — the guardrails. Keep `require_approval_for_any_spend_change: true` unless you
  later decide to let it run fully autonomously.

## 6. Schedule + notifications
- `schedule.run_local_time` — when the daily run happens (Asia/Kolkata).
- `approval.notify_email` — where proposals and confirmations are sent.

---

### Getting IDs quickly
You don't have to hunt these down manually. Once you tell the agent the ad account name and
paste the Google Sheet link + Drive folder link, it can resolve the IDs for you and fill in
`config.json`. The `TODO_` placeholders are just so nothing runs half-configured.
