# dsh-mcp-manager 开发文档

本仓库的开发 / 调试 / 发布指南。面向需要改动 `dsh-mcp-manager` 源码的人。

## 一、项目概览

`dsh-mcp-manager` 是 DeepSeek Harness (DSH) 的标准双半结构插件：

- **宿主半（host）**：`src/index.ts` → 编译产物 `lib/index.js`。对象形态 Cordis 插件（`{ name, inject, apply }`），注册模型工具与 `POST /dsh-mcp-manager/api` 路由，管理 `cordis.patch.yml` 里的 `@deepseek-ai/dsh-mcp-client` 行与 DSH 技能（Skill）的禁用状态。
- **浏览器半（client）**：`lib/client.js`。手写 ModuleLoader CJS bundle，注册 **设置 → MCP 管理**（`settings.section` 槽位，order 16）与 **设置 → Skills 管理**（order 17）页面，通过同源 `fetch('/dsh-mcp-manager/api')` 与宿主通信，不直接访问文件系统。

## 二、目录结构

```
.
├── src/index.ts          # 宿主半源码（TypeScript，唯一需要编译的部分）
├── lib/
│   ├── index.js          # 宿主半编译产物（tsc 生成，入库）
│   └── client.js         # 浏览器半（手写，无构建步骤）
├── tests/
│   ├── skills.test.mjs   # Skills 模块集成测试（fake-ctx）
│   └── guard.test.mjs    # 双挂载 guard 短路顺序回归测试
├── cordis.patch.yml      # bundle patch：`dsh plugin add` 自动挂载的 loader 行
├── docs/
│   └── plans/2026-08-27-skills-management.md  # Skills 模块设计与实现记录
└── package.json          # dsh.bundle.patch + dsh.client.inject 声明
```

## 三、构建与测试

```bash
npm install
npm run build        # tsc -p tsconfig.json → lib/index.js
npm test             # build + node --test（tests/*.test.mjs）
```

- 宿主半是纯 JS 运行（零运行时依赖），构建只需要 devDependencies。
- `lib/index.js` 是入库的编译产物——改 `src/index.ts` 后必须 `npm run build` 并提交产物。
- `lib/client.js` 手写，无构建步骤，改完 `node --check lib/client.js` 验证语法即可。

## 四、宿主半（src/index.ts）

### 4.1 插件形态

```ts
// 对象形态 Cordis 插件：框架保证 inject 的服务就绪后才运行 apply；
// 任一依赖服务消失时框架自动重载插件（这是插件跨 DSH 升级存活的机制）。
inject: ['timer', 'fs', 'settings', 'sandboxPolicy', 'webServer', 'tools', 'skills']
```

### 4.2 模型工具

用 `ctx.tools.register(defineTool(...))` 注册 4 个 `mcp_manager_*` 工具（list / set_enabled / restart / add）。这是 DSH 官方规定的"给模型加能力"的方式。

### 4.3 HTTP API 与 CSRF 闸门

`POST /dsh-mcp-manager/api`，请求体 `{ op, args }`，响应 `{ ok, ... }`。

跨站防护（CSRF）三重闸门：
1. 仅接受 POST
2. 必须带请求头 `x-dsh-plugin: dsh-mcp-manager`
3. Origin 必须同源（本机脚本不带 Origin 也可以）

**可选 token 鉴权（H1，2.1.3+）**——写操作的最后一道闸，**目的**：CSRF 三重闸门只防"跨站浏览器请求"，它隐含的信任模型是"DSH web 只监听 127.0.0.1"。一旦端口被暴露（端口转发、`dsh-web-lan-access` 类插件、反向代理），任何能访问该端口的人都直接获得完整写权限——`mcpm-add` 的 `stdio.command` 可填任意可执行文件，即**远程任意代码执行**。token 让暴露场景下写操作仍须密钥：`config.token`（loader 行 override，经 Cordis `apply(ctx, config)` 第二参数注入）或环境变量 `DSH_MCP_MANAGER_TOKEN`；启用后 `WRITE_OPS` 集合内所有 op（add/edit/remove/set-enabled/restart/export/import/skill-toggle）校验 `x-dsh-token` 头，读操作（list/version）保持开放以便 UI 渲染。默认关闭，本地单机无需配置。

新增 op 时在 `handleApi` 的 op 分支里加一个 handler 即可（被 `withWriteLock` 包住，按文件写锁串行化）。

### 4.4 Skills 管理模块

- `skill-list` / `skill-toggle` 两个 op。
- **禁用机制**：rank-0 override provider（常量 `dsh-mcp-manager-override`），全层级可禁用——用 `ctx.skills.registerProvider` 注册一个 `list()` 返回 `overrideSkills` 的 provider（rank 0 压过所有内置 provider）。
- **持久化**：`<profileDir>/dsh-skill-manager.json`，`{ version: 1, disabledSkills: string[] }`。
- **热重载**：`ctx.on('skills/change', ...)` 触发 `ensureRestored()` 懒重放禁用状态。
- **skill 名校验**：`/^[a-z0-9]+(?:-[a-z0-9]+)*$/`，非法名直接拒绝。

### 4.5 已知坑：`ctx.on` 的类型

`ctx.on('skills/change', ...)` 在 TS 下报类型错，需断言：

```ts
;(ctx.on as (event: string, callback: () => void) => unknown)('skills/change', () => { void ensureRestored() })
```

## 五、浏览器半（lib/client.js）

- 用 `window.__ModuleLoader__.load({ id, factory })` 注册 bundle，**注册两个 id**：`dsh-mcp-manager`（loader entry id）与 `@xxxyz/dsh-mcp-manager`（scoped 包名）——未用到的那个是惰性的。
- MCP 管理页：`slots.inject('settings.section', ...)`，`id: 'mcp-manager'`，order 16。
- Skills 管理页：`slots.inject('settings.section', ...)`，`id: 'skill-manager'`，order 17。分组逻辑 `skmLevelOf(s)` 互斥判定（项目级/运行时/自定义/用户级/内置/插件自带），再按 provider 折叠。
- 与宿主通信一律 `fetch('/dsh-mcp-manager/api')` 带 `x-dsh-plugin` 头。

## 六、双挂载 guard（cordis.patch.yml）——**不要改短路顺序**

`cordis.patch.yml` 是 `dsh plugin add` 的 bundle patch，插入的 loader 行带一个 `disabled: !!js` 表达式：

```yaml
- insert:
    - id: dsh-mcp-manager
      name: '@xxxyz/dsh-mcp-manager'
      disabled: !!js "[...ctx.loader.entries()].some((e) => e.options.id !== 'dsh-mcp-manager' && (e.options.name === 'dsh-mcp-manager' || e.options.name === '@xxxyz/dsh-mcp-manager') && !e.disabled)"
```

- **用途**：当已有启用的旧挂载（旧 install.mjs 通道的 `id: mcp-manager` 行，或聚合包以别的 id 挂载本包）时，本行自动退让，避免 `/dsh-mcp-manager/api` 重复路由导致整个 plugin tree 启动失败。
- **短路顺序是硬约束**：`!e.disabled` 必须放**最后**。因为 `Entry.disabled` 是**无缓存 getter**（`_disabled → disabledOf → evaluate` 每次重算），若先访问 `e.disabled`，遍历到自身条目时会触发自身 getter 重入同一个表达式 → 无限递归 → boot 时 `Maximum call stack size exceeded`（2026-08-27 实测踩过，把整个 dsh web 打挂）。
- 回归测试 `tests/guard.test.mjs` 用会抛错的 `disabled` getter 证明表达式永不触碰自身 / 无关条目；改动 `cordis.patch.yml` 必须跑 `npm test`。

## 七、本机调试 / 部署

```text
npm run build && npm pack          # 生成 xxyz-dsh-mcp-manager-<版本>.tgz
dsh plugin --profile web remove @xxxyz/dsh-mcp-manager   # 先卸（清 node_modules）
dsh plugin --profile web add ./xxxyz-dsh-mcp-manager-<版本>.tgz
```

- **必须 remove 再 add**：若 node_modules 里同名包目录已存在，pnpm 不重新解压 tarball，装的还是旧内容。
- 改完重启 dsh web（host 半变更）+ 浏览器硬刷新（Cmd/Ctrl+Shift+R，client 变更）。
- 本机 profile 依赖优先用 npm registry 版本：`dsh plugin --profile web add @xxxyz/dsh-mcp-manager@^<版本> --registry=https://registry.npmjs.org`（国内镜像有同步延迟）。

## 八、发布流程

```bash
npm test                                  # 全绿
npm version <新版本号>                      # 更新版本 + 打 tag（semver：新功能 minor）
git push origin main --tags
npm publish --registry=https://registry.npmjs.org   # prepublishOnly 自动 build
```

发布后：
- 本机 `~/.dsh/profiles/web/package.json` 的依赖若是本地 tarball，需改回 `@xxxyz/dsh-mcp-manager@^<版本>` 再 `pnpm install`（否则删掉 tarball 后重装会挂）。
- 验证 `npm view @xxxyz/dsh-mcp-manager version --registry=https://registry.npmjs.org`。
- 重启 dsh web 确认新版本生效（mcpm-list / skill-list 探测 API）。
- **pnpm 11 minimumReleaseAge 供应链策略**：新版本发布不足 24h 时，`dsh plugin add @latest` 会**静默回退到旧版本**（`downloaded 0` + dependencies 仍写旧版 `^x.y.z`，无任何报错）。解法：把**新版本号追加进** `~/.dsh/profiles/web/pnpm-workspace.yaml` 的 `minimumReleaseAgeExclude`，且**必须用带版本号的 OR 范围形式**（`- '@xxxyz/dsh-mcp-manager@2.1.0 || 2.1.1 || ... || 2.1.4'`）——**不带版本号的形式实测 pnpm 不认**，装不上新版本。诊断命令：`pnpm add <pkg>@latest --lockfile-only --config.minimumReleaseAge=0`（若解析到新版本即坐实是 release-age 拦截）。
- **pnpm packument metadata 缓存**：即使 whitelist 正确，`pnpm add @latest` 也可能仍解析到旧版本（而 `pnpm view` 已见新版本）——`%LOCALAPPDATA%\pnpm-cache\v11\metadata\registry.npmjs.org\@xxxyz\dsh-mcp-manager.jsonl` 缓存了旧 dist-tags。解法：删除该缓存（`node -e "fs.rmSync(path, {recursive:true, force:true})"`）再 `dsh plugin add`。
- **Cordis 插件读取 loader 行 config**：entry 配置是 Cordis 调 `callback(ctx, config)` 时作为**第二参数**传给 `apply` 的（`apply(ctx, config)`）——**绝不能从 `ctx.config` 读**：它不是注入服务，访问抛 `cannot get property "config" without inject`，整个 plugin tree 启动失败（2.1.3 实测翻车、2.1.4 修复）。函数/对象形态插件相同。

## 九、维护注意

- **Windows PowerShell 写文件会带 BOM**：`Set-Content -Encoding UTF8` 在 Windows PowerShell 5.1 下写 UTF-8 **with BOM**，`JSON.parse` 会崩（`Unexpected token '﻿'`）。改 `~/.dsh/profiles/web/package.json` 用 node 读写，不用 PowerShell。
- 文档约定：`README.md` 中文主文档、`README_EN.md` 英文，不再有 `README.zh-CN.md`。
- `screenshots.json`（`["show.png"]`）按 awesome-dsh-plugin 收录策略保留在仓库根。

## 十、v3.0.0 · DSH 0.2.x 适配记录（2026-10-03）

本仓库（qianmang1 fork）针对 DeepSeek Harness 0.2.1-alpha.1 完成适配；上游 v2.2.7 面向 DSH 0.1.x。主要断点与修复：

1. **`settings.prepareDocument()` 语义变化**（`src/index.ts` → `ensurePaths`）：DSH ≥0.2.x 返回**当前 profile 的补丁路径**（`<home>/profiles/<profile>/cordis.patch.yml`，配置编辑器文档），不再是 home 级文档。旧逻辑"取父目录当主目录"会推导出错误的 home，导致项目级/全局补丁路径全错。现按文档路径形状识别：形如 `<…>/profiles/<profile>/cordis.patch.yml` 时直接推导 home（上溯两级）、profileDir（父目录）、projectPatch（文档本身）、globalPatch（home 下 `cordis.patch.yml`）；非该形状回退旧的探测逻辑（兼容老版本布局）。
2. **fs 服务 target 化**（≥0.2.x）：`fs.resolve()` 返回不透明 target 而非路径字符串，`stat/readText/writeText/listDir` 均接收 target；`writeText` 第 3/4 参变为写入意图守卫（`FsWriteIntent`）与 `AbortSignal`（第 5 参仍是 sandbox policy）。本插件所有调用本就遵循 resolve→use 模式，位置参数恰好兼容，调用点无需改动；仅更新了本地结构类型与注释。错误码 `FS_NOT_FOUND` 不变；`sandboxPolicy.resolve` 变为同步（对返回值 `await` 无害，模式 `danger-full-access` 仍在）。fs 沙箱只拦写入，`resolve`/读取不受 workspace 限制。
3. **浏览器端无 `timer` 服务**（`lib/client.js`）：DSH ≥0.2.x 的浏览器 context 不提供 `timer`（官方 client 插件都用 window 定时器）。`inject: ['timer']` 会让 bundle fiber 永久挂起、设置页永不出现。改为 `inject: ['slots']` + 原生 `setInterval/setTimeout`（useEffect 内正确清理）；槽位服务改经 `ctx.slots` 直接访问。
4. **安装期 peer 兼容检查**（≥0.2.0-rc.2 新增）：`dsh plugin add` 按 peerDependencies 校验 `@deepseek-ai/dsh-*` 范围（`semver.satisfies(runtimeVersion, range, {includePrerelease: true})`），不满足则拦截并回滚安装（可用 `dsh plugin allow-version <pkg@ver> --dsh-version <v> --accept-risk` 精确豁免）。peer 范围必须**覆盖用户实际运行的 runtime**，而不是只写最新版：v3.0.0 写 `^0.2.1-alpha.1` 时，DSH 0.2.0-rc.2 runtime 安装即被拒（0.2.0-rc.2 < 0.2.1-alpha.1 下界）；v3.0.1 放宽为 **`^0.2.0-rc.2`**（includePrerelease 下同时覆盖 0.2.0-rc.2 与 0.2.1-alpha.1，已用 DSH 同款判定式验证）。devDependencies 仍锁 `^0.2.1-alpha.1`（仅构建期类型，不参与安装检查）+ `@deepseek-ai/cordis@^4.0.5-alpha.1`（dsh-tools 0.2.1-alpha.1 的 peer 要求）。
5. **经验证未变、无需改动的兼容面**：`webServer.register({kind:'exact'})`（`@deepseek-ai/dsh-host-webserver`，handler 仍收 node 原生 req/res）；`defineTool`（parameters 属性表 + `required: true` 注解、`output.schema` + `render` 必填——本插件本就满足）；`tools.schemas()`（仍返回 `name/description/parameters` JSON Schema，`mcpm-tools` 预览逻辑不变）；skills provider（candidate 需带 `rank`/`locator`——override provider 本就返回二者；`get(name)` 单参调用兼容）；`'skills/change'` 事件；`ctx.loader.entries()` / `options.id/name` / `disabled` getter（双挂载 guard 表达式）；patch 方言 `!!js`；`dsh.bundle.patch` / `dsh.client`（platform/inject，inject 现为纯信息性字段；client.js 只 require `react`，在 PLATFORM_MODULES 基线内，无需 `external`）；`@deepseek-ai/dsh-mcp-client` config 字段（`serverName/transport/url/command/args/env/headers/toolCallTimeoutMs` 全保留，`disabled` 仍走 loader 行级）；`mcp__<serverName>__` 工具前缀；同源 fetch 假设（web 默认端口仍 3080）；`pluginInventory` 服务（`entryId/moduleName/enabled/fiberPhase` 字段不变，web-app bundle 有挂载）。
