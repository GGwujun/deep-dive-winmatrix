# 企业微信双轨接入：长连接 AiBot + Webhook

> 这是《WinMatrix 开发经验文集》第 21 篇。讲一个企业级 AI 平台绕不开、但官方文档讲不清楚的话题：怎么把 AI 接进企业微信？群里 @ 机器人能响应、单聊能对话、创建的文档能在企微里打开——这些场景背后要接几条链路？代码来自 WinMatrix 后端真实实现。

如果你做过企业微信集成，大概率被它的接入方式绕晕过。企微开放平台给了好几种消息接收方式——Webhook 回调、应用消息、智能机器人（AiBot）长连接——每种的能力边界、配置方式、消息格式都不一样。很多人第一版接的是 Webhook，跑起来发现"群聊里 @ 机器人收不到消息"——因为 Webhook 只能接收特定类型的消息。换成 AiBot 长连接，又发现"主动推消息给用户"走不通——因为长连接是双向会话，不能任意群发。

WinMatrix 跑在生产里，这些场景全都要支持：群里 @ 机器人要响应、单聊要能对话、定时任务要能主动推消息到群、AI 创建的微文档要能在企微里打开。我们的解法是**双轨接入**——长连接 AiBot 和 Webhook 并存，各管一摊。这篇讲这两条轨道怎么分工、怎么把不同来源的消息统一到同一条处理管线下。

---

## 两条轨道：长连接 AiBot + Webhook

先看全景。WinMatrix 的企微接入有两条并行的轨道：

```
┌──────────────────────────────────────────────────────────┐
│                      企业微信                              │
│                                                          │
│   ┌──────────────┐              ┌──────────────┐         │
│   │  AiBot 长连接 │              │   Webhook    │         │
│   │  （双向 WS）   │              │ （单向 HTTP） │         │
│   └──────┬───────┘              └──────┬───────┘         │
└──────────┼──────────────────────────────┼────────────────┘
           │                              │
           ▼                              ▼
   WsFrame → WeChatMessage        HTTP POST → WeChatMessage
           │                              │
           └──────────┬───────────────────┘
                      ▼
           WeChatMessageService.processMessage()
              （统一的处理管线）
                      │
                      ▼
              AI 响应 / 工具执行 / 流式输出
```

- **长连接 AiBot**：通过企微 SDK 建立一条 WebSocket 长连接，企微把 @ 机器人的消息、单聊消息、进群事件等推过来。双向，能接收也能主动发。
- **Webhook**：企微在特定事件（比如群聊消息）发生时，POST 一个请求到我们的 HTTP endpoint。单向，只能接收，回包就是响应。

为什么要两条并存？因为它们的能力边界互补：

| 能力 | AiBot 长连接 | Webhook |
|------|-------------|---------|
| 接收 @ 机器人消息 | 支持 | 部分支持 |
| 单聊对话 | 支持 | 不支持 |
| 主动推送消息 | 支持（双向） | 不支持（单向） |
| 群聊事件 | 支持 | 支持 |
| 配置复杂度 | 高（要建机器人、配密钥、建长连接） | 低（一个 URL） |
| 运维成本 | 高（长连接要维护、重连） | 低（无状态 HTTP） |

简单场景用 Webhook 就够了，复杂场景（对话、主动推、双向交互）必须上 AiBot 长连接。两条并存，让用户按需选择。

**教训：不要指望一条接入链路覆盖所有场景。** 企业级 IM 平台的接入方式是按"场景"分的，不是按"技术先进度"分的。长连接和 Webhook 不是"替代"关系，而是"分工"关系。把它们设计成可并行的双轨，比纠结"选哪个"务实得多。

---

## 长连接轨道：WsFrame 转统一消息格式

AiBot 长连接的消息，进来时是企微 SDK 的 `WsFrame` 格式——一套企微自己的数据结构，包含 headers、body、event type 等。这套格式和 WinMatrix 内部的消息模型（WeChatMessage）不一样。

如果让每个处理消息的地方都直接面对 WsFrame，整个系统就会被企微 SDK 的数据结构绑死。我们的做法是加一层桥接——`WeComAiBotMessageBridge`，把 WsFrame 转成统一的 WeChatMessage：

```typescript
// interface/channel/channels/wecom-aibot/aibot/WeComAiBotMessageBridge.ts（第 1-14 行，文件头）
/**
 * 企微智能机器人长连接消息桥接
 *
 * 将 SDK 的 WsFrame（aibot_msg_callback / aibot_event_callback）转为
 * 已有的 WeChatMessage 格式，复用 WeChatMessageService.processMessage() 管线。
 */
```

核心的转换逻辑在 `handleMessage`：

```typescript
// WeComAiBotMessageBridge.ts（第 56-78 行）
async handleMessage(frame: WsFrame<BaseMessage>): Promise<void> {
  const body = frame.body;
  if (!body) return;

  try {
    logger.info({
      botId: body.aibotid,
      msgid: body.msgid,
      reqId: frame.headers.req_id,
      chattype: body.chattype,          // 'single' | 'group'
      chatid: body.chatid ?? null,
      fromUserid: body.from.userid,
      msgtype: body.msgtype,
    }, '[AiBotBridge] 收到消息回调');

    const content = this.extractTextContent(body);

    await this.options.onProcessMessage({
      digitalEmployeeId: this.client.digitalEmployeeId,
      botId: body.aibotid,
      msgid: body.msgid,
      content,
      fromUserid: body.from.userid,
      chatid: body.chatid,
      chattype: body.chattype,
      reqId: frame.headers.req_id,
    });
  } catch (error) {
    logger.error(`[AiBotBridge] 处理消息失败: msgid=${body.msgid}, error=${getErrorMsg(error)}`);
  }
}
```

注意几个设计：

1. **转换后的 `onProcessMessage` 回调，参数是一个扁平化的结构**（digitalEmployeeId、botId、msgid、content、fromUserid、chatid、chattype）。这正是 Webhook 轨道也能产出的结构——两条轨道最终都汇入同一个 `processMessage` 管线。

2. **`chattype` 区分单聊（single）和群聊（group）**。这是个关键字段——单聊和群聊的处理逻辑完全不同（群聊要判断是不是 @ 了机器人、要从群里解析项目上下文）。

3. **错误只记 log 不抛**。`handleMessage` 里的 try/catch 把异常吃掉了。为什么？因为这是长连接的消息回调，如果抛异常，可能影响 SDK 的连接状态或后续消息接收。单条消息处理失败不该拖垮整条长连接。

4. **reqId 贯穿**。`frame.headers.req_id` 是企微给每条消息的请求 ID，转换后保留，用于全链路追踪。

**教训：不同来源的消息，要尽早转换成统一的内部格式。** 系统的后续处理（决策、工具调用、AI 响应）不该关心"这条消息是 AiBot 来的还是 Webhook 来的"。Adapter 层负责把外部协议翻译成内部模型，后续一切基于内部模型——这是经典的"防腐败层"（Anti-Corruption Layer）模式。

---

## 群聊自动发现：先收着，再补全映射

企微接入里有个很现实的痛点：**怎么知道一个群聊对应哪个项目？**

企微的群聊没有"项目"概念，它只有一个 `chatId`。但 WinMatrix 的所有 AI 能力都是围绕"项目"组织的——知识库挂在项目下、数字员工归属项目、文档存在项目空间。一条群消息进来，如果不知道它是哪个项目的，AI 就没法用项目的知识库和上下文去响应。

常规做法是：管理员手动建一个"chatId → projectId"的映射表。但这个做法有个鸡生蛋问题——管理员怎么知道群的 chatId？企微不暴露群名到 chatId 的查询接口，你只有群聊发消息时才能拿到 chatId。

WinMatrix 的解法是**群聊自动发现**：先收着，再补全映射。看数据模型：

```prisma
// prisma/schema.prisma（第 1112-1122 行，已配置的群聊映射）
model wechat_chat_mappings {
  chatId            String  @id @map("chat_id")
  projectId         String  @map("project_id")
  webhookUrl        String? @map("webhook_url")
  projectWebhookUrl String? @map("project_webhook_url")
}

// prisma/schema.prisma（第 1124-1138 行，已发现但未配置的群聊）
/// 已发现但未完成配置的企微群聊（用于自动获取群聊ID）
model discovered_wechat_chats {
  chatId       String  @id @map("chat_id")
  /// 关联的项目ID，未绑定项目时为 null
  projectId    String? @map("project_id")
  /// 最后活跃时间（收到消息时更新）
  lastSeenAt   String  @map("last_seen_at")
  /// 首条消息内容，作为群聊识别提示
  firstMessage String? @map("first_message")
}
```

两张表分工：

- **`wechat_chat_mappings`**：已配置的群聊。chatId → projectId 绑定完成，AI 会正常响应。
- **`discovered_wechat_chats`**：已发现但未配置的群聊。一个群发消息进来，但 `wechat_chat_mappings` 里没它的映射，就把它记到 `discovered_wechat_chats` 里，存上 chatId、首条消息内容（作为识别提示）、最后活跃时间。

这套机制的流程：

```
群消息进来
    │
    ▼
查 wechat_chat_mappings 有没有这个 chatId？
    │
    ├── 有 → 正常处理（AI 响应）
    │
    └── 没有 → 写入/更新 discovered_wechat_chats
                    │
                    ▼
              管理员在 UI 看到"这些群发了消息但还没绑定项目"
                    │
                    ▼
              管理员补全 chatId → projectId 映射
                    │
                    ▼
              下次这个群的消息就能正常处理了
```

`firstMessage` 字段尤其聪明——管理员面对一堆 chatId 是懵的（chatId 是一串无意义的字符串），但看到"首条消息内容"就能认出"哦这是张三拉的那个项目群"。用内容辅助识别，比给一串 UUID 让人猜强太多。

**教训：当外部系统的标识符（chatId）对你不可见、不可控时，用"发现 + 人工补全"的半自动流程。** 别指望全自动化——有些映射（群到项目）本质上需要人判断，系统要做的是"把需要人判断的东西清晰地呈现出来"，而不是假装能自动猜对。

---

## 微文档登记：docid vs pad_id 的一字之差

企微生态里有个很隐蔽的坑：**微文档（Wedoc）有两种 ID——API 返回的 docid 和浏览器地址栏里的 pad_id**。

AI 在企微里帮用户创建一份微文档，创建接口（`create_doc`）返回的是 `docid`。用户点开文档链接，浏览器地址栏里显示的是 `pad_id`。这两个 ID 不一样！如果你只存了 docid，后续想程序化访问这份文档（读内容、更新），用 docid 是可以的；但如果用户把浏览器地址栏的链接发给你，你想从链接里解析出文档、和你的记录对上，你对的是 pad_id，对不上。

这听起来是个小事，但它会让"AI 创建的文档"和"用户后来分享的文档"对不上号——AI 以为创建了文档，但用户分享的是另一个 ID，AI 不认识，无法继续操作。

WinMatrix 的解法是**专门建一张表登记，且明确记录的是 docid**：

```prisma
// prisma/schema.prisma（第 1775-1793 行）
model WecomConversationWedocDoc {
  id                String  @id @default(cuid())
  conversationId    String  @map("conversation_id")
  docid             String  @map("docid")          // API 返回的 docid，非浏览器 pad_id
  url               String?
  docName           String  @map("doc_name")
  docType           String  @map("doc_type")
  projectId         String? @map("project_id")
  digitalEmployeeId String? @map("digital_employee_id")
  agentId           String? @map("agent_id")

  @@unique([conversationId, docid], map: "uq_wecom_conv_wedoc_doc")
}
```

注释里那句"API docid；非浏览器 pad_id"是核心。这张表登记的是**程序化访问用的 docid**，不是浏览器地址栏的 pad_id。这样后续 AI 要读/写这份文档，用存下来的 docid 调 API 就行。同时 `url` 字段存了文档的可访问链接，用户点开能看。两条路径（程序访问 + 人类访问）各取所需。

**教训：同一个实体在不同接口里可能有不同的标识符，务必搞清楚"我存的是哪个 ID、它支持什么操作"。** docid 支持 API 操作，pad_id 是浏览器友好。存错了 ID，后续操作全对不上。这种"一字之差"的坑，不看源码注释、不看官方文档的细节，根本发现不了。

---

## 绑定状态机：五态 + 配对码

AiBot 长连接的建立不是"配好密钥就能连"，而是要经过一套绑定流程：创建机器人 → 配置密钥 → 测试连接 → 配对到具体用户 → 正常运行。中间任何一步出问题，都可能卡在某个中间态。

WinMatrix 用一个**五态状态机**管理这个流程：

```typescript
// UserChannelRegistrationRepository.ts（第 34 行）
export type WecomBindingStatus = 'configured' | 'testing' | 'paired' | 'error' | 'disabled';
```

```
configured → testing → paired（正常工作）
                │           │
                ▼           ▼
              error       disabled
```

- **configured**：密钥已配置，还没测试过连接
- **testing**：正在测试连接是否通畅
- **paired**：已配对到具体企微用户，长连接正常工作
- **error**：某步出错了（密钥错、连接失败）
- **disabled**：手动禁用

每个状态对应不同的运维动作。而密钥（secret）不是明文存的：

```typescript
// UserChannelRegistrationRepository.ts（第 36-49 行）
export interface UserWecomAibotBindingRecord {
  id: string;
  ownerUserId: string;
  botId: string;
  secretCiphertext: string;        // 密文存的 secret
  pairedUserid: string | null;
  status: WecomBindingStatus;
  pairingCodeHash: string | null;  // 配对码的 hash（不存明文）
  pairingExpiresAt: Date | null;   // 配对码过期时间
  lastError: string | null;
  lastConnectedAt: Date | null;
}
```

注意 `secretCiphertext`（密文）和 `pairingCodeHash`（hash）。secret 用对称加密存（需要还原成明文去建连），配对码用单向 hash 存（只需要验证"用户输入的配对码对不对"，不需要还原）。这是密码存储的基本原则——**需要还原的用加密，只需验证的用 hash**。

配对码（pairing code）是用来"确认这个绑定确实是这个用户操作"的凭证，有过期时间（`pairingExpiresAt`）。用户在企微侧发起配对，拿到一个临时配对码，在 WinMatrix 里输入这个码完成绑定。过期了就得重新生成。

**教训：涉及外部系统凭证的绑定流程，要用显式状态机管理。** 别搞成"配了就能用"的二元状态——中间有太多可能出错（密钥错、网络不通、权限不够、配对超时）。每个状态对应清晰的运维含义，出问题时一眼能看出"卡在哪一步"。凭证存储遵循"需还原用加密、需验证用 hash"。

---

## 企微 API 按服务拆分

除了消息收发，WinMatrix 还要调用企微的各种 API——发应用消息、查通讯录、传文件、管理日程、操作微文档。这些 API 我们没有塞进一个大类，而是**按服务拆成 7 个独立封装**：

```
integration/connectors/wechat/
├── WeComAccessTokenProvider.ts   # token 管理（所有 API 的基础）
├── WeComAppMessageService.ts     # 应用消息推送
├── WeComContactService.ts        # 通讯录
├── WeComFileService.ts           # 文件操作
├── WeComMediaService.ts          # 媒体素材
├── WeComScheduleService.ts       # 日程
├── WeComWedocService.ts          # 微文档
└── constants.ts
```

为什么拆这么细？因为企微的各个 API 域是**正交**的——发消息的不需要懂通讯录，管微文档的不需要懂日程。塞一个大类里，任何一个域的改动（比如企微更新了微文档 API）都要动这个大类，冲突风险高。拆开后，每个服务独立演进，改微文档 API 只动 `WeComWedocService`。

注意有个基础的 `WeComAccessTokenProvider`——企微所有 API 调用都要带 access_token，这个 token 是共享的（一个企业一套）。单独抽出来，统一管理 token 的获取、缓存、刷新，避免 7 个服务各自去取 token 造成重复请求和配额浪费。

**教训：第三方 API 的封装要按"业务域"拆，不是按"调用频率"或"实现难度"拆。** 业务域是正交的拆分维度，改一个域不影响其他域。共享的基础能力（如 token 管理）单独抽出，避免 N 个服务各搞一套。

---

## 双轨汇合：统一的处理管线

回到最开始那张图——两条轨道最终都汇入 `WeChatMessageService.processMessage()`。这是双轨设计的核心收益：**不管消息从哪条轨道来，后续处理完全一样**。

```
AiBot 长连接 → WsFrame → Bridge 转换 → { botId, content, chatid, chattype, ... }
                                              │
Webhook → HTTP POST → 解析 → { botId, content, chatid, chattype, ... }
                                              │
                                              ▼
                          onProcessMessage 统一回调
                                              │
                                              ▼
                   WeChatMessageService.processMessage()
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                          路由决策         项目上下文        AI 响应
                                        解析（chatId→
                                         projectId）
```

汇合之后，`processMessage` 做的事情包括：

1. **解析项目上下文**：用 chatid 查 `wechat_chat_mappings`，找到 projectId，加载项目的知识库、成员、文档。
2. **路由决策**：决定让哪个数字员工、用哪个技能响应（这部分是 WinMatrix 的决策引擎，和 WebSocket/Webhook 来源无关）。
3. **AI 响应**：执行 Turn、流式输出。响应结果再通过对应轨道发回企微——长连接来的走长连接发，Webhook 来的走 HTTP 响应或主动推送。

这种"多入口、单管线"的设计，让消息来源对核心逻辑透明。未来如果加第三条轨道（比如钉钉、飞书接入），只要写新的 Adapter 转成统一格式，核心管线一行不用改。

**教训：多渠道接入系统，核心是"统一的内部消息模型 + 各渠道独立的 Adapter"。** 让渠道差异死在 Adapter 层，别让它渗透到业务逻辑里。渠道越多，这个原则越值钱——否则你的业务代码里会塞满 `if (fromWecom) ... else if (fromDingTalk) ...` 的分支地狱。

---

## 给后来者的几条总结

1. **IM 接入要双轨甚至多轨并存**。长连接和 Webhook 能力互补，不是替代关系。让用户按场景选。
2. **不同来源的消息尽早转成统一内部格式**。Adapter 层是防腐败层，业务逻辑不该感知渠道差异。
3. **群聊到项目的映射用"发现 + 人工补全"**。外部标识符不可控时，半自动流程比强求全自动务实。
4. **微文档的 docid 和 pad_id 要分清**。存 API 用的 docid，别存浏览器地址栏的 pad_id。
5. **绑定流程用显式状态机**。五态（configured/testing/paired/error/disabled）比二元状态清晰得多。
6. **凭证存储：需还原用加密，需验证用 hash**。secret 用加密，配对码用 hash + 过期。
7. **第三方 API 按业务域拆服务**。共享基础能力（token）单独抽，业务域正交演进。
8. **消息回调里的错误要吃掉只记 log**。单条消息失败不该拖垮整条长连接。

企业微信接入是个"看着简单、做起来全是坑"的领域——官方文档不全、ID 体系混乱、各接入方式能力边界模糊。把这些坑趟平，你的 AI 才能真正"住进"企业微信，而不是浮在一个独立的网页里。

---

> **下一篇**：[《PMDOC 项目容器：项目是协作的物理容器》](./22-pmdoc-project-container.md)——为什么 WinMatrix 把"项目"设计成一个文件系统容器？PMDOC 的固定阶段目录是怎么约定的？讲透"项目即容器"的设计哲学。
