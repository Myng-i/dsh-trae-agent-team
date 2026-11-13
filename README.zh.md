# dsh-trae-agent-team

DSH 当总管、TRAE 当成员：派活、验收、记账的一套纪律。单文件技能，无运行时依赖。

**中文** ｜ [English](README.md)

## 十条纪律

1. 先算账再开会：成员 2（上限 4）、每人 3 轮、返工 3 轮、时限 30 min，超了停手报人；
2. 三个角色：lead 分派、maker 执行、checker 验收；
3. 任务契约六必填：验收定义、写作用域、依赖、不可逆标记、QA 清单、证据槽；
4. 状态机：一人一个 in_progress、等验收不许 claim、只有本人能改状态；
5. maker ≠ checker：写的人不验自己，验收换全新上下文的子代理；
6. 每轮只给下一步；
7. 闸门：轮次、子代理数、并行度都是硬数字；
8. 逐笔账本：时间、派单、产物、耗时、返工；
9. 收工：产物路径 + 判据复跑 + 账本收口；
10. 跨端：任务板、信箱、双向叫醒（可选）。

无闸门配置下，一句「只回一句话、不要派活」实测跑出 57 次模型请求、约 73 万输入 token、6 名成员被唤醒——所以上面这些数字是硬上限，不是建议。

## 安装

```bash
mkdir -p ~/.dsh/skills/dsh-trae-agent-team
curl -fsSL https://raw.githubusercontent.com/Myng-i/dsh-trae-agent-team/main/SKILL.md \
  -o ~/.dsh/skills/dsh-trae-agent-team/SKILL.md

mkdir -p ~/.trae-cn/skills/dsh-trae-agent-team
cp ~/.dsh/skills/dsh-trae-agent-team/SKILL.md ~/.trae-cn/skills/dsh-trae-agent-team/SKILL.md
```

装完重启对应工具。备份目录不要留在技能根里，同名备份会被当成正式技能加载。

## 怎么用

| | DSH 原生 Agent Teams | 跨端（DSH ⇄ TRAE） |
|---|---|---|
| 接口 | `spawn_teammate` / `team_task_*` / `send_message` | `bus-cli.mjs` 任务板 + `mailbox.mjs` + `wake-*.mjs` |
| 适合 | 一个 DSH 会话里多角色 | 让 TRAE 的免费子代理当成员 |
| 依赖 | 无 | 配套仓库 [dsh-trae-channel](https://github.com/Myng-i/dsh-trae-channel) |

派活模板：[`templates/task-body-template.md`](templates/task-body-template.md)，十二段，缺一段别发单。

## 并行度（实测）

判定用文件系统时间戳；子代理自报时间会偏 2~3.6 s，不作为证据。

| 形态 | 结果 |
|---|---|
| 一次发 3 个任务 | 串行，零重叠 |
| 一单里起 3 个子代理 | 真并行，重叠 4 s |
| 4 路 | 重叠 7.211 s |
| 5 路 | 重叠 14.761 s |
| 6 路 | 第 6 个直接报错 `concurrent subagent limit reached (limit: 5)`，不排队 |

⇒ 并行要「一单多子代理」，不要「多单并行」；上限 5，超了减路重派。

## 许可

MIT，见 [LICENSE](LICENSE)。
