# opencode-go-export

Unofficial documentation for the **OpenCode Console usage export API** — a CSV export of OpenCode Go / Console usage that is no longer listed in the public docs (the old page `opencode.ai/v2/docs/console/usage` now returns 404).

Verified against the live API on **2026-09-27**.

## TL;DR

```bash
curl --fail-with-body --get \
  "https://opencode.ai/console/api/v2/usage/export" \
  --header "Authorization: Bearer ${SERVICE_API_KEY}" \
  --header "Accept: text/csv" \
  --data-urlencode "range=7d" \
  --output "usage-7d.csv"
```

## Authentication

- Send the key as `Authorization: Bearer <key>`.
- The key must be a **service account API key** created in the [Console](https://opencode.ai/console) (`oc_sk_...`). User session tokens are rejected.
- A valid key without sufficient permissions gets `403`. Per the current Console docs for the sibling [budgets API](https://opencode.ai/v2/docs/console/budgets/), *"Inference-only keys cannot access budget data"* — usage export appears to require a service account with broader ("All") permissions.
- Invalid, expired, or revoked keys get `401`.

## Endpoint

```
GET https://opencode.ai/console/api/v2/usage/export
```

### Query parameters

| Parameter | Required | Values | Meaning |
| --- | --- | --- | --- |
| `range` | yes | `7d`, `30d`, `90d` | Whole UTC days, including today. Not a rolling window. |
| `user_id` | no | `user_...` or `svcacct_...` | Restrict to one member or one service account. Omit for the whole organization. |

The behavior annotation in the Console web app describes it as:

> Streams daily usage totals, newest day first, with one row per UTC day, member or service account, provider, and model. `range` covers whole UTC days including today. `user_id` accepts a member or service account ID. Totals come from daily rollups and can lag recent requests; the endpoint returns 503 until rollups are available. The operation is read-only and safe to retry.

Notes:

- The whole range is returned as a **single streamed CSV**; there is no pagination.
- `range=24h` is **not** valid on v2 (it was on the deprecated v1 endpoint). Anything else returns `400`.

### Examples

Last 7 days, whole organization:

```bash
curl --fail-with-body --get \
  "https://opencode.ai/console/api/v2/usage/export" \
  --header "Authorization: Bearer ${SERVICE_API_KEY}" \
  --header "Accept: text/csv" \
  --data-urlencode "range=7d" \
  --output "usage-org-7d.csv"
```

Last 90 days:

```bash
curl --fail-with-body --get \
  "https://opencode.ai/console/api/v2/usage/export" \
  --header "Authorization: Bearer ${SERVICE_API_KEY}" \
  --header "Accept: text/csv" \
  --data-urlencode "range=90d" \
  --output "usage-org-90d.csv"
```

One member or service account:

```bash
curl --fail-with-body --get \
  "https://opencode.ai/console/api/v2/usage/export" \
  --header "Authorization: Bearer ${SERVICE_API_KEY}" \
  --header "Accept: text/csv" \
  --data-urlencode "range=30d" \
  --data-urlencode "user_id=svcacct_..." \
  --output "usage-service-account-30d.csv"
```

Drop `--output` to print the CSV to stdout.

## Response

Observed header (2026-09-27):

```csv
day,user_type,user_id,user_name,provider,model,requests,input_tokens,output_tokens,cache_read_tokens,cache_write_5m_tokens,cache_write_1h_tokens,cost_micro_cents,last_active_at
```

| Column | Description |
| --- | --- |
| `day` | UTC date the usage is attributed to (`YYYY-MM-DD`). Rows are newest day first. |
| `user_type` | `user` or `service_account`. |
| `user_id` | Member ID (`user_...`) or service account ID (`svcacct_...`). |
| `user_name` | Display name / email of the member or service account. |
| `provider` | Provider catalog key, e.g. `opencode-go`, `opencode`, `anthropic`, `deepseek`. |
| `model` | Model identifier reported for the requests. |
| `requests` | Number of model requests in that day/group. |
| `input_tokens` | Uncached input tokens. |
| `output_tokens` | Output tokens **including reasoning tokens**. |
| `cache_read_tokens` | Prompt tokens served from cache. |
| `cache_write_5m_tokens` | Tokens written to a 5-minute prompt cache. |
| `cache_write_1h_tokens` | Tokens written to a 1-hour prompt cache. |
| `cost_micro_cents` | Amount charged, in **microcents**. `100,000,000` microcents = `$1` USD. Free/failed requests are `0`. |
| `last_active_at` | ISO 8601 UTC timestamp of the latest request in the group. |

Handy derivations:

```bash
# total cost in USD
awk -F, 'NR>1 { c += $13 } END { printf "%.4f USD\n", c/100000000 }' usage-7d.csv

# cache hit rate (share of prompt tokens served from cache)
awk -F, 'NR>1 { hit += $10; denom += $8 + $10 + $11 + $12 } END { printf "%.1f%%\n", 100*hit/denom }' usage-7d.csv

# per-model cache hit rate
awk -F, 'NR>1 { hit[$6] += $10; denom[$6] += $8 + $10 + $11 + $12 } END {
  for (m in hit) printf "%-32s %.1f%%\n", m, 100*hit[m]/denom[m]
}' usage-7d.csv
```

## Errors

| Status | Meaning | Typical cause |
| --- | --- | --- |
| `400` | Invalid request | Missing/unknown `range`, or unsupported value such as `24h`. |
| `401` | Unauthorized | Missing, invalid, expired, or revoked service account key. |
| `403` | Forbidden | Valid key without usage-read permission (e.g. an inference-only service account). |
| `503` | Rollups unavailable | Daily rollups are not ready yet; retry later. |

Use `--fail-with-body` (as above) so these produce a non-zero exit code while still printing the error body. Error bodies are JSON, e.g. `{"_tag":"Unauthorized"}` / `{"_tag":"Forbidden"}`.

## Deprecated v1 endpoint

```
GET https://opencode.ai/console/api/v1/usage/export
```

This is the shape that the removed docs page described. It accepts `scope` + `range` (`24h`, `7d`, `30d`, midnight UTC start) with scope-specific filters:

| Scope | Additional parameters | Result |
| --- | --- | --- |
| `organization` | none | All usage in the workspace. |
| `member` | `user_email` | Usage attributed to one member. |
| `service_account` | `service_account_id` | Usage attributed to one service account. |
| `model` | `provider` and `model` | Usage for one provider/model pair. |

**v1 no longer works for migrated organizations.** The Console web app annotates it as:

> Deprecated: use `/api/v2/usage/export`. Available only to organizations that have not yet moved to v2; others receive 403.

So: if v1 returns `403` for every valid scope, use v2.

## How this was discovered

- The public docs page `https://opencode.ai/v2/docs/console/usage/` was removed (archived copy: <https://web.archive.org/web/20260922102914/https://opencode.ai/v2/docs/console/usage/>).
- The current Console web app bundle (`/console/assets/index-*.js`) contains the live API definitions for both endpoints, including their schemas, permission rules, and behavior notes quoted above.
- The examples and columns in this README were executed/observed against the live API.

## Unverified / open questions

- Exact permission label required for usage export on a service account. Empirically, a key that can read the budgets API and export v2 usage works; test your own with the error table above.
- Whether v2 ever emits non-inference rows (the v1 docs mentioned `web-search` service rows in org/member exports); none were observed in the 7d export used here.
- Column stability is not guaranteed; this is an undocumented endpoint.

## Disclaimer

Not affiliated with OpenCode / Anomaly. This documents an undocumented, internal-facing API that may change or disappear at any time.
