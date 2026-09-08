# Pi Agent 两天学习计划

目标:理解一个开源 coding agent 的核心架构与扩展体系,能亲笔写出可运行的扩展。

原则:只读 6 个文件 + 动手跑 2 次,其余文档扫一眼即可。

---

## Day 1:Agent 循环(核心中的核心)

### 上午:跑起来 + 建立直觉(2h)

- [x] `npm install --ignore-scripts` && `./pi-test.sh`,问它一个问题,观察工具调用
- [x] 读完 `packages/coding-agent/README.md` 的 Philosophy 一节(记住"不做清单":No MCP / No sub-agents / No permission popups / No plan mode / No built-in to-dos / No background bash)

### 下午:精读 agent 包(4h)

| 顺序 | 文件 | 要回答的问题 |
|---|---|---|
| 1 | `packages/agent/src/types.ts` | `AgentMessage` 和 `Message` 差在哪?`StreamFn` 契约是什么? |
| 2 | `packages/agent/src/agent-loop.ts` | 一 turn 内事件顺序?工具结果如何回灌给 LLM?什么时候循环结束? |
| 3 | `packages/agent/src/agent.ts` | `Agent` 类如何包住 loop?`convertToLlm`/`transformContext` 插在哪? |

### Day 1 检验标准

- [x] 能画出 `prompt()` 从提交到回复的完整事件时序图
- [x] 能说清"为什么只在 LLM 边界做类型转换"(内部表示宽松,边界收敛)

**D1 完成。关键学到的:**
- AgentMessage vs Message 的边界转换
- 双层 while 的 steering/follow-up 注入机制
- 工具执行四段管线(prepare/execute/finalize/emit)
- acceptingUpdates 防止迟到事件撕裂序列

---

## Day 2:产品层(工具 + 扩展)

### 上午:工具实现(3h)

- [x] `packages/coding-agent/src/core/tools/bash.ts`:理解流式 update、truncate、output guard
- [x] `packages/coding-agent/src/core/tools/read.ts`:对比 bash 的更简单实现
- [ ] 跳读 `tools/index.ts` 看四个工具如何在 coding-agent 里组装

### 下午:扩展系统(3h)

| 顺序 | 文件/材料 | 要回答的问题 |
|---|---|---|
| 1 | `packages/coding-agent/examples/extensions/` 挑 1 个最短示例 | 一个扩展的最小形态? |
| 2 | `src/core/extensions/types.ts` 只读 `ExtensionAPI` 接口(1252 行起) | API 面有哪些能力域? |
| 3 | `src/core/extensions/loader.ts` 只看 `VIRTUAL_MODULES` | 单二进制怎么让扩展 import 宿主模块? |
| 4 | 亲笔写一个扩展:注册 `/hello` 命令 + 一个返回固定字符串的 tool | 能不能装上、被模型调用? |

### Day 2 检验标准

你的扩展被 pi 加载并完成一次工具调用。

---

## 明确跳过的(第二遍再看)

harness 分层(`agent/src/harness/`)、session JSONL 树、compaction、tui、ai 包的 provider 抽象。这些不影响理解主线。

---

## 附录 A:架构速记

分层依赖链(build 顺序即依赖顺序):

```
tui → telemetry → ai → agent → session-backends → protocol → client → server → coding-agent
```

核心数据流:

```
AgentMessage[] ──transformContext()──▶ AgentMessage[] ──convertToLlm()──▶ Message[] ──▶ LLM
                  (剪枝/注入上下文)                        (过滤 UI 消息)
```

事件序列:

```
agent_start → turn_start → message_start → message_update(流式) → message_end
→ [tool_execution_start → update → end] → turn_end → (有工具调用则再来一 turn) → agent_end
```

品味要点:

1. 运行时(`agent`)和产品(`coding-agent`)分离,循环被锁在极小的纯库里
2. 内部表示宽松(`AgentMessage`),只在 LLM 调用边界收敛(`Message`)
3. 一切皆事件,UI/遥测/扩展订阅同一套事件流
4. 一切皆扩展:prompts / skills / themes / extensions 四层,分发靠 `pi install npm:` 或 `git:`
5. 反功能堆砌:core 最小,工作流选择权交给用户

## 附录 B:扩展最小骨架

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (ctx) => {
      ctx.ui.notify("hello from extension");
    },
  });

  pi.registerTool({
    name: "greet",
    description: "Returns a greeting",
    parameters: { type: "object", properties: { name: { type: "string" } } },
    execute: async (args) => ({ content: [{ type: "text", text: `hi ${args.name}` }] }),
  });
}
```

放置位置:`~/.pi/agent/extensions/`(全局)或 `.pi/extensions/`(项目)。
