# 本地改造：技能索引折叠（技能级）

> 时间：2026-11-09
> 影响文件：`agent/prompt_builder.py`、`agent/system_prompt.py`、`config.yaml`（`agent.skill_compact_skills`）
> 备份：
> - `cache/scratch/hermes_baks/system_prompt.py.bak_skillcompact_20261109_020337`
> - `config.yaml.bak_compactskills_20261109_023326`
> 脚本：`cache/scratch/apply_compact_skills.py`、`verify_compact_e2e.py`

---

## 1. 改了什么

### 1.1 新增能力：技能级折叠

上游只有**分类级**折叠（`compact_skill_categories`），粒度太粗——`devops` 里既有 `searxng-local`（高频）也有 `hermes-desktop-setup`（低频），无法区分。

本次新增 **`compact_skills`**（技能名级）：命中名单的技能在系统提示索引里只显示名字，**省略描述**。

**代码改动 4 处**（`agent/prompt_builder.py`）：

| 位置 | 改动 |
|---|---|
| `build_skills_system_prompt()` | 新增参数 `compact_skills` |
| `_build_skills_system_prompt_inner()` | 新增参数 + **加入 cache_key**（避免缓存串味） |
| `_render_skills_index()` | 新增参数；命中名单的 `desc` 置空 → 渲染成 `- name` |
| `_skills_prompt()`（`agent/system_prompt.py`） | 从 config 读 `agent.skill_compact_skills` 传入 |

### 1.2 绕过 coding/focus 门槛

上游 `RuntimeMode.compact_skill_categories()` 有硬门槛：

```python
if not self.is_coding or self.config_mode != "focus":
    return frozenset()
```

本地改造在**调用点**（`_skills_prompt`）加一层：`agent.skill_compact_categories` / `agent.skill_compact_skills` 里的条目**无条件折叠**，不依赖编程姿态。

`RuntimeMode` 本身未改（避免每次读配置的开销），改动集中在调用点。

### 1.3 安全设计

- 全链路 `try/except`：配置读失败、类型错误一律降级为 `frozenset()`，**绝不影响会话启动**
- 只省略描述，**不删除任何条目**（技能仍可 `skill_view` 正常加载）
- 名单里的技能名若不存在，写配置时会被过滤（防止配置漂移）

---

## 2. 折叠名单（71 个）

策略：**专项/低频/名字自解释** 的折叠；核心高频保留完整描述。

**折叠（71 个，按主题）**：

| 主题 | 技能 |
|---|---|
| 音乐/视频生成 | ace-step-1-5-local, chiptune-composition, ltx25-windows-rocm, minimax-h3-comfyui, music3-windows-rocm |
| 创意/写作 | pixelization-gan-pipeline, op-production, pv-master-timeline, novel-prose-deslop, longform-fiction-rewrite, companion-fresh-elements, character-canon-design |
| 游戏素材/UI | city-map-ghibli-pipeline, retro-game-{asset-format-reverse-engineering,image-upscale,launch-repair,magpie-upscale,video-upscale}, oni-game-text-translation, wechat-minigame-3d, threejs-game-prototype, game-numeric-cloning, game-ui-ai-training, game-character-sprite-pipeline, html2figma-plugin-maintenance, design-visual-diff, fgui-{dict-reverse-engineering,game-ui-modification,modification-review}, moa-od-game-ui-pipeline, gameui-{corpus-evidence,criteria-pipeline,criteria-judgment}, interaction-analysis, game-interaction-design |
| 文档/研究 | video-text-extraction, ai-companion-platform-teardown, ocr-and-documents, microsoft-office, handoff-documentation |
| 开发流程/元技能 | mcp-builder, hermes-desktop-plugins, kimi-webbridge, chatbot-platform-integration, llm-output-control, local-patch-maintenance, multi-session-intent-recall, timeout-notification, using-superpowers, dispatching-parallel-agents, executing-plans, brainstorming, receiving-code-review, requesting-code-review, verification-before-completion, systematic-debugging |
| 模型评估/选型 | llm-judge-validation, local-inference-benchmarking, local-multigpu-planning, cloud-llm-billing-quota, model-selection-research, hindsight-multiprovider-chain |
| Hermes 运维（低频） | hermes-backup, hermes-desktop-setup, hermes-version-desync-fix, hermes-to-dsh-migration, background-task-supervision |
| 数据处理 | large-dataset-batch-processing, wechat-db-extraction, llm-backend-proxy, local-llm-writing-dispatch |

**明确保留描述（不折叠）**：

`hermes-agent`、`iwin-vision-toolkit`、`hermes-multi-model-orchestration`、`citytravler-companion-system`、`citytravler-storyline-design`、`renpy`、`renpy-development`、`renpy-city-travel`、`renpy-llm-game`、`searxng-local`、`hermes-error-recovery`、`hermes-performance-diagnostics`、`hermes-memory-system`、`local-vlm-ops`、`qwen35-llama-cpp-hip-deploy`、`qwen35-llama-vision-deploy`、`strata-inference-engine`、`strata-r9700-deploy`、`vllm-amd-wsl2`、`wsl2-windows-docker`、`windows-native-cli-from-msys`、`cn-ai-model-download`、`comfyui`、`time-consciousness`、`hindsight-io-rules`、`hindsight-backend-diagnostics`、`hindsight-bank-sql-migration`、`writing-plans`、`kanban`、`kanban-codex-lane`、`coding-agent-delegation`、`web-recon-and-batch-crawl`、`kimi-web-search`、`web-research-fallbacks`、`companion-roleplay-interaction`、`rocm-stack-version-review`、`debugging-hermes-tui-commands`、`hermes-lcm`、`hermes-atlas`、`hermes-provider-management`、`hermes-mcp-configuration`

---

## 3. 实测收益

**脚本**：`cache/scratch/verify_compact_e2e.py`（从真实 config 读，走真实 `build_skills_system_prompt`）

```
config 读到: skill_compact_skills=71 条, skill_compact_categories=0 条
defer 清单: 28 条

不折叠: 10,268 字符 ≈ 2,567 tokens
折叠后: 6,833 字符 ≈ 1,708 tokens
节省  : 3,435 字符 ≈ 858 tokens (33%)

名单里名字丢失的: 无 ✅
核心技能描述丢失: 无 ✅
```

**结论**：技能索引 **省 858 tokens/轮（33%）**，且零能力损失。

---

## 4. 怎么回滚

**只回滚名单（保留代码能力）**：

```bash
cp J:/HermesShared/data/config.yaml.bak_compactskills_20261109_023326 J:/HermesShared/data/config.yaml
```

或直接把 `agent.skill_compact_skills` 改成 `[]`（代码保持，效果为零）。

**完全回滚代码**：

```bash
cp J:/HermesShared/data/cache/scratch/hermes_baks/system_prompt.py.bak_skillcompact_20261109_020337 \
   J:/HermesShared/data/hermes-agent/agent/system_prompt.py
cd J:/HermesShared/data/hermes-agent && git checkout -- agent/prompt_builder.py
```

（`prompt_builder.py` 的改动未单独备份，用 git 还原；它在版本控制内且本轮未提交。）

---

## 5. 生效条件

**必须重启 Hermes**（配置与代码都在进程启动时加载）。

---

## 6. 待办

- [ ] 重启后读 `agent.log` 确认真实 token 降幅（估算值需实测校准）
- [ ] 观察模型是否会漏用被折叠的技能（尤其 `systematic-debugging`、`verification-before-completion` 这类流程技能——如果发现忘了用，把它们从名单移除）
- [ ] 考虑把 `compact_categories` 也启用（当前为 0 条）
