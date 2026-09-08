# D1 下午笔记:agent 包精读

## 站点 1:`packages/agent/src/types.ts`(~560 行,全读)

### 必须抓住的 6 个类型(按行号)

| 行号 | 类型 | 一句话 |
|---|---|---|
| 28 | `StreamFn` | 唯一的 LLM 入口。契约:**不许 throw**,错误编码进 stream 的 stopReason |
| 351 | `AgentMessage` | `Message \| CustomAgentMessages[keyof CustomAgentMessages]` —— 联合类型 + 声明合并实现扩展 |
| 342 | `CustomAgentMessages` | 空接口,app 用 declaration merging 塞自己的消息类型 |
| 361 | `AgentState` | systemPrompt / model / thinkingLevel / tools / messages + 4 个只读运行时字段 |
| 416 | `AgentTool` | label + execute(throw 报错)+ 可选 executionMode 覆盖 |
| 461 | `AgentEvent` | 10 个字面量事件的 discriminated union |

### 值得停下来的设计细节

1. **`AgentMessage` 扩展机制**(351):不是继承,是 `CustomAgentMessages` 空接口 + 声明合并。app 侧 `declare module` 加字段,联合类型自动扩大。零运行时成本。
2. **错误处理约定统一**:所有 hook(`convertToLlm`/`transformContext`/`getApiKey`/...)的 JSDoc 都写"must not throw or reject"。循环不 try/catch 你,你不守约就破坏事件序列。约定代替防御。
3. **`AgentState` 用 getter/setter**(361):赋值数组时内部做 slice 拷贝,防外部改引用。见 `agent.ts` 的 `createMutableAgentState`。
4. **`AgentTool.execute` 契约**:"Throw on failure instead of encoding errors in content" —— 失败靠异常,结果里只放正常内容。错误转译由循环统一做。
5. **`BeforeToolCallResult.terminate`**(64):批量早停是"全部工具结果都设 terminate 才停",不是有一个就停。

### 自测问题

- [x] UI-only 消息(比如一个卡片)应该放 `Message` 还是 `AgentMessage`?谁负责在 LLM 调用前把它过滤掉?
- [x] `beforeToolCall` 返回什么可以拦截一次工具调用?
- [x] `StreamFn` 为什么禁止 throw?

---

## 站点 2:`packages/agent/src/agent-loop.ts`(重点 140-350 行)

### 主循环骨架(`runLoop`,144 行起)

双层 while:

```
外层 while(true)          ← follow-up 消息让它再跑
  内层 while(有工具调用 || 有pending消息)   ← 工具调用让它再跑
    turn_start
    注入 pendingMessages(steering)
    streamAssistantResponse()      ← 唯一 LLM 调用点
    stopReason 是 error/aborted → turn_end + agent_end,退出
    有 toolCall → executeToolCalls() → 结果 push 回 context
    turn_end
    prepareNextTurn() 可换 context/model
    shouldStopAfterTurn() → true 则退出
    取 getSteeringMessages() 作为下一轮 pending
  外层取 getFollowUpMessages(),非空则继续
agent_end
```

### `streamAssistantResponse`(325 行起)——边界转换发生地

```
transformContext(messages)   → AgentMessage[]   (剪枝/注入)
convertToLlm(messages)      → Message[]        (过滤/转换)
llmContext = { systemPrompt, messages, tools }
streamFn(model, llmContext) → 事件流
  start        → push partial,发 message_start
  *_delta 等   → 替换末尾 partial,发 message_update
  done/error   → 换成 final,发 message_end,return
```

**注意**:partial message 一开始就 push 进 `context.messages`,流式过程中不断原地替换末尾元素。UI 和 context 共享同一条消息轨道。

### 一个精细防御(332 行注释)

`stopReason === "length"`(输出被 token 限流截断)时,该消息里所有 toolCall 的参数可能被截断——即使 JSON salvage 解析通过也不可信,所以**全部直接判失败**并让模型重发。这是个值得学的教训:截断响应上的工具调用绝不能执行。

### 自测问题

- [x] steering 和 follow-up 的注入时机差在哪?
- [x] 一次 `prompt()` 最多产生多少次 LLM 调用?由什么控制?
- [x] 工具结果(toolResult)是在事件序列的哪个点回灌进 context 的?

### 答案(2026-09-03 过站点 1 后精读确认)

1. **steering vs follow-up**:steering 在内层循环注入——当前 turn 的工具执行完、`turn_end` 发出后,`getSteeringMessages()` 的结果作为下一轮 pending,在下一次 LLM 调用**前**注入(用户在 agent 工作中途插话)。follow-up 在外层循环注入——agent 本来已经要停了(无工具调用、无 steering),`getFollowUpMessages()` 有货就继续跑(用户在 agent 收尾时排队的新任务)。
2. **LLM 调用次数 = 内层循环次数**,由"本消息是否含 toolCall"控制:有工具调用就再转一圈,没有就停。加上外层 follow-up,理论上无上限;正常单次 prompt = 1 次 LLM 调用 + N 次工具回合。
3. **toolResult 回灌点**:`executeToolCalls*` 返回 messages 后,在 `turn_end` **之前** push 进 `currentContext.messages`(runLoop 189-192 行)。即下一轮 LLM 调用时模型已能看到工具结果。

### 工具执行管线(442 行起,executeToolCalls)

四段式,每段都是显式命名函数:

```
prepareToolCall()          → 查表/prepareArguments/validateToolArguments/beforeToolCall 拦截
  ├─ kind: "immediate"    → 一步到位的错误结果(找不到工具/被 block/abort/校验抛异常)
  └─ kind: "prepared"     → 进执行段
executePreparedToolCall()  → 调 tool.execute,收集 tool_execution_update 流式回调
finalizeExecutedToolCall() → afterToolCall 钩子,逐字段合并覆盖(无深合并)
emitToolExecutionEnd()     → 发事件,createToolResultMessage
```

两种执行模式(`executeToolCalls` 447 行分派):

- **sequential**:一个接一个,`tool_execution_end` 按源顺序
- **parallel**(默认):**preflight 顺序、执行并发**。`finalizedCalls` 数组里混放已定局的 outcome 和返回 Promise 的闭包,`Promise.all` 统一收口。完成顺序 = 实际执行快慢,但 toolResult 消息仍按 assistant 源顺序发
- 判定规则(451 行):全局 `toolExecution` 配置 OR 任一工具声明 `executionMode: "sequential"` → 整批降级为 sequential

关键防御细节:

- `validateToolArguments` 在 beforeToolCall **之前**,hook 拿到的是已验证参数
- `executePreparedToolCall` 里 `acceptingUpdates` 标志:工具 promise 定局后到达的迟到 update 被丢弃,防止流式回调撕裂事件序列
- 工具 throw → `createErrorToolResult(error.message)`,isError: true——异常在这里被转译,不向上冒
- `shouldTerminateToolBatch`(620 行):`every(terminate === true)`,印证 types.ts 里"全部才停"的约定

### 自测问题(站点 2)

- [x] parallel 模式下 `tool_execution_end` 事件顺序和 toolResult 消息顺序为什么不一致?各自按什么排?
- [x] beforeToolCall 被调用时,工具参数是否已经过 schema 校验?
- [x] 一个不存在的工具名会被怎样处理?走哪条路径?

---

## 站点 3:`packages/agent/src/agent.ts`

只看三件事:

1. `defaultConvertToLlm`(31 行):就一行 filter,只留 user/assistant/toolResult——**默认实现即文档**
2. `createMutableAgentState`(76 行):getter/setter 拷贝数组
3. `Agent` 类如何把 `AgentLoopConfig` 的每个 hook 原样接出来

---

## 进度

- [ ] 站点 1:types.ts
- [ ] 站点 2:agent-loop.ts
- [ ] 站点 3:agent.ts
- [ ] 画时序图
