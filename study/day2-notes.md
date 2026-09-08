# D2 笔记:工具实现 + 扩展系统

## 概览:为什么要读工具

D1 学的是循环的**形**(事件序列、manage、steering);D2 要看**肉**(工具怎么实现、扩展怎么装载)。

工具是模型直接调用的东西,也是产品跟操作系统交互的边界。两个工具对比:

- **bash.ts**:复杂、有流式 update(throttle)、有 truncate 防御、有 abort 清理——**生产级实现样板**
- **read.ts**:简单、同步读文件——工具的最小形态

通读这两个,你就能写自己的工具。

---

## 站点 1:`packages/coding-agent/src/core/tools/bash.ts`

### 要回答的问题

- [x] execute 为什么有三个参数 `(command, timeout?, ctx)`?ctx 用来干什么?
- [x] onUpdate 如何做节流(throttle)?为什么需要节流?
- [x] acceptingOutput 和 updateDirty/updateTimer 三个状态变量协作做了什么?
- [x] 工具返回的 details 里包的 truncation/fullOutputPath 是什么?

### 关键概念

**流式 update(line 362-392)**

bash 命令执行时,进程的 stdout 实时到达。工具不能每个字节都 call onUpdate——会**爆事件**。所以:

```
handleData → output.append() + setDirty = true
scheduleOutputUpdate() → 不是立即 emit,而是 setTimeout(throttle)
隔 BASH_UPDATE_THROTTLE_MS(150ms) 再 emit 一次
```

查看行 362-407,重点看 `scheduleOutputUpdate` 和 `emitOutputUpdate` 的逻辑。

**Truncate 防御(OutputAccumulator)**

bash 输出可能巨大(GB 级日志)。工具需要自己做 truncate,不能让 context 溢出。查 `core/tools/output-accumulator.ts` 的逻辑(当前笔记暂不深入,知道存在即可)。工具返回时带 `truncation: { truncated, maxBytes, byteCount }` 信息。

**Abort 清理(finishOutput)**

进程中断或超时时,pending 的 setTimeout 必须被清掉,否则下一行(迟到的 emit)会在事件序列里撕洞。

### 自测问题

- [x] 如果删掉 `acceptingOutput` 的所有检查,会发生什么?(Hint:abort 后残留数据)
- [x] 工具返回的结果里为什么要包 `fullOutputPath`?(Hint:UI 需要知道完整日志在哪)
- [x] 为什么 `clearUpdateTimer` 要放在 `finishOutput` 里?

---

## 站点 2:`packages/coding-agent/src/core/tools/read.ts`

### 要回答的问题

- [x] read 的 execute 参数是什么?
- [x] 为什么 read 没有 onUpdate?
- [x] 读文件错误怎么处理?throw 还是返回?

### 对比 bash

- bash:异步、有流式、需要节流、可能中断
- read:同步、一次到位、返回内容或错误

工具契约的灵活性:同一个 `AgentTool<T>` 既能支持流式,也能支持同步,onUpdate 可选。

### 自测问题

- [ ] 如果 read 的文件路径超出 cwd 沙箱,怎么防御?(看代码里有没有 resolveProjectPath)

---

## 站点 3:`packages/coding-agent/src/core/tools/index.ts`

### 要回答的问题

- [ ] 四个工具(read/write/bash/edit)在 index.ts 怎么组装成 AgentTool 数组?
- [ ] 有没有条件性打开某个工具(比如 Windows 上的 powershell)?

### 跳读要点

找到 export 的主函数,看它怎么:

- 创建每个工具对象
- 传入配置(cwd、tempFilePrefix 等)
- 返回工具数组给 Agent

---

## 进度

- [ ] bash.ts
- [ ] read.ts
- [ ] index.ts
- [ ] 能回答上面所有自测问题
