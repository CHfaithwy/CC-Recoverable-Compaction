# Recoverable Compaction


`Recoverable Compaction` 是一个面向 Clow Code 类 Agent 的长上下文增强方案。让 Agent 在压缩上下文之后，仍然可以按需找回旧日志、旧报错、旧 `stack trace` 和子 Agent 的长输出。

---

## 1. Clow Code 类 Agent 的问题

现有 `compact` 机制是“有损”压缩，而不是“证据可恢复压缩”。

在真实使用里，主要有 4 个痛点：

1. `compact summary` 更适合继续做任务，不适合精确追溯旧证据。
2. 用户后续追问“刚才完整报错是什么”“之前那次 `stack trace` 贴出来”时，模型容易只能复述摘要，而不能恢复原始内容，模型必须重新下达指令重新运行才能看到完整报错。
3. 子 Agent / Worker 的长日志、测试输出、工具结果只返回给主agent report 而不是完整内容。
4. compact 后主 Agent 知道“之前发生过什么”，但不知道“应该去哪里把原始证据找回来”，从而容易`重复`读文件、`重复`跑命令，浪费 token 和 时间。

---

## 2. 核心思路


> 在现有 `Clow Code 的 compact` 机制上，增加一层可恢复证据索引和按需恢复能力。

当前实现遵循这几个原则：

1. 复用现有机制，不新建独立 transcript store，不重写 Agent Loop。
2. 低侵入增强，在 compact 完成后追加 indexing / metadata / restore 能力。
3. 优先使用稳定指针，如 `messageUuid`、`toolUseId`、`toolResultPath`。
4. 默认不把证据全文重新注入活跃上下文，只保留摘要和轻量提示。
5. 恢复出的旧证据只能辅助决策，不能替代当前状态验证。

可以把它理解成：

```text
active context = compact summary + 少量恢复提示
recoverable store = 薄索引 + 正文归档
```

其中：

- 薄索引：`compact-blocks.jsonl` 负责保存“怎么找到证据” 
- 正文归档：`projects/.agentguard/compaction/evidence/...` 负责保存“证据正文快照”


---

## 3. 核心数据结构


### 3.1 CompactBlock

表示一次 compact 的结果，记录：

- `compactBoundaryUuid`
- `tokenBefore / tokenAfter`
- `naturalSummary`
- `summaryItems`
- `evidence`


### 3.2 CompactSummaryItem

表示 compact summary 里的单条结构化结论，记录：

- `kind`
- `text`
- `confidence`
- `evidenceIds`
- `queryHints`

作用：让“摘要句子”和“可恢复证据”之间建立映射。

### 3.3 EvidencePointer

表示一条可恢复证据的索引项，当前实现可指向：

- `messageUuid`
- `toolUseId`
- `toolResultPath`
- `archivedToolResultPath`
- `sourceScope = main_agent | sub_agent`
- `agentRunId`
- `contentTypes`
- `anchors`


---

## 4. 接入点


### 4.1 compact 完成后建索引

在 compact 完成后：

- 扫描本次被摘要掉的消息范围
- 提取 message / tool result / sub-agent transcript
- 生成 `CompactBlock`
- 为 summary item 绑定 `EvidencePointer`

### 4.2 大工具输出落盘时补 pointer

复用现有 large tool result persistence：

- 大输出写入持久化文件
- 建立 `toolUseId -> toolResultPath`
- 如果后续被 compact，直接转成 recoverable evidence

### 4.3 sub-agent / Worker 输出纳入同一套证据层

当前实现会把主 Agent 和 sub-agent / Worker 的输出统一纳入 recoverable store，并额外记录：

- `sourceScope`
- `agentRunId`
- `parentMessageUuid`
- `contentTypes`

这样 compact 后主 Agent 仍然能追溯到 Worker 当时看到的长日志。

### 4.4 用户追问前自动注入恢复结果

如果会话里已经存在 compact summary，且用户最新 query 命中恢复策略，则系统会在真正送入模型前：

- 读取当前 session 的 `compact-blocks.jsonl`
- 检索匹配的 evidence
- 恢复对应正文
- 以 `system-reminder` 形式注入到最新 user message 前


---

## 5. 当前已实现的触发机制


### 5.1 触发前提

需要同时满足：

1. 当前会话里已经存在 compact summary
2. 用户最新问题命中恢复策略

### 5.2 已实现的三类触发意图

#### `explicit_detail`

适用于：

- “刚才测试失败的具体报错是什么？”
- “输出所有 `stack trace`”
- “把之前那次 worker 看到的日志恢复出来”

#### `historical_verification`

适用于：

- “你确定是这个原因吗？”
- “这个判断有证据吗？”

#### `continue_previous_failure`

适用于：

- “按那个错误继续修”
- “按之前那次失败继续排查”

### 5.3 如何检索 evidence

当前实现不是把整个 `compact-blocks.jsonl` 丢给 LLM，而是先由代码做结构化检索，再把命中的结果交给模型使用：

```text
用户 query
   ->
读取当前 session 的 compact-blocks.jsonl
   ->
展开 block.evidence
   ->
按 query 做 routing / 过滤 / 排序
   ->
优先命中和当前问题最相关的 evidence
   ->
恢复对应正文
   ->
以 system-reminder 形式注入到本轮上下文
   ->
LLM 基于恢复结果回答
```

当前检索逻辑会优先偏向：

- `sub_agent`
- `tool_result`
- `stack_trace / error / test_output`

### 5.4 当前如何恢复正文

命中 evidence 后，恢复优先级为：

1. `archivedToolResultPath`
2. `toolResultPath`
3. transcript / `messageUuid`
4. `preview`


