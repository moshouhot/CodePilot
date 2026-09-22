# Windows 无签名原生自动更新与差分下载

> 创建时间：2026-08-26
> 最后更新：2026-08-26
> 当前状态：🚧 `v0.67.8` 不可变失败 tag 已保留，`v0.67.9` 修复候选正在门禁。Windows 采用用户确认过的“无 Authenticode、GitHub 为单一信任根”模型；Linux 保持手动安装并明确延期。仓库保护已启用，管理员 no-bypass 确认状态与 live ruleset 防漂移已补齐；Windows RC-A → RC-B 真实升级仍待完成。

## 用户问题与取舍

用户希望先交付 Windows 自动更新，不申请 Microsoft/Azure Trusted Signing，也不配置 PFX 发布者证书。此前讨论已经确认：Windows 的 per-user NSIS 安装不依赖管理员写入权限或证书；证书解决的是独立发布者身份，不是文件写入权限。用户接受不使用独立发布者签名，并以固定的 `op7418/CodePilot` GitHub Release、HTTPS、builder metadata 中的 SHA-512、不可变 Release、受保护 tag、最小 Actions 权限和完整资产审计作为当前信任模型。

这个取舍仍有明确残余风险：若仓库管理权限、Release workflow 和首次发布同时失守，攻击者可以一起替换 installer、metadata 和 hash；客户端不能像 Authenticode 那样用独立 publisher identity 识别这种情况。GitHub attestation 是人工/外部核验凭据，当前 `electron-updater` 不会在安装前自动验证它。

## 状态

| Phase | 用户会看到什么变化 | 状态 | 明确不做什么 |
|-------|--------------------|------|--------------|
| Phase 0 | 结论从聊天记录落成可审计合同 | ✅ 已完成 | 不申请微软签名，不把 SHA-512 宣称为发布者签名 |
| Phase 1 | Windows 正式包能发现、下载并在用户确认后安装新版本 | ✅ 本地完成 | 既有 `v0.67.7` 不会凭空获得 updater；第一跳仍需手动安装 bootstrap 版本 |
| Phase 2 | 后续 Windows 更新优先差分下载，失败时自动回退完整安装包 | ✅ 本地完成 | 不承诺固定节省比例；节省量由两个真实版本的 blockmap 决定 |
| Phase 3 | 发布者能看到仓库信任门禁和完整 Windows updater 资产审计 | ✅ 仓库设置与权限可见性修复完成 | 门禁未满足时不公开 `latest.yml`，不把 attestation 冒充客户端强制校验 |
| Phase 4 | Windows RC-A → RC-B 的真实升级结果可验收 | 🔄 `v0.67.8` 发布 gate fail；`v0.67.9` 候选与真实 RC 待执行 | Linux updater、包管理器提权安装与 Linux manifest 不进入本轮 |

## 执行清单

### Phase 0 — 决策与现状审计

- [x] 回看此前聊天记录，确认用户选择的是 unsigned Windows auto-update，而不是等待另一类 Windows 证书。
- [x] 核对当前仓库：Windows stable provenance=`0`，Release 排除 `latest.yml`/EXE blockmap；`v0.67.7` 只能手动 bootstrap。
- [x] 核对 `electron-updater@6.8.3` / `electron-builder@26.8.1` 精确实现：NSIS 默认支持 installer blockmap 差分，安装器会缓存 `installer.exe`，差分失败会回退完整下载。
- [x] 只读核对 GitHub：Immutable Releases=`false`；`main` ruleset=`disabled`；Actions 默认 token=`read`，但 third-party Action SHA pin 强制项=`false`。

### Phase 1 — Windows official updater 与诚实风险 UI

- [x] stable Windows 构建嵌入 official updater provenance；fork/local/Linux 继续 no-op。
- [x] unsigned 模式同时关闭 builder 强制签名与 publisher verification metadata，校验 installer/app 为 `NotSigned` 且 `app-update.yml` 不含 `publisherName`。
- [x] Main 从 packaged `app-update.yml` 读取 `publisherName` 生成真实 publisher breadcrumb；UI 仅在 `nsis + none` 显示 unsigned 提示，unknown 关闭 native updater，不能由平台常量猜测。
- [x] 保持 Main-owned feed、窄 IPC、活动任务 fail-closed、用户确认重启与手工 fallback。

### Phase 2 — 差分资产与回退

- [x] builder 显式生成 NSIS differential package；Main 显式保持 differential download 开启。
- [x] Windows artifact/checksum/attestation/release graph 包含同版本 `latest.yml`、NSIS installer 与 installer blockmap。
- [x] central verifier 要求 metadata URL 是当前版本裸 basename，校验 SHA-512、size、blockMapSize 和精确 checksum 集合；拒绝旧版本/额外 payload 与所有 Linux updater metadata。
- [x] source-contract 覆盖缺 metadata、缺 blockmap、错 hash、外部 URL、旧版本 payload、checksum 前缀蒙混、Linux feed 混入和 full-download fallback pin。

### Phase 3 — GitHub 单一信任根门禁

- [x] 所有 workflow 的 `uses:` 固定到完整 40 位 commit SHA，并由测试禁止恢复 mutable tag。
- [x] stable Release 发布前 fail-closed 核对：Immutable Releases 管理员确认只接受当天/前一 UTC 日；默认分支与 `v*` tag ruleset 各恰好一个、active、无 bypass/exclude，并分别禁止删除/非快进与删除/更新。
- [x] 保持 workflow 顶层 `contents: read`，仅 release job 获得 contents/OIDC/attestation 写权限，完整资产先 draft 后一次公开。
- [x] 用户授权发版后启用 active `main` 与 `stable-release-tags` ruleset，并设置新鲜的 Immutable Releases 管理员确认。
- [x] 修复 `v0.67.8` 正式 run 暴露的权限语义：Actions `GITHUB_TOKEN` 不会返回敏感 `bypass_actors`；管理员完整响应先生成 `CODEPILOT_RULESETS_CONFIRMED_STATE`，CI 再以 live `id` / `updated_at` 精确匹配，缺状态或任何漂移都 fail closed。

### Phase 4 — 验证与发布前人验

- [x] Claude 审查修复后重新完成 targeted tests、`npm run test`、production build、Electron bundle 与 workflow YAML 解析。
- [x] 首个 Windows updater 版本升级版本号时同步当版 `RELEASE_NOTES.md`；`v0.67.8` 未公开且不可复用，当前候选递增为 `v0.67.9`，不回写已发布的 `v0.67.7` 历史说明。
- [ ] Windows clean VM 手动安装 RC-A（bootstrap），保留自定义安装目录和用户数据。
- [ ] RC-A → RC-B 从 `latest.yml` 自动发现、下载；分别收集差分成功与 blockmap 不可用时完整下载回退证据。
- [ ] 更新下载后存在 active chat/bridge/task 时拒绝退出；idle 后确认重启，安装 handoff 成功且新版本可启动。
- [ ] SmartScreen 的真实表现记为产品限制，不把“可能提示”写成“不会提示”。

## 详细设计

### 信任边界

```text
official stable Windows app
  → 固定 GitHub provider: op7418/CodePilot
  → latest.yml（同一 draft→public Release）
  → installer SHA-512 + external blockmap
  → 下载完成后用户确认重启
  → unsigned NSIS 安装
```

`latest.yml` 与 installer 的一致性由 central audit 保证；HTTPS/GitHub、immutable Release、rulesets 与 SHA-pinned Actions 共同构成唯一信任根。客户端不会声称验证了 Microsoft/Authenticode publisher。

### Bootstrap 与差分语义

- 已发布的 `v0.67.7` 编译时 official updater provenance 为 `0`，不能通过远端 metadata 改造成 updater 客户端。因此用户必须手动安装首个开启 Windows updater 的版本。
- 标准 NSIS 安装器会把本次 installer 缓存为 updater cache 的 `installer.exe`。从该 bootstrap 版本开始，下一版本可以利用旧/新 blockmap 请求差异区间。
- “增量更新”是优先路径，不是独立的小补丁格式。blockmap 缺失、旧 installer 不可用或 Range 下载失败时，`electron-updater` 自动下载完整 installer；更新不能因差分不可用而永久卡住。

### 发布边界

- stable：macOS 保持签名/公证的 ZIP updater；Windows 增加 unsigned NSIS updater；Linux 仍只有手动包。
- preview：本轮不扩 Windows prerelease feed，避免在没有真实 Windows RC smoke 前扩大渠道。
- tag/Release 不覆盖、不移动、不复用；坏版本只发更高 patch。

## 验收标准

1. Windows official stable 包的 updater snapshot 为 `supported=true`、`packageType=nsis`、`publisherVerification=none`；签名 fixture 为 `authenticode`，缺失/畸形 packaged config 为 `publisher_verification_unknown` 并禁用 native；Linux 仍返回 `linux_trust_not_enabled`。
2. Release graph 同时有 `latest-mac.yml`、`latest.yml`、Mac ZIP blockmap 与 Windows EXE blockmap；没有任何 `latest-linux*.yml`。
3. unsigned Windows 包的两个 CodePilot EXE 均为 `NotSigned`，客户端配置无 `publisherName`，UI 明示没有独立发布者签名。
4. 仓库保护未启用、Immutable 确认过期、ruleset 有 exclude/重名时 release job 在公开 Release 前失败；main 禁删除/force-push，`v*` tag 禁删除/更新，不能留下用户可见的半套 feed。
5. 真实 Windows RC-A → RC-B 完成差分尝试、完整下载 fallback、活动任务拦截、安装接管和数据保留 smoke，之后才能标 `Release ready`。

## Smoke Ledger（真实凭据 / UI / E2E 验证记录）

| Date | Runtime | Provider | Model | 凭据形态 | 场景 | Result | Evidence |
|------|---------|----------|-------|---------|------|--------|----------|
| 2026-08-26 | GitHub repository | `op7418/CodePilot` | n/a | 只读 `gh api`，未读取 secret value | unsigned updater trust prerequisites | ❌ 发布阻断 | Immutable Releases=false；main ruleset disabled；Actions SHA policy=false；default workflow token=read |
| 2026-08-26 | local macOS workspace | Node 22 / Electron bundle | n/a | 无真实发布凭据 | Claude findings 修复后 source contract + 全量单测 | ✅ Tests pass | targeted 26/26；`npm run test` 5389 pass / 0 fail / 1 skip；首次沙箱运行的 localhost EPERM 通过沙箱外原命令复验消除 |
| 2026-08-26 | isolated local build | Next 16 / esbuild | n/a | official-build compile-time fixture | production build + Electron Main/Preload bundle | ✅ Build pass | 原工作区 dev lock 保留；隔离副本 `next build` 与 official updater bundle 成功，产物固定 differential=true / web installer=false / officialBuild=true，并包含 packaged publisher breadcrumb fail-closed 逻辑 |
| 2026-08-26 | GitHub Actions macOS + Intel + Windows + Linux | stable `v0.67.8` | n/a | 正式 Apple secrets；Windows signer 全缺；Actions `GITHUB_TOKEN` | 正式三平台 package + single-trust-root publish gate | ❌ Release blocked，未公开 | run [`32989409053`](https://github.com/op7418/CodePilot/actions/runs/32989409053)：source、Windows unsigned package、Linux 双架构、Mac 签名/公证/Intel ABI 全绿；release job 因 API 不暴露 `bypass_actors` 在公开前 fail-closed。tag 保留，不创建/不修改 Release |
| 2026-08-26 | GitHub repository API | `op7418/CodePilot` | n/a | 管理员 token + anonymous read；未记录 token value | no-bypass 管理员确认状态生成与 Actions 等价防漂移复验 | ✅ Gate fixture pass | 管理员响应确认两条 ruleset 的 `bypass_actors=[]`；公开响应隐藏该字段但 live `id` / `updated_at` 与 `CODEPILOT_RULESETS_CONFIRMED_STATE` 精确一致；定向契约 8/8 |
| _待执行_ | Windows packaged Electron | GitHub Release | n/a | unsigned NSIS | RC-A bootstrap → RC-B differential/full fallback | ⏳ | clean VM run / installer logs / version+data checks |

## 决策日志

- 2026-08-26：用户明确“先做 Windows，Linux 延后，不申请微软签名”。采用此前讨论过的 GitHub single-trust-root unsigned updater；不再把 Windows auto-update 与 signer 配置绑定。
- 2026-08-26：基于当前工作树 `a5450197` 完成现状审计。发现三项仓库保护尚未满足，因此实现采用 publish-time fail-closed，而不是先发布 unsigned feed 再补门禁。
- 2026-08-26：差分更新沿用 exact-pinned `electron-updater@6.8.3` 的 NSIS blockmap 机制；显式固定 differential 开启并保留完整 installer fallback，不自造 patch 协议。
- 2026-08-26：在未提交工作树完成实现与本地验证；保留正在运行的 Next dev 进程，通过隔离副本完成 production build。当前状态只能报 `Tests pass`，必须等仓库保护启用和 Windows RC-A → RC-B 真实 smoke 后才能报 `Release ready`。
- 2026-08-26：Claude 只读审查报告 2 个 P2、4 个 P3，逐项复核均成立；本轮不以 tech-debt 关闭，直接修复真实 publisher breadcrumb、当前版本资产白名单、裸 metadata URL、精确 checksum 集合、ruleset exclude/重名/分页与短时 Immutable acknowledgement，并补负例测试。
- 2026-08-26：用户授权发版后，启用 Immutable Releases 管理员 acknowledgement、active main no-delete/no-force 和 `v*` no-delete/no-update ruleset；`v0.67.8` 正式 run 的所有平台构建通过，但 final release job 发现 Actions `GITHUB_TOKEN` 因没有 ruleset write access 而不返回 `bypass_actors`。这是权限可见性与 verifier 假设不匹配，不是规则缺失；按不可变发布纪律保留失败 tag，不重跑/移动。
- 2026-08-26：不把缺失 `bypass_actors` 降级为空，也不把管理员 PAT 放进 Actions。改为管理员完整响应验证 no-bypass 后生成短时确认状态，CI 实时核对同一 ruleset 的 shape、`id` 与 `updated_at`；任何规则改动都会要求管理员重新确认。修复版本递增为 `v0.67.9`。
