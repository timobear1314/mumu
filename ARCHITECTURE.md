# 项目架构概览

本文档基于仓库目录结构整理，提供对整体架构、子项目与关键技术点的简洁说明，便于快速上手与定位代码位置。

## 项目类型

- 单一仓库的多包（monorepo）工程，使用 pnpm workspace 管理多个子包与多套前端演示应用。
- 使用 TypeScript、Vite、pnpm、turbo 等现代前端工具链；包含若干 UI 框架的示例实现。

## 主要目录说明

- `apps/`：运行级示例与服务
  - `backend-mock/`：轻量 mock 后端（含 `nitro.config.ts`、路由与中间件），用于本地开发与 API 联调。
- `web-antd/`, `web-antdv-next/`, `web-ele/`, `web-naive/`, `web-tdesign/`：多套基于不同 UI 框架的前端演示/实现，均为 Vite 项目，便于对比不同组件库的适配。
- `packages/`：可复用的内部包与组件库
  - `@core/`、`constants/`、`icons/`、`locales/`、`stores/`、`utils/` 等，封装业务逻辑、状态、国际化、样式与基础组件。
- `internal/`：内部工具与共享配置
  - 代码风格、lint、formatter 和其他内部开发工具配置。
- `docs/`：文档站点源码与示例，用于生成项目文档与演示。
- `playground/`：实验/测试用的小型项目与测试用例。
- `scripts/`：构建、清理与部署脚本集合。

## 核心技术栈与工具

- 语言：TypeScript
- 包管理：pnpm（workspace）
- 构建与开发：Vite（前端示例）、Nitro（mock 后端）
- 单体/增量构建：turbo（存在 `turbo.json`）
- 测试：Vitest（存在 `vitest.config.ts`）
- 代码质量：ESLint、Stylelint、commitlint、lefthook 等钩子与规则配置

## 开发与运行（快速提示）

- 安装依赖：`pnpm install`（在仓库根目录）
- 启动某个演示应用：进入对应子目录（例如 `web-antd`），运行 `pnpm dev` 或 `pnpm run dev`（各子项目的 `package.json` 中定义具体脚本）。
- 启动 mock 后端：进入 `apps/backend-mock`，根据 `package.json` 脚本运行（通常为 `dev` 或 `start`）。

## 代码组织与协作建议

- 将通用逻辑放在 `packages/` 下（例如 `@core`、`utils`），通过 workspace 引用复用。
- 每个 `web-*` 项目保留独立的 `vite.config.ts` 与 `tsconfig`，便于单独调试。
- 使用 `internal/lint-configs` 中的共享配置，保持多个子项目一致的代码风格。

## 构建与发布

- 可通过根目录或各包内的脚本执行构建（例如 `pnpm -w build` 或使用 turbo 串联多个任务）。
- 发布私有包或组件库前，先在 `packages/*` 中完成打包配置并在 `package.json` 中确认 `exports`/`types` 字段。

## 后续改进建议

- 在根目录添加统一的开发脚本（例如 `pnpm dev:all`）以便同时启动多个服务/演示。
- 补充 README 或 CONTRIBUTING 指南，说明本地开发、测试与提交流程。

---

更多细节可参阅各子项目的 `package.json` 与对应源码目录以获得运行脚本与配置细节。

## 权限验证（认证与授权）

下面给出一份面向本仓库架构的完整权限验证设计与实现建议（含后端 mock 实现思路、前端守卫/组件集成、Token 存储策略与安全注意事项），便于在 `apps/backend-mock` 与各 `web-*` 项目中快速落地。

### 目标与概念

- 认证（Authentication）：确认用户身份，通常通过账号/密码或第三方登录获得访问凭证（如 JWT 或会话 cookie）。
- 授权（Authorization）：基于用户身份与权限判断能否访问特定路由、API 或 UI 操作（RBAC / 权限点）。

本仓库目标：实现可在本地联调的权限体系——后端由 `apps/backend-mock` 提供 mock 授权接口；共享认证逻辑放入 `packages/`（例如 `packages/@core/auth`）；前端在 `web-*` 中使用路由守卫与权限指令/组件控制视图。

### 设计决策（推荐）

- Token 类型：使用短期访问令牌（`accessToken`，JWT）+ 长期刷新令牌（`refreshToken`）。
- 存储位置：建议把 `refreshToken` 放在 HTTP-only Secure cookie，由后端在登录时设置；`accessToken` 可放在内存或尽量避免长期存储于 localStorage，以降低 XSS 风险。若无法使用 HTTP-only cookie，可把 `accessToken` 放在内存并在页面刷新时通过 `refreshToken` 获取新的 `accessToken`。
- 鉴权方式：API 请求统一在请求拦截器中添加 `Authorization: Bearer <accessToken>`（或依靠 cookie 自动加入）。后端中间件负责校验 token 并注入 `req.user` / `event.context.auth`。

### API 约定（示例）

- `POST /api/auth/login` — 接受 `{ username, password }`，返回 `{ accessToken, expiresIn }`，并通过 Set-Cookie 设置 `refreshToken`（HTTP-only）。
- `POST /api/auth/refresh` — 使用 cookie 中的 `refreshToken` 颁发新的 `accessToken`。
- `POST /api/auth/logout` — 清除 `refreshToken`（响应 Set-Cookie 清除）。
- `GET /api/user/me` — 返回当前用户信息与角色/权限点（如 `{ id, name, roles: ['admin'], permissions: ['user:list','user:edit'] }`）。

### 后端实现要点（在 `apps/backend-mock` 中）

- 在 `apps/backend-mock` 中添加中间件 `auth`：
  - 解析 `Authorization` header（Bearer token）或依赖 cookie；
  - 校验 JWT 签名/过期；
  - 将解码后的用户信息挂载到请求上下文（例如 `event.context.user`）。
- 路由保护：在需要保护的路由上检查 `context.user` 是否存在并校验权限点或角色。
- Mock 实现建议：在 `apps/backend-mock/utils` 中放一组模拟用户与权限表，支持登录、刷新与权限查询接口，便于前端联调。

示例伪代码（Nitro 风格中间件）：

```js
// apps/backend-mock/middleware/auth.js
export default defineEventHandler(async (event) => {
  const authHeader = getRequestHeader(event, 'authorization');
  if (!authHeader) return; // 继续，部分路由可以公开
  const token = authHeader.replace('Bearer ', '');
  try {
    const payload = verifyJwt(token);
    event.context.user = payload;
  } catch (e) {
    throw createError({ statusCode: 401, statusMessage: 'Invalid token' });
  }
});
```

### 前端实现要点（在 `web-*` 项目）

- 将通用 auth 逻辑抽象到 `packages/@core/auth`：包含 `useAuth()` composable、`authService`（封装 login/refresh/logout）、和权限工具函数 `hasRole` / `hasPermission`。
- 路由守卫：在 `router` 中实现全局前置守卫 `router.beforeEach`：
  - 检查路由元信息（如 `meta.requiresAuth`、`meta.permissions`）；
  - 若无 `accessToken` 则跳转到登录页，或尝试调用 `refresh` 接口获取新的 `accessToken`；
  - 若 `accessToken` 存在并通过 `user.permissions` 校验，则允许访问。

示例伪代码（路由守卫）：

```js
router.beforeEach(async (to) => {
  const auth = useAuth();
  if (to.meta.requiresAuth) {
    if (!auth.isAuthenticated()) {
      const ok = await auth.tryRefresh(); // 使用 refreshToken 刷新
      if (!ok) return { name: 'Login', query: { redirect: to.fullPath } };
    }
    if (to.meta.permissions && !auth.hasPermissions(to.meta.permissions)) {
      return { name: '403' };
    }
  }
});
```

UI 层控制：提供 `v-permission` 指令或 `Permission` 组件，根据 `user.permissions` 显示/隐藏按钮与菜单项。

### 请求拦截与错误处理

- 在 `packages/@core/http`（或各前端项目内）实现请求拦截器：
  - 自动在请求头注入 `Authorization`；
  - 在 401 响应时尝试一次刷新（调用 `POST /api/auth/refresh`），刷新成功则重试原请求；若刷新失败跳转登录。

示例逻辑：

1. 请求 A 带 `accessToken`；
2. 若返回 401：
   - 调用 `refresh`（一次且阻塞正在进行的刷新请求）；
   - 刷新成功：更新 `accessToken` 并重试 A；
   - 刷新失败：登出并跳转登录。

### Token 生命周期与安全注意事项

- 使用短期 `accessToken`（如 5-15 分钟）+ 长期 `refreshToken`（HTTP-only cookie，周期可长一些）。
- 强烈建议将 `refreshToken` 设置为 `HttpOnly; Secure; SameSite=Strict`，以减少 XSS/CSRF 风险。
- 若无法使用 cookie，前端需妥善保管 `refreshToken`（仍建议走后端代理或加强前端安全策略）。

### 授权模型建议（简单 RBAC + 权限点）

- 角色（roles）：如 `admin`, `editor`, `user`。
- 权限点（permissions）：细粒度字符串，如 `user:list`, `user:edit`。
- 用户返回的数据结构示例：

```json
{
  "id": "u123",
  "name": "Alice",
  "roles": ["editor"],
  "permissions": ["user:list", "post:edit"]
}
```

### 本地联调建议（针对本仓库）

- 在 `apps/backend-mock` 中实现一个简单的认证模块，返回上述结构并支持 `refresh`。参考目录：[apps/backend-mock](apps/backend-mock)。
- 在 `packages/` 创建 `@core/auth`、`@core/http` 两个包：
  - `@core/auth` 提供 `useAuth()`、权限判断工具与类型定义；
  - `@core/http` 提供统一的请求封装（含拦截器、重试刷新逻辑）。
- 前端各 `web-*` 项目在入口处注入 auth 状态并在 `router` 使用前置守卫。

### 用户体验与错误提示

- 登录失败需返回明确错误原因（账号不存在 / 密码错误 / 账户被禁用）。
- 当权限不足时，前端应提供统一的 403 页面，并在 UI 中灰显或隐藏不可操作元素。

### 测试与验证

- 为后端 mock 编写单元测试，验证 token 签发/刷新/失效流程。
- 使用 Vitest 为 `packages/@core/auth` 编写测试，覆盖登录、刷新与权限判断逻辑。

---

如需，我可以：（选一项）

- A) 在 `apps/backend-mock` 下添加一个最小的 mock-auth 实现并提交示例代码；
- B) 在 `packages/` 下创建 `@core/auth` 与 `@core/http` 的骨架实现（含基本类型与 composable）；
- C) 仅将本节再细化为“开发者逐步实现指南”并加入 `ARCHITECTURE.md`。

请告诉我你想要我继续执行的下一步（A/B/C）。

## 详细架构图

下面用一个简单的 Mermaid 图描述仓库中的主要关系（monorepo、packages、apps 与各 `web-*` 演示项目之间的依赖与交互）：

```mermaid
flowchart TD
  repo["vue-vben-admin (monorepo)"]
  subgraph PACKAGES [packages/]
    core[@core]
    utils[utils]
    locales[locales]
  end
  subgraph APPS [apps/]
    backend_mock[backend-mock]
  end
  subgraph WEBS [web-* projects]
    web_antd[web-antd]
    web_antdv[web-antdv-next]
    web_ele[web-ele]
  end
  repo --> PACKAGES
  repo --> APPS
  repo --> WEBS
  WEBS -->|import| PACKAGES
  WEBS -->|calls API| APPS
  APPS -->|mock| PACKAGES
```

## 运行示例（逐步）

下面给出在本地启动常见场景的示例命令 —— 在执行前建议检查对应子项目的 `package.json` 脚本以确认具体命令与端口。

1. 在仓库根目录安装依赖：

```bash
pnpm install
```

2. 启动 mock 后端（在单独终端窗口）：

```bash
cd apps/backend-mock
pnpm dev
# 或者 pnpm run dev / pnpm start，详情见 apps/backend-mock/package.json
```

3. 启动某个前端演示（另一个终端）：

```bash
cd web-antd
pnpm dev
# 默认 vite 会在 5173 端口启动；若端口冲突请查看 vite.config.ts 或子项目 package.json
```

4. 同时运行多个演示：在各自目录分别运行 `pnpm dev`，或在根目录创建并使用统一脚本（如 `pnpm -w run dev`，视 workspace scripts 而定）。

## 架构要点与调试建议

- 若在运行时出现模块解析错误，优先检查 `tsconfig.json` 中的 `paths` 配置与 workspace 依赖引用是否正确。
- 若 mock 后端未生效，查看 `apps/backend-mock` 中的中间件与路由文件，确认启动日志输出的端口与路由前缀。
- 在本地联调时，建议把业务共享代码放在 `packages/` 下并在子项目中以 workspace 方式引用，这样变更可以热重载并快速验证。

---

如需我把这部分内容拆成独立的架构图文件、生成 PNG/SVG，或把运行脚本加入根目录快捷脚本（如 `dev:all`），我可以继续实现。
