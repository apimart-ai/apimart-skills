# APIMart 接入方式导航

APIMart 提供两种**相互独立**的接入方式。任选一种即可完成模型查询、参数读取、图片上传、图片/视频生成和任务查询。

| 方式 | 适合谁 | 是否需要另一种方式 |
| --- | --- | --- |
| [远程 MCP 接入](mcp-integration-guide.zh-CN.md) | 客户端原生支持 Streamable HTTP MCP，希望直接获得 7 个在线工具 | 不需要安装 Skill |
| [本地 Skill 接入](skill-integration-guide.zh-CN.md) | 客户端支持 Agent Skills 和本地命令，希望由 Skill 自带脚本直连 APIMart API | 不需要配置 MCP |

两种方式可以同时安装，但彼此没有调用关系。为了避免重复创建计费任务，同一个生成需求只选择一条路径执行。

共同要求：

- 使用用户自己的 APIMart API Key。
- 不要把真实 Key 发到聊天、截图、配置文件或 Git 仓库。
- 查询模型或参数不会自动授权生成；只有用户明确要求生成时才提交计费请求。
- 上传和生成结果链接有效期为 72 小时，请及时下载保存。
