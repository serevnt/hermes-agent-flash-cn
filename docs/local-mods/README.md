# Hermes 本地魔改说明

本仓库是 `NousResearch/hermes-agent` 的本地 fork（`serevnt/hermes-agent-flash-cn`）。
此目录记录**每一处对上游代码/配置的偏离**，目的是让任何接管这台机器的 Agent 能看懂改了什么、为什么改、怎么退回去。

## 铁律

1. 每处魔改必须在 `docs/local-mods/` 留一份说明，与代码同一次提交。
2. push 到远程前必须由张剑确认无问题，不得自行推送。
3. 改动前备份到仓库外（`J:\HermesShared\data\cache\scratch\hermes_baks\`），不要留在工作区制造噪声。

## 分支与远程

| 项 | 值 |
|---|---|
| 工作分支 | `cn-main`（从 detached HEAD `2bd378ed1b` 建） |
| `origin` | `https://github.com/serevnt/hermes-agent-flash-cn.git` |
| `upstream` | `https://github.com/NousResearch/hermes-agent.git`（只读参考，不 merge） |
| 本地领先 origin | 2984 个提交（fork 停在 2026-10-27，本地到 2026-11-05） |
| 更新方式 | `hermes update` 现在只拉自己的 fork |

---

## 改动记录

### M-001 补装 snowballstemmer，启用工具按需折叠

- **日期**：2026-11-09
- **类型**：依赖补装（未改代码）
- **现象**：每轮工具 schema 达 63,857 字符（≈21,285 tokens），工具数量 32 个全量注入。
- **根因**：`model_tools` 的 tool_search 折叠依赖 `snowballstemmer` 做中文/英文词干匹配。该模块缺失时，折叠流程**静默跳过**（只在日志打一行 `Tool search assembly skipped: No module named 'snowballstemmer'`），不报错，于是全部工具照常注入。
- **改动**：`.venv` 内安装 `snowballstemmer==3.1.1`（`uv pip install`，venv 自带的 python 无 pip）。
- **效果（实测）**：工具 32 → 20；schema 63,857 → 37,588 字符；每轮节省约 8,756 tokens。
- **回滚**：`uv pip uninstall snowballstemmer`，折叠失效，回到全量注入（功能不坏，只是变胖）。
- **影响面**：只影响 prompt 体积与工具可见性；被折叠的工具仍可经 `tool_search` 调用。

### M-002 技能索引移出 3 个目录级备份副本

- **日期**：2026-11-09
- **类型**：文件整理（未改代码）
- **现象**：技能索引里出现 `game-ui-ai-training.bak_20261013_0214`、`gameui-criteria-judgment.bak_20261008_1035`、`gameui-criteria-judgment.bak_20261013_0214` 三个"技能"——实际是整目录级备份，被当成独立技能列出。
- **改动**：三个目录移到 `J:\HermesShared\data\skills\.archive\`（`.archive` 以点开头，扫描时跳过）。
- **效果**：活跃技能 191 → 188；索引里减少 3 条重复项。
- **回滚**：从 `.archive` 移回原位。

### M-003 建立 git fork 结构

- **日期**：2026-11-09
- **改动**：
  - `origin` 从 `NousResearch/hermes-agent` 改为 `serevnt/hermes-agent-flash-cn`
  - 新增 `upstream` 指向官方仓库
  - 设置仓库级 `user.name=serevnt` / `user.email=serevnt@users.noreply.github.com`
  - 创建分支 `cn-main`（暂未切换，HEAD 仍为 detached —— 切换需停机，见下）
- **未完成**：`git switch cn-main` 被 Hermes 运行时保护拦截（"would rewrite live source checkout"）。需退出 Hermes 后在外部执行。
- **回滚**：`git remote set-url origin https://github.com/NousResearch/hermes-agent.git`

---

## 运行中不能改的东西

Hermes 在运行时保护自己的源码树：`git switch`/`checkout` 会被拦截，理由是"会混用模块版本"。因此涉及切分支、切 commit 的操作**必须停机执行**：

1. 完全退出 Hermes（含托盘）
2. 在外部终端执行 git 操作
3. 重新启动

改单个 `.py` 文件内容不受此限制，但要重启才生效。

## 待办

- [ ] 停机切到 `cn-main` 分支（一次性）
- [ ] 首次 push 到 `serevnt/hermes-agent-flash-cn`（需张剑确认）
- [ ] 技能索引压缩：加配置开关让「只列名字」在非 coding 场景生效
- [ ] 工具 schema 继续瘦身（`manage_connections` 4,477 字符 / `delegate_task` 4,634 字符）
