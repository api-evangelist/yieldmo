---
name: Profile a Yieldmo audience and pull attribution
description: Describe who a Yieldmo campaign actually reached — demographic composition, segments and browser topics — then pull the advertiser's retargeting and attributed-conversion rows.
api: openapi/yieldmo-dcs-mcp-openapi.json
base_url: https://api.yieldmo.com/dcs/mcp
operations:
  - campaign_summary_canned_reports_campaign_summary_get
  - campaign_segment_demographic_canned_reports_campaign_segment_demographic_get
  - segment_description_canned_reports_segment_description_get
  - topic_id_canned_reports_topic_id_get
  - conversion_types_canned_reports_conversion_types_get
  - advertiser_data_canned_reports_advertiser_data_get
generated: '2026-08-12'
method: generated
source: Grounded in openapi/yieldmo-dcs-mcp-openapi.json; every operationId above appears verbatim in that spec.
---

# Profile a Yieldmo audience and pull attribution

## Before you start

- Authenticate with an OAuth 2.0 Bearer token (Amazon Cognito) — see
  `authentication/yieldmo-authentication.yml`.
- **This skill handles audience and conversion data.** Treat every response as confidential
  advertiser data. Do not echo raw segment or conversion rows into a shared surface; summarise.
- **Two date formats live in this one skill.** Steps 2–4 take integer `YYYYMMDD` date keys
  (`start_date_key` / `end_date_key`). Step 6 takes ISO strings (`start_time` / `end_time`, format
  `YYYY-MM-DD`). Mixing them up is the most common `422` on this API.

## Steps

1. **Get the advertiser.** Call `campaign_summary_canned_reports_campaign_summary_get` with the
   `campaign_id`. You need the advertiser ID from here for step 6 — it is described in the spec as
   the "Snowflake advertiser ID", an internal warehouse key, so do not expect it to match an
   advertiser ID from a DSP or from your own CRM.

2. **Pull demographic composition.** Call
   `campaign_segment_demographic_canned_reports_campaign_segment_demographic_get` with
   `campaign_ids` and the date range. Returns audience demographic composition with **Experian index
   scores**. Index scores are relative to a baseline, not percentages — an index of 140 means 1.4x
   the baseline concentration, not "140% of the audience". Say which it is when you report it.

3. **Resolve the segments.** The demographic report gives segment IDs. Call
   `segment_description_canned_reports_segment_description_get` with `segment_ids` (array) to get
   human-readable descriptions. Never present a bare segment ID to a person.

4. **Pull browser topics.** Call `topic_id_canned_reports_topic_id_get` with `campaign_ids`, `kpi`
   and the date range. Returns the **top 10** browser Topics API topics by campaign impressions with
   audience index scores. Ten is the fixed ceiling — it is not a `limit` you can raise.

5. **Load the conversion vocabulary.** Call
   `conversion_types_canned_reports_conversion_types_get`. It takes **no parameters** and returns the
   full conversion-type reference list. Fetch it once and cache it for the session; you will need it
   to make step 6's rows readable.

6. **Pull attribution.** Call `advertiser_data_canned_reports_advertiser_data_get` with
   `advertiser_id` (integer), `start_time` and `end_time` (**ISO `YYYY-MM-DD` strings, not date
   keys**). `start_time` defaults to `1900-01-01`; `end_time` defaults to today and is **exclusive**.
   Both must be valid dates and `start_time` must be strictly before `end_time`.

   It returns one object with three keys — read them correctly:
   - `retargeting_dated` — retargeting rows **filtered by your date range**
   - `retargeting_all` — retargeting rows for **all dates**, ignoring your range
   - `attributed_conversions` — attributed conversion rows

   Reporting `retargeting_all` as if it were the period figure is the single easiest mistake to make
   here. Use `retargeting_dated` for anything period-scoped.

## Interpretation notes

- Index scores (Experian, topics) are comparative, not absolute. Always state the comparison basis.
- Yieldmo is cookieless and ID-independent by design, so this audience data is modelled and
  probabilistic. Do not describe it as deterministic user-level identity.
- Join the conversion-type list from step 5 onto the `attributed_conversions` rows before
  summarising, or the output is a wall of unlabelled codes.

## Errors

- `422` — check `detail[].loc`. On this skill it is almost always a date-format mismatch (step 6
  wants `YYYY-MM-DD`, steps 2–4 want `20260101`-style integers), or `start_time` not before
  `end_time`.
- `401` — token missing or expired.
- Empty results are a valid answer: not every campaign has segment, topic or conversion coverage.
  Say the data is absent rather than inferring zero performance.
