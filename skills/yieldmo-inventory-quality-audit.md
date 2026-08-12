---
name: Audit Yieldmo inventory quality by URL
description: Find where a Yieldmo campaign's impressions actually landed — best and worst performing URLs, biddable-object and segment-level breakdowns — so spend can be steered away from weak inventory.
api: openapi/yieldmo-dcs-mcp-openapi.json
base_url: https://api.yieldmo.com/dcs/mcp
operations:
  - campaign_id_from_name_canned_reports_campaign_id_from_name_get
  - web_campaign_check_canned_reports_web_campaign_check_get
  - top_urls_by_campaign_canned_reports_top_urls_by_campaign_get
  - top_bottom_urls_canned_reports_top_bottom_urls_get
  - campaign_biddable_object_url_performance_canned_reports_campaign_biddable_object_url_performance_get
  - campaign_feature_list_kpi_canned_reports_campaign_feature_list_kpi_get
generated: '2026-08-12'
method: generated
source: Grounded in openapi/yieldmo-dcs-mcp-openapi.json; every operationId above appears verbatim in that spec.
---

# Audit Yieldmo inventory quality by URL

## Before you start

Authenticate first (OAuth 2.0 Bearer, Amazon Cognito — see
`authentication/yieldmo-authentication.yml`). Set `start_date_key` explicitly as a `YYYYMMDD`
integer; the default `19000101` requests all history and will hit the row cap.

## Steps

1. **Resolve the campaign** with
   `campaign_id_from_name_canned_reports_campaign_id_from_name_get` if you were given a name.

2. **Check there is URL data at all — do this first.** Call
   `web_campaign_check_canned_reports_web_campaign_check_get` with `campaign_ids`,
   `start_date_key` and `end_date_key`. This exists precisely because URL-level data is not
   available for every campaign or every window. If it comes back empty, stop and say so rather
   than reporting an empty URL report as "no good inventory found".

3. **Get the top URLs.** Call
   `top_urls_by_campaign_canned_reports_top_urls_by_campaign_get` with `campaign_ids`, the date
   range, `min_impressions` (default `0`) and `limit` (default `100`). **Always set
   `min_impressions`** — at `0` the ranking fills with single-impression URLs at 100% CTR, which is
   noise, not signal. A floor of a few hundred impressions is the point of the parameter.

4. **Get both ends of the distribution.** Call
   `top_bottom_urls_canned_reports_top_bottom_urls_get` twice — once with `top: "DESC"` and once
   with `top: "ASC"`. Its parameters:
   - `filter` — the column to sort by: `total_imps` (default), `total_spend`, or `total_kpi`
   - `top` — `DESC` for best, `ASC` for worst
   - `kpi` — which metric `total_kpi` means (default `ctr`)
   - `min_imps` — impression floor (note: spelled `min_imps` here, `min_impressions` on the
     operation above; they are the same idea with different names)
   - `num_entries` — row cap, default `50`
   The bottom list is the actionable half: those are the URLs to exclude.

5. **Go one level deeper.** Call
   `campaign_biddable_object_url_performance_canned_reports_campaign_biddable_object_url_performance_get`
   for performance joined at biddable object / segment / URL level (`limit` default `1000`). This is
   what tells you whether a weak URL is weak everywhere or only for one segment.

6. **Cross-cut by environment.** Call
   `campaign_feature_list_kpi_canned_reports_campaign_feature_list_kpi_get` with `feature_list` —
   an array of dimension names such as `device_type`, `browser`, `os` — plus `kpi`. Use this to
   check whether a URL's poor performance is really a device or browser problem.

## Output discipline

- Rank on a KPI the buyer named. Default `ctr` is rarely the right answer for a video campaign —
  prefer `vcr_play` or `cpcv` there, and `attn` when the conversation is about attention.
- Always report the impression floor you applied. A top-URL list without a stated `min_impressions`
  is not interpretable.
- If a result set comes back at exactly the row cap, it was truncated. There is no pagination —
  narrow the date range and re-run, and say the list is partial.

## Errors

`422` means a parameter failed validation — check `detail[].loc`. `filter` and `top` are typed as
bare strings, so a typo (`total_impressions` instead of `total_imps`, or `desc` instead of `DESC`)
may not be caught client-side; verify the values before calling. `401` means the token expired.
