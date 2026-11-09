# 本地改造：压缩指针（需求 2）

> 时间：2026-11-09
> 需求：2 — 长久记忆（借鉴 openseek-lcm 的 compaction pointer）
> 改动：`plugins/hermes-lcm/compaction.py`、`plugins/hermes-lcm/engine.py`
> 备份：`backups/plugins/hermes-lcm_20261009_122823/`（compaction.py + engine.py）
> 验证：`scripts/local-mods/verify_compaction_pointer.py`（13 项断言全过）

---

## 1. 解决的问题

hermes-lcm 压缩后，**模型不知道自己丢了东西**。上下文被折叠成摘要，模型无法判断「刚才那段对话还在不在」，要么重复问用户，要么凭摘要猜细节。

openseek-lcm 用「压缩指针」解决这件事：压缩后的第一轮，自动注入一条**确定性**说明——丢了多少、丢的是哪段、怎么按确定路径找回来。

---

## 2. 改了什么

### 2.1 记录端：`compaction.py`

`compress()` 原本是 `_compress_impl()` 的一层薄包装（只做错误状态标记）。现在多记一笔：

```python
_before_node_id = self._pointer_max_node_id()   # 压缩前最大 node_id
_before_count = len(messages)                  # 压缩前消息条数
result = self._compress_impl(...)              # 原逻辑不动
# 成功后：比对条数差 + 查 node_id > before 的新 node
self._record_compaction_pointer(...)
```

新增两个方法：

| 方法 | 作用 |
|---|---|
| `_pointer_max_node_id()` | 查 `SELECT COALESCE(MAX(node_id),0) FROM summary_nodes`；失败返回 0（指针自动跳过） |
| `_record_compaction_pointer()` | 算 `removed = before_count - len(result)`；查新 node 的 node_id / depth / source_token_count / earliest_at / latest_at / session_id；挂到 `self._pending_compaction_pointer` |

**关键设计**：记录全程包在 `try/except Exception: pass` 里——**指针是锦上添花，记录失败绝不能影响压缩本身**。

### 2.2 注入端：`engine.py` 的 `LCMEngine`

新增 `select_context()`。这是 `ContextEngine` 基类的钩子：

```python
def select_context(self, request_messages, *, conversation_messages=None,
                   incoming_message=None, budget_tokens=0):
    pointer = getattr(self, "_pending_compaction_pointer", None)
    if not pointer:
        return None                          # 没有指针 → 请求一字不改
    self._pending_compaction_pointer = None   # 消费式，只注入一次
    out = list(request_messages)
    # 插在 system 区末尾（不打断 tool_call/tool 配对，不冒充用户）
    out.insert(insert_at, {"role": "system", "content": text})
    return out
```

**host 侧判定**（`agent/conversation_loop.py:1358`）：

```python
def _engine_overrides_hook(engine, name):
    return getattr(hook, "__func__", None) is not getattr(ContextEngine, name)
```

只要 LCMEngine 自己定义了 `select_context`（不是继承 ABC 的空实现），host 就会调它。已验证 ✅。

**返回值约束**（读源码确认）：
- 返回 `None` → 用原请求
- 返回**非空** `list[dict]` → **替换整个请求**（所以必须返回完整列表，不是增量）
- 返回 `[]` 或含非 dict → 视为非法，警告后用原请求

### 2.3 指针文本长什么样

实测渲染结果：

```
⟪LCM-COMPACTION-POINTER⟫ 上一轮发生过上下文压缩：移出活动上下文的原始消息 42 条
（压缩前约 88,000 tokens），已折叠进摘要节点 node_id=100, 101（depth=1，
覆盖 11-09 10:29→11-09 11:59）。需要原文时用 lcm_expand(node_id=...) 展开，
或用 lcm_grep(session_scope="session", session_id="20261109_test") 回查。
本条只是定位线索，不是对话内容，也不需要复述给用户。
```

**四项信息齐全**（用户要求）：移除条数 / seq 区间（用时间区间表达，因为 LCM 的 seq 是 `source_ids`） / token 估算 / 找回路径。

**约 200 字符 ≈ 100 tokens**，只在压缩后注入一次，成本可忽略。

---

## 3. 为什么用 `select_context` 而不是别的路径

| 备选 | 为什么不选 |
|---|---|
| 改 system prompt 构建 | 每次重建 prompt 都要判断，且压缩发生在轮次中间，prompt 已缓存，改它会破坏 prompt cache |
| 往消息列表 append | 会污染持久化 transcript（`select_context` 的返回值是**请求专用**，不落盘） |
| `pre_llm_call` hook | 那是全局 hook，拿不到 context engine 的压缩状态；且不能替换消息列表 |
| tool result 里夹带 | 模型容易当成工具输出，不当成上下文事实 |

`select_context` 的语义正好：**每个 provider 请求前跑、可以替换请求、不落盘**。

---

## 4. 验证（13 项断言全过）

```
1. select_context 已被 override                          ✅
2. 无指针 → 返回 None                                     ✅
3. 有指针 → 注入一条 system，原消息未破坏                  ✅
   含移除条数 / node_id / token 估算 / 两条找回路径        ✅
4. 消费式：第二次调用不再注入                              ✅
5. nodes 为空 / request_messages 为空 → 返回 None         ✅
6. 只有一条 system 时插在 index 1                         ✅
```

**记录端 SQL 实测**（只读查生产库）：

```
当前 max node_id = 625
模拟 before=623 时新增 node 条数: 2
  node_id=624 depth=0 src_tokens=110404 session=20261109_015212_9508c7
  node_id=625 depth=1 src_tokens=15071  session=20261109_015212_9508c7
```

字段齐全，SQL 正确。

---

## 5. 生效条件

**必须重启 Hermes**——插件是启动时加载的，改 `.py` 不会热生效。

重启后第一次自动压缩（或手动 `/compress`）之后，下一轮的请求里就会多一条 system 消息。

**怎么验证真的生效**：

1. 看日志有没有指针注入：`grep "COMPACTION-POINTER" agent.log`
2. 或者手动 `/compress` 一次，看下一轮模型的回复是否主动提到「上下文被压缩过」
3. 直接查：`lcm_grep(query="COMPACTION-POINTER")` —— 如果指针进了消息列表，应该能搜到（取决于是否落库；`select_context` 的返回值不落盘，所以**搜不到是正常的**）

**推荐验法**：开新的长会话，跑到自动压缩触发，然后问「刚才我们聊到哪了、有没有丢东西」——有指针时模型会明确说出丢了 N 条、node_id 是多少。

---

## 6. 影响面

| 项 | 影响 |
|---|---|
| 压缩逻辑 | **不变**（只加了一层记录包装，失败静默） |
| 每轮 token | 普通轮 **0**（没有 pending pointer 就返回 None，请求一字不改） |
| 压缩后那一轮 | +约 100 tokens（一条 system 消息） |
| 持久化 transcript | **不变**（`select_context` 返回值是请求专用） |
| prompt cache | 压缩后那一轮会变（必然，压缩本身已经改了上下文）；之后恢复 |

---

## 7. 回滚

```bash
BK=J:/HermesShared/data/backups/plugins/hermes-lcm_20261009_122823
cp $BK/compaction.py $BK/engine.py J:/HermesShared/data/plugins/hermes-lcm/
# 重启 Hermes
```

---

## 8. 已知限制（不是 bug）

1. **provider 重试不重复注入**：指针消费一次就清空，同一次请求的重试不再带。设计如此——重试是同一轮的重复发送，重复注入反而浪费。
2. **指针不落盘**：`select_context` 的返回值是请求专用的，不进 transcript。所以 `lcm_grep` 搜不到指针文本。
3. **只报本次压缩**：如果连续两次压缩之间模型没发过请求（罕见），后一次会覆盖前一次的指针，只报最后一次。要改的话把 `_pending_compaction_pointer` 改成列表，但收益不大。
4. **深度 condensation**（D1/D2 节点生成）也会计入「新增 node」，指针里会显示多个 node_id。
