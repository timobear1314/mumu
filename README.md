# mumu

基于 Vue Vben Admin 5.7.0 的 pnpm monorepo。本项目使用 Vue 3、Vite、TypeScript、Turbo、pnpm workspace，并提供多套 UI 框架示例应用、文档站、mock 后端和 playground。

## 环境要求

- Node.js: `^20.19.0 || ^22.18.0 || ^24.0.0`
- pnpm: `>=10.0.0`
- 当前仓库声明的包管理器: `pnpm@10.33.0`

建议使用 Corepack 管理 pnpm:

```bash
corepack enable
corepack prepare pnpm@10.33.0 --activate
```

## 安装依赖

在项目根目录执行:

```bash
pnpm install
```

根脚本 `pnpm bootstrap` 等价于 `pnpm install`。

## 项目结构

```text
apps/
  backend-mock/      Nitro mock API 服务
  web-antd/          Ant Design Vue 版本
  web-antdv-next/    Antdv Next 版本
  web-ele/           Element Plus 版本
  web-naive/         Naive UI 版本
  web-tdesign/       TDesign Vue Next 版本
docs/                VitePress 文档站
playground/          集成示例与 e2e 测试入口
packages/            业务包、基础组件、工具包与样式等 workspace 包
internal/            内部配置包，如 tsconfig、vite、lint、tailwind
scripts/             工程脚本与 CLI 工具
```

Workspace 范围由 `pnpm-workspace.yaml` 管理，依赖版本主要通过 catalog 统一维护。

## 本地开发

启动默认开发任务:

```bash
pnpm dev
```

按应用启动:

```bash
pnpm dev:antd
pnpm dev:antdv-next
pnpm dev:ele
pnpm dev:naive
pnpm dev:tdesign
pnpm dev:docs
pnpm dev:play
```

启动 mock 后端:

```bash
pnpm -F @vben/backend-mock start
```

## 构建

构建全部项目:

```bash
pnpm build
```

按应用构建:

```bash
pnpm build:antd
pnpm build:ele
pnpm build:naive
pnpm build:tdesign
pnpm build:docs
pnpm build:play
```

## 检查与测试

```bash
pnpm lint          # 代码检查
pnpm format        # 格式化
pnpm check         # 循环依赖、依赖、类型和拼写检查
pnpm test:unit     # 单元测试
pnpm test:e2e      # e2e 测试
```

也可以直接对某个 workspace 包运行脚本，例如:

```bash
pnpm -F @vben/web-antd typecheck
pnpm -F @vben/playground test:e2e
```

## 常用维护命令

```bash
pnpm clean         # 清理构建产物和缓存
pnpm reinstall     # 清理锁文件后重新安装
pnpm changeset     # 创建 changeset
pnpm update:deps   # 检查并更新依赖
```

## 账号

本地示例默认测试账号1:

```text
vben / 123456
```

## 参考

- 上游仓库: <https://github.com/vbenjs/vue-vben-admin>
- 在线预览: <https://vben.pro/>
- 文档: <https://doc.vben.pro/>
- 许可证: [MIT](./LICENSE)
