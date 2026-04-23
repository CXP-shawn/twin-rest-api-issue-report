# Twin 公开 REST API Bug 报告 — openapi server URL 错误、start-run 不可用、probe 端点 UNIMPLEMENTED

## 摘要

在基于 Twin 团队提供的权威 openapi schema 集成 Twin 公开 REST API 时，我们实测发现三个致命问题。合在一起，任何外部 x-api-key 调用方都无法通过 openapi 声明的接口闭合 `present_options` 人机交互回路：

1. openapi 声明 `servers[0].url = https://builder.twin.so`，但该域实际是 Twin Web UI 前端 SPA（CloudFront + S3）。真正接受 `x-api-key` 的 REST/gRPC 网关在 `https://build.twin.so`。
2. `POST /v1/agents/{agent_id}/runs` 对 `run_mode` 的任意字符串值都返回 `400 "run_mode must be explicitly set to Builder, Run, or SubAgent"`；即使 openapi 声明 `run_mode: string | null`（没有 enum）、即使我们传的就是错误消息里列出的三个字面量之一。
3. 8 个"提交用户输入"候选端点里，6 个（`/messages`、`/input`、`/answer`、`/reply`、`/respond`、`/resume`）被 gateway 路由但返回 `HTTP 200 + grpc-status: 12`（UNIMPLEMENTED）；1 个仅在 DELETE 上连线；openapi 里根本没有"提交答复"类端点。

## 严重程度

**高** — 阻断任何使用 `present_options` 的外部 REST 集成（即任何人机交互工作流）。纯 CRUD 集成不受影响。

## 测试环境

| 字段 | 取值 |
|---|---|
| 测试日期 (UTC) | 2026-04-23 |
| 客户端 | 等价 curl 调用（内部动态 HTTP 工具），shell `base64` 编码 |
| 鉴权 | `x-api-key: ***REDACTED***`（build.twin.so 明示不支持 Bearer）|
| 测试账号 | 已脱敏 |
| 测试过的 base | `https://api.twin.so` / `https://builder.twin.so` / `https://build.twin.so` |
| 总请求次数 | 52 (v1=32 + v2=20)，本文只引用 v2 |

---

## 发现 1 · openapi `servers[0].url` 指向的是 SPA 前端而非 API 网关

openapi 文档里写：

```json
"servers": [ { "url": "https://builder.twin.so", "description": "Production" } ]
```

但我们在 `builder.twin.so` 上探测的所有路径全部返回 Twin Web UI HTML（AmazonS3 托管 SPA，CloudFront 对未知路径 fallback 到 `index.html`）：

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


真正的 API host `build.twin.so` 的 CORS header `access-control-allow-origin: https://builder.twin.so`（见 v2 trace seq 4-20）反向印证：`builder` 是浏览器 origin，`build` 是后端。

**影响**：任何严格按 openapi 集成的开发者，请求会全部打到 CDN，返回 200 + HTML，没有任何 API 语义。这是**静默错误**——连 4xx 都不会给。

## 发现 2 · `POST /v1/agents/{agent_id}/runs` 拒收任何 `run_mode` 值

openapi 声明：

```json
"StartRunBody": {
  "properties": {
    "run_mode":          { "type": ["string","null"] },
    "skip_deploy_check": { "type": ["boolean","null"] },
    "user_message":      { "type": ["string","null"] }
  }
}
```

没有 enum。实测服务器对我们传的每个字符串都返回 `400 Bad Request`，`detail: "run_mode must be explicitly set to Builder, Run, or SubAgent"`：

**Evidence (api_trace_v2 seq 6)** — 尝试 1: run_mode=Run

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","user_message":"开始 present_options 回路实验"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request"}`
- **Timestamp**: 2026-04-23T09:36:00Z


**Evidence (api_trace_v2 seq 7)** — 尝试 2: run_mode=Builder

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Builder","user_message":"测试 Builder 模式"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}`
- **Timestamp**: 2026-04-23T09:36:44Z


**Evidence (api_trace_v2 seq 8)** — 尝试 3: run_mode=Run + skip_deploy_check=true

```bash
curl -sS -X POST -H 'x-api-key: ***REDACTED***' -H 'Content-Type: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs' \
  -d '{"run_mode":"Run","skip_deploy_check":true,"user_message":"极简 body 第 3 次尝试"}'
```

- **HTTP Status**: `400`
- **Response headers (key ones)**: `{"content-type":"application/problem+json"}`
- **Response body** (truncated): `{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent"}`
- **Timestamp**: 2026-04-23T09:37:10Z


（v1 我们额外试过 `{"run_mode": {"Run": {}}}`、`{"run_mode": {"Run": null}}`、`{"run_mode": 1}`、`{"run_mode": {"type": "Run"}}` 以及 `{}`。前三者返回 `422 invalid type: <map|integer>, expected a string`，证实 serde 反序列化期望 string；后两者与上表同样 400。）

**影响**：外部 x-api-key 调用方无法启动 run。创建 agent（`POST /v1/agents` → 201）和写 instructions（`PUT .../instructions` → 200）都能成功，然后就走不下去了。

## 发现 3 · 6 个交互端点返回 `HTTP 200 + grpc-status: 12` UNIMPLEMENTED

gateway 明显是 HTTP-gRPC 桥接：即使普通 HTTP JSON 请求，响应 header 也带 `content-type: application/grpc` 和 `grpc-status` trailer。8 个"提交用户输入"候选路径里 6 个被路由但未实现：

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


PATCH run 资源明确只接受 DELETE：

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


复用 `POST /v1/agents/{id}/runs` + `run_mode=Run + run_id=...`（尝试 "resume via same endpoint"）撞上和发现 2 同样的墙：

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


**对照**：openapi 声明的 events 端点是真的实现了——即使对 placeholder `run_id` 也返回 `200 {"events":[],"total_count":0}`：

**Evidence (api_trace_v2 seq 9)** — GET events (placeholder run_id)

```bash
curl -sS -H 'x-api-key: ***REDACTED***' -H 'Accept: application/json' \
  'https://build.twin.so/v1/agents/019db9b1-ecad-7222-992d-9e0a03231ac9/runs/placeholder-run/events'
```

- **HTTP Status**: `200`
- **Response headers (key ones)**: `{"content-type":"application/json"}`
- **Response body** (truncated): `{"events":[],"total_count":0}`
- **Timestamp**: 2026-04-23T09:37:10Z


所以 gateway 不是大面积未实现，它就是**恰好对交互输入 RPC 留了一个洞**。

## 发现 4 · openapi 完全没有"提交用户输入"端点

openapi 里 run 相关端点只有：

- `POST /v1/agents/{agent_id}/runs`（启动）
- `GET /v1/agents/{agent_id}/runs`（列出）
- `DELETE /v1/agents/{agent_id}/runs/{run_id}`
- `POST /v1/agents/{agent_id}/runs/{run_id}/cancel`
- `GET /v1/agents/{agent_id}/runs/{run_id}/events`

没有任何端点让调用方回复 `present_options` interrupt。runtime 在 `WaitingForInput` 状态暂停后，外部集成方没有任何 API 可以把它唤醒。

## 复现步骤

```bash
KEY="<your-twin-api-key>"

# 1. 验证 openapi 声明的 server 指向 SPA
curl -sS -i https://builder.twin.so/v1/me      # 返回 200 text/html (AmazonS3)

# 2. 在真实 API host 创建 agent
curl -sS -X POST -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents -d '{"is_subagent": false}'   # → 201 返回 agent_id

AID=<粘贴 agent_id>

# 3. 写 instructions
curl -sS -X PUT -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents/$AID/instructions -d '{"content": "ping"}'   # → 200

# 4. 尝试启动 run — 所有变体都失败
curl -sS -X POST -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents/$AID/runs -d '{"run_mode":"Run","user_message":"hi"}'
# → 400 "run_mode must be explicitly set to Builder, Run, or SubAgent"

# 5. 尝试回传端点 — gRPC UNIMPLEMENTED
curl -sS -i -X POST -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  https://build.twin.so/v1/agents/$AID/runs/any-run-id/input -d '{"value":"x"}'
# → HTTP 200, grpc-status: 12

# 6. 清理
curl -sS -X DELETE -H "x-api-key: $KEY" https://build.twin.so/v1/agents/$AID   # → 204
```

## 对照表（openapi 声明 vs 实测）

| 方面 | openapi 声明 | 实测行为 |
|---|---|---|
| Base URL | `https://builder.twin.so` | 该域是 UI SPA。真实 API 在 `https://build.twin.so` |
| `StartRunBody.run_mode` | `string | null`，无 enum | 必须是字面量 `"Builder"/"Run"/"SubAgent"` 之一，但传这些字符串仍拒绝 |
| 回复 `present_options` | —（未声明） | 8 个候选路径里 6 个 gRPC UNIMPLEMENTED；无替代方案 |

## 建议的修复方案

**a.** 修正 openapi 里 `servers[0].url`，指向真实 API 网关（`https://build.twin.so` 或正式生产 hostname）。或者让后端自己吐 openapi，杜绝漂移。

**b.** 给 `StartRunBody.run_mode` 加 enum `["Builder", "Run", "SubAgent"]`，把它标 required 或定义 omit 时的默认值；同时修反序列化器，让 openapi 声明的字符串真能被接受——当前错误消息自相矛盾（"must be Builder/Run/SubAgent" 但传这些字符串仍然失败）。

**c.** 要么把 `/runs/{rid}/messages|input|answer|reply|respond|resume` 背后的 gRPC 方法真的实现（让 probe 不再 grpc-status 12），**要么**从 gateway 路由表删掉它们；然后在 openapi 里把官方的"提交用户输入"端点声明出来，给集成方闭合 `present_options` 回路的第一类方法。

## 附录

脱敏后的完整 trace JSON（`x-api-key` 与任何 token 类字段已替换）：

- [`appendix-traces.json`](./appendix-traces.json) — 20 条来自 `api_trace_v2`
- [`openapi-citations.md`](./openapi-citations.md) — 逐字引用本文涉及的 openapi 片段
- [`bug-report-en.md`](./bug-report-en.md) — 英文版

---

本报告于 2026-04-23 由外部集成测试产出，不含任何私密账号数据。
