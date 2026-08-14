# 流式输出这件事：从 LLM token 到前端逐字的完整链路

> 这是《WinMatrix 开发经验文集》第 4 篇。"流式输出"听起来简单——LLM 吐一个 token，前端显示一个 token，不就完了？真做起来你会发现一堆坑：背压、半断开、重连串流、token 和结构化事件的冲突。这篇讲我们在 WinMatrix 里怎么把这条链路做扎实。

很多人对"流式"的理解停留在这样一个心智模型：

```
LLM --token--> 后端 --token--> 前端
```

像一根水管，水从一头流到另一头。但真实的流式系统远不止于此。一个用户在地铁上看着 AI 逐字打字，中间网络抖了一下、应用切到后台又切回来——这中间发生了什么？后端正在生成的 token，是丢了、重发了、还是接着前面的继续？

这篇文章拆解 WinMatrix 的流式链路，看每一环是怎么处理这些问题的。

---

## 选 WebSocket 还是 SSE？

流式输出的底层协议，最常见的选择是 SSE（Server-Sent Events）和 WebSocket。很多教程会告诉你"SSE 更简单，单向推送够用了"。

我们选了 WebSocket，原因不是"双向"，而是**会话语义**。

在 WinMatrix 里，一个 WebSocket 连接承载的不只是"AI 的流式回答"这一个方向。它还要承载：

- 用户发消息（上行）
- 用户点"停止"（上行控制帧）
- 服务端推送思考过程、工具调用过程、最终回答（下行，多种事件类型）
- 心跳、重连同步（双向控制帧）

SSE 是纯单向的，要支持上述场景就得"SSE 下行 + 另一个 POST 上行"组合，状态管理复杂。WebSocket 的双向全双工，把这些都装进一个连接里，**状态和生命周期是统一的**。

我们的主入口是单一 WS：`/api/v1/agents/chat/stream`。

```typescript
// interface/api/agents/agentWebSocket.ts（第 375-379 行）
export async function agentWebSocketRoutes(server: FastifyInstance) {
  const jwtService = await container.resolve<JwtService>('JwtService');
  const jwtAuth = createJwtAuthMiddleware(jwtService);
  server.get('/chat/stream', { websocket: true, preHandler: [jwtAuth] }, (connection, req) => {
    // ...
  });
}
```

注意 `preHandler: [jwtAuth]`——WebSocket 升级前先过 JWT 认证。别以为 WS 就不用鉴权，升级握手也是 HTTP 请求，照样要走中间件。

**教训：选协议不是看"谁更简单"，而是看你的连接语义。** 单向推送选 SSE，需要双向、多事件类型、控制帧混在一起的会话，选 WS。

---

## 第一个坑：浏览器的"半断开"

TCP 有一个臭名昭著的失败模式叫**半开连接**（half-open）——一端的进程已经死了，另一端还以为连接好好的。在 WebSocket 上，这表现为：用户的手机切到后台、网络断了，但服务端的 `socket.on('close')` 永远不触发，于是这个连接占着资源，直到超时。

很多人用"应用层心跳"来检测——每隔几秒发一个心跳帧，对方不回就认为断了。但这有个问题：浏览器的 WebSocket 实现对应用层心跳的支持参差不齐。

我们的做法是**用服务端原生的 ws ping/pong 帧**：

```typescript
// interface/api/agents/agentWebSocket.ts（第 391-438 行）
// 服务端原生 ping/pong 半断开检测（浏览器自动响应 ping 帧）
const PING_INTERVAL_MS = 30_000;
const PONG_DEADLINE_MS = 10_000;
let pongReceived = true;
if (canPing) {
  rawSocket.on?.('pong', () => { pongReceived = true; clearTimeout(pongTimer); });
  pingTimer = setInterval(() => {
    if (!pongReceived) { rawSocket.terminate(); return; }  // 上一轮未 pong → terminate
    pongReceived = false;
    rawSocket.ping();
    pongTimer = setTimeout(() => { if (!pongReceived) rawSocket.terminate(); }, PONG_DEADLINE_MS);
  }, PING_INTERVAL_MS);
}
```

这个机制的精妙之处：

1. **用协议层的 ping/pong，不是应用层消息**。浏览器的 WebSocket 会自动响应协议层的 ping 帧以 pong，不需要应用代码配合。这绕开了"应用层心跳需要前端配合"的坑。
2. **轮换 + 截止时间**。每 30 秒发一次 ping，但不是"发了就等 30 秒"——而是给上一轮设一个 10 秒的 pong 截止时间。如果上一轮的 pong 10 秒内没回来，立即 `terminate()`。
3. **状态机式的判定**。`pongReceived` 标志位：每轮 ping 前先检查上一轮的标志，如果还是 `false`（上一轮没收到 pong），直接终止。

为什么 30 秒 ping + 10 秒截止？因为我们要在"早发现半断开"和"误杀慢网络"之间找平衡。30 秒一轮，意味着最多 40 秒（一轮的尾巴 + 下一轮的 10 秒截止）能发现一个死连接。对于长连接场景，这个延迟是可接受的。

**教训：半断开是长连接系统的头号隐形杀手。** 应用层心跳依赖前端配合且不可靠；优先用协议层的 ping/pong，让浏览器自动响应。给 pong 设截止时间，而不是无限等待。

---

## 第二个坑：token 到底怎么从 LLM 流到前端

这是流式系统最核心的一环。LLM 在后端逐 token 生成，这些 token 怎么变成前端看到的逐字效果？

我们的链路是 **Emitter（发射器）+ 通道订阅**：

```typescript
// infrastructure/channel/WebSocketChannel.ts（第 196-216 行）
private subscribeToEmitter(emitter: Emitter): void {
  this.unsubscribe = emitter.subscribe((event: StreamEvent) => {
    if (!this.isAlive()) return;
    if (!streamEventTargetsChannel(event, 'websocket')) return;  // 通道过滤
    try {
      const json = JSON.stringify(event);
      this.socket.send(json);   // 直接转发为 WS 文本帧
      this._lastOutputTime = Date.now();
    } catch (error) {
      logger.warn(`[WebSocketChannel] 事件转发失败: ${errMsg}`);
    }
  });
}
```

逻辑很清晰：WebSocketChannel 在构造时订阅 Emitter，每来一个事件，序列化成 JSON，`socket.send` 发出去。

LLM 侧怎么产生事件？通过 Emitter 的领域便捷方法：

```typescript
// infrastructure/observability/spans/Emitter.ts（第 227-237 行）
content = {
  start: (_span?) => { this.hub.stream.content.start(_span); },
  delta: (text, _span?) => { this.hub.stream.content.delta(text, _span); },  // 每个 token delta
  end: (content, _span?) => { this.hub.stream.content.end(content, _span); },
};
```

LLM 每生成一段 token，调 `emitter.content.delta(text)`，这段 text 就变成一个 `content:delta` 事件，被订阅它的 WebSocketChannel 拿到，转发给前端。

这里有几个关键设计：

### 通道过滤：一个事件，多端分发

注意 `streamEventTargetsChannel(event, 'websocket')` 这个判断。同一个事件，可能要同时推给 Web 端（websocket）、企业微信端（wecom）、监控大屏（dashboard）。事件的 `targetChannels` 字段决定它去哪些通道。WebSocketChannel 只转发标记了 `websocket` 的事件。

这意味着**一个 Emitter 可以被多种 Channel 订阅，事件按通道定向分发**。用户在网页上看的回答，可以同时（或异步）通过企业微信推一份给他的同事。

### 发送失败要容错，不能崩

`socket.send` 包在 try/catch 里。如果发送失败（比如 socket 已经关闭但事件还在来），只记个 warn，不让异常向上传播打爆整个流式管线。**流式系统的一个原则：单个事件的发送失败，不能影响后续事件的发送。**

---

## 第三个坑：事件类型不能各搞各的

流式系统里，"事件"远不止"一个 token"。光是我们定义的就有八大类：

```
thinking:start | thinking:delta | thinking:end      （思考过程）
content:start | content:delta | content:replace | content:end  （正文）
tool:start | tool:result:* | tool:end               （工具调用）
agent:start | agent:end | run:start | run:end | run:error  （生命周期）
decision:start | decision:end                       （决策）
plan:init | plan:step | plan:end                    （计划）
span_started | span_event | span_ended              （可观测性 hub）
```

命名规范是 `namespace:action`。所有事件共享一个统一结构：

```typescript
// infrastructure/streaming/types.ts（第 112-135 行）
export interface StreamEvent<T> {
  type: string;
  runId: string;
  seq: number;              // ← 关键：per-conversation 递增的序列号
  conversationId?: string;
  parentTaskId?: string;
  targetChannels?: string[];
  ts: number;
  spanId?: string;
  traceId?: string;
  parentSpanId?: string;
  payload: T;
}
```

为什么要把所有事件塞进一个统一结构？因为**前端要按统一的方式处理它们**。如果 thinking 事件是一种结构、content 事件是另一种结构、tool 事件又是一种，前端的处理代码会变成 if-else 地狱。统一结构 = 统一处理管线。

而且这个文件是**单一真相源**——所有流式事件类型都在 `infrastructure/streaming/types.ts` 定义，前后端共用。前端拿到的事件结构和后端发出的一模一样，不会出现"后端发了 A 前端按 B 解析"。

---

## 第四个坑：seq 防串流

注意上面那个 `seq` 字段。它是**每个会话内递增的序列号**。为什么需要它？

考虑这个场景：用户开了两个标签页，登录的是同一个账号、同一个会话。两个 WebSocket 连接都在收事件。如果两个连接的事件流没有序列号，前端怎么知道哪个事件先、哪个后、有没有丢？

`seq` 解决的就是这个问题。每个会话内，事件按 `seq` 递增。前端拿到事件后，可以检查 `seq` 是否连续——如果有跳号，说明中间丢了事件，可以请求重发或重新同步。

这比靠时间戳可靠得多——时间戳在不同机器上有时钟漂移，而 `seq` 是逻辑单调递增的。

**教训：任何"一个会话内的事件流"都需要一个单调递增的序列号。** 它是检测丢失、乱序、重连同步的基础。时间戳不够，时钟漂移会咬你。

---

## 第五个坑：重连之后，怎么接上之前的进度

这是流式系统里最难的坑。用户网络断了 10 秒，这 10 秒里 LLM 又生成了 50 个 token。重连后，前端怎么拿到这 50 个 token？是重头开始、还是从断点续传？

我们的方案是**一个专用的重连控制帧：`connection:sync`**：

```typescript
// interface/api/agents/ws/connectionSyncHandler.ts（第 35-134 行）
export async function tryHandleConnectionSync(
  message, socket, authenticatedUserId, registerPushForConversation?,
): Promise<boolean> {
  // 重连 sync 帧：
  // 1. 批量鉴权
  // 2. 为授权会话重新注册 push registry（在状态查询之前，避免 terminal 竞态）
  // 3. 批量读 canonical 状态
  // 4. 返回 connection:sync_result
}
```

这个流程有几个讲究：

### 鉴权失败的会话，绝不注册 push

重连时要重新鉴权该用户对这个会话的权限。鉴权失败的会话，**绝对不能**给它注册 push 回调。否则就是越权——一个已经没权限的用户，重连后还能继续收这个会话的事件。

### 先注册 push，再读状态（顺序很重要）

注意步骤 2 和 3 的顺序：**先为授权会话注册 push registry，再去读 canonical 状态**。这个顺序不能反。

为什么？想象反过来：先读状态，再注册 push。如果在这两步之间，恰好这个会话产生了 terminal 事件（最终结束），那么：
- 你读到的状态是"还没结束"
- 但 terminal 事件已经发了，而你还没注册 push，收不到
- 于是前端永远等不到结束信号

**先注册 push 再读状态，保证你不会错过注册窗口期内的任何事件。** 这是分布式系统里经典的"注册-读取顺序"问题。

### sync 帧有 schema 版本

重连协议本身也要演进。`SYNC_SCHEMA_VERSION` 让前后端能协商一致的 sync 协议版本，避免老客户端用新协议、或反过来。

### sync 帧绕过限流

注意 `connection:sync` 是**绕过 agentChatLimiter**（对话并发限制器）的。因为 sync 不是一次新的对话，只是恢复一个已有的会话状态——它不该占用对话并发配额。

**教训：重连同步是流式系统最容易出 bug 的地方。** 它的本质是"在一个分布式状态机上做断点续传"。关键三点：鉴权要重新做、注册要在读取之前、协议要带版本。

---

## 第六个坑：流式输出的"结尾"必须可观测、可持久化

流式系统里，"开头"很容易——发第一个 token 就行。但"结尾"很难：什么时候算这个回答真的结束了？是最后一个 token 发完？还是某个结束事件？

如果不把"结尾"定义清楚，会出现各种诡异问题：前端永远转圈、同一个回答被存两次、停止按钮一直亮着。

我们的做法是**用一个统一的 terminal 事件收口，且必须持久化成功后才发布**。一个 Turn 结束时，根据不同结局发布不同的 terminal：

| 结局 | terminal 类型 |
|------|--------------|
| 正常完成 | `buildCompletedTerminal` |
| 用户手动停止 | `buildStoppedTerminal` |
| 执行失败 | `buildFailedTerminal` |
| 等待用户输入 | `buildWaitingUserInputTerminal` |

terminal 是**根回合的唯一收口点**——一个 Turn 只会发布一个 terminal。而且发布的前提是**持久化已经成功**。如果先发 terminal 再持久化，持久化失败时用户已经看到"完成了"，但数据没存下来，就是数据丢失。

**教训：流式系统的"结尾"必须是一个明确的、唯一的、持久化后才发布的事件。** 模糊的结尾定义，是前端体验和数据一致性问题的共同来源。

---

## 一条完整的 token 流转链

把上面六点串起来，一个 token 从 LLM 到前端的完整旅程：

```
正常流转：
  LLM --emitter.content.delta("你")--> Emitter
  Emitter --content:delta 事件(带 seq)--> WebSocketChannel
  WebSocketChannel --streamEventTargetsChannel(websocket)?是--> socket.send(JSON)
  Socket --WS 文本帧--> 前端
  （前端检查 seq 连续性）

  LLM --emitter.content.delta("好")--> Emitter --> ... --> 前端

断线重连：
  [网络抖动，连接断开 5 秒]
  前端 --重连,发 connection:sync 帧--> Socket
  Socket --tryHandleConnectionSync--> WebSocketChannel
  WebSocketChannel 内部：批量鉴权 → 注册 push(先) → 读状态(后)
  WebSocketChannel --connection:sync_result(含漏掉的事件)--> 前端
  前端：按 seq 补齐缺失 token，继续渲染
```

---

## 给后来者的几条总结

1. **选协议看连接语义，不看简单程度。** 单向用 SSE，双向会话用 WS。
2. **半断开用协议层 ping/pong 检测**，别依赖应用层心跳。给 pong 设截止时间。
3. **token 流转用 Emitter + 通道订阅**。一个 Emitter 可被多通道订阅，事件按 `targetChannels` 定向分发。
4. **事件类型统一结构、单一真相源**。前后端共用一份类型定义，命名遵循 `namespace:action`。
5. **每个会话内的事件流要有单调递增的 seq**。这是检测丢失、乱序、重连同步的基础，时间戳不够。
6. **重连同步是断点续传，三个关键点**：重新鉴权、注册在读取之前、协议带版本。
7. **结尾必须是唯一、明确、持久化后才发布的 terminal 事件**。模糊的结尾是体验和数据问题的共同来源。

流式输出是 AI 产品最直观的"体感"来源——它顺不顺滑，用户第一时间感知到。而顺滑的背后，是半断开检测、事件统一、seq 防串流、重连同步这些"看不见"的工程。把这些做扎实，你的 AI 产品才算真的"活"了。

---

> **下一篇**：[《工具调用不止 function calling：企业级工具治理怎么做》](./05-tool-governance.md)——LLM 想调一个工具，凭什么让它调？企业场景下的工具治理怎么做。
