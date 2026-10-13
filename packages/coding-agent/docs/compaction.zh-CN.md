# 压缩参考

本参考文档介绍自动 compaction（压缩）、分支摘要（branch summarization）、持久化条目以及 extension（扩展）钩子。关于用户工作流，请参阅 [Sessions and Context](sessions.md#manage-conversation-context)。

**源文件**（[pi](https://github.com/earendil-works/pi)）：
- [`packages/coding-agent/src/core/compaction/compaction.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) - 自动压缩逻辑
- [`packages/coding-agent/src/core/compaction/branch-summarization.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) - 分支摘要
- [`packages/coding-agent/src/core/compaction/utils.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/utils.ts) - 共享工具函数（文件跟踪、序列化）
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts) - 条目类型（`CompactionEntry`、`BranchSummaryEntry`）
- [`packages/coding-agent/src/core/extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts) - 扩展事件类型

如需在你的项目中查看 TypeScript 类型定义，请检查 `node_modules/@earendil-works/pi-coding-agent/dist/`。

## 概述

Pi 有两种摘要机制：

| 机制 | 触发条件 | 用途 |
|-----------|---------|---------|
| 压缩（Compaction） | 上下文超过阈值，或执行 `/compact` | 对旧消息生成摘要以释放上下文空间 |
| 分支摘要（Branch summarization） | 通过 `/tree` 导航 | 切换分支时保留上下文 |

两者使用密切相关的结构化格式，并累积跟踪文件操作。摘要请求会禁用提示缓存（prompt cache）写入，因为这些一次性提示不太可能被复用。

## 压缩

### 触发时机

当满足以下条件时触发自动压缩：

```
contextTokens > contextWindow - reserveTokens
```

默认情况下，`reserveTokens` 为 16384 个 token（令牌）（可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）。这为 LLM 的响应预留了空间。

在多轮智能体运行期间，Pi 会在工具执行完毕且其结果被追加之后、开始下一次助手响应之前，检查规范的投影上下文（canonical projected context）。如果超过阈值，Pi 会在 `prepareNextTurn` 期间执行压缩，然后在 `turn_start` 之前执行现有的追赶式引导轮询（catch-up steering poll）。如果刚完成的工具批次终止了本次运行，且没有排队的消息需要再次响应，Pi 会跳过这一轮间检查。Pi 还会在新的用户提示之前进行检查，并在底层运行结束后执行最终尝试的溢出恢复。

提供方返回的上下文溢出错误，或过早出现的最终 `stopReason: "length"`，可以触发一次"压缩并重试"的恢复尝试。带有工具调用的 length 响应会保留其合成的失败工具结果，并遵循普通的工具/队列调度器，而不是强制结束运行。

你也可以通过 `/compact [instructions]` 手动触发，其中可选的 instructions 用于让摘要聚焦于特定内容。

### 工作原理

1. **查找切分点**：在最终确定的会话投影中向后遍历，累加 token 估算值，直到达到 `keepRecentTokens`（默认 20k，可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）
2. **提取消息**：收集从上一个保留边界（或会话开始处）到切分点之间的投影消息
3. **生成摘要**：调用 LLM 按结构化格式生成摘要；如果存在上一次的摘要，则将其作为迭代上下文传入
4. **追加条目**：保存包含摘要和 `firstKeptEntryId` 的 `CompactionEntry`
5. **重建上下文**：会话为下一次请求重建上下文，使用摘要加上从 `firstKeptEntryId` 开始的消息

```
Before compaction:

  entry:  0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
                └────────┬───────┘ └──────────────┬──────────────┘
               messagesToSummarize            kept messages
                                   ↑
                          firstKeptEntryId (entry 4)

After compaction (new entry appended):

  entry:  0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│ cmp │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┴─────┘
               └──────────┬──────┘ └──────────────────────┬───────────────────┘
                 not sent to LLM                    sent to LLM
                                                         ↑
                                              starts from firstKeptEntryId

What the LLM sees:

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ system │ summary │ usr │ ass │ tool │ tool │ ass │ tool │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    prompt   from cmp          messages from firstKeptEntryId
```

在重复压缩时，被摘要的范围从上一次压缩的保留边界（`firstKeptEntryId`）开始，而不是从压缩条目本身开始；如果在路径中找不到该保留条目，则回退到上一次压缩之后的那个条目。不保留任何消息（retain-none）的压缩会将其自身 ID 记录为 `firstKeptEntryId`；重复压缩会从该条目之后开始。这样，在上一次压缩中保留下来的消息也会被纳入下一次摘要过程，从而得以保留。在写入新的 `CompactionEntry` 之前，Pi 还会根据重建后、经过上下文编辑的会话投影重新计算 `tokensBefore`，使 token 计数反映被替换的实际压缩前上下文。被省略的原始条目仍会保存，但不会影响切分点选择、摘要、检查点或 token 估算。

### 溢出与长度恢复的顺序

恢复过程会保持现有的生命周期和队列顺序。已完成的尝试对 `turn_end` 和 `agent_end` 仍然可见；随后，运行后恢复会在全新重试之前修复已持久化的模型上下文：

```text
persist final assistant response
→ extension/public turn_end
→ extension/public agent_end
→ append context_edit omissions for the selected attempt
→ for overflow/length: run session_before_compact and append compaction on success
→ start the retry as a fresh run
```

如果恢复压缩失败或被取消，Pi 会保留省略编辑，不追加压缩条目，也不安排内部重试。已排队的现有工作仍受普通的引导（steering）和后续（follow-up）规则约束。`agent_before_settle` 会看到恢复处理之后修复过的投影。原始对话记录历史、导出、计费总计以及历史搜索扩展仍然可以查看被省略的尝试。

### 拆分的用户消息跨度

用户消息跨度（user-message span）以一条用户消息开始，包含直到下一条用户消息之前的所有轮次。通常，压缩会在用户消息边界处切分。

当某个用户消息跨度超过 `keepRecentTokens` 时，切分点会落在该跨度内部的一条助手消息处。这就是拆分的用户消息跨度：

```
Split user-message span (one span exceeds budget):

  entry:  0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ hdr │ usr │ ass │ tool │ ass │ tool │ tool │ ass │ tool │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
                ↑                                     ↑
         turnStartIndex = 1                  firstKeptEntryId = 7
                │                                     │
                └──── turnPrefixMessages (1-6) ───────┘
                                                      └── kept (7-8)

  isSplitTurn = true
  messagesToSummarize = []  (no earlier user-message spans)
  turnPrefixMessages = [usr, ass, tool, ass, tool, tool]
```

对于拆分的用户消息跨度，Pi 会生成两份摘要并将其合并：
1. **历史摘要**：之前的上下文（如果有）
2. **用户消息跨度前缀摘要**：被拆分的用户消息跨度的前半部分

### 切分点规则

有效的切分点包括：
- 用户消息
- 助手消息
- BashExecution 消息
- 自定义消息（custom_message、branch_summary）

绝不在工具结果处切分（工具结果必须与其工具调用保持在一起）。

只有当某个对上下文不可见的后缀中包含一次被省略的助手尝试、且不包含任何未被省略的上下文生成条目时，准备阶段才会将保留边界推进到该后缀中。恢复产生的 `context_edit` 省略满足此规则；本身对上下文不可见的元数据可以与之共存。仅有元数据或新追加的自定义消息不会移动切分点。影响候选输入或被摘要前缀的替换编辑也会阻止推进，因为被省略的助手响应针对的是编辑前的输入；而对最终被省略的后缀条目所做的替换仍然是安全的。这使得超出预算的已恢复输入可以被摘要，同时保留那些使被放弃的尝试保持省略状态的编辑，并且不会让簿记操作改变新的模型输入是否被原样保留。

### CompactionEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface CompactionEntry<T = unknown> {
  type: "compaction";
  id: string;
  parentId: string | null;
  timestamp: string;
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  usage?: Usage;       // LLM usage that generated the summary
  fromHook?: boolean;  // true if provided by extension (legacy field name)
  details?: T;         // implementation-specific data
}

// Default compaction uses this for details (from compaction.ts):
interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

扩展可以在 `details` 中存储任何可 JSON 序列化的数据。默认压缩会跟踪文件操作，但自定义的扩展实现可以使用自己的结构。生成的摘要和扩展提供的摘要在可用时都会存储其 LLM `usage`，以便会话总计包含摘要工作的消耗。

具体实现请参阅 [`prepareCompaction()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) 和 [`compact()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)。如需直接以编程方式生成摘要，`generateSummary()` 返回摘要文本，`generateSummaryWithUsage()` 返回 `{ text, usage }`。

## 分支摘要

### 触发时机

当你使用 `/tree` 导航到另一个分支时，Pi 会提供对你正在离开的工作生成摘要的选项。这会将离开的分支中的上下文注入到新分支中。

### 工作原理

1. **查找共同祖先**：新旧位置共享的最深节点
2. **收集条目**：从旧的叶节点向回遍历到共同祖先
3. **按预算准备**：在 token 预算内纳入消息（最新的优先）
4. **生成摘要**：调用 LLM 按结构化格式生成摘要
5. **追加条目**：在导航点保存 `BranchSummaryEntry`

```
Tree before navigation:

         ┌─ B ─ C ─ D (old leaf, being abandoned)
    A ───┤
         └─ E ─ F (target)

Common ancestor: A
Entries to summarize: B, C, D

After navigation with summary:

         ┌─ B ─ C ─ D
    A ───┤
         └─ E ─ F ─ [summary of B,C,D] (new leaf)
```

### 累积文件跟踪

默认压缩和分支摘要都会累积跟踪文件。两者都会从被摘要消息中的工具调用里提取文件操作。压缩还会沿用上一次由 Pi 生成的压缩中的文件列表。分支摘要会沿用其所摘要条目中由 Pi 生成的分支摘要里的文件列表。

因此，文件跟踪会在多次默认压缩和嵌套的默认分支摘要之间不断累积。对于 `fromHook` 字段为 `true` 的扩展生成摘要，Pi 不会自动沿用其文件列表；扩展需自行管理其 `details` 格式。

### BranchSummaryEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface BranchSummaryEntry<T = unknown> {
  type: "branch_summary";
  id: string;
  parentId: string | null;
  timestamp: string;
  summary: string;
  fromId: string;      // Entry we navigated from
  usage?: Usage;       // LLM usage that generated the summary
  fromHook?: boolean;  // true if provided by extension (legacy field name)
  details?: T;         // implementation-specific data
}

// Default branch summarization uses this for details (from branch-summarization.ts):
interface BranchSummaryDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

与压缩相同，扩展可以在 `details` 中存储自定义数据。

具体实现请参阅 [`collectEntriesForBranchSummary()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`prepareBranchEntries()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) 和 [`generateBranchSummary()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)。

## 摘要格式

两种格式都包含 Goal、Constraints & Preferences、Progress、Key Decisions 和 Next Steps。压缩摘要还包含 Critical Context。分支摘要在 Next Steps 之后结束。在相关时，Pi 会向任一格式追加文件列表。

压缩摘要使用以下格式：

```markdown
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
- [Requirements mentioned by user]

## Progress
### Done
- [x] [Completed tasks]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues, if any]

## Key Decisions
- **[Decision]**: [Rationale]

## Next Steps
1. [What should happen next]

## Critical Context
- [Data needed to continue]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

### 消息序列化

在生成摘要之前，消息会通过 [`serializeConversation()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/utils.ts) 序列化为文本：

```
[User]: What they said
[Assistant thinking]: Internal reasoning
[Assistant]: Response text
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: Output from tool
```

这可以防止模型将其视为一段需要继续的对话。

在序列化过程中，工具结果会被截断为 2000 个字符。超出该限制的内容会被替换为一个标记，指明截断了多少字符。这能使摘要请求保持在合理的 token 预算内，因为工具结果（尤其是来自 `read` 和 `bash` 的结果）通常是上下文大小的最大来源。

## 通过扩展自定义摘要

扩展可以拦截并自定义压缩和分支摘要。事件类型定义请参阅 [`extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts)。

### session_before_compact

在自动压缩或 `/compact` 之前触发。可以取消操作或提供自定义摘要。请参阅类型文件中的 `SessionBeforeCompactEvent` 和 `CompactionPreparation`。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, reason, willRetry, signal } = event;

  // preparation.messagesToSummarize - messages to summarize
  // preparation.turnPrefixMessages - user-message-span prefix (if isSplitTurn)
  // preparation.previousSummary - previous compaction summary
  // preparation.fileOps - extracted file operations
  // preparation.tokensBefore - context tokens before compaction
  // preparation.firstKeptEntryId - where kept messages start
  // preparation.settings - effective settings after applying model overrides

  // branchEntries - all entries on current branch (for custom state)
  // reason - "manual" (/compact), "threshold", or "overflow"
  // willRetry - whether the aborted turn is retried after compaction (overflow recovery)
  // signal - AbortSignal (pass to LLM calls)

  // Cancel:
  return { cancel: true };

  // Custom summary:
  return {
    compaction: {
      summary: "Your summary...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      // usage: summaryResponse.usage, // Optional; included in session totals
      details: { /* custom data */ },
    }
  };
});
```

#### 将消息转换为文本

如需使用你自己的模型生成摘要，请使用 `serializeConversation` 将消息转换为文本：

```typescript
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;
  
  // Convert AgentMessage[] to Message[], then serialize to text
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  // Returns:
  // [User]: message text
  // [Assistant thinking]: thinking content
  // [Assistant]: response text
  // [Assistant tool calls]: read(path="..."); bash(command="...")
  // [Tool result]: output text

  // Now send to your model for summarization
  const { summary, usage } = await myModel.summarize(conversationText);
  
  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      usage,
    }
  };
});
```

使用其他模型的完整示例请参阅 [custom-compaction.ts](../examples/extensions/custom-compaction.ts)。

### session_compact_failed

在手动或自动压缩失败或被中止时触发。对于需要将 `session_before_compact` 尝试与最终结果配对的遥测扩展，这非常有用。

```typescript
pi.on("session_compact_failed", async (event, ctx) => {
  const { reason, errorMessage, aborted, willRetry, fromExtension } = event;
  // reason - "manual" (/compact), "threshold", or "overflow"
  // errorMessage - present for non-abort failures
  // aborted - true for canceled/aborted compactions
  // willRetry - whether the aborted turn would have retried after compaction
  // fromExtension - whether extension-provided compaction content was being used
});
```

### session_before_tree

在 `/tree` 导航之前触发。无论用户是否选择生成摘要，该事件总会触发。可以取消导航或提供自定义摘要。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;

  // preparation.targetId - where we're navigating to
  // preparation.oldLeafId - current position (being abandoned)
  // preparation.commonAncestorId - shared ancestor
  // preparation.entriesToSummarize - entries that would be summarized
  // preparation.userWantsSummary - whether user chose to summarize

  // Cancel navigation entirely:
  return { cancel: true };

  // Provide custom summary (only used if userWantsSummary is true):
  if (preparation.userWantsSummary) {
    return {
      summary: {
        summary: "Your summary...",
        // usage: summaryResponse.usage, // Optional; included in session totals
        details: { /* custom data */ },
      }
    };
  }
});
```

请参阅类型文件中的 `SessionBeforeTreeEvent` 和 `TreePreparation`。

## 设置

在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置压缩：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 设置 | 默认值 | 说明 |
|---------|---------|-------------|
| `enabled` | `true` | 启用自动压缩 |
| `reserveTokens` | `16384` | 为 LLM 响应预留的 token 数 |
| `keepRecentTokens` | `20000` | 保留的最近 token 数（不进行摘要） |

使用 `"enabled": false` 可禁用自动压缩。你仍然可以通过 `/compact` 手动压缩。

### 按模型覆盖

使用 `compaction.modelOverrides` 为不同模型调整 token 预算：

```json
{
  "compaction": {
    "reserveTokens": 16384,
    "keepRecentTokens": 20000,
    "modelOverrides": {
      "some-provider/big-model": {
        "reserveTokens": 400000
      }
    }
  }
}
```

对于上下文窗口为 1M 的模型，此覆盖会在超过 600K token 时触发压缩，并保留普通设置中的 20000 个最近 token。其他模型仍使用普通的 16384 token 预留。`reserveTokens` 还会影响摘要的输出上限（以模型的最大输出 token 数为上限）；它并不仅仅是一个触发阈值。

键是精确且区分大小写的 `provider/modelId` 值，包括模型 ID 中的任何斜杠。每个 `reserveTokens` 和 `keepRecentTokens` 值都会独立地依次回退：从模型覆盖到普通设置，再到内置默认值。值必须是非负的安全整数。匹配的模型覆盖中的无效值会在读取时报错；只有被省略的字段才会回退到普通设置。模型覆盖条目必须是对象。普通 token 设置中的无效值会在读取时报错，即使当前模型有有效的覆盖也是如此。只有被省略的普通值才会使用内置默认值。`enabled` 仍是全局设置，不针对特定模型。

这些解析后的值会用于手动压缩、所有自动阈值检查、溢出恢复以及扩展可见的 `preparation.settings`。切换模型会影响后续的检查和压缩，但不会更改普通设置。正在进行中的压缩使用为该操作捕获的模型和设置。分支摘要的设置不受影响。

覆盖在全局设置和项目设置中均有效。两个文件会在查找前递归合并，因此全局的模型特定值优先于项目范围的回退值；项目必须覆盖该模型条目才能更改它。详情请参阅 [Settings](settings.md#per-model-compaction-overrides)。
