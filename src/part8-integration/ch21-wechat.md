# 第 21 章 企业微信集成

> "让 AI 走进员工日常使用的沟通工具。"

企业微信（WeCom）集成是 WinMatrix 连接企业协作场景的关键桥梁。它不仅包括 OAuth 登录，还包括 AI Bot 消息通道、企微文档/日程/联系人工具。本章将分析这些集成的实现。

## 21.1 企业微信 OAuth 认证

`WeChatOAuthService` 实现了企业微信扫码登录：

```typescript
// src/infrastructure/auth/WeChatOAuthService.ts（第 7-55 行）
export interface WeChatUserInfo {
  userId: string;          // 企业微信用户 ID
  name?: string;
  avatar?: string;
  email?: string;
  mobile?: string;
  gender?: number;
  position?: string;
  department?: number[];
  isLeader?: number[];
  directLeader?: string[];
  alias?: string;
  status?: number;         // 激活状态
}

export interface WeChatOAuthConfig {
  corpId: string;
  agentId: string;
  corpSecret: string;
  redirectUri: string;
}

export class WeChatOAuthService {
  private config: WeChatOAuthConfig;
  private accessTokenCache: { token: string; expiresAt: number } | null = null;

  /**
   * 生成企业微信授权登录 URL
   */
  generateAuthUrl(state?: string): string {
    const params = new URLSearchParams({
      appid: this.config.corpId,
      agentid: this.config.agentId,
      redirect_uri: this.config.redirectUri,
      state: state || '',
      usertype: 'member',    // 成员登录
    });
    return `https://open.weixin.qq.com/connect/oauth2/authorize?${params.toString()}#wechat_redirect`;
  }

  /**
   * 生成企业微信扫码登录 URL (PC端)
   */
  generateQrConnectUrl(state?: string): string {
    // 注意：不要对 redirect_uri 预先 encodeURIComponent
    // URLSearchParams.toString() 会统一编码，否则会二次编码导致企业微信返回「参数错误」
    const params = new URLSearchParams({
      appid: this.config.corpId,
      agentid: this.config.agentId,
      redirect_uri: this.config.redirectUri,
      state: state || '',
      login_type: 'CorpApp',
    });
    // ...
  }
}
```

### Access Token 缓存

```typescript
private accessTokenCache: { token: string; expiresAt: number } | null = null;
```

企业微信 API 调用需要 access_token，该 token 有有效期。`accessTokenCache` 缓存 token 并记录过期时间，避免每次 API 调用都重新获取。

### 编码陷阱

注释中记录了一个真实的编码陷阱：`redirect_uri` 不能预先 `encodeURIComponent`，因为 `URLSearchParams.toString()` 会再次编码，导致二次编码。这种经验性的注释对后续维护者非常有价值。

## 21.2 WeCom AI Bot 消息通道

`src/interface/channel/channels/wecom-aibot/`（37 个文件）实现了企微 AI Bot 的完整消息通道：

```
src/interface/channel/channels/wecom-aibot/
├── aibot/                    # 核心通道
│   ├── WeComAiBotChannel.ts        # 通道主类
│   ├── WeComAiBotClient.ts         # 客户端
│   ├── WeComAiBotManager.ts        # 管理器
│   ├── WeComAiBotMessageBridge.ts  # 消息桥接
│   ├── WeComAiBotPushQueue.ts      # 推送队列
│   └── WeComAiBotConfigBus.ts      # 配置总线
├── commands/                 # 命令处理
│   ├── BindProjectChatCommand.ts
│   ├── BindWecomCommand.ts
│   ├── ListAgentsCommand.ts
│   ├── RunAgentCommand.ts
│   └── WhoamiCommand.ts
├── context/                  # 上下文管理
│   ├── WeComContextStore.ts
│   └── WeComPrivateProjectContextStore.ts
└── (其他): WeChatMessageService.ts, WeChatUserMappingService.ts,
          WeComBindingService.ts, messageTypes.ts, sendWebhook.ts
```

### 命令系统

企微 AI Bot 支持斜杠命令：

| 命令 | 功能 |
|------|------|
| `/bind` | 绑定企微账号 |
| `/bindproject` | 绑定项目聊天 |
| `/agents` | 列出可用 Agent |
| `/run` | 运行指定 Agent |
| `/whoami` | 查看当前身份 |

### 消息去重

```typescript
// WecomMessageDedupStore.ts
// 企微可能重复推送消息（至少一次投递语义）
// 通过消息 ID 去重，避免重复处理
```

### Markdown 截断

```typescript
// wecomMarkdownTruncate.ts
// 企微消息有长度限制
// 长 Markdown 内容需要截断
```

## 21.3 企微业务工具

WinMatrix 提供三类企微业务工具：

### 企微联系人工具

```typescript
// src/business-tools/wecom-contact/index.ts（第 14-18 行）
export function registerTools(registry: IToolRegistry): void {
  registry.register(new WeComContactGetUserTool());
  registry.register(new WeComContactListDepartmentUsersSimpleTool());
  registry.register(new WeComContactListDepartmentUsersDetailTool());
}
```

### 企微文档工具（13 个）

```typescript
// src/business-tools/wecom-document/index.ts（第 45-59 行）
export function registerTools(registry: IToolRegistry): void {
  // Wedoc 文档操作
  registry.register(new WeComCreateWedocTool());
  registry.register(new WeComListConversationWedocDocsTool());
  registry.register(new WeComRenameWedocTool());
  registry.register(new WeComGetWedocBaseInfoTool());
  registry.register(new WeComGetWedocAuthTool());
  registry.register(new WeComGetWedocDocumentTool());
  registry.register(new WeComBatchUpdateWedocDocumentTool());
  // SmartSheet 智能表格
  registry.register(new WeComSmartSheetGetSheetTool());
  registry.register(new WeComSmartSheetGetFieldsTool());
  registry.register(new WeComSmartSheetGetRecordsTool());
  registry.register(new WeComSmartSheetAddRecordsTool());
  // Spreadsheet 传统表格
  registry.register(new WeComSpreadsheetGetSheetPropertiesTool());
  registry.register(new WeComSpreadsheetGetSheetRangeDataTool());
}
```

### 企微日程工具（6 个）

```typescript
// src/business-tools/wecom-schedule/index.ts（第 20-27 行）
export function registerTools(registry: IToolRegistry): void {
  registry.register(new WeComScheduleAddTool());
  registry.register(new WeComScheduleUpdateTool());
  registry.register(new WeComScheduleGetTool());
  registry.register(new WeComScheduleDeleteTool());
  registry.register(new WeComScheduleListByCalendarTool());
  registry.register(new WeComCalendarGetTool());
}
```

## 21.4 用户映射

```typescript
// WeChatUserMappingService.ts
// 企业微信用户 ID ↔ WinMatrix 用户的映射
// OAuth 登录时自动建立
// 支持手动绑定（createUserSchema 的 wechatUserId 字段）
```

## 21.5 iframe 跨域处理

企微工作台通常在 iframe 中嵌入应用，OAuth 回调需要特殊处理（见第 6 章）：

```typescript
// src/interface/api/auth.ts
function renderWechatRedirectHtml(relativePath: string, targetBasePath = basePath): string {
  // 使用 window.top.location.href 逃逸 iframe
  // 避免 iframe 跨域 302 跳转问题
}
```

## 本章小结

本章深入分析了 WinMatrix 的企业微信集成：

1. **OAuth 认证**：扫码登录 + access_token 缓存 + 编码陷阱规避
2. **AI Bot 通道**：37 个文件，完整消息通道 + 命令系统
3. **斜杠命令**：bind / bindproject / agents / run / whoami
4. **消息去重**：至少一次投递语义下的幂等处理
5. **三类企微工具**：联系人（3）+ 文档（13）+ 日程（6）
6. **用户映射**：企微用户 ID ↔ WinMatrix 用户
7. **iframe 跨域**：window.top 逃逸 + noscript 回退

在下一章中，我们将分析 MCP 协议与外部 Agent。
