# Hermes 改造需求（2026-11-09 拍板）

> 逐项过完需求后的定案。**需求层**，不含实现细节。实现方案另文。

## 总原则

- 除 Hindsight 外，所有外部能力**内建为 Hermes 原生插件**，MCP 保留为可选方案。
- 所有可变量（路径、服务地址、开关、人格档）**走 Setting 配置**，不写死。

---

## 需求 1：技能折叠

- 188 个技能，索引 5,236 tokens 每轮全量注入。
- `compact_skill_categories()` 机制已存在，但只在 `is_coding and config_mode == "focus"` 时生效。
- **目标**：加配置开关，让"只列名字不列描述"在任何场景可用；低频技能移 `.archive`（不删）。

## 需求 2：长久记忆（借鉴 openseek-lcm）

对照对象：`openseek-lcm@0.2.0`（DSH 侧，Pub-Polaris，移植自 `opencode-lcm`）

**借鉴点一：压缩指针（compaction pointer）**
压缩发生后的第一轮，自动注入一条确定性指针，包含：
- 被移除的消息条数
- seq 区间
- token 估算
- 回到原文的确切路径（`lcm_expand` 用哪个节点 / `lcm_grep --scope session`）

Hermes 的 hermes-lcm 现在**没有这个**——压缩后模型不知道自己丢了东西。

**借鉴点二：确定性重排，不用纯相似度**
openseek-lcm 的 README 明确写："`lcm_retrieval_debug` 与 `/lcm debug` 已标记 Deprecated —— 相似度召回默认关闭"。
它改用四维确定性打分：**覆盖度 / 短语命中 / 时效 / 来源种类**。

**结论**：长久记忆的关键不是"存了多少"，是"丢失时有没有人告诉你，以及告诉你之后能不能按确定路径找回来"。

## 需求 3：跨会话记忆

**方案 B：按需检索，不自动注入。**

- 不在新会话开头自动加载旧上下文（避免污染、避免每轮多花 token）。
- 靠 `lcm_recall` 主动检索。
- **核心工作量在检索质量**：语义检索必须真能用。

## 需求 4：本地记忆筛选

**方案丙：检索结果重排 + 降级链。**

```
检索请求
 ├─ Hindsight 在   → 走 Hindsight
 └─ Hindsight 不在 → 走 LCM 本地（FTS5 + 可选 embedding）
        ↓
   本地重排（覆盖度 / 短语命中 / 时效 / 来源种类）
        ↓
   返回 top N
```

- 重排**只决定"先看哪几条"，不决定"给不给"**——搜错了还能往后翻。
- 不做 LLM 判断式筛选（openseek-lcm 实战后关了相似度召回，同一道理）。

## 需求 5：人格系统强化（温蘅当角色卡）

**并存 + Setting 可切换**（工作模式 / 陪伴模式）。

按层的取舍：

| 层 | 内容 | 是否移植 |
|---|---|---|
| 人格定义 | 温蘅角色卡（说话方式、正典、关系锚点） | ✅ 要 |
| 机制层 | 9 维情绪机、半衰期衰减、单轮 cap | 待定（先并存最小版） |
| 记忆层 | 三档召回、`her_own_words.jsonl` 原话账本、经历层 | 待定 |
| 触发层 | TRIGGERS 正则 + 语义规则 | 待定 |
| 后处理 | R1 句号、ECP 视角投影 | 待定 |

**约束**：Hermes 同时是工作助手，陪伴人格不能污染工作场景，所以必须可切换，不是替换。

## 需求 6：iWIN 内置

**方案 A：全套转 Hermes 原生插件。**

| 项 | 实测 |
|---|---|
| iWIN 依赖 | 99 包 / 393 MB |
| 重依赖 | **无** torch / onnxruntime / transformers / cv2 |
| 推理方式 | 纯 HTTP 调视觉 API，不在本地跑模型 |
| 工具数 | 23 个 |
| 现有接入 | stdio MCP（`iwin-core.exe`） |

**外部服务依赖无法消除，但要写进 Setting 可配**：
- 视觉 API 地址（默认 `http://127.0.0.1:8080/v1`，备选 1241 / DeepSeek 官方）
- ASR 服务地址（默认 `http://127.0.0.1:8766`）
- ffmpeg 路径（`media_info` / `read_video` 依赖）
- MCP 挂载方式保留但默认关，可随时切回

---

## 共同前置（跨多项需求）

| 前置 | 涉及需求 | 现状 |
|---|---|---|
| **embedding 修好** | 2 / 3 / 4 | `fastembed` 未装，`.env` 却写着 provider=fastembed，语义检索全程静默降级 |
| **Hermes `.venv` 补基础包** | 2 / 3 / 4 / 6 | 实测缺 numpy / PIL / openai |
| **Setting 配置层** | 5 / 6 | 需要一套统一的配置读写 + 热加载机制 |

## 依赖顺序

```
① 补依赖（numpy / PIL / openai / fastembed + onnxruntime + tokenizers + huggingface_hub）
     ↓
② 修 embedding，实测 lcm_grep(mode=semantic) 真走向量
     ↓
③ 重排层（需求 4）+ 压缩指针（需求 2）  ← 都依赖 ②
     ↓
④ 技能 compact 开关（需求 1）  ← 不依赖 ②，可并行
⑤ Setting 配置层 + 人格切换（需求 5）
⑥ iWIN 内建（需求 6）
```
