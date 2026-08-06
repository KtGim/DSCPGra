
> 项目：mg-dscp-admin（Vue 2.6 + Vuex 3.6 + Vue Router 3）  
> 分析范围：src/store、路由守卫、请求拦截器、布局、页签、权限、商户选择和主要调用方  

## 1. 总体结论

当前 Vuex Store 规模不大，共有 6 个模块：

- app：侧边栏、设备类型、界面尺寸。
- user：Token、用户信息、角色和权限。
- permission：后端动态路由和侧边栏路由。
- tagsView：已访问页签和 keep-alive 缓存名单。
- settings：主题和布局设置。
- merchant：订单页面共享的商户筛选条件。

整体属于典型 Vue 2 管理后台 Store。大部分业务页面仍将查询、表格和弹窗状态保留在页面内部，没有把 Vuex 变成全量接口数据仓库，这是合理的。

当前主要问题：

1. 会话没有统一生命周期：退出时没有统一清理用户、权限、路由、页签和商户状态。
2. 模块规范不一致：user、permission 非 namespaced，其他模块 namespaced。
	1. stroe.dispatch(tagViews/XXX), namespaced
	2. stroe.dispatch(/login), 非 namespaced
3. 路由、权限与 Store 强耦合：permission 同时负责接口、业务修正、组件解析和状态写入。
4. 状态契约缺乏校验：存在缺失字段、未实现 action、权限默认放行和跨用户持久化残留。
5. 页面大量直接读取嵌套 state，公共 getter 覆盖不完整。

当前无需立即迁移 Pinia。应先在 Vuex 3 内完成会话重置、模块契约、错误处理和状态归属治理，再在 Vue 3 阶段选择 Vuex 4 或 Pinia。

## 2. Store 结构

~~~mermaid
flowchart TD
    V["Vue 页面与布局"] --> G["根 getters"]
    V --> D["dispatch / commit"]
    G --> A["app"]
    G --> U["user"]
    G --> P["permission"]
    G --> T["tagsView"]
    G --> S["settings"]
    G --> M["merchant"]

    U --> C1["Admin-Token Cookie"]
    A --> C2["sidebarStatus / size Cookie"]
    M --> L1["merchantId / merchantName localStorage"]
    T -. "钉住页签在组件中维护" .-> L2["pinnedTabs localStorage"]
    P --> API1["GET /getRouters"]
    U --> API2["login / getInfo / logout"]
~~~

### 2.1 根 Store 与 getters

src/store/index.js 静态注册 6 个模块。src/store/getters.js 暴露布局、页签、用户、路由和商户的部分常用状态。

已发现两个契约问题：

- introduction getter 读取 state.user.introduction，但 user state 没有该字段。
- userInfo 被多个页面使用，却没有 getter，调用方直接读取 state.user.userInfo。

### 2.2 app 模块

状态包括 sidebar.opened、sidebar.withoutAnimation、device、size。

持久化方式：

- sidebarStatus 写 Cookie。
- size 写 Cookie。
- device 只保留在内存。

主要消费者是 Layout、Navbar、Sidebar、ResizeHandler 和 SizeSelect。该模块职责清晰，属于稳定的 UI Shell 状态。

### 2.3 user 模块

状态包括 token、name、avatar、roles、permissions、userInfo。

主要 action：

- Login：请求登录，写 Token Cookie 和 Store。
- GetInfo：请求用户、角色、权限，建立用户上下文。
- LogOut：调用服务端退出，成功后清 Token、roles、permissions。
- FedLogOut：不调用服务端，只清 Store Token 和 Cookie。

该模块是登录态核心，但没有完整定义“会话开始、初始化、过期、退出”时所有用户相关状态应该如何变化。

### 2.4 permission 模块

状态包括 routes、addRoutes、sidebarRouters。

GenerateRoutes 当前流程：

1. 调用 /getRouters。
2. 修改 Local/edit 的 meta.activeMenu。
3. JSON 深拷贝两份后端路由。
4. 分别构建 Sidebar 路由和 Router 动态路由。
5. 把字符串 component 映射为 Layout、ParentView 或 views 下的 Vue 文件。
6. 追加通配 404。
7. 提交 routes、addRoutes 和 sidebarRouters。
8. 返回 accessRoutes 给 router.addRoutes。

它同时承担接口调用、DTO 适配、业务特例、组件解析、树转换和状态写入，职责过重。

### 2.5 tagsView 模块

状态包括 visitedViews 和 cachedViews。前者控制顶部页签，后者传给 keep-alive include。

TagsView 组件还自行维护 pinnedTabs 并写入 localStorage，因此页签系统实际存在两套状态：

- Vuex：visitedViews、cachedViews。
- 组件 data + localStorage：pinnedTabs。

### 2.6 settings 模块

状态包括 theme、sideTheme、showSettings、tagsView、fixedHeader、sidebarLogo，通过通用 CHANGE_SETTING 修改。

页面存在对 settings.showTop 的读取，但 Store 中没有声明该字段。

### 2.7 merchant 模块

保存 merchantId、merchantName，并同步写 localStorage。多个订单状态页共享该选择，页面通常在离开时手工 dispatch 空值清理。

## 3. 登录与初始化流转

~~~mermaid
sequenceDiagram
    participant L as Login.vue
    participant U as user Store
    participant API as 后端 API
    participant C as Cookie
    participant Guard as permission.js
    participant P as permission Store
    participant R as Vue Router

    L->>U: dispatch Login
    U->>API: POST /login
    API-->>U: token
    U->>C: setToken
    U->>U: SET_TOKEN
    L->>R: 跳转

    R->>Guard: beforeEach
    Guard->>C: getToken
    alt roles.length === 0
        Guard->>U: dispatch GetInfo
        U->>API: GET /getInfo
        API-->>U: user / roles / permissions
        U->>U: 设置用户上下文
        Guard->>P: dispatch GenerateRoutes
        P->>API: GET /getRouters
        API-->>P: 后端路由树
        P->>P: 转换动态组件与菜单
        P-->>Guard: accessRoutes
        Guard->>R: addRoutes
        Guard->>R: replace 当前导航
    else 已有 roles
        Guard->>R: 继续导航
    end
~~~

当前关键特征：

- Cookie 是否存在决定是否登录。
- roles.length === 0 同时充当“用户尚未初始化”的标志。
- 动态路由只在 GetInfo 成功后生成。
- 刷新后 Vuex 内存丢失，但 Token Cookie 保留，因此重新请求用户和路由。

## 4. 主要状态流

### 4.1 权限

/getInfo → user.roles / user.permissions → 根 getters → 路由守卫、权限指令、auth 插件及页面手工判断。

### 4.2 动态菜单

/getRouters → permission.GenerateRoutes → routes / addRoutes / sidebarRouters → router.addRoutes → Sidebar 和 TagsView affix 初始化。

### 4.3 keep-alive

路由变化 → tagsView/addView → visitedViews + cachedViews → AppMain keep-alive include → 按 route.name 缓存页面组件。

### 4.4 商户筛选

订单页面选择商户 → merchant/settingMerchant → mutation 写 state 和 localStorage → 其他订单页面读取 → 页面离开时手工清空。

## 5. 缺陷与不足

### 5.1 P0：FedLogOut 只清 Token

FedLogOut 仅清 SET_TOKEN 和 Admin-Token Cookie，没有清理：

- name、avatar、roles、permissions、userInfo。
- permission.routes、addRoutes、sidebarRouters。
- visitedViews、cachedViews。
- merchantId、merchantName。
- 用户维度的 pinnedTabs。

目前通常依赖 location.href 整页刷新让内存状态被动消失。未来若改为 SPA 内部跳转、HttpOnly Session 或不刷新退出，旧用户和旧权限会残留。

建议建立根级 resetSession action，由所有退出和 401 流程统一调用。

### 5.2 P0：LogOut 接口失败时无法本地退出

LogOut 只有服务端 /logout 成功后才清理本地状态。网络中断、后端 500 或 Session 已失效时，用户无法安全退出。

建议尽力调用服务端 logout，但无论成功失败都在 finally 执行本地 resetSession；服务端失败只做非阻断记录或提示。

### 5.3 P0：权限缺失默认赋予超级权限

GetInfo 使用 res.permissions || ['*:*:*']。后端漏传、返回 null 或字段异常时，前端会自动获得通配权限，属于 fail-open。

建议：

- 有效数组：规范化后使用。
- 有效字符串：转成数组。
- 缺失或非法：使用空数组。
- 只有后端明确返回超级权限时才允许 *:*:*。

后端必须继续执行最终权限校验。

### 5.4 P0：部分页面再次默认超级权限

部分业务页面和公共组件读取 permissions 时也使用 permissions || ['*:*:*']。即使 Store 改成空权限关闭，这些调用方仍会重新放行。

建议所有权限判断统一经过 hasPermission；空权限必须返回 false，禁止页面自行提供通配默认值。

### 5.5 P0：GenerateRoutes 不会 reject

GenerateRoutes 手工创建 Promise，但只提供 resolve。getRouters 失败、响应结构异常或树遍历抛错时：

- action 不 resolve。
- action 不 reject。
- 路由守卫持续等待。
- NProgress 可能一直显示。
- 页面可能空白或卡死。

建议改成 async action，让异常自然抛给路由守卫。

### 5.6 P0：假定所有顶层路由都有 children

当前直接执行 item.children.forEach。后端返回顶层叶子路由、children 为 null 或字段缺失时会抛 TypeError，并触发 GenerateRoutes 永久 pending。

建议所有树遍历使用 Array.isArray 守卫，并将 activeMenu 特例并入纯转换 helper。

### 5.7 P0：Tab 插件调用不存在的 action

src/plugins/tab.js 调用：

- tagsView/delLeftTags。
- tagsView/delRightTags。

tagsView 模块没有这两个 action 或 mutation，调用时会出现 unknown action type，功能不生效。

建议先核查真实调用方：产品需要则补齐 action、mutation、affix 保护和测试；无人使用则删除虚假的公共 API。

### 5.8 P1：roles.length 被当成初始化状态

路由守卫使用 roles.length === 0 判断是否加载用户。它混淆：

- 用户信息尚未加载。
- 用户真实没有角色。

当前通过 ROLE_DEFAULT 哨兵值规避重复加载，但它不是真实业务角色。

建议增加 initialized 和 status：idle、loading、ready、error。守卫根据 initialized 判断。

### 5.9 P1：初始化缺少并发锁

初始化期间 roles 仍为空。并发导航可能重复 dispatch GetInfo 和 GenerateRoutes。当前没有共享 in-flight Promise、loading 状态和可控重试。

建议由 bootstrapSession 保存单个初始化 Promise，所有导航复用同一任务。

### 5.10 P1：user 与 permission 没有命名空间

app、tagsView、settings、merchant 使用 namespaced；user、permission 使用根命名空间，造成 Login 与 app/toggleSideBar 两套调用风格，并增加 mutation/action 冲突风险。

建议渐进迁移为 user/login、user/fetchCurrentUser、permission/generateRoutes。可先保留根 action 兼容转发，不能一次性切换全部调用方。

### 5.11 P1：根 getter 与 state 契约不一致

introduction getter 指向不存在的 state 字段，始终 undefined。userInfo 是常用状态却没有 getter。

建议删除无效 getter，增加 currentUser、currentUserId、isAuthenticated 等稳定领域 getter。

### 5.12 P1：页面直接耦合 userInfo 内部路径

多个页面直接读取 this.$store.state.user.userInfo.userId。userInfo 初始为 null，部分读取没有空值保护，可能在用户初始化前报错。

建议通过 currentUserId getter 读取，并由 getter 对未初始化状态提供稳定的 null。

### 5.13 P1：GetInfo 原地修改接口响应

权限规范化直接修改 res.permissions 数组，导致 API 原始响应对象被 Store 改写，不利于调试和复用。

建议建立纯函数 normalizePermissions 和 normalizeUser，返回新数组/新对象后再 commit。

### 5.14 P1：GetInfo 缺少响应结构校验

当前假设 res.user 存在、avatar 是字符串、userName 存在。异常响应可能在 avatar.includes 处抛错。

建议先校验用户上下文；非法响应转换为明确错误并执行安全退出。

### 5.15 P1：Token 在 Cookie 与 Store 中重复

请求拦截器和路由守卫读取 Cookie，logout API 使用 state.token。Cookie 才是真正认证来源，Store token 是可能漂移的镜像。

建议确定单一事实源：

- Header Token 阶段由认证适配器统一管理，Store 仅提供登录状态。
- HttpOnly Cookie 阶段 Store 不保存 Session ID，只保存 authenticated 和用户上下文。

### 5.16 P1：动态路由没有 reset

permission 模块只有 SET，没有 RESET。退出或切换用户后，Store 路由和 Vue Router 3 matcher 可能保留旧用户路由。

SPA 退出必须同时：

1. 清 permission Store。
2. 调用 resetRouter 重建 matcher。
3. 清页签和缓存。
4. 再进入登录页。

### 5.17 P1：permission 模块职责过重

模块混合 API、特定业务路由修正、JSON 深拷贝、ParentView 展平、组件解析、404 注入和 Store commit，无法方便地做纯函数测试。

建议拆分：

~~~text
src/store/modules/permission.js
src/router/dynamic/normalize.js
src/router/dynamic/resolveView.js
src/router/dynamic/buildRoutes.js
src/router/dynamic/const.js
~~~

### 5.18 P1：settings.showTop 未声明

会员权益页面读取 state.settings.showTop，但 Store 中没有字段，条件永远得到 undefined。

建议确认归属：确需全局则在默认 settings 和 Store 中声明；仅属于页面则移回局部 state；已废弃则删除读取。

### 5.19 P1：merchant 持久化跨账号残留

merchantId 和 merchantName 写入 localStorage，退出时未统一清理。下一位用户可能继承上一用户筛选条件。

建议纳入 resetSession，并按 userId 隔离持久化 key。后端必须根据当前用户校验 merchantId。

### 5.20 P1：merchant mutation 包含持久化副作用

settingMerchant mutation 同时写 localStorage 和 state。虽然同步，但降低 mutation 的纯度和可测试性。

建议 action 负责持久化，mutation 只修改 state，或使用统一 persistence adapter。

### 5.21 P1：pinnedTabs 与 Vuex 分裂

pinnedTabs 在 TagsView 组件 data 中，另行写 localStorage；visitedViews 在 Vuex。初始化时组件再把钉住页签注入 Store。

问题包括：

- 两套结构需要手工同步。
- 退出时不按用户清理。
- 后端路由变化后旧标签可能失效。
- JSON.parse 没有异常保护。
- 组件混合持久化、校验、合并和 UI。

建议将 pinnedViews 纳入 tagsView 模块，并使用带版本和用户作用域的持久化 helper。

### 5.22 P1：visitedViews 只按 path 去重

同一路径不同 query 只能有一个页签。若产品需要同时打开多个详情对象，当前模型无法表达。

建议明确页签 key：

- 单实例页面使用 path。
- 多实例页面使用 fullPath 或 route.meta.tabKey。

### 5.23 P1：cachedViews 缺少 name 校验

ADD_CACHED_VIEW 直接使用 view.name。动态路由缺少 name 时可能加入 undefined；组件 name 与 route name 不一致时 keep-alive 不生效。

建议动态路由构建时校验唯一 name，需要缓存的页面同时校验组件 name。

### 5.24 P1：tagsView 存在大量无意义 Promise

多个 action 使用 new Promise 包装同步 commit。Vuex dispatch 本身会返回 Promise，这些包装增加复杂度。

建议 action 直接 commit 并 return 新状态；组合 action 使用 await 或同步 commit。

### 5.25 P2：直接读取嵌套 state 较多

静态检索发现约 49 个文件显式使用 Store，布局和多个业务页面直接读取 `$store.state.<module>.<field>`。

建议关键语义使用 getter；单一模块内简单 UI 状态可使用 mapState；权限、用户 ID、商户等必须使用领域 getter。

### 5.26 P2：settings 持久化策略不一致

sidebar 和 size 使用 Cookie；主题、固定 Header 和 Logo 只保留内存；pinnedTabs 与 merchant 使用 localStorage。用户无法预期刷新后的行为。

建议分类：

- 设备临时状态：内存。
- 用户偏好：用户作用域 localStorage 或后端配置。
- 会话业务上下文：退出时清理。
- 敏感认证信息：不进 localStorage。

### 5.27 P2：持久化 key 没有命名空间和版本

现有 key 包括 Admin-Token、sidebarStatus、size、merchantId、merchantName、pinnedTabs，没有应用前缀、用户维度和格式版本。

建议使用类似：

~~~text
dscp-admin:v1:<userId>:merchant
dscp-admin:v1:<userId>:pinned-tabs
dscp-admin:v1:preferences
~~~

### 5.28 P2：命名风格不统一

action 同时出现 Login、GetInfo、settingMerchant、toggleSideBar；mutation 同时有大写下划线和小写驼峰。

建议新代码统一小写驼峰 action，如 login、fetchCurrentUser、resetSession；mutation 在项目内保持一种风格。

### 5.29 P2：缺少严格模式和测试

Store 没有开发环境 strict，也缺少模块测试、初始化失败状态和状态重置矩阵。

建议优先测试 normalizePermissions、动态路由构建和 resetSession，再考虑仅开发环境开启 strict。

### 5.30 P2：AppMain computed 持续打印缓存状态

cachedViews computed 每次计算都会 console.log，造成控制台噪声并暴露内部路由名称。

建议删除或仅在显式 debug 开关下输出。

## 6. 推荐状态分层

| 层级 | 典型状态 | 推荐位置 |
| --- | --- | --- |
| 会话级 | 当前用户、角色、权限、初始化状态 | session/user |
| 路由级 | 动态路由、菜单、路由就绪状态 | permission |
| Shell 级 | Sidebar、设备、主题 | app/settings |
| 导航级 | visited、cached、pinned tabs | tagsView |
| 跨订单页面上下文 | 当前商户 | merchant，按用户隔离 |
| 页面级 | 查询、表格、弹窗草稿 | 页面 data |
| 请求缓存 | 列表和详情响应 | 默认不放 Vuex |

## 7. 推荐目标结构

~~~text
src/store/
├── index.js
├── getters.js
├── modules/
│   ├── app.js
│   ├── session.js
│   ├── permission.js
│   ├── tagsView.js
│   ├── settings.js
│   └── merchant.js
├── helpers/
│   ├── normalizeUser.js
│   ├── normalizePermissions.js
│   ├── resetState.js
│   └── persistence.js
└── README.md
~~~

动态路由纯逻辑应放在 src/router/dynamic，不继续堆入 Store。

## 8. 推荐会话状态机

~~~mermaid
stateDiagram-v2
    [*] --> anonymous
    anonymous --> authenticating: login
    authenticating --> authenticated: success
    authenticating --> anonymous: failed
    authenticated --> bootstrapping: fetch user and routes
    bootstrapping --> ready: valid context
    bootstrapping --> expired: invalid or 401
    ready --> expired: session expired
    ready --> loggingOut: logout
    expired --> loggingOut: resetSession
    loggingOut --> anonymous: reset user-scoped state
~~~

建议状态：

~~~text
status: anonymous | authenticating | bootstrapping | ready | expired | loggingOut
initialized: boolean
currentUser: null | User
roles: []
permissions: []
bootstrapError: null | ErrorSummary
~~~

路由守卫只依赖 status 和 initialized，不再用 roles.length 猜测初始化状态。

## 9. 统一 resetSession

退出、401 和用户切换使用同一入口：

~~~text
session/resetSession
  -> 清认证凭证
  -> 清 currentUser / roles / permissions
  -> permission/resetRoutes
  -> resetRouter matcher
  -> tagsView/resetViews
  -> merchant/resetMerchant
  -> 清用户维度 pinnedTabs
  -> 保留非用户设备偏好
~~~

建议保留：device、Sidebar 偏好、UI size、非用户敏感主题偏好。

必须清理：身份、权限、动态路由、页签缓存、当前商户、用户钉住页签及其他业务数据。

## 10. Vuex 开发规范

### 10.1 state

- 只保存真正跨页面共享的数据。
- 初始值类型稳定。
- 所有字段显式声明。
- 每个用户级字段说明退出清理规则。

### 10.2 getters

- 提供 currentUserId、isAuthenticated、hasPermission 等业务语义。
- 不保留永远 undefined 的 getter。
- 不产生副作用。
- 不允许权限超级兜底。

### 10.3 mutations

- 同步、短小，只改 state。
- 不发请求、不跳路由、不显示 Message。
- 持久化放 action 或 adapter。
- reset 使用初始状态工厂，避免漏字段。

### 10.4 actions

- 编排请求、持久化和多个 mutation。
- 使用 async/await，避免无意义 new Promise。
- 明确返回值和异常。
- 初始化、退出和动态路由必须可 reject。
- 并发初始化复用同一个 Promise。

### 10.5 命名空间

- 新模块必须 namespaced。
- 旧 user/permission 通过兼容层渐进迁移。
- 页面 dispatch 使用完整模块路径。

### 10.6 持久化

- 记录 owner、作用域、版本和清理时机。
- 用户数据按 userId 隔离。
- JSON.parse 捕获异常并回退。
- 不持久化 Route 实例、组件、Error、Promise。
- Token 不进入 localStorage。

## 11. 分阶段治理

### 第一阶段：确定性缺陷

1. 增加统一 resetSession。
2. FedLogOut 和 LogOut 都调用它。
3. 权限缺失使用空数组，删除页面超级权限兜底。
4. GenerateRoutes 改为可 reject 的 async action。
5. children 遍历增加数组保护。
6. 修复或删除 delLeftTags、delRightTags。
7. 明确 settings.showTop 归属。
8. 登出清理 merchant 和用户 pinnedTabs。

### 第二阶段：建立契约

1. 增加 session status 和 initialized。
2. 增加 currentUser、currentUserId、isAuthenticated getter。
3. 删除 introduction 死 getter。
4. 为路由、权限、页签定义 JSDoc。
5. 建立 Store README 和状态重置矩阵。

### 第三阶段：拆纯逻辑

1. 权限规范化抽 helper。
2. 用户响应规范化抽 helper。
3. 动态路由转换迁入 router/dynamic。
4. persistence 使用统一 adapter。
5. pinnedTabs 纳入 tagsView。

### 第四阶段：统一调用方式

1. user、permission 开启 namespaced。
2. 保留根 action 兼容层。
3. 分模块迁移页面 dispatch。
4. 关键 state 迁移到 getter/mapState。

### 第五阶段：Vue 3 准备

短期继续 Vuex 3。Vue 3 阶段可先升级 Vuex 4保持契约，或逐模块迁移 Pinia。优先迁移 app/settings/merchant；session/permission/tagsView 最后迁移。

不要在同一版本同时重写 Store、Router 和认证协议。

## 12. 建议测试清单

### 12.1 登录初始化

- 无 Token 访问白名单和受保护路由。
- 登录后只请求一次 GetInfo 和 getRouters。
- 用户无角色但账号合法。
- permissions 为空、null、字符串、数组和超级权限。
- getInfo 数据异常。
- getRouters 失败、空数组、顶层叶子和未知组件。

### 12.2 退出与过期

- 正常退出。
- logout 接口失败仍能本地退出。
- HTTP 401 和业务 401。
- 多请求同时 401 只 reset 一次。
- 退出后用户、权限、路由、页签和商户全部清空。
- A 用户退出后 B 用户不继承 A 状态。

### 12.3 动态路由

- 多级 ParentView。
- 顶层叶子。
- children null 或缺失。
- 未找到页面组件。
- 404 只添加一次。
- 切换用户后旧路由不可访问。

### 12.4 页签与缓存

- 新增、关闭、关闭其他、关闭全部。
- 左右侧关闭功能存在时正常工作。
- affix 不被删除。
- 同 path 不同 query 符合产品预期。
- route.name 缺失不污染 cachedViews。
- pinnedTabs 损坏安全回退。
- 退出后按用户清理。

### 12.5 商户与设置

- 多订单页面共享商户。
- 页面离开和退出清理。
- 用户切换不继承 merchantId。
- 后端拒绝无权限 merchantId。
- Sidebar、size、theme 刷新行为。
- showTop 明确归属后显示正确。

## 13. 影响范围

当前 Store 影响：

- 登录、用户信息、退出和 401。
- 动态路由、Sidebar、404 和页面访问。
- 按钮权限和页面权限判断。
- 页签及 keep-alive。
- 主题、Sidebar 和响应式布局。
- 订单模块跨页面商户选择。
- 约 49 个显式使用 Store 的源码文件，以及通过权限指令、路由和布局间接依赖的页面。

session、permission、tagsView 属于高风险基础设施，必须分批上线。app/settings 风险较低；merchant 清理影响多个订单页面，需要重点回归。

## 14. 回归测试点

实施 Store 修改后至少验证：

1. 登录、刷新恢复、GetInfo 和动态路由。
2. 正常退出、失败退出和会话过期。
3. 不同角色连续登录后的权限隔离。
4. Sidebar 菜单和按钮权限。
5. 动态路由叶子和未知组件异常。
6. 页签新增、刷新、关闭、关闭其他和缓存。
7. 商户在多个订单页面间共享和清理。
8. 主题、Sidebar、size 和移动端布局。
9. 持久化损坏和旧版本兼容。
10. reset 后是否符合“保留偏好、清理用户数据”的矩阵。

## 15. 最终建议

当前 Store 最需要的不是增加更多模块，而是建立统一的会话生命周期和可靠状态契约：

1. resetSession 统一登录失效、主动退出和用户切换。
2. 权限默认关闭，禁止缺失权限变成 *:*:*。
3. 用显式状态机替代 roles 数组初始化哨兵。
4. GenerateRoutes 正常 reject，路由转换拆成纯函数。
5. 修复 Tab 缺失 action 和 settings 缺失字段。
6. merchant、pinnedTabs 按用户隔离。
7. 统一 namespaced、getter、action 和 reset 规范。
8. 页面级业务状态继续保留在页面内部。
9. 完成 Vuex 治理后再迁移 Vue 3/Pinia。

按此顺序治理，可以在保持现有 Vue 2 页面稳定的同时，降低权限残留、导航卡死、跨用户状态污染和未来升级风险。

