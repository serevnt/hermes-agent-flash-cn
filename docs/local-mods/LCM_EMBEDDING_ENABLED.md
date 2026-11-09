# LCM embedding 启用实录（2026-11-09）

> 脚本：
> - `scripts/local-mods/lcm_backfill.py`（warmup + backfill）
> - `scripts/local-mods/verify_lcm_semantic.py`（语义检索验证）
> 备份：`backups/lcm/lcm.db.bak_20261109_110343_prebackfill`（765 MB）

---

## 1. 做了什么（全部实测通过）

| 步骤 | 结果 |
|---|---|
| 备份 lcm.db | ✅ 765 MB → `..._prebackfill` |
| warmup 注册 embedding profile | ✅ `lcm_embedding_profile` 2 行 |
| backfill summary 语料 | ✅ **441 条全部向量化**，`failed: 0`，`remaining: 0` |
| 建表验证 | ✅ `lcm_embedding_meta` 441 行、`lcm_embedding_vectors` 441 行 |
| 编码耗时 | 200 条 6.4s + 241 条 10.9s（约 30ms/条） |

**表结构**：

```
lcm_embedding_profile  (2 行)   — provider/model/dim/dtype/corpus/…
lcm_embedding_meta     (441 行) — embedded_id, embedded_kind='summary', identity_hash, …
lcm_embedding_vectors  (441 行) — embedded_id, identity_hash, vec
lcm_embedding_binary   (0 行)   — 未使用（float32 走 vectors 表）
```

**profile 内容**：
- summary corpus：`fastembed` / `paraphrase-multilingual-MiniLM-L12-v2` / 384 维 / float32 / **441 条**
- chunk corpus：同模型 / 384 维 / **0 条**（message chunks 还没跑）

---

## 2. ⚠️ 关键澄清：跨会话语义检索是「设计上」不返回 summary 向量

跑验证脚本看到：

```json
{
  "coverage": "none",
  "degraded_to_fts": true,
  "degraded_reason": "broader scopes return raw-message hits only"
}
```

**这不是 bug，是 LCM 的设计**：

- **`session_scope=current`**：走 summary 向量 KNN（需要 `engine.current_session_id` 正确绑定）
- **`session_scope=session` / `all`**：**故意只返回原始消息命中**，不返回 summary 向量

源码：`tools.py` 的 `_lcm_grep_semantic`，在 KNN 之前按 scope 分流。

**验收标准要调整**：

| scope | 语义检索行为 |
|---|---|
| `current` | 走 summary 向量（**唯一能验证向量的路径**） |
| `session` / `all` | 原始消息命中（需要另补 362K 条 message chunks） |

---

## 3. 脚本环境拿不到 current_session_id

```
engine.current_session_id = ''    ← 脚本独立构造的 engine
```

`LCMEngine(config, hermes_home)` 只能拿到配置，**会话上下文是 Hermes 主进程注入的**。

因此**独立脚本无法验证 `session_scope=current` 的语义检索**——session_id 为空 → coverage=none。

**验证必须在 Hermes 主进程里做**（重启后直接 `lcm_grep(mode="semantic")`，不传 session_scope，默认就是 current）。

---

## 4. 踩过的坑（全部已修，脚本可复用）

| 坑 | 现象 | 解法 |
|---|---|---|
| 模块结构 | `mod.LCMConfig` / `mod.LCMEngine` AttributeError | 走子模块 `mod.config.LCMConfig`；`mod` 顶层可能连 `config` 属性都没有，要 `__import__("probe.lcm.config", ...)` |
| 未 warmup | `no current embedding profile is registered` | backfill 前必须先调 `_embedding_warmup_text(engine)` |
| `--corpus` 无效 | 传 `--corpus summaries` 返回 help | 该子命令只认 `[--apply] [--limit N]`，不传即默认 leaf summary |
| 工具函数名 | `_lcm_grep` 不存在 | 实际是 `lcm_grep`（无下划线前缀） |
| 返回类型 | 以为是 dict | 实际是 JSON 字符串，要 `json.loads(out)` |
| 前台超时 | 全量跑 > 600s | `--limit` + `--batches` 分批 |

---

## 5. 下一步（按价值排序）

### 5.1 重启后验证 current scope（必做，最快）

```
重启 Hermes
→ lcm_grep(query="城市旅人", mode="semantic")     # 默认 session_scope=current
→ 看返回 degraded_to_fts 是否为 false
```

**如果仍 degraded**：当前会话可能还没有 summary node（长会话要压缩过才有），或 profile 与当前会话不匹配。

### 5.2 是否补 message chunks（355,109 条）

**先别急着跑**：

- 跨会话（`session`/`all`）**设计上就不返回 summary 向量**
- 355K 条 × 30ms ≈ **3 小时**，纯向量约 545 MB
- 需要先验证 5.1，看 current scope 效果够不够

**要补的话**：

```bash
python scripts/local-mods/lcm_backfill.py --apply --corpus chunks --limit 2000 --batches 200
```

（本地 fastembed 不出本机，不需要 `--confirm-raw-text`）

### 5.3 需求 3（跨会话记忆）路径要重估

既然跨会话语义**默认不用 summary 向量**，需求 3 不能只靠 `lcm_grep(mode=semantic, scope=all)`。

**可能路径**：
- 先 `lcm_recall`（跨会话 summary 检索，另一条路径）
- 或自写重排层（需求 4 的「丙：重排」）

---

## 6. 回滚

```bash
cp J:/HermesShared/data/backups/lcm/lcm.db.bak_20261109_110343_prebackfill \
   J:/HermesShared/data/lcm.db
```

或只删向量表：

```sql
DROP TABLE IF EXISTS lcm_embedding_binary;
DROP TABLE IF EXISTS lcm_embedding_vectors;
DROP TABLE IF EXISTS lcm_embedding_meta;
DROP TABLE IF EXISTS lcm_embedding_profile;
```
