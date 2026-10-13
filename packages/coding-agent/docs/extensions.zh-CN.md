# 扩展

扩展（extension）是为 Pi 添加可执行行为的 TypeScript 模块。当工作流需要的是工具、命令、事件处理器、模型提供方、会话（session）状态或终端 UI，而不仅仅是指令时，就应使用扩展。

扩展在 Pi 进程内运行，拥有相同的操作系统权限。它可以查看提示词、工具调用、文件、凭据和会话历史，因此只应从你信任的来源加载扩展。

典型的扩展会添加一个智能体工具、保护路径、确认危险命令、响应会话事件、修改上下文、提供一个命令，或显示持久的状态信息。

<a id="quick-start"></a>
<a id="writing-an-extension"></a>
<a id="create-an-extension"></a>

## 创建并加载扩展

扩展默认导出一个接收 `ExtensionAPI` 的工厂函数。该工厂函数为当前扩展运行时注册各项能力。

创建 `~/.pi/agent/extensions/hello.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("hello", {
    description: "Show a greeting",
    handler: async (name, ctx) => {
      ctx.ui.notify(`Hello, ${name || "world"}!`, "info");
    },
  });
}
```

启动 Pi 并运行 `/hello`。开发期间可以直接加载文件：

```bash
pi --extension ./hello.ts
```

Pi 使用 `jiti`，因此本地 TypeScript 扩展无需单独的编译步骤。分发扩展及其依赖时，请使用 [Pi 包](packages.md)。

<a id="extension-locations"></a>
<a id="available-imports"></a>
<a id="choose-where-it-loads"></a>

## 将扩展添加到 Pi

将扩展放在用户或项目的扩展目录中。Pi 会加载其中的 TypeScript 或 JavaScript 文件，以及包含 `index.ts` 或 `index.js` 入口文件的子目录。

小型扩展使用单个文件，多文件实现则使用目录。将 npm 依赖写在邻近的 `package.json` 中。约定的存放位置见 [Configuration](configuration.md)，额外路径见 [Settings](settings.md#resources)。

重新加载会替换扩展运行时，因此 `await ctx.reload()` 之后的代码不得复用旧运行时中的状态。只有个人扩展和通过命令行显式指定的扩展才能参与 `project_trust` 事件，该事件在项目扩展加载之前运行。

<a id="understand-the-lifecycle"></a>

## 遵守运行时生命周期

工厂函数可以是同步的，也可以是异步的。对于异步工厂函数，Pi 会等待其完成后再继续启动，因此它可以获取配置或注册启动期间所需的提供方。

不要在工厂函数中启动进程、套接字、监视器或定时器，因为某些调用会在不启动会话的情况下加载扩展。
请在 `session_start` 中，或在需要这些资源的命令或工具中启动长期存在的资源。
在幂等的 `session_shutdown` 处理器中关闭会话范围的资源。

一次运行从输入和 `before_agent_start` 开始，经过模型、消息和工具事件，直到 `agent_end`。
之后，自动重试、恢复、压缩（compaction）或排队的工作可能继续进行。
<a id="agent_start--agent_end--agent_before_settle--agent_settled"></a>

`agent_before_settle` 是最后一个可操作的边界：它可以追加条目并请求一次继续执行。
`agent_settled` 是最终事件，仅用于通知；当集成需要知道 Pi 不会再自动继续时使用它。

<a id="extensionapi-methods"></a>

## 选择集成点

| 能力 | 主要 API |
|---|---|
| 观察或修改生命周期行为 | `pi.on()` |
| 添加可由模型调用的操作 | `pi.registerTool()` |
| 添加 `/` 命令 | `pi.registerCommand()` |
| 添加快捷键或 CLI 标志 | `pi.registerShortcut()` 或 `pi.registerFlag()` |
| 发送用户消息或自定义消息 | `pi.sendUserMessage()` 或 `pi.sendMessage()` |
| 持久化不进入上下文的会话数据 | `pi.appendEntry()` |
| 更改活动工具、模型或思考级别 | `pi` 上的会话控制方法 |
| 添加模型提供方 | `pi.registerProvider()` |
| 添加 MCP 服务器 | `pi.registerMcpServer()` |
| 将每个请求路由到某个模型 | [`pi.registerVirtualModel()`](virtual-models.md) |
| 添加终端渲染 | 渲染器注册和 `ctx.ui` |
| 与其他扩展通信 | `pi.events` |

确切的事件、上下文、工具和结果类型，请参考 [`extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts) 中导出的声明。

## 遵循扩展契约

<a id="events"></a>
<a id="work-with-events"></a>

### 事件与并发

处理器按扩展的加载顺序和注册顺序运行。`pi.on()` 返回一个用于取消该注册的函数；这种变更不会影响已在进行中的分发。
有些事件仅作通知；另一些事件会转换数据、替换结果或取消操作。
请使用每个事件声明的结果类型，而不要假定每个返回值都会生效。

事件涵盖资源发现、会话、智能体与消息生命周期、提供方、工具和原始输入。

`before_agent_start` 同时提供当前提示词及其结构化的 `systemPromptOptions`。优先修改提示词片段、所选工具或准则，这样 Pi 可以追加一个转录（transcript）增量。返回 `systemPrompt` 或设置 `forceSystemPrompt` 会在该次运行中替换整个提示词，而转录仍会继续记录结构化的片段。提供方会收到强制指定的文本，作为其开头的系统提示词。

`message_end` 可以在保留角色的前提下替换已定稿的消息。`tool_call` 可以修改输入或阻止执行。`tool_result` 处理器可以组合使用，每个处理器都能看到之前处理器所做的更改。

<a id="provider_stream_event"></a>

`provider_stream_event` 会在 Pi 规范化之前，针对每个已解析的提供方流事件触发。该事件标识提供方、API 和模型；`event.data` 是 Pi 能获得的最早的结构化值，不一定是原始的 HTTP 字节或 SSE 帧。请将其视为只读，因为修改它可能影响规范化过程。该事件仅用于通知，且不会被持久化。

处理器按流的顺序被等待执行，因此处理缓慢的处理器会延迟流的消费。处理器错误会被报告，但不会改变提供方的响应。可选启用的查看器示例见 [`debug-provider.ts`](../examples/extensions/debug-provider.ts)，它按助手消息对原始事件进行分组。

<a id="context_with_system"></a>

`context` 在不含提示词和工具系统消息的情况下转换对话消息；Pi 之后会恢复这部分状态。只有当某个仅作用于当前请求的转换必须掌控完整转录时，才使用 `context_with_system`，并且要在索引零处保留一条系统消息。

`turn_end` 和 `agent_before_settle` 是可操作的边界。它们的处理器可以串联提议的 `custom`、`custom_message`、`context_edit` 或 `compaction` 条目，并返回 `continue: true` 以发起下一次模型请求。请为继续执行的条件加上保护，因为无条件的继续执行可能导致循环。完整的校验和排序契约请参考导出的事件声明。

<a id="cache_warming_decision"></a>

`cache_warming_decision` 可以用 `{ action: "warm" }` 或 `{ action: "stop" }` 覆盖空闲时的提示词缓存刷新。最后一个返回 action 的处理器生效。

同一条助手消息中的多个工具调用可能并行运行。
当某个工具事件运行时，不要假定同级的调用或结果已经存在。
对于由活动轮次所拥有的嵌套工作，请使用 `ctx.signal`；命令和空闲时的会话事件通常没有操作信号。

返回 `undefined` 的 `user_bash` 处理器会将命令传递给下一个处理器；如果没有处理器处理它，则交给本地执行。返回 `operations` 或 `result` 会停止传播。处理器失败会阻止该命令，而不是回退到本地执行。

<a id="custom-tools"></a>
<a id="register-tools"></a>

### 工具

自定义工具定义名称、面向模型的描述、TypeBox 参数模式（schema）和 `execute()` 函数。
其结果需要包含面向模型的 `content`，以及用于渲染或状态重建的 `details` 字段。
没有结构化详情时使用 `details: undefined`。如果工具进行了嵌套的模型调用，请在结果中包含它们的 `usage`，以确保会话总计保持准确。

从 `execute()` 中抛出异常会产生失败的工具结果。
返回一个对象并不会将其标记为错误。
仅当该批次中每个已完成的工具都同意终止、且智能体应当跳过其自动后续操作时，才返回 `terminate: true`。

当多个工具共享可变的内存状态时，请使用顺序执行。
修改文件的工具应使用 `withFileMutationQueue()` 包裹完整的读取-修改-写入操作。
截断较大的面向模型的结果，并告诉模型在哪里读取完整输出。

当结果是数据时，声明 `outputSchema` 并返回与之匹配的 `structuredContent`。模型仍会收到 `content`；而 codemode 脚本等程序化调用方收到的是 `structuredContent` 而不是文本。没有 `outputSchema` 的工具会以其文本内容传递给脚本。若要报告一个仍携带数据的失败，请返回带有 `isError: true` 的结果，而不是抛出异常：模型会看到错误，而脚本仍会收到 `structuredContent`。

工具可以通过 `ctx.executeTool(name, args, { signal, onUpdate })` 运行其他工具。嵌套调用与模型发起的调用一样，会经过参数校验以及 `tool_call` 和 `tool_result` 处理器，并触发 `tool_execution_start`、`tool_execution_update` 和 `tool_execution_end`；所有这些事件都携带 `parentToolCallId`，其 `toolCallId` 由 pi 分配为 `<parent id>/<n>`。这些 id 不会作为工具调用或工具结果出现在转录中。嵌套调用不会添加转录条目：它们的结果只会到达发起调用的工具，由该工具自行报告，例如通过 `onUpdate` 和 `details`。会话会在发起调用的工具的结果消息上，以 `nestedCalls` 的形式保留这些调用的有限记录（名称、参数、状态、耗时、错误；从不包含结果）。该记录用于压缩时的文件列表，并显示在 HTML 导出中。单次调用超过 8 KiB 或单个工具结果超过 32 KiB 的参数会被省略，最多保留 256 次调用，`complete: false` 表示该记录有内容丢失。各层级嵌套结果的 `usage` 都会累加到发起调用的工具的结果 `usage` 中，因此工具只需报告自身的用量，而不必报告其调用的工具的用量。`ctx.tools` 列出 `ctx.executeTool()` 可以调用的工具。对 `content` 进行脱敏的 `tool_result` 处理器也应替换 `structuredContent`；只替换 `content` 会导致其被丢弃。

参见 [`hello.ts`](../examples/extensions/hello.ts)、[`todo.ts`](../examples/extensions/todo.ts)、[`dynamic-tools.ts`](../examples/extensions/dynamic-tools.ts) 和 [`truncated-tool.ts`](../examples/extensions/truncated-tool.ts)。

### 工具暴露方式

`exposure` 控制模型如何访问某个工具。“可调用”是指可以通过 `ctx.executeTool()`（`ctx.tools`）从其他工具中调用，就像 `codemode` 工具的脚本那样：

- `direct`（默认）：处于活动状态时向模型声明，并且在活动时可调用。
- `model-only`：处于活动状态时向模型声明，但从不可调用。适用于编排其他工具或向用户提问的工具。
- `codemode`：只要已注册即可调用，并由 `codemode` 工具列出。除非被显式激活，否则不会向模型声明。
- `deferred`：与 `codemode` 类似，但 codemode 工具不会列出它；`tool_search` 可以找到并激活它。
- `hidden`：已注册但无法访问。由于工具无法取消注册，可以用 `exposure: "hidden"` 重新注册工具来将其撤回。

`namespace: { name, description, instructions }` 将相关工具分组，就像 MCP 服务器所做的那样。Codemode 工具会将一个命名空间列在一个标题下，并附上其 `description`。`instructions` 存放较长的使用说明；它不会被列出，codemode 脚本通过 `describeNamespace(name)` 读取它。

注册 `direct` 或 `model-only` 工具会将其激活；其他暴露方式在注册时不会被激活。活动集合（`pi.getActiveTools()`、`pi.setActiveTools()`）就是向模型声明的工具集合。`pi.getAllTools()` 会报告每个工具的 `exposure`、`namespace` 和 `annotations`。

`annotations` 是关于工具行为的提示，其含义与 MCP 工具注解相同：`readOnlyHint`、`destructiveHint`、`idempotentHint` 和 `openWorldHint`。MCP 工具携带其服务器声明的提示。缺失的提示采用 MCP 默认值：工具不是只读的，可能具有破坏性，并且可能访问开放世界。这些提示不会被验证，但权限扩展可以用它们来决定需要确认哪些调用。下面的代码会确认 Codex 需要请求批准的那些调用：

```typescript
pi.on("tool_call", async (event, ctx) => {
  const hints = pi.getAllTools().find((tool) => tool.name === event.toolName)?.annotations;
  const needsApproval =
    hints?.destructiveHint === true ||
    (!hints?.readOnlyHint && ((hints?.destructiveHint ?? true) || (hints?.openWorldHint ?? true)));
  if (needsApproval && !(await ctx.ui.confirm("Allow tool call?", event.toolName))) {
    return { block: true, reason: `${event.toolName} was not approved` };
  }
});
```

编排其他工具的工具可以在其处于活动状态时，通过 `prepareLoadout(loadout)` 调整模型所看到的内容。每当活动工具发生变化时它都会运行，并接收已声明的工具、可调用的工具，以及每个已注册的工具及其暴露方式、命名空间和提示词准则。它返回已声明工具（包括它自己）的替换 `descriptions`，以及 `hiddenDeclarations`：这些活动工具的声明会从请求中省略，但它们仍保持活动且可调用。默认系统提示词会在其工具列表和规则中省略隐藏的工具，并且当文件读取工具被隐藏时，不会在技能提示中提及任何工具，因此编排工具应自行呈现这些工具的准则。`codemode` 仅使用此钩子、`exposure` 和 `ctx.executeTool()`，因此其他工具可以用不同的名称实现相同的行为。

### 动态激活工具

先注册所有工具，让可选工具保持非活动状态，然后在一个加载器工具中使用 `pi.setActiveTools()` 选择所需的活动工具。名称必须已经注册；未知名称会被忽略。

Pi 会在转录的第一条系统消息中记录初始提示词和工具集，然后在下一次模型请求之前追加工具和提示词的变更。对于无法表示这种转换的提供方，会收到一个完整的转录检查点，这可能使已缓存的前缀失效。

### 工具渲染

工具的 `renderCall` 和 `renderResult` 负责在交互式转录和 HTML 导出中绘制其调用。`pi.registerToolRenderer((toolName, next) => renderers)` 为任意工具的调用选择渲染器，包括尚未注册的工具，例如在恢复的会话中、其服务器连接之前的 MCP 工具。`next()` 返回其余解析器（按扩展加载顺序）以及已注册工具将要使用的渲染器，因此 `next() ?? mine` 只起补充作用。

### MCP 服务器

`pi.registerMcpServer(name, config)` 为当前会话添加一个 MCP 服务器。`config` 的结构与 [`mcp.json`](mcp.md) 中的 `mcpServers` 条目相同：stdio 服务器使用 `command`、`args`、`env` 和 `cwd`，HTTP 服务器使用 `url`、`headers` 和 `oauth`，另外还有 `exposure`、`toolExposure`、`description`、`enabled` 和 `timeout`。

```typescript
pi.registerMcpServer("jira", { url: "https://mcp.example.com/jira", exposure: "codemode" });
pi.unregisterMcpServer("jira");
```

在扩展加载期间注册的服务器会在会话启动时与 `mcp.json` 中的服务器一起连接；之后注册的服务器会立即连接，而 `pi.unregisterMcpServer()` 会关闭连接，并使该服务器的工具无法访问。注册不会被保存：每次加载时都需要重新注册，例如根据扩展自身的设置进行注册。`mcp.json` 中同名的服务器优先，`/mcp` 会显示这种覆盖。再次注册相同名称会替换该扩展之前的注册；注册由其他扩展注册的名称、无效名称或无效配置都会抛出异常。

内置的 MCP 支持会连接已注册的服务器。如果没有任何组件进行连接（因为另一个扩展替换了内置支持，参见 [MCP](mcp.md#other-mcp-extensions)），每一项注册都会被报告为扩展错误。其他 MCP 扩展也可以连接已注册的服务器：在 `session_start` 时用 `pi.getMcpServers()` 读取它们，并处理 `mcp_servers_change` 事件以获取后续变更。

<a id="extensioncontext"></a>
<a id="extensioncommandcontext"></a>
<a id="use-extension-context"></a>

### 上下文与会话变更

`ExtensionContext` 提供工作目录、模式、UI、会话管理器、模型运行时、中止信号、上下文用量，以及用于压缩和关闭的控制。
使用 `ctx.modelRegistry.streamSimple()` 进行与提供方无关的嵌套模型调用。

命令处理器接收 `ExtensionCommandContext`，它增加了等待空闲、重新加载、树导航和会话替换等操作。
这些操作仅限命令使用，因为从生命周期处理器中调用它们可能导致运行时死锁。

会话替换会使旧的上下文失效。切换前只捕获纯数据，然后使用提供给 `withSession` 的新上下文执行与会话绑定的工作。

<a id="state-management"></a>
<a id="persist-state"></a>

### 状态

根据状态参与对话的方式选择存储方式：

| 状态 | 存储方式 |
|---|---|
| 跟随活动分支的工具状态 | 工具结果的 `details` |
| 不进入模型上下文的持久数据 | `pi.appendEntry()` |
| 存储并发送给模型的自定义内容 | `pi.sendMessage()` |
| 单个会话之外的数据 | 外部存储 |

在 `session_start` 期间，从 `ctx.sessionManager.getBranch()` 重建与分支相关的状态。
不要从文件中的每个条目重建它，因为被放弃的分支代表的是另一种历史。
当自定义存储的内容需要出现在转录中时，请注册条目或消息渲染器。

<a id="custom-ui"></a>
<a id="mode-behavior"></a>
<a id="interact-with-the-user"></a>
<a id="account-for-each-mode"></a>

### UI 与模式

`ctx.ui` 提供对话框、通知、状态文本、小组件、标题、编辑器访问和自定义组件。
仅当交互需要自己的渲染和输入时，才使用 `ctx.ui.custom()`。
关于组件、焦点、浮层、主题和性能的指导，参见 [Terminal UI](tui.md)。

扩展会在交互、RPC、JSON 和 print 模式下加载。
交互模式提供完整的终端 UI。
RPC 可以通过 [RPC Extension UI protocol](rpc-extension-ui.md) 转发受支持的对话框和通知，但不支持自定义终端组件；JSON 和 print 模式没有 UI。
用 `ctx.mode === "tui"` 保护仅限终端的行为，并对交互客户端和 RPC 客户端都支持的交互使用 `ctx.hasUI`。

让工具和事件行为独立于渲染，以便非交互模式仍能正常工作。

<a id="error-handling"></a>
<a id="handle-errors-and-shutdown"></a>

### 错误与清理

Pi 会报告处理器错误，并在可能的情况下继续运行。`tool_call` 处理器失败会作为故障保护措施阻止该工具；工具执行失败会成为提供给模型的错误结果。

即使正常运行时已尝试过清理，也要在 `session_shutdown` 中释放资源。
保持清理操作幂等，因为取消、重新加载、会话替换和进程退出可能汇聚到同一条路径上。
使用 `ctx.shutdown()` 请求有序地关闭进程。

<a id="examples-reference"></a>
<a id="use-examples-as-the-implementation-reference"></a>

## 示例与参考

经过检查的[扩展示例](../examples/extensions/)涵盖工具、生命周期事件、命令、标志、快捷键、状态、渲染、提供方、OAuth、远程执行和终端组件。
请从与你的集成点相匹配的最小示例开始。

模型服务集成请参考 [Custom Providers](custom-provider.md)，自定义组件请参考 [Terminal UI](tui.md)，与其他资源一起安装或分发扩展请参考 [Pi Packages](packages.md)。
