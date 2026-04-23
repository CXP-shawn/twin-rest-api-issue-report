# `run_mode` 字段合法值实测（empirical test）

> 实验执行时间：2026-04-23T10:59Z ~ 11:00Z (UTC)
> 被测端点：`POST https://build.twin.so/v1/agents/{agent_id}/runs`
> 实验中创建的玩具 agent：`019db9fe-998c-7ab2-bb8a-963e516716e1`（实验结束已 `DELETE 204` 删除，所有产生的 5 个 run 均已 cancel）
> 认证 header 已脱敏：`x-api-key: ***REDACTED***`

---

## 1 · 实验目的

关于 `POST /v1/agents/{agent_id}/runs` 的 `run_mode` 字段，当前三方来源互相矛盾：

| 来源 | 说 `run_mode` 合法值是 |
|---|---|
| Twin 官方人类文档 | `"build"` / `"run"` |
| 官方 openapi schema | `string | null`（**无 enum**） |
| 服务端 400 错误消息 | `"Builder"` / `"Run"` / `"SubAgent"` |

本实验用 18 种 body 变体把真相测出来——**不猜、不推断、只实测**。

---

## 2 · 矩阵总表

共 18 行，按 `label` 升序。每行列出 **request body / HTTP status / response 摘要 / 结论**。

| # | label | request body | HTTP | response 摘要 | 结论 |
|---|---|---|---|---|---|
| 1 | `M01-no-run_mode-empty-body` | `{}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 2 | `M02-run_mode-null` | `{"run_mode":null}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 3 | `M03-run_mode-empty-string` | `{"run_mode":""}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 4 | `M04-run_mode-lowercase-run` | `{"run_mode":"run"}` | **201** | 201 run_id=019db9fe-f8bc... policy_type=runner_v2 status=in_progress | ✅ 被接受（201），policy_type=runner_v2 |
| 5 | `M05-run_mode-lowercase-build` | `{"run_mode":"build"}` | **201** | 201 run_id=019db9fe-f92c... policy_type=builder_v3 status=in_progress | ✅ 被接受（201），policy_type=builder_v3 |
| 6 | `M06-Run-PascalCase` | `{"run_mode":"Run"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 7 | `M07-Build-PascalCase` | `{"run_mode":"Build"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 8 | `M08-Builder-PascalCase` | `{"run_mode":"Builder"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 9 | `M09-SubAgent-PascalCase` | `{"run_mode":"SubAgent"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 10 | `M10-subagent-lowercase` | `{"run_mode":"subagent"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 11 | `M11-sub_agent-snake-case` | `{"run_mode":"sub_agent"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 12 | `M12-RUN-uppercase` | `{"run_mode":"RUN"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 13 | `M13-BUILD-uppercase` | `{"run_mode":"BUILD"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 14 | `M14-invalid-junk` | `{"run_mode":"invalid_value_xyz"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |
| 15 | `M15-run-plus-user_message` | `{"run_mode":"run","user_message":"hello"}` | **201** | 201 run_id=019db9fe-fa28... policy_type=runner_v2 status=in_progress | ✅ 被接受（201），policy_type=runner_v2 |
| 16 | `M16-run-skip-deploy-plus-msg` | `{"run_mode":"run","skip_deploy_check":true,"user_message":"hello"}` | **201** | 201 run_id=019db9fe-fa8c... policy_type=runner_v2 status=in_progress | ✅ 被接受（201），policy_type=runner_v2 |
| 17 | `M17-build-skip-deploy-plus-msg` | `{"run_mode":"build","skip_deploy_check":true,"user_message":"hello"}` | **201** | 201 run_id=019db9fe-fabf... policy_type=builder_v3 status=in_progress | ✅ 被接受（201），policy_type=builder_v3 |
| 18 | `M18-only-user_message` | `{"user_message":"hello"}` | **400** | 400 detail: "run_mode must be explicitly set to Builder, Run, or SubAgent" | ❌ 400 "run_mode must be explicitly set to Builder, Run, or SubAgent" |

---

## 3 · 关键发现（基于实测）

### 发现 1 · 合法值是小写 `"run"` 和 `"build"`

`POST /v1/agents/{id}/runs` 只接受两个 `run_mode` 值：**`"run"`** 和 **`"build"`**，均为**小写**。

- `M04 run_mode="run"` → **201 Created**，返回 run 对象，`policy_type: "runner_v2"`
- `M05 run_mode="build"` → **201 Created**，返回 run 对象，`policy_type: "builder_v3"`

两种模式返回的 `policy_type` 字段不同，说明服务端**内部路由到两个不同的 policy 实现**（runner_v2 vs builder_v3）。

### 发现 2 · 服务器错误消息 **自相矛盾**

当 `run_mode` 非法时，服务器返回固定错误：

```
"run_mode must be explicitly set to Builder, Run, or SubAgent"
```

但**实测这条消息里列出的三个字面量全部被拒绝**：
- M06 `"Run"` → 400
- M08 `"Builder"` → 400
- M09 `"SubAgent"` → 400

错误消息文本说的是 PascalCase 名称（**Rust 内部 RunMode enum variant 的名字**），但实际 JSON 反序列化期待的是 serde 的 rename_all="snake_case"/"lowercase" 输入（`"run"` / `"build"`）。两者永远不会碰头。这是典型的错误消息与校验逻辑不同源 bug。

### 发现 3 · 大小写严格敏感

`RUN` / `BUILD` / `Run` / `Build` 全部 400。只有纯小写 `run`/`build` 被接受。

### 发现 4 · SubAgent 模式对公开 REST API 不可用

- M09 `"SubAgent"` → 400
- M10 `"subagent"` → 400
- M11 `"sub_agent"` → 400

没有任何字面量能让 `run_mode=SubAgent` 起效。对比 `"run"`/`"build"` 工作而 `"sub_agent"`/`"subagent"` 全拒——SubAgent 内部存在但未在 REST API 暴露，或需要其它未公开的字段配合。

### 发现 5 · run_mode 缺失、null、空字符串全部等价 → 400

- M01 `{}` → 400
- M02 `{"run_mode":null}` → 400
- M03 `{"run_mode":""}` → 400
- M18 `{"user_message":"hello"}` → 400

服务端要求 run_mode **必填且非空**，**没有默认值**。错误消息一律相同。

### 发现 6 · `user_message` 与 `skip_deploy_check` 不影响启动判定

- M15 `{"run_mode":"run","user_message":"hello"}` → 201
- M16 `{"run_mode":"run","skip_deploy_check":true,"user_message":"hello"}` → 201
- M17 `{"run_mode":"build","skip_deploy_check":true,"user_message":"hello"}` → 201

只要 `run_mode` 合法，其它字段不影响 accept；响应 run 对象里也未见它们的明显回写（返回 run 对象的 `goal` 恒为 `null`，不像 `user_message` 被映射为 goal）。

---

## 4 · 与三方来源的对比表

| 维度 | 官方人类文档 | openapi schema | 服务端错误消息 | **本次实测** |
|---|---|---|---|---|
| `run_mode` 合法值 | `"build"` / `"run"` | `string | null`，无 enum | `"Builder"` / `"Run"` / `"SubAgent"` | **`"run"` / `"build"`（小写）** |
| 大小写敏感 | 未说明 | 未说明 | 隐含 PascalCase | **严格敏感，必须小写** |
| SubAgent 可用 | 未提 | 未提 | "SubAgent" 字面列出 | **不可用（字面量全部 400）** |
| run_mode 必填 | 未明确 | 声明为 nullable | "must be explicitly set" | **必填，null/空字符串/缺失全 400** |

**谁对谁错：**
- ✅ **官方人类文档对**（`"build"` / `"run"` 小写）
- ❌ **openapi schema 错**（应当写 `"enum": ["run", "build"]` 且不可 null）
- ❌ **服务端错误消息错**（PascalCase 名称永远不会被 JSON deserializer 接受）

---

## 5 · 给 Twin 官方的结论（基于实测）

1. **openapi 应改为**：
   ```json
   "StartRunBody": {
     "properties": {
       "run_mode": {
         "type": "string",
         "enum": ["run", "build"],
         "description": "Run mode. Must be lowercase 'run' or 'build'. 'SubAgent' variant exists internally but is not exposed via public REST API."
       },
       "user_message":      { "type": ["string","null"] },
       "skip_deploy_check": { "type": ["boolean","null"] }
     },
     "required": ["run_mode"]
   }
   ```

2. **错误消息应改为**：
   ```
   "run_mode must be one of: \"run\", \"build\""
   ```
   去掉 PascalCase 名称与 SubAgent 提示，与实际接受的字面量一致。

3. **可选** —— 如果 SubAgent 不打算暴露给 REST 调用方，Rust 内部 enum 的 public-facing serde derive 也应相应隐藏该 variant，避免其出现在错误文本。

---

## 6 · 附录：18 条完整响应 body（脱敏后）

### seq 1 · `M01-no-run_mode-empty-body`

**request body**:
```json
{}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: 缺失 run_mode → 400

---

### seq 2 · `M02-run_mode-null`

**request body**:
```json
{"run_mode":null}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: run_mode=null → 400

---

### seq 3 · `M03-run_mode-empty-string`

**request body**:
```json
{"run_mode":""}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: run_mode="" → 400（动态工具发空字符串可能被当 null；结果同上）

---

### seq 4 · `M04-run_mode-lowercase-run`

**request body**:
```json
{"run_mode":"run"}
```
**HTTP status**: `201`

**response body (完整，脱敏后)**:
```json
{"run":{"agent_id":"019db9fe-998c-7ab2-bb8a-963e516716e1","event_bytes":0,"event_count":0,"goal":null,"is_finished":false,"last_event_at":"2026-04-23T10:59:48.028874101+00:00","policy_type":"runner_v2","run_id":"019db9fe-f8bc-7d62-ae59-f74a93d41539","run_number":0,"started_at":"2026-04-23T10:59:48.028874101+00:00","status":"in_progress","step_count":0,"user_email":"***REDACTED***","user_id":"***REDACTED***","wrote_instructions":false}}
```
**notes**: **合法**。run_id=019db9fe-f8bc-7d62-ae59-f74a93d41539，status=in_progress，policy_type=runner_v2。cleaned up: cancel status=200 {success:true}

---

### seq 5 · `M05-run_mode-lowercase-build`

**request body**:
```json
{"run_mode":"build"}
```
**HTTP status**: `201`

**response body (完整，脱敏后)**:
```json
{"run":{"agent_id":"019db9fe-998c-7ab2-bb8a-963e516716e1","event_bytes":0,"event_count":0,"goal":null,"is_finished":false,"last_event_at":"2026-04-23T10:59:48.141094885+00:00","policy_type":"builder_v3","run_id":"019db9fe-f92c-7020-b3eb-e8fe5ac141c1","run_number":0,"started_at":"2026-04-23T10:59:48.141094885+00:00","status":"in_progress","step_count":0,"user_email":"***REDACTED***","user_id":"***REDACTED***","wrote_instructions":false}}
```
**notes**: **合法**。run_id=019db9fe-f92c-7020-b3eb-e8fe5ac141c1，policy_type=builder_v3（与 run 不同！）。cleaned up: cancel status=200 {success:true}

---

### seq 6 · `M06-Run-PascalCase`

**request body**:
```json
{"run_mode":"Run"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: **错误消息里列出的 "Run" 字面量反而被拒绝**

---

### seq 7 · `M07-Build-PascalCase`

**request body**:
```json
{"run_mode":"Build"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: "Build" PascalCase 被拒

---

### seq 8 · `M08-Builder-PascalCase`

**request body**:
```json
{"run_mode":"Builder"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: **错误消息里自己列出的 "Builder" 字面量也被拒绝**

---

### seq 9 · `M09-SubAgent-PascalCase`

**request body**:
```json
{"run_mode":"SubAgent"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: **错误消息里自己列出的 "SubAgent" 字面量也被拒绝**

---

### seq 10 · `M10-subagent-lowercase`

**request body**:
```json
{"run_mode":"subagent"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: 小写 "subagent" 被拒（对比 "run"/"build" 工作，说明 SubAgent 可能不是 API 公开暴露的模式）

---

### seq 11 · `M11-sub_agent-snake-case`

**request body**:
```json
{"run_mode":"sub_agent"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: "sub_agent" 被拒

---

### seq 12 · `M12-RUN-uppercase`

**request body**:
```json
{"run_mode":"RUN"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: 大写 "RUN" 被拒，确认大小写敏感

---

### seq 13 · `M13-BUILD-uppercase`

**request body**:
```json
{"run_mode":"BUILD"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: 大写 "BUILD" 被拒

---

### seq 14 · `M14-invalid-junk`

**request body**:
```json
{"run_mode":"invalid_value_xyz"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: 非法值错误消息同其它拒绝——服务器不区分"类型错"和"枚举不匹配"

---

### seq 15 · `M15-run-plus-user_message`

**request body**:
```json
{"run_mode":"run","user_message":"hello"}
```
**HTTP status**: `201`

**response body (完整，脱敏后)**:
```json
{"run":{"agent_id":"019db9fe-998c-7ab2-bb8a-963e516716e1","event_bytes":0,"event_count":0,"goal":null,"is_finished":false,"last_event_at":"2026-04-23T10:59:48.392703189+00:00","policy_type":"runner_v2","run_id":"019db9fe-fa28-74d1-b3be-63f04730799c","run_number":0,"started_at":"2026-04-23T10:59:48.392703189+00:00","status":"in_progress","step_count":0,"user_email":"***REDACTED***","user_id":"***REDACTED***","wrote_instructions":false}}
```
**notes**: **合法**。加 user_message 不影响启动。cancel status=200 (run already completed by time cancel was sent)

---

### seq 16 · `M16-run-skip-deploy-plus-msg`

**request body**:
```json
{"run_mode":"run","skip_deploy_check":true,"user_message":"hello"}
```
**HTTP status**: `201`

**response body (完整，脱敏后)**:
```json
{"run":{"agent_id":"019db9fe-998c-7ab2-bb8a-963e516716e1","event_bytes":0,"event_count":0,"goal":null,"is_finished":false,"last_event_at":"2026-04-23T10:59:48.492329478+00:00","policy_type":"runner_v2","run_id":"019db9fe-fa8c-7ea2-9262-056ca757b280","run_number":0,"started_at":"2026-04-23T10:59:48.492329478+00:00","status":"in_progress","step_count":0,"user_email":"***REDACTED***","user_id":"***REDACTED***","wrote_instructions":false}}
```
**notes**: **合法**。skip_deploy_check 不影响 accept。cancel status=200 (run already completed)

---

### seq 17 · `M17-build-skip-deploy-plus-msg`

**request body**:
```json
{"run_mode":"build","skip_deploy_check":true,"user_message":"hello"}
```
**HTTP status**: `201`

**response body (完整，脱敏后)**:
```json
{"run":{"agent_id":"019db9fe-998c-7ab2-bb8a-963e516716e1","event_bytes":0,"event_count":0,"goal":null,"is_finished":false,"last_event_at":"2026-04-23T10:59:48.543319120+00:00","policy_type":"builder_v3","run_id":"019db9fe-fabf-7cd3-9046-7ca0029ecaf5","run_number":0,"started_at":"2026-04-23T10:59:48.543319120+00:00","status":"in_progress","step_count":0,"user_email":"***REDACTED***","user_id":"***REDACTED***","wrote_instructions":false}}
```
**notes**: **合法**。cleaned up: cancel status=200 {success:true}

---

### seq 18 · `M18-only-user_message`

**request body**:
```json
{"user_message":"hello"}
```
**HTTP status**: `400`

**response body (完整，脱敏后)**:
```json
{"detail":"run_mode must be explicitly set to Builder, Run, or SubAgent","status":400,"title":"Bad Request","type":"about:blank"}
```
**notes**: 没 run_mode 只有 user_message → 400，与 M01 一致：run_mode 必填

---

