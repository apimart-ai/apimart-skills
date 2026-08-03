# APIMart MCP + Skill `dev` 版接入教程

> 适用对象：第一次接触 MCP、Skill 或 AI 编程助手的测试人员与开发者。
>
> 渠道：`dev`（测试版）。正式用户应安装 `main`，不要把本教程中的 `dev` 安装地址当成生产版地址。
>
> MCP 地址：`https://mcp.apimart.asia/mcp`
>
> 更新时间：2026-08-03

**重要说明：**本文中的 `dev` 只指 APIMart Skill 的 GitHub `dev` 分支，不代表免费沙箱，也不代表客户端应该自行猜测一个 `mcp-dev` 域名。当前 `dev` Skill 仍连接上面给出的 MCP 地址，调用图片或视频生成工具会产生真实任务并可能计费。若运维以后提供独立测试地址，应统一替换本文中的 MCP URL。

## 目录

1. [先理解 MCP 和 Skill](#1-先理解-mcp-和-skill)
2. [接入前准备](#2-接入前准备)
3. [Codex 桌面端接入（macOS，推荐）](#3-codex-桌面端接入macos推荐)
4. [Codex CLI 接入（macOS/Linux）](#4-codex-cli-接入macoslinux)
5. [Codex 接入（Windows）](#5-codex-接入windows)
6. [Cursor 接入](#6-cursor-接入)
7. [Claude Code 接入](#7-claude-code-接入)
8. [其他 MCP 客户端接入原则](#8-其他-mcp-客户端接入原则)
9. [用 Apipost 独立检查 MCP](#9-用-apipost-独立检查-mcp)
10. [推荐验收流程](#10-推荐验收流程)
11. [常见问题](#11-常见问题)
12. [从 dev 切回正式 main](#12-从-dev-切回正式-main)
13. [安全检查清单](#13-安全检查清单)

## 1. 先理解 MCP 和 Skill

接入 APIMart 时需要配置两部分，它们解决的问题不同：

| 部分 | 大白话解释 | 负责什么 |
| --- | --- | --- |
| MCP | AI 助手连接 APIMart 的“工具接口” | 列出模型、读取文档、上传参考图片、提交图片/视频任务、查询任务结果 |
| Skill | 告诉 AI 助手“这些工具应该按什么顺序使用” | 先读模型文档、正确拼参数、避免重复计费、轮询任务 |

只装 Skill、没有配置 MCP 时，AI 助手看得到操作说明，但没有在线工具可调用。

只配置 MCP、没有安装 Skill 时，AI 助手能看到工具，但不一定每次都按 APIMart 推荐流程使用。

当前接入方式使用用户自己的 APIMart API Key 作为 Bearer Token，不需要另外执行 MCP OAuth 登录。MCP 模式下也不需要在客户端设置 `APIMART_BASE_URL`；该变量只用于 Skill 自带的本地 API 备用脚本，绝不能把它设置成 MCP URL。

正常的生成流程是：

```text
用户提出需求
  → Skill 判断是图片还是视频
  → get_model_docs 读取该模型的实时 Markdown 文档
  → 必要时 get_model_schema 确认图片/视频操作类型
  → 有本地参考图片时，upload_image 上传一次并取得 URL
  → generate_image 或 generate_video 提交一次任务
  → 返回 task_id
  → get_task 按建议间隔查询
  → 返回最终图片或视频地址
```

APIMart MCP 当前应提供 7 个工具：

1. `list_models`
2. `get_model_docs`
3. `get_model_schema`
4. `upload_image`
5. `generate_image`
6. `generate_video`
7. `get_task`

其中 `get_model_docs` 返回的模型 Markdown 文档，才是该模型真实参数、可选值和组合规则的主要依据。`get_model_schema` 是兼容性契约，不能把其中的通用字段全部当成某个模型都支持的参数。

## 2. 接入前准备

### 2.1 必须准备

- 一个可用的 APIMart API Key。
- 可以访问 `https://mcp.apimart.asia/mcp` 的网络。
- Node.js 22.20 或更高版本（第三方 `skills@1.5.21` 安装器的要求；Skill 自带脚本本身支持 Node.js 20+）。
- 已安装并能打开准备接入的客户端，例如 Codex 桌面端或 Codex CLI；Windows 即使暂时没有 `codex` 命令，也可以按第 5.3 节手工配置。
- 一个支持远程 Streamable HTTP MCP 的 AI 客户端。

检查 Node.js 和 npm：

```bash
node -v
npm -v
```

如果 `node -v` 低于 `v22.20.0`，先升级 Node.js。若提示 `command not found`，先从 [Node.js 官网](https://nodejs.org/) 安装满足版本要求的 LTS 版本。

### 2.2 API Key 安全规则

- 不要把 API Key 发到聊天、工单、截图或群聊中。
- 不要把 API Key 写进 Git 仓库、Markdown、JSON 示例或 URL。
- `--bearer-token-env-var` 后面填写的是环境变量名 `APIMART_API_KEY`，不是以 `sk-` 开头的真实密钥。
- PowerShell 的 `Read-Host "请输入……"` 中，引号里的内容只是屏幕提示，不能替换成真实 Key；只有出现提示后，才在隐藏输入位置粘贴 Key。
- 如果密钥曾出现在聊天或截图中，应立即作废并重新生成。
- 测试建议使用测试 Key，并设置合理额度。
- **先安装并检查 Skill，再设置 API Key。**`npx` 安装器及其子进程可以读取当前进程的环境变量；不要在带着 Key 的终端里运行不可信安装命令。
- `launchctl`、普通环境变量和配置文件都不是专业的密钥保险箱。它们只是避免把 Key 直接写进命令或 Git；生产环境应优先使用系统凭据库或企业密钥管理方案。

本教程固定使用已验证的 `skills@1.5.21`，避免每次执行时下载不同版本的安装器。GitHub 的 `dev` 分支仍是会变化的测试来源；需要严格复现某次测试时，把安装 URL 中的 `dev` 换成团队已经审核过的完整 Git commit SHA。

以下写法是错误的：

```bash
# 错误：把真实 Key 当成了环境变量名
codex mcp add apimart-dev \
  --url https://mcp.apimart.asia/mcp \
  --bearer-token-env-var sk-xxxx
```

正确写法永远是：

```bash
--bearer-token-env-var APIMART_API_KEY
```

### 2.3 同一台电脑上的全局 Skill 版本

`skills@1.5.21` 的全局安装通常把 `~/.agents/skills/` 作为共享副本，再让不同客户端直接发现或链接到它。因此同一台电脑上，Codex、Cursor 和 Claude Code **不能被当成三份完全独立的全局版本**：给其中一个客户端全局安装 `dev`，可能改变另一个客户端实际读取的内容。

建议一台测试电脑在同一时间只使用一个全局版本。若要从 `main` 切到 `dev`：

1. 备份自己手工改过的已安装 Skill，并退出所有 AI 客户端。
2. 在 macOS/Linux 终端执行 `unset APIMART_API_KEY`。Windows PowerShell 同时清除当前进程和用户级变量：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```
3. 移除该 Skill 对所有客户端的全局链接与共享副本。Windows 如果 `npx` 只在 CMD 可用，完成上面的 PowerShell 清理后先关闭 PowerShell，再打开 CMD 执行下面这一整行；macOS/Linux 或能使用 `npx` 的 PowerShell 也执行同一行：

```text
npx --yes skills@1.5.21 remove apimart-generate-media -g --agent "*" -y
```

4. 再按后续章节，为你实际使用的每个客户端安装同一个 `dev` 来源。

若确实要在同一台电脑上同时测试 `main` 和 `dev`，不要使用共享的 `-g` 全局安装；应使用不同系统账号、容器，或分别放在隔离项目里的客户端原生 Skill 目录。

## 3. Codex 桌面端接入（macOS，推荐）

Codex 桌面端、Codex CLI 和 Codex IDE 扩展在同一台 Codex 主机上共享 MCP 配置。通常配置一次即可。

### 3.1 安装 `dev` 版 Skill

这里使用第三方 `skills` 安装器把 GitHub `dev` 分支中的完整 Skill 安装到 Codex 可发现的位置。它不是 `codex` 自带子命令。

如果这台电脑以前安装的是正式 `main` 版本，先按第 2.3 节完成全局切换。普通的 `skills update` 会继续跟随原安装来源，不负责把 `main` 切换成 `dev`。全新安装可直接继续。

在终端执行：

```bash
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/dev \
  -g \
  --agent codex \
  --skill apimart-generate-media \
  -y
```

首次运行时，`npx` 会下载固定的 `skills@1.5.21`。只应在确认 npm 包名、版本和 GitHub 仓库地址无误后执行；此时尚未设置 API Key。

检查安装结果：

```bash
npx --yes skills@1.5.21 list -g --json
```

结果中应该能找到：

```text
apimart-generate-media
```

如果电脑没有 `rg` 命令，可以用系统自带的 `grep` 检查新版内容：

```bash
grep -nF "get_model_docs" ~/.agents/skills/apimart-generate-media/SKILL.md
```

若 `npx --yes skills@1.5.21 list -g --json` 显示了不同的安装路径，请把上面命令中的路径替换成实际路径。

### 3.2 设置 API Key

为了避免把 Key 留在 shell 历史中，先使用隐藏输入：

```zsh
unset APIMART_API_KEY
read -s "APIMART_API_KEY?请输入 APIMart 测试 API Key: "
echo
launchctl setenv APIMART_API_KEY "$APIMART_API_KEY"
unset APIMART_API_KEY
```

说明：

- 输入 Key 时终端不会显示字符，这是正常现象。
- `launchctl setenv` 让之后重新启动的 macOS 图形应用能够读取该变量。
- `launchctl` 变量对同一 macOS 用户会话中之后启动的进程可见，并不是加密存储；这里只应放低额度测试 Key。
- 设置完成后必须彻底退出并重新打开 Codex。
- 不要运行 `launchctl getenv APIMART_API_KEY` 截图给别人，因为它会输出真实密钥。

需要清除时执行：

```bash
launchctl unsetenv APIMART_API_KEY
```

### 3.3 添加 APIMart MCP

如果终端可以直接使用 `codex`：

```bash
codex mcp add apimart-dev \
  --url https://mcp.apimart.asia/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

如果提示 `zsh: command not found: codex`，使用 Codex 桌面端自带的 CLI：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp add apimart-dev \
  --url https://mcp.apimart.asia/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

验证配置：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp list
```

预期能看到类似内容：

```text
Name         Url                           Bearer Token Env Var  Status
apimart-dev  https://mcp.apimart.asia/mcp  APIMART_API_KEY       enabled
```

这里显示的是环境变量名，不是实际 Key。

### 3.4 重启并做只读测试

1. 完全退出 Codex 桌面端。
2. 重新打开 Codex。
3. 新建一个任务。
4. 发送下面的提示词：

```text
使用 APIMart Image & Video Skill，只调用 APIMart MCP 的只读工具：
先确认可用工具数量，再查询 Omni-Flash-Ext 的实时模型文档。
告诉我 duration 支持哪些值，不要生成内容。
```

预期结果：

- MCP 有 7 个工具。
- 能调用 `get_model_docs`。
- `Omni-Flash-Ext` 的 `duration` 显示为 `4、6、8、10`。
- 没有调用 `generate_image` 或 `generate_video`。

### 3.5 手工编辑 `config.toml`（可选）

如果不想使用 `codex mcp add`，可以编辑：

```text
~/.codex/config.toml
```

加入：

```toml
[mcp_servers.apimart-dev]
url = "https://mcp.apimart.asia/mcp"
bearer_token_env_var = "APIMART_API_KEY"
```

保存后重启 Codex。不要把真实 API Key 直接写到 `config.toml`。

## 4. Codex CLI 接入（macOS/Linux）

如果你主要在终端使用 Codex，先在**尚未设置 API Key** 的终端里安装并检查 Skill：

```bash
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/dev \
  -g \
  --agent codex \
  --skill apimart-generate-media \
  -y
npx --yes skills@1.5.21 list -g --json
```

确认来源和安装结果正确后，再让 Key 只存在于当前终端会话。先看你的 shell：

```bash
echo "$SHELL"
```

如果结果以 `zsh` 结尾，使用：

```zsh
unset APIMART_API_KEY
read -s "APIMART_API_KEY?请输入 APIMart 测试 API Key: "
echo
export APIMART_API_KEY
```

如果结果以 `bash` 结尾，使用：

```bash
unset APIMART_API_KEY
read -rsp "请输入 APIMart 测试 API Key: " APIMART_API_KEY
echo
export APIMART_API_KEY
```

不要在 zsh 中使用 `read -s -p ...`；zsh 会把 `-p` 理解为从协进程读取，并可能报 `no coprocess`。先执行 `unset` 可以防止读取失败时误用上一次残留的 Key。

添加 MCP：

```bash
codex mcp add apimart-dev \
  --url https://mcp.apimart.asia/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

检查连接：

```bash
codex mcp list
```

启动 Codex 后，可在终端 UI 中输入：

```text
/mcp
```

查看 MCP 是否已连接。

完成测试后执行 `unset APIMART_API_KEY`；关闭当前终端也会清除这个会话的变量。下一次使用时需要重新设置，或者通过操作系统的安全凭据管理方案注入。

## 5. Codex 接入（Windows）

下面是面向小白的固定流程。Windows 上按这个顺序操作，不要跳步，也不要把 CMD 与 PowerShell 命令混在同一个窗口。

```text
CMD 安装 Skill
  → PowerShell 隐藏输入并保存 Key
  → 配置远程 HTTP MCP
  → 完全退出并重新启动 Codex
  → 新建任务做只读测试
```

先看提示符判断当前终端：

| 终端 | 提示符示例 | 本节用途 |
| --- | --- | --- |
| CMD（命令提示符） | `C:\Users\alice>` | 运行 `npx` 单行安装命令 |
| PowerShell | `PS C:\Users\alice>` | 隐藏输入 Key、编辑和检查配置 |

看到 `C:\...>` 时不要执行以 `$` 开头的 PowerShell 命令。看到 `PS C:\...>` 时，如果 `npm` 或 `npx` 不可用但 CMD 可用，直接回到 CMD 安装 Skill；也可以尝试 `npx.cmd`。本节所有 CMD 命令均为单行，不要自行加入反斜杠、反引号或换行符。

**整节最重要的规则：**`APIMART_API_KEY` 是固定的环境变量名称。除 PowerShell 显示“请输入新的 APIMart 测试 API Key:”后的隐藏输入外，不要在任何命令、提示文字、TOML、聊天或截图中粘贴以 `sk-` 开头的真实 Key。

### 5.1 在 CMD 安装 Skill

此时先不要设置 API Key。如果以前配置过 APIMart Key，先按第 2.3 节在 PowerShell 清除当前进程和用户级变量，再关闭 PowerShell。然后打开 CMD，确认提示符类似 `C:\Users\alice>`，复制并执行下面这一整行：

```cmd
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/dev -g --agent codex --skill apimart-generate-media -y
```

检查安装结果：

```cmd
npx --yes skills@1.5.21 list -g --json
```

结果中应出现 `apimart-generate-media`。如果安装器仍打开 Agent 多选界面，按 `Ctrl+C` 取消，重新复制上面包含 `--agent codex` 的完整单行命令；不要在多选界面里盲目安装到所有客户端。

### 5.2 在 PowerShell 减少明文暴露并保存测试 Key

先从 APIMart 测试环境生成一枚有模型权限、额度受控的新 Key。然后打开 PowerShell，确认提示符类似 `PS C:\Users\alice>`，把下面代码块原样执行：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)

$secureKey = Read-Host "请输入新的 APIMart 测试 API Key" -AsSecureString
$apiKey = [System.Net.NetworkCredential]::new("", $secureKey).Password
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $apiKey,
    [System.EnvironmentVariableTarget]::User
)
Remove-Variable apiKey, secureKey
```

执行到 `Read-Host` 那一行后，PowerShell 会显示：

```text
请输入新的 APIMart 测试 API Key:
```

此时才粘贴真实 Key 并按 Enter。屏幕不会显示字符，这是正常现象。第一行引号里的中文只是提示文字，**不要把它替换成真实 Key**。

检查是否写入成功；下面的命令只检查格式，不会打印 Key，也不能证明该 Key 已获得模型权限：

```powershell
$key = [Environment]::GetEnvironmentVariable(
    "APIMART_API_KEY",
    [System.EnvironmentVariableTarget]::User
)

if ($key -and $key.StartsWith("sk-")) {
    "Key 已正确写入"
} else {
    "Key 未正确写入"
}

Remove-Variable key
```

设置后不要执行 `Remove-Item Env:APIMART_API_KEY`，也不要立即清除用户变量。用户级变量不是专业密钥保险箱，但能让之后重新启动的 Codex 读取测试 Key。测试结束后再用下面命令清除：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

如果真实 Key 曾出现在命令、终端历史、聊天或截图中，仅清除本机变量是不够的；必须在 APIMart 平台作废该 Key 并重新生成。

### 5.3 配置远程 HTTP MCP

#### 方式 A：`codex` 命令可用

下面是一整行，`APIMART_API_KEY` 必须原样保留，不能替换成真实 Key：

```cmd
codex mcp add apimart-dev --url https://mcp.apimart.asia/mcp --bearer-token-env-var APIMART_API_KEY
```

#### 方式 B：`codex` 命令不可用（直接编辑配置）

不要在 Codex 的“本地命令”表单里创建一个空的 `command`。APIMart 是远程 Streamable HTTP MCP，需要配置 `url`。在 PowerShell 原样执行：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex" | Out-Null
notepad "$env:USERPROFILE\.codex\config.toml"
```

如果记事本询问是否创建文件，选择“是”。如果文件中已经存在 `[mcp_servers.apimart-dev]`，先删除这个标题及其下面的旧配置行，直到下一个以 `[` 开头的配置段为止。然后在文件末尾加入且只保留一份：

```toml
[mcp_servers.apimart-dev]
url = "https://mcp.apimart.asia/mcp"
bearer_token_env_var = "APIMART_API_KEY"
```

保存时文件名必须是 `config.toml`，不能变成 `config.toml.txt`。使用上面的 `notepad` 命令打开文件，可以避免手工找错目录；若记事本弹出“另存为”，把“保存类型”改为“所有文件”。

下面两种写法都是错误的：

```toml
# 错误：把远程 HTTP MCP 写成空的本地启动命令
command = ""

# 错误：把真实 Key 当作环境变量名称
bearer_token_env_var = "sk-xxxx"
```

保存并关闭记事本，再做脱敏检查。下面只输出 5 个判断结果，不会打印配置文件内容：

```powershell
$configPath = "$env:USERPROFILE\.codex\config.toml"
$configText = Get-Content -Raw $configPath

[pscustomobject]@{
    has_apimart_section = $configText -match '(?m)^\[mcp_servers\.apimart-dev\]\s*$'
    has_mcp_url = $configText -match 'https://mcp\.apimart\.asia/mcp'
    has_env_name = $configText -match 'bearer_token_env_var\s*=\s*"APIMART_API_KEY"'
    has_empty_command = $configText -match '(?m)^\s*command\s*=\s*""\s*$'
    contains_literal_key = $configText -match 'sk-[A-Za-z0-9_-]{8,}'
}

Remove-Variable configPath, configText
```

前三项应为 `True`，后两项应为 `False`。如果 `contains_literal_key` 为 `True`，不要打印或分享配置内容；立即在 APIMart 平台作废误写的 Key，并在本机把配置改回上面的正确内容。

### 5.4 完全重启 Codex

1. 退出所有 Codex 桌面端、CLI 和 IDE 窗口，不能只关闭当前任务。
2. 关闭安装或设置变量时打开的终端。
3. 重新打开 CMD 或 PowerShell；需要 CLI 时，再从这个新终端运行 `codex`。
4. 重新打开 Codex，并新建一个任务。旧任务不会动态加载刚添加的 MCP。

若仍提示没有环境变量，先回到第 5.2 节检查用户级变量，再确认第 5.3 节保存的是环境变量名 `APIMART_API_KEY`。

### 5.5 做只读验收

在 Codex CLI 中可以先输入：

```text
/mcp
```

确认 `apimart-dev` 已启动。然后在新任务中输入：

```text
使用 apimart-dev MCP 的 list_models，列出 5 个模型，不要生成内容。
```

正常结果应调用 `apimart-dev.list_models` 并返回至少一个模型。常见结果按下面判断：

| 现象 | 含义与处理 |
| --- | --- |
| `Environment variable sk-... is not set` | 把真实 Key 错写进了 `bearer_token_env_var`。立即作废暴露的 Key，并按第 5.3 节改成 `APIMART_API_KEY`。 |
| `command` 为空或启动命令无效 | 把远程 MCP 配成了本地 stdio MCP。删除 `command = ""`，改用第 5.3 节的 `url` 配置。 |
| `401` 或 `403` | Key 无效、被禁用、环境不匹配或权限不足。重新生成目标测试环境的 Key。 |
| `total: 0`、`models: []` | MCP 已经连通，但 APIMart 测试 API 对当前 Key 返回空模型列表。检查该 Key 所属测试账号或分组是否配置了可用模型；这不是 Skill 安装失败。 |
| 返回模型但工具少于 7 个 | MCP 服务版本或客户端缓存仍旧，按第 11.5 节排查。 |

“Key 已正确写入”只说明字符串格式看起来像 Key，不代表它属于正确环境或拥有模型权限。不要把真实 Key 发给 AI 代为检查。

## 6. Cursor 接入

### 6.1 安装 `dev` Skill

下面使用的是第三方 `skills` 安装器，不是 Cursor 自带命令；它会把完整 Skill 目录复制到 Cursor 能识别的位置：

如果之前装过正式 `main`，先按第 2.3 节统一切换共享全局版本。不要尝试让全局 Codex 使用 `main`、全局 Cursor 使用 `dev`。

```bash
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/dev -g --agent cursor --skill apimart-generate-media -y
```

### 6.2 设置 API Key

- macOS 桌面版 Cursor：使用第 3.2 节的 `launchctl` 隐藏输入命令，随后完全退出并重开 Cursor。
- macOS/Linux 从终端启动 Cursor：使用第 4 节与你的 shell 对应的命令，并从同一终端启动 Cursor。
- Windows：按第 5.2 节设置用户级变量，完全退出并重新打开 Cursor；无需从保存 Key 的 PowerShell 窗口启动。测试结束后按该节说明同时清除当前进程和用户级变量。

### 6.3 配置 MCP

Cursor 官方支持两种配置位置：

- 当前项目：`<项目目录>/.cursor/mcp.json`
- 当前用户的所有项目：`~/.cursor/mcp.json`

由于 APIMart 包含会计费的生成工具，本教程建议先在专门的测试项目中使用项目级配置；需要所有项目都能调用时，再改用用户级配置。项目级文件不要误提交到业务仓库，可以把 `.cursor/mcp.json` 加入当前仓库的 `.git/info/exclude`。

在以下两种文件中任选一个：

```text
<测试项目>/.cursor/mcp.json   # 推荐，只在这个项目生效
~/.cursor/mcp.json            # 可选，对当前用户的所有项目生效
```

配置示例：

```json
{
  "mcpServers": {
    "apimart-dev": {
      "url": "https://mcp.apimart.asia/mcp",
      "headers": {
        "Authorization": "Bearer ${env:APIMART_API_KEY}"
      }
    }
  }
}
```

`${env:APIMART_API_KEY}` 是 Cursor 官方支持的环境变量插值写法。不要写成 `$APIMART_API_KEY` 或 `${APIMART_API_KEY}`，也不要为了省事把真实 Key 提交到项目级 `.cursor/mcp.json`。若文件里已有其他 `mcpServers`，只合并 `apimart-dev` 这一项，不要整份覆盖。

配置完成后：

1. 完全退出并重新打开 Cursor。
2. 打开 `Customize`，找到 `apimart-dev`。
3. 确认 `apimart-dev` 为已连接状态。
4. 打开 Output 面板（macOS：`Cmd+Shift+U`；Windows/Linux：`Ctrl+Shift+U`），选择 `MCP Logs` 查看连接日志。
5. 在 Agent 模式发送第 3.4 节的只读测试提示词。

MCP 日志可能包含请求参数，某些客户端版本还可能显示认证信息。不要直接把完整日志发到群里或工单；分享前搜索并遮盖 `Authorization`、`Bearer` 和任何 `sk-` 开头的内容。

如果安装了 Cursor CLI，也可以验证：

```bash
agent mcp list
agent mcp list-tools apimart-dev
```

第二条命令应列出 7 个 APIMart 工具。

Cursor 官方会发现 `~/.agents/skills/` 和 `~/.cursor/skills/` 中的用户级 Skill。如果不使用第三方安装器，可以手工把仓库中的整个目录：

```text
skills/apimart-generate-media/
```

复制到：

```text
~/.cursor/skills/apimart-generate-media/
```

必须复制整个目录，不能只复制 `SKILL.md`，因为其中还包含 `references/` 和 `scripts/`。

## 7. Claude Code 接入

### 7.1 安装 `dev` Skill

下面使用的是第三方 `skills` 安装器，不是 Anthropic 自带命令；它会把完整 Skill 目录复制到 Claude Code 的 Skill 目录：

如果之前装过正式 `main`，先按第 2.3 节统一切换共享全局版本。不要尝试让不同客户端的全局安装分别跟随不同分支。

```bash
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/dev -g --agent claude-code --skill apimart-generate-media -y
```

### 7.2 设置环境变量

在启动 Claude Code 的同一终端中设置。zsh 与 bash 请使用第 4 节各自对应的隐藏输入命令。Windows 按第 5.2 节写入用户级变量后，关闭旧终端并重新打开 PowerShell，再从新窗口启动 Claude Code。必须先清除旧变量、确认新 Key 写入成功后再启动，避免误用另一个账号残留的 Key。

### 7.3 添加远程 MCP

先进入一个专门的测试项目，再使用本地作用域；这样带计费能力的工具不会自动出现在所有项目中：

```bash
claude mcp add-json --scope local apimart-dev '{"type":"http","url":"https://mcp.apimart.asia/mcp","headers":{"Authorization":"Bearer ${APIMART_API_KEY}"}}'
```

外层必须使用单引号，避免 shell 在写配置时把环境变量提前替换成真实 Key。

验证：

```bash
claude mcp get apimart-dev
claude mcp list
```

进入 Claude Code 后运行：

```text
/mcp
```

确认服务器已连接，再执行第 3.4 节的只读测试。

`claude mcp add` 输出 `Added` 只表示配置已写入，不代表认证成功；应以 `claude mcp list` 的 `Connected` 状态和会话内 `/mcp` 为准。

Claude Code 官方识别的用户级 Skill 目录是：

```text
~/.claude/skills/apimart-generate-media/
```

如果不使用第三方安装器，可以手工复制完整的 `skills/apimart-generate-media/` 目录到上述位置。不要只复制 `SKILL.md`，否则本 Skill 的引用文档和本地脚本会缺失。可在 Claude Code 中输入：

```text
/apimart-generate-media
```

检查 Skill 是否可见。若首次创建 `~/.claude/skills`，请重启 Claude Code。

## 8. 其他 MCP 客户端接入原则

只要客户端同时满足以下条件，就可以尝试接入：

- 支持 Streamable HTTP MCP。
- 支持 `Authorization: Bearer ...` 请求头。
- 能从环境变量或安全凭据存储中读取 API Key。

通用配置是：

| 配置项 | 值 |
| --- | --- |
| 名称 | `apimart-dev` |
| 传输方式 | Streamable HTTP |
| URL | `https://mcp.apimart.asia/mcp` |
| Header | `Authorization: Bearer <APIMart API Key>` |

如果客户端不支持 Skill，它仍然可以直接使用 MCP 工具，但应手工遵循下面的顺序：

```text
list_models（未指定模型时）
→ get_model_docs（每次生成前）
→ get_model_schema（必要时确认图片/视频类型）
→ upload_image（仅当有本地参考图片时，上传一次并使用返回 URL）
→ generate_image / generate_video（只提交一次）
→ get_task（按建议间隔轮询）
```

音频和视频没有对应上传工具，只能把公开 HTTP(S) URL 放进模型文档指定的
字段；如果用户只有本地音频或视频附件，应先让用户自行上传并提供 URL，
不能直接把附件、base64、data URI 或本地路径发给生成接口。

ChatGPT 网页版不会读取你电脑上的 `~/.codex/config.toml`。本地 API Key 透传方案主要面向 Codex、Cursor、Claude Code 等本地 MCP 客户端。若要在 ChatGPT 网页版提供给其他用户，需要走插件/工作区分发与对应的认证方案，不能假设网页端会自动读取本机环境变量。

无论使用哪种客户端，都建议先限定到专门的测试项目，并保留 `generate_image`、`generate_video` 的人工批准。提示词里的“不要生成”不是权限边界；测试结束后应禁用或删除 MCP 配置，避免其他项目中的提示内容误触发计费工具。

## 9. 用 Apipost 独立检查 MCP

Apipost 适合排查“MCP 服务是否正常”，但它不会自动执行 Skill 工作流。当前 APIMart MCP 使用无会话（stateless）HTTP 模式并启用了 JSON 响应，因此下面的每个 POST 都可以独立执行，不需要保存 `Mcp-Session-Id`。这个简化流程是 APIMart 当前服务的特性，不应直接照搬到其他 MCP 服务。

### 9.1 请求设置

```http
POST https://mcp.apimart.asia/mcp
Authorization: Bearer <你的 APIMart API Key>
Content-Type: application/json
Accept: application/json, text/event-stream
```

请把 Key 放在 Apipost 的私密环境变量或 Secret 类型变量中，再让请求头引用该变量；不要直接把 Key 粘贴进请求头示例或保存进公开接口文档。分享或导出 Apipost 项目前，确认环境值与 `Authorization` 请求头已被排除或遮盖。响应也可能以 `text/event-stream` 返回；若界面显示 SSE 事件，请读取其中 `data:` 后面的 JSON-RPC 内容。

### 9.2 查看 7 个工具

请求体：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

检查 `result.tools`，应包含本文第 1 节列出的 7 个工具。

### 9.3 查询模型文档（只读）

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_model_docs",
    "arguments": {
      "model": "Omni-Flash-Ext"
    }
  }
}
```

正常结果应包含：

- `doc_url`
- `markdown_url`
- `markdown`
- `fetched_at`
- `cache_ttl_seconds`（当前为 `604800`，即 168 小时）
- `stale`

`stale: false` 表示返回的是缓存有效期内或刚刚拉取的最新成功文档。`stale: true` 表示刷新失败后返回了最近一次成功内容。

### 9.4 上传一张极小 PNG（会写入对象存储，不调用生成）

下面的 `image_base64` 是一个测试用小 PNG，不含 API Key：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "upload_image",
    "arguments": {
      "image_base64": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNk+A8AAQUBAScY42YAAAAASUVORK5CYII=",
      "filename": "mcp-upload-check.png"
    }
  }
}
```

正常结果包含 `url`、`filename`、`content_type`、`bytes` 和
`created_at`。上传不调用计费生成接口，但会在对象存储中创建一个图片对象，
因此不要反复执行。上传 URL 有效期为 72 小时，请及时使用或下载保存。音频
或视频 base64 会被拒绝。

## 10. 推荐验收流程

### 10.1 第一阶段：只读验收（不计费）

依次确认：

- Skill 能被客户端识别。
- MCP 显示已连接。
- `tools/list` 有 7 个工具。
- `list_models` 返回至少 1 个模型；`total: 0` 或 `models: []` 视为未通过，不继续做计费测试。
- `get_model_docs` 能返回指定模型的 Markdown。
- `get_model_schema` 能识别正确的 `image_generation` 或 `video_generation`。

推荐提示词：

```text
使用 APIMart Image & Video Skill，列出 5 个图片或视频模型，
再读取 Omni-Flash-Ext 的实时文档和兼容性 Schema。
不要提交生成任务。
```

### 10.2 图片上传验收（不计生成费）

准备一张 JPEG、PNG、GIF 或 WebP 图片，在支持附件的客户端中新建任务并
发送；不要为了客户端测试而主动压缩或裁剪文件：

```text
使用 APIMart Image & Video Skill，只把我附加的图片上传成可访问 URL。
只上传一次，返回 URL、文件类型和字节数；不要生成任何内容。
```

验收点：

- 只上传一次，不调用 `generate_image` 或 `generate_video`。本地可读文件优先
  由 Skill 使用 `upload-image --file` 直连 APIMart；其他客户端可调用 MCP
  `upload_image`。
- 返回 `http://` 或 `https://` URL，且该地址可以访问。
- Skill 和 MCP 不按图片字节数预先拒绝，最终以上传接口响应为准。
- 实际文件不是 JPEG、PNG、GIF、WebP 时，上传失败。
- 不会把用户电脑上的本地路径传给远程 MCP；远程 Pod 读取不到该路径。

再做两次拒绝测试，均不应提交生成任务：

```text
我附了本地 demo.mp4，把它直接作为参考视频生成新视频。
```

```text
音频是 data:audio/wav;base64,...，请直接传给模型生成。
```

当前没有音频或视频上传工具，AI 应要求用户提供公开的 HTTP(S) URL。
`file://`、本地路径、base64、原始字节和音视频 data URI 都会在计费提交前
被拒绝。已有的公开图片 URL 不需要重复上传。

### 10.3 第二阶段 A：图片生成前检查（不提交、不计生成费）

先发送下面这条，并等待 AI 停下来：

```text
使用 APIMart Image & Video Skill，只做图片生成前检查：
读取候选模型的实时文档，给出准确模型 ID、完整请求参数、是否计费，
以及文档能确认的价格或预计费用上限；查不到价格就明确说“未知”，不要猜。
绝对不要调用 generate_image，展示方案后停止，等待我单独确认。
```

确认模型、参数、账号余额和可接受预算后，再进入下一步。AI 如果在这一轮已经提交任务，说明人工确认流程没有生效，应立即停止后续测试并检查是否产生账单。

### 10.4 第二阶段 B：单独确认图片生成（会计费）

只有你确实接受上一轮方案时，才单独发送：

```text
我明确确认按上一轮展示的模型和参数生成 1 张图片。
首次提交前创建并记录一个新的幂等键；只调用 generate_image 一次。
如果返回 task_id，只在 should_poll 为 true 时按 next_poll_after_seconds 查询这个任务；
最多查询 10 分钟或 120 次，先到即停止并把 task_id 返回给我。
失败时不要自动换模型或再创建任务。
```

验收点：

- 有明确模型 ID。
- 提交前读取了 `get_model_docs`。
- 返回了任务 ID 或同步结果。
- 异步任务最终进入 `completed` 或 `failed` 终态。
- `completed` 时返回的图片地址可以打开。
- 没有在失败后擅自重新提交第二个计费任务。

### 10.5 幂等键与超时恢复（防止重复计费）

幂等键不是 API Key，不是秘密；它是一次“逻辑生成请求”的唯一编号。例如：

```text
apimart-img-20260801T120000Z-a1b2c3
```

要求如下：

- 使用 1～191 个可见 ASCII 字符，不含空格。
- 在第一次调用 `generate_image`/`generate_video` **之前**生成并保存。
- 与准确的 `model` 和完整 `input` 一起记录在当前任务或测试记录中。
- 同一次逻辑请求的恢复调用必须复用完全相同的幂等键、模型和输入。
- 参数改变后属于新请求，必须再次确认，并使用新的幂等键。

如果请求超时、断网或客户端崩溃：

1. 若已经拿到 `task_id`，不要再调用生成工具，只用 `get_task` 查询原任务。
2. 若没有拿到 `task_id`，结果属于“未知”，不要用新幂等键重新生成。
3. 只有保留了原幂等键、准确模型和完整输入时，才可用三者完全相同的 `generate_*` 调用恢复；服务端会按幂等记录返回原请求，而不是创建另一笔不同请求。
4. 如果任一项没有保存，停止操作并让管理员按日志或账单核对，不能猜一个新键继续。

### 10.6 第三阶段：视频生成验收（可选，会计费）

如果图片端到端已经通过、当前需求不要求视频，可以跳过本阶段。需要验证视频时，同样先拆成两轮。

第一轮只做预览：

```text
使用 APIMart Image & Video Skill，只做视频生成前检查：
读取低成本候选模型的实时文档，展示准确模型、最低支持时长、完整参数，
以及能确认的价格或预计费用上限；不要调用 generate_video，等待我确认。
```

接受方案后再单独发送：

```text
我明确确认按上一轮展示的模型和参数生成 1 个视频。
首次提交前创建并记录一个新的幂等键；只调用 generate_video 一次。
如果返回 task_id，只在 should_poll 为 true 时按 next_poll_after_seconds 查询；
最多查询 10 分钟或 120 次，先到即停止并把 task_id 返回给我。
失败时不要自动重提。
```

验收点与图片相同，同时确认视频可以打开或下载。

## 11. 常见问题

### 11.1 `codex: command not found`

macOS Codex 桌面端可尝试：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp list
```

如果这个路径也不存在，请在 Codex 设置中使用 MCP Servers 页面配置，或确认应用安装位置。

Windows 不需要先解决 `codex` 命令才能接入：直接按第 5.3 节“方式 B”编辑 `%USERPROFILE%\.codex\config.toml`。APIMart 是远程 HTTP MCP，配置中必须有 `url`，不能创建空的 `command = ""`。

### 11.2 `rg: command not found`

`rg` 是 ripgrep，不是 macOS 默认命令。改用：

```bash
grep -nF "get_model_docs" ~/.agents/skills/apimart-generate-media/SKILL.md
```

### 11.3 已添加 MCP，但提示没有 `APIMART_API_KEY`

- 确认 `--bearer-token-env-var` 后写的是 `APIMART_API_KEY`。
- 如果错误中出现 `Environment variable sk-... is not set`，说明真实 Key 被误写成了环境变量名。立即作废该 Key，把配置改回 `APIMART_API_KEY`。
- 重新设置环境变量。
- Windows 用户级环境变量只会被之后启动的进程继承；完全退出所有 Codex 窗口和旧终端，再打开新终端和新任务。
- 不要把真实 Key 发到聊天里让 AI 检查。

### 11.4 返回 401 或 403

- Key 不存在、已过期、被禁用或权限不足。
- 确认使用的是目标环境对应的 Key。
- 在 APIMart 平台重新生成 Key 后，更新本机环境变量并重启客户端。

### 11.5 工具少于 7 个，缺少 `get_model_docs` 或 `upload_image`

- 客户端仍连接旧版本 MCP。
- 完全重启客户端并重新查看工具列表。
- 确认 URL 为 `https://mcp.apimart.asia/mcp`。
- 如果服务刚发布，确认对应 CI/CD 和 Pod 已更新。

### 11.6 Skill 仍使用旧流程

执行第三方安装器前，先清除当前终端里的 Key。macOS/Linux：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 update apimart-generate-media -g -y
```

Windows 请先在 PowerShell 清除当前进程和用户级变量：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

再回到 CMD 执行更新；若 `npx` 在 PowerShell 也可用，执行同一行也可以：

```cmd
npx --yes skills@1.5.21 update apimart-generate-media -g -y
```

然后重新注入 Key，并重启客户端或新建任务。`skills list -g --json` 只能确认名称、路径和仓库来源，不能证明具体分支；检查全局锁文件中的 `ref`：

```bash
grep -n '"ref"' ~/.agents/.skill-lock.json
```

PowerShell 使用：

```powershell
Select-String -Path "$HOME\.agents\.skill-lock.json" -Pattern '"ref"'
```

从本教程地址安装的测试版应显示 `"ref": "dev"`。若 `ref` 不存在或来源不明确，按第 2.3 节清理共享全局版本，再重新执行第 3.1 节的 `dev` 安装命令。

### 11.7 参数看起来比模型文档多很多

这通常表示 AI 把兼容性 Schema 的通用字段误当成了模型参数。明确要求：

```text
请以 get_model_docs 返回的 Markdown 为真实参数来源，
get_model_schema 只用于确认操作和传输契约。
```

### 11.8 文档返回 `stale: true`

表示上游文档刷新失败，但系统返回了最近一次成功副本。一般仍可使用；如果正在验证刚更新的参数，应等待文档服务恢复或让管理员检查模型管理中的“开发文档链接”。

### 11.9 任务失败后会自动退款吗

以 APIMart 返回的任务状态和账单记录为准。Skill 遇到终态失败时应停止轮询并报告错误，不会自行更换模型或再提交一次。需要重试时，应由用户明确确认。

### 11.10 图片或视频地址打不开

- 上传及生成结果 URL 的有效期为 72 小时，请及时下载保存。
- 确认任务状态确实为 `completed`。
- 尝试在浏览器中直接打开原始 URL。
- 若持续失败，保留任务 ID 交给 APIMart 排查，不要先重复提交计费任务。

### 11.11 是否需要配置 OAuth 或 `APIMART_BASE_URL`

- 当前 MCP 使用 APIMart API Key 透传，不需要 `codex mcp login` 或其他 OAuth 登录。
- MCP 客户端只需配置 MCP URL 与 `APIMART_API_KEY`。
- `APIMART_BASE_URL` 只属于本地 API 备用模式，不属于 MCP 配置。
- 不要把 `APIMART_BASE_URL` 设置成 `https://mcp.apimart.asia/mcp`。

### 11.12 `upload_image` 返回 413

- 如果 `413` 来自 APIMart 上传接口，说明接口拒绝了本次文件。原样反馈，
  不要自动重试、拆分文件或改参数绕过。
- 如果请求尚未调用工具就被 MCP 客户端、Ingress 或 HTTP body parser 拒绝，
  这是通用传输限制。对本地可读图片使用 Skill 的 `upload-image --file`
  直连 multipart 流程，让 APIMart 上传接口作最终判断。
- Skill 和 MCP 不维护另一套图片字节数规则，也不把基础设施传输上限描述成
  APIMart 的图片大小限制。
- 不要通过改扩展名绕过限制；服务会按真实文件内容识别 MIME 类型。
- 不要改用音频或视频上传尝试绕过，当前只开放图片上传。

### 11.13 Windows 下 CMD 与 PowerShell 命令互相报错

- 提示符是 `C:\Users\alice>` 时是 CMD，不能执行 `$key = ...`、`Remove-Item` 或其他 PowerShell 语法。
- 提示符是 `PS C:\Users\alice>` 时是 PowerShell。若这里找不到 `npm`/`npx`，但 CMD 可以使用，就在 CMD 运行 Skill 安装命令，在 PowerShell 只完成第 5.2、5.3 节的 Key 和配置步骤。
- `finally` 不能在上一段 `try/catch` 已执行完成后单独粘贴。清理普通临时变量可直接执行 `Remove-Variable key, headers, result -ErrorAction SilentlyContinue`；不要在接入尚未验收时清除用户级 `APIMART_API_KEY`。

### 11.14 `list_models` 返回 `total: 0` 或 `models: []`

这表示 MCP 调用已经到达 APIMart，并收到了一个有效但为空的模型列表；它不表示 Skill 安装失败，也不等同于“7 个 MCP 工具可见”。依次检查：

1. Key 是否来自本教程要求的目标测试环境，而不是另一套环境。
2. Key 所属账号、分组或渠道是否配置了至少一个可用模型。
3. 让 APIMart 管理员确认该 MCP 部署实际配置的 API 上游，并在**同一个上游**检查该 Key 的 `/v1/models?limit=5` 结果。不要根据 `dev` 这个 Skill 分支名自行猜测 `api-dev` 域名；MCP URL 与后端 API URL 是两层不同地址。

若管理员确认的同一上游也返回空列表，把 HTTP 状态、`total`、`count` 等脱敏字段交给管理员检查账号/分组配置。若同一上游直连有模型而 MCP 仍为空，则由管理员检查 MCP 的上游配置和路由。普通接入用户无需在命令中自己拼 `Authorization` 请求头；这样更容易把 Key 留在历史或截图中。

空列表时只读验收未通过，不要继续调用图片或视频生成。排查时只提供 HTTP 状态、`total`、`count` 和空数组等脱敏结果，不要提供 `Authorization` 请求头或真实 Key。

## 12. 从 dev 切回正式 main

`dev` 用于测试，不建议长期提供给普通用户。正式发布前，应先把已验收内容合并进 `main`；若团队发布了不可变 release tag，正式用户优先安装 tag。下面至少显式指定 `/tree/main`，不依赖仓库默认分支。

切换共享全局版本会影响这台电脑上使用该副本的所有客户端。先备份手工改过的文件、退出所有 AI 客户端，并清除当前终端里的 Key。

macOS/Linux：

```bash
unset APIMART_API_KEY
```

PowerShell：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

然后移除该 Skill 的共享全局安装。Windows 如果 `npx` 只在 CMD 可用，完成上面的 PowerShell 清理后关闭 PowerShell，再打开 CMD。下面是一整行，在能使用 `npx` 的终端中执行：

```text
npx --yes skills@1.5.21 remove apimart-generate-media -g --agent "*" -y
```

再为实际使用的每个客户端安装同一个 `main` 来源。以下每个代码块都只有一行，只复制实际需要的客户端命令，不要把多个代码块一起粘贴。

Codex：

```text
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent codex --skill apimart-generate-media -y
```

Cursor：

```text
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent cursor --skill apimart-generate-media -y
```

Claude Code：

```text
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent claude-code --skill apimart-generate-media -y
```

`skills list -g --json` 不能证明具体分支。macOS/Linux 用下面的命令检查锁文件：

```bash
grep -n '"ref"' ~/.agents/.skill-lock.json
```

PowerShell 使用：

```powershell
Select-String -Path "$HOME\.agents\.skill-lock.json" -Pattern '"ref"'
```

正式安装应显示 `"ref": "main"`。确认后重新注入 API Key，重启客户端，并重做第 10.1 节的只读验收。

`apimart-dev` 只是本机 MCP 配置的显示名称，不决定服务器环境；真正决定连接位置的是 URL。为了避免正式用户误解，生产配置建议改名为 `apimart`。Codex 可以执行：

```bash
codex mcp remove apimart-dev
codex mcp add apimart --url https://mcp.apimart.asia/mcp --bearer-token-env-var APIMART_API_KEY
```

Windows 上 `codex` 命令不可用时，用第 5.3 节“方式 B”打开 `%USERPROFILE%\.codex\config.toml`，删除 `[mcp_servers.apimart-dev]` 标题及其 `url`、`bearer_token_env_var` 配置行。若该段后面还有别的 `[mcp_servers...]` 或其他配置段，不要删除它们。保存后完全重启 Codex；需要正式配置时，再以 `[mcp_servers.apimart]` 为标题加入同样的 URL 和环境变量名。

Cursor 可把 `mcp.json` 中的键名从 `apimart-dev` 改为 `apimart`；Claude Code 可以删除本地测试条目后，以 `apimart` 为名称重新执行第 7.3 节的 `add-json` 命令。改名不会改变 URL，也不会改变计费规则。

## 13. 安全检查清单

接入完成后逐项确认：

- [ ] API Key 没有出现在聊天、截图、文档或 Git 提交中。
- [ ] PowerShell 的 `Read-Host` 提示文字未被替换成真实 Key，真实 Key 只在隐藏提示后输入。
- [ ] MCP 配置保存的是环境变量名，而不是裸 Key。
- [ ] Codex 的远程 MCP 配置包含 `url` 和 `bearer_token_env_var`，没有空的 `command = ""`。
- [ ] 测试使用低额度测试 Key。
- [ ] MCP URL 使用 HTTPS。
- [ ] 线上工具数量为 7。
- [ ] `list_models` 返回至少 1 个模型；没有把 7 个工具可见误当成模型权限已通过。
- [ ] 每次生成前读取该模型的实时文档。
- [ ] `upload_image` 能上传允许格式的参考图片，错误格式和超大图片会被拒绝。
- [ ] 音频和视频只使用公开 HTTP(S) URL，没有上传本地文件、base64 或 data URI。
- [ ] 至少完成一次所需媒体类型的端到端测试；不需要视频时没有为了验收而额外计费。
- [ ] 首次提交前保存了幂等键、模型和完整输入；恢复时三者完全复用。
- [ ] 结果 URL 可以打开，并已在 72 小时有效期内及时保存。
- [ ] `dev` 验证通过后，已把验收内容合并进 `main`，并核对正式安装来源；有 release tag 时优先使用不可变 tag。

## 参考资料

- [OpenAI Codex：Model Context Protocol](https://learn.chatgpt.com/docs/extend/mcp)
- [OpenAI Codex：Build skills](https://learn.chatgpt.com/docs/build-skills)
- [OpenAI Codex：Configuration Reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [Cursor：Model Context Protocol](https://cursor.com/docs/mcp)
- [Cursor：Agent Skills](https://cursor.com/docs/skills)
- [Anthropic Claude Code：MCP](https://code.claude.com/docs/en/mcp)
- [Anthropic Claude Code：Skills](https://code.claude.com/docs/en/slash-commands#where-skills-live)
- [APIMart 文档](https://docs.apimart.ai/cn/)
- [skills.sh CLI](https://skills.sh/docs/cli)
