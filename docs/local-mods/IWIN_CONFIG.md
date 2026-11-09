# 本地改造：iWIN 配置结构化（需求 6 第一步）

> 时间：2026-11-09
> 需求：6 — iWIN 内置；用户明确要「ffmpeg 路径、视觉 API、ASR 地址全部写成 Setting 可配置项」
> 脚本：`J:/HermesShared/data/scripts/local-mods/{add_iwin_block,sync_iwin_config}.py`

---

## 1. 改了什么

### 1.1 config.yaml 新增顶层 `iwin:` 段

原来 iWIN 的配置全部埋在 `mcp_servers.iwin.env` 里（一堆环境变量名），改起来要记 `OPENAI_BASE_URL` / `QWEN_MM_DEFAULT_MODEL` 这种键名。

现在提到顶层结构化段：

```yaml
iwin:
  enabled: true
  core_exe: C:/Users/swall/iwin_venv/Scripts/iwin-core.exe
  vision:
    base_url: http://127.0.0.1:8080/v1      # 本地 Strata 3.8 Flash
    model: qwen3.8-flash-next-iq3_xxs
    api_key_env: OPENAI_API_KEY
    ask_before_fallback: false
  fallback:
    enabled: true
    base_url: https://opencode.ai/zen/go/v1  # opencode-go DS4.1-flash
    model: deepseek-v4.1-flash
    api_key_env: OPENCODE_GO_API_KEY
  asr:
    url: http://127.0.0.1:8766
  ffmpeg: ''
```

### 1.2 同步脚本

`sync_iwin_config.py` 把 `iwin.*` 写回 `mcp_servers.iwin.env`：

| 顶层配置 | MCP env 键 |
|---|---|
| `iwin.vision.base_url` | `OPENAI_BASE_URL` |
| `iwin.vision.model` | `QWEN_MM_DEFAULT_MODEL` |
| `iwin.asr.url` | `ASR_SERVER_URLS` |
| `iwin.ffmpeg`（非空时） | `FFMPEG_PATH` |
| `iwin.core_exe` | `mcp_servers.iwin.command` |

**只同步这 5 项**，不碰 `OPENAI_API_KEY`（那个是真 key，脚本不管理）。

### 1.3 用法

```bash
# 检查差异（不写）
python J:/HermesShared/data/scripts/local-mods/sync_iwin_config.py --check

# 实际同步
python J:/HermesShared/data/scripts/local-mods/sync_iwin_config.py

# 重启 Hermes 生效
```

改动流程：**改 `iwin:` 段 → 跑 sync → 重启**。

---

## 2. 验证（已完成）

```
=== add_iwin_block.py ===
iwin 段: ['enabled', 'core_exe', 'vision', 'fallback', 'asr', 'ffmpeg']
vision.base_url = http://127.0.0.1:8080/v1
asr.url = http://127.0.0.1:8766
OK

=== sync_iwin_config.py --check ===
无差异，配置已同步
```

**「无差异」是正确的**：`mcp_servers.iwin.env` 现有 4 个键的值与新的 `iwin:` 段**完全一致**——

```
OPENAI_API_KEY = <redacted>
OPENAI_BASE_URL = http://127.0.0.1:8080/v1
QWEN_MM_DEFAULT_MODEL = qwen3.8-flash-next-iq3_xxs
ASR_SERVER_URLS = http://127.0.0.1:8766
```

`FFMPEG_PATH` 不在 env 里，但 `iwin.ffmpeg` 是空串（脚本跳过空值）→ 无差异，符合预期。

---

## 3. 为什么不是「完整的 Setting 层」

原计划 P3 是「统一 Setting 配置层（读写 + 热加载）」。实际做了更轻的方案，原因：

1. **Hermes 已有 config 系统**（`config.yaml` + `hermes config set` + `load_config()`），再造一层是重复。
2. **热加载对 MCP 无效** ：MCP server 是进程启动时 spawn 的，改 env 必须重启。做热加载只是把「重启」包装成假象。
3. **用户真实需求是「好改」** ，不是「有个 Setting 界面」。结构化配置段 + 同步脚本已经满足。

**如果以后要 Setting UI**：`hermes config set iwin.vision.model xxx` 已经能改，再跑 sync 即可。

---

## 4. 待做（需求 6 剩余部分）

| 项 | 状态 |
|---|---|
| 配置结构化 | ✅ 完成 |
| 同步脚本 | ✅ 完成 |
| **23 工具转原生 Hermes 插件** | ⬜ 未做（当前仍走 MCP，功能完整） |
| ffmpeg 路径生效验证 | ⬜ 需要真有 ffmpeg 调用时才能验 |

**关于「转原生插件」的重新评估**：

用户原定案是「A：23 工具整套转 Hermes 原生插件」。但调研后发现：
- iWIN 的 MCP server（`iwin-core.exe`，`qwen-mm-plugins-core`）已经是标准 stdio MCP，Hermes 原生支持
- **转原生插件 = 把 23 个工具的 schema + HTTP 客户端逻辑重写一遍**，工作量约 1-2 天
- **收益**：少一层 MCP 进程；**成本**：23 个工具的行为要逐个对齐，回归风险高

**建议**：保持 MCP 挂载（现在能用），只在需要时（比如 MCP 成为性能瓶颈）再转。**这个判断需要用户确认**——不在今晚自行决定范围。

---

## 5. 怎么回滚

```bash
# 配置回滚（去掉 iwin: 段 + 恢复 mcp env）
cp J:/HermesShared/data/config.yaml.bak_iwinblock_20261109_095031 J:/HermesShared/data/config.yaml
```

**注意**：`mcp_servers.iwin.env` 本轮**没有被修改**（因为无差异），所以只需回滚 `iwin:` 段的添加。

---

## 6. 相关文件

| 文件 | 位置 |
|---|---|
| 配置段 | `J:/HermesShared/data/config.yaml`（文件末尾的 `iwin:` 段） |
| 添加脚本 | `scripts/local-mods/add_iwin_block.py` |
| 同步脚本 | `scripts/local-mods/sync_iwin_config.py` |
| 备份 | `config.yaml.bak_iwinblock_20261109_095031` |
