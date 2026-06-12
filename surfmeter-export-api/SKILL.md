---
name: surfmeter-export-api
description: Access and query AVEQ Surfmeter performance monitoring data via the read-only Export API, or manage clients, measurements, anomalies, keys, and settings via the read-write Surfmeter (management) API. Use when fetching video quality metrics, web performance data, network measurements, or any Surfmeter analytics data, and when modifying server-side resources over HTTP. Supports Elasticsearch queries for aggregations and filtering.
---

# Surfmeter Export API

Access AVEQ Surfmeter performance and quality monitoring data.

Surfmeter is a AVEQ's solution for measuring and analyzing the quality of experience (QoE) of video streaming, web performance, and network connectivity from the end-user perspective. The Surfmeter Export API provides a read-only interface to query and retrieve measurement data, client information, reports, and user feedback collected by Surfmeter probes deployed across the network. There are different types of Surfmeter clients:

- Surfmeter Automator: desktop/Docker-based probes for scheduled headless measurements
- Surfmeter Mobile SDK: mobile library for in-app measurements, e.g. for drive testing
- Surfmeter Player SDK: library for video players to provide client analytics

## Two HTTP APIs: pick the right one

A Surfmeter server exposes two separate HTTP APIs.

- **Export API** (`/export_api/v1`, this skill's focus) — read-only data export. Simplest option for pulling measurements, reports, clients, and feedback. Use it whenever you only need to *read* data.
- **Surfmeter (management) API** (`/client_admin_api/v1`) — the read-write API behind the Admin Dashboard. Use it when you need to *modify* something (update client labels/tags, resolve anomaly events, manage keys, users, ISPs, settings) or query resources the Export API doesn't expose (anomaly events, license usage, capabilities). See [Surfmeter Management API](#surfmeter-management-api-read-write) below.

## Full API Reference

Endpoint-level details (parameters, response shapes, all resources) are documented at:

<https://docs.aveq.info/surfmeter-docs/api/export-api/resources/>

Fetch this page (or its sub-pages) when you need anything beyond what is summarized below — for example, exact field schemas, less-common endpoints, or new parameters that may have been added since this skill was written.

## Base URL

```
https://surfmeter-server.<customer>-analytics.aveq.info/export_api/v1
```

Ask the user for their `<customer>` if you don't already know it.

## Authentication

All requests need the `X-API-KEY` header. Use `EXPORT_API_KEY` from `.env` if present; otherwise ask the user. Keys are created at `https://surfmeter-server.<customer>-analytics.aveq.info/app/export-api-keys`.

```bash
curl -H "Accept: application/json" \
     -H "X-API-KEY: $EXPORT_API_KEY" \
     "https://surfmeter-server.<customer>-analytics.aveq.info/export_api/v1/measurements"
```

## Elasticsearch Search (measurements only)

For analytics, filtering, and aggregation over **measurement data**, use `/search` rather than fetching from `/measurements` and filtering client-side. It is far more efficient.

Important: `/search` only indexes measurement data. The other resources below (`/clients`, `/client_reports`, `/event_bundles`, `/network_requests`, `/user_feedbacks`) are **not** in Elasticsearch — you must hit their dedicated routes.

**GET/POST** `/search`

Parameters:

- `body` (required): Elasticsearch query body (JSON)
- `scroll`: Enable scrolling with timeout (e.g. `1m`) for NDJSON streaming over large result sets
- `source_only`: Boolean — return only source documents without ES metadata

Use the `.keyword` suffix for exact matches on text fields (e.g. `type.keyword`, `client_label.keyword`). This matters more than it looks: a `term` filter on the analyzed text field (without `.keyword`) **silently matches nothing** — no error, just zero hits — so a query that returns empty is often a missing `.keyword`, not missing data. The same field used in an `aggs` or `sort` instead errors loudly (`Fielddata is disabled on [field]`). When in doubt, suffix every text field with `.keyword` in filters, aggregations, and sorts alike.

Simple filter:

```json
POST /search
{
  "body": {
    "query": {
      "term": { "client_label.keyword": "my-custom-label" }
    }
  }
}
```

Time-bucketed aggregation:

```json
POST /search
{
  "body": {
    "size": 0,
    "query": {
      "bool": {
        "must": [
          { "range": { "created_at": { "gte": "now-30d/d", "lte": "now/d" } } },
          { "term":  { "type.keyword": "VideoMeasurement" } }
        ]
      }
    },
    "aggs": {
      "loading_delay_over_time": {
        "date_histogram": { "field": "created_at", "calendar_interval": "hour" },
        "aggs": {
          "avg_loading_delay": { "avg": { "field": "statistic_values.initial_loading_delay" } }
        }
      }
    }
  }
}
```

## Resource Endpoints (Summary)

For measurements: reach for `/measurements` when you need raw, paginated records rather than aggregations — otherwise prefer `/search` above. For everything else (clients, reports, events, etc.) the dedicated route is the only option. See the docs URL above for full parameter and schema details.

- **`/measurements`** — video, web, network, speedtest, or conferencing measurements. Filter by `type` (`VideoMeasurement`, `WebMeasurement`, `NetworkMeasurement`, `SpeedtestMeasurement`, `ConferencingMeasurement`), `subject`, `study_id`.
- **`/client_reports`** — auxiliary reports captured alongside measurements (Netflix and YouTube "Stats for Nerds", ICMP Ping, Traceroute, P.1203 input data including the video session representation, etc.). Filter by `type` (e.g. `P1203ClientReport`, `NetflixClientReport`, `YoutubeClientReport`, `IcmpPingClientReport`).
- **`/event_bundles`** — individual events (quality switches, stalling, etc.) with timestamps. Rarely needed; aggregate stats from `/measurements` usually suffice. Requires `type` (`VideoEventBundle` or `WebEventBundle`).
- **`/network_requests`** — Chrome DevTools Protocol network data, only present if explicitly captured during the measurement (off by default).
- **`/clients`** — registered clients (uuid, type, label, tags, registration key, timestamps).
- **`/user_feedbacks`** — user feedback submitted from the QoE Test App.

### Common parameters across endpoints

- `start_time`, `end_time` — ISO-8601 filter on `created_at`. Accepted formats include `2023-05-17T12:36:13.000Z`, `20230517T12:36:13`, `2023-05-17 12:36:13 UTC`.
- `measurement_id` — fetch a single measurement (overrides everything else).
- `client_uuid` — filter by client (overrides tags).
- `tags` — `tags[]=foo&tags[]=bar` or `tags=foo,bar` (all must match).
- `label` — filter by client label.

### Pagination

`?page=2&per_page=100`, max `per_page=1000`. Response headers: `Per-Page`, `Total`, and RFC 8288 `Link`.

## Conceptual Schema

### Measurement types and their `study.subject`

- **VideoMeasurement** — streaming quality (`youtube`, `netflix`, etc.)
- **WebMeasurement** — web performance (`website`, `lighthouse`)
- **NetworkMeasurement** — network diagnostics (`dns`, `traceroute`, `icmppping`, `ooklaspeedtest`, `tlsmeter`)
- **SpeedtestMeasurement** — browser bandwidth tests (`fastcom`, `ookla`, `breitbandmessung`, `nperf`)
- **ConferencingMeasurement** — video conferencing quality (`googlemeet`, `microsoftteams`)

### Key video quality fields under `statistic_values`

Bitrate and resolution: `average_audio_bitrate`, `average_video_bitrate`, `average_total_bitrate`, `initial_resolution`, `largest_played_video_size`, `longest_played_chunk`.

Loading and stalling: `initial_loading_delay`, `video_response_time`, `number_of_stalling_events`, `total_stalling_time`, `average_stalling_time`, `stalling_ratio`, `total_play_time`.

Quality switches: `number_of_quality_switches`, `quality_switch_up_count`, `quality_switch_down_count`.

ITU-T P.1203 QoE model: `p1203_overall_mos` (overall MOS), `p1203_max_theoretical_mos`, `p1203_overall_audiovisual_quality`, `p1203_average_video_quality`, `p1203_average_audio_quality`, `p1203_stalling_quality`.

Some customers attach domain-specific context under `study.metadata` (e.g. flight, vessel, or site identifiers). When such fields exist, you can filter on them via Elasticsearch like any other field. If the user references such metadata and you don't know the schema, ask them or fetch the customer-specific documentation.

## Best Practices

1. For measurement queries and aggregations, default to `/search`; only hit `/measurements` when you genuinely need raw records.
2. Non-measurement resources are not in Elasticsearch — use their dedicated routes.
3. Filter at the API level rather than fetching everything and filtering locally.
4. Cache large pulls locally as `.json` or `.json.gz`.
5. Use `.keyword` for exact text matches in `/search` queries.
6. Respect the 1,000-element pagination cap.

## Minimal Python Example

```python
#!/usr/bin/env python3
# /// script
# dependencies = ["requests"]
# ///
import os, requests
from datetime import datetime, timedelta, timezone

customer = "your-customer"
base_url = f"https://surfmeter-server.{customer}-analytics.aveq.info/export_api/v1"
headers = {"Accept": "application/json", "X-API-KEY": os.environ["EXPORT_API_KEY"]}

end = datetime.now(timezone.utc)
start = end - timedelta(days=7)

r = requests.get(
    f"{base_url}/measurements",
    headers=headers,
    params={"type": "VideoMeasurement",
            "start_time": start.isoformat(),
            "end_time": end.isoformat(),
            "per_page": 1000},
)
r.raise_for_status()
print(f"Retrieved {len(r.json())} measurements")
```

## Surfmeter Management API (read-write)

The Surfmeter API is the HTTP interface behind the Admin Dashboard. Unlike the Export API it can both read and write, and it covers resources the Export API does not (anomaly events, license usage, capabilities, ISPs, users, keys, data-owner settings).

### Full API Reference

Endpoint-level details (parameters, request/response schemas) are documented at:

<https://docs.aveq.info/surfmeter-docs/api/surfmeter-api/>

The complete OpenAPI 3.1 spec is a single YAML file at <https://docs.aveq.info/surfmeter-docs/assets/openapi/surfmeter_api-v1.yml> — fetch it for exact field schemas. Locally it lives at `surfmeter/docs/surfmeter-docs/docs/assets/openapi/surfmeter_api-v1.yml`.

### Base URL

```
https://surfmeter-server.<customer>-analytics.aveq.info/client_admin_api/v1
```

The `client_admin_api` prefix is a historical Rails namespace; it is the stable public URL. For FOO use `https://surfmeter-server.surfmeter.telekom-dienste.de/client_admin_api/v1`. All requests and responses are JSON.

### Authentication

Same header as the Export API: `X-API-KEY`. Keys are created in the Dashboard under **Keys → Surfmeter API Keys** (`https://surfmeter-server.<customer>-analytics.aveq.info/app/client-admin-api-keys`). Each key inherits the role of the user who created it, so a request can do exactly what that user can do in the dashboard. This is a different key type from the Export API key — a `client_admin-...` key, not an `EXPORT_API_KEY`. Use `SURFMETER_API_KEY` from `.env` if present; otherwise ask the user.

```bash
curl -H "Accept: application/json" \
     -H "X-API-KEY: $SURFMETER_API_KEY" \
     "https://surfmeter-server.<customer>-analytics.aveq.info/client_admin_api/v1/clients"
```

Errors return a JSON `message`; validation errors (HTTP 422) add a structured `errors` object. Auth failures are 401, authorization failures 403, unknown resources 404. Collection endpoints accept `page`/`per_page` (max 1000) and return a top-level `meta` with `current_page`, `total_pages`, `total_count`.

### Resources at a glance

Read-only (GET): `/version`, `/measurement_options`, `/license_usage` (with `?extended=true` for the monthly breakdown), `/data_owner/anomaly_detection_config`, `/ai_assistant/status`, `/ai_assistant/usage`.

Read-write (GET plus POST/PUT/PATCH/DELETE):

- **`/clients`**, **`/clients/{id}`** — list and update clients. `PUT /clients/{id}` modifies `label`, `extra_label`, `tags`, `client_group_ids`. `POST /clients/{id}/disable` disables a client. `GET /clients/check_label_availability`.
- **`/client_groups`**, **`/client_groups/{id}`** — manage client groups.
- **`/clients/{client_id}/isp_contracts`**, **`/isps`** — ISPs and per-client ISP contracts.
- **`/client_admin_users`**, **`/client_admin_users/{id}`** (+ `/email_change`) — dashboard users.
- **`/export_api_keys`**, **`/client_admin_api_keys`** — create/revoke API keys of both kinds.
- **`/registration_keys`**, **`/capabilities`** — client registration keys and capability profiles.
- **`/measurements/search`**, **`/measurements/{id}`** (+ `/p1203_client_report`, `/network_performance_report`) — search and fetch measurements.
- **`/anomaly_events/search`**, **`/anomaly_events/{anomaly_id}`**, **`/anomaly_events/resolve`** — search, fetch, and bulk-resolve anomaly events.
- **`/data_owner/settings`** — read and update organization-wide settings.

### Searching measurements and anomalies (Elasticsearch)

`POST /measurements/search` and `POST /anomaly_events/search` take the same Elasticsearch Query DSL passthrough as the Export API's `/search`. The Query DSL must be wrapped in a top-level `body` key — posting the raw ES query directly returns `{"error": "Missing body"}`. Alongside `body`, the top-level `source_only`, `size`, `from`, `search_after` (and `scroll` for measurements) control the response shape and pagination. The video quality fields under `statistic_values` and the `.keyword` rule above (silent empty results on text-field `term` filters; loud `Fielddata is disabled` errors on text-field aggregations) apply identically here.

```bash
# Aggregate anomaly events by probe label for the latest cycle.
# Note: every text field carries .keyword — in the filter, the terms aggs, and any sort.
curl -s -X POST \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "X-API-KEY: $SURFMETER_API_KEY" \
  "https://surfmeter-server.<customer>-analytics.aveq.info/client_admin_api/v1/anomaly_events/search" \
  -d '{
    "body": {
      "size": 0,
      "query": { "bool": { "filter": [
        { "term":  { "measurement_type.keyword": "NetworkMeasurement" } },
        { "range": { "last_seen_at": { "gte": "now-1h" } } }
      ] } },
      "aggs": { "by_probe": {
        "terms": { "field": "client_label.keyword", "size": 200 },
        "aggs": { "cats": { "terms": { "field": "diagnostic_category.keyword" } } }
      } }
    }
  }'
```

Anomaly events live in `anomaly_events-*` indices and are not in the Export API at all — searching here is the only HTTP way to reach them. The documents key off `client_label`, `measurement_type`, `subject`, and the `first_seen_at` / `last_seen_at` dates; to see the rest, pull one sample (`{"body": {"size": 1}}`).

### Resolving anomaly events

`PATCH /anomaly_events/resolve` closes one or more active anomaly episodes (as if the analyzer's automatic resolution sweep had run). The body is `{"anomaly_ids": [...]}` where each id is the Elasticsearch `_id` of the episode (`{rule_id}::{detection_pass}::{first_seen_epoch}`, found in search hits). Idempotent: already-resolved or unknown ids silently no-op. Max 500 ids per call; the response reports `{updated, requested}` (`updated < requested` is still success).

### Updating a client

```bash
curl -s -X PUT \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "X-API-KEY: $SURFMETER_API_KEY" \
  "https://surfmeter-server.<customer>-analytics.aveq.info/client_admin_api/v1/clients/123" \
  -d '{"label": "new-label", "tags": ["fleet-a", "berlin"]}'
```
