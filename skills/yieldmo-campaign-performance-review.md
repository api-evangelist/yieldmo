---
name: Review Yieldmo campaign performance
description: Resolve a Yieldmo YMax campaign by name, pull its headline performance and daily spend, and break the numbers out by KPI and creative — the standard "how is this campaign doing?" pass.
api: openapi/yieldmo-dcs-mcp-openapi.json
base_url: https://api.yieldmo.com/dcs/mcp
operations:
  - campaign_id_from_name_canned_reports_campaign_id_from_name_get
  - child_campaign_ids_from_parent_name_canned_reports_child_campaign_ids_from_parent_name_get
  - campaign_summary_canned_reports_campaign_summary_get
  - campaign_performance_canned_reports_campaign_performance_get
  - campaign_daily_spend_canned_reports_campaign_daily_spend_get
  - campaign_daily_performance_canned_reports_campaign_daily_performance_get
  - campaign_metrics_canned_reports_campaign_metrics_get
  - campaign_creative_performance_canned_reports_campaign_creative_performance_get
generated: '2026-08-12'
method: generated
source: Grounded in openapi/yieldmo-dcs-mcp-openapi.json; every operationId above appears verbatim in that spec.
---

# Review Yieldmo campaign performance

## Before you start

- **Auth is required and there is no API key path.** Complete the OAuth 2.0 authorization-code flow
  against `https://yieldmo-cuba.auth.us-east-1.amazoncognito.com/oauth2/authorize`, exchange at
  `/oauth2/token`, and send `Authorization: Bearer <token>`. Without it every call returns
  `401 {"error": "invalid_token"}`. See `authentication/yieldmo-authentication.yml`.
- **Scopes do not narrow anything.** Only `openid`, `profile`, `email` exist. You cannot request
  read-only or single-advertiser access — the token's identity is the whole authorization boundary.
- **Dates are integer keys, not strings.** `start_date_key` and `end_date_key` are integers in
  `YYYYMMDD` form. `start_date_key` defaults to `19000101`, so **always set it** — an unset call
  requests all history.
- **`campaign_ids` is an array even for one campaign.** Repeat the query parameter.

## Steps

1. **Resolve the campaign.** If you were given a name, call
   `campaign_id_from_name_canned_reports_campaign_id_from_name_get` with `campaign_names`.
   If the name is a *parent* campaign, use
   `child_campaign_ids_from_parent_name_canned_reports_child_campaign_ids_from_parent_name_get`
   instead — it takes `parent_campaign_names_or_ids` (names **or** ids on the one parameter, never
   both kinds mixed) and returns every child `campaign_id`. Report on the children, not the parent.

2. **Confirm you have the right campaign.** Call
   `campaign_summary_canned_reports_campaign_summary_get` with the single `campaign_id`. It returns
   the campaign name plus advertiser ID and advertiser name. Check the advertiser matches what the
   user expects before showing any numbers — campaign names are not unique across advertisers.

3. **Pull the headline.** Call
   `campaign_performance_canned_reports_campaign_performance_get` with `campaign_ids`,
   `start_date_key` and `end_date_key`. Returns impressions, clicks, CTR, spend and win rate.
   `limit` defaults to 5000 rows.

4. **Pull the trend.** Call
   `campaign_daily_spend_canned_reports_campaign_daily_spend_get` for the daily spend and impression
   curve, and `campaign_daily_performance_canned_reports_campaign_daily_performance_get` for the
   daily KPI curve. The second takes a `kpi` parameter (default `ctr`).

5. **Pull period-over-period.** Call
   `campaign_metrics_canned_reports_campaign_metrics_get` for impressions, KPI, spend, CPM and cost
   per action compared across periods. Same `kpi` parameter.

6. **Break out by creative.** Call
   `campaign_creative_performance_canned_reports_campaign_creative_performance_get`. Returns
   AdBuilder creative ID, format, impressions, clicks, MRC viewable impressions, video
   starts/completions (VCR) and spend.

## KPI values

`kpi` is typed as a bare string in the spec but only these are valid — validate before calling:

`ctr`, `attn`, `mrc`, `mrc_cpm_non_video`, `vcr_imps`, `vcr_play`, `play_rate`, `cpm`,
`cpm_non_video`, `cpm_video`, `cpc`, `cpcv`, `groupm`, `groupm_viewability`

`attn` is Yieldmo's proprietary attention metric — it is the reason to use this API rather than a DSP's.

## Interpreting nulls

- `adbuilder_creative_id` is **null for programmatic campaigns** where the DSP hosts the creative.
  Do not report that as missing data; it means Yieldmo did not build the creative.
- `vcr` is **null for non-video creatives**. Do not average it across a mixed-format campaign.

## Errors

- `422` — a parameter failed validation. Read `detail[].loc` for the offending parameter and
  `detail[].type` for the Pydantic error class. Most common cause: a date passed as a string
  instead of a `YYYYMMDD` integer.
- `401` — token missing or expired; re-run the OAuth flow.
- No `429` and no rate-limit headers exist. Do not build retry logic against a `Retry-After` that
  will never arrive; back off on connection errors instead.

## Truncation warning

There is **no pagination** — no cursor, no offset, no total count. If a result set comes back at
exactly the `limit` (5000 / 1000 / 100 / 50 depending on the operation), assume it was truncated and
narrow the date range rather than raising the limit. Say so in the output; do not present a truncated
set as complete.
