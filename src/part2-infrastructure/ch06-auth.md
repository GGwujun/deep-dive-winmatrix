# 第 6 章 认证与授权

> "安全不是功能，而是责任。"

WinMatrix 的认证授权系统覆盖了从 JWT 签发到 RBAC 权限检查的完整链路。它支持多种登录方式（密码、企业微信 OAuth、SSO、Personal Access Token），并通过 Redis 黑名单实现 Token 撤销。本章将深入这些安全机制的实现。

## 6.1 JWT 认证

`JwtService` 是认证系统的核心，负责 Token 的生成、验证和撤销：

```typescript
// src/infrastructure/auth/JwtService.ts（第 8-19 行）
export interface JwtPayload {
  userId: string;
  username: string;
  isAdmin: boolean;
  iat?: number;   // 签发时间（自动添加）
  exp?: number;   // 过期时间（自动添加）
}

export interface ThirdPartyJwtPayload {
  username: string;
  systemId: string;        // 第三方系统标识
  expTimestamp: number;    // Token 过期时间（Unix 时间戳，秒）
  employeeName?: string;
  iat?: number;
}
```

### Token 生成

```typescript
// src/infrastructure/auth/JwtService.ts（第 52-84 行）
export class JwtService {
  private secret: string;
  private expiresIn: string;
  private redis?: Redis;

  constructor(secret: string, expiresIn: string = '24h', redis?: Redis) {
    if (!secret || secret.length < 32) {
      throw new Error('JWT_SECRET must be at least 32 characters long');
    }
    this.secret = secret;
    this.expiresIn = expiresIn;
    this.redis = redis;
  }

  generateToken(userId: string, username: string, isAdmin: boolean): string {
    const payload: JwtPayload = { userId, username, isAdmin };
    const token = jwt.sign(payload, this.secret, {
      expiresIn: this.expiresIn,
      algorithm: 'HS256',   // 显式指定算法
    });
    return token;
  }
}
```

关键安全设计：

1. **密钥强度校验**：构造函数强制要求 secret 至少 32 字符
2. **显式算法**：`algorithm: 'HS256'`，防止算法替换攻击

### Token 验证与防算法替换

```typescript
// src/infrastructure/auth/JwtService.ts（第 90-106 行）
verifyToken(token: string): JwtPayload {
  try {
    const decoded = jwt.verify(token, this.secret, {
      algorithms: ['HS256'],   // 关键：限定算法
    }) as JwtPayload;
    return decoded;
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new Error('Token expired');
    } else if (error instanceof jwt.JsonWebTokenError) {
      throw new Error('Invalid token');
    }
    throw error;
  }
}
```

`algorithms: ['HS256']` 是防算法替换攻击的关键。如果不限定，攻击者可以构造一个使用 `none` 算法的 Token，绕过签名验证。

### Token 撤销（黑名单）

JWT 本身是无状态的——一旦签发，在过期前一直有效。WinMatrix 通过 Redis 黑名单实现了主动撤销：

```typescript
// src/infrastructure/auth/JwtService.ts（第 108-146 行）
async verifyAndCheckBlacklist(token: string): Promise<JwtPayload> {
  const payload = this.verifyToken(token);
  if (this.redis) {
    const isRevoked = await this.isTokenRevoked(token);
    if (isRevoked) {
      throw new Error('Token has been revoked');
    }
  }
  return payload;
}

async revokeToken(token: string): Promise<void> {
  if (!this.redis) {
    logger.warn('Redis not available, cannot revoke token');
    return;
  }
  // 解析 Token 获取过期时间（不验证签名，因为可能已过期）
  const decoded = jwt.decode(token) as JwtPayload | null;
  if (!decoded || !decoded.exp) return;

  const now = Math.floor(Date.now() / 1000);
  const ttl = decoded.exp - now;   // TTL = Token 剩余生命期

  // 只有未过期的 Token 才需要加入黑名单
  if (ttl > 0) {
    const key = `jwt:blacklist:${token}`;
    await this.redis.setex(key, ttl, '1');  // 设置 TTL，自动清理
  }
}
```

精妙之处在于 TTL 的设置——黑名单条目的 TTL 等于 Token 的剩余生命期。Token 过期后，黑名单条目自动被 Redis 清理，避免黑名单无限增长。

## 6.2 JWT 中间件

`jwtAuth` 中间件从请求中提取 Token 并验证用户身份：

```typescript
// src/interface/middleware/jwtAuth.ts（第 90-110 行）
export function createJwtAuthMiddleware(jwtService: JwtService) {
  return async function jwtAuth(request: FastifyRequest, reply: FastifyReply): Promise<void> {
    try {
      const token = getTokenFromRequest(request);
      if (!token) {
        return reply.status(401).send({ success: false, error: 'Missing authorization token' });
      }
      const user = await attachUserFromToken(request, jwtService, token);
    } catch (error) {
      return reply.status(401).send({ success: false, error: 'Invalid or expired token' });
    }
  };
}
```

### 多来源 Token 提取

```typescript
// src/interface/middleware/jwtAuth.ts（第 120-140 行）
export function getTokenFromRequest(request: FastifyRequest): string | null {
  // 1) Authorization: Bearer <token>
  const authHeader = request.headers.authorization;
  const fromHeader = JwtService.extractTokenFromHeader(authHeader);
  if (fromHeader) return normalizeToken(fromHeader);

  // 2) X-Auth-Token（某些代理链路可能丢弃 Authorization）
  const rawXToken = request.headers['x-auth-token'];
  if (typeof rawXToken === 'string') return normalizeToken(rawXToken);

  // 3) query.token（WebSocket 升级等场景）
  const q = request.query as { token?: string };
  if (q?.token && typeof q.token === 'string') return normalizeToken(q.token);

  return null;
}
```

三种 Token 来源支持不同的客户端场景：

- **Authorization Header**：标准 REST 请求
- **X-Auth-Token**：某些反向代理会丢弃 Authorization，提供备用 header
- **query.token**：WebSocket 握手（浏览器无法设置 WS header）

### 身份归一化

```typescript
// src/interface/middleware/jwtAuth.ts（第 40-65 行）
async function normalizeAuthenticatedUser(payload: JwtPayload): Promise<AuthenticatedUser> {
  const fallback: AuthenticatedUser = {
    userId: payload.userId,
    username: payload.username,
    isAdmin: payload.isAdmin,
  };
  try {
    const userRepository = await container.resolve<IUserRepository>('IUserRepository');
    const user = payload.userId
      ? await userRepository.findById(payload.userId)
      : payload.username
        ? await userRepository.findByUsername(payload.username)
        : null;
    if (!user) return fallback;

    // 检测 JWT 与数据库身份不一致
    if (user.id !== payload.userId || user.username !== payload.username
        || user.isAdmin !== payload.isAdmin) {
      logger.warn({
        tokenUserId: payload.userId,
        resolvedUserId: user.id,
      }, 'JWT identity normalized from user repository');
    }
    return { userId: user.id, username: user.username, isAdmin: user.isAdmin };
  } catch (error) {
    return fallback;  // 数据库查询失败时回退到 JWT payload
  }
}
```

身份归一化确保了 JWT 中的身份信息与数据库一致——如果用户在 Token 签发后被重命名或权限变更，这里会用数据库的最新值为准。

## 6.3 RBAC 权限模型

`PermissionService` 实现了基于角色的访问控制（RBAC）：

```typescript
// src/business/domain/rbac/PermissionService.ts（第 10-18 行）
export class PermissionService {
  private memberRepository: IMemberRepository;
  private redis?: Redis;
  private cacheTtl: number = 300;  // 5 分钟缓存

  constructor(memberRepository: IMemberRepository, redis?: Redis) {
    this.memberRepository = memberRepository;
    this.redis = redis;
  }
}
```

### 权限检查（Fail-Closed）

```typescript
// src/business/domain/rbac/PermissionService.ts（第 23-49 行）
async checkPermission(
  userId: string, projectId: string, permission: string
): Promise<boolean> {
  try {
    // 支持旧权限格式
    const normalizedPermission = migrateLegacyPermission(permission);

    // 尝试从缓存获取
    const cachedPermissions = await this.getCachedPermissions(userId, projectId);
    if (cachedPermissions) {
      return cachedPermissions.has(normalizedPermission);
    }

    // 从数据库获取
    const permissions = await this.getUserPermissions(userId, projectId);
    await this.cachePermissions(userId, projectId, permissions);

    return permissions.has(normalizedPermission);
  } catch (error) {
    logger.error('Permission check failed:', error);
    return false;  // Fail-closed：出错时拒绝
  }
}
```

关键设计：

1. **Fail-Closed**：`catch` 块返回 `false`——出错时拒绝访问，而非放行。这是安全系统的核心原则。
2. **缓存优先**：5 分钟 Redis 缓存，避免每次请求都查数据库
3. **格式兼容**：`migrateLegacyPermission` 处理旧权限格式

### 项目级权限作用域

```typescript
// src/business/domain/rbac/PermissionService.ts（第 78-88 行）
async getUserPermissions(userId: string, projectId: string): Promise<Set<string>> {
  const members = await this.memberRepository.findByProject(projectId);
  const member = members.find(m => m.userId === userId);
  if (!member) return new Set();
  return new Set(member.permissions.map(p => migrateLegacyPermission(p)));
}
```

权限是**项目级**的——同一个用户在不同项目中有不同的权限。这种设计支持多项目隔离。

### 复合权限检查

```typescript
// src/business/domain/rbac/PermissionService.ts（第 54-73 行）
async hasAnyPermission(userId, projectId, permissions: string[]): Promise<boolean> {
  for (const permission of permissions) {
    if (await this.checkPermission(userId, projectId, permission)) return true;
  }
  return false;
}

async hasAllPermissions(userId, projectId, permissions: string[]): Promise<boolean> {
  for (const permission of permissions) {
    if (!(await this.checkPermission(userId, projectId, permission))) return false;
  }
  return true;
}
```

- `hasAnyPermission`：OR 语义（任一权限即可）
- `hasAllPermissions`：AND 语义（需要全部权限）

## 6.4 权限中间件管线

`permission.ts` 中间件提供了 5 种权限守卫：

```typescript
// src/interface/middleware/permission.ts（第 50-100 行）
export function createPermissionMiddleware(permissionService: PermissionService) {
  function requirePermission(permission: string) {
    return async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
      const gate = runProjectScopedPermissionGate(request, reply);
      if (gate.status === 'replied') return;
      if (gate.status === 'allow-sysadmin-no-project') return;  // 系统管理员放行
      const { userId, projectId } = gate;
      const hasPermission = await permissionService.checkPermission(userId, projectId, permission);
      if (!hasPermission) {
        createPermissionDeniedResponse(reply, { permissionType: 'permission', userId, projectId, permission });
      }
    };
  }

  return { requirePermission, requireAnyPermission, requireAllPermissions, requireRole, requireAdmin };
}
```

### 三态门控

```typescript
function runProjectScopedPermissionGate(request, reply): ProjectScopedGateResult {
  const userId = request.user?.userId;
  if (!userId) {
    createUnauthorizedResponse(reply);
    return { status: 'replied' };        // 401
  }
  const projectId = getOptionalProjectId(request);
  if (!projectId) {
    if (request.user?.isAdmin) {
      return { status: 'allow-sysadmin-no-project', userId };  // 系统管理员无项目也放行
    }
    createProjectRequiredResponse(reply);
    return { status: 'replied' };        // 400
  }
  return { status: 'allow', userId, projectId };  // 继续权限检查
}
```

三态决策：

1. **未认证** → 401
2. **无项目上下文 + 非管理员** → 400（需要项目 ID）
3. **无项目上下文 + 管理员** → 放行（系统级操作）
4. **有项目上下文** → 继续具体权限检查

## 6.5 多登录方式

### 密码登录

```typescript
// src/interface/api/auth.ts（第 59-62 行）
const loginSchema = z.object({
  username: z.string().min(1, 'Username is required'),
  password: z.string().min(1, 'Password is required'),
});
```

密码使用 bcrypt 哈希存储，最小长度 8 字符。

### 企业微信 OAuth

```typescript
// src/interface/api/auth.ts（第 26-52 行）
// 企微回调跳转 HTML
function renderWechatRedirectHtml(relativePath: string, targetBasePath = basePath): string {
  const safePath = (targetBasePath + relativePath).replace(/["<>]/g, '');  // XSS 过滤
  return `<!doctype html><html>...
    <script>
    (function () {
      var target = ${JSON.stringify(safePath)};
      try {
        if (window.top && window.top !== window.self) {
          window.top.location.href = target;  // 顶层窗口导航，避免 iframe 跨域
        } else {
          window.location.href = target;
        }
      } catch (e) { window.location.href = target; }
    })();
    </script>
    <noscript><meta http-equiv="refresh" content="0;url=${safePath}"></noscript>
  ...`;
}
```

企微回调处理的几个安全细节：

1. **XSS 过滤**：`replace(/["<>]/g, '')` 清理路径中的危险字符
2. **iframe 逃逸**：使用 `window.top` 导航，避免企微 iframe 跨域问题
3. **noscript 回退**：`<meta http-equiv="refresh">` 为禁用 JS 的浏览器提供回退

### SSO 单点登录

`ThirdPartyJwtPayload` 支持第三方系统的 SSO：

```typescript
export interface ThirdPartyJwtPayload {
  username: string;
  systemId: string;        // 第三方系统标识
  expTimestamp: number;
  employeeName?: string;
}
```

第三方系统签发自己的 JWT，WinMatrix 验证后映射到内部用户。

## 6.6 会话管理

WinMatrix 使用 `@fastify/secure-session` 管理会话：

```typescript
// src/startup/api.ts（第 130-145 行）
const sessionKey = createHash('sha256').update(config.sessionSecret, 'utf8').digest();
await apiServer.register(secureSession, {
  key: sessionKey,   // SHA-256 派生密钥
  cookie: {
    secure: config.nodeEnv === 'production',        // 生产环境仅 HTTPS
    httpOnly: true,                                  // 防 XSS 读取
    sameSite: config.nodeEnv === 'production' ? 'none' : 'lax',  // 跨站策略
    maxAge: 86400000,                               // 24 小时
  },
});
```

会话 Cookie 的安全配置：

- **secure**：生产环境仅通过 HTTPS 传输
- **httpOnly**：JavaScript 无法读取，防 XSS
- **sameSite**：生产环境 `none`（支持跨站，需配合 secure），开发环境 `lax`
- **SHA-256 密钥派生**：从 sessionSecret 派生加密密钥

## 本章小结

本章深入分析了 WinMatrix 的认证与授权系统：

1. **JWT 认证**：32 字符密钥强制 + HS256 显式算法 + 防算法替换攻击
2. **Token 撤销**：Redis 黑名单，TTL = Token 剩余生命期，自动清理
3. **多来源 Token**：Authorization Header / X-Auth-Token / query.token
4. **身份归一化**：JWT payload 与数据库身份对齐
5. **RBAC 权限**：项目级作用域，5 分钟缓存，Fail-Closed 原则
6. **三态门控**：未认证 / 无项目+非管理员 / 无项目+管理员 / 有项目
7. **多登录方式**：密码（bcrypt）+ 企业微信 OAuth + SSO + PAT
8. **安全 Cookie**：secure + httpOnly + sameSite + SHA-256 派生

在下一章中，我们将深入 REST API 设计。
