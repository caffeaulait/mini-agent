# Mini Agent

一个使用 TypeScript 从零实现的本地编码 Agent。项目通过 OpenAI 兼容接口驱动模型，在终端 TUI 中提供代码浏览、文件修改、命令执行、会话管理、长期记忆和可扩展技能等能力。

## 项目亮点

- **终端交互界面**：基于 `@earendil-works/pi-tui`，支持流式输出、输入编辑、上下文与用量面板。
- **OpenAI 兼容模型**：通过环境变量配置 API 地址、密钥和模型，默认使用 DeepSeek 兼容接口。
- **工具调用循环**：模型可以按需调用文件、目录、搜索、Shell、网页抓取等工具，并持续执行直到生成最终回答。
- **安全权限控制**：通过 `.miniagent/permissions.json` 为每个工具配置 `allow`、`ask` 或 `deny` 策略；文件操作限制在配置的项目根目录内，写入前展示 diff 并支持确认。
- **任务规划与委派**：内置 `todo_write` 和 `delegate_task`，复杂任务可以拆解、跟踪进度，并串行委派边界清晰的子任务。
- **长期记忆**：可将用户偏好、项目事实和重要决定保存到 `.miniagent/memory.json`，并在后续会话中按关键词检索。
- **多会话管理**：支持创建、切换、列出、保存和恢复多个对话，会话数据保存到 `.miniagent/sessions.json`。
- **技能系统**：通过 `src/skills/*/SKILL.md` 声明技能指令和工具白名单，当前内置代码浏览和代码审查技能。
- **历史压缩与撤销**：上下文过长时自动压缩旧消息；最近一次文件写入或补丁操作可以使用 `/undo` 撤销。

## 技术栈

- TypeScript
- OpenAI SDK
- `@earendil-works/pi-tui`

## 环境要求

- Node.js 22 或更高版本
- 一个 OpenAI 兼容的模型服务及 API Key
- 真实终端环境（程序需要 TTY，不支持管道或重定向输入）

## 安装

```bash
npm install
```

创建 `.env` 并填写模型配置：

```dotenv
OPENAI_API_KEY=your-api-key
# 可选：默认 https://api.deepseek.com
OPENAI_BASE_URL=https://api.deepseek.com
# 可选：默认 deepseek-v4-flash
OPENAI_MODEL=deepseek-v4-flash
```

`OPENAI_BASE_URL` 可以替换为其他支持 OpenAI Chat Completions 接口的服务地址。

## 运行

开发模式（直接运行 TypeScript）：

```bash
npm run dev
```

构建项目：

```bash
npm run build
```

运行构建产物：

```bash
npm start
```

类型检查：

```bash
npm run typecheck
```

当前项目未配置自动化测试，`npm test` 仍是占位脚本。

## 常用命令

在 Agent 输入区可以使用以下命令：

| 命令 | 作用 |
| --- | --- |
| `/help` | 显示帮助 |
| `/compact` | 立即压缩旧对话 |
| `/undo` | 撤销最近一次 `write` / `patch` 写入 |
| `/todos` | 查看当前 TODO |
| `/memory` | 查看长期记忆 |
| `/skills` | 列出可用技能 |
| `/use <name>` | 加载指定技能 |
| `/unuse` | 卸载当前技能 |
| `/save` | 保存全部会话 |
| `/load` | 恢复已保存的会话 |
| `/new <id>` | 新建并切换会话 |
| `/open <id>` | 切换到已有会话 |
| `/sessions` | 列出内存中的会话 |
| `/reset` | 清空当前会话历史，但保留长期记忆 |
| `/exit` | 保存会话并退出 |

## 工具能力

核心工具集中注册在 `src/tools.ts`，包括：

- `get_current_time`：获取本地时间
- `ls`、`glob`、`read`、`search`：浏览和检索项目文件
- `run_shell`：执行 Shell 命令
- `fetch`：抓取外部网页文本
- `write`、`patch`：创建或修改文件
- `todo_write`：维护任务清单
- `delegate_task`：委派子任务
- `memory_write`、`memory_search`：管理长期记忆

默认权限会对命令执行、联网和文件写入进行确认，对只读工具和任务管理工具直接放行。

## 项目结构

```text
src/
├── index.ts          # 程序入口、命令处理和 TUI 事件编排
├── chat.ts           # OpenAI 请求、流式响应、工具调用和历史压缩
├── tools.ts          # 工具注册表及内置工具实现
├── tui.ts            # 终端界面、输入区和状态面板
├── config.ts         # 环境变量配置
├── permissions.ts    # 工作区边界与工具权限
├── sessions.ts       # 多会话管理
├── storage.ts        # 会话持久化
├── memory.ts         # 长期记忆
├── todos.ts          # TODO 和子任务委派
├── skills.ts         # 技能发现、加载和卸载
├── instructions.ts   # 加载项目级 AGENTS.md
├── undo.ts           # 文件修改备份与撤销
└── skills/
    ├── explore/      # 只读代码浏览技能
    └── code-review/  # 代码审查技能及 git_diff 工具
```

## 权限与运行数据

首次启动时，程序会创建 `.miniagent/permissions.json`。其中 `root` 指定 Agent 可以访问的工作区根目录，`tools` 指定各工具的权限策略：

```json
{
  "root": "..",
  "tools": {
    "read": "allow",
    "run_shell": "ask",
    "write": "ask"
  }
}
```

运行过程中产生的权限配置、会话、记忆等数据位于 `.miniagent/`，该目录已加入 `.gitignore`，不会被提交。项目根目录的 `AGENTS.md` 会作为额外项目上下文加载到模型提示词中。

## 开发说明

项目使用 ESM 和 TypeScript，源码位于 `src/`，编译输出位于 `dist/`。新增工具应通过工具注册表接入；新增技能可在 `src/skills/` 下创建目录，并提供 `SKILL.md`，如果需要技能专属工具，则添加导出 `tools: Tool[]` 的 `tools.ts`。

## 许可证

当前 `package.json` 声明的许可证为 ISC。
