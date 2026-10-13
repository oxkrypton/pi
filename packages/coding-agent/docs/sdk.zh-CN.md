# SDK

`@earendil-works/pi-coding-agent` 将 Pi 嵌入到 Node.js 或 Bun 进程中。它提供对命令行应用所用的 agent、会话、工具、模型和资源的直接 TypeScript 访问。

在进程内进行 TypeScript 集成时使用 SDK。如需与语言无关或相互隔离的子进程方式，请参阅 [CLI Integration](cli-integration.md)。

```typescript
import { createAgentSession } from "@earendil-works/pi-coding-agent";

const { session } = await createAgentSession();

try {
  await session.prompt("What files are in the current directory?");
  console.log(session.getLastAssistantText());
} finally {
  session.dispose();
}
```

这会使用当前工作目录、已发现的资源、已保存的设置以及已配置的凭据。`prompt()` 在运行（run）结束时 resolve。

[完整的最小示例](../examples/sdk/01-minimal.ts) 还演示了流式输出文本事件。所有 [SDK 示例](../examples/sdk/) 都随仓库一起进行类型检查。

<a id="session-management"></a>

## 会话生命周期

`createAgentSession()` 创建一个 `AgentSession`。一个 session（会话）拥有一段对话、对话的模型和工具、排队的消息、compaction（压缩）状态，以及 extension（扩展）运行时。

通过 `session.messages`、`session.model`、`session.thinkingLevel`、`session.systemPrompt` 和 `session.getActiveToolNames()` 读取当前状态。

`session.systemPrompt` 是只读的，返回当前生效的系统提示词，包括尚未发送给模型的更改。工具变更会在下一次请求之前声明给模型。

<a id="sessionmanager-api"></a>

### 会话存储

会话默认是持久化的。`SessionManager` 拥有持久化的或内存中的条目树，并跟踪其当前活动的叶节点（leaf）。分支操作会改变该叶节点，但不会删除被放弃的分支。当 Pi 重建模型上下文时，管理器会选择当前活动分支并应用压缩。

对于已最终确定的模型上下文，`SessionManager` 是权威来源。要恢复外部历史，请使用包含这些条目的管理器来构造会话。给 `session.agent.state.messages` 赋值并不会替换持久化的上下文。

当宿主不需要会话文件时，使用内存管理器：

```typescript
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
});
```

关于创建、打开、继续、列出和分叉（fork）会话，请参阅经过检查的 [会话示例](../examples/sdk/11-sessions.ts)。[Session File Format](session-format.md) 定义了持久化的 JSONL 约定，[Message Types](message-types.md) 定义了对话记录（transcript）中的值。确切的方法和签名请以导出的 TypeScript 声明或 [`session-manager.ts`](../src/core/session-manager.ts) 为准。

`cwd` 选择用于项目资源发现、上下文文件、会话分组以及内置工具路径的工作区。当目标与 `process.cwd()` 不同时，请显式传入。

`session.dispose()` 会中止正在进行的工作、使扩展上下文失效、与 agent 断开连接，并移除事件监听器。当不再需要该会话时调用它。

`AgentSessionRuntime` 增加了 `newSession()`、`switchSession()`、`fork()` 和 `importFromJsonl()`。每个操作都会替换当前活动的 `AgentSession`，并为目标工作目录重新创建服务。

运行时替换之后，原有订阅属于旧的 `AgentSession`，必须重新绑定。请参阅 [会话运行时示例](../examples/sdk/13-session-runtime.ts)。

## 发送提示

`prompt()` 会处理扩展命令，并在普通用户消息进入 agent 之前展开基于文件的提示词模板。对于被接受的 agent 运行，它会在运行结束（包括自动重试）之后 resolve。

在会话已经处于流式输出状态时发送的提示，必须指定它是引导（steer）当前运行还是在其之后跟进（follow up）。在未做出该选择的情况下调用 `prompt()` 会被拒绝，而不是进行猜测。

引导消息在当前助手轮次及其工具调用之后进入。跟进消息在当前运行完成其待处理工作之后进入。`steer()` 和 `followUp()` 直接提供这两种行为：如果输入已被排队（包括被扩展转换之后），返回 `"queued"`；如果被扩展消费，返回 `"handled"`。

`abort()` 停止当前活动的操作并等待会话变为空闲。`waitForIdle()` 只等待而不中止。

## 订阅事件

当宿主需要流式输出时，请在发送提示之前订阅：

```typescript
const unsubscribe = session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

try {
  await session.prompt("Explain this repository");
} finally {
  unsubscribe();
}
```

会话事件会报告消息更新、工具执行、队列、压缩、重试以及运行生命周期的变化。

`message_end` 包含权威的已完成消息。`agent_end` 标志一次底层 agent 运行的结束，但其后仍可能有自动恢复或排队的工作。

当宿主需要知道 Pi 不会再自动继续时，使用 `agent_settled`。

## 配置会话

在没有覆盖项的情况下，工厂函数会创建一个 `ModelRuntime`、基于文件的 `SettingsManager`、持久化的 `SessionManager`、`DefaultResourceLoader`，以及已配置的默认工具。

每个边界都可以显式提供：

- `modelRuntime`、`model`、`thinkingLevel` 和 `scopedModels` 控制模型的访问与选择。
- `settingsManager` 提供合并后的设置或内存中的配置。
- `sessionManager` 提供持久化的或内存中的对话历史。
- `resourceLoader` 提供扩展、技能（skills）、提示词模板、主题和上下文文件。
- `tools`、`noTools`、`excludeTools` 和 `customTools` 控制当前活动的工具集。

当你需要标准发现机制并只覆盖部分内容时，使用 `DefaultResourceLoader`。当宿主完全自行负责资源的存储与发现时，提供自定义的 `ResourceLoader`。

<a id="inlineextension"></a>

内联扩展工厂可以通过 `DefaultResourceLoader` 提供。只有当某个内联扩展需要在诊断信息和启动输出中拥有稳定名称时，才为其指定 `InlineExtension` 名称。带有 `replaceable: true` 的具名内联扩展，在另一个扩展注册了与它在加载期间注册的同名工具、命令或标志时会被排除，而不是两者都加载并产生冲突。CLI 内置的 codemode、工具搜索（tool search）和 MCP 扩展都是可替换的。带有 `builtin: true` 的具名条目不是内联扩展：它提供 `builtin:<name>` 扩展的代码，该扩展像已配置的扩展文件一样加载。它默认加载，会在 `pi config` 中列出，并可通过 `extensions` 设置中的 `-builtin:<name>` 或 `noExtensions` 禁用；`additionalExtensionPaths: ["builtin:<name>"]` 会显式加载它。它在项目信任（project trust）确定之后才加载，因此无法处理 `project_trust`。CLI 的内置扩展使用的就是这种方式。

<a id="codemode-mcp"></a>

CLI 会将 `codemode`、`tool_search` 和 MCP 作为内置扩展加载。SDK 会话不会；请将 `createCodemodeExtension()`、`createToolSearchExtension()` 和 `createMcpExtension()` 添加到 `DefaultResourceLoader` 的 `extensionFactories` 中。`codemode` 和 `tool_search` 注册时处于非活动状态：可以通过 `defaultTools` 设置启用它们（`["+codemode", "+tool_search"]` 会保留其他默认工具），或者由 MCP 扩展激活它们：对于 `codemode` 暴露方式的服务器激活 `codemode`，对于 `deferred` 暴露方式的服务器激活 `tool_search`。MCP 扩展在 `session_start` 时连接其服务器，因此需要调用 `session.bindExtensions()`。请参阅 [Codemode and MCP](../examples/sdk/14-codemode-mcp.ts)。

请参阅关于 [模型](../examples/sdk/02-custom-model.ts)、[工具](../examples/sdk/05-tools.ts)、[扩展](../examples/sdk/06-extensions.ts) 和 [完全控制](../examples/sdk/12-full-control.ts) 的专题示例。

## 示例

| 示例 | 用途 |
|---|---|
| [Minimal](../examples/sdk/01-minimal.ts) | 创建会话、发送提示、观察并释放会话 |
| [Custom model](../examples/sdk/02-custom-model.ts) | 选择模型和思考级别 |
| [System prompt](../examples/sdk/03-custom-prompt.ts) | 替换或追加系统提示词 |
| [Skills](../examples/sdk/04-skills.ts) | 发现、过滤和添加技能 |
| [Tools](../examples/sdk/05-tools.ts) | 选择内置工具及其工作目录 |
| [Extensions](../examples/sdk/06-extensions.ts) | 加载基于文件的扩展和内联扩展 |
| [Context files](../examples/sdk/07-context-files.ts) | 添加或替换项目说明 |
| [Prompt templates](../examples/sdk/08-prompt-templates.ts) | 添加文件形式的提示词模板 |
| [Credentials](../examples/sdk/09-api-keys-and-oauth.ts) | 配置凭据和模型存储 |
| [Settings](../examples/sdk/10-settings.ts) | 提供基于文件的或内存中的设置 |
| [Sessions](../examples/sdk/11-sessions.ts) | 控制会话持久化与恢复 |
| [Full control](../examples/sdk/12-full-control.ts) | 替换默认的发现与状态服务 |
| [Session runtime](../examples/sdk/13-session-runtime.ts) | 安全地替换当前活动会话 |
| [Codemode and MCP](../examples/sdk/14-codemode-mcp.ts) | 添加 `codemode`、`tool_search` 和 MCP 扩展 |

<a id="exports"></a>

## 资源

- [Choose a Model](models.md) 介绍模型选择和兼容的端点；[Providers](providers.md) 介绍凭据和特定提供商的设置。
- [Configuration](configuration.md) 说明常规的发现机制和设置；[Settings](settings.md) 列出所有设置项。
- [Sessions and Context](sessions.md) 说明会话行为；[Session Format](session-format.md) 定义持久化条目；[Message Types](message-types.md) 定义共享的对话记录值。
- [Extensions](extensions.md)、[Skills](skills.md) 和 [Prompt Templates](prompt-templates.md) 记录了通过 `ResourceLoader` 提供的资源。
- [CLI Integration](cli-integration.md) 介绍 print、JSON 和 RPC 等替代进程内 SDK 集成的方式。
