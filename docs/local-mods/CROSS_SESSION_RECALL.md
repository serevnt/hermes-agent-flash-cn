# 本地改造：跨会话记忆的可用路径（需求 3）

> 时间：2026-11-09
> 需求：3 — 跨会话记忆（方案 B：按需检索、不自动注入）
> 改动：`.env` 加 `LCM_RECALL_QUERY_TIMEOUT_S=20`
> 备份：`.env.bak_recalltimeout_20261109_142*`

---

## 1. 定案方案回顾

**方案 B**：不在新会话开头自动加载旧上下文（不污染、不多花 token），靠主动检索。

**现状**：Hermes 本来就没有跨会话自动注入——**方案 B 的"前半句"天然满足**。工作全在"检索质量"。

---

## 2. 实测：两条检索路径的真实表现

两个工具都能跨会话（`session_scope="all"`），但特性完全不同。以下全部是 **355,109 条消息 / 543 个 summary** 的真实数据。

### 2.1 `lcm_recall`（三 arm 串行：FTS + summary KNN + chunk KNN）

默认预算 **8.0s**（`LCM_RECALL_QUERY_TIMEOUT_S`）：

| 查询 | 8s 预算 | 20s 预算 |
|---|---|---|
| `城市旅人`（单词） | 3 hits ✅ 5.2s | — |
| `温蘅`（单词） | 3 hits ✅ 5.1s | — |
| `城市旅人 温蘅`（多词） | **0 hits** ❌ 8.0s 超时 | **5 hits** ✅ 17.4s |
| `Hermes fork 改造`（三词） | **0 hits** ❌ 8.0s | **0 hits** ❌ 20.0s |
| `iwin 视觉降级` | 5 hits ✅ 6.3s | 5 hits ✅ 4.6s |

**降级原因链**（读代码 + 实测确认）：

```
多词查询 → FTS arm 扫描慢 → 吃光 8s 总预算
        → FTS arm 返回 "full-text arm unavailable"
        → total_results = 0
```

**关键**：这不是"没找到"，是**预算耗尽**。`_lcm_recall_fts_arm()` 把 deadline 传给 `_lcm_grep_full_text_with_deadline()`，超时就直接 unavailable。

**已做的调整**：`.env` 里加 `LCM_RECALL_QUERY_TIMEOUT_S=20`，让多词查询至少能出结果（代价 15-17s）。

### 2.2 `lcm_grep`（单 arm，带四维重排）

| 查询 | 耗时（两次） | 结果 | 降级 |
|---|---|---|---|
| `城市旅人` | 3.05 / 3.05s | 20 | 无 |
| `iwin 视觉降级` | 4.97 / 5.00s | 20 | 无 |
| `城市旅人 温蘅` | 8.07 / 8.12s | 20 | 无 |
| `Hermes fork 改造` | 12.95 / 12.96s | 20 | 无 |

**稳定性**：两次测量几乎完全一致（±0.05s），**零降级**。

---

## 3. 结论：什么时候用哪个

| 场景 | 用哪个 | 理由 |
|---|---|---|
| **日常跨会话找东西**（"我记得跟你说过 XX"） | **`lcm_grep(query=..., sort="fourdim", session_scope="all")`** | 3-13s、稳定、多词可用、四维排序把最相关的顶上来 |
| **需要可引用的精确证据**（要 exact_ref / 原文 span 做引用） | `lcm_recall(detail="answer_ready")` | 只有它带 verified span 和 citable entries；代价是慢 |
| 当前会话内检索 | `lcm_grep`（默认 `session_scope="current"`） | 最快，3s 级 |

**一句话**：**`lcm_recall` 是「取证」工具，不是「找东西」工具。** 它设计目标是证据可引用性（`_lcm_recall_reference_shape`、`_lcm_recall_verified_span`、`_lcm_recall_citable_entries` 一整套校验），不是低延迟检索。

---

## 4. 为什么 `lcm_recall` 多词慢（技术原因，不是 bug）

`lcm_recall` 顺序跑三个 arm：

```
FTS arm（扫 355K 行消息，多词 AND 最慢）
   ↓
summary KNN arm（384 维向量，快）
   ↓
chunk KNN arm（chunk vectors 未 backfill，直接跳过）
   ↓
RRF 融合 + hydration + 可选 rerank
```

注释里写得很清楚（`config.py:714`）：

> Dedicated deadline for lcm_recall. It fans out three sequential arms (FTS + summary KNN + chunk KNN) plus fusion, hydration, and an optional rerank, so it needs more headroom than lcm_grep's single-arm query deadline.

**它是为"一轮问一次、要精确"设计的，不是为"反复查"设计的。**

---

## 5. 已知局限（不粉饰）

1. **多词查询 8-13s** 仍然偏慢。根因是 FTS5 在 35 万条上的 AND 扫描。要根治得上 trigram tokenizer 或倒排优化——**改动大、风险高，本轮不做**。
2. **`chunk vectors are unavailable`** —— message chunks 还没 backfill（355K 条，约 3 小时）。补了之后 recall 的 chunk arm 才生效，但 FTS arm 的耗时不会变。
3. **`Hermes fork 改造` 20s 仍 0 结果** —— 三个词全 AND 且分散在不同消息里，FTS5 天然查不到。这是 AND 语义的固有边界；用户实际检索时应该用更少的词。
4. **方案 4 的降级链（Hindsight 优先 → LCM 兜底）没实现在代码里** —— 那是调用侧策略。当前保证的是「LCM 兜底时，检索可靠 + 排序合理」。

---

## 6. 回滚

```bash
# 去掉 .env 里的 LCM_RECALL_QUERY_TIMEOUT_S=20（恢复默认 8s）
```

---

## 7. 给未来的建议（不在本轮范围）

若要让跨会话检索真正快起来，按性价比排序：

1. **FTS5 trigram tokenizer**（中文本就适合）——能大幅加速子串匹配，但需要重建索引（355K 条，约 10 分钟）
2. **补 message chunks 向量**（3 小时）——让 semantic/hybrid 路径覆盖原始消息，绕过 FTS 瓶颈
3. **给 `lcm_recall` 加 FTS 优先短路**——FTS arm 结果够了就跳过 KNN arm，省 2-5s；但要改 arm 编排逻辑，回归风险中等
