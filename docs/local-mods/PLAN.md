# Hermes 改造整体计划

> 本文是可核查的改造总账，不是单点清单。每一行都能对应到磁盘上的实证。
> 状态：🔴待做 / 🟡进行中 / 🟢已完成

## 零、已发现但未修的缺陷（最高优先，都是"配了但没生效"）

| ID | 缺陷 | 实证 | 影响 | 状态 |
|---|---|---|---|---|
| D1 | `fastembed` 未安装，但 `.env` 里 `LCM_EMBEDDING_PROVIDER=fastembed` | import 报 ModuleNotFoundError | **LCM 语义检索全程静默降级为全文检索** | 🔴 |
| D2 | `onnxruntime` / `tokenizers` / `numpy` / `huggingface_hub` 全缺 | 同上 | D1 的前置依赖 | 🔴 |
| D3 | `snowballstemmer` 缺失导致 tool_search 折叠静默跳过 | 日志 `Tool search assembly skipped` | 每轮多花 8,756 tokens | 🟢 已装 3.1.1 |
| D4 | 技能索引把目录级备份当独立技能列出 | 索引出现 `.bak_20261013_0214` 等 3 条 | 索引噪声 | 🟢 已移入 .archive |
| D5 | `agent.personalities` 塞了 14 个人格预设 | config.yaml L34-47 | 若注入则纯浪费；需实测确认 | 🔴 待查 |

## 一、每轮固定成本（改造对象）

| # | 组件 | 实测成本 | 说明 |
|---|---|---|---|
| 1 | 内建工具 schema | 20 个 / 12,529 tokens | 折叠后；折叠前 32 个 / 21,285 |
| 2 | 插件工具 lcm_* | **15 个**（未计入上面） | 子进程测不到，插件在主进程注册 |
| 3 | 插件工具 hindsight_* | 3 个 | recall / retain / reflect |
| 4 | 技能索引 | 188 个 / 5,236 tokens | 15,709 字符 |
| 5 | 内置记忆 | 4,271 字符（94% 满） | 上限 4,500 |
| 6 | 用户档案 | 2,166 字符（98% 满） | 上限 2,200 |
| 7 | SOUL 人格 | 全量注入 | — |
| 8 | 环境提示 + 时间戳 hook | — | 每轮必过 |

## 二、改造项全表

### A. 依赖与失效配置（地基）
- **A1** 装 fastembed + onnxruntime + tokenizers + numpy + huggingface_hub，修好 LCM 语义检索
- **A2** 装完实测 `lcm_grep(mode='semantic')` 是否真走向量
- **A3** 查 `personalities` 14 个预设是否进 prompt，进则删

### B. 工具面改造
- **B1** `tools.tool_search.defer` 写满（当前无此字段，用默认保守集）
- **B2** 常驻集定为 6 个：terminal / memory / clarify / read_file / patch / tool_search
- **B3** 插件工具分层：lcm 15 → 4 常驻（grep/recall/recent/expand）+ 11 按需
- **B4** hindsight 3 个全留（都是高频）
- **B5** 单个超大 schema 精简：`manage_connections` 4,477 / `delegate_task` 4,634 / `skill_manage` 3,665 / `execute_code` 3,033
- **B6** `threshold_pct: 10` 复查（决定折叠激活门槛）

### C. 自动筛选（新建）
- **C1** L1：BM25 对"工具名+描述"与用户消息打分，纯 Python 零依赖
- **C2** L2：中文意图词表（看图→vision，改文件→patch，跑命令→terminal）
- **C3** L3：embedding 检索，复用 A1 装的模型
- **C4** 保守策略：相似度接近时多给几个，宁可多不可漏
- **C5** 会话级缓存：同话题连续轮次工具面不变，保前缀缓存
- **C6** 降级链：L3 挂→L2→L1→全给

### D. 技能改造
- **D1** `coding_context.py` 的 `compact_skill_categories()` 目前只在 `is_coding and config_mode=="focus"` 生效，需加配置开关让任意场景可用
- **D2** 188 → 高频/低频分类（可交给本地小模型打标）
- **D3** 低频移 `.archive`（不删，可随时移回）

### E. 记忆改造
- **E1** 内置记忆 4,271/4,500 已 94% 满，压缩到 2,000 上下
- **E2** 迁出的内容进 Hindsight（不能丢）
- **E3** 用户档案 2,166/2,200 同理
- **E4** 记忆分层策略：铁律留内置，项目/案例/流程迁 Hindsight

### F. 人格改造
- **F1** SOUL.md 分层：恒常人格核心 vs 情境段（按平台/模式加载）
- **F2** 实测 SOUL 占多少 tokens

### G. 跨会话记忆（原始四项诉求之一）
- **G1** LCM 库现状核查（1,239 会话 / 335,515 消息）
- **G2** `lcm_recall` 检索质量实测（能否捞回三个月前某句话）
- **G3** 检索质量不达标时的改进（换 embedding、调 RRF 权重）

### H. 持久记忆（原始四项诉求之二）
- **H1** Hindsight 76k units 的召回质量实测
- **H2** memory 与 Hindsight 的职责边界写清
- **H3** 双写策略：memory 工具 + retain

### I. 与 DSH 的对照（能力缺口）
- **I1** tool-folder 的 RRF 融合、意图词表扩展、camelCase 拆分、CJK bigram 过滤 → 抽思路进 C
- **I2** tool-folder 的 budget 控制（每次给几个工具的上限）
- **I3** tool-folder 的 rerank（body-level 重排）
- **I4** ChainGuard 防火墙（工具调用链安全）→ 评估是否需要

## 三、依赖图

```
A1 装依赖
 ├→ A2 验证语义检索
 │   └→ C3 L3 可用
 ├→ G3 跨会话检索质量
 └→ H1 持久记忆质量

B1 defer 配置（不依赖 A）
 └→ B3 插件工具分层需先确认插件工具是否受 defer 管

C 自动筛选
 ├→ 依赖 A1（L3）
 └→ 依赖 D2（技能也能一起筛）

D1 技能 compact（不依赖 A/B/C，可独立做）
E 记忆分层（不依赖其他）
F 人格分层（不依赖其他）
```

## 四、可独立并行的工作

| 谁 | 做什么 | 依赖 |
|---|---|---|
| 我 | A1 装依赖 + A2 验证 | 无 |
| 我 | B1/B2 defer 配置 | 无 |
| 我 | D1 技能 compact 开关 | 无 |
| **本地小模型** | D2 188 个技能分类打标 | 无（纯读+判断） |
| **本地小模型** | 读 tool-folder 3904 行出摘要 | 无（纯读） |

## 五、尚未确定、需要你拍板

1. 常驻集 6 个够不够，还是要更少（只留 terminal/memory/clarify）
2. lcm 15 个留 4 个，还是全按需
3. 自动筛选的 L3 用 fastembed，还是保持纯 BM25
4. 内置记忆压到多少（2000？还是更少）
5. SOUL 要不要分层
