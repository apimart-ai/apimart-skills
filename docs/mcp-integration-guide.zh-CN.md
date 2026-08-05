# APIMart MCP 独立接入教程

> 适用对象：使用 Codex、Cursor、Claude Code 或其他支持远程 MCP 的客户端。
>
> MCP 地址：`https://mcp.apimart.ai/mcp`
>
> 本方式不需要安装 APIMart Skill，也不需要 Node.js 或 npm。

## 1. 你会得到什么

连接成功后，客户端会看到 7 个工具：

1. `list_models`：列出当前 Key 可用的模型。
2. `get_model_docs`：读取某个模型的实时 Markdown 参数文档。
3. `get_model_schema`：确认图片/视频操作和传输契约。
4. `upload_image`：上传本地参考图片并取得 HTTP(S) URL。
5. `generate_image`：提交图片生成。
6. `generate_video`：提交视频生成。
7. `get_task`：查询异步任务状态。

它们由远程 MCP 服务直接调用 APIMart API，与 Skill 仓库没有依赖关系。

## 2. 接入前准备

- 一枚有效且有模型权限的 APIMart API Key。
- 能访问 `https://mcp.apimart.ai/mcp` 的网络。
- 客户端支持 Streamable HTTP MCP 和 Bearer Token。

安全规则：

- 把真实 Key 保存到名为 `APIMART_API_KEY` 的环境变量或客户端密钥存储中。
- 配置项 `bearer_token_env_var` 填写 `APIMART_API_KEY`，不能填写 `sk-...` 真实 Key。
- 不要把 Key 写入 URL、普通 JSON/TOML 配置、聊天、截图或 Git。
- 如果真实 Key 曾出现在截图或命令中，请立即在 APIMart 平台作废并重新生成。

## 3. Codex 接入（macOS）

### 3.1 设置环境变量

Codex 桌面端需要从 macOS 图形应用环境读取变量：

```zsh
read -s "APIMART_API_KEY?请输入 APIMart API Key: "
echo
launchctl setenv APIMART_API_KEY "$APIMART_API_KEY"
unset APIMART_API_KEY
```

输入时不显示字符是正常现象。不要运行会打印真实 Key 的检查命令。

### 3.2 添加 MCP

如果 `codex` 命令可用：

```bash
codex mcp add apimart \
  --url https://mcp.apimart.ai/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

如果安装的是 macOS Codex 桌面端，但终端提示 `codex: command not found`：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp add apimart \
  --url https://mcp.apimart.ai/mcp \
  --bearer-token-env-var APIMART_API_KEY
```

检查配置：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp list
```

预期看到 `apimart`、正确 URL、`APIMART_API_KEY` 和 `enabled`。这里只显示环境变量名称，不会显示真实 Key。

### 3.3 手工配置（可选）

编辑 `~/.codex/config.toml`，加入：

```toml
[mcp_servers.apimart]
url = "https://mcp.apimart.ai/mcp"
bearer_token_env_var = "APIMART_API_KEY"
```

不要加入 `command = ""`，也不要把真实 Key 写入文件。保存后彻底退出并重新打开 Codex，再新建任务。

## 4. Codex 接入（Windows）

### 4.1 在 PowerShell 保存 Key

先确认窗口提示符以 `PS` 开头。把下面代码原样执行：

```powershell
$secureKey = Read-Host "请输入 APIMart API Key" -AsSecureString
$apiKey = [System.Net.NetworkCredential]::new("", $secureKey).Password
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $apiKey,
    [System.EnvironmentVariableTarget]::User
)
Remove-Variable apiKey, secureKey
```

只有出现输入提示后才粘贴真实 Key。输入过程不显示字符。

只检查是否已写入，不打印 Key：

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

### 4.2 `codex` 命令可用

在 CMD 或 PowerShell 执行一整行：

```text
codex mcp add apimart --url https://mcp.apimart.ai/mcp --bearer-token-env-var APIMART_API_KEY
```

### 4.3 `codex` 命令不可用

在 PowerShell 执行：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex" | Out-Null
notepad "$env:USERPROFILE\.codex\config.toml"
```

在记事本中加入：

```toml
[mcp_servers.apimart]
url = "https://mcp.apimart.ai/mcp"
bearer_token_env_var = "APIMART_API_KEY"
```

注意：

- 文件名必须是 `config.toml`，不能是 `config.toml.txt`。
- 如果已有同名配置段，只保留一份。
- 不要配置空的 `command = ""`。
- `bearer_token_env_var` 的值必须是变量名，不能是 `sk-...`。

保存后退出所有 Codex 窗口和旧终端，重新打开终端并启动 Codex，再新建任务。

## 5. Cursor 接入

设置 `APIMART_API_KEY` 后，在项目的 `.cursor/mcp.json` 或用户级 `~/.cursor/mcp.json` 中合并：

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

`${env:APIMART_API_KEY}` 是环境变量引用，不能替换成真实 Key。项目级配置不要提交到 Git。

完全重启 Cursor，在 MCP/Tools 设置中确认 `apimart` 已连接。日志分享前必须遮盖 `Authorization`、`Bearer` 和所有 `sk-` 开头内容。

## 6. Claude Code 接入

在已设置 `APIMART_API_KEY` 的终端中进入项目，然后执行：

```bash
claude mcp add-json --scope local apimart '{"type":"http","url":"https://mcp.apimart.ai/mcp","headers":{"Authorization":"Bearer ${APIMART_API_KEY}"}}'
```

检查：

```bash
claude mcp get apimart
claude mcp list
```

进入 Claude Code 后运行 `/mcp`，确认状态为 Connected。

## 7. 其他客户端的通用配置

| 配置项 | 值 |
| --- | --- |
| 名称 | `apimart` |
| 传输方式 | Streamable HTTP |
| URL | `https://mcp.apimart.ai/mcp` |
| Header | `Authorization: Bearer <用户自己的 APIMart API Key>` |

优先使用客户端的 Secret/环境变量能力，不要把裸 Key 保存在普通配置文件中。

## 8. 只读验收

重启客户端并新建任务，发送：

```text
直接使用 apimart MCP：确认工具总数和名称，列出 5 个可用模型，
再读取 Omni-Flash-Ext 的实时模型文档。不要上传文件，不要生成内容。
```

通过标准：

- 工具总数为 7。
- `list_models` 返回至少一个模型。
- `get_model_docs` 返回目标模型的 Markdown。
- 没有调用 `upload_image`、`generate_image` 或 `generate_video`。

`total: 0` 或 `models: []` 说明连接和认证已完成，但当前 Key 没有可用模型权限。

## 9. 使用 Apipost 排查

请求：

```http
POST https://mcp.apimart.ai/mcp
Authorization: Bearer <APIMart API Key>
Content-Type: application/json
Accept: application/json, text/event-stream
```

查询工具：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

查询模型文档：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_model_docs",
    "arguments": { "model": "Omni-Flash-Ext" }
  }
}
```

响应可能使用 SSE；读取 `data:` 后面的 JSON-RPC 数据。Key 应保存为 Apipost 私密环境变量，不要写进共享接口文档。

## 10. 生成时的安全顺序

```text
list_models（未指定模型时）
→ get_model_docs（每次生成前）
→ get_model_schema（必要时确认操作）
→ upload_image（需要本地参考图片时）
→ generate_image / generate_video（明确确认后只提交一次）
→ get_task（按返回建议轮询）
```

- 模型真实参数以 `get_model_docs` 为准，不能把通用 Schema 的所有字段都当成模型参数。
- 图片可通过 `upload_image` 上传；音频和视频目前只能使用公开 HTTP(S) URL。
- 生成属于计费操作。首次提交前创建幂等键；同一次未知结果重试必须复用同一个键、模型和输入。
- 已取得 `task_id` 后只查询任务，不要再次提交生成。
- 返回的媒体链接有效期为 72 小时，请及时下载保存。

## 11. 常见问题

### 提示 `Environment variable sk-... is not set`

真实 Key 被误写成了环境变量名称。立即作废已暴露的 Key，把配置改为 `APIMART_API_KEY`，重新设置新 Key 后重启客户端。

### 提示空 `command` 或 MCP 未就绪

远程 MCP 被误配成本地命令。删除 `command = ""`，保留 `url` 和认证变量配置。

### 返回 401 或 403

Key 无效、已禁用或权限不足。请在 APIMart 控制台检查 Key 和模型权限。

### 工具少于 7 个

完全重启客户端并新建任务；确认 URL 正确。仍异常时联系 APIMart 支持。

### 图片上传返回 413

原样反馈服务响应，不要自动重试或改扩展名绕过。客户端或网络入口也可能在工具执行前限制请求大小。

## 12. 卸载

Codex：

```bash
codex mcp remove apimart
```

macOS 桌面端自带 CLI：

```bash
/Applications/ChatGPT.app/Contents/Resources/codex mcp remove apimart
launchctl unsetenv APIMART_API_KEY
```

Windows 没有 `codex` 命令时，打开 `%USERPROFILE%\.codex\config.toml`，只删除 `[mcp_servers.apimart]` 配置段。清除用户级 Key：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```
