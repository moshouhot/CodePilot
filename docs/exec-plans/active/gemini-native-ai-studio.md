# Gemini AI Studio Native 接入与能力补齐

> 创建时间：2026-09-18
> 最后更新：2026-09-18

## 用户目标与取舍

用户在三 Runtime 调研后明确授权先接 Native，同时对照 AI SDK 文档补齐 Native 的相关能力。产品目标是在官方 API 中添加 AI Studio key 后，能选择 Gemini 3.8 Flash 写作、调整思考深度、使用工具并持续修改文章。

采用已有 Google generateContent SDK；不切换服务端存储语义，不升级依赖。Gemini 此轮只开放 Native，Codex 与 Claude 暂不开放。补齐实际发现的历史状态、参数、输出截断缺口；已有 MCP、权限、来源展示和 usage 实现先核验，不重复建系统。

## 状态

| Phase | 用户可见变化 | 状态 | 验证 |
|---|---|---|---|
| 0 | 明确支持范围 | 已完成 | 官方网页 + 安装包内 AI SDK 文档/源码 |
| 1 | 官方 API 添加 AI Studio，Native 可选 3.8 Flash | Code complete | 预设/身份/Runtime 排除/目录 |
| 2 | 思考档位、工具与重开续聊完整工作 | Tests pass（离线） | 真 SDK + 模拟 Google SSE + 真 DB 多回合、Bridge owner gate |
| 3 | 能力审计与验证 | Tests pass；UI Smoke passed（fixture）；用户报告 Claude 已复核 | full 5584 pass / 1 skip，两份 spec 四 worker 并行 4 pass；真实 API 与 Windows packaged 尚未执行 |

## 设计与回归边界

- 使用独立 google 文本 preset，图像 Provider 与 Vertex 不变；不改既有默认 Provider。
- 精确 3.8 模型规则：low/medium/high，默认 medium，always thinking；剔除不支持的采样参数。
- Native 从 AI SDK response.messages 保存每 step 的重放数据，绑定 Provider/model；只在相同路线回放。用户可见文本仍使用现有 content blocks，后续未完成 step 不丢弃；旧记录/换路由继续普通历史重放。
- 新的隐藏元数据不进入 UI 正文、状态提示或遥测日志；collector 沿用 owner gate。
- 截断保留已有输出并明确提示，不能把 token 上限结束当作完整写作。
- Native 能力审计记录在本计划，不能把 SDK 支持列表直接标成产品已支持。

## AI SDK 能力审计与本轮结果

| 能力 | CodePilot 当前状态与本轮处理 |
|---|---|
| Google 文本/流式/工具 | 复用已安装 AI SDK 7 + Google 4；新增 AI Studio preset、Google 原生连接测试，避免误走 Anthropic |
| 思考与参数 | 3.8 精确映射 low/medium/high，默认 medium；关思考/过高档位有调整通知；所有 generateText/streamText 剔除 temperature/topP/topK 与旧 thinkingBudget |
| 多轮工具与签名 | 本轮补齐逐 step canonical messages 的桌面/bridge 持久化与相同路线恢复；跨路线只带可见历史；已完成 step 后的 partial tail 不丢弃 |
| 长输出 | 仅 3.8 上限从通用 16,384 提升至 65,536；所有 Native Provider 的 length 结束都保留正文、结束循环并显示桌面本地化通知（非持久化消息标记）。如果该 step 已执行工具，不自动继续下一 step |
| 长对话压缩 | 所有 Native Provider 均修复重新读 DB 时忽略摘要覆盖边界的问题；摘要合并首条 user、保留附件，覆盖行剔除。签名元数据不重复计入估算。仍保留原先最多 200 行历史读取，不宣称无限上下文 |
| 模型发现 | 完整分页、ID 去前缀、方法过滤、去重；失败无 partial diff，key 不进 URL/错误体 |
| 文件/终端/MCP/权限/取消 | 已有 Native tool loop 与权限/中止机制，继续复用；全量单测验证既有行为，无需另造 agent 框架 |
| 来源与 Google 托管搜索/URL Context | SDK 支持，不等于产品已接；当前 Google 未装配这些 provider tools，本轮未开放 |
| PDF/音视频 | Google 模型支持不等于 Native 消息输入支持；现有 replay 是文本/图片，本轮未添加 PDF 能力承诺 |
| Usage/缓存 | 沿用 SDK 真实 token 用量；Google cache 明细尚未完整映射，保持未知，不填假 0；本轮不扩展会计 schema |
| Interactions API | 已安装 SDK 有此能力，但改变服务端存储/续接语义；本轮保持 generateContent，无依赖升级 |

## 初轮验证记录（P2 follow-up 结果见文末）

- `npm run test`：typecheck、Harness boundary、5581 pass / 1 skip / 0 fail。
- `gemini-native.test.ts`：7 个行为回归，使用真实 AI SDK、模拟 Google wire、真实隔离 DB；包括完整工具 round-trip、正文签名、换路线、截断、摘要边界、bridge/stale owner、连接失败脱敏、发现分页。
- ESLint：新增文件无错误；触及既有大文件的检查保留已有 warnings，未扩大清理范围。
- Playwright `gemini-native.spec.ts`：1 passed（15.6s）；真实设置页添加/保存 AI Studio、连接测试请求协议、真实 models API 三 Runtime 过滤、聊天框选择 Gemini 与 1M 上下文显示。连接/发现网络使用 fixture，截图 `/tmp/gemini-native-composer.png` 已目视检查。
- `lint:hooks`、`lint:docs-drift`、`git diff --check` 通过。初次 UI 测试使用旧设置 query 路由及默认 Claude lane，修正为当前设置导航和显式 Native 后通过；不改变应用默认 Runtime。
- 不使用真实凭据；fixture 仅证明本地协议/持久化与 UI，不证明 Google 账户权限、配额、真实签名服务端校验或模型写作质量。

## 文档依据

- [AI SDK Google](https://ai-sdk.dev/providers/ai-sdk-providers/google-generative-ai)；网页抓取不稳定，使用已安装 `@ai-sdk/google/docs/15-google.mdx` 对照实际 4.0.6 实现。
- [AI SDK 文本生成](https://ai-sdk.dev/docs/ai-sdk-core/generating-text)、[工具调用](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling)。
- [Gemini 3.8 迁移](https://ai.google.dev/gemini-api/docs/latest-model)。
- [前期调研](../../research/gemini-ai-studio-three-runtimes-2026-09-18.md)。

## Smoke Ledger

| Date | Runtime | Provider | Model | 凭据形态 | 场景 | Result | Evidence |
|---|---|---|---|---|---|---|---|
| 2026-09-18 | Native | AI Studio | gemini-3.8-flash | 尚未使用真实 key | 真实模型写作/工具/续聊 | 未执行 | 不以离线 fixture 代替 |
| 2026-09-18 | Native | AI Studio | gemini-3.8-flash | fixture，无上游请求 | 添加服务/协议/真实过滤 API/聊天模型选择 | UI Smoke passed | `gemini-native.spec.ts` 1 pass；`/tmp/gemini-native-composer.png` |
| 2026-09-18 | Native | AI Studio | gemini-3.8-flash | 无 | Windows/macOS packaged 真实调用 | 未执行 | 本轮只验证 macOS 源码/本地 web UI |
| 2026-09-18 | Native / TokenDance 三 Runtime picker | AI Studio + TokenDance | fixture 模型 | fixture，无云端调用 | 两份 spec 四 worker 并行：添加/授权取消/模型选择/兼容筛选 | UI Smoke passed：4/4 | `/tmp/gemini-review-parallel.log`，15.8s；Gemini 使用 page-scoped 语言 |

## 决策日志

- 2026-09-18：用户授权实现，仅 Native；保留现有工作区中 #685 等未提交改动。
- 2026-09-18：Code complete + Tests pass；本地 UI fixture smoke 通过。未 commit/push/发版；真实 API smoke 保持待执行。

## Claude 独立审查跟进（2026-09-18）

用户回传审查：无 P1，P2-1 是压缩后连续 user wire 风险，P2-2 是 Gemini spec 写全局设置导致并行 TokenDance 页面失败。独立原版本 full 5581 pass / 1 skip、Gemini UI 1 pass、TokenDance 单独 3/3 不构成这两项关闭证据。

- **P2-1 Signal/Triage**：真实 Google SDK 的 converter 不合并连续 user。先补协议回归，在旧实现下 current-only / retained-text / retained-image 三项均失败，wire 实际出现连续 user；assistant-first 对照通过。未使用真实 Key，不能声称已复现 Google 服务端 400。
- **Fix**：摘要合并进首条 user 内容；字符串拼接、multipart 前置 text part，其余附件原样保留。历史以 assistant 开头时仍用独立摘要 user，保证不产生新的同角色相邻消息。不把摘要提升为 system，不改签名/工具历史。
- **P2-2 Fix**：移除测试对共享 locale/agent_runtime 的 PUT，仅在当前 Playwright page 的 GET 响应中覆盖显示语言；Runtime 由已有 picker 显式选 Native。拒绝只在 finally 还原全局设置的方案，因为并行窗口依然存在。
- **Verify**：定向 10/10 通过，四种压缩边界均检查真实 SDK wire 角色、摘要恰好一次、旧行排除；图片对照检查 inlineData 原字节未丢。两份 spec 四 worker 并行 4/4（15.8s）；`npm run test` typecheck + Harness boundary + 5584 pass / 1 skip / 0 fail（31.2s）。scoped ESLint 0 error / 4 处原有 warnings。日志：`/tmp/gemini-review-before.log`、`/tmp/gemini-review-targeted.log`、`/tmp/gemini-review-parallel.log`、`/tmp/gemini-review-full.log`。
- **Guardrail**：Runtime 明确摘要合并、多模态保留、压缩与 length 的 Native-wide 范围。非阻塞 P3 记录到 tech-debt #92–94；本轮不扩展传输/存储结构或测试 runner 重构。
- **残余**：真实 AI Studio 写作/工具/压缩后再发/签名校验，Windows packaged 多 Provider 三轮与重开仍待执行。#685 历史 upstream 无唯一 live 映射时，升级后仍需显式重选，不承诺全部自动恢复。

2026-09-18 用户补充“刚才修复那两个 Claude 复核了”。记录为用户已报告 Claude 完成复核；当前消息未附复核裁决原文，不虚构新增测试或扩大真实 smoke 的验证范围。以上修复与测试证据保持有效。

- 2026-09-18：用户明确授权正式发版；准备 v0.67.16，真实 API / Windows packaged 缺口写入 Release Notes，不以发布授权冒充 smoke 证据。

## v0.67.16 发布准备（2026-09-18）

用户明确授权发版，Jev 暂不接入。范围为 #685 路由身份、Gemini Native、TokenDance 卡片位置，以及工作区已有的保存失败/快捷建议修复；不新增依赖或改变分发拓扑。

- 最终全量：typecheck + Harness boundary + 5584 pass / 1 skip / 0 fail（28.4s），日志 `/tmp/codepilot-06716-test.log`。
- Gemini/TokenDance 隔离 UI：四 worker 4/4（16.2s），日志 `/tmp/codepilot-06716-ui.log`。
- `package.json` 与 lock 均为 0.67.16，仅版本字段变化；离线 npm install 因缓存缺失未完成，改用 npm version 同步，未刷新依赖解析。
- 管理员 API 实测 Immutable Releases enabled=true；main / stable-release-tags active、无 bypass/exclude、ID/更新时间与确认状态一致。确认日期已更新为 2026-09-18，18 个发布 Action 均使用精确 SHA。
- Release Notes 明示真实 Gemini 与 Windows packaged 缺口，不宣称该部分 Smoke passed；CI 将执行签名、公证、三平台启动与资产图门禁。尚未标记 Shipped。

- 保存失败提示独立隔离 UI：5/5 通过，日志 `/tmp/codepilot-06716-save-ui.log`；已清理本次 E2E 自动生成的 tsconfig 路径，保留原配置。
