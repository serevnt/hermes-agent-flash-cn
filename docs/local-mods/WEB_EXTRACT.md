# 本地改造：web_extract 修复（native provider）

> 时间：2026-11-09
> 症状：`web_extract` 完全不可用，只能靠 `curl -k` 手工兜底
> 插件位置：`J:/HermesShared/data/plugins/web/native/`（**用户插件目录，不在 git 仓库内**）

---

## 1. 病因链（三层，逐层查出来的）

### 第一层：`extract_backend: auto` 是非法值

`config.yaml` 原来是：

```yaml
web:
  backend: ''
  search_backend: searxng
  extract_backend: auto      # ← 错
```

`auto` **不是一个合法 provider 名**，registry 会拿它当名字去查，查不到就报：

```
web is configured to use 'auto' (set via hermes tools),
but no registered web extract provider has that name.
```

**默认值应该是 `""`**（`hermes_cli/config_defaults.py:383`：`"extract_backend": ""`）。

### 第二层：环境里 6 个 extract provider 全部缺 key

实测 `list_providers()` 结果：

| provider | supports_search | supports_extract | is_available |
|---|---|---|---|
| brave-free | ✅ | ❌ | ❌ |
| ddgs | ✅ | ❌ | ❌ |
| **exa** | ✅ | ✅ | ❌ 缺 key |
| **firecrawl** | ✅ | ✅ | ❌ 缺 key |
| **keenable** | ✅ | ✅ | ❌ 缺 key |
| openai-native | ✅ | ❌ | ❌ |
| **parallel** | ✅ | ✅ | ❌ 缺 key |
| **perplexity** | ✅ | ✅ | ❌ 缺 key |
| searxng | ✅ | ❌ | **✅ 唯一可用** |
| **tavily** | ✅ | ✅ | ❌ 缺 key |
| xai | ✅ | ❌ | ❌ |

`.env` 里 `EXA_API_KEY` 和 `FIRECRAWL_API_KEY` **都是注释状态**。

**所以：唯一能用的 searxng 只做 search，不做 extract → 没有任何 provider 能做 extract。**

这就是「必须 curl -k」的真实原因。

### 第三层（顺带发现）：Clash 那条老问题早就解决了

`config.yaml:696`：`security.fake_ip_ranges: [198.18.0.0/16]` —— Clash TUN 的 fake-ip 网段已豁免。
**「Clash 判内网」已经不是问题**，记忆里的旧状态可以更新了。

---

## 2. 解法：写一个 native extract provider

### 2.1 为什么自己写而不是配 key

- 6 个付费 provider 都要注册账号 + 拿 key，还有额度限制
- 抓取本身不需要外部服务：httpx 直接 GET + HTML 转文本就够
- 零 key、零额度、零外部依赖

### 2.2 文件清单

| 文件 | 作用 |
|---|---|
| `plugins/web/native/plugin.yaml` | manifest（**必需**，见第 3 节坑） |
| `plugins/web/native/__init__.py` | `register(ctx)` 注册 provider |
| `plugins/web/native/provider.py` | `NativeExtractProvider` 实现 |

### 2.3 provider 能力

- **name**: `native`
- **is_available()**: httpx 在就 True（Hermes 既有依赖）
- **supports_search()**: False（只做 extract）
- **supports_extract()**: True
- **extract(urls, **kwargs)**: async，返回 `[{url, title, content, raw_content, metadata}]`

**实现要点**：
- httpx.AsyncClient，`follow_redirects=True`
- 并发信号量 4（避免打爆目标站）
- 单页上限 3MB（`max_bytes`）
- 超时 30s（`timeout`）
- **`verify_ssl` 可配置**（对应 curl -k 场景）
- HTML→文本：先去 `script/style/noscript/svg/iframe/template`，块级标签转换行，去标签，解码实体（含 `&#9660;` 数字实体），压空白
- **4xx/5xx 直接返回 error**（不把 404 错误页当内容）
- 非法 scheme（ftp/file 等）报 error

### 2.4 配置

```yaml
web:
  extract_backend: native
  native:
    verify_ssl: true        # false = 跳过证书校验（curl -k 等价）
    timeout: 30
    max_bytes: 3000000
    user_agent: ""          # 空 = 内置 Chrome UA

plugins:
  enabled:
    - hermes-lcm
    - hindsight
    - homeassistant
    - review-board
    - web/native            # 必须显式列出来
```

---

## 3. ⚠️ 关键坑：用户插件目录必须有 `plugin.yaml`

`hermes_cli/plugins_discovery.py:160-190` 的 `scan_directory`：

```python
manifest_file = next((f for f in (child / "plugin.yaml", child / "plugin.yml") if f.exists()), None)
portable_file = child / "plugin.json"
...
if manifest_file is not None:      # 有 manifest → 收
    ...
elif has_portable:                  # 有 plugin.json → 收
    ...
elif depth >= 1:                    # ← 到这里就跳过！
    logger.debug("Skipping %s (no plugin.yaml, depth cap reached)", child)
else:                               # depth == 0 才递归进子目录
    manifests.extend(scan_directory(child, source, prefix=sub_prefix, depth=depth + 1))
```

**用户插件目录**（`$HERMES_HOME/plugins/`）扫描时 `depth=0`：
- `plugins/web/` —— 没有 manifest，depth=0 → **递归进去**，depth 变 1
- `plugins/web/native/` —— depth=1，**没有 `plugin.yaml` 就 `continue` 跳过**

**我第一版只写了 `__init__.py` + `provider.py`，所以插件完全没被发现**（`list_plugins()` 里根本没有它），排查了两轮才定位到。

**`plugin.yaml` 最小内容**：

```yaml
name: web-native
version: 1.0.0
description: "..."
author: local-mod
kind: backend
provides_web_providers:
  - native
```

**注意**：`plugins.enabled` 里写的是**目录路径** `web/native`，不是 manifest 的 `name`（`web-native`）。

---

## 4. 实测结果

### 4.1 插件被发现 + provider 被选中

```
插件: web/native enabled=True err=None
native provider 注册: YES
active extract provider: native
```

### 4.2 抓取实测

```
https://example.com          status=200  len=171    err=None
https://www.python.org       status=200  len=6742   err=None
https://httpbin.org/html     status=200  len=3594   err=None
```

内容质量（python.org 前 200 字）：

```
Welcome to Python.org
Notice: This page displays a fallback because interactive scripts did not run.
Possible causes include disabled JavaScript or failure to load scripts or stylesheets.
Skip to content
▼ Close
```

### 4.3 错误处理实测

```
https://example.com/nonexistent-404-xyz  err=HTTP 404
ftp://bad.scheme/x                       err=unsupported scheme (only http/https)
not-a-url                                err=unsupported scheme (only http/https)
```

### 4.4 数字实体

修复前：`&#9660;` 原样留在文本里
修复后：`还有数字实体: False` ✅

---

## 5. 生效条件

**必须重启 Hermes**（插件在进程启动时 discover）。

重启后 `web_extract` 工具应该直接可用，不再需要 `curl -k`。

---

## 6. 怎么回滚

```bash
# 1) 配置回滚
cp J:/HermesShared/data/config.yaml.bak_webnative_20261109_094614 J:/HermesShared/data/config.yaml

# 2) 或者只禁用插件（保留代码）
#    编辑 config.yaml 把 plugins.enabled 里的 web/native 去掉，extract_backend 改回 ''
```

**插件本身没有依赖，删除目录即可完全移除**：

```bash
rm -rf J:/HermesShared/data/plugins/web/native
```

---

## 7. 影响面

| 项 | 影响 |
|---|---|
| `web_extract` | **从不可用 → 可用**（无 key 依赖） |
| `web_search` | 无影响（仍走 searxng） |
| 其他 provider | 无影响（native 是新增，不覆盖） |
| 资源 | 每个 URL 一个 httpx 连接，并发上限 4，内存峰值约 4×3MB |

---

## 8. 待办

- [ ] 重启后实测 `web_extract` 工具调用（本会话注册表是旧的）
- [ ] 考虑给 provider 加 `robots.txt` 尊重（当前不检查）
- [ ] 考虑加 PDF 处理（当前只处理 HTML）
- [ ] 若以后配了 EXA/Tavily 的 key，可把 `extract_backend` 切回去做 A/B 对比抓取质量
