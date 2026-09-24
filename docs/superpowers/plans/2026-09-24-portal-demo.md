# 企业案件门户演示版 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **本轮只编写计划，不执行任何任务。**

**Goal:** 在现有前端新增可点击、刷新后保留状态的中文企业案件门户演示页，覆盖设计文档所列 2.1—2.8 功能。

**Architecture:** 新增 `/[locale]/demo` 独立路由，将演示领域类型、虚构种子数据、客户端持久化和纯函数选择器放在 `src/features/portal-demo`。页面组件只调用该模块，不调用 NestJS、Zoho 或现有 NextAuth 登录。共享 `DemoState` 保证权限、案件进展、资料和报表口径一致。

**Tech Stack:** 项目现有 Next.js 16、React 19、TypeScript、Tailwind 4、`@o2s/ui`、Vitest、Playwright；浏览器 `localStorage` 与 Web Crypto。

## Global Constraints

- 设计依据：`docs/superpowers/specs/2026-09-24-portal-demo-design.md`，实施前先由用户审阅。
- 只修改 `apps/frontend` 下的演示功能及必要的前端测试配置；不改 NestJS、Prisma、现有登录页、首页、环境文件和 `package-lock.json`。
- 用户入口 `/demo` 应进入演示页，实际页面 `/en/demo` 可直接访问；界面中文。只使用虚构 `.example` 邮箱及案件。
- 预置演示账号密码为 `Demo123!`；持久化时只存盐和派生摘要，不存明文密码或上传文件内容。
- 文件仅接受 PNG、JPEG、CSV、XLS、XLSX，单文件不超过 10 MB；保存资料元数据，不提供刷新后的文件下载。
- 所有浏览器端权限均只用于演示，界面应标明模拟注册、验证码、PG 更新和审核。
- 保留本工作区已有的 `apps/api-harmonization/.env.local` 与 `package-lock.json` 改动，不暂存或覆盖它们。

---

## 文件结构与接口

| 路径 | 职责 |
| --- | --- |
| `apps/frontend/src/app/[locale]/demo/page.tsx` | 演示页入口与静态元数据；输出 `<body><PortalDemo /></body>`，符合现有 `[locale]/layout.tsx` 输出 `<html>` 的结构。 |
| `apps/frontend/src/features/portal-demo/model.ts` | `DemoState`、`DemoUser`、`DemoCase`、`DemoDocument`、`DemoFilters` 等领域类型。 |
| `apps/frontend/src/features/portal-demo/seed.ts` | 两家公司、办理人白名单、预置账号、项目、案件、进展和通知。 |
| `apps/frontend/src/features/portal-demo/security.ts` | Web Crypto 密码摘要、验证及模拟验证码生成。 |
| `apps/frontend/src/features/portal-demo/store.tsx` | 客户端状态初始化、版本化 localStorage、动作、重置和状态提示。 |
| `apps/frontend/src/features/portal-demo/selectors.ts` | 可见案件、树、筛选、资料/通知可见性和报表统计的纯函数。 |
| `apps/frontend/src/features/portal-demo/PortalDemo.tsx` | 导航、响应式壳层和工作区切换。 |
| `apps/frontend/src/features/portal-demo/AuthPanel.tsx` | 登录、注册和模拟邮箱验证。 |
| `apps/frontend/src/features/portal-demo/CasesPanel.tsx` | 树、列表、组合筛选、详情、进展和模拟 PG 更新。 |
| `apps/frontend/src/features/portal-demo/PermissionsPanel.tsx` | 管理员成员列表及逐案授权。 |
| `apps/frontend/src/features/portal-demo/DocumentsPanel.tsx` | 文件元数据登记及模拟 PG 审核。 |
| `apps/frontend/src/features/portal-demo/ReportsPanel.tsx` | 基于可见案件的统计和筛选。 |
| `apps/frontend/src/features/portal-demo/ProfilePanel.tsx` | 用户资料与改密。 |
| `apps/frontend/src/features/portal-demo/*.spec.ts` | 领域规则、持久化和数据隔离单元测试。 |
| `apps/frontend/vitest.config.mjs`、`apps/frontend/package.json` | 给前端新增局部 Vitest 配置与固定的 `test:demo` 测试命令。 |

关键接口统一为：`getVisibleCases(state: DemoState, userId: string): DemoCase[]`；`filterCases(cases: DemoCase[], filters: DemoFilters): DemoCase[]`；`getReport(cases: DemoCase[]): DemoReport`。组件经 `useDemoStore(): { state, dispatch, storageWarning }` 访问状态，动作只能修改当前公司范围内的数据。

### Task 1: 演示数据模型与持久化底座

**Files:** Create `model.ts`, `seed.ts`, `security.ts`, `store.tsx`, `store.spec.ts`, `apps/frontend/vitest.config.mjs`; modify `apps/frontend/package.json` to add `test:demo`.

**Interfaces:** Produces `DemoState`, `DemoAction`, `createSeedState(): Promise<DemoState>`, `loadDemoState(): Promise<DemoState>`, `saveDemoState(state): void`, `useDemoStore()`。`DemoState` 包含 `version: 1`、`users`、`handlers`、`companies`、`projects`、`cases`、`documents`、`notices`、`sessionUserId`；`DemoCase.caseScope` 固定为 `'案件' | '企业案件'`，种子数据两类均至少一条。使用固定键 `o2s-portal-demo-v1`。

- [ ] 写 `store.spec.ts`：首访生成两家公司与账号；损坏 JSON/旧版本安全重置；状态写入、刷新读取后保留权限和进展；存储失败回退内存。
- [ ] 加入最小 Vitest 配置和 `test:demo` 脚本，运行 `npm run test:demo --workspace=@o2s/frontend -- src/features/portal-demo/store.spec.ts`，确认缺失实现导致失败。
- [ ] 实现模型、种子数据与状态层。密码摘要使用 Web Crypto PBKDF2、每个账号独立随机盐；示例：`derivePassword(password, salt) -> Promise<string>`，不把原密码写入 `DemoState`。对 localStorage 解析结果做版本及必要字段校验；挂载后读取以避免 SSR hydration 不一致。
- [ ] 运行上述局部测试并通过；检查 `JSON.stringify(state)` 不包含 `Demo123!`。仅提交本任务涉及的前端文件。

测试及接口示例：

```ts
const state = await createSeedState();
expect(state.version).toBe(1);
expect(new Set(state.companies.map((company) => company.id)).size).toBe(2);
expect(JSON.stringify(state)).not.toContain('Demo123!');
export const DEMO_STORAGE_KEY = 'o2s-portal-demo-v1';
```

### Task 2: 登录注册、办理人绑定和个人资料

**Files:** Create `AuthPanel.tsx`, `ProfilePanel.tsx`, `auth.spec.ts`; modify `store.tsx` with账号动作。

**Interfaces:** Consumes `useDemoStore()` 与 `handlers` 白名单。`issueDemoCode(email)` 返回 `{ email, code, expiresAt }`；`register(state, { email, password, code }, challenge, now)` 返回 `Promise<{ state, user }>`；`signIn(state, { email, password })` 返回 `Promise<{ ok, userId? }>`。组件动作 `signOut()`、`updateProfile({ name, phone })`、`changePassword({ currentPassword, nextPassword })` 经 store 更新状态。注册结果从办理人记录复制 `companyId` 与 `pgHandlerId`，不得由用户自由指定。

- [ ] 写 `auth.spec.ts`：白名单邮箱注册成功并绑定正确公司/办理人 ID；重复注册、未知邮箱、错误验证码失败；错密码不能登录；正确旧密码才能改密；明文密码不进入持久化对象。
- [ ] 运行局部测试，确认预期失败。
- [ ] 实现登录/注册两种视图及模拟验证码：每次发码生成 6 位数字，当前注册会话内显示，有效期 5 分钟，验证成功或重发后旧码失效；页面用醒目文字说明不会发送真实邮件。加入个人资料和改密界面，错误与成功状态可见。
- [ ] 重跑局部测试，确认状态序列化和重新读取后账号可再次登录；浏览器验证留到 Task 6。仅提交本任务文件。

测试及动作示例：

```ts
const challenge = issueDemoCode('new.handler@north.example');
const registered = await register(state, { email: challenge.email, password: 'Example123!', code: challenge.code }, challenge, Date.now());
expect(registered.user.companyId).toBe('company-north');
expect(registered.user.pgHandlerId).toBe('pg-handler-new');
expect(await signIn(registered.state, { email: 'new.handler@north.example', password: 'wrong' })).toEqual({ ok: false });
```

种子办理人名单必须包含上述测试邮箱及绑定 ID，注册角色固定为 `member`，初始 `allowedCaseIds` 为空。

### Task 3: 可见性、案件树、详情和进展

**Files:** Create `selectors.ts`, `selectors.spec.ts`, `CasesPanel.tsx`; modify `store.tsx` with `advanceCase` 动作。

**Interfaces:** Produces `getVisibleCases(state, userId)`、`filterCases(cases, filters)`、`getCaseTree(cases, projects, companies)`、`getVisibleDocuments(state, userId)`、`getVisibleNotices(state, userId)`；`DemoCase.caseScope` 为 `'案件' | '企业案件'`，与业务类型字段分开。store 动作 `advanceCase(caseId)` 仅推进当前用户可见案件的预定义阶段，并同时写入更新时间、进展节点及通知。

- [ ] 写 `selectors.spec.ts`：管理员可见本公司全部案件；成员只见授权案件；跨公司 ID 无效；“案件/企业案件”各有样例且可分别查询或合并查看；关键词、案件类别、状态、阶段、业务类型、地区、负责人、日期区间组合筛选；排序；树为空分支不显示。
- [ ] 运行局部测试，确认预期失败。
- [ ] 实现公司 → 项目 → 案件树、字段丰富的列表、案件类别切换和详情时间线。筛选器使用同一个 `DemoFilters` 状态，列表与详情均从可见集合取值；模拟 PG 更新按有限状态机推进并新增通知，界面显示“演示操作”。
- [ ] 重跑局部测试，确认不同身份的树与详情数据以及模拟进展刷新后的状态。浏览器验证留到 Task 6。仅提交本任务文件。

测试及选择器示例：

```ts
const visible = getVisibleCases(state, 'member-north');
expect(visible.every((item) => item.companyId === 'company-north')).toBe(true);
expect(filterCases(visible, { status: ['进行中'], keyword: '许可' }).every((item) => item.status === '进行中')).toBe(true);
```

`DemoFilters` 的每个字段均可为空；空字段不限制结果。排序默认按最后更新时间降序。

### Task 4: 成员权限、资料与模拟审核

**Files:** Create `PermissionsPanel.tsx`, `DocumentsPanel.tsx`, `permissions-documents.spec.ts`; modify `store.tsx` with `setCaseAccess`、`addDocument`、`reviewDocument` 动作。

**Interfaces:** 纯函数 `setCaseAccess(state, actorId, memberId, caseId, allowed)` 只允许同公司管理员操作本公司普通成员；`addDocument(state, actorId, { caseId, fileName, mimeType, size })` 只接受当前用户可见案件；`reviewDocument(state, actorId, documentId, status, reason)` 只允许该公司管理员。store 调用这些函数并持久化返回状态。`DemoDocument` 只存文件元数据。

- [ ] 写局部测试：普通成员不可改权限或审核；跨公司目标被拒绝；授权取消后案件、资料、通知从该成员视图消失；非法类型或超过 10 MB 被拒绝；审核退回必须填写理由；状态在刷新后保留。
- [ ] 运行局部测试，确认预期失败。
- [ ] 实现逐案勾选授权矩阵和图片/表格选择界面。用文件扩展名与 MIME 双重校验，允许 PNG/JPEG/CSV/XLS/XLSX；只从 `File` 读取名称、类型、大小，不调用 `FileReader`，不保存字节。审核控件明确标注“模拟 PG 审核”。
- [ ] 重跑局部测试，确认成员动作被拒绝、管理员动作更新状态；浏览器验证留到 Task 6。仅提交本任务文件。

动作拒绝示例：

```ts
expect(() => setCaseAccess(state, 'member-north', 'member-north', 'case-north-1', true)).toThrow('仅公司管理员可设置权限');
expect(() => addDocument(state, 'member-north', { caseId: 'case-south-1', fileName: 'x.csv', mimeType: 'text/csv', size: 12 })).toThrow('无权访问该案件');
```

`setCaseAccess`、`addDocument` 和 `reviewDocument` 接口首参统一为 `DemoState`，次参为操作人 ID；成功时返回新的 `DemoState`，不就地修改旧状态。

### Task 5: 报表、壳层与路由

**Files:** Create `ReportsPanel.tsx`, `PortalDemo.tsx`, `apps/frontend/src/app/[locale]/demo/page.tsx`, `reports.spec.ts`; modify `selectors.ts` only for报表纯函数。

**Interfaces:** `getReport(filterCases(getVisibleCases(state, userId), filters))` 返回总数、按状态/阶段计数、逾期数、近期更新数；报表与案件页共享同一筛选模型及可见性来源。

- [ ] 写 `reports.spec.ts`：权限收紧后报表数量减少；模拟进展后阶段分布变化；筛选后统计一致；空结果显示 0 而非错误。
- [ ] 运行局部测试，确认预期失败。
- [ ] 完成中文导航、总览、案件、资料、报表、成员权限、个人资料与退出入口。管理员才显示权限工作区。服务端页面设静态标题并输出 `<body><PortalDemo /></body>`；不触发 SDK 页面请求。宽屏与窄屏均可操作，交互控件有标签和键盘焦点。
- [ ] 重跑局部测试；运行 `npm exec --workspace=@o2s/frontend -- tsc --noEmit` 和针对新增文件的 ESLint，修复本次引入的错误。仅提交本任务文件。

报表口径示例：

```ts
const scopedCases = filterCases(getVisibleCases(state, userId), filters);
const report = getReport(scopedCases);
expect(report.total).toBe(scopedCases.length);
expect(Object.values(report.byStatus).reduce((a, b) => a + b, 0)).toBe(report.total);
```

### Task 6: 浏览器验收与交付说明

**Files:** 本任务不新增文件；记录浏览器验收结果并修复本轮实现引入的问题。

- [ ] 在不修改现有环境文件的前提下启动 `npm run dev --workspace=@o2s/frontend`，分别访问 `http://localhost:3000/demo` 与 `http://localhost:3000/en/demo`。如果国际化中间件未让 `/demo` 到达演示页，在 Task 5 的路由范围内补明确跳转后重新验证。若启动依赖项缺失，先记录完整错误日志再修复演示版范围内的问题。
- [ ] 走通注册 → 验证 → 登录 → 个人资料 → 案件筛选/树/详情 → 模拟 PG 更新 → 资料上传/审核 → 管理员授权 → 成员视图 → 报表流程。刷新后核对持久化，重置后核对回到种子状态；用两家公司账号检验隔离。
- [ ] 用 `npm run test:demo --workspace=@o2s/frontend`、前端类型检查和新增文件 ESLint 复验；检查 git diff 只包含演示版文件与测试配置，确认原有用户改动未覆盖。
- [ ] 向用户交付访问地址、演示账号、启动命令、已验证事项和模拟功能限制。只有在验证通过后才报告完成。

## 范围外与后续接入

本计划不实现 Zoho API、真实邮件、真实文件上传、生产账号安全或服务端权限。正式对接阶段应新增 NestJS 端领域接口，映射 Zoho 办理人 ID/企业/案件，接入真实身份认证、文件存储与事件同步，并将前端本地状态替换为 API 数据源；不可把此演示状态机制直接上线。
