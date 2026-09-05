# Runtime 会话所有权与交接 — 技术交接

> 产品取舍见 [docs/insights/runtime-thread-ownership-and-handoff.md](../insights/runtime-thread-ownership-and-handoff.md)
>
> 实施计划与未完成真实 smoke 见 [docs/exec-plans/active/runtime-thread-ownership-and-handoff.md](../exec-plans/active/runtime-thread-ownership-and-handoff.md)

## 解决的问题

旧实现允许一个聊天原地切 Claude Code、CodePilot、Codex。三者的续聊机制不同：Claude 重包装 DB 历史，Native 重放消息，Codex 续自己的 thread。界面看起来还是同一聊天，底层可能已经丢前文或回到旧分支；创建 session 后再 PATCH Runtime 还会产生半套 route。

现在的合同是：第一次真实执行接受时锁定 Runtime owner；Composer 左侧 Runtime lane 同步置灰；同 Runtime 变化由 capability 决定。跨 Runtime 的 handoff 后端仍可创建带交接包的新聊天，但普通 Picker 不再触发它或突然跳转；重新开放时必须使用独立、明确且带确认的入口。

## 数据模型

`chat_sessions` 新增：

- `runtime_binding_state`: `unbound | bound | legacy_unbound`
- `runtime_bound_at`
- `runtime_binding_source`
- `route_revision`: 只为 route/binding 变化递增的 CAS 版本

`runtime_pin` 在 `bound` 状态下就是 owner。完整 route 为 `runtime_pin + provider_id + model`。

另有两张 additive 表：

- `chat_session_handoffs`: 来源、目标、source rowid boundary、版本化 payload 与幂等键
- `chat_session_compaction_events`: 触发来源、覆盖边界、辅助 route、估算节省和底层 session 是否重建

关键规则：route transaction 一次最多把 revision 加一；消息、标题、原生 thread ref、handoff 来源只读均不加。跨 Runtime 拒绝、stale CAS 和 no-op 全部零写入。

## 执行入口

`src/lib/runtime/thread-execution-binding.ts` 提供 legacy 分类与 trigger 决策。`POST /api/chat` 在 resolve Provider、读取执行历史和启动 child 之前完成以下裁决：

1. `bound`: 只使用 owner，忽略当前全局默认；请求 Runtime 不一致则拒绝。
2. `unbound + manual/bridge`: 完整 route 已持久化时 CAS 绑定。
3. `unbound + auto/retry/queue`: fail closed。
4. `legacy_unbound`: 所有执行 fail closed，等用户显式 recovery。

助理、heartbeat、task、bridge、导入和派生 worktree 等系统入口在创建 session 时保存明确 route 并直接绑定。managed child 要求父 session 已绑定，child Runtime 必须与父 owner 一致，但 child Provider+Model 不改父 route。

## 原子 route 与 continuation

已有聊天用 `PATCH /api/chat/sessions/:id/route`，请求必须带完整 route 和 `expected_route_revision`。通用 session PATCH 收到 route 字段会返回 `ATOMIC_ROUTE_REQUIRED`。

多窗口并发时，输掉 CAS 的客户端收到 `ROUTE_REVISION_CONFLICT` 及权威 session 快照。`ChatView` 会同步最新 Runtime、Provider、Model、binding state、owner 与 revision，保留本次草稿且不产生 optimistic 消息；用户再点一次发送即可，无需刷新。自动 catalog/model 纠正只更新 Composer 本地状态，不提前写 route revision。

`src/lib/runtime/continuation-policy.ts` 当前矩阵：

| Runtime | 换模型 | 换 Provider | 底层行为 |
|---|---|---|---|
| Claude Code | `replay_context` | `replay_context` | 清 Claude SDK ref，使用 canonical DB history 建新底层 session |
| CodePilot | `replay_context` | `replay_context` | DB history 重放；无原生 session ref 要清 |
| Codex | `in_session` | `new_session` | 同 thread 可换模型；Provider 变化走 handoff 新聊天 |

跨 Runtime 永远是 `new_session`。这个矩阵是服务端事实，Picker 只消费 owner Runtime 内的结果，不按名字猜；bound 会话的 Runtime lane disabled，不能把 `new_session` 偷偷伪装成一次普通下拉选择。

## Handoff

`POST /api/chat/sessions/:id/handoff` 接受目标完整 route、source route revision、preview 返回的 source boundary 和幂等 key。transaction 会验证来源已绑定且 idle、route/boundary 未前进，然后创建一个已绑定目标和 handoff record；来源完全不变。

该 API 当前不是 Composer 普通 Runtime lane 的交互出口。2026-09-01 实机测试确认“选择后直接跳到新聊天”缺少动作预期，因此普通 Picker 已移除自动 handoff；未来调用方必须提供独立文案、缓存重建说明与用户确认。

payload v1 是确定性的最近 16 条 eligible transcript window，最多约 18K 字符：

- 排除 heartbeat ack 与内部 Runtime marker；
- 附件 metadata 只保留“已省略”事实；
- 文本经过通用 telemetry sanitizer；
- 只保存 project basename，不保存绝对 workspace 路径；
- 明确记录 boundary、是否截断和 `recent_transcript` 来源。

目标第一次执行将其编译为 `<runtime_handoff>` fragment。三 Runtime 消费同一事实，不复制 Claude SDK id 或 Codex thread id。目标 UI 显示来源链接、覆盖边界和缓存需重建提示。

## Usage 与费用真实性

assistant 消息仍保留历史 `TokenUsage` 外壳，同时新增 `normalized.schemaVersion=2`：Runtime、Provider、Model、未缓存输入、cache read/write、输出、费用和来源均为可选事实。

- Claude：SDK 给什么 bucket 就存什么。
- Native：`input_tokens` 是总输入；只有 provider raw usage 明确带 read/write 字段时才接受 AI SDK detail 并推导 uncached。SDK 合成的两个 0 不算证据。
- Codex：总输入减 cached 得 uncached；不虚构 cache write。

统计只有在 cohort 内每个 usage turn 都有足够 source 时才显示费用或 cache rate。缺一轮就显示 unknown/partial coverage；历史 v1 标 legacy，不反向补 Runtime 或费用。

## Compaction

`RuntimeCompactionPolicy` 当前为：Claude/Native `proactive`，Codex `reactive_only/runtime_managed`。80% 主动阈值保留。

manual、auto、reactive 三入口通过 `commitSessionCompaction()` 原子写 summary、coverage boundary 和事件。Claude proactive/reactive 会重建 SDK session；Native 保留；Codex 不使用 CodePilot summary 冒充 thread 压缩。刷新后 session GET 会返回最近 compaction 事实，UI 继续显示边界和缓存影响。

## 关键文件

- `src/lib/runtime/thread-execution-binding.ts`
- `src/lib/runtime/continuation-policy.ts`
- `src/lib/runtime/route-validation.ts`
- `src/lib/runtime/handoff-payload.ts`
- `src/lib/runtime/turn-usage.ts`
- `src/lib/runtime/compaction-policy.ts`
- `src/lib/runtime/runtime-display.ts`
- `src/lib/db.ts`
- `src/app/api/chat/route.ts`
- `src/app/api/chat/sessions/[id]/route/route.ts`
- `src/app/api/chat/sessions/[id]/handoff/route.ts`
- `src/components/chat/ChatView.tsx`

## 验证与剩余边界

单元合同覆盖 migration、CAS、零写入、handoff 幂等/并发、marker、child gate、usage unknown 和 compaction policy。真实三 Runtime account smoke 仍必须按执行计划 Smoke Ledger 跑；在完成之前不能宣称某个 Provider 的实际缓存命中率或费用下降百分比。
