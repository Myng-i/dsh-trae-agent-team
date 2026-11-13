---
name: dsh-trae-agent-team
description: 派活与验收纪律：DSH 原生 Agent Teams（spawn_teammate / team_task_* / send_message），或跨端用 TRAE 派活台 + 它的子代理做成员（DSH 总管）。任务契约、maker≠checker 独立验收、PMO 每轮只给下一步、成本与爆炸半径闸门、逐笔账本。
whenToUse: 任务能拆成 2 个以上独立可验收子任务、需要并行或需要干净上下文做独立复查时；用户说"派给子代理 / 组个团队 / 并行做 / 找人复核"时；或说"让 TRAE 干活 / TRAE 派给子代理 / DSH 当总管"时；已经起了 teammate 要决定怎么验收与收工时。
---

# DSH ⇄ TRAE 派活与验收纪律

版本 v0.1.22（2026-11-13）｜单文件技能，无外部依赖
修订：v0.1.1 依据 TRAE 审阅第 1 轮（T-000025，`docs/skill-审阅-20261113-trae侧.md` 9 条）与 DSH 独立实测（2 条硬断点）重写 §十；v0.1.2 依据第 2 轮（T-000026，`docs/skill-审阅-20261113-第2轮.md` 5 条）修 10.2/10.3/10.6/10.7 并新增 10.9 工具缺口；v0.1.3 按第 3 轮核验（T-000027）补一句 stats 不含 result。

借鉴 @suifei/dsh-ai-team（文渊阁）的团队管理骨架——任务契约、claim 状态机、maker≠checker、PMO 导航——但**去掉它的自动扩散**，换成可核算的成本闸门。
设计依据来自 2026-11-13 的隔离试装实测：一句话（"只回一句话，不要派活"）跑出 57 次模型请求、约 73 万输入 token、6 个成员全被唤醒。

## 零、先算账，再开会

**只有同时满足两条才值得组队**：① 能拆成 ≥2 个**独立可验收**子任务；② 并行或上下文隔离带来的收益 > 协调成本。单点改动、半小时能做完的、纯问答——自己干更便宜。

开第一条消息之前，把预算写进任务描述和 todo：

| 预算项 | 默认值 | 说明 |
|---|---|---|
| 成员数 | 2（上限 4） | lead 自己不算 |
| 每人轮次 | 3 | 一轮 = 一次派活到回执 |
| 返工轮次 | 3 | 超过 3 轮说明设计有问题，停手报人 |
| 时限 | 30 min | 到点汇报现状，不硬撑 |
| 停止条件 | 全部任务 done | **不许「顺便再优化一下」** |

DSH 没有 token 计量器，用**轮次 + 产物字节 + 时间**当代理指标，记在账本里（见 §七）。没有预算的团队 = 不许开。

## 一、编制：三个角色就够

| 角色 | 职责 | 人格一句话（写进 spawn prompt） |
|---|---|---|
| lead（自己） | 定义可验收结果、拆任务、做决定、管预算 | 不替执行者写实现，只在关键时刻决定 |
| maker（1 人） | 认领任务、交付可运行结果 | 先交付结果和证据再汇报；遇到阻塞先找 lead，不要自己扩大范围 |
| checker（1 人） | 独立验收，只按验收定义判断 | 不接受自述，只认复跑证据；不确定就退回 |

- 领域专家（UX / 业务 / 安全）**按需加，加之前必须回答**："为什么这个角色不能由 checker 兼？" 答不上来就不加。
- 所有成员共享同一工作目录与文件系统：**写作用域必须互不重叠**，重叠就排期（blocked_by），不要靠"我们注意点"。

## 二、任务契约：六个必填

派活前逐项写清，缺一项就不许派：

1. **验收定义**——怎么算做完，必须可复核（"跑 X 命令输出包含 Y" / "文件 Z 第 N 行是 …"）。
2. **写作用域**——文件或目录前缀清单，互不重叠；对应 `team_task_create` 的 `write_scopes`。
3. **依赖**——`blocked_by`，前置任务没 done 不许开工。
4. **不可逆标记**——删数据、改权限、发布、发消息给外部：标 `irreversible`，**只能人确认**。
5. **QA 清单**——≤12 条逐条可勾的检查项。
6. **证据槽**——提交时至少 1 条证据（命令 + 原始输出 / 文件:行号 / 截图路径）。

> "验收定义"写不出来，说明这个任务还没想清楚——先自己想，不要丢给子代理去悟。

## 三、状态机与三条硬约束

```
pending ──claim──▶ in_progress ──submit(带证据)──▶ pending_acceptance ──accept──▶ done
                        ▲                                    │
                        └────────changes-requested───────────┘
```

1. **一人同时只干一个任务**：已有 in_progress 的不许再 claim。
2. **等验收时不许领新任务**：pending_acceptance 悬空就先去催验收。
3. **claim 过的人永远不能当验收人**（见 §四）。

DSH 原生映射：

| 契约动作 | 工具 |
|---|---|
| 建任务（附契约） | `team_task_create`（subject + description 写全六项 + write_scopes + blocked_by） |
| 开工 | `team_task_get` 拿 revision → `team_task_update` action=claim |
| 提交 | `team_task_update` action=complete 只能表示"提交"，**不等于验收通过** |
| 打回 | action=reopen，描述里写 ≤3 条问题 |
| 派活/催办 | `send_message`（自包含：目标、验收定义、证据格式、别做什么） |
| 等待 | `wait_agent`，醒来先 `list_agents` + `team_task_list` 复核 |

## 四、maker ≠ checker（整套纪律的重心）

- **验收必须换干净的上下文**：用 `subagent`（fresh），**不用 `subagent_fork`**——fork 会把写代码那条推理链带过去，等于自己审自己。
- **验收动作是复跑，不是阅读**：声称"已修复"不算，要自己跑一遍拿到原始输出；对照验收定义逐条打勾。
- **输出三选一**：
  ```
  STATUS: DONE | CHANGES | NEED_HUMAN
  结论：一句话
  问题：≤3 条，每条带证据（文件:行号 或 命令原始输出）
  下一步：一条可执行指令
  ```
- **不为挑刺而挑刺**：没有实质问题就 DONE。**锚点之外的改进建议一律不提**（防范围扩张）。
- **同一问题连续两轮未解决 / 需求有歧义 / 涉及不可逆操作 → NEED_HUMAN**，不要自己拍。
- 人自己验的，标记为"非独立验收"，别写进"已通过独立复查"。

## 五、推进节奏：每轮只给下一步

lead 每轮只做三件事：核实 → 对照验收定义找偏离与漏项 → 给当前最重要的一步。
- 用户中途插入的意见 **优先于原锚点**，并视为锚点更新。
- 汇报给用户时只给：做完了什么 / 证据在哪 / 卡在哪 / 下一步；不搬运成员原话。

## 六、成本与爆炸半径闸门（血泪教训）

| 风险 | 实测现象 | 闸门 |
|---|---|---|
| 自动扩散 | 团队章程优先级高于人类留言：明确写"不要派活"仍建任务、交 QA、触发全员开会 | 用户说"停/别派活"是**最高优先级**：立即中断、不再 spawn；该轮只做收尾 |
| 自动开会 | PMO 自动把 6 个成员全部唤醒各跑一轮 | 默认**不自动开会、不自动唤醒**；要开必须人显式批准一次 |
| 预算算错维度 | 只限"重建次数 6 次/30 min"，token 与子代理数无上限 | 自己按轮次+时间记账，超预算停下报告，不静默续跑 |
| 静默失败 | 成员失败无回执，lead 以为在跑 | 每个成员必须回执（阶段 + 证据）；无回执视为失败，**最多重试 1 次**，第二次失败转人 |
| 上下文丢失 | 成员会话 NOT_RESUMABLE 后静默重建，历史全丢 | 关键结论落盘到交接文件，不依赖成员记忆 |
| 权限放大 | 有的团队插件把 sandbox/approval 写死成最宽档 | 团队只许跑在专用隔离 home/profile；绝不在日常 profile 放开权限 |

**每条 spawn 都要写理由**：为什么需要这个成员 / 为什么 lead 自己不能做。写不出来就不 spawn。

## 七、账本：可复盘才叫管理

派活与验收各追加一行到 `<工作区>\_团队\<YYYYMMDD>-ledger.md`：

```
| 时间 | 事件 | 任务 | 谁 | 证据 | 结论 |
|---|---|---|---|---|---|
| 22:41 | spawn | — | checker | fresh 上下文 | 起 |
| 22:52 | submit | T1 | maker | npm test 12 passed | 待验 |
| 22:58 | accept | T1 | checker | 独立复跑同样 12 passed | DONE |
```

理由：DSH 的任务板与 session 日志都不落 usage，事后算成本、追责、写交接全靠这个文件。收工时补三行汇总：总轮次 / 成员数 / 用时。

跨端（§十）时**以 `bus\events.jsonl` 为事实源**：账本只写三行汇总（任务 id / 谁做 / 验收结论 + 证据路径），逐笔明细引 bus 事件而不抄一遍——两处都写会漂。

## 八、收工清单

- [ ] 所有任务 done 或 cancelled，**没有 pending_acceptance 悬空**
- [ ] `list_agents` 确认成员全部 inactive（不留后台作业）
- [ ] 后台 job 全部 `job_output` 收掉或 `job_kill`
- [ ] 账本写汇总行；结论追加到交接文件（用户原话 / 改动清单 路径:行号 / 证据 / 未验证项）
- [ ] 向用户报：做了什么、证据、成本代理指标、没做什么

## 九、一页流程

```
1 写验收定义 + 定预算
2 spawn ≤2 人（fresh，各自一句人格）
3 team_task_create：六项契约 + 不重叠 write_scopes
4 成员 claim → 干活 → 提交带证据
5 新起 fresh checker 复跑 → DONE / CHANGES(≤3)
6 CHANGES 回 4，最多 3 轮；第 3 轮不收敛 → 停手报设计问题
7 DONE → 记账 → 收工清单
```

> 跨端（TRAE 派活台 + 它的子代理做成员）见 §十；本页流程的 3~6 步换成 bus `post/claim/done` + DSH 独立复跑。

## 十、跨端编制：DSH 总管 + TRAE 派活台 + TRAE 子代理

**什么时候用**：活多、可并行，且 TRAE 在跑（有 IDE 上下文与自家子代理额度）。**DSH 永远是唯一对用户负责的一方**——验收定义、预算、独立验收、账本不外放。

```
用户 ── DSH 总管（定验收 / 定预算 / 独立验收 / 记账 / 汇报）
          │  bus post：验收标准 + 写作用域 + 不可逆标记
          ▼
       TRAE 派活台（claim，把任务拆给自家子代理）
          │
          ├─ TRAE 子代理 A（成员）→ done --result + --artifact
          └─ TRAE 子代理 B（成员）
          ▼
       DSH 独立复跑 → 通过记账 / 打回重派
```

**三条边界（谁都不越界）**

| 角色 | 只做 | 绝不做 |
|---|---|---|
| DSH 总管 | 定验收、派单、独立复跑、记账、对用户汇报 | 把验收交给 TRAE；**改别人已认领任务的状态**；替成员写实现（真要亲自修就另记一笔"总管兼做"并降级验收口径） |
| TRAE 派活台 | 按自己身份 claim、拆给子代理、progress、`done` 带证据 | 改验收标准、扩范围、替成员宣布验收通过 |
| TRAE 子代理 | 一人一任务、交付 + 证据 | 跨任务改别人文件、自验通过、等验收时领新活 |

**唯一的批准权在用户**：安装/覆盖技能目录、发布、删除数据这类不可逆动作，两边都只能停手报人，任务板上的 `--accept` 也不构成授权。

### 10.0 前提（照做前先对一遍）

- 所有命令都在协作根 `DSH-TRAE\` 下执行（`tools/`、`bus/`、`mailbox/` 都在它下面）；换目录 → `exit 5`（未初始化）。
- 事实源：板子 `bus/state.json`、流水 `bus/events.jsonl`、信件 `mailbox/`。
- `bus-cli --help` **不完整**：`fail` 子命令与 `--priority` 实测可用但没列进用法；自查一律以 `board`/`show` 的实际状态为准。

### 10.1 派单（DSH → 任务板）

```bash
cd DSH-TRAE
node tools/bus-cli.mjs post \
  --title "[<成员名>] <一句话动作>" \
  --as dsh-lead --to trae-ide --priority high \
  --body-file .tmp/task-<短名>.md \
  --accept "<一行可复核判据：跑什么命令 / 看哪个文件第几行>"
node tools/wake-trae.mjs
```

- **标题不许预写任务号**：id 由 store 自增分配，发单瞬间不可知（预写必错位）；以 `post` 返回的 `T-0000xx` 为准。返工在标题写「返工 N」。
- **六项契约在板上没有对应字段**（`post` 只认 title/body/assignee/accept/priority/tags/parent）：写作用域、依赖、不可逆标记、QA 清单**全部写进 `--body-file` 正文**；依赖再在标题前缀标 `[blocked_by T-0000xx]`。`--accept` 只放一行判据，不要塞整份验收文档。
- **信息必须承载在「标题 + body」上，别指望信件**：`wake-trae` 自动生成的叫醒文案只有「`[task] T-0000xx 标题`」和「`[mail] id 主题`」，**不含正文**；而 TRAE 会把这段文案**原样转给子代理** ⇒ 子代理实际只看到任务标题与信件主题。因此：① 标题必须自带动作与对象（`[成员名] 动词 + 对象`），禁用「同上」「详见」；② 背景、输入绝对路径、步骤、产物路径与格式、判据全写进 body，body 头部（模板的「本任务执行者」行）写「执行者 + 先 `bus-cli show <id>` 读全文再动手」；③ 要它照做的结论抄进 body，禁止「详见信箱 2026xxxx-xxx」。2026-11-13 T-000029 实测：它**能**读信箱文件（用文件工具读到完整哈希与字节数），只是正文不在输入里、默认不会主动读。
- **一个任务 = 一个成员 = 一个产物 ≤30 min**：DSH 先把活拆到单任务粒度再派；body 顶部写「本任务执行者：<成员名>」「**不做**：<明确排除>」。多个独立产物拆成多个任务并用 `[blocked_by T-0000xx]` 串依赖，别在 body 里写「顺便把 Y 也做了」——它会把「顺便」当主任务做。
- **一次最多 4 个任务在跑**（任务数经验值，非实测）。这 4 个任务若各自再派子代理，**子代理总数另受 §10.1.1 第 4 条 ≤5 约束**，两者不可相乘。多任务在单会话里实测是**串行**（T-000031/032/033 零重叠）⇒ 要并行请用 §10.1.1 的一单多子代理派法。
- `wake-trae.mjs` 退出码：0 已叫醒/无欠件 · 2 参数错 · **3 无调试端口（不要重试，让用户开端口）** · 5 未初始化 · 6 板子损坏。

### 10.1.1 并行度：多任务 ≠ 并行（2026-11-13 两组对照实验实测）

| 派法 | 实测 | 结论 |
|---|---|---|
| **一次 post 3 个任务 + 一次叫醒** | T-000031/032/033 串行：子代理起于 23:41:31 / 46 / 59，**零重叠**，3 单共 55 s | ❌ TRAE 是**一个会话按轮处理**，多任务只会排队 |
| **一条 body 里显式要求「同时起 3 个子代理」** | T-000034 子代理三个**同起 23:42:54**，公共重叠 4 s，全单 61 s | ✅ 真并行成立 |

⇒ 要并行就这样派：
1. **一单多子代理**：body 写「**同时**起 N 个子代理（并行，不要一个做完再起下一个）」，逐个列清「子代理 i → 做 X → 先落 `docs/<任务号>-sub-i.start` → 写文件 `docs/sub-i.md`」。
2. **每个子代理一个独立产物文件**：别让两个子代理写同一份文件（并发写冲突无法仲裁）。
3. **判并行靠文件戳，不靠自报**：子代理**第一步**先落一个标记文件 `docs/<任务号>-sub-i.start`（**判完立刻删**）、最后写产物 `docs/sub-i.md`；DSH 取 `.start` 的 `CreationTime` 与 `.md` 的 `LastWriteTime` 算重叠（OS 打戳，旁人可复核）。判据两条：① **每份** `.start` 的 `CreationTime` **晚于派单时刻**（防沿用陈旧标记）；② **N 份**（N = 本单子代理数）`.start` 都早于**最早**那个 `.md` 的 `LastWriteTime`，且公共重叠 ≥1 s。**「派单时刻」= 本单 `post` 记录里的 `created_at`**（`node tools/bus-cli.mjs show <id>` 可读到），不用叫醒时刻、也不用你收单时刻，否则几个时钟差几分钟就对不上账。子代理自报的 `HH:MM:SS` 只作交叉验证——T-000034 实测自报「结束」比真实落盘**早 2~4 s**（`sub-1.md` 自报 23:43:00 / 实际 23:43:03.639），自报只能证明「它说自己在跑」，证不了「没串行」。
   - **坑 A：`.start` 同名复用会判出假并行**。Windows 覆盖写**不改** `CreationTime`（2026-11-13 本机实测：同一路径二次 `writeFileSync`，birth 不变、只 mtime 前进）⇒ 若沿上一单的同名 `.start`，判据会得出「N 份都很早开始」。办法：文件名带任务号（`docs/<任务号>-sub-i.start`）、**判完立刻删**，别留到下一单污染目录计数。
   - **坑 B：`.start` 别写出写作用域之外**。派单常把写作用域写成「只可改 `docs/xxx.md`」，子代理再去落 `docs/…start` 就自相矛盾、按不可逆口径该停手问人 ⇒ 判据自己把活卡死。派单时把标记文件写进写作用域，或明写「`.start` 属派发开销、不受写作用域限制」。
   - **坑 C：陈旧标记不会报错，只会静默失真**。T-000036 反事实测算：若沿用上轮 23:50:18.123 的标记，重叠会从真实 2.866 s 虚算成 **36.014 s（12.6 倍）**，而「`.start` 都早于最早产物」这条**照样通过** ⇒ 判据不会报警。所以「先删同名标记再重建」必须写进派单 body 当硬要求，并用上面判据①（`CreationTime` 晚于派单时刻）兜底。
4. 并发上限：TRAE 子代理并发 ≤5，**一单建议 3~4**；要更多并行就在 DSH 侧用原生 `spawn_teammate` 分担，而不是拆更多 bus 任务。**实测口径（4 路 T-000037 / 5 路 T-000038）**：4 路公共重叠 7.211 s、第 4 路晚 0.69~0.88 s；5 路（每路睡 15 s 长活）公共重叠 **14.761 s**、全单 37.6 s，但**启动打散 8.073 s**——5 路里只有 1 路在 5 s 内起跑。其中 3 路报「`Remove-Item` 在默认沙箱 exit 1，关沙箱重试后成功」，重试发生在写 `.start` 之前，把各自的启动推后数秒；但第 3 路**未重试**仍晚于做过重试的第 1、4 路，故启动差不能只归因于重试，**排队/调度因素仍未排除**。⇒ 准确说法是「**≤5 路能在同一窗口并行运行，但不是同时起跑**」；本轮 4 路实测启动差 <1 s（**单次样本**，T-000037，别当规律）。
   - **T-000039 反证（2026-11-14 00:00）**：6 路时第 6 路**直接报错** `Subagent start failed: concurrent subagent limit reached (limit: 5).`——0 秒失败、无排队、`r6-6.md` 未生成 ⇒ **本机 TRAE 运行时实测 ≤5 为硬上限**（DSH 原生 `spawn_teammate`、跨版本/跨机未测），超限要减路重派，别等它自己补位。同单 5 路在「删标记失败不要关沙箱重试」的指令下启动打散仅 **1.232 s**（对比 T-000038 的 8.073 s）⇒ 那 8 s 主要是自找的重试开销，不是并发排队。
   - **别为了删标记去关沙箱重试**：文件名已带任务号，本身防了陈旧复用；`.start` 删不掉就留着（DSH 收尾统一清），别把 8 s 花在 `Remove-Item` 上。
5. 汇总裁决写进单独产物（如 `docs/sub-summary.md`），交付时 `--artifact` 指它。

### 10.2 执行（TRAE 侧五条）

1. **领活节奏**：`claim <id> --as trae-ide` 后立刻 `progress --note "开始"`。同一时刻最多 4 个任务在跑、每个子代理只跑一个；**每个子代理上一个没 `done` 之前，不给它派第二个**。第 4 个之后再来的任务留在 pending。
   - **两个上限不能叠加**：这个「4」是**任务数**的经验值（来源是早期口径，非本轮实测——本轮实测多任务是**串行**，T-000031/032/033 零重叠，所以「4 个任务同时跑」在单会话里复现不出来；多窗口/多会话未验证）。这 4 个任务若都走多子代理派发，**所有在跑任务派出的子代理总数仍受 §10.1.1 第 4 条 ≤5 约束**，超了按 4×N 叠加是错的；且**子代理超限是直接报错**（`concurrent subagent limit reached (limit: 5)`），不会排队补跑。
   - **`progress` 只追加说明、不改状态**：`bus-store.mjs:305` 只把 `pending` 提升成 `in_progress`，已 `claimed` 的永远是 `claimed`（第 2 轮实测）⇒ 判停滞看 `updated_at` + `nudge`，别看状态字。
2. 开工前读 `--accept` 与 body：**验收标准之外的不做**；歧义发信箱，不自己拍。
3. 交付：`done <id> --as trae-ide --result "<成员名>：做了什么 + 关键数字" --artifact <路径>`，artifact 必须真实存在、可被对方直接打开。
4. 卡住 >10 min 先 `progress` 写卡点；做不完用 `fail <id> --as trae-ide --result "<原因>"`——**不许静默消失，也不许拿半成品当 done**。
5. **不可逆动作停手**：安装/覆盖技能目录、发布、删除数据——即使 `--accept` 写了也不执行，报用户逐次确认。

### 10.3 验收（maker≠checker 的跨端版）

- TRAE 是 maker 端，**不能自己验自己**；DSH 也不许只读 `result` 就签收。
- 动作三选一按风险定：自己跑验收命令 / 看 diff / 起 fresh `subagent` 复跑；输出仍是 `DONE / CHANGES / NEED_HUMAN`（TRAE 侧对应"新起一个未继承本会话上下文的子代理"，不要复用写代码那条会话）。
- **`done` 只代表提交，不代表验收通过**。
- **DSH 不能改别人认领中的任务状态**：对 `claimed_by=trae-ide` 的任务执行 `fail`/`progress` 会被 `owned by trae-ide` 拒绝（exit 1）。DSH 只能改**自己发的、还没被认领的** pending 任务。
- **打回 = 信箱 + 新任务**（两段式）：
  ```bash
  # 1) 写 CHANGES：≤3 条，每条带证据，不发正文只落盘
  node tools/mailbox.mjs post --from dsh-lead --to trae-ide \
    --subject "[CHANGES] T-0000xx 第 2 条未过" --body-file .tmp/changes-T-0000xx.md
  # 2) 新任务用新 id，标题写「返工 1」；body 复用同一份 CHANGES（契约不丢）
  node tools/bus-cli.mjs post --title "[<成员名>] 返工 1：…" --as dsh-lead --to trae-ide \
    --body-file .tmp/changes-T-0000xx.md --accept "…"
  node tools/wake-trae.mjs
  ```
  旧任务由 TRAE 自己 `fail --result "CHANGES-REWORK: 见 mailbox/<id>"`；**`--result` 一律用这个前缀**，收工时才能把「打回重做」与「执行失败」分开统计。**不要在同一任务上反复翻转状态**。
- 通过后再记账。

### 10.4 叫醒与等待（两边都没有后台常驻）

- 派完立刻 `wake-trae.mjs`；**内容只走文件，CDP 只发 id**（单入口规则见 `dsh-trae-channel` §十一）。别用 `trae-inject send` 发正文。叫醒文案优先写「`<任务号>` + 一句话 + 产物路径」：它默认不读信箱（能读，但正文不在输入里），别指望它自己去找背景。
- TRAE 收尾：先 `node tools/wake-dsh.mjs status`，READY 才 `send --text "【TRAE 叫醒】T-0000xx done，见 bus/"`；**不是 READY（exit 3 = 端口没开）就写明"未唤醒，请用户转达"，不重试刷屏**。
- DSH 等活：`bus-cli board` / `list --status pending,claimed,in_progress`（**不要加 `--as`**）/ `show T-0000xx`。`poll --as dsh-lead` 看不到派给 trae-ide 的任务（过滤条件是 `assignee === as`），照它会误判"TRAE 没动"。停滞用 `nudge.mjs`。
- **收到的叫醒是排队送达，不是重复推送**：DSH 会话在跑时外部消息入队、按**轮边界**逐条吐出，表现为「贴着每轮回合来一条、id 各不相同」。先看节奏再判故障——**贴着轮边界、每次一条且 id 不同 = 正常排队**；**同一轮里连发多条同 id** 才是真重复。别因此给 `wake-trae`/`wake-dsh` **新增第二层**去重：`wake-trae` 本就有 `watch/traeWake.json`（`mailSeen`/`tasksSeen`，`tools/wake-trae.mjs:38,44,104,107`）负责「同一批件只叫一次」；`wake-dsh` 没有游标，也别加——真需要再叫一次时会被吞掉。2026-11-13 实测三条 `T-000025/026/027 done` 到达时刻 23:30:21 / 23:30:40 / 23:31:03，间隔等于各回合时长（据此误开过修 wake 的任务，已撤回）。
- **自唤醒默认关**，用户没明确要求不许开。超预算时限没回执：记"未交付"并转人，不无限等。

### 10.5 成本闸门（默认数字，与 §零 同口径）

| 闸门 | 默认 | 说明 |
|---|---|---|
| 成员数 | ≤4 | TRAE 子代理并发上限 **5（本机实测，硬上限）**：第 6 路直接报 `concurrent subagent limit reached`，0 秒失败、不排队 |
| 同时在跑任务 | ≤4 | 任务数是经验值；多任务在单会话实测串行，见 §10.1.1 |
| 每任务打回 | ≤2 次 | 第 3 次转人（与 §四 三轮熔断同口径） |
| 每任务交付物 | 1 个 artifact | 聊天记录不算交付物 |
| 时限 | 30 min | 到点汇报现状，不硬撑 |

### 10.6 首单实操（复制即用）

派单 body 直接抄模板：`DSH-TRAE/templates/task-body-template.md`（清单 11 项：执行者 / 不做 / 边界〔写作用域·依赖·不可逆〕/ 派发形态 / 背景 / 输入 / 步骤 / 产物 / 判据 / 证据槽 / 交付；模板首行另有 `# 任务：` 标题 ⇒ 模板文件共 12 段，缺一段就别发单）。

```bash
# DSH 侧
cd DSH-TRAE
node tools/bus-cli.mjs post --title "[审阅员] 审阅 XX 技能" --as dsh-lead --to trae-ide --priority high \
  --body-file .tmp/task-review.md --accept "docs/xx.md ≥40 行，≥5 条带原文引用"
node tools/wake-trae.mjs
node tools/bus-cli.mjs show <任务号>          # 以 post 返回值为准；别看 poll --as dsh-lead
# TRAE 侧
node tools/bus-cli.mjs claim <任务号> --as trae-ide
node tools/bus-cli.mjs progress <任务号> --as trae-ide --note "开始：读 SKILL.md"
node tools/bus-cli.mjs done <任务号> --as trae-ide --result "审阅员：…" --artifact docs/xx.md
node tools/wake-dsh.mjs status
node tools/wake-dsh.mjs send --text "【TRAE 叫醒】<任务号> done，见 bus/"
```

### 10.7 收工

- `node tools/bus-cli.mjs stats`：没有 pending，也没有 claimed 悬空；没做完的显式 `fail` 并写原因。
- 停滞判据是 `updated_at` 超过时限 + `nudge` 无力，**不是**看 `in_progress`（该状态只能由未认领任务 progress 出来，见 10.9）。
- `node tools/mailbox.mjs threads`：没有等我回的信。
- 账本三行汇总 + 每个任务的验收结论与证据路径。
- 对用户报：**过了几条 / 退回几条 / 谁做的 / 成本代理指标 / 没做什么**。

### 10.8 已知坑

| 坑 | 后果 | 正确做法 |
|---|---|---|
| assignee 写成员名 | TRAE poll 看不到任务，永远不动 | 一律 `--to trae-ide` |
| 只在信箱说"请执行 X" | 不算派活，状态机无记录 | 必须 `bus post` |
| 用 CDP 注入正文 | 无 id、无回执，漏件（2026-11-13 实测事故） | 先落盘，再 `wake-trae` |
| 标题预写 `T-0000xx` | id 由 store 分配，对账错位 | 真实 id 以 `post` 返回为准 |
| DSH 直接改别人已认领的任务 | `owned by <agent>` 拒绝，白改一次 | 状态只由认领方改；打回走 CHANGES + 新任务 |
| 等验收时又领下一个 | 交付顺序错乱、验收对不上号 | done 之后再领 |
| 一次甩一堆 pending | TRAE 并发第 6 个直接失败（不排队） | 一次 ≤4 |
| 打完回还在原任务上改 | 验收历史被覆盖，说不清哪版过的 | `fail` + 新任务 id |
| `wake-trae` 连积压信一起注入 | 一次塞十几件（2026-11-13 实测 13 件），认领顺序被旧信带偏 | 先 `mailbox inbox --as trae-ide` 看积压，必要时 `wake-trae --init` 建基线 |
| 照 `bus-cli --help` 判断有没有 `fail` | 实测有人因此误发一次 `fail` 把任务置成 failed | 以 `board`/`show` 实际状态为准 |

### 10.9 工具层缺口（改文档绕不过去的，写在这里免得下个人重踩）

| 缺口 | 证据 | 当前绕法 |
|---|---|---|
| `progress` 不把 `claimed` 提升为 `in_progress` | `tools/bus-store.mjs:305` 只处理 `pending` | 判停滞看 `updated_at` |
| 完成/失败只有认领方能做 | `tools/bus-store.mjs:317` → `owned by <agent>` | DSH 打回走 CHANGES + 新任务 |
| `post` 没有写作用域 / blocked_by 字段 | `bus-cli post --help` 只有 title/body/to/accept | 全部写进 `--body-file`，标题标 `[blocked_by]` |
| `--help` 不列 `fail` 与 `--priority` | `bus-cli.mjs --help` grep 无 `fail` | 以 `board`/`show` 实际状态为准 |
| `stats.failed` 混算「打回重做」与「真失败」 | 两段式收口后旧任务终态就是 failed；`stats` 只出 `by_status`/`last_5`，**不含 result** | `--result` 前缀 `CHANGES-REWORK:`，区分靠 `show` 或 `state.json` |

要根治其中任一条，**另开任务改工具**，不要在技能里再绕第二层。

### 10.10 安装与自检（附录）

| 副本 | 路径 |
|---|---|
| 上游（本仓库） | `https://github.com/Myng-i/dsh-trae-agent-team` → `SKILL.md` |
| DSH 安装 | `~/.dsh/skills/dsh-trae-agent-team/SKILL.md` |
| TRAE 安装 | `~/.trae-cn/skills/dsh-trae-agent-team/SKILL.md` |

- 四份（含工作 home `~/.dsh-work/skills/…`）必须**逐字节相同**；改完各拷一份，再用 `Get-FileHash SKILL.md -Algorithm SHA256` 对一遍（四行哈希必须一致）。
- 改名历史：`dsh-agent-team` → `dsh-trae-agent-team`（2026-11-13）；旧目录已移出技能根，备份放在**技能根之外**（如 `~/skill-backups/`）。
- **备份目录不要留在任一 skills 根里（两边都算）**：`~/.dsh/skills/.xxx.backup-*` 与 `~/.trae-cn/skills/…` 的备份目录都含 `SKILL.md`、技能名与正式版相同，加载器按名去重时会选中备份。2026-11-13 实测：`~/.dsh/skills` 有 6 个 `dsh-trae-channel` 备份时，`skill("dsh-trae-channel")` 返回 12 983 字符（= v0.3.0 正文），而现装 v0.3.7 是 15 379 字符；TRAE 侧同样解析到 `.backup-v0.3.6`。修法：整目录 rename 到**技能根之外**（如 `~/skill-backups/skills-backup-<日期>/`）；TRAE 侧目录被占用（rename 连报 EPERM）时，把里面的 `SKILL.md` 移走即可让它不再被扫到。移完重调 `skill("<名>")` 核字符数/版本行。
- TRAE 侧没有技能索引文件，靠目录扫描；装完要**重启 Trae CN**，列表里出现才算生效。
