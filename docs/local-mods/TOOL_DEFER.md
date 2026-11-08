# 本地改造：工具折叠（tool defer）

> 时间：2026-11-09
> 影响文件：`J:/HermesShared/data/config.yaml`（第 654 行起 `tools.tool_search`）
> 备份：`J:/HermesShared/data/config.yaml.bak_defer_20261109_015534`
> 实施脚本：`J:/HermesShared/data/cache/scratch/apply_defer.py`

---

## 1. 改了什么

在 `tools.tool_search` 下新增 `defer` 字段，共 **28 个工具名**：

- **官方默认 19 项**（原样保留，一项不少）
- **本地新增 9 项**：LCM 冷门工具

常驻（不进 defer）的 LCM 工具 6 个：`lcm_grep` / `lcm_recall` / `lcm_recent` / `lcm_expand` / `lcm_load_session` / `lcm_describe`。

`hindsight_*`（3 个）保持常驻，未列入。

---

## 2. 为什么这么改

### 2.1 显式列表是「整体替换」语义

源码 `tools/tool_search.py:57-58`：

```python
def effective_defer_tools(self) -> frozenset:
    return _DEFAULT_DEFERRED_TOOLS if self.defer_tools is None else self.defer_tools
```

`None` = 用内置默认集；**任何显式 list = 完全替换**（包括 `[]` = 全部不折叠）。
因此本改动必须把官方 19 项**逐条复制**，漏一项就会让该工具从折叠变常驻，prefill 反而变大。

### 2.2 为什么只折叠 9 个 LCM 工具，不全折

- 6 个高频工具（grep/recall/recent/expand/load_session/describe）覆盖 90% 日常检索，走 tool_search 三步桥会多 2 轮往返，得不偿失。
- 9 个冷门工具（compute / compile_evidence / evidence_pack / retrieve / query_state / expand_query / status / inspect / doctor）只在精确核对、编译证据、诊断时用，走桥值得。

### 2.3 clarify 为什么永不折叠

源码 `tools/tool_search.py:144-146` 记录了官方 A/B 实测：

> ``clarify`` is deliberately absent: A/B showed deferring it collapsed structured-clarify usage (**18/18 -> 7/18**) — the ask-the-user affordance must be ambient, a stub is not enough.

折叠后模型懒得问，结构化提问使用率掉 61%。这是**行为代价**，不是 token 账。

---

## 3. 判定逻辑（供接手的 Agent 理解）

`tools/tool_search.py:150-166`：

```python
def is_deferrable_tool_name(name, defer_tools=None):
    if name in BRIDGE_TOOL_NAMES: return False          # 桥自身不折叠
    if defer_tools is not None and name in defer_tools: return True   # 显式命中
    if name in _core_tool_names(): return False          # 核心工具永不折叠
    toolset = _registry_toolset(name)
    return toolset is not None and (
        toolset.startswith("mcp-") or toolset not in _DIRECT_SURFACE_TOOLSETS)
```

**注意**：即使不写进 `defer`，插件工具在默认情况下也会被判定为可折叠（最后一行）。显式列表的作用是**反向锁定**——把要常驻的插件工具排除在外很难，所以做法是：把冷的写进 defer，热的靠「不在 defer 里 + toolset 判定」保持常驻。

---

## 4. 验证方法（可复现）

```bash
cd /j/HermesShared/data/hermes-agent
unset PYTHONPATH PYTHONHOME
.venv/Scripts/python.exe -c "
from tools.tool_search import is_deferrable_tool_name, load_config_readonly
cfg = load_config_readonly(); dt = cfg.effective_defer_tools
print('集合大小:', len(dt))
for n in ['lcm_grep','lcm_doctor','clarify','terminal','hindsight_recall']:
    print(n, is_deferrable_tool_name(n, dt))
"
```

预期输出：`集合大小: 28`，`lcm_grep False`，`lcm_doctor True`，`clarify False`，`terminal False`，`hindsight_recall False`。

2026-11-09 02:16 实测结果：**14 项断言全过，FAIL 0**。

---

## 5. 生效条件

**必须重启 Hermes 才生效**（配置在进程启动时读取）。

gateway 模式：`hermes gateway restart`
TUI/桌面：退出后重新启动。

**未验证项**：重启后实际 prefill 下降多少，需读 `agent.log` 里的 tool 统计行确认。

---

## 6. 怎么回滚

```bash
cp J:/HermesShared/data/config.yaml.bak_defer_20261109_015534 J:/HermesShared/data/config.yaml
# 然后重启
```

删除 `defer` 字段效果等同于回到官方默认集（19 项）。

---

## 7. 影响面

| 项 | 影响 |
|---|---|
| prefill token | 预计省 5,000-7,000/轮（lcm 9 个工具的 schema） |
| 工具可用性 | 9 个 LCM 工具需经 `tool_search` -> `tool_describe` -> `tool_call` 三步 |
| 行为风险 | 冷门工具可能被遗忘（已知代价，接受） |
| clarify | **无影响**（未折叠） |
| 其他插件 | 无影响 |

---

## 8. 待办

- [ ] 重启后读 `agent.log` 确认实际 token 降幅
- [ ] 若降幅不明显，检查 `threshold_pct: 10` 是否让 listing 预算过大（当前 listing_max_tokens 默认 4000）
- [ ] 考虑是否把 `kanban_*`（12 个）也折叠（未做，需评估使用频率）
