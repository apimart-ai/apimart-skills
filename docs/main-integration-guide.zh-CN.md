# APIMart MCP + Skill `main` 正式版接入教程

> 适用对象：第一次接触 MCP、Skill 或 AI 编程助手的 APIMart 正式用户。
>
> Skill 来源：`apimart-ai/apimart-skills` 的 `main` 分支。
>
> 生产 MCP 地址：`https://mcp.apimart.ai/mcp`
>
> 更新时间：2026-08-05

**不要混用环境：**生产 Skill、生产 MCP 地址和用户在 APIMart 正式平台生成的 API Key 必须配套使用。测试环境仍使用 `dev` Skill 与 `https://mcp.apimart.asia/mcp`，不要把测试地址复制到本教程的生产配置中。

## 目录

1. [先理解 MCP 和 Skill](#1-先理解-mcp-和-skill)
2. [接入前准备](#2-接入前准备)
3. [Codex 桌面端接入（macOS，推荐）](#3-codex-桌面端接入macos推荐)
4. [Codex CLI 接入（macOS/Linux）](#4-codex-cli-接入macoslinux)
5. [Codex 接入（Windows）](#5-codex-接入windows)
6. [Cursor 接入](#6-cursor-接入)
7. [Claude Code 接入](#7-claude-code-接入)
8. [其他 MCP 客户端接入原则](#8-其他-mcp-客户端接入原则)
9. [使用 Apipost 独立检查 MCP](#9-使用-apipost-独立检查-mcp)
10. [正式环境验收流程](#10-正式环境验收流程)
11. [更新、卸载与从 dev 切换到 main](#11-更新卸载与从-dev-切换到-main)
12. [常见问题](#12-常见问题)
13. [安全检查清单](#13-安全检查清单)

## 1. 先理解 MCP 和 Skill

接入 APIMart 需要配置两部分：

| 部分 | 大白话解释 | 负责什么 |
| --- | --- | --- |
| MCP | AI 助手连接 APIMart 的“在线工具接口” | 查询模型、读取文档、上传参考图片、创建图片/视频任务、查询任务结果 |
| Skill | 告诉 AI 助手“这些工具应该按什么顺序使用” | 先读模型文档、正确拼参数、避免重复计费、按规则轮询任务 |

只安装 Skill 而没有配置 MCP 时，AI 助手只有操作说明，没有在线工具可调用。只配置 MCP 而没有安装 Skill 时，工具虽然可用，但 AI 助手不一定每次都按 APIMart 推荐流程执行。

生产接入固定使用以下配置：

| 配置项 | 正式环境值 |
| --- | --- |
| Skill 分支 | `main` |
| MCP 名称 | `apimart` |
| MCP URL | `https://mcp.apimart.ai/mcp` |
| 认证方式 | 用户自己的 APIMart API Key，通过 Bearer Token 透传 |
| OAuth | 不需要 |

MCP 模式下不需要设置 `APIMART_BASE_URL`。这个变量只供 Skill 自带的本地 API 备用脚本使用，默认生产 API 地址为 `https://api.apimart.ai`；它不能填写 MCP URL。

正常生成流程如下：

```text
用户提出需求
  → Skill 判断需要图片还是视频
  → list_models 查找模型（用户未指定模型时）
  → get_model_docs 读取该模型的实时 Markdown 文档
  → get_model_schema 必要时确认图片/视频操作类型
  → 有本地参考图片时，upload_image 上传一次并取得 URL
  → generate_image 或 generate_video 只提交一次
  → 返回 task_id
  → get_task 按建议间隔查询
  → 返回最终图片或视频地址
```

APIMart MCP 正式环境应提供 7 个工具：

1. `list_models`
2. `get_model_docs`
3. `get_model_schema`
4. `upload_image`
5. `generate_image`
6. `generate_video`
7. `get_task`

`get_model_docs` 返回的实时 Markdown 才是对应模型真实参数、可选值和组合规则的主要依据。`get_model_schema` 是兼容性契约，不能把其中的通用字段全部当成某个模型都支持的参数。

## 2. 接入前准备

### 2.1 必须准备

- 在 APIMart 正式平台生成的一枚可用 API Key。
- 可以访问 `https://mcp.apimart.ai/mcp` 的网络。
- Node.js 22.20 或更高版本（第三方 `skills@1.5.21` 安装器的要求；Skill 自带脚本支持 Node.js 20+）。
- 已安装准备接入的客户端，例如 Codex、Cursor 或 Claude Code。
- 客户端支持远程 Streamable HTTP MCP 和 Bearer Token 请求头。

检查 Node.js 与 npm：

```bash
node -v
npm -v
```

如果 `node -v` 低于 `v22.20.0`，请先升级。若提示找不到命令，请从 [Node.js 官网](https://nodejs.org/) 安装满足版本要求的 LTS 版本，然后关闭并重新打开终端。

### 2.2 API Key 安全规则

- 不要把 API Key 发到聊天、群聊、工单或截图中。
- 不要把 API Key 写入 Git 仓库、Markdown、示例 JSON、URL 或 MCP 配置文件。
- `--bearer-token-env-var` 后填写的是环境变量名 `APIMART_API_KEY`，不是以 `sk-` 开头的真实 Key。
- 先安装并检查 Skill，再设置 API Key。不要让不可信安装程序继承包含 Key 的环境变量。
- 正式使用建议创建独立、限额、可撤销的 Key，不要共用管理员 Key。
- 如果真实 Key 曾出现在命令、聊天或截图中，应立即在 APIMart 平台作废并重新生成。
- 环境变量不是专业密钥保险箱；企业环境应优先使用操作系统凭据库或统一密钥管理方案。

错误示例：

```bash
# 错误：真实 Key 被当成环境变量名称
codex mcp add apimart \
  --url https://mcp.apimart.ai/mcp \
  --bearer-token-env-var sk-xxxx
```

正确写法固定为：

```text
--bearer-token-env-var APIMART_API_KEY
```

### 2.3 已安装 dev 版本时先切换

同一台电脑上的全局 Skill 通常共享 `~/.agents/skills/`，不同 AI 客户端可能读取同一份副本。因此不要长期让一部分客户端使用 `dev`、另一部分使用 `main`。

如果之前安装过 `dev`，先退出所有 AI 客户端并清除当前环境中的 Key。macOS/Linux：

```bash
unset APIMART_API_KEY
```

Windows PowerShell：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

然后在能够使用 `npx` 的终端执行：

```text
npx --yes skills@1.5.21 remove apimart-generate-media -g --agent "*" -y
```

Windows 如果只有 CMD 能使用 `npx`，就在 PowerShell 清除 Key 后关闭 PowerShell，再打开 CMD 执行上面这一整行。

卸载 Skill 不会自动删除旧 MCP。若 `codex mcp list` 中存在 `apimart-dev`，请执行 `codex mcp remove apimart-dev`；若旧测试配置也叫 `apimart`，但 URL 是 `https://mcp.apimart.asia/mcp`，请先执行 `codex mcp remove apimart`。Windows 没有 `codex` 命令时，按第 5.3 节方式 B 打开配置文件，删除指向 `.asia` 测试地址的旧配置段。

## 3. Codex 桌面端接入（macOS，推荐）

Codex 桌面端、Codex CLI 和 Codex IDE 扩展在同一台 Codex 主机上通常共享 MCP 配置，配置一次即可。

### 3.1 安装 `main` 正式版 Skill

先确保当前终端没有 `APIMART_API_KEY`，然后执行：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main \
  -g \
  --agent codex \
  --skill apimart-generate-media \
  -y
```

检查安装结果：

```bash
npx --yes skills@1.5.21 list -g --json
```

结果中应出现：

```text
apimart-generate-media
```

还可以检查正式版是否包含实时模型文档流程：

```bash
grep -nF "get_model_docs" ~/.agents/skills/apimart-generate-media/SKILL.md
```

若安装器报告了其他安装路径，请使用实际路径。

### 3.2 设置 API Key

使用隐藏输入，避免 Key 留在 shell 历史中：

```zsh
unset APIMART_API_KEY
read -s "APIMART_API_KEY?请输入 APIMart API Key: "
echo
launchctl setenv APIMART_API_KEY "$APIMART_API_KEY"
unset APIMART_API_KEY
```

输入时终端不显示字符属于正常现象。`launchctl setenv` 让之后重新启动的 macOS 图形应用能够读取变量，但它不是加密存储。设置完成后必须彻底退出并重新打开 Codex。

需要清除时执行：

```bash
launchctl unsetenv APIMART_API_KEY
```

不要运行 `launchctl getenv APIMART_API_KEY` 后截图，因为该命令会输出真实 Key。

### 3.3 添加生产 MCP

如果终端可以使用 `codex`：

```bash
codex mcp add apimart \
  --url https://mcp.apimart.ai/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

如果提示 `zsh: command not found: codex`，尝试 Codex 桌面端自带的 CLI：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp add apimart \
  --url https://mcp.apimart.ai/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

验证配置：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp list
```

预期看到类似内容：

```text
Name     Url                         Bearer Token Env Var  Status
apimart  https://mcp.apimart.ai/mcp  APIMART_API_KEY       enabled
```

这里只会显示环境变量名称，不会显示真实 Key。

### 3.4 重启并做只读测试

1. 完全退出 Codex 桌面端。
2. 重新打开 Codex。
3. 新建一个任务，旧任务不会动态加载刚添加的 MCP。
4. 发送下面的提示词：

```text
使用 APIMart Image & Video Skill，只调用 apimart MCP 的只读工具：
先确认工具数量，再列出 5 个可用模型并查询 Omni-Flash-Ext 的实时模型文档。
告诉我 duration 支持哪些值，不要生成内容。
```

预期结果：

- MCP 有 7 个工具。
- `list_models` 至少返回一个模型。
- 能调用 `get_model_docs`。
- 没有调用 `generate_image` 或 `generate_video`。

### 3.5 手工编辑 `config.toml`（可选）

如果不使用 `codex mcp add`，编辑：

```text
~/.codex/config.toml
```

加入：

```toml
[mcp_servers.apimart]
url = "https://mcp.apimart.ai/mcp"
bearer_token_env_var = "APIMART_API_KEY"
```

不要加入空的 `command = ""`，也不要把真实 Key 写进文件。保存后彻底重启 Codex。

## 4. Codex CLI 接入（macOS/Linux）

先在没有 Key 的终端中安装正式 Skill：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main \
  -g \
  --agent codex \
  --skill apimart-generate-media \
  -y
npx --yes skills@1.5.21 list -g --json
```

确认来源无误后，再让 Key 只存在于当前终端。先查看 shell：

```bash
echo "$SHELL"
```

zsh：

```zsh
unset APIMART_API_KEY
read -s "APIMART_API_KEY?请输入 APIMart API Key: "
echo
export APIMART_API_KEY
```

bash：

```bash
unset APIMART_API_KEY
read -rsp "请输入 APIMart API Key: " APIMART_API_KEY
echo
export APIMART_API_KEY
```

添加 MCP：

```bash
codex mcp add apimart \
  --url https://mcp.apimart.ai/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

检查并启动：

```bash
codex mcp list
codex
```

进入 Codex 后可以输入 `/mcp` 查看连接状态，再执行第 3.4 节的只读测试。结束当前会话后执行 `unset APIMART_API_KEY`，或直接关闭终端。

## 5. Codex 接入（Windows）

Windows 请固定按以下顺序操作，不要把 CMD 与 PowerShell 命令混在同一个窗口：

```text
CMD 安装 Skill
  → PowerShell 隐藏输入并保存 Key
  → 添加远程 HTTP MCP
  → 完全退出并重启 Codex
  → 新建任务做只读测试
```

| 终端 | 提示符示例 | 用途 |
| --- | --- | --- |
| CMD | `C:\Users\alice>` | 运行 `npx` 单行安装命令 |
| PowerShell | `PS C:\Users\alice>` | 隐藏输入 Key、编辑和检查配置 |

看到 `C:\...>` 时不能执行以 `$` 开头的 PowerShell 命令。看到 `PS C:\...>` 时，如果 `npm`/`npx` 不可用但 CMD 可用，返回 CMD 安装 Skill，也可以尝试 `npx.cmd`。

### 5.1 在 CMD 安装正式 Skill

打开 CMD，复制下面完整的一行：

```cmd
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent codex --skill apimart-generate-media -y
```

检查安装结果：

```cmd
npx --yes skills@1.5.21 list -g --json
```

结果中应出现 `apimart-generate-media`。如果出现 Agent 多选界面，按 `Ctrl+C` 取消，再重新复制包含 `--agent codex` 的完整命令。

安装后先在 PowerShell 核对该 Skill 的来源分支：

```powershell
$lock = Get-Content -Raw "$HOME\.agents\.skill-lock.json" | ConvertFrom-Json
$lock.skills.'apimart-generate-media'.ref
Remove-Variable lock
```

结果必须是 `main`；如果显示 `dev`，先按第 2.3 节移除共享安装，再重新执行本节的 `tree/main` 安装命令。

### 5.2 在 PowerShell 隐藏输入并保存 Key

打开 PowerShell，确认提示符以 `PS` 开头，把下面代码原样执行：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)

$secureKey = Read-Host "请输入 APIMart API Key" -AsSecureString
$apiKey = [System.Net.NetworkCredential]::new("", $secureKey).Password
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $apiKey,
    [System.EnvironmentVariableTarget]::User
)
Remove-Variable apiKey, secureKey
```

只有看到下面提示后，才粘贴真实 Key：

```text
请输入 APIMart API Key:
```

输入过程不显示字符。引号里的中文只是提示文字，不能替换成真实 Key。

检查是否写入；下面只检查格式，不打印 Key：

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

“Key 已正确写入”只说明格式看起来正确，不代表该 Key 有模型权限。

### 5.3 配置远程 HTTP MCP

#### 方式 A：`codex` 命令可用

在 CMD 执行下面一整行，`APIMART_API_KEY` 必须原样保留：

```cmd
codex mcp add apimart --url https://mcp.apimart.ai/mcp --bearer-token-env-var APIMART_API_KEY
```

#### 方式 B：`codex` 命令不可用

在 PowerShell 原样执行：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex" | Out-Null
notepad "$env:USERPROFILE\.codex\config.toml"
```

如果记事本询问是否创建文件，选择“是”。若已有 `[mcp_servers.apimart]`，先删除该标题及其旧配置行，直到下一个以 `[` 开头的配置段为止。然后在文件末尾加入且只保留一份：

```toml
[mcp_servers.apimart]
url = "https://mcp.apimart.ai/mcp"
bearer_token_env_var = "APIMART_API_KEY"
```

保存时文件名必须是 `config.toml`，不能是 `config.toml.txt`。不要配置 `command = ""`，不要把 `bearer_token_env_var` 的值写成真实 Key。

保存后使用下面的脱敏检查，它不会打印配置文件或 Key：

```powershell
$configPath = "$env:USERPROFILE\.codex\config.toml"
$configText = Get-Content -Raw $configPath
$sectionMatch = [regex]::Match(
    $configText,
    '(?ms)^\[mcp_servers\.apimart\]\s*.*?(?=^\[|\z)'
)
$apimartBlock = $sectionMatch.Value

[pscustomobject]@{
    has_apimart_section = $sectionMatch.Success
    has_mcp_url = $apimartBlock -match 'https://mcp\.apimart\.ai/mcp'
    has_test_url = $configText -match 'https://mcp\.apimart\.asia/mcp'
    has_env_name = $apimartBlock -match 'bearer_token_env_var\s*=\s*"APIMART_API_KEY"'
    has_empty_command = $apimartBlock -match '(?m)^\s*command\s*=\s*""\s*$'
    contains_literal_key = $configText -match 'sk-[A-Za-z0-9_-]{8,}'
}

Remove-Variable configPath, configText, sectionMatch, apimartBlock
```

`has_apimart_section`、`has_mcp_url`、`has_env_name` 应为 `True`；`has_test_url`、`has_empty_command`、`contains_literal_key` 应为 `False`。如果 `contains_literal_key` 为 `True`，不要展示配置内容，应立即作废误写的 Key 并修正配置。

### 5.4 完全重启并验收

1. 退出所有 Codex 桌面端、CLI 和 IDE 窗口。
2. 关闭旧终端。
3. 重新打开 CMD 或 PowerShell；需要 CLI 时，从新终端运行 `codex`。
4. 新建 Codex 任务。
5. 输入 `/mcp`，确认 `apimart` 已连接。
6. 发送：

```text
使用 APIMart Image & Video Skill，只调用生产 apimart MCP 的只读工具：
确认工具总数和名称，列出 5 个可用模型，再读取 Omni-Flash-Ext 的实时文档
和兼容性 Schema。不要调用 upload_image、generate_image 或 generate_video。
```

正常结果应看到 7 个工具、至少一个模型，并成功返回文档与操作类型。常见异常：

| 现象 | 原因与处理 |
| --- | --- |
| `Environment variable sk-... is not set` | 真实 Key 被写成了环境变量名；立即作废暴露的 Key，并改回 `APIMART_API_KEY`。 |
| `command` 为空 | 把远程 MCP 配成了本地 stdio MCP；删除 `command = ""` 并配置 `url`。 |
| `401` 或 `403` | Key 无效、被禁用或权限不足；在 APIMart 正式平台重新生成。 |
| `total: 0`、`models: []` | MCP 已连通，但当前 Key 没有可用模型；检查账号、分组或模型权限。 |

测试结束后如需删除用户级变量，在 PowerShell 执行：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

## 6. Cursor 接入

### 6.1 安装正式 Skill

在尚未设置 Key 的终端执行：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent cursor --skill apimart-generate-media -y
```

Windows 如果只有 CMD 能使用 `npx`，执行下面这一整行：

```cmd
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent cursor --skill apimart-generate-media -y
```

### 6.2 设置 API Key

- macOS 图形版 Cursor：使用第 3.2 节的 `launchctl` 隐藏输入流程，然后彻底退出并重新打开 Cursor。
- macOS/Linux 从终端启动 Cursor：使用第 4 节与你的 shell 对应的环境变量命令，并从同一终端启动 Cursor。
- Windows：使用第 5.2 节设置用户级变量，再关闭旧窗口并重新启动 Cursor。

### 6.3 配置 MCP

Cursor 支持项目级与用户级配置：

```text
<项目目录>/.cursor/mcp.json   # 只对当前项目生效，推荐先使用
~/.cursor/mcp.json            # 对当前用户的所有项目生效
```

项目级配置不要提交到业务仓库。可将 `.cursor/mcp.json` 加入当前仓库的 `.git/info/exclude`。

配置示例：

```json
{
  "mcpServers": {
    "apimart": {
      "url": "https://mcp.apimart.ai/mcp",
      "headers": {
        "Authorization": "Bearer ${env:APIMART_API_KEY}"
      }
    }
  }
}
```

`${env:APIMART_API_KEY}` 是环境变量插值，不能替换成真实 Key。如果已有其他 `mcpServers`，只合并 `apimart` 这一项，不要覆盖整个文件。

完成后：

1. 完全退出并重新打开 Cursor。
2. 打开 `Customize`，找到 `apimart` 并确认已连接。
3. 打开 Output 面板，选择 `MCP Logs` 查看连接日志。
4. 在 Agent 模式执行第 3.4 节的只读测试。

MCP 日志可能包含请求参数甚至认证信息，分享前必须遮盖 `Authorization`、`Bearer` 和所有 `sk-` 开头的内容。

如安装了 Cursor CLI，可以执行：

```bash
agent mcp list
agent mcp list-tools apimart
```

第二条命令应列出 7 个工具。

## 7. Claude Code 接入

### 7.1 安装正式 Skill

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills/tree/main -g --agent claude-code --skill apimart-generate-media -y
```

Windows CMD 使用同一条单行命令，但不要执行开头的 `unset`。

### 7.2 设置环境变量

- zsh/bash：使用第 4 节对应的隐藏输入命令，并从同一终端启动 Claude Code。
- Windows：按第 5.2 节设置用户级变量，关闭旧终端后重新打开 PowerShell，再启动 Claude Code。

### 7.3 添加远程 MCP

先进入准备使用 APIMart 的项目，再添加本地作用域配置：

```bash
claude mcp add-json --scope local apimart '{"type":"http","url":"https://mcp.apimart.ai/mcp","headers":{"Authorization":"Bearer ${APIMART_API_KEY}"}}'
```

外层单引号用于避免 shell 在写配置时把变量提前替换成真实 Key。验证：

```bash
claude mcp get apimart
claude mcp list
```

进入 Claude Code 后运行 `/mcp`，确认服务器为 Connected，再执行第 3.4 节的只读测试。命令显示 `Added` 只代表配置已保存，最终以 `claude mcp list` 和会话内 `/mcp` 为准。

Claude Code 的用户级 Skill 目录通常为：

```text
~/.claude/skills/apimart-generate-media/
```

如果不使用安装器，应复制仓库中的完整 `skills/apimart-generate-media/` 目录，不能只复制 `SKILL.md`，因为 Skill 还包含引用文档和脚本。

## 8. 其他 MCP 客户端接入原则

客户端同时满足以下条件即可接入：

- 支持 Streamable HTTP MCP。
- 支持 `Authorization: Bearer ...` 请求头。
- 能从环境变量或安全凭据存储读取 Key。

通用配置：

| 配置项 | 值 |
| --- | --- |
| 名称 | `apimart` |
| 传输方式 | Streamable HTTP |
| URL | `https://mcp.apimart.ai/mcp` |
| Header | `Authorization: Bearer <用户自己的 APIMart API Key>` |

如果客户端不支持 Skill，仍可以直接调用 MCP，但应遵循：

```text
list_models（未指定模型时）
→ get_model_docs（每次生成前）
→ get_model_schema（必要时确认操作类型）
→ upload_image（仅有本地参考图片时上传一次）
→ generate_image / generate_video（只提交一次）
→ get_task（按建议间隔轮询）
```

图片可以通过 `upload_image` 上传。音频和视频目前没有上传工具，只能把公开 HTTP(S) URL 放入模型文档指定的字段。不能直接传本地音频/视频文件、base64、data URI、`file://` 或本地路径。

ChatGPT 网页版不会读取用户电脑上的 `~/.codex/config.toml`。本教程主要面向 Codex、Cursor、Claude Code 等能够配置本地远程 MCP 的客户端。

## 9. 使用 Apipost 独立检查 MCP

Apipost 可以排查 MCP 服务是否正常，但不会自动执行 Skill 工作流。当前 APIMart MCP 使用无会话 HTTP 模式，每个 POST 可以独立执行。

### 9.1 请求设置

```http
POST https://mcp.apimart.ai/mcp
Authorization: Bearer <用户自己的 APIMart API Key>
Content-Type: application/json
Accept: application/json, text/event-stream
```

请使用 Apipost 的私密环境变量或 Secret 保存 Key，不要把它直接写进共享接口文档。响应可能使用 SSE；此时读取 `data:` 后面的 JSON-RPC 内容。

### 9.2 查看 7 个工具

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

检查 `result.tools`，应包含第 1 节列出的 7 个工具。

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

`stale: false` 表示缓存有效或刚完成刷新；`stale: true` 表示刷新失败后返回了最近一次成功内容。

### 9.4 上传一张测试图片（不调用计费生成）

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

正常结果包含 `url`、`filename`、`content_type`、`bytes` 和 `created_at`。上传会在对象存储创建图片，但不会调用计费生成接口，因此不要反复执行。上传 URL 有效期为 72 小时。

## 10. 正式环境验收流程

### 10.1 第一阶段：只读验收

依次确认：

- Skill 能被客户端识别。
- MCP `apimart` 显示已连接。
- `tools/list` 有 7 个工具。
- `list_models` 返回至少一个模型。
- `get_model_docs` 返回目标模型的 Markdown。
- `get_model_schema` 能识别正确的 `image_generation` 或 `video_generation`。

推荐提示词：

```text
使用 APIMart Image & Video Skill，调用生产 apimart MCP：
列出 5 个图片或视频模型，再读取 Omni-Flash-Ext 的实时文档和兼容性 Schema。
不要提交任何生成任务。
```

`total: 0` 或 `models: []` 表示当前 Key 没有可用模型，不能继续计费验收。

### 10.2 图片上传验收（不计生成费）

在支持附件的客户端上传一张 JPEG、PNG、GIF 或 WebP，然后发送：

```text
使用 APIMart Image & Video Skill，只把我附加的图片上传成可访问 URL。
只上传一次，返回 URL、文件类型和字节数；不要生成内容。
```

验收点：

- 只上传一次，不调用生成工具。
- 返回的 HTTP(S) URL 可以访问。
- 不会把用户电脑上的本地路径直接传给远程 Pod。
- Skill/MCP 不自行设置图片大小业务限制，最终以上传接口响应为准。
- URL 有效期为 72 小时。

音频或视频附件应被拒绝并要求用户提供公开 HTTP(S) URL。

### 10.3 生成前检查（不提交）

先发送：

```text
使用 APIMart Image & Video Skill，只做生成前检查：
读取候选模型的实时文档，展示准确模型 ID、完整请求参数和能确认的费用信息；
查不到价格就明确说未知。不要调用 generate_image 或 generate_video，等待我确认。
```

确认模型、参数和预算后，再单独发送生成确认。

### 10.4 单独确认生成（会计费）

```text
我明确确认按上一轮展示的模型和参数生成 1 个结果。
首次提交前创建并记录新的幂等键，只调用对应的 generate_image 或 generate_video 一次。
返回 task_id 后，只在 should_poll 为 true 时按 next_poll_after_seconds 查询；
最多查询 10 分钟或 120 次，失败时不要换模型或重新创建任务。
```

幂等规则：

- 幂等键不是 API Key，不是秘密。
- 新的逻辑生成请求使用新键。
- 同一次请求超时或断线恢复时，必须复用相同幂等键、模型和完整输入。
- 已拿到 `task_id` 时只使用 `get_task`，不要再次调用生成工具。
- 参数变化属于新请求，必须重新确认并使用新键。
- 结果不确定时不能换新键重试，否则可能重复计费。

生成完成后应返回媒体 URL；链接有效期为 72 小时，请及时下载保存。

## 11. 更新、卸载与从 dev 切换到 main

### 11.1 更新正式 Skill

更新前先退出 AI 客户端并清除当前终端中的 Key。macOS/Linux：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 update apimart-generate-media -g -y
```

Windows 先在 PowerShell 清除当前进程和用户级变量，再到 CMD 执行：

```cmd
npx --yes skills@1.5.21 update apimart-generate-media -g -y
```

更新后检查锁文件中的来源分支。macOS/Linux：

```bash
grep -A 8 '"apimart-generate-media"' ~/.agents/.skill-lock.json
```

Windows PowerShell：

```powershell
$lock = Get-Content -Raw "$HOME\.agents\.skill-lock.json" | ConvertFrom-Json
$lock.skills.'apimart-generate-media'.ref
Remove-Variable lock
```

正式安装应显示 `"ref": "main"`。重新设置 Key 并重启客户端后，执行第 10.1 节只读验收。

### 11.2 卸载 Skill

先清除 Key，再执行：

```text
npx --yes skills@1.5.21 remove apimart-generate-media -g --agent "*" -y
```

### 11.3 卸载 Codex MCP

```bash
codex mcp remove apimart
```

如果 macOS 只能使用桌面端自带 CLI：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp remove apimart
```

Windows 没有 `codex` 命令时，打开 `%USERPROFILE%\.codex\config.toml`，只删除 `[mcp_servers.apimart]` 及其 `url`、`bearer_token_env_var` 行，不要误删其他配置段。

最后清除环境变量。macOS：

```bash
launchctl unsetenv APIMART_API_KEY
```

Windows PowerShell：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

### 11.4 从 dev 切换到 main

1. 按第 2.3 节卸载共享的 dev Skill。
2. 按对应客户端章节安装 `tree/main`。
3. 删除 MCP 配置 `apimart-dev`；如果旧测试配置名称也是 `apimart`，应根据 `.asia` URL 判断并删除。
4. 新增 `apimart`，URL 必须是 `https://mcp.apimart.ai/mcp`。
5. 使用 APIMart 正式平台生成的 Key。
6. 完全重启客户端并重新做只读验收。

不能只把 MCP 显示名称从 `apimart-dev` 改为 `apimart`；真正决定环境的是 URL 和 Key。

## 12. 常见问题

### 12.1 `codex: command not found`

macOS Codex 桌面端尝试：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp list
```

Windows 直接按第 5.3 节方式 B 编辑 `%USERPROFILE%\.codex\config.toml`。远程 MCP 必须配置 `url`，不能创建空的 `command = ""`。

### 12.2 已配置 MCP，但提示缺少环境变量

- `bearer_token_env_var` 必须填写 `APIMART_API_KEY`。
- 错误中如果出现 `Environment variable sk-... is not set`，表示真实 Key 被误写成环境变量名；应立即作废暴露的 Key。
- 完全退出所有客户端与旧终端，重新设置变量后再启动。
- 不要把真实 Key 发到聊天中检查。

### 12.3 返回 401 或 403

- Key 不存在、已过期、被禁用或权限不足。
- 确认 Key 来自 APIMart 正式平台，而不是测试环境。
- 更新 Key 后必须重启客户端，让新进程重新读取环境变量。

### 12.4 返回 `total: 0` 或 `models: []`

这表示 MCP 已经连通，但当前 Key 没有可用模型。检查账号、分组或模型权限；不要继续生成，也不要把空列表误认为 Skill 安装失败。

### 12.5 工具少于 7 个

- 客户端可能仍缓存旧工具列表。
- 完全重启客户端并新建任务。
- 确认 URL 为 `https://mcp.apimart.ai/mcp`。
- 仍异常时让管理员确认生产 MCP Pod 与镜像版本。

### 12.6 模型参数看起来比文档多

AI 可能把兼容性 Schema 的通用字段当成了模型参数。明确要求：

```text
以 get_model_docs 返回的 Markdown 为真实参数来源；
get_model_schema 只用于确认操作和传输契约。
```

### 12.7 文档返回 `stale: true`

表示上游文档刷新失败，服务返回了最近一次成功副本。通常仍可使用；如果正在验证刚更新的参数，应等待刷新恢复或让管理员检查模型管理中的开发文档链接。

### 12.8 图片上传返回 413

- 如果由 APIMart 上传接口返回，原样反馈，不要自动重试或改扩展名绕过。
- 如果在调用工具前就被客户端、Ingress 或 HTTP 解析器拒绝，这是传输层限制。
- 对本地可读图片可使用 Skill 的 `upload-image --file` 直连上传接口。
- 当前没有音频或视频上传接口，不能通过其他格式绕过。

### 12.9 是否需要 OAuth 或 `APIMART_BASE_URL`

- 生产 MCP 使用用户 APIMart API Key 透传，不需要 OAuth 登录。
- MCP 客户端只配置生产 MCP URL 和 `APIMART_API_KEY`。
- `APIMART_BASE_URL` 只属于本地 API 备用模式，不属于 MCP 配置。
- 不要把它设置为 `https://mcp.apimart.ai/mcp`。

### 12.10 图片或视频地址打不开

- 上传和生成结果 URL 有效期为 72 小时。
- 确认任务状态为 `completed`。
- 尝试直接打开原始 URL。
- 保留任务 ID 交给 APIMart 排查，不要先重复提交计费任务。

### 12.11 Windows CMD 与 PowerShell 命令报错

- `C:\Users\...>` 是 CMD，不能执行 `$key = ...` 或 `Remove-Item`。
- `PS C:\Users\...>` 是 PowerShell。
- PowerShell 找不到 `npx`、但 CMD 能用时，在 CMD 安装 Skill，在 PowerShell 处理 Key 与配置。
- `finally` 不能在 `try/catch` 已结束后单独执行；普通临时变量可用 `Remove-Variable ... -ErrorAction SilentlyContinue` 清理。

## 13. 安全检查清单

接入完成后逐项确认：

- [ ] 安装的是 `apimart-skills/tree/main`，锁文件中的 `ref` 为 `main`。
- [ ] MCP 名称为 `apimart`，URL 为 `https://mcp.apimart.ai/mcp`。
- [ ] API Key 来自 APIMart 正式平台，且独立、限额、可撤销。
- [ ] API Key 没有出现在聊天、截图、文档、配置文件或 Git 提交中。
- [ ] MCP 配置保存的是环境变量名 `APIMART_API_KEY`，不是裸 Key。
- [ ] Codex 配置包含 `url` 与 `bearer_token_env_var`，没有空的 `command = ""`。
- [ ] MCP 使用 HTTPS，并且已显示 Connected/enabled。
- [ ] 工具数量为 7。
- [ ] `list_models` 至少返回一个模型。
- [ ] 每次生成前读取对应模型的实时文档。
- [ ] 本地参考图片先上传并使用返回 URL。
- [ ] 音频和视频只使用公开 HTTP(S) URL。
- [ ] 生成前经过用户明确确认，没有把只读查询变成计费生成。
- [ ] 首次提交前保存幂等键、模型和完整输入；恢复时三者完全复用。
- [ ] 任务结果 URL 可以打开，并在 72 小时内下载保存。

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
