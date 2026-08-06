## 一、技术栈
- 开发能力： Vue 2.6 + Vue CLI 4 + Vuex 3 + Element UI 2.x
	- Vue2.x 本身的性能较最新版本已经落后不少，如果出现框架方面的问题，治理起来比较棘手
		- 大对象组件内存占用高，由其组成的页面内存占用也会显著提升
		- 写法升级需要引入依赖，或者升级 到 vue2.7 已经支持，不需要引入插件（优先考虑，改动小、无破坏性）
		- 但组件内仍需写 `options` 结构（占用代码行数），`setup` 作为新增选项；没有 `<script setup>`，仅限 `setup()` 函数
		- less,css: 目前没有统一的 UI 样式模板。大部分组件或者页面内部样式内容占比不小。需要优化治理
			- 提取项目基础样式如：
				- 布局样式：padding, margin，line-height 等
				- 字体样式：font-size（标题1，2，3，4），font-family(不同浏览器会有差异)等
				- 颜色：主、次、默认、低级1等等
				- 弹框，提示，propmt （样式可以直接基于ui库）等
				- 组件风格（紧缩型，宽放型）等
	- 直接项目内升级 Vue3 不可行，需要接入微前端，进行模块拆分。代价较大，组件及页面都不能直接共用，vue2,3运行时的实例不一样，能力不兼容。
- Axios 0.21.0
	- `CancelToken` 是“Promise 通知 + 底层 abort”，可以中止客户端等待和网络连接，但不能撤销服务端业务。它最大的缺点是基于废弃提案、存在生命周期和内存管理问题、API 不通用
	- 升级 Axios 并且使用 `AbortController` API 接口替代原有的 CancelToken。相较 Axios `CancelToken`，`AbortController` 最大的优势是：它是 Web 标准，取消信号可以跨 API、跨请求复用，并且生命周期管理更成熟。
	- 当前版本的 CancelToken 类似于单例模式，升级后意识发布订阅机制处理，性能会有提升
- Element UI 2.x 配套 vue2.x 无法升级，要先将项目代码升级到 vue3 才能
- eslint： DSCP 没有做形成业务代码的质量规则体系
	- 依赖版本：
		- ESLint 7.15.0 
		- eslint-plugin-vue 7.2.0 ·
		- babel-eslint 10.1.0 
		- prettier 2.8.8（未接入）
		- lint-staged 10.5.3
	- 以下是eslint现状清单：由于目前项目文件较多，历史包袱较重，规则可以暂时限制新增文件生效。历史巨型文件待更进一步拆分需求后接入。历史小型文件可在开发需求过程中接入规则校验。

| 项              | 现状                                                                                     | 可补充                                                                                              |
| -------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 规则集            | `plugin:vue/essential`（Vue 最低级别，仅语法错误/严重反模式）+ `eslint:recommended`（基础错误规则），`rules` 为空  | rules添加单一文件行数限制，遍历层级限制，未catch error 处理等等功能                                                       |
| 解析器            | `babel-eslint`（官方已停维护）                                                                 | 可升级 @babel/eslint-parser                                                                         |
| lint-staged    | `src/**/*.{js,vue}` → `eslint --fix` + `git add`（**但未查看到 husky/yorkie hook 触发，疑似死配置**） | 补充相关配置项，保证 add 或者 commit 前进行lint 校验                                                              |
| Prettier       | 已装 + `.prettierrc`（3 项），**未与 ESLint/lint 流程集成**                                        | 需要接入，统一项目编码风格，预防出现千人千面的风格<br>`eslint-plugin-prettier` + `eslint-config-prettier` + `.prettierrc` |
| --max-warnings | ESLint CLI 的**警告数量参数**：lint 结果中 warning 数量超过该值，进程以非零码退出（CI 即失败）                        | 可以接入 CI 流程                                                                                       |
| vue相关          | 无                                                                                      | 补充：props、watch、computed、生命周期和事件规范                                                                |

- 自动化测试能力空白
	- 针对组件：props、事件、slots 和 ref API 缺少契约验证
	- 针对页面：helper、查询参数和导出逻辑缺少边界测试，大型页面拆分时无法证明行为保持不变
	- 异步失败、竞态和重复提交依赖人工验证
	- UI 测试？
## 二、代码现状
- 大型 Vue SFC 已成为主要架构风险，ExcelToPdfTool，cudQuotationNew，quotation/index.vue，afterSales/index.vue
	- 33 个 `.vue` 文件超过 1000 行
	- 14 个超过 2000 行
	- 4 个超过 4000 行
	- 最大文件超过 8400 行
	- 单组件和跨组件重复代码和逻辑较多
- 类型契约缺失，目前不适合直接迁移 ts,可以使用 jsDoc 补足单个功能点类型，再使用 ai 辅助强制校验。
	- API 请求参数和响应结构
	- DSCP 标准响应外层和分页结构
	- 组件 props 类型
	- 函数参数类型
	- 上传、PDF、Excel 等复杂数据结构
- 公共组件及 utils 工具函数现状
	- 公共组件抽象较弱，组件耦合 vue2 的模板语法与具体的业务功能，偏向于专一用途组件。开发边界模糊，大部分趋向于在同一个 index.vue 中编写模板，样式, 逻辑。功能不易拓展，到后期也容易形成难以维护的组件。
	- utils 内容较多，主要集中在 src/utils,但部分方法直接绑定到了 vue 实例上，容易被混淆和篡改。函数功能较全，但部分函数糅杂业务属性，且没有文档和辅助提示，导致开发使用起来不友好，有一定的使用成本。
	- 公共 utils 应该以纯函数或者纯功能形为主，命名要清晰具体，并且最好按功能收敛。
## 三、开发规范及模板demo
- 模板总览

| 模板     | 存放位置                                    | 使用场景           | 核心边界                 |
| ------ | --------------------------------------- | -------------- | -------------------- |
| 公共组件模板 | `src/components/ComponentName/`         | 跨页面复用          | 不包含页面路由和业务编排         |
| 业务页面模板 | `src/views/<module>/<page>/`            | 查询、表格、表单、弹窗页面  | `index.vue` 只负责状态和编排 |
| 页面私有组件 | `src/views/<module>/<page>/components/` | 页面专属弹窗、抽屉、复杂区域 | 不允许其他页面直接引用          |
- 组件开发
	- 目录范式
	```text
		src/components/BusinessSelect/
		├── index.vue      # Props、状态、computed、事件和生命周期
		├── const.js       # 有稳定常量时创建
		├── helper.js      # 有纯转换逻辑时创建
		└── readme.md      # Props、Events、Slots、ref API 和示例
	```
	- api 设计范式，基于 element ui 组件开发的组件，需要兼容原有组件的能力，不能导致开发后能力收缩，除非是特定用途的组件

| 能力            | 推荐方式             | 禁止方式             |
| ------------- | ---------------- | ---------------- |
| 数据输入          | Props            | 读取父页面状态          |
| 值变更           | `input`、`change` | 修改 prop          |
| 结构扩展          | Slots            | 大量 HTML 配置 props |
| 主动操作          | 精简 ref 方法        | 暴露完整组件实例         |
| Element UI 属性 | $attrs、作用域 Props | 丢弃未声明属性          |
| Element UI 事件 | $listeners 过滤透传  | 重复触发同一事件         |
| 父子通信          | Events           | $parent          |
| 数据转换          | `helper.js`      | 在模板中复杂计算         |
- 业务模板开发
```text
	src/views/order/order-list/
	├── index.vue                  # 状态、API 编排、组件协调
	├── const.js                   # 枚举、默认值、固定 UI 配置
	├── helper.js                  # 查询构建、校验、转换
	├── tableColumns.js            # 表格列工厂和渲染
	└── components/                # 页面专属业务区域
	    ├── index.js
	    ├── readme.md              # 备注完善的情况下，可以不需要
	    └── OrderEditDialog.vue    # 名称尽量表明功能用途
```

- 标准开发流程范式

| 阶段      | 公共组件                        | 业务页面                                         |
| ------- | --------------------------- | -------------------------------------------- |
| 1. 调研   | 检查复用证据和调用方                  | 检查需求、接口、路由和历史逻辑                              |
| 2. 定边界  | 明确 Props、Events、Slots       | 明确页面状态和业务流程                                  |
| 3. 分文件  | `index/const/helper/readme` | `index/const/helper/tableColumns/components` |
| 4. 实现   | 保留 Element UI 原生能力          | 优先复用公共组件和 API                                |
| 5. 异步处理 | 防重复、防竞态、可销毁                 | loading、异常和局部刷新                              |
| 6. 整理   | 清理无用 API 和样式                | 清理 Mock、无用状态和重复逻辑                            |
| 7. 验收   | 检查全部调用方                     | 检查主流程及相关既有功能                                 |
 - 模板使用边界
 
|场景|应采用的模板|
|---|---|
|一个页面独有的编辑弹窗|页面私有组件模板|
|多个页面复用的选择控件|公共组件模板|
|查询列表页面|业务页面完整模板|
|简单详情页|只保留需要的 `index.vue`、`const.js`、`helper.js`|
|没有复杂表格列|不创建 `tableColumns.js`|
|没有稳定常量或纯函数|不创建空的 `const.js`、`helper.js`|
|修改历史页面|只在需求涉及范围应用模板，不整体迁移|

## 五、分支管理规范及review 加强

- 遵从[研发规范](https://page.weixin.qq.com/smartpage/p/b1_AbgAyQZOAAcUKbirOF3SUC1EOeK?p=1jigjo&roomid=Person%3A1688856990820141%3A1688854299588079&open_source=wecomprivate&clickStart=1785907975843&_cef_session_id_=5ccfa9542e81df4f9abf958d1eb0a534&HadConsumeCst=1)约定，按规范，开新分支，合并代码
- 增强 review 和 pr 流程
	- 多人相互之间 review， 通过 pr 的差异代码重点检查：改动范围带来的影响，改动内容是都符合开发规范，代码风格是否符合组件及业务页面规范
	- 大需求必要时需要单独抽时间重点集中 review
	- 组件开发的话，review 前需要告知修改的功能
	- 以上都可以通过代码校验攻击，ai辅助等能力实现，但是人工最后一道墙是防止代码修改触及到个人未知的业务范围内，像可阅读性约束还是需要人为把关一下
	- 使用 ai 生成，此次commit 的影响范围及回归范围，保证自测和测试环节问题回归的全面性
## 六、commit 规范
- 遵从[研发规范](https://page.weixin.qq.com/smartpage/p/b1_AbgAyQZOAAcUKbirOF3SUC1EOeK?p=1jigjo&roomid=Person%3A1688856990820141%3A1688854299588079&open_source=wecomprivate&clickStart=1785907975843&_cef_session_id_=5ccfa9542e81df4f9abf958d1eb0a534&HadConsumeCst=1)约定，尽量详细描述本次修改内容，可借助 ai 能力自动生成。
	- 可以按功能逐个生成 commit 并提交，推荐这个方法，防止其中某一个功能因为特殊原因不用上线时更友好的剔除对应功能的 commit 内容。
	- 可以一次性生成所有功能相关信息并作为一次 commit 提交。出现上述问题时，需要找到对应功能文件逐个回退，如果改功能涉及到大量文件修改时，回退成本高。
## 七、项目重构或者升级
- 基于目前的技术栈直接在现有的项目上升级成本比较大，原因如下
	- vue2,vue3 不向下兼容，vue3 运行时很多内置方法，vue2 无法使用。两个实例同时存在也会产生bug。在当前 Vue CLI 工程中直接混编 Vue 2、Vue 3 `.vue` 文件，可行性低，维护成本和风险接近整体升级。
	
| 问题             | 影响                                 |
| -------------- | ---------------------------------- |
| 两套 SFC 编译器     | Webpack 配置复杂，容易错误匹配                |
| 两套 Vue Runtime | 包体积和运行内存增加                         |
| Router 不兼容     | Vue 3 页面不能直接作为 Router 3 的普通组件      |
| Vuex 注入不兼容     | Vue 3 子树无法自然使用现有 $store            |
| UI 库不兼容        | Element UI 不能用于 Vue 3              |
| 插件体系不同         | `Vue.use()`、`Vue.prototype` 无法直接复用 |
| DevTools       | 两套 Runtime 调试体验不稳定                 |
| HMR            | Vue Loader 与热更新链路容易冲突              |
| `keep-alive`   | Vue 2 不能正常缓存 Vue 3 组件实例            |
| 生命周期           | 页面切换时容易发生挂载和卸载遗漏                   |
| 测试成本           | 每次构建配置升级都需验证双编译链                   |
- 可采用vue3 微前端主应用 + vue3， vue3 子应用  开发模式，逐步使用vue3 开发新业务，并且替换原有的业务，最终实现 vue3 完全实现升级。后续也可以实现对现有业务功能的应用拆分，并且也能使用 react 子应用
	```text
	Vue 3 主应用 Shell
	├── 登录认证
	├── Layout
	├── 菜单与权限
	├── 主路由
	├── 标签页与导航
	├── 全局通知
	├── 微前端生命周期管理
	│
	├── Vue 3 子应用 A：订单域
	├── Vue 3 子应用 B：物流域
	├── Vue 3 子应用 C：财务域
	├── React 子应用 D：考勤
	│
	└── Vue 2 Legacy 子应用
	    ├── 未迁移系统页面
	    ├── Vue Router 3
	    ├── Vuex 3
	    └── Element UI
	```
- ```mermaid
flowchart TB
    User["用户 / 浏览器"]

    subgraph Shell["Vue 3 主应用 Shell"]
        direction TB

        Entry["统一应用入口"]

        subgraph Platform["平台基础能力"]
            direction LR
            Auth["登录认证"]
            Layout["Layout 布局"]
            Permission["菜单与权限"]
            Router["主路由"]
            Navigation["标签页与导航"]
            Notification["全局通知"]
        end

        Lifecycle["微前端生命周期管理<br/>注册 · 加载 · 挂载 · 卸载 · 通信 · 异常处理"]

        Entry --> Auth
        Auth --> Permission
        Permission --> Router
        Router --> Layout
        Layout --> Navigation
        Notification -.全局能力.-> Layout
        Router --> Lifecycle
    end

    User --> Entry

    subgraph Apps["业务子应用层"]
        direction LR

        subgraph Vue3Apps["Vue 3 业务子应用"]
            direction TB
            Order["Vue 3 子应用 A<br/>订单域"]
            Logistics["Vue 3 子应用 B<br/>物流域"]
            Finance["Vue 3 子应用 C<br/>财务域"]
        end

        ReactApp["React 子应用 D<br/>考勤"]

        subgraph Legacy["Vue 2 Legacy 子应用"]
            direction TB
            LegacyPages["未迁移系统页面"]
            VueRouter3["Vue Router 3"]
            Vuex3["Vuex 3"]
            ElementUI["Element UI"]

            LegacyPages --> VueRouter3
            LegacyPages --> Vuex3
            LegacyPages --> ElementUI
        end
    end

    Lifecycle -->|"路由分发 / 生命周期"| Order
    Lifecycle -->|"路由分发 / 生命周期"| Logistics
    Lifecycle -->|"路由分发 / 生命周期"| Finance
    Lifecycle -->|"路由分发 / 生命周期"| ReactApp
    Lifecycle -->|"兼容加载 / 生命周期"| Legacy

    Permission -.权限上下文.-> Apps
    Notification -.通知服务.-> Apps
    Navigation -.导航状态.-> Apps

    Shared["共享基础设施<br/>组件库 · API SDK · Design Token<br/>监控埋点 · 国际化 · 工具库"]

    Shared -.共享能力.-> Order
    Shared -.共享能力.-> Logistics
    Shared -.共享能力.-> Finance
    Shared -.共享能力.-> ReactApp
    Shared -.兼容适配.-> Legacy
```
- 职责划分

| 能力         | 主应用负责  | 子应用负责       |
| ---------- | ------ | ----------- |
| 登录与退出      | 是      | 否           |
| Token 生命周期 | 是      | 消费          |
| 用户信息       | 是      | 消费          |
| 权限基础数据     | 是      | 消费          |
| 一级菜单       | 是      | 提供菜单元数据或不参与 |
| 主路由分发      | 是      | 内部路由        |
| Layout     | 是      | 业务内容区域      |
| 标签页        | 是      | 通知路由变化      |
| 全局消息       | 提供统一服务 | 调用服务        |
| 业务 API     | 否      | 是           |
| 业务状态       | 否      | 是           |
| 业务组件       | 否      | 是           |
| 子应用加载和卸载   | 是      | 实现生命周期      |
- 组件开发
	- 独立组件库项目，提供完整的文档网站，demo 实例
	- 通过 npm 依赖引入项目中
	- 为了兼容 vue2 vue3 项目，使用 **共享内核、单一契约、双 Vue 适配层**  的方案，对于同一组件保证核心业务能力不重新设计和实现，共用一套 api 参数和使用方法
	- 打包产出两套组件分别针对 vue2项目 和 vue3项目