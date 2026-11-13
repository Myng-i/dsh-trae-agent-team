# dsh-trae-agent-team

> 派活与验收纪律：让 DSH 当总管、TRAE 当派活台（或直接用 DSH 原生 Agent Teams），maker ≠ checker 独立验收。
> 单文件技能，无外部依赖，装进技能目录即可用。

[English](README.md) ｜ **中文**

## 这是什么

一个把「多 agent 干活」变成**可核算、可验收**的作业规范，不是框架、不是插件、不需要跑服务。核心就十条：

1. **先算账再开会**——成员 2 / 上限 4、每人 3 轮、返工 3 轮、时限 30 min，超了停手报人；
2. **三个角色**：lead（分派）/ maker（做）/ checker（验），人格各一句话；
3. **任务契约六必填**：验收定义、写作用域、依赖、不可逆标记、QA ≤12 条、证据槽；
4. **状态机三硬约束**：一人一个 in_progress、等验收不许 claim、状态只能本人改；
5. **maker ≠ checker**：写的人不验自己，用 fresh context 子 Agent，换模型更好；
6. **每轮只给下一步**，不一次倒十步；
7. **成本与爆炸半径闸门**：轮次、子代理数、并行上限都写死；
8. **逐笔账本**：时间、派单、产物、耗时、返工次数；
9. **收工清单**：产物路径 + 判据复核 + 账本收口；
10. **跨端编制**：DSH ⇄ TRAE 的任务板 / 信箱 / 双向叫醒（可选，纯 Node 脚本）。

## 安装

单文件技能，把 `SKILL.md` 放进任一技能根：

```bash
# Claude Code / DSH 风格：~/.dsh/skills/<name>/SKILL.md
mkdir -p ~/.dsh/skills/dsh-trae-agent-team
curl -fsSL https://raw.githubusercontent.com/Myng-i/dsh-trae-agent-team/main/SKILL.md \
  -o ~/.dsh/skills/dsh-trae-agent-team/SKILL.md

# Trae CN / 兼容目录：~/.trae-cn/skills/<name>/SKILL.md
mkdir -p ~/.trae-cn/skills/dsh-trae-agent-team
cp ~/.dsh/skills/dsh-trae-agent-team/SKILL.md ~/.trae-cn/skills/dsh-trae-agent-team/SKILL.md
```

装完重开（或重启）对应工具，技能列表里出现即生效。**备份目录别留在技能根里**——同名备份会被加载器优先选中，实测曾把 v0.3.7 的技能加载成 v0.3.0。

## 怎么用

两条路，按你手上有什么选：

| | DSH 原生 Agent Teams | 跨端（DSH ⇄ TRAE） |
|---|---|---|
| 接口 | `spawn_teammate` / `team_task_*` / `send_message` | `bus-cli.mjs` 任务板 + `mailbox.mjs` 信箱 + `wake-*.mjs` 叫醒 |
| 适合 | 同一个 DSH 里多角色 | 让 TRAE 的免费子代理当成员 |
| 依赖 | 无（内置） | 配套仓库 [dsh-trae-channel](https://github.com/Myng-i/dsh-trae-channel) |

派活模板见 [`templates/task-body-template.md`](templates/task-body-template.md)（11 项清单 + 标题行 = 12 段，缺一段别发单）。

## 并行度：实测数据（不是猜测）

全部用文件系统时间戳判定（子代理自报时间实测偏 2~3.6 s，不可作证据）：

| 形态 | 结果 |
|---|---|
| 一次发 3 个任务 | **串行**，三个子代理零重叠（单会话按轮处理） |
| 一单里「同时起 3 个子代理」 | 真并行，重叠 4 s |
| 4 路 | 重叠 7.211 s，第 4 路晚 0.69~0.88 s |
| 5 路 | 重叠 14.761 s；启动打散 8.073 s（主因是子代理自找的沙箱重试，改指令后降到 1.232 s） |
| 6 路 | 第 6 个**直接报错** `concurrent subagent limit reached (limit: 5)`，0 秒失败、不排队 |

⇒ **并行要「一单多子代理」，不要「多单并行」；上限 5 是硬上限，超了减路重派。**

## 成本纪律

一句话（"只回一句话，不要派活"）在无闸门配置下实测跑出 **57 次模型请求、约 73 万输入 token、6 个成员全被唤醒**——团队章程会自动扩散，人类留言压不住它。所以技能里把预算、并行、返工次数全写成硬数字，超了停手报人，而不是静默续跑。

## 许可

MIT，见 [LICENSE](LICENSE)。
