# P0 侦察报告（夜间自动执行）

> 对应《dsh-master-改造方案.md》§1。执行者：ZCode（pstack 流程）。
> 仓库：gass88wei/dsh-master-plugin @ ad0c392，分支 `p0/recon-baseline`。
> 状态标记：✅ 已实证（附证据指针）/ ❌ 受阻（注明原因）。
> 日期：2026-11-17 凌晨

---

## 1. 结论一览

| 项 | 状态 | 结论 |
|---|---|---|
| 仓库获取 | ✅ | fork 已克隆，分支 `p0/recon-baseline`（基点 ad0c392） |
| Node 版本假设 | ✅ 修正 | 方案写"Node 24"，实际 `engines: >=22.19`（package.json:23），本机 22.23.4 直接可用 |
| 单测基线 | ✅ | **31 文件 291 测试全绿**，typecheck 零错误，deepin 上开箱即过 |
| e2e 基线 | ❌ 受阻 | 需 PATH 上有 `dsh` CLI（本机无）+ 无头 Chrome（testkit 路径硬编码 macOS）；smoke 用例装好 dsh 后可单跑 |
| 四个注入点 | ✅ | 全部定位到文件:行号，方案文档的"推测"全部坐实（见 §3） |
| Linux 适配 | ✅ 摸清 | 插件本体无 darwin 硬依赖；风险集中在 testkit 的 Chrome 路径与 e2e 前置（见 §4） |
| pnpm 环境 | ✅ | 本机 pnpm 垫片死循环根因已查明（DSH 桌面版垫片 + PATH 重复目录），用 corepack 绕行 |
| git 工作流 | ✅ | 分支已建、报告已提交、本地合并到 main；**未推送远端**（等你醒来决定） |

## 2. 环境与网络（实证记录）

- deepin x86_64，Node v22.23.4，git 2.51.0，磁盘剩约 5.7G
- 网络：github.com 直连被掐（TLS unexpected eof）；`git -c http.proxy=http://127.0.0.1:10808` 可用；镜像 ghfast.top / gh-proxy.com 直连可用备用；github.moeyy.xyz 已失效
- gh CLI keyring token 失效（未影响本次工作；未做任何需要认证的操作）
- **意外发现**：`/home/QQQ/.local/bin/pnpm` 是 DeepSeek Harness Desktop 生成的命令垫片，头部注明"桌面应用启动时会重新生成，勿手改"。死循环根因：PATH 同时含 `/home/QQQ/.local/bin` 与 `/home/QQQ/.local/share/../bin`（同一目录两种写法），垫片按字符串跳过自身目录，把"另一个"转发目标指回了自己，16 次触发保护
- **意外发现 2**：DSH 桌面版"装了壳缺核"——垫片指向的运行时 `/usr/lib/DSH Tauri/runtime/linux-x64/` 不存在，`dsh` 命令也不在 PATH。意味着 P0.1 剩下的活是把 DSH CLI 真正装起来（e2e 与日常使用都需要它）

## 3. 四个注入点（全部实证，替代方案文档中的"推测"）

### 3.1 master_review 流程（验收闸门）

- 判定函数：`Core.review()`，`packages/dsh-master/src/core.ts:420-442`
- verdict 用正则 `/^(通过|打回)$/` 校验（core.ts:422）；状态映射 core.ts:429：打回→任务状态「打回」；通过+final→「完成」；轻任务特殊处理 core.ts:426-428（通过即结案、打回重开）
- 打回信息流：core.ts:431-432 组装文本 = 抬头 `验收：<task> 闸门 <gate> 打回` + 总管逐条 notes 原文 + 指令「按上面逐条改，改完再用 master_report(state=就绪) 回报」；经 `enqueue` 以 `mode:'followup'`、`label:'验收'` 直投负责会话（core.ts:434）
- gate 概念来自 `task_spec_write` 的 `gates` 参数（core.ts:247-252）；看板渲染 `packages/dsh-master/src/panel-service.ts:144,175`
- 工具仅总管可得：`masterTools()` core.ts:228-231，角色分发 `toolsFor` core.ts:165-168

### 3.2 任务认领（claim）守卫

- 软认领纯函数：`packages/dsh-master/src/claims.ts`；派活校验 core.ts:364-370；运行时拦截 `packages/dsh-master/src/index.ts:369-382`
- 越界判定 `normClaims()`（claims.ts:20-36）：路径须在项目根内（:29）、不能是根本身（:30）、不能碰 `.dsh-master/`（产物目录 `.dsh-master/out/` 放行，:31）、最多 20 条（:34）
- 冲突判定 `conflictsWith()`（claims.ts:39-46）：路径相同或互为上下级即重叠；派活命中且无 `claims_reason` 直接抛 ToolInputError 给总管（core.ts:369-370）
- 写时拦截：index.ts:370-382 在 `tools/pre-execute` 对成员 write/edit 与 bash 可见写目标（`bashWriteTargets`，guard.ts:58-71）调 `claimHitFor`（core.ts:286-289），命中即 deny，`denyText()`（claims.ts:60-63）告知冲突任务与认领人，要求 `master_report(state=受阻)` 协调
- 另一层路径守卫（总管状态目录/机密目录/重活/noGit）：`packages/dsh-master/src/guard.ts:83-119`

### 3.3 devlog 召回上下文注入（共三处，全在 `packages/dsh-master/src/index.ts`）

1. **系统提示常驻段**：index.ts:233-236 把 `u.mem.residentSummary()` 拼在角色提示后；摘要格式 `packages/dsh-master-devlog/src/memory.ts:311-325`，首行标记 `## 记忆层摘要（dsh-master 自动注入；完整内容用 devlog_recall / memory_search 查）`
2. **写文件前按文件推送**：index.ts:383-391，以 deny 形式返回，正文以 `[dsh-master 前情提醒（自动推送，不是用户说的，也不是执行失败）]`（PUSH_FILE，index.ts:89）开头 + `formatPriors()`；去重 `packages/dsh-master/src/recall-push.ts:12-31`（同会话同文件同一前情集合只拦一次）
3. **每步开始前按话题推送**：index.ts:440-482（`agent/pre-step`），命中则在步末追加一条 `role:'user'`、`source:{kind:'dsh-master',form:'notice'}` 消息，正文以 `[dsh-master 相关前情（自动推送，不是用户说的）]`（PUSH_TOPIC，index.ts:90）开头；已点名的 slug 不回推（recall-push.ts:34-38）
- 手动召回工具 `devlog_recall`：`packages/dsh-master-devlog/src/tools.ts:43-45`，底层 `MemoryLayer.recall()`（memory.ts:233-244）
- **IDF 权重**：`packages/dsh-master-devlog/src/weights.ts`，`tagWeights()` weights.ts:37-50，**IDF=`ln((n+1)/c)` 在 weights.ts:46**；停用表 STOP_TAGS weights.ts:9-22（DF_MAX 0.15）

### 3.4 cordis.patch.yml agent 预设与工具白名单

- 两个 preset（`packages/dsh-master/cordis.patch.yml`）：
  - 成员 `dsh-master-exec`（:7-104）：persona、bash/pwsh 沙盒（5 秒超时）、tool-fs 可读写、tool-fs-search、tool-jobs、skill、compaction、tool-subagent、tool-todo、tool-web(fetch:false)；:87 注释明确不装 `tool-subagent-control`
  - 总管 `dsh-master-master`（:105-173）：比成员少 persona、tool-fs-search、delegation、tool-todo、tool-web
- **总管不能写文件**：yml :3（注释）+ :143-147（`tool-fs` 的 `write:false`、`edit:false`）
- **成员不能 git commit**：不在 yml，是运行时守卫——`guard.ts:31`（NO_GIT 文案）、:33（GIT_MUTATE 正则，拦 add/commit/push/pull/merge/rebase/reset/branch -d/checkout 等，只读 git 放行）、:105（`role==='exec' && cfg.noGit` 时拦截）；noGit 判定 index.ts:361-363（项目成员且不在 worktree）
- 插件侧硬禁 6 个宿主工具 EXEC_DENY（index.ts:88，应用在 :238-240）：send_message / list_agents / interrupt_agent / schedule_create / schedule_update / schedule_delete

## 4. Linux(deepin) 适配差异清单

| # | 位置 | 问题 | Linux 上后果 | 处置建议 |
|---|---|---|---|---|
| L1 | `packages/testkit/src/cdp.ts:26` | 无头 Chrome 路径硬编码 macOS | 所有 import `launchChrome` 的 e2e（g5e-*、g4c、g5f、g10-*、g12、g6-5-open-ui 等）spawn ENOENT 直接挂 | P1 前顺手修：改 `google-chrome` 或环境变量可配 |
| L2 | 环境缺 `dsh` CLI | e2e 前置（sandbox.ts:126 要求 `>=0.2.1-rc.5 <0.3.0`） | e2e 整体受阻；只有 smoke 可在装好后单跑 | 安装 dsh CLI（npm 全局或 DSH 桌面版补装运行时） |
| L3 | `packages/dsh-master/src/heavy.ts:14-24` | `freeMemGb()` 仅 darwin 跑 `vm_stat`，Linux 走 `os.freemem()` 兜底（:23） | 不崩；口径略保守（不算 inactive/speculative 页） | 无需改；方案文档原"free-memory 检查需重写或禁用"可降级为"已有兜底，仅口径差异" |
| L4 | `guard.ts:24` | WRITE_OPS 正则含 `osascript` | 无副作用，正则永不命中 | 不动 |
| L5 | `packages/testkit/src/host.ts:11` | `/usr/bin/which` 硬编码 | Linux 一般存在；找不到仅返回 undefined | 不动 |
| L6 | 环境 | pnpm 垫片死循环（PATH 重复目录） | 仓库脚本内嵌套 `pnpm` 调用全部失败 | 不改垫片；统一用 `corepack pnpm ...` 绕行；写进 README 的开发注意事项 |
| L7 | 环境 | DSH 桌面版运行时缺失（`/usr/lib/DSH Tauri/...` 不存在） | 桌面版本体不可用（与本插件仓库无直接关系） | 单独处理：要么补装桌面版运行时，要么只走 CLI 路线 |
| L8 | 单测过程 | 非致命 "不是 git 仓库" 报错 + node:sqlite 实验警告 | 不影响通过 | 记录备查 |

## 5. 测试基线（P0.3）

| 套件 | 命令 | 结果 |
|---|---|---|
| typecheck | `corepack pnpm typecheck` | ✅ 零错误 |
| 单测 | `corepack pnpm test` | ✅ **31 文件 / 291 测试全绿**（1.45s） |
| build | `corepack pnpm -r --if-present run build` | ✅ 成功（dsh-master 产物 269KB） |
| e2e | `corepack pnpm exec vitest run --config vitest.e2e.config.ts` | ❌ 受阻于 L1/L2（23 文件 26 用例；g10-shots 仅 DSHM_SHOTS=1 才跑） |

- e2e 不需要 API key/外网/数据库：本地起 OpenAI 兼容假模型（testkit/mock-llm.ts:41-86，假 key `DSHM_MOCK_KEY='dummy-not-a-secret'`）
- e2e 总目录默认 `~/dsh-master-e2e` 且必须在任何 git 仓库外（testkit/guard.ts:11-13,28），护栏比对真实仓库足迹（e2e/repo-guard.setup.ts:15-30）

## 6. 推测 vs 实证对照（方案文档回填清单）

| 方案文档原表述 | 实证结果 |
|---|---|
| P0.1 "Node 24 + DSH 0.2.2-rc.1 装到 deepin" | Node 只需 ≥22.19（本机已满足）；e2e 实际要求 dsh CLI `>=0.2.1-rc.5 <0.3.0` |
| P0.1 "重活队列 free-memory 检查在 Linux 需重写或禁用" | 降级：heavy.ts 已有 `os.freemem()` 兜底，仅口径保守 |
| P0.1 "不通则先修平台问题，其他全部顺延" | 单测/typecheck/build 全通过，无需"先修"；仅 e2e 受阻（L1/L2），不阻塞 P1 开工 |
| P1 "master_review 流程怎么判 pass/send-back" | 坐实：Core.review() core.ts:420-442，正则校验 + followup 直投 |
| P0.2 ②"任务认领守卫实现位置" | 坐实：claims.ts + core.ts:364-370 + index.ts:369-382（三层：派活校验/写时拦截/乐观锁提示） |
| P0.2 ③"devlog 召回注入点(memory.ts)" | 细化：注入点三处在 index.ts（常驻段/文件推送/话题推送），memory.ts 只管生成内容；IDF 在 weights.ts:46 |
| P0.2 ④"cordis.patch.yml 预设与白名单" | 坐实并补充：成员"不能 git commit"是 guard.ts 运行时守卫，不在 yml |

## 7. 决策账本

`decisions.tsv`（工作区根目录，随行追加，含每条决策的 why 与证据指针）。

## 8. 明早建议（按优先级）

1. **看本报告 §6 回填后的改造方案**，确认 P1 开工范围——P0 结论是"地基比预想好，P1 可以直接开工"
2. 决定 dsh CLI 安装方式（补装 DSH 桌面版运行时 vs npm 装 CLI），补齐后跑 smoke.e2e 收全基线
3. 修 testkit/cdp.ts:26 的 Chrome 路径（改成环境变量可配，macOS 默认值不变）——一处小改，解锁全部 UI 用例，适合作为分支上第一个正式改动练手
4. 决定是否把 `p0/recon-baseline` 分支推送到你的 fork（我只做了本地合并，推送等你点头）
5. 顺手处理 PATH 重复目录问题（`.local/share/../bin`），否则 DSH 垫片会一直死循环
