# SETUP_TOKEN.md — 腾讯会议 Token 初始化文档

> **适用文件：** 本文件与 `get-token.sh` / `get-token.ps1` 位于同一目录（`output/`）。

---

## 1. 脚本位置

Token 获取脚本与本文件位于同一目录：

```
output/
├── SETUP_TOKEN.md         ← 本文件
├── get-token.sh           ← Bash 版 Token 获取脚本
└── get-token.ps1          ← PowerShell 版 Token 获取脚本
```

在下文所有示例中，请将 `<SCRIPT_PATH>` **替换为本文件所在目录的绝对路径**。

**示例：**
- macOS/Linux：`<SCRIPT_PATH>` → `/Users/yourname/skills/tencent-meeting-mcp/output`
- Windows：`<SCRIPT_PATH>` → `C:\Users\yourname\skills\tencent-meeting-mcp\output`

---

## 2. 核心规则：每次命令内联获取 Token

**Token 必须在每条命令中独立获取，不可通过 `export` 持久化。**

原因：AI 代理不保留跨命令的环境变量状态，`export` 设置的变量在下次 API 调用时已失效。

---

## 3. Bash 使用方式

### ✅ 正确用法（内联获取）

每次调用 MCP 工具时，通过 `$()` 子 shell 内联获取 Token：

```bash
TENCENT_MEETING_TOKEN=$(bash '<SCRIPT_PATH>/get-token.sh') \
  mcporter call tencent-meeting-mcp schedule_meeting \
  --args '{"subject": "周会", "start_time": "1773280800", "end_time": "1773284400"}'
```

```bash
TENCENT_MEETING_TOKEN=$(bash '<SCRIPT_PATH>/get-token.sh') \
  mcporter call tencent-meeting-mcp get_meeting \
  --args '{"meeting_id": "xxx"}'
```

### ❌ 错误用法（export 无效）

```bash
# ❌ 这种方式无效 — export 的变量在 AI 环境中无法跨命令持久
export TENCENT_MEETING_TOKEN=$(bash '<SCRIPT_PATH>/get-token.sh')
mcporter call tencent-meeting-mcp get_meeting --args '{"meeting_id": "xxx"}'
```

**为什么 export 无效：** AI 代理（如 Claude）每次执行 Bash 命令都是独立子 shell，`export` 设置的环境变量不会传递到下一次命令执行，Token 随子 shell 退出而消失。

---

## 4. PowerShell 使用方式

> ⚠️ **中文编码兼容性**：Windows PowerShell 默认编码为 GBK（CP936），传递中文参数给 mcporter 时会产生乱码。**每次执行包含中文参数的 mcporter 命令前**，必须在同一脚本块内先设置 UTF-8 编码。

```powershell
# ⚠️ 必须先设置 UTF-8 编码，否则中文参数会乱码
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
$token = & "<SCRIPT_PATH>\get-token.ps1"
$env:TENCENT_MEETING_TOKEN = $token
mcporter call tencent-meeting-mcp schedule_meeting --args '{"subject": "周会", "start_time": "1773280800", "end_time": "1773284400"}'
```

> **注意：** 在 PowerShell 中，同一脚本块内的 `$env:` 赋值对后续命令有效，但跨 PowerShell 会话不持久。建议在同一脚本块内完成获取 + 调用。UTF-8 编码设置同样仅在当前脚本块内有效，每次新的脚本块都需要重新设置。

---

## 5. Token 获取失败处理

如果运行 `get-token.sh` 时出现以下错误：

```
ERROR: 获取 Token 失败，HTTP 状态码: ...
ERROR: 凭证服务返回错误 (ret=...)
ERROR: Token 字段为空
```

**解决步骤：**

1. 打开应用内**集成面板**
2. 找到**腾讯会议**集成项
3. 完成**授权绑定**（登录腾讯会议账号并授权）
4. 授权完成后，重新运行命令

> 本地凭证代理服务（`localhost:19000`）须处于运行状态，Token 获取才能成功。

---

## 6. 环境变量参考

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `CREDENTIAL_PLATFORM` | `tencent_meeting` | 凭证平台标识，通常无需修改 |
| `AUTH_GATEWAY_PORT` | `19000` | 本地代理端口，通常无需修改 |
| `BUILD_ENV` | `production` | 环境选择：`production`（默认）或 `test` |
