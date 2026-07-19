# Live lead data via Supabase (Option B)

**Why:** the daily agent cannot read the live Google Sheet from its environment (the Drive
connector returns a multi-day-stale cached copy, and direct Google fetch is network-blocked).
So leads are mirrored into a Supabase table the agent CAN read live, and counted with duplicates
removed (by phone).

## Table
- Project: **IPC Studios** (`yhsambvuciyoopgblzrh`, ap-south-1)
- Table: **`public.meta_ad_leads`**
- Insert endpoint: `POST https://yhsambvuciyoopgblzrh.supabase.co/rest/v1/meta_ad_leads`
- Auth: the project's **publishable key** (`sb_publishable_...`) — a public, insert-only key
  (RLS allows insert, not read), safe to place in the form/Apps Script. **Do not commit the key to
  this public repo** — keep it in the Apps Script only.
- Columns mirror the sheet: `lead_ts, name, phone, email, annual_revenue, source, utm_source,
  utm_medium, utm_campaign, adset_name, ad_name, utm_term, utm_content, landing_page`. Plus a
  generated `phone_norm` (last 10 digits of phone) used for de-duplication.

## Bridge: Google Apps Script (paste into the lead sheet)
The sheet is the current destination of leads, and Apps Script (running as the sheet owner in
Google's cloud) CAN both read the sheet and reach Supabase. It mirrors every new row.

1. Open the lead sheet → **Extensions → Apps Script**.
2. Paste the script below; set `SUPABASE_KEY` to the publishable key.
3. **Triggers** (clock icon) → add a **time-driven** trigger → `syncLeadsToSupabase` → every 5 min.
4. Run it once manually to authorise + backfill history.

```javascript
const SUPABASE_URL = 'https://yhsambvuciyoopgblzrh.supabase.co/rest/v1/meta_ad_leads';
const SUPABASE_KEY = 'PASTE_PUBLISHABLE_KEY_HERE'; // sb_publishable_...

function syncLeadsToSupabase() {
  const sh = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
  const data = sh.getDataRange().getValues();
  const props = PropertiesService.getScriptProperties();
  const lastRow = Number(props.getProperty('lastSyncedRow') || 1); // row 1 = header
  const rows = [];
  for (let i = lastRow; i < data.length; i++) {
    const r = data[i];
    if (!r[0]) continue;
    rows.push({
      lead_ts: r[0], name: r[1], phone: String(r[2]), email: r[3], annual_revenue: r[4],
      source: r[5], utm_source: r[6], utm_medium: r[7], utm_campaign: String(r[8]),
      adset_name: r[9], ad_name: r[10], utm_term: String(r[11]), utm_content: String(r[12]),
      landing_page: r[13]
    });
  }
  if (!rows.length) return;
  const resp = UrlFetchApp.fetch(SUPABASE_URL, {
    method: 'post', contentType: 'application/json',
    headers: { apikey: SUPABASE_KEY, Authorization: 'Bearer ' + SUPABASE_KEY, Prefer: 'return=minimal' },
    payload: JSON.stringify(rows), muteHttpExceptions: true
  });
  if (resp.getResponseCode() < 300) props.setProperty('lastSyncedRow', String(data.length));
  else Logger.log('Supabase error ' + resp.getResponseCode() + ': ' + resp.getContentText());
}
```

## How the agent reads leads (deduped by phone)
Per-ad-set leads for a day (Asia/Kolkata), duplicates removed by phone:
```sql
select utm_term as adset_id, count(distinct phone_norm) as leads
from public.meta_ad_leads
where utm_campaign = '120249797872700412'
  and (lead_ts at time zone 'Asia/Kolkata')::date = '<YYYY-MM-DD>'
group by utm_term;
```
CPL = Meta spend (per ad set) ÷ these deduped leads. Meta's "Website leads" is kept as a
cross-check; if the two diverge a lot, note it in the report.
