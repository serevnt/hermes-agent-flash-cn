# trigram 中文检索索引

## 问题

`lcm.db` 的 `messages_fts` 用 FTS5 默认 `unicode61` 分词器。**unicode61 不对中文分词**——整段连续中文被当成一个 token，所以：

```sql
-- 实测：4 条测试数据的独立复现
unicode61  '城市'      -> 0 命中
unicode61  '城市旅人'   -> 0 命中      ← 索引里根本没有这个词
unicode61  '温蘅'      -> 0 命中
unicode61  'Hermes'   -> 1 命中      ← 英文正常
```

后果：`search_query.py::requires_like_fallback()` 对任何含 CJK 的查询返回 `True`，检索**完全绕过 FTS 索引**，落到 `store.py::_search_like()`：

1. `content LIKE '%词%'` 全表扫描（357,659 行 / 1.96 亿字符）
2. `ORDER BY` 里对**每一行**算打分：

```sql
((LENGTH(LOWER(content)) - LENGTH(REPLACE(LOWER(content), LOWER(?), ''))) / NULLIF(LENGTH(?), 0))
```

即每行 2 次 `LOWER` 全内容转换 + 2 次 `REPLACE` + 长度计算。这是 8-13 秒的来源——**不是数据库大，是排序表达式在全表上跑**。

## 方案

建一个 trigram 分词器的 FTS5 表，与 `messages` 同步，让 CJK 查询走索引。

```sql
CREATE VIRTUAL TABLE messages_fts_trigram USING fts5(
    content,
    content='messages',
    content_rowid='store_id',
    tokenize='trigram'
);
```

三个触发器与现有 `msg_fts_*` 一一对应（`INSERT` / `DELETE` / `UPDATE OF content`），保证新消息自动进索引。

### 关键坑：trigram 裸词 ≠ 短语

| 查询 | 裸词 | `"加引号"` 短语 | LIKE |
|---|---|---|---|
| `Hermes fork` | **359** ❌ | **20** ✅ | 20 |
| `城市旅人` | 2971 ✅ | 2971 ✅ | 2971 |
| `温蘅系统` | 807 ✅ | 807 ✅ | 807 |

裸 trigram 是**逐 3 字符片段 AND**（`Her`/`erm`/`rme`… 各出现即算命中），不是短语匹配。**必须用引号包成短语查询**才能与 LIKE 语义对齐。中文因无空格分段，裸词与短语恰好等价，英文会放大。

### 关键限制：最少 3 字符

```
'温蘅'  -> 0 命中    （2 字，trigram 索引不到）
'城市'  -> 0 命中
'Kar'   -> 276 命中  （3 字符 OK）
```

2 字符查询必须继续走 LIKE。代码里按 term 长度分流。

## 实施

脚本：`scripts/local-mods/build_trigram_index.py`

```bash
python build_trigram_index.py --check    # 体检：表/触发器是否存在
python build_trigram_index.py            # 建表（缺则建）+ 补触发器
python build_trigram_index.py --rebuild  # 强制重建索引内容
```

实测建索引 **132 秒**（357,659 行），库 734 MB → 1294 MB（**+560 MB**）。

### 代码改动

`store.py`：

1. 新增 `MessageStore._trigram_index_available()` —— 查 `sqlite_master` 探测表是否存在，结果缓存在实例 `_trigram_probe` 上（索引由运维脚本建一次，进程内不变）。**任何异常返回 `False`，完全退回原 LIKE 路径**，保证老库不炸。
2. `_search_like()` 构建 `where` 时按 term 分流：

```python
if trigram_ready and len(stripped) >= 3:
    # 走索引
    like_clauses.append("store_id IN (SELECT rowid FROM messages_fts_trigram "
                        "WHERE messages_fts_trigram MATCH ?)")
    args.append('"' + stripped.replace('"', '""') + '"')   # 必须引号短语
else:
    like_clauses.append("content LIKE ? ESCAPE '\\'")
    args.append(f"%{escape_like(term)}%")
```

候选集从 357K 缩到几百到几千行，后续 `count_expr` 打分只在候选集上跑。

**`_search_like` 后面的排序、`add_rows`、分页、`candidate_cap` 逻辑一律未动。**

## 实测

索引建成后，同一 SQL 形态（含原打分表达式）对照：

| 查询 | trigram | LIKE | 加速 | 结果一致 |
|---|---|---|---|---|
| `城市旅人` | 322.7 ms | 841.7 ms | 3× | 20/20 ✅ |
| `温蘅系统` | 129.1 ms | 715.3 ms | 6× | 20/20 ✅ |
| `压缩指针` | **4.0 ms** | 613.5 ms | **153×** | 20/20 ✅ |
| `记忆系统` | 54.0 ms | 700.2 ms | 13× | 20/20 ✅ |
| `Hermes fork` | 8.0 ms | 620.6 ms | 78× | 20/20 ✅ |

完整 `MessageStore.search(sort='fourdim')` 链路：

| 查询 | 耗时 | 命中量 |
|---|---|---|
| `压缩指针` | **12.0 ms** | — |
| `Hermes fork` | 253.5 ms | — |
| `温蘅系统` | 735.0 ms | 807 |
| `城市旅人` | 1709.3 ms | 2971 |
| `城市`（2字，LIKE） | 3349.1 ms | — |

**命中量大的查询仍然慢**：`城市旅人` 命中 2971 条，打分表达式要在 2971 行上各跑 4 次 `LOWER`/`REPLACE`。要再快得改打分方式（如用 trigram 的 `bm25()` 替代自算 count），本轮未做。

## 验证方法

```bash
# 索引与触发器状态
python scripts/local-mods/build_trigram_index.py --check

# 结果一致性（脚本内置，逐条比对 trigram 短语 vs LIKE）
python scripts/local-mods/build_trigram_index.py --rebuild
```

## 回滚

```sql
DROP TRIGGER IF EXISTS msg_trigram_insert;
DROP TRIGGER IF EXISTS msg_trigram_delete;
DROP TRIGGER IF EXISTS msg_trigram_update;
DROP TABLE IF EXISTS messages_fts_trigram;
```

代码回滚：`backups/plugins/hermes-lcm_trigram_20261109_132046/store.py`

库备份：`backups/lcm/lcm.db.bak_20261109_131317_pretrigram`（734 MB）

代码在无索引时会自动退回 LIKE（`_trigram_index_available()` 返回 False），所以**删表也不会崩**。

## 影响面

- `lcm_grep` 的 CJK 查询路径（所有中文检索）
- `lcm_recall` 的 FTS arm（同样走 `search()`）
- 库体积 +560 MB
- 写放大：每条消息 insert 多写一份 trigram 索引
- 未触及：英文 FTS 路径（`messages_fts` 照旧）、summary 检索（`nodes_fts`）、向量检索
