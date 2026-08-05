# APIMart Skill 独立接入教程

> Skill 来源：`https://github.com/apimart-ai/apimart-skills`
>
> Skill 名称：`apimart-generate-media`
>
> 本方式由 Skill 自带本地脚本直接调用 APIMart API，不需要配置远程 MCP。

## 1. 你会得到什么

安装完整 Skill 目录后，AI 助手可以通过自带脚本：

- 查询当前 Key 可用的图片和视频模型。
- 获取每个模型实时 Markdown 文档及兼容性 Schema。
- 上传本地参考图片并取得 HTTP(S) URL。
- 提交图片或视频生成。
- 查询异步任务直至完成或失败。
- 使用幂等键安全恢复不确定的生成请求。

Skill 不会查找或调用 MCP 工具。它只运行：

```text
node <Skill目录>/scripts/apimart-media.mjs <命令>
```

## 2. 接入前准备

- Node.js 20 或更高版本。
- 一枚有效且有模型权限的 APIMart API Key。
- AI 客户端支持 Agent Skills，并允许 Skill 运行本地 `node` 命令。
- 能访问 `https://api.apimart.ai`。

检查：

```bash
node -v
npm -v
```

第三方 `skills@1.5.21` 安装器需要 Node.js 22.20 或更高版本。若只手工复制完整 Skill 目录，运行脚本只需要 Node.js 20+。

安全规则：

- 先安装并检查 Skill 来源，再设置 API Key，避免不可信安装脚本继承 Key。
- 不要把真实 Key 发到聊天、截图、命令参数、JSON 文件或 Git。
- Skill 从 `APIMART_API_KEY` 环境变量读取 Key；`API_KEY` 仅作为旧兼容变量。
- 正常生产使用无需设置 `APIMART_BASE_URL`，默认值是 `https://api.apimart.ai`。

## 3. 安装 Skill

### 3.1 Codex

macOS/Linux：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills \
  -g \
  --agent codex \
  --skill apimart-generate-media \
  -y
```

Windows CMD（复制为一整行）：

```cmd
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills -g --agent codex --skill apimart-generate-media -y
```

### 3.2 Cursor

macOS/Linux：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills \
  -g \
  --agent cursor \
  --skill apimart-generate-media \
  -y
```

Windows CMD：

```cmd
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills -g --agent cursor --skill apimart-generate-media -y
```

### 3.3 Claude Code

macOS/Linux：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills \
  -g \
  --agent claude-code \
  --skill apimart-generate-media \
  -y
```

Windows CMD：

```cmd
npx --yes skills@1.5.21 add https://github.com/apimart-ai/apimart-skills -g --agent claude-code --skill apimart-generate-media -y
```

### 3.4 检查安装

```bash
npx --yes skills@1.5.21 list -g --json
```

结果中应出现 `apimart-generate-media`。安装器的常见全局目录是：

```text
~/.agents/skills/apimart-generate-media/
```

目录内至少应有：

```text
SKILL.md
agents/openai.yaml
references/api-contract.md
scripts/apimart-media.mjs
```

不能只复制 `SKILL.md`，否则本地脚本缺失，Skill 无法独立工作。

## 4. 设置 API Key

### 4.1 macOS Codex 桌面端

```zsh
read -s "APIMART_API_KEY?请输入 APIMart API Key: "
echo
launchctl setenv APIMART_API_KEY "$APIMART_API_KEY"
unset APIMART_API_KEY
```

彻底退出并重新打开 Codex，再新建任务。输入时不显示字符是正常现象。

### 4.2 macOS/Linux 终端客户端

zsh：

```zsh
read -s "APIMART_API_KEY?请输入 APIMart API Key: "
echo
export APIMART_API_KEY
```

bash：

```bash
read -rsp "请输入 APIMart API Key: " APIMART_API_KEY
echo
export APIMART_API_KEY
```

从同一个终端启动 AI 客户端。关闭终端后变量消失；结束时也可执行 `unset APIMART_API_KEY`。

### 4.3 Windows PowerShell

确认提示符以 `PS` 开头：

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

关闭所有 AI 客户端和旧终端，再重新打开客户端。下面命令只检查格式，不打印真实 Key：

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

Windows 中 `C:\Users\...>` 是 CMD，不能执行 `$key = ...`；`PS C:\Users\...>` 才是 PowerShell。PowerShell 找不到 `npx`、但 CMD 能使用时，在 CMD 安装 Skill，在 PowerShell设置 Key。

## 5. 独立验证

先做不会生成、不会计费的检查。

### 5.1 直接检查本地脚本

macOS/Linux：

```bash
SKILL_DIR="$HOME/.agents/skills/apimart-generate-media"
node "$SKILL_DIR/scripts/apimart-media.mjs" --help
node "$SKILL_DIR/scripts/apimart-media.mjs" models --limit 5
node "$SKILL_DIR/scripts/apimart-media.mjs" docs --model "Omni-Flash-Ext"
```

Windows PowerShell：

```powershell
$skillDir = "$HOME\.agents\skills\apimart-generate-media"
node "$skillDir\scripts\apimart-media.mjs" --help
node "$skillDir\scripts\apimart-media.mjs" models --limit 5
node "$skillDir\scripts\apimart-media.mjs" docs --model "Omni-Flash-Ext"
Remove-Variable skillDir
```

通过标准：

- `--help` 显示 `models`、`docs`、`schema`、`upload-image`、`generate-image`、`generate-video` 和 `task`。
- `models --limit 5` 至少返回一个模型。
- `docs` 返回目标模型的 Markdown 文档。
- 没有提交任何生成任务。

### 5.2 在 AI 客户端验证

重启客户端并新建任务，发送：

```text
使用 APIMart Image & Video Skill，通过 Skill 自带本地客户端列出 5 个
图片或视频模型，再读取 Omni-Flash-Ext 的实时模型文档。不要上传文件，
不要生成内容。
```

如果 AI 只解释操作但没有执行，确认客户端允许运行本地命令，并且安装的是完整 Skill 目录。

## 6. 正确使用流程

```text
models（未指定模型时）
→ docs（每次生成前读取该模型真实参数）
→ schema（必要时确认图片/视频操作）
→ upload-image（有本地参考图片时）
→ key（首次提交前创建幂等键）
→ generate-image / generate-video（明确确认后只提交一次）
→ task（按返回建议轮询）
```

重要规则：

- 模型参数、可选值和组合规则以 `docs` 返回的实时 Markdown 为准。
- `schema` 只用于兼容性确认，不能把通用字段全部当成某个模型都支持的参数。
- 本地图片先用 `upload-image` 上传，再把返回 URL 放入模型文档指定字段。
- 本地音频和视频不能上传，只能使用公开 HTTP(S) URL。
- 查询模型和文档不是生成授权。只有用户明确要求生成时才提交计费请求。
- 同一个逻辑请求的未知结果重试，必须复用同一个幂等键、模型和完整输入。
- 已取得 `task_id` 后只查询任务，不要再次提交生成。
- 上传和生成结果链接有效期为 72 小时，请及时下载保存。

## 7. 本地脚本命令速查

```text
node <Skill目录>/scripts/apimart-media.mjs key
node <Skill目录>/scripts/apimart-media.mjs models [--query 文本] [--limit 1-200]
node <Skill目录>/scripts/apimart-media.mjs docs --model 模型ID
node <Skill目录>/scripts/apimart-media.mjs schema --model 模型ID
node <Skill目录>/scripts/apimart-media.mjs upload-image --file 图片路径
node <Skill目录>/scripts/apimart-media.mjs generate-image --model 模型ID --input-file 请求.json --idempotency-key 幂等键
node <Skill目录>/scripts/apimart-media.mjs generate-video --model 模型ID --input-file 请求.json --idempotency-key 幂等键
node <Skill目录>/scripts/apimart-media.mjs task --task-id 任务ID --language zh
```

生成命令属于计费操作，不要把示例直接当测试执行。

## 8. 常见问题

### `npx` 在 PowerShell 不可用，但 CMD 可用

在 CMD 执行安装、更新和卸载命令；在 PowerShell 设置 Key。不要把两种 shell 的语法混在同一个窗口。

### AI 提示缺少可调用工具

这通常表示安装了旧版 Skill，或者只复制了 `SKILL.md`。更新 Skill，检查 `scripts/apimart-media.mjs` 存在，然后新建任务。独立 Skill 不要求外部工具列表。

### 返回 401 或 403

Key 无效、已禁用或权限不足。更新 Key 后彻底重启客户端，让新进程重新读取环境变量。

### 返回空模型列表

本地客户端已连接 API，但当前 Key 没有可用模型。检查 APIMart 账号、分组和模型权限。

### 图片上传返回 413

Skill 不自行设置图片大小限制。原样反馈上传接口响应，不自动重试或改扩展名绕过。

### 文档返回 `stale: true`

表示文档刷新失败，当前返回最近一次成功副本。一般仍可使用；若正在验证刚更新的参数，请稍后重试或联系 APIMart 支持。

### 结果地址打不开

确认任务状态是 `completed`，并在返回后 72 小时内打开或下载。保留任务 ID 供排查，不要先重复提交计费任务。

## 9. 更新与卸载

更新前先退出 AI 客户端，并确保安装命令所在终端没有真实 Key：

```bash
unset APIMART_API_KEY
npx --yes skills@1.5.21 update apimart-generate-media -g -y
```

Windows 在 CMD 执行：

```cmd
npx --yes skills@1.5.21 update apimart-generate-media -g -y
```

更新后彻底重启客户端并重新执行只读验证。

卸载：

```text
npx --yes skills@1.5.21 remove apimart-generate-media -g --agent "*" -y
```

清除 macOS 图形应用环境变量：

```bash
launchctl unsetenv APIMART_API_KEY
```

清除 Windows 用户级环境变量：

```powershell
Remove-Item Env:APIMART_API_KEY -ErrorAction SilentlyContinue
[Environment]::SetEnvironmentVariable(
    "APIMART_API_KEY",
    $null,
    [System.EnvironmentVariableTarget]::User
)
```

卸载本 Skill 不会影响任何远程工具配置，因为二者相互独立。
