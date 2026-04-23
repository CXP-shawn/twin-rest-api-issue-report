# twin-rest-api-issue-report

外部集成方对 Twin 公开 REST API 做了一轮实测，发现上几项会阻断 `present_options` 人机交互回路的问题。本仓库提交 bug 报告给 Twin 工程团队。

## 核心问题速览

1. **openapi `servers[0].url` 指向错误域名** — 指向 Web UI SPA（`builder.twin.so`），不是真实 API 网关（`build.twin.so`）。
2. **`POST /v1/agents/{id}/runs` 的 `run_mode` 合法值是 `"run"` / `"build"`（小写）** — 服务端 400 错误消息写的是 `"Builder"/"Run"/"SubAgent"`，但那些字面量**全部被拒绝**。详见实测报告。
3. **6 个"提交用户输入"端点 `HTTP 200 + grpc-status: 12` UNIMPLEMENTED** — gateway 路由但底层 handler 未实现。

## 文件清单

| 文件 | 内容 |
|---|---|
| [`bug-report-en.md`](./bug-report-en.md) | English bug report (for Twin engineering team) |
| [`bug-report-zh.md`](./bug-report-zh.md) | 中文 bug 报告（逐节对应英文版） |
| [`openapi-citations.md`](./openapi-citations.md) | 引用的 openapi 关键片段 |
| [`appendix-traces.json`](./appendix-traces.json) | 完整脱敏 HTTP trace（20 条记录） |
| [`run-mode-empirical-test.md`](./run-mode-empirical-test.md) | **run_mode 字段合法值 18 变体实测报告**（揭示真正合法值是小写 `"run"`/`"build"`，错误消息自相矛盾） |

## 测试环境

- 测试日期 (UTC)：2026-04-23
- 客户端：curl-equivalent 等价调用
- 鉴权：`x-api-key` header（已脱敏）
- 测试过的 base URL：`api.twin.so`、`builder.twin.so`、`build.twin.so`
- 总请求次数：70 (v1=32 + v2=20 + v3 run_mode 矩阵=18)

## 相关实验原始报告

本 bug report 背后的完整实测 trace 与分析见外部仓库：
<https://github.com/CXP-shawn/twin-rest-api-live-test-report-cn/blob/main/09-present-options-roundtrip.md>

## License / 许可

本报告以 CC-BY 4.0 发布，欢迎 Twin 团队直接复用。
