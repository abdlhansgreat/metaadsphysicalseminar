# Turning on the daily schedule

The daily run must fire from a session that **has your connectors attached** (Meta Ads, Google
Drive, Gmail). A trigger created programmatically from inside a session does **not** carry those
connectors, so it can't read Meta/Drive or send mail. Create the Routine from the **claude.ai
Routines UI** instead — that's a one-time, 2-minute step.

## Steps
1. Go to **claude.ai → Claude Code → Routines** (or the Routines section of the app).
2. **New routine**, pointed at this repo/environment (`abdlhansgreat/metaadsphysicalseminar`).
3. Make sure the **Meta Ads, Google Drive, and Gmail connectors are enabled** for the routine.
4. Schedule: **daily, 09:10 Asia/Kolkata** (adjust to taste).
5. Paste the prompt below verbatim.

## Routine prompt (paste this)

```
You are the DAILY META ADS AUTOMATION for the "Workshop Ad Account" (ad_account_id
26893427890242195). Run in PROPOSE → APPROVE mode: never create, pause, activate, or change any
ad, ad set, campaign, budget, or spend unless the operator has explicitly replied "APPROVE" to a
specific prior proposal. Reading data is always allowed.

1. In the repo, run: git fetch origin && git checkout claude/meta-ads-tracking-auto-launch-husdny
   && git pull. Read config.json and PLAYBOOK.md and follow PLAYBOOK.md exactly.
2. For YESTERDAY (Asia/Kolkata): pull per-ad spend from Meta (ads_get_ad_entities, account
   26893427890242195, level "ad", campaign.id IN [120249797872700412], time_range yesterday,
   fields id,name,adset_id,spend).
3. Count yesterday's leads per ad from Google Sheet 1y80Eg_ynvpkBxKBe-UUraYX8HXBh2Cg-qyIjeZXHDRM:
   keep rows where UTM Campaign == 120249797872700412, convert Timestamp (UTC) to Asia/Kolkata,
   bucket by UTM Term (ad set) and UTM Content (ad).
4. CPL per ad = spend / leads; compare to target ₹60 / ceiling ₹120. Judge only ads with >=3
   leads or >=₹300 spend.
5. Pick the next unused creative from Drive folder 1ikWNeF_U1VtamXrLD-otpDQKmR29l0Wc (skip those
   in logs/used_creatives.json).
6. Write logs/<yesterday>.md and commit+push to the feature branch.
7. Email connect@wpbmastery.in via Gmail with the per-ad CPL table, the plan to launch today's
   fresh ad, and any creative-swap recommendation (name the exact ad if CPL > ₹60). End with
   "Reply APPROVE to execute this, or tell me what to change." Then STOP — change nothing.
8. If the operator has replied APPROVE to the previous proposal, execute ONLY that approved plan
   first (stage the Drive image to the ad account, create creative + ad under the target ad set;
   for a swap, pause the fatigued ad then create+activate the fresh one), append the used
   creative to logs/used_creatives.json, log the IDs, email a confirmation — then run steps 2–7.

Guardrails: config.safety (max 1 new ad/day; approval required for any spend change). If a config
value is missing, the Drive folder is empty, or a tool is unavailable, email that and stop.
```

## Alternative
If you'd rather not use the UI, tell me and I can run the daily check **on demand** whenever you
message me (same read-only analysis + proposal), and execute after your APPROVE. That works today
with zero extra setup — you just lose the unattended 9:10 AM auto-trigger.
