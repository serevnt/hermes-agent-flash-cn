# 本地改造：依赖补装与 LCM embedding 状态

> 时间：2026-11-09
> 对应需求：六项需求的共同前置 P1 / P2

---

## 1. 装了什么

```bash
uv pip install --python .venv/Scripts/python.exe \
  numpy fastembed onnxruntime tokenizers huggingface_hub
```

**结果**（venv `J:/HermesShared/data/hermes-agent/.venv`）：

| 包 | 版本 |
|---|---|
| numpy | 2.5.4 |
| fastembed | 0.9.1 |
| onnxruntime | 1.31.1 |
| tokenizers | 0.23.3 |
| huggingface-hub | 1.33.0 |
| **pillow** | **12.2.0 → 13.0.0（被动升级）** |

**注意**：pillow 是被 fastembed 依赖链带上来的升级，**需验证 iwin 图像处理不受影响**。

---

## 2. embedding 实测（fastembed 独立可用）

**模型**：`sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`（`.env` 里 `LCM_EMBEDDING_MODEL` 指定的）

**镜像**：`HF_ENDPOINT=https://hf-mirror.com`（国内下载 11 秒完成）

**结果**：

```
模型加载 11.0 s
编码 2 条 0.01 s
维度: 384
余弦相似度(无关句: "城市旅人是一个视觉小说项目" vs "The quick brown fox...") = -0.0448
余弦相似度(近义句: "城市旅人是一个视觉小说项目" vs "...工程") = 0.9828
```

✅ **语义区分正确**：近义 0.98、无关 -0.04。

**模型缓存位置**：
- fastembed 默认探到 `C:\Users\swall\AppData\Local\Temp\fastembed_cache`（**临时目录，会被清理**）
- 已手动复制到 **`C:\Users\swall\.cache\fastembed`（241 MB，持久）**
- LCM 的 `FastembedProvider` 默认 `cache_dir = Path.home()/".cache"/"fastembed"`（`embedding_provider.py:73`）→ **与手动复制的位置一致** ✅

---

## 3. ⚠️ 重要发现：LCM 生产检索路径可能不用 embedding

### 3.1 lcm.db 里没有任何向量存储

16 张表的完整清单：

| 表 | 行数 |
|---|---|
| messages | 355,109 |
| messages_fts（+config/data/docsize/idx） | FTS5 全文索引 |
| summary_nodes | 543 |
| nodes_fts（+config/data/docsize/idx） | 543 |
| metadata | 3,764 |
| lcm_lifecycle_state | 196 |
| lcm_migration_state | 4 |

**`messages` 表列**：`store_id, session_id, source, conversation_id, role, content, tool_call_id, tool_calls, tool_name, timestamp, token_estimate, pinned, ...` —— **无 embedding 列**

**`summary_nodes` 表列**：`node_id, session_id, depth, summary, token_count, source_token_count, source_ids, source_type, created_at, earliest_at, latest_at, expand_hint` —— **无 embedding 列**

**结论**：embedding **不持久化在 lcm.db**。

### 3.2 `LCM_EMBEDDINGS_ENABLED` 的使用点几乎全在 benchmark

```
./benchmarking/longmemeval.py           ← 评测脚本
./benchmarking/h5_state_semantic_replay.py
./benchmarking/state_embedding_backfill.py
./benchmarking/w3b_diversity_replay.py
./benchmarks/qa-harness/...             ← QA 测试桥
./command.py:64                          ← 仅 import 错误类型
```

**生产路径**：`tools.py:3055` 的 `_lcm_grep_semantic()` 是 semantic 模式入口，它搜 `embedded summaries`（工具描述原文）。

### 3.3 待验证（必须重启）

**当前会话的 LCM 插件是旧进程加载的**（无 fastembed），所以现在调 `lcm_grep(mode='semantic')` 一定降级为全文。**无法在重启前验证**。

**重启后需要做的验证**：

1. 调 `lcm_grep(query="城市旅人", mode="semantic")`，看返回里有没有语义命中（非关键词命中）的结果
2. 调 `lcm_doctor`，看它报的 embedding 状态
3. 若仍降级，用 `lcm_grep(mode="hybrid")` 对比 hybrid 与 full_text 的结果差异
4. 若 semantic 完全不可用 → **说明这版 LCM 的 semantic 只是软降级占位**，需要按需求 3（跨会话记忆）自己实现重排层

---

## 4. 待办

- [ ] 重启后验证 `lcm_grep(mode='semantic')` 是否真走 embedding
- [ ] 验证 pillow 13.0.0 对 iwin 无影响（`iwin-core.exe` 的图像工具）
- [ ] 若 semantic 无效，考虑把 embedding 能力接到自研重排层（需求 3）

---

## 5. 回滚

**卸载新增包**（不影响已有的 pillow 版本之外的依赖）：

```bash
uv pip uninstall --python J:/HermesShared/data/hermes-agent/.venv/Scripts/python.exe \
  fastembed onnxruntime tokenizers huggingface_hub
```

**注意**：pillow 已被升到 13.0.0，若需回退：

```bash
uv pip install --python J:/HermesShared/data/hermes-agent/.venv/Scripts/python.exe pillow==12.2.0
```

**删除模型缓存**（241 MB）：

```bash
rm -rf C:/Users/swall/.cache/fastembed
```
