# Twin Public REST API — openapi server URL, start-run unusable, probe endpoints UNIMPLEMENTED

## Summary

While integrating against the Twin public REST API (openapi schema shared by the Twin team), we found three blocking issues that together make it impossible for an external x-api-key holder to complete the `present_options` interactive round-trip through the documented surface:

1. The openapi document declares `servers[0].url = https://builder.twin.so`, but that host is in fact the Twin Web UI SPA (CloudFront + S3). The REST/gRPC gateway that actually accepts `x-api-key` lives at `https://build.twin.so`.
2. `POST /v1/agents/{agent_id}/runs` rejects every value of `run_mode` with `400 "run_mode must be explicitly set to Builder, Run, or SubAgent"`, even though the openapi declares `run_mode: string | null` with no enum and we pass the exact strings named in the error.
3. Six of the plausible "submit user input" endpoints (`/messages`, `/input`, `/answer`, `/reply`, `/respond`, `/resume`) are routed by the gateway but return `HTTP 200 + grpc-status: 12` (UNIMPLEMENTED); one is only wired up for DELETE; no endpoint in openapi lets the caller submit an answer to a `present_options` interrupt.

## Severity

**High** — blocks any external REST API integration that uses `present_options` (i.e. any real human-in-the-loop workflow). CRUD-only integrations are unaffected.

## Environment

| Field | Value |
|---|---|
| Test date (UTC) | 2026-04-23 |
| Client | curl-equivalent via internal dynamic HTTP tools, shell `base64` for encoding |
| Auth | `x-api-key: ***REDACTED***` header (Bearer explicitly unsupported, per server response on build.twin.so) |
| Test account | redacted |
| Twin API base(s) tested | `https://api.twin.so`, `https://builder.twin.so`, `https://build.twin.so` |
| Total HTTP requests | 52 (32 in v1 + 20 in v2); only v2 is cited below |

---

## Finding 1 · openapi `servers[0].url` points to the SPA, not to the API gateway

The openapi document shared with users declares:

```json
"servers": [ { "url": "https://builder.twin.so", "description": "Production" } ]
```

However every path we probed on `builder.twin.so` returns the Twin Web UI HTML (AmazonS3-served SPA, with CloudFront fallback to `index.html`):

**Evidence (api_trace_v2 seq 1)** — builder-spa-discovery

```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://builder.twin.so/v1/me'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"server":"AmazonS3","content-type":"text/html; charset=utf-8","content-length":"3769"}`
- **Response body** (truncated): `<!doctype html>\n<html lang="en">... (Twin UI SPA 首页 HTML, content-length=3769)`
- **Timestamp**: 2026-04-23T09:35:20Z


**Evidence (api_trace_v2 seq 2)** — builder-api-path-probe-1

```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://builder.twin.so/openapi.json'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"server":"AmazonS3","content-type":"text/html"}`
- **Response body** (truncated): `<!doctype html>... (相同 SPA HTML)`
- **Timestamp**: 2026-04-23T09:35:29Z


**Evidence (api_trace_v2 seq 3)** — builder-api-path-probe-2

```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://builder.twin.so/api/v1/me'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"server":"AmazonS3","content-type":"text/html"}`
- **Response body** (truncated): `<!doctype html>... (相同 SPA HTML)`
- **Timestamp**: 2026-04-23T09:35:29Z


CORS headers on the actual API host `build.twin.so` (`access-control-allow-origin: https://builder.twin.so`, see every seq 4-20 of our trace) confirm that `builder` is the browser origin and `build` is the backend.

**Impact**: any integrator who faithfully follows the openapi document will send every request to a CDN that will return 200 OK with HTML and no API semantics. The bug is silent — not even a 4xx to signal the mistake.

## Finding 2 · `POST /v1/agents/{agent_id}/runs` rejects every `run_mode` value

Openapi declares:

```json
"StartRunBody": {
  "properties": {
    "run_mode":          { "type": ["string","null"] },
    "skip_deploy_check": { "type": ["boolean","null"] },
    "user_message":      { "type": ["string","null"] }
  }
}
```

No enum. In practice, the server rejects every string we send with `400 Bad Request`, `detail: "run_mode must be explicitly set to Builder, Run, or SubAgent"`:

**Evidence (api_trace_v2 seq 6)** — Try 1: run_mode=Run

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","user_message":"开始 present_options 回路实验"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request"}`
- **Timestamp**: 2026-04-23T09:36:00Z


**Evidence (api_trace_v2 seq 7)** — Try 2: run_mode=Builder

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Builder","user_message":"测试 Builder 模式"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}`
- **Timestamp**: 2026-04-23T09:36:44Z


**Evidence (api_trace_v2 seq 8)** — Try 3: run_mode=Run + skip_deploy_check=true

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","skip_deploy_check":true,"user_message":"极简 body 第 3 次尝试"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}`
- **Timestamp**: 2026-04-23T09:37:10Z


(In v1 we additionally tried `{"run_mode": {"Run": {}}}`, `{"run_mode": {"Run": null}}`, `{"run_mode": 1}`, `{"run_mode": {"type": "Run"}}` and `{}`. The first three get `422 invalid type: <map|integer>, expected a string`, confirming serde wants a string; the last two get the same 400 as above.)

**Impact**: no external x-api-key caller can start a run. Creating the agent (`POST /v1/agents` → 201) and writing instructions (`PUT .../instructions` → 200) both succeed, then the flow dead-ends.

## Finding 3 · Six interactive endpoints return `HTTP 200 + grpc-status: 12` UNIMPLEMENTED

The gateway is clearly an HTTP-gRPC bridge: the response headers contain `content-type: application/grpc` and a `grpc-status` trailer even on a vanilla HTTP JSON request. Six of the plausible "submit user input" paths are routed but not implemented:

**Evidence (api_trace_v2 seq 10)** — probe-A

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/messages' \
  -d '{"content":"红"}'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/grpc","grpc-status":"12","content-length":"0"}`
- **Response body** (truncated): `(空 body，grpc-status=12 UNIMPLEMENTED)`
- **Timestamp**: 2026-04-23T09:37:23Z


**Evidence (api_trace_v2 seq 11)** — probe-B

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/input' \
  -d '{"value":"红"}'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/grpc","grpc-status":"12"}`
- **Response body** (truncated): `(空 body，grpc-status=12)`
- **Timestamp**: 2026-04-23T09:37:23Z


**Evidence (api_trace_v2 seq 12)** — probe-C

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/answer' \
  -d '{"answer":"红"}'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/grpc","grpc-status":"12"}`
- **Response body** (truncated): `(空 body，grpc-status=12)`
- **Timestamp**: 2026-04-23T09:37:23Z


**Evidence (api_trace_v2 seq 15)** — probe-F

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/reply' \
  -d '{"reply":"红"}'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/grpc","grpc-status":"12"}`
- **Response body** (truncated): `(空 body，grpc-status=12)`
- **Timestamp**: 2026-04-23T09:37:23Z


**Evidence (api_trace_v2 seq 16)** — probe-G

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/respond' \
  -d '{"response":"红"}'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/grpc","grpc-status":"12"}`
- **Response body** (truncated): `(空 body，grpc-status=12)`
- **Timestamp**: 2026-04-23T09:37:23Z


**Evidence (api_trace_v2 seq 17)** — probe-H

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/resume' \
  -d '{"input":"红"}'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/grpc","grpc-status":"12"}`
- **Response body** (truncated): `(空 body，grpc-status=12)`
- **Timestamp**: 2026-04-23T09:37:23Z


PATCH on the run resource is explicitly wired up only for DELETE:

**Evidence (api_trace_v2 seq 13)** — probe-D (PATCH)

```bash
curl -sS -X PATCH -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run' \
  -d '{"user_input":"红"}'
```

- **HTTP Status**: `405`
- **Response headers (key ones)**: `{"allow":"DELETE","content-length":"0"}`
- **Response body** (truncated): `(空 body，Allow: DELETE)`
- **Timestamp**: 2026-04-23T09:37:23Z


Re-using `POST /v1/agents/{id}/runs` with `run_mode=Run + run_id=...` (the "resume via same endpoint" hypothesis) hits the same Finding 2 wall:

**Evidence (api_trace_v2 seq 14)** — probe-E

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","run_id":"placeholder-run","user_message":"红"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}`
- **Timestamp**: 2026-04-23T09:37:23Z


**Contrast**: the events endpoint, which is in openapi, is implemented — even for a placeholder `run_id` it returns `200 {"events":[],"total_count":0}`:

**Evidence (api_trace_v2 seq 9)** — GET events (placeholder run_id)

```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/events'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/json"}`
- **Response body** (truncated): `{"events":[],"total_count":0}`
- **Timestamp**: 2026-04-23T09:37:10Z


So the gateway is not blanket-unimplemented; it has a specific gap on interactive-input RPCs.

## Finding 4 · openapi declares no "submit user input" endpoint

The run-related endpoints declared in the openapi we received are:

- `POST /v1/agents/{agent_id}/runs` (start)
- `GET /v1/agents/{agent_id}/runs` (list)
- `DELETE /v1/agents/{agent_id}/runs/{run_id}`
- `POST /v1/agents/{agent_id}/runs/{run_id}/cancel`
- `GET /v1/agents/{agent_id}/runs/{run_id}/events`

There is no documented endpoint to reply to a `present_options` interrupt. When the runtime pauses the run on `WaitingForInput`, external integrators have no API to unblock it.

## Reproduction Steps

```bash
KEY="<your-twin-api-key>"

# 1. Confirm the openapi-declared server is the SPA
curl -sS -i https://builder.twin.so/v1/me      # returns 200 text/html (AmazonS3)

# 2. Create an agent on the real API host
curl -sS -X POST -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents -d '{"is_subagent": false}'   # → 201 with agent_id

AID=<paste agent_id>

# 3. Write instructions
curl -sS -X PUT -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents/$AID/instructions -d '{"content": "ping"}'   # → 200

# 4. Try to start a run — every variant fails
curl -sS -X POST -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents/$AID/runs -d '{"run_mode":"Run","user_message":"hi"}'
# → 400 "run_mode must be explicitly set to Builder, Run, or SubAgent"

# 5. Try the reply endpoints — gRPC UNIMPLEMENTED
curl -sS -i -X POST -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents/$AID/runs/any-run-id/input -d '{"value":"x"}'
# → HTTP 200, grpc-status: 12

# 6. Cleanup
curl -sS -X DELETE -H "x-api-key: $KEY" https://build.twin.so/v1/agents/$AID   # → 204
```

## Expected vs Actual

| Area | openapi says | Actual behavior |
|---|---|---|
| Base URL | `https://builder.twin.so` | That host is the UI SPA. Real API is `https://build.twin.so` |
| `StartRunBody.run_mode` | `string | null`, no enum | Requires literally `"Builder"/"Run"/"SubAgent"`, yet still rejects those exact strings |
| Reply to `present_options` | — (not declared) | 6 plausible paths return gRPC UNIMPLEMENTED; no alternative exists |

## Suggested Fixes

**a.** Fix `servers[0].url` in the openapi document to point to the real API gateway (`https://build.twin.so` or whatever the canonical production hostname is). Or publish openapi from the backend itself so it can never drift.

**b.** Add the enum `["Builder", "Run", "SubAgent"]` to `StartRunBody.run_mode`. Either make it required, or define the default behavior when omitted. Either way, fix the deserializer so the documented strings are accepted — today the error message is self-contradictory ("must be Builder/Run/SubAgent" when we send exactly those).

**c.** Either implement the interactive-input gRPC method(s) that back `/runs/{rid}/messages|input|answer|reply|respond|resume` (so probes stop returning grpc-status 12), **or** remove those routes from the gateway. Then declare the official "submit user input" endpoint in openapi so integrators have a first-class way to close the `present_options` loop.

## Appendix

Complete de-identified trace JSON (`x-api-key` and any token-like fields redacted):

- [`appendix-traces.json`](./appendix-traces.json) — 20 records from `api_trace_v2`
- [`openapi-citations.md`](./openapi-citations.md) — verbatim quotes of the openapi fragments cited above
- [`bug-report-zh.md`](./bug-report-zh.md) — Chinese translation of this report

---

Report produced on 2026-04-23 by an external integration test. No private account data is included.
