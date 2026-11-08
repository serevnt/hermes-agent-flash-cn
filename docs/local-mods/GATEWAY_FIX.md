# 本地改造：Gateway 修复

> 时间：2026-11-09
> 对应上游 issue：#150103、#101033

---

## 修复 1：#150103 — `session.create` 内联阻塞 WS 读循环

### 改了什么

**文件**：`J:/HermesShared/data/hermes-agent/tui_gateway/server.py`
**位置**：第 187 行，`_LONG_HANDLERS` 集合

```diff
 _LONG_HANDLERS = frozenset({
+    "session.create",  # 本地改造 20261109（#150103）
     "session.foreign.list", "session.foreign.preview", "session.foreign.import",
```

**备份**：`J:/HermesShared/data/cache/scratch/hermes_baks/server.py.bak_sesscreate_20261109_015725`（109,439 B）

### 为什么

`tui_gateway/rpc_dispatch.py:66-68`：

```python
if normalized[1] not in _LONG_HANDLERS:
    return handle_request(req)      # ← 内联跑在 WS 读循环上
...
future = _pool.submit(lambda: ctx.run(run))   # ← 走 RPC 池
```

`session.create` 不在集合里 → 创建会话时整个 WebSocket 读循环被占住 → 同一 socket 上所有其他 RPC（`session.resume`、`model.options` 等）排队等待 → **桌面端看起来整个冻死**。

上游 issue 实测时间线：请求发出后 **3 分 47 秒**响应才就绪，期间客户端已超时关闭连接四次（12:06:12 / 12:06:44 / 12:07:21 / 12:08:14），最后 `ws response send failed`。

对比：`session.resume`、`session.list`、`session.save` 都已在集合里，唯独漏了 `session.create`。

### issue 官方验证结论（原文）

> A local fix that works: add `"session.create"` to `_LONG_HANDLERS`. It then runs on the RPC pool like `session.resume` already does. With a stubbed create handler that sleeps, `dispatch()` returns `None` at once. The next inline RPC on the same transport answers in under 2 s, and the pool worker still writes the create response on the bound transport. Existing `tests/tui_gateway` tests for the inline/pool split still pass.

### 我的验证

| 验证项 | 结果 |
|---|---|
| 语法检查（ast.parse） | ✅ OK |
| 差异范围 | ✅ 仅 2 行（1 行代码 + 1 行注释），`diff` 确认 |
| 官方点名测试 `test_inline_rpc_gil_starvation` | ❌ **环境阻塞**（见下） |
| 回滚对照实验 | ✅ 回滚后同样失败 → **与本改动无关** |

### ⚠️ 已知环境问题（不是本改动引起）

跑 `tests/tui_gateway/` 测试时报：

```
ImportError: cannot import name 'ArgsKwargs' from 'pydantic_core._pydantic_core'
...\installs\07f131753a387a3f\environments\1f657fa5605f490599826df454a9de97\venv\Lib\site-packages\pydantic\...
```

**根因**：测试 fixture 用 `patch.dict("sys.modules", {"hermes_state": MagicMock(), ...})` mock 掉四个模块后，导入链 `tui_gateway.server` → `contracts/billing_delegation_pets.py:11` → `from pydantic import Field` 时，解析到了 **managed install 环境的旧 pydantic**（源码树 `.venv` 里是 2.13.4，正常）。

**证据**：单独写 probe 测试 `import pydantic` → 从 `.venv` 加载 2.13.4，`sys.path` 无 installs 路径 ✅。回滚 `session.create` 改动后跑同一测试 → **同样报错** ✅。

**结论**：测试环境预先存在依赖冲突，与本改动无关。生产运行不走这个 fixture 路径，不受影响。

**后续待办**：修测试环境（可能需要在 conftest 里强制 sys.path 优先级，或清理 managed install 里的旧 pydantic）。

### 怎么回滚

```bash
cp J:/HermesShared/data/cache/scratch/hermes_baks/server.py.bak_sesscreate_20261109_015725 \
   J:/HermesShared/data/hermes-agent/tui_gateway/server.py
```

### 生效条件

**必须重启 Hermes gateway**：`hermes gateway restart`（或退出桌面端后重开）。

---

## 修复 2：#101033 — `run_in_executor` 无超时冻结 gateway 120 秒

**状态**：⬜ 未开始

### 问题

`gateway/run.py` 里调用平台适配器的 `run_in_executor` 没有超时。一个挂死的适配器会阻塞事件循环 120 秒，期间整个 gateway + cron 全部停摆。

### 上游建议修法

用 `asyncio.wait_for` 包一层，超时 30 秒可配置，超时后 cancel + 记日志 + 打点指标。

### 待做

- [ ] 定位所有裸 `run_in_executor` 调用点（已 grep：`gateway/control_socket.py:422`、`gateway/platforms/api_server_runs.py:89/114`、`gateway/relay/media.py:104/151/222`）
- [ ] 判断哪些是需要超时的（短 IO 不需要，平台适配器调用需要）
- [ ] 加 `asyncio.wait_for` + 超时常量 + 日志
- [ ] 写测试

---

## 影响面

| 项 | 影响 |
|---|---|
| 桌面端新建会话 | 不再阻塞其他 RPC，只延迟自己 |
| 其他平台（Slack/TUI/API） | 无影响（同一 dispatch 逻辑，但 `session.create` 走池对它们同样是改进） |
| 并发 | `_rpc_pool_workers` 默认 8，创建会话占用一个 worker |
| 风险 | 极低（与 `session.resume` 完全同类，后者已在集合里运行多时） |
