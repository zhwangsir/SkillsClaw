---
name: qclaw-openclaw
description: |
  [MANDATORY] 本环境没有全局 openclaw CLI。任何需要执行 openclaw 命令的场景（配置管理、
  定时任务、模型设置、Skills/Plugins 管理等）都必须通过本 skill 提供的脚本执行。
  禁止直接调用 `openclaw` 命令，禁止执行服务管理命令（gateway/daemon），
  OpenClaw 服务由 QClaw Electron 守护进程管理。

  [MANDATORY] 创建 cron 定时任务规则（无论何时均强制执行）：
  1. 必须读取本 skill 的 SKILL.md 了解完整参数格式，禁止凭记忆猜测
  2. sessionTarget 优先 "isolated"，payload.kind 优先 "agentTurn"
  3. payload.message 格式：「直接输出以下提醒内容，禁止调用message工具：{具体提醒内容}」，禁止写"发送消息给用户/调用企微"等动词
  4. 渠道判断：sender=openclaw-control-ui 且未提外部渠道 → cron内置工具；sender=openclaw-control-ui 且提到企微/飞书/钉钉 → CLI脚本+读channel-defaults.json；有Conversation info → CLI脚本
  Keywords: 提醒, 闹钟, 定时, 每天, 叫我, 周期, 重复, 定期, 打卡, 日报, 周报, 签到, cron, remind, schedule, timer, 天气, 喝水, 取消提醒, 定时任务.
metadata: {"openclaw": {"emoji": "⚙️"}}
---

# QClaw OpenClaw CLI

## MANDATORY — 必读

**本环境没有全局安装 `openclaw` CLI。** 直接执行 `openclaw` 命令会失败。

所有需要调用 openclaw CLI 的操作（包括但不限于 `config`、`skills`、`plugins`、`models`、`status` 等），
**必须且只能**通过本 skill 提供的 wrapper 脚本执行。不要尝试：

- ❌ `openclaw config get ...`（全局命令不存在）
- ❌ `npx openclaw ...`（环境变量不正确）
- ❌ 直接调用 `node openclaw.mjs ...`（缺少必要的环境变量和路径）

正确做法是使用本 skill 的脚本，它会自动设置所有必要的环境变量和路径。

OpenClaw 服务由 QClaw Electron 守护进程管理（自动拉起、熔断保护），禁止通过 CLI 直接启停服务。

## 执行方式

本 skill 提供跨平台脚本，位于 skill 目录的 `scripts/` 下。脚本会自动从 `~/.qclaw/qclaw.json` 读取运行时路径和环境变量。

### macOS

```bash
bash <skill_dir>/scripts/openclaw-mac.sh <command> [args...]
```

### Windows

```cmd
<skill_dir>\scripts\openclaw-win.cmd <command> [args...]
```

> `<skill_dir>` 是本 SKILL.md 所在的目录路径。

## 允许的命令

### config — 配置管理

```bash
# 读取配置值
bash <skill_dir>/scripts/openclaw-mac.sh config get <dot.path>

# 设置配置值
bash <skill_dir>/scripts/openclaw-mac.sh config set <dot.path> <value>

# 删除配置值
bash <skill_dir>/scripts/openclaw-mac.sh config unset <dot.path>
```

示例:
```bash
# 查看当前网关端口
bash <skill_dir>/scripts/openclaw-mac.sh config get gateway.port

# 设置默认模型
bash <skill_dir>/scripts/openclaw-mac.sh config set agents.defaults.model.primary "claude-sonnet-4-20250514"
```

> 配置生效规则：OpenClaw 的配置修改统一支持热加载，修改后自动在进程内生效。
> 严禁执行任何服务重启行为（包括但不限于 `gateway/daemon restart`、`kill PID`、`taskkill`）。

#### [MANDATORY] 修改前官方 Schema 校验规则

对所有会修改 `openclaw.json` 的操作，必须先完成官方文档校验：

1. **先查官方文档**：修改前必须先查询 OpenClaw 官方文档中对应配置项（字段路径、类型、可选值、约束、默认值）。
2. **无法确认即拒绝**：若未找到对应官方文档，或无法确认该字段符合官方 schema，必须拒绝本次修改，不得猜测字段或试错写入。
3. **只允许写入官方定义字段**：禁止新增或写入官方 schema 未定义字段，禁止写入类型不匹配值。
4. **先确认再落盘**：只有在确认变更符合官方 schema 后，才允许执行 `config set/unset` 或其他写入操作。

> 拒绝话术要求：当缺少官方 schema 依据时，明确回复"缺少 OpenClaw 官方 schema 依据，拒绝修改 `openclaw.json`"。

#### [MANDATORY] 配置修改失败自动回滚规则

对所有修改 openclaw.json 配置文件的操作，必须遵循以下事务化规则：

1. **先备份后修改**：执行修改前，先保存 `openclaw.json` 的修改前快照（仅用于本次变更回滚）。
2. **失败即回滚**：如果上下文日志/执行结果显示本次修改失败（无论失败原因），必须立即回滚到本次修改前快照。
3. **只回滚本次变更**：回滚目标是撤销当前这一次修改，不得覆盖更早的历史有效配置。
4. **禁止跳过回滚**：检测到失败后，不允许继续后续配置写入，必须先完成回滚再继续。
5. **回滚后复核**：回滚完成后，必须重新读取目标配置项，确认已恢复到修改前状态。

### cron — 定时任务

> 🚨 **[MANDATORY]** 用户提到「提醒/闹钟/定时/每天X点/X分钟后/叫我/周期/重复/打卡/日报/周报/签到/天气/喝水」等时，**必须创建 cron 任务**。口头承诺无效，只有注册 cron 才能真正定时执行。

#### 渠道判断（MANDATORY）

| 渠道类型 | 判断依据 | 创建方式 |
|---------|---------|---------|
| **内部本地**（本地提醒） | sender=`openclaw-control-ui`，且用户**未提及**企微/飞书/钉钉等外部渠道名 | `cron` 内置工具，delivery 不传 channel/to |
| **客户端→外部渠道**（从客户端创建，推送到外部） | sender=`openclaw-control-ui`，且用户**明确提到**企微/飞书/钉钉等渠道名 | CLI 脚本 + 读 `channel-defaults.json` 获取 to |
| **外部渠道直发**（从渠道内创建） | 有 Conversation info，sender 为用户 ID | CLI 脚本，channel/to 从 Conversation info 提取 |

> 🚨 **[MANDATORY]** 用户在客户端说"在企业微信/飞书/钉钉提醒我"时，属于第二行"客户端→外部渠道"场景，**必须读 `~/.qclaw/channel-defaults.json`** 获取 `to` 值，不可走内部渠道逻辑。

> 🚨 **[MANDATORY] CLI 失败处理规则**：CLI 脚本报错时，**最多重试一次**，仍失败则直接告知用户"定时任务创建失败，请稍后再试"，**禁止反复调试 gateway 连接、禁止尝试其他方式绕过**。

#### 内部渠道：`cron` 内置工具

```json
{
  "action": "add",
  "job": {
    "name": "<任务名>",
    "schedule": {"kind":"every","everyMs":1800000},
    "sessionTarget": "isolated",
    "payload": {"kind":"agentTurn","message":"直接输出以下提醒内容，禁止调用message工具：<提醒内容>"},
    "delivery": {"mode":"announce"}
  }
}
```

schedule 三选一：`{"kind":"every","everyMs":N}` / `{"kind":"at","at":"20m"}` / `{"kind":"cron","expr":"0 8 * * *","tz":"Asia/Shanghai"}`

> 🚨 **[MANDATORY] 一次性 vs 周期性**：用户说具体时刻（X点/X分钟后）→ `{"kind":"at",...}` + `"deleteAfterRun":true`；有周期词（每天/每小时）→ `every` 或 `cron`；**无明确周期词默认一次性**。

> 绝对时间必须带 `+08:00`，如 `2026-03-26T15:00:00+08:00`，裸 ISO 当 UTC 差 8 小时。一次性任务加 `"deleteAfterRun":true`。

delivery：本地不传 channel/to；外部渠道填 `"channel":"<渠道>","to":"<sender_id>"`。`bestEffort` 由插件自动注入，不需手填。

#### 外部渠道：CLI 脚本

> 🚨 **[MANDATORY] 一次性 vs 周期性判断**：
> - 用户说"X点/X分钟后/明天/今天X时"等**具体时刻** → **一次性任务**，用 `--at`，必须加 `--delete-after-run`
> - 用户说"每天/每小时/每X分钟/每周"等**周期词** → **周期任务**，用 `--every` 或 `--cron`
> - 没有明确周期词时，**默认按一次性处理**

```bash
bash <skill_dir>/scripts/openclaw-mac.sh cron add \
  --name "<任务名>" \
  --every 30m \          # 或 --at 20m / --cron "0 8 * * *" --tz "Asia/Shanghai"
  --session isolated \
  --agent <agentId> \    # 见下方说明，主 agent 不传此参数
  --message "直接输出以下提醒内容，禁止调用message工具：<提醒内容>" \
  --announce --channel <渠道> --to <sender_id>
  # 一次性任务加 --delete-after-run
```

> 🚨 **[MANDATORY] `--agent` 参数规则**：
> - **主 agent**（cwd 为 `~/.qclaw/workspace`，sessionKey 为 `agent:main:...`）→ **不传 `--agent`**，默认归属 main
> - **非主 agent**（cwd 包含 `workspace-agent-xxx`，sessionKey 为 `agent:<agentId>:...`）→ **必须传 `--agent <agentId>`**
> - agentId 从**当前对话的** sessionKey 第二段提取，禁止使用历史对话中见到的 agentId

> 🚨 **[MANDATORY] payload/message 写法规则**：
> - **禁止**在 message 中使用"发送消息给用户"、"调用飞书/企微"等动词指令 → 会触发 AI 调用 `message` 工具导致报错
> - **正确写法**：直接描述提醒内容，并在末尾追加：`禁止调用message。`
> - delivery 的 `announce` 机制会自动把 AI 输出投递到目标渠道，AI 只需输出文字即可

#### 客户端创建外部渠道 cron

sender=`openclaw-control-ui` 且需推送到外部渠道时，**没有 `to` 值**，必须：
1. 读取 `~/.qclaw/channel-defaults.json`，找到对应渠道的 `to` 值
2. 找不到 → 告知"请先在 {渠道} 发一条消息，系统会自动记录你的 ID"，**不要创建缺 `to` 的任务**

> `channel-defaults.json` 由 `cron-delivery-guard` 插件在收到外部渠道消息时自动维护，无需手动编辑。

#### 管理命令

> 🚨 **[MANDATORY] 暂停/停止/禁用 ≠ 删除**：用户说"暂停/停止/先不提醒/禁用"时，使用 **disable/update** 操作，**不要 remove**；用户明确说"删除"时才用 remove。

内部渠道用 `cron` 工具：

| 操作 | 内置工具 |
|------|---------|
| 列表 | `{"action":"list"}` |
| 暂停/禁用 | `{"action":"update","jobId":"<id>","patch":{"enabled":false}}` |
| 恢复/启用 | `{"action":"update","jobId":"<id>","patch":{"enabled":true}}` |
| 删除 | `{"action":"remove","jobId":"<id>"}` |
| 立即执行 | `{"action":"run","jobId":"<id>"}` |

外部渠道用 CLI：`cron list` / `cron edit <id> --enabled false` / `cron edit <id> --enabled true` / `cron rm <id>` / `cron run <id>`

#### 回复模板

一次性：`⏰ 好的，{时间}后提醒你{内容}~` | 周期：`⏰ 收到，{周期}提醒你{内容}~` | 取消：`✅ 已取消"{名称}"`

> 🚨 外部渠道**只输出确认话术**，严禁输出推理过程或工具调用说明。


### models — 模型配置

```bash
# 查看已配置模型
bash <skill_dir>/scripts/openclaw-mac.sh models list

# 查看模型状态
bash <skill_dir>/scripts/openclaw-mac.sh models status

# 设置默认模型
bash <skill_dir>/scripts/openclaw-mac.sh models set <model_id>

# 设置图像模型
bash <skill_dir>/scripts/openclaw-mac.sh models set-image <model_id>

# 管理模型别名
bash <skill_dir>/scripts/openclaw-mac.sh models aliases --help

# 管理 fallback 列表
bash <skill_dir>/scripts/openclaw-mac.sh models fallbacks --help
```

### skills — Skills 管理

```bash
# 列出所有 skills
bash <skill_dir>/scripts/openclaw-mac.sh skills list

# 查看 skill 详情
bash <skill_dir>/scripts/openclaw-mac.sh skills info <skill_name>

# 检查 skills 就绪状态
bash <skill_dir>/scripts/openclaw-mac.sh skills check
```

### plugins — 插件管理

```bash
# 列出所有插件
bash <skill_dir>/scripts/openclaw-mac.sh plugins list

# 查看插件详情
bash <skill_dir>/scripts/openclaw-mac.sh plugins info <plugin_id>

# 启用/禁用插件
bash <skill_dir>/scripts/openclaw-mac.sh plugins enable <plugin_id>
bash <skill_dir>/scripts/openclaw-mac.sh plugins disable <plugin_id>

# 安装/卸载插件
bash <skill_dir>/scripts/openclaw-mac.sh plugins install <path_or_spec>
bash <skill_dir>/scripts/openclaw-mac.sh plugins uninstall <plugin_id>

# 诊断插件问题
bash <skill_dir>/scripts/openclaw-mac.sh plugins doctor
```

### agents — Agent 工作区管理

```bash
# 列出 agents
bash <skill_dir>/scripts/openclaw-mac.sh agents list

# 添加新 agent
bash <skill_dir>/scripts/openclaw-mac.sh agents add <name>

# 删除 agent
bash <skill_dir>/scripts/openclaw-mac.sh agents delete <name>

# 设置 agent 身份
bash <skill_dir>/scripts/openclaw-mac.sh agents set-identity <name> --emoji "🤖"
```

### channels — 通道管理

```bash
# 列出通道
bash <skill_dir>/scripts/openclaw-mac.sh channels list

# 查看通道状态
bash <skill_dir>/scripts/openclaw-mac.sh channels status

# 查看通道能力
bash <skill_dir>/scripts/openclaw-mac.sh channels capabilities

# 查看通道日志
bash <skill_dir>/scripts/openclaw-mac.sh channels logs
```

### gateway — 网关状态查询（只读）

```bash
# 查看网关详细状态（只读，不操作服务）
bash <skill_dir>/scripts/openclaw-mac.sh gateway status
```

### 其他允许的命令

```bash
# 系统状态
bash <skill_dir>/scripts/openclaw-mac.sh status

# 网关健康检查
bash <skill_dir>/scripts/openclaw-mac.sh health

# 诊断
bash <skill_dir>/scripts/openclaw-mac.sh doctor

# 安全审计
bash <skill_dir>/scripts/openclaw-mac.sh security audit

# 记忆搜索
bash <skill_dir>/scripts/openclaw-mac.sh memory search <query>
bash <skill_dir>/scripts/openclaw-mac.sh memory status

# 会话列表
bash <skill_dir>/scripts/openclaw-mac.sh sessions list

# 日志查看
bash <skill_dir>/scripts/openclaw-mac.sh logs --follow

# 执行审批管理
bash <skill_dir>/scripts/openclaw-mac.sh approvals get
bash <skill_dir>/scripts/openclaw-mac.sh approvals allowlist --help

# 更新检查（仅查看状态，不执行更新）
bash <skill_dir>/scripts/openclaw-mac.sh update status
```

## 禁止的命令

以下命令**绝对禁止执行**，OpenClaw 服务生命周期由 QClaw Electron 守护进程统一管理：

| 命令 | 原因 |
|------|------|
| `gateway run/start/stop/restart` | 服务由 Electron ProcessSupervisor 管理 |
| `gateway install/uninstall` | 系统服务安装由 Electron 控制 |
| `daemon start/stop/restart` | 同上，daemon 是 gateway 的别名 |
| `daemon install/uninstall` | 同上 |
| `node start/stop` | Node host 服务管理 |
| `reset` | 破坏性操作，会清除所有本地配置和状态 |
| `uninstall` | 破坏性操作，会卸载服务和数据 |

> **注意**: `gateway status` 是**允许的**，它只是查询状态，不操作服务。

## 配置热加载与进程内生效

OpenClaw 的配置修改统一采用热加载机制：

- 通过本 skill 执行 `config set/unset` 后，无需重启服务
- 配置会自动在当前进程内重载并生效
- 禁止任何形式的服务重启操作（包括 `gateway/daemon restart`、`kill`、`taskkill`）

如配置未即时体现，请先用只读命令核对当前状态：

```bash
bash <skill_dir>/scripts/openclaw-mac.sh config get <dot.path>
bash <skill_dir>/scripts/openclaw-mac.sh status
bash <skill_dir>/scripts/openclaw-mac.sh health
```

## 故障排查

### `~/.qclaw/qclaw.json` 不存在

QClaw 桌面应用未启动或未成功启动 OpenClaw 服务。请先启动 QClaw 应用。

### PID 无效（进程不存在）

配置热加载过程中状态可能短暂更新。等待几秒后重新查询 `status/health` 即可。

### 命令执行报 Gateway 连接失败

Gateway 服务可能未就绪。先检查健康状态：

```bash
bash <skill_dir>/scripts/openclaw-mac.sh health
```

如果持续失败，执行 `doctor` 并收集日志，不要进行任何重启操作。

### 脚本报错找不到 Node 二进制或 openclaw.mjs

`qclaw.json` 中的路径可能已过期（应用升级后路径变化）。请执行 `status/doctor` 重新校验元信息并收集日志反馈。
