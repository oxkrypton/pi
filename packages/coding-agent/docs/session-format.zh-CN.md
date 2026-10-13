# 会话文件格式

会话（session）以 JSONL（JSON Lines）文件形式存储。每一行都是一个带有 `type` 字段的 JSON 对象。会话条目通过 `id`/`parentId` 字段构成树结构，从而可以在原文件内创建分支，而无需创建新文件。

关于以编程方式创建、持久化和导航树结构，请参阅 [`SessionManager` API](sdk.md#sessionmanager-api)。


## 文件位置

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<session-id>.jsonl
```

默认情况下，`<session-id>` 是一个 UUID。调用方可以通过 SDK 或 `--session-id` 提供自定义 ID。对于 `<path>`，Pi 会去掉开头的路径分隔符，并将 `/`、`\\` 和 `:` 替换为 `-`。

## 删除会话

可以通过删除 `~/.pi/agent/sessions/` 下对应的 `.jsonl` 文件来移除会话。

Pi 也支持在 `/resume` 中交互式删除会话（选中一个会话并按 `Ctrl+D`，然后确认）。如果系统中有 `trash` CLI，pi 会使用它，以避免永久删除。

## 会话版本

会话在头部包含一个版本字段：

- **Version 1**：线性条目序列（旧格式，加载时自动迁移）
- **Version 2**：使用 `id`/`parentId` 链接的树结构
- **Version 3**：将 `hookMessage` 角色重命名为 `custom`（扩展（extension）统一）

已有会话在加载时会自动迁移到当前版本（v3）。

## 源文件

GitHub 上的源码（[pi](https://github.com/earendil-works/pi)）：
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts) - 会话条目类型和 SessionManager
- [Message Types](message-types.md) - 共享的消息和内容块参考
- [`packages/coding-agent/src/core/messages.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/messages.ts) - 扩展的消息类型
- [`packages/ai/src/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts) - 基础消息和内容块类型
- [`packages/agent/src/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/agent/src/types.ts) - 可扩展的 `AgentMessage` 联合类型

如需在你的项目中查看 TypeScript 定义，请查看 `node_modules/@earendil-works/pi-coding-agent/dist/` 和 `node_modules/@earendil-works/pi-ai/dist/`。

## 消息

`message` 条目存储一个 [`AgentMessage`](message-types.md)。消息内容块、角色、用量以及消息时间戳的定义见 [Message Types](message-types.md)。

会话条目的时间戳是 ISO 8601 字符串。嵌套在消息内的时间戳是以毫秒为单位的 Unix 时间戳。

## 条目基础结构

所有条目（`SessionHeader` 除外）都继承 `SessionEntryBase`：

```typescript
interface SessionEntryBase {
  type: string;
  id: string;           // Usually an 8-char hex ID; may fall back to a full UUID
  parentId: string | null;  // Parent entry ID (null for a root entry)
  timestamp: string;    // ISO timestamp
}
```

## 条目类型

### SessionHeader

文件的第一行。仅包含元数据，不属于树结构（没有 `id`/`parentId`）。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
```

对于有父会话的会话（通过 `/fork`、`/clone` 或 `newSession({ parentSession })` 创建）：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project","parentSession":"/path/to/original/session.jsonl"}
```

### SessionMessageEntry

对话中的一条消息。`message` 字段包含一个 `AgentMessage`。系统消息承载提示词和工具配置：会话的第一次请求会持久化一条包含所有提示词段落和工具声明的系统消息，之后的变更则以系统消息的形式持久化，这些系统消息按名称修补 `sections`（`null` 表示移除某个段落），并列出 `toolsAdded`/`toolsRemoved`。按顺序重放这些消息即可得到当前的提示词和工具；不存在单独的提示词状态条目。

```json
{"type":"message","id":"a0b1c2d3","parentId":null,"timestamp":"2024-12-03T14:00:00.000Z","message":{"role":"system","content":"","sections":{"preamble":"You are an expert coding assistant...","tools":"<tools>\n- read: ...\n</tools>","cwd":"/project"},"toolsAdded":[{"name":"read","description":"...","parameters":{}}],"timestamp":1733234400000}}
{"type":"message","id":"d4e5f6g7","parentId":"c3d4e5f6","timestamp":"2024-12-03T14:04:00.000Z","message":{"role":"system","content":"","sections":{"skills":"<skills>...</skills>"},"toolsRemoved":[{"name":"write"}],"timestamp":1733234640000}}
```

在系统消息出现之前创建的会话没有开头的系统消息；其第一次请求会以一条后续的系统消息声明当前提示词，重放方式相同。

```json
{"type":"message","id":"a1b2c3d4","parentId":"prev1234","timestamp":"2024-12-03T14:00:01.000Z","message":{"role":"user","content":"Hello","timestamp":1733234401000}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:00:02.000Z","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}],"api":"anthropic-messages","provider":"anthropic","model":"claude-sonnet-4-5","usage":{...},"stopReason":"stop","timestamp":1733234402000}}
{"type":"message","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"2024-12-03T14:00:03.000Z","message":{"role":"toolResult","toolCallId":"call_123","toolName":"bash","content":[{"type":"text","text":"output"}],"isError":false,"timestamp":1733234403000}}
```

助手消息会注明生成它们的模型。较新的消息还会记录 `thinkingLevel`，即该次响应所请求的 Pi 思考级别。

### ModelChangeEntry

当用户在会话中途切换模型时产生。最新的该条目即为所选模型，它可能是一个[虚拟模型](virtual-models.md)；此时助手消息会注明实际作答的物理模型。

```json
{"type":"model_change","id":"d4e5f6g7","parentId":"c3d4e5f6","timestamp":"2024-12-03T14:05:00.000Z","provider":"openai","modelId":"gpt-4o"}
```

### ThinkingLevelChangeEntry

当用户更改思考/推理级别时产生。

```json
{"type":"thinking_level_change","id":"e5f6g7h8","parentId":"d4e5f6g7","timestamp":"2024-12-03T14:06:00.000Z","thinkingLevel":"high"}
```

### UsageEntry

记录归属于某个模型、但不是助手消息且不参与 LLM 上下文的用量。`kind` 是用于标识操作的任意字符串；例如，缓存预热使用 `"cache_warm"`。

```json
{"type":"usage","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"2024-12-03T14:08:00.000Z","kind":"cache_warm","provider":"anthropic","model":"claude-sonnet-4-5","usage":{"input":0,"output":0,"cacheRead":50000,"cacheWrite":0,"totalTokens":50000,"cost":{"input":0,"output":0,"cacheRead":0.015,"cacheWrite":0,"total":0.015}}}
```

用量条目计入会话的 token（令牌）和费用总计。Pi 会在对话树中隐藏它们。使用方应将未知的 `kind` 值视为普通用量，而不是拒绝它们。

### CompactionEntry

在上下文被压缩（compaction）时创建。存储较早消息的摘要以及一个完整的系统提示词/工具检查点。

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"2024-12-03T14:10:00.000Z","summary":"User discussed X, Y, Z...","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000,"systemMessage":{"role":"system","content":"You are a coding assistant.","toolsAdded":[],"timestamp":1733235000000}}
```

`firstKeptEntryId` 为必填字段。它标识压缩条目之前被保留的第一个条目。重建上下文时，Pi 用压缩摘要替换较早的已摘要条目，并保留从该条目开始的范围。不保留任何条目的压缩会在此字段中存储它自身的 ID，因此不会保留任何之前的条目。

可选字段：
- `systemMessage`：压缩边界处重放得到的提示词段落和工具声明；它会成为压缩后上下文开头的系统消息，保留条目中的系统消息会被丢弃，以它为准。较旧的会话条目中没有此字段。
- `usage`：生成摘要时的 LLM 用量；计入会话的 token 和费用总计
- `details`：特定于实现的数据（例如，默认实现为 `{ readFiles: string[], modifiedFiles: string[] }`，扩展则为自定义数据）
- `fromHook`：若由扩展生成则为 `true`，若由 pi 生成则为 `false`/`undefined`（旧字段名）

### ContextEditEntry

对某个较早的、会产生上下文的条目进行的仅追加式编辑。它只改变之后的模型上下文；目标条目及其元数据在原始历史、UI、导出和会话统计中保持不变。

```json
{"type":"context_edit","id":"g6h7i8j9","parentId":"f6g7h8i9","timestamp":"2024-12-03T14:11:00.000Z","targetId":"c3d4e5f6","replacement":null}
```

目标可以是用户、助手、工具结果或自定义消息条目。`replacement: null` 会将目标从模型上下文中省略。非 null 的 `replacement` 只替换目标消息的内容。对于助手和工具结果条目，字符串形式的替换内容会被规范化为一个文本块，因为这些角色要求内容为数组。如果有多个编辑指向同一条目，则以活动分支上最新的编辑为准。编辑是相对于分支的：导航到编辑之前的某个位置时，目标的原始内容会重新出现在上下文中。

### BranchSummaryEntry

在通过 `/tree` 切换分支时创建，包含由 LLM 生成的、对被离开分支（直到共同祖先为止）的摘要。用于保留被放弃路径上的上下文。

```json
{"type":"branch_summary","id":"g7h8i9j0","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:15:00.000Z","fromId":"f6g7h8i9","summary":"Branch explored approach A..."}
```

`parentId` 是新分支继续延伸的起点条目。`fromId` 是之前的叶节点，其被放弃的路径已被摘要。

可选字段：
- `usage`：生成摘要时的 LLM 用量；计入会话的 token 和费用总计
- `details`：文件跟踪数据（默认实现为 `{ readFiles: string[], modifiedFiles: string[] }`），扩展则为自定义数据
- `fromHook`：若由扩展生成则为 `true`，若由 pi 生成则为 `false`/`undefined`（旧字段名）

### CustomEntry

扩展状态持久化。**不**参与 LLM 上下文。

```json
{"type":"custom","id":"h8i9j0k1","parentId":"g7h8i9j0","timestamp":"2024-12-03T14:20:00.000Z","customType":"my-extension","data":{"count":42}}
```

使用 `customType` 在重新加载时识别你的扩展的条目。交互模式可以通过 `pi.registerEntryRenderer(customType, renderer)` 渲染自定义条目，但它们仍然不参与 LLM 上下文。

Pi 将[虚拟模型](virtual-models.md)路由器状态存储为自定义条目，其 `customType` 为 `pi.virtual-model-state`，`data` 为 `{ provider, modelId, state }`。

### CustomMessageEntry

由扩展注入、**会**参与 LLM 上下文的消息。

```json
{"type":"custom_message","id":"i9j0k1l2","parentId":"h8i9j0k1","timestamp":"2024-12-03T14:25:00.000Z","customType":"my-extension","content":"Injected context...","display":true}
```

字段：
- `content`：字符串或 `(TextContent | ImageContent)[]`（与 UserMessage 相同）
- `display`：`true` = 在 TUI 中以特殊样式显示，`false` = 隐藏
- `details`：可选的扩展特定元数据（不发送给 LLM）

### LabelEntry

用户在某个条目上定义的书签/标记。

```json
{"type":"label","id":"j0k1l2m3","parentId":"i9j0k1l2","timestamp":"2024-12-03T14:30:00.000Z","targetId":"a1b2c3d4","label":"checkpoint-1"}
```

将 `label` 设为 `undefined` 可清除标签。

### SessionInfoEntry

会话元数据（例如用户定义的显示名称）。可通过 `/name`、`--name` / `-n`，或在扩展中调用 `pi.setSessionName()` 设置。

```json
{"type":"session_info","id":"k1l2m3n4","parentId":"j0k1l2m3","timestamp":"2024-12-03T14:35:00.000Z","name":"Refactor auth module"}
```

设置了会话名称后，会话选择器（`/resume`）中会显示该名称，而不是第一条消息。

## 树结构

条目通常构成一棵树，但导航 API 可以创建多个根：
- 根条目的 `parentId` 为 `null`；第一个条目最初就是根
- 每个非根条目通过 `parentId` 指向其父条目
- 分支会从较早的条目创建新的子条目
- “叶节点”（leaf）是树中的当前位置
- 调用 `resetLeaf()` 或 `branchWithSummary(null, ...)` 可以让之后的条目成为另一个根

```
[user msg] ─── [assistant] ─── [user msg] ─── [assistant] ─┬─ [user msg] ← current leaf
                                                            │
                                                            └─ [branch_summary] ─── [user msg] ← alternate branch
```

## 上下文构建

`buildContextEntries()` 从当前叶节点遍历到根，在遵循压缩规则的前提下生成活动条目列表：

1. 收集路径上的所有条目
2. 如果路径上有一个或多个 `CompactionEntry`，则使用最新的一个：
   - 首先包含该压缩条目
   - 包含从 `firstKeptEntryId` 开始、直到（但不包括）该压缩条目之间的非系统条目
   - 包含该压缩条目之后的条目
3. 保留所选范围内的非消息条目，以便交互模式可以渲染它们

`buildSessionProjection()` 随后为每个选中的目标应用最新的 `context_edit`。它返回模型可见的消息及其来源条目。被省略的目标不产生消息；替换会保留来源条目的角色和元数据，只改变内容。原始的选中条目不会被修改。

`buildSessionContext()` 在该投影的基础上生成发送给 LLM 的消息列表：

1. 从完整路径中提取当前的模型和思考级别设置
2. 将选中的条目转换为消息：
   - `message` -> 存储的 `AgentMessage`
   - `compaction` -> 完整的系统检查点，后跟 `compactionSummary`
   - `branch_summary` -> `branchSummary`
   - `custom_message` -> `CustomMessage`
   - `context_edit` -> 自身不产生上下文消息
   - `usage` 和 `custom` -> 不产生上下文消息

压缩摘要替换 `firstKeptEntryId` 之前的条目。压缩之前的系统消息会被并入完整检查点，而不是从保留范围中重放。保留的非系统条目以及压缩之后的所有条目对 LLM 仍然可见。

## 解析示例

```typescript
import { readFileSync } from "fs";

const lines = readFileSync("session.jsonl", "utf8").trim().split("\n");

for (const line of lines) {
  const entry = JSON.parse(line);

  switch (entry.type) {
    case "session":
      console.log(`Session v${entry.version ?? 1}: ${entry.id}`);
      break;
    case "message":
      console.log(`[${entry.id}] ${entry.message.role}: ${JSON.stringify(entry.message.content)}`);
      break;
    case "compaction":
      console.log(`[${entry.id}] Compaction: ${entry.tokensBefore} tokens summarized`);
      break;
    case "branch_summary":
      console.log(`[${entry.id}] Branch from ${entry.fromId}`);
      break;
    case "usage":
      console.log(`[${entry.id}] Usage (${entry.kind}): ${entry.usage.totalTokens} tokens`);
      break;
    case "custom":
      console.log(`[${entry.id}] Custom (${entry.customType}): ${JSON.stringify(entry.data)}`);
      break;
    case "custom_message":
      console.log(`[${entry.id}] Extension message (${entry.customType}): ${entry.content}`);
      break;
    case "label":
      console.log(`[${entry.id}] Label "${entry.label}" on ${entry.targetId}`);
      break;
    case "model_change":
      console.log(`[${entry.id}] Model: ${entry.provider}/${entry.modelId}`);
      break;
    case "thinking_level_change":
      console.log(`[${entry.id}] Thinking: ${entry.thinkingLevel}`);
      break;
  }
}
```
