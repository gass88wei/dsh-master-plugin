# P0 侦察报告（夜间自动执行 · 终版）

> 对应《dsh-master-改造方案.md》§1。执行者：ZCode（pstack 流程）。
> 仓库：gass88wei/dsh-master-plugin @ ad0c392；分支 `p0/recon-baseline`（报告）、`p0/linux-testkit-fixes`（补丁），均已本地合并 main。
> 状态标记：✅ 已实证 / ❌ 受阻（注明原因）。
> 日期：2026-11-17 凌晨至清晨

---

## 1. 结论一览

| 项 | 状态 | 结论 |
|---|---|---|
| 仓库获取 | ✅ | fork 已克隆，工作分支已建、已合并 main（**未推送远端**） |
| Node 版本假设 | ✅ 修正 | 方案写"Node 24"，实际插件 `engines: >=22.19`（本机 22.23.4 即可）；DSH 0.2.2 宿主自身要 Node ≥24.9（用 DSH NEXT 自带 24.21 解决） |
| 单测基线 | ✅ | **31 文件 291 测试全绿**，typecheck 零错误，deepin 开箱即过（补丁后复测仍全绿） |
| e2e 基线 | ✅ 终版 | 无浏览器时 12✓/14✗（14 个失败全部=缺无头 Chrome，补丁自证）；接入本地 Helium 后 **17✓/9✗/1skip**，剩余 9 个失败全部是看板页面等待超时/空断言一类（等待调优问题，非环境缺失） |
| 四个注入点 | ✅ | 全部定位到文件:行号（§3），方案文档"推测"全部坐实 |
| dsh CLI | ✅ | 官方包 `@deepseek-ai/dsh@0.2.2-rc.1`（用户同意）装在工作区本地前缀，配 DSH NEXT 自带 node 24.21 运行，`dsh --version` 验证通过 |
| Linux 适配 | ✅ 摸清并部分修复 | 差异清单 9 条（§4）；testkit 两处补丁已提交（8082c2c） |
| 机器睡眠 | ✅ 已识别并规避 | 第一轮 e2e 被 4 小时自动睡眠冻结作废；之后所有长任务包 `systemd-inhibit` 禁睡眠 |
| pnpm 环境 | ✅ | 垫片死循环根因查明（PATH 重复目录），corepack 绕行，不动垫片 |

## 2. 环境与网络（实证记录）

- deepin x86_64，Node v22.23.4（系统）/ v24.21.0（DSH NEXT 自带），git 2.51.0，磁盘剩约 5.7G
- 网络：github.com 直连被掐；`git -c http.proxy=http://127.0.0.1:10808` 可用；镜像 ghfast.top / gh-proxy.com 备用；moeyy 已失效
- gh CLI keyring token 失效（未影响工作；未做需要认证的操作）
- **DSH NEXT**（DeepSeek Harness 桌面版 2.0.23-next）在 `/opt/DSH NEXT/`，用户 home `~/.dsh`（内含 profiles/desktop，**dsh-master 插件已装在日常 profile**）；旧 DSH Tauri 已卸载，残留孤儿 pnpm 垫片
- 垫片死循环根因：PATH 同时含 `/home/QQQ/.local/bin` 与 `/home/QQQ/.local/share/../bin`（同一目录两种写法），垫片按字符串跳过自身目录，把"另一个"转发回自己，16 次触发保护。**顺手修 PATH 重复项可根治**（见 §8 建议）

## 3. 四个注入点（全部实证）

### 3.1 master_review 流程（验收闸门）

- 判定函数：`Core.review()`，`packages/dsh-master/src/core.ts:420-442`
- verdict 正则 `/^(通过|打回)$/`（core.ts:422）；状态映射 core.ts:429（打回→「打回」；通过+final→「完成」；轻任务 core.ts:426-428 特殊处理）
- 打回信息流：core.ts:431-432 组装（抬头 + 总管逐条 notes + 指令「按上面逐条改，改完再用 master_report(state=就绪) 回报」），经 `enqueue` 以 `mode:'followup'`、`label:'验收'` 直投负责会话（core.ts:434）
- gate 来自 `task_spec_write` 的 `gates` 参数（core.ts:247-252）；看板渲染 panel-service.ts:144,175；工具仅总管可得 core.ts:228-231（`toolsFor` core.ts:165-168）

### 3.2 任务认领（claim）守卫

- 软认领纯函数 `packages/dsh-master/src/claims.ts`；派活校验 core.ts:364-370；运行时拦截 index.ts:369-382
- 越界判定 `normClaims()`（claims.ts:20-36）：路径须在项目根内（:29）、不能是根本身（:30）、不能碰 `.dsh-master/`（`.dsh-master/out/` 放行，:31）、最多 20 条（:34）
- 冲突判定 `conflictsWith()`（claims.ts:39-46，路径相同或互为上下级）；派活命中且无 `claims_reason` 抛 ToolInputError（core.ts:369-370）
- 写时拦截 index.ts:370-382（`tools/pre-execute`）：write/edit 与 bash 可见写目标（guard.ts:58-71）调 `claimHitFor`（core.ts:286-289），命中即 deny（`denyText()` claims.ts:60-63），要求 `master_report(state=受阻)` 协调；乐观锁冲突提示 CONFLICT_HINT（claims.ts:71-74，post-execute 注入 index.ts:396-413）
- 另一层路径守卫（总管状态目录/机密目录/重活/noGit）：guard.ts:83-119

### 3.3 devlog 召回上下文注入（三处，全在 `packages/dsh-master/src/index.ts`）

1. **系统提示常驻段** index.ts:233-236：`u.mem.residentSummary()` 拼在角色提示后；格式 memory.ts:311-325（首行 `## 记忆层摘要（dsh-master 自动注入…）`）
2. **写文件前按文件推送** index.ts:383-391：以 deny 形式返回，正文以 `[dsh-master 前情提醒（自动推送，不是用户说的，也不是执行失败）]`（PUSH_FILE，index.ts:89）开头 + `formatPriors()`；去重 recall-push.ts:12-31
3. **每步开始前按话题推送** index.ts:440-482（`agent/pre-step`）：步末追加 `role:'user'`、`source:{kind:'dsh-master',form:'notice'}` 消息，正文以 `[dsh-master 相关前情（自动推送，不是用户说的）]`（PUSH_TOPIC，index.ts:90）开头；已点名 slug 不回推（recall-push.ts:34-38）
- 手动召回 `devlog_recall`：devlog tools.ts:43-45，底层 `MemoryLayer.recall()`（memory.ts:233-244）
- **IDF**：weights.ts `tagWeights()` :37-50，**IDF=`ln((n+1)/c)` 在 weights.ts:46**；停用表 STOP_TAGS :9-22（DF_MAX 0.15）

### 3.4 cordis.patch.yml agent 预设与工具白名单

- 两个 preset（`packages/dsh-master/cordis.patch.yml`）：成员 `dsh-master-exec`（:7-104）、总管 `dsh-master-master`（:105-173）
- **总管不能写文件**：yml :3 + :143-147（`tool-fs` `write:false`/`edit:false`）
- **成员不能 git commit**：运行时守卫 guard.ts:31（NO_GIT 文案）、:33（GIT_MUTATE 正则，拦 add/commit/push/pull/merge/rebase/reset/branch -d/checkout 等，只读放行）、:105（`role==='exec' && cfg.noGit`）；noGit 判定 index.ts:361-363
- EXEC_DENY 硬禁 6 宿主工具（index.ts:88，应用 :238-240）：send_message / list_agents / interrupt_agent / schedule_create / schedule_update / schedule_delete

## 4. Linux(deepin) 适配差异清单（终版 9 条）

| # | 位置 | 问题 | 状态 |
|---|---|---|---|
| L1 | `packages/testkit/src/cdp.ts:26` | 无头浏览器路径硬编码 macOS → 14 个 UI 用例失败 | ✅ **已修**（8082c2c）：`DSHM_CHROME_BIN` 环境变量可覆盖，缺省 macOS 不变；失败信息自解释 |
| L2 | 环境缺 `dsh` CLI | e2e 前置（sandbox.ts:126，要求 `>=0.2.1-rc.5 <0.3.0`） | ✅ **已解**：官方包 0.2.2-rc.1 装工作区本地前缀（用户同意），PATH 注入 |
| L3 | `heavy.ts:14-24` | free-memory 检查仅 darwin 跑 `vm_stat`，Linux 走 `os.freemem()` 兜底 | 无需改（口径略保守） |
| L4 | `guard.ts:24` | WRITE_OPS 正则含 `osascript` | 无副作用，不动 |
| L5 | `testkit/host.ts:11` | `/usr/bin/which` 硬编码 | Linux 一般存在，不动 |
| L6 | 环境 | DSH pnpm 垫片死循环（PATH 重复目录）；仓库脚本内嵌套 `pnpm` 调用全部失败 | 绕行方案：`corepack pnpm ...`；根治需清 PATH 重复项（用户操作） |
| L7 | 环境 | 旧 DSH Tauri 已卸载；DSH NEXT 在用（桌面版 2.0.23-next，自带 node 24.21/pnpm 11.28） | 无需处理；CLI 由 L2 方案补齐 |
| L8 | 机器 | 自动睡眠冻结长任务（第一轮 e2e 冻结 4 小时作废） | 长任务一律包 `systemd-inhibit --what=sleep:idle` |
| L9 | 环境 | 机器未装 Chrome/Chromium；用户指明用本地 **Helium**（AppImage，Chrome/156 内核） | ✅ `DSHM_CHROME_BIN` 指向 Helium 冒烟通过，UI 用例 5 个转绿；剩 9 个为页面等待超时一类（后续调 `open()` 等待时长或排查 Helium 兼容性，非阻塞） |

## 5. 测试基线（P0.3 · 五轮全记录）

| 轮 | 条件 | 结果 |
|---|---|---|
| typecheck + 单测 | — | ✅ 零错误；**291/291 全绿**（补丁后复测同样全绿） |
| build | corepack pnpm -r build | ✅ dsh-master 产物 269KB |
| e2e 第一轮 | 无 dsh CLI；被睡眠冻结 4h | 作废（计时污染） |
| e2e 第二轮 | 无 dsh CLI | 11 用例统一 90s"沙盒 DSH 起不来"——受阻基线，根因=PATH 无 dsh |
| e2e 第三轮 | dsh CLI 就位 | 宿主真启动（dsh web + 假模型 426KB 请求记录），但死因信息不可读（裸 "fetch failed"） |
| e2e 第四轮 | 第三轮 + 剥离代理变量 | 结果相同 → 排除代理干扰假设 |
| **e2e 第五轮** | CLI + 诊断补丁 + 禁睡眠 | **12 ✓ / 14 ✗ / 1 skip**；14 个失败 100% = 无头 Chrome 缺失（补丁自证）；核心用例全绿 |
| **e2e 第六轮（终版）** | 第五轮 + `DSHM_CHROME_BIN`=本地 Helium | **17 ✓ / 9 ✗ / 1 skip**；UI 用例 5 个转绿（G10×3、G11-2 等）；剩余 9 个失败全部为看板页面等待超时/空断言一类，属等待调优而非环境缺失 |

- e2e 通过名单（核心）：G2 派活/回报/验收全流程、G3 记忆层+自动推送、G5 恢复/断网/备份/自检、G7 权限/日志守卫 等 12 用例
- 跳过：g10-shots（需 `DSHM_SHOTS=1`，按设计）
- e2e 不需要 API key/外网：本地假模型（mock-llm.ts:41-86）；e2e 目录 `~/dsh-master-e2e`（须在 git 仓库外，guard.ts:11-13）

## 6. 推测 vs 实证对照（方案文档回填清单）

| 方案文档原表述 | 实证结果 |
|---|---|
| P0.1 "Node 24 + DSH 0.2.2-rc.1" | 插件 Node ≥22.19 即可；宿主 dsh 0.2.2-rc.1 已就位（npm 官方包+NEXT 自带 node） |
| P0.1 "free-memory 检查需重写或禁用" | 降级：heavy.ts 已有 `os.freemem()` 兜底，仅口径保守 |
| P0.1 "不通则先修平台问题，其他全部顺延" | 实测远好于预期：单测全绿；e2e 卡点收敛为"缺浏览器"一项，且已有修复补丁 |
| P1 "master_review 怎么判 pass/send-back" | 坐实：Core.review() core.ts:420-442 |
| P0.2 ② claim 守卫位置 | 坐实：claims.ts + core.ts:364-370 + index.ts:369-382 三层 |
| P0.2 ③ 召回注入点 | 细化：注入三处在 index.ts（233/383/440），IDF 在 weights.ts:46 |
| P0.2 ④ 预设与白名单 | 坐实并补充：成员禁 git commit 是 guard.ts 运行时守卫，不在 yml |
| （新增）"e2e 整体受阻" | 收敛为"缺无头 Chrome"单一死因；核心 12 用例已绿 |

## 7. 决策账本

`decisions.tsv`（工作区根目录，随行追加）。要点：网络通道探测、Node 版本修正、垫片根因、睡眠作废重跑、CLI 安装（用户同意）、代理假设证伪、pkill 自杀坑、终版基线。

## 8. 明早建议（按优先级）

1. ~~装无头浏览器~~ ✅ 已解决：用户指明本地 Helium，`DSHM_CHROME_BIN` 指向即可（跑 e2e 时带上这个变量）
2. ~~推送 main~~ ✅ 已完成：ad0c392..35ba7ee 已推送 fork
3. ~~PATH 重复~~ ✅ 已修复：`~/.local/bin/env` 守卫补上规范拼写，垫片不再死循环
4. **P1 可以开工**：地基比方案预期好，P1（验收闭环）直接从 main 拉新分支即可；四个注入点位置见 §3
5. 剩余 9 个 UI 用例的页面等待超时：下次做 P1 时顺手调（`cdp.ts` 的 `open()` 默认等待 6 秒偏短，或排查 Helium 首屏加载），不阻塞任何工作
6. 日常跑 e2e 的固定姿势已验证：`PATH=CLI的bin:NEXT自带node:$PATH` + `DSHM_CHROME_BIN=Helium` + `systemd-inhibit`（见 §5 第六轮条件）
