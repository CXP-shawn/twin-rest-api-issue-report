# openapi-citations.md

本文件逐字引用 Twin 官方 openapi schema 里与本 bug report 相关的 3 段关键片段，并在每段下说明为什么它与实测矛盾。所有引用均来自用户提供的官方 openapi 文档。

---

## 引用 1 · `servers[0].url`

```json
{
  "servers": [
    { "url": "https://builder.twin.so", "description": "Production" }
  ]
}
```

**矛盾点**：对该 URL 的任何路径发起 HTTPS 请求都会返回 AmazonS3 托管的 Twin Web UI SPA 首页（`content-type: text/html`，`server: AmazonS3`，content-length=3769 字节的 index.html）。真正承载 `x-api-key` REST API 的是 `build.twin.so`。详见 [`bug-report-en.md#finding-1`](./bug-report-en.md) 以及 trace seq 1-3。

---

## 引用 2 · `StartRunBody` schema

```json
"StartRunBody": {
  "properties": {
    "run_mode":          { "type": ["string", "null"] },
    "skip_deploy_check": { "type": ["boolean", "null"] },
    "user_message":      { "type": ["string", "null"] }
  },
  "type": "object"
}
```

**矛盾点**：
1. openapi 里 `run_mode` 是 `string | null`，**没有 enum**，没有列出合法值；
2. 实际服务端业务校验要求 `run_mode` 的字面值必须是 `"Builder"` / `"Run"` / `"SubAgent"` 三者之一（从错误消息反推）；
3. 但即使我们精确传这三个字符串中的任何一个，服务器仍返回 `400 "must be explicitly set to Builder, Run, or SubAgent"`——错误消息自相矛盾。详见 [`bug-report-en.md#finding-2`](./bug-report-en.md) 以及 trace seq 6-8。

---

## 引用 3 · run 相关端点总览（openapi 声明）

openapi 声明的 run 相关路径只有 5 条：

```
POST   /v1/agents/{agent_id}/runs                         # start run
GET    /v1/agents/{agent_id}/runs                         # list runs
DELETE /v1/agents/{agent_id}/runs/{run_id}                # delete run
POST   /v1/agents/{agent_id}/runs/{run_id}/cancel         # cancel run
GET    /v1/agents/{agent_id}/runs/{run_id}/events         # stream events
```

**矛盾点**：`present_options` 是 Twin runtime 原生的"暂停 run、等待用户输入"机制。runtime 进入 `WaitingForInput` 后，openapi 列表里**没有任何端点**让外部调用方把用户答复传回来 —— 既没有 `POST .../messages`、也没有 `POST .../input`、也没有 `POST .../answer`、也没有 `PATCH /runs/{rid}` 的 resume 语义。实测 6 种常见命名（`messages`/`input`/`answer`/`reply`/`respond`/`resume`）在 gateway 路由层存在但全部返回 `grpc-status: 12 UNIMPLEMENTED`（详见 [`bug-report-en.md#finding-3`](./bug-report-en.md) 以及 trace seq 10-17）。

---

## 其它旁证

`GET /v1/agents/{id}/runs/{rid}/events` 对 placeholder `run_id` 返回 `200 {"events":[],"total_count":0}`（见 trace seq 9），证实 events 端点是真正实现的 REST 端点——这就是 Finding 3 的对照：gateway 不是大面积未实现，而是恰好漏掉了交互输入 RPC。
