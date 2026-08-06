## 一、分析背景

本文基于 2026-08-05 生成的新 Graphify 结果，以及 `mg-dscp-admin` 当前源码和工程配置，对项目下一阶段需要补足的前端专项能力进行分析。

本次 Graphify 结果来自纯代码 AST 提取：

- 扫描代码文件：593 个
- 图谱节点：7681 个
- 图谱边：10662 条
- 社区：580 个
- Token 消耗：0
- 未检测到导入循环

当前项目是 Vue 2.6 + Vue CLI 4 + Vuex 3 + Element UI 2.x 的 JavaScript 管理后台工程。现有 Skill 已较好覆盖业务页面、API、公共组件、主题样式、前端规范和变更审查，下一阶段更需要补充的是质量保障、复杂度治理和风险控制能力。

## 二、总体结论

当前最需要补足的不是一个泛化的“前端开发 Skill”，而是以下专项能力：

| 优先级 | 建议补充的 Skill | 核心职责 |
|---|---|---|
| P0 | `dscp-eslint-quality-guard` | ESLint 规则、静态质量检查和增量质量门禁 |
| P0 | `dscp-vue2-testing` | Vue 2 组件、纯函数和业务编排自动化测试 |
| P0 | `dscp-large-sfc-refactoring` | 超大型 Vue 单文件组件拆分和遗留页面治理 |
| P1 | `dscp-type-contracts` | JSDoc、`@ts-check` 和渐进式 TypeScript 类型契约 |
| P1 | `dscp-async-error-handling` | 请求竞态、异常处理、loading 和重复提交治理 |
| P1 | `dscp-frontend-security` | 上传、导出、富文本、OSS、权限和文件安全 |
| P2 | `dscp-accessibility` | 焦点、键盘、弹窗、表单错误提示和语义化交互 |
| P2 | `dscp-dependency-modernization` | Vue CLI、Axios、ESLint 等旧工具链的渐进升级 |

推荐优先创建前三个 Skill。TypeScript 不应排在 ESLint、测试和大型页面治理之前。

## 三、现状与主要风险

### 3.1 ESLint 只具备最低限度配置

当前 ESLint：

- 仅继承 `plugin:vue/essential` 和 `eslint:recommended`
- `rules` 为空
- 使用较旧的 `babel-eslint`
- lint 仅覆盖 `.js`、`.vue`
- lint-staged 执行 `eslint --fix` 后直接 `git add`

这说明项目已安装静态检查工具，但尚未形成适合 DSCP 业务代码的质量规则体系。

主要缺口包括：

- Promise 和异步错误处理检查
- props、watch、computed、生命周期和事件规范
- 空 `catch`、不可达代码、重复分支和危险类型转换检查
- 模板 `key`、属性透传和事件处理检查
- 新增代码与存量代码的分级质量门禁
- ESLint 自动修复对用户已有改动的安全控制

### 3.2 自动化测试能力基本空白

当前 `src` 目录约包含：

- Vue 文件：395 个
- JavaScript 文件：197 个
- API 文件：89 个
- 公共组件一级目录：45 个

未识别到 Jest、Vue Test Utils、Cypress、Playwright 等完整测试体系。名称包含 `test` 的文件主要是业务演示或模拟数据，不能构成自动化回归保障。

主要风险是：

- 公共组件的 props、事件、slots 和 ref API 缺少契约验证
- helper、查询参数和导出逻辑缺少边界测试
- 大型页面拆分时无法证明行为保持不变
- 异步失败、竞态和重复提交依赖人工验证

### 3.3 大型 Vue SFC 已成为主要架构风险

项目中：

- 33 个 `.vue` 文件超过 1000 行
- 14 个超过 2000 行
- 4 个超过 4000 行
- 最大文件超过 8400 行

典型热点包括：

- `src/components/FrontTools/excelFiles/ExcelToPdfTool.vue`
- `src/views/productsLib/quotation/components/cudQuotationNew.vue`
- `src/views/productsLib/quotation/index.vue`
- `src/views/afterSalesManage/afterSales/index.vue`

Graphify 也显示 PDF、Excel、上传、报价、查询构建等区域形成了高复杂度社区，并存在大量同名 `handleQuery()` 等页面方法。

同名方法不能直接认定为重复代码，但说明查询编排和页面职责分散，需要专项治理。

### 3.4 类型契约缺失，但暂不适合全面 TypeScript 迁移

当前源码中没有 `.ts` 或 `.tsx` 文件，Graphify 中也没有 TypeScript 结构节点。

类型能力的真实需求主要来自：

- API 请求参数和响应结构
- DSCP 标准响应外层和分页结构
- 表格行数据和列配置
- 动态表单配置
- Dialog/Drawer 的 `open(payload)` 参数
- 上传、PDF、Excel 等复杂数据结构

考虑到当前是 Vue 2.6 + Vue CLI 4 + Babel ESLint 工程，直接全面迁移 TypeScript 会产生较高的构建、模板类型和存量代码改造成本。应先补类型契约能力，再根据收益决定是否迁移。

### 3.5 异步和异常处理缺少统一模型

Graphify 高连接节点包含多个 `handleQuery()`、`handleCalcShippingFee()` 和 `getTickets()`，反映出大量页面承担请求编排职责。

需要重点治理：

- loading 生命周期未统一闭合
- 请求竞态和旧响应覆盖新响应
- 页面销毁后的异步状态写入
- 重复提交和批量操作的部分成功
- Blob 下载错误响应解析
- 只输出日志、不反馈用户的异常分支
- API 层和页面层错误职责边界不清

### 3.6 文件、富文本和权限场景安全风险较高

项目包含 OSS、文件上传、PDF、Excel、富文本、iframe、文件预览和权限指令等功能。

应重点补足：

- 扩展名、MIME 和文件大小联合校验
- 下载文件名和响应头安全解析
- Object URL 创建与释放
- `v-html` 和富文本 XSS 边界
- iframe、外链和预览 URL 白名单
- OSS 临时凭证生命周期
- Excel 公式注入防护
- 日志和导出内容中的敏感信息检查
- 前端显示权限与后端鉴权的职责边界

## 四、建议新增 Skill 细节

### 4.1 `dscp-eslint-quality-guard`

#### 目标

建立适用于 Vue 2 + JavaScript 存量项目的静态质量基线，并支持渐进收紧规则。

#### 应覆盖的能力

- 检查现有 ESLint、Babel 和 Vue 插件版本兼容性
- 区分错误、警告和建议规则
- 为新增文件启用严格规则，为存量文件保留可执行基线
- 检查 Promise、空 `catch`、未处理异常和重复代码信号
- 检查 Vue props、watch、computed、事件和生命周期用法
- 检查模板 `key`、危险 HTML、无效属性和事件修饰符
- 审核 lint-staged 是否会修改或暂存无关文件
- 输出规则变更的影响文件和分阶段整改方案
- 默认只执行局部检查，不扩大到全项目 lint

#### 与现有 Skill 的边界

- `frontend-spec`：定义通用编码规范
- `dscp-eslint-quality-guard`：把规范转换成可执行检查和质量门禁
- `vue-admin-change-reviewer`：审查具体变更风险

### 4.2 `dscp-vue2-testing`

#### 目标

为 Vue 2 项目建立可落地的测试策略，为公共组件和大型页面重构提供行为保护。

#### 测试分层

1. 纯函数测试
   - `helper.js`
   - 查询参数构建
   - 日期、金额和状态转换
   - 导出文件名解析
   - 数组过滤、合并和去重

2. 组件契约测试
   - props 默认值与显式值
   - `v-model`
   - `input`、`confirm`、`cancel` 事件顺序
   - slots
   - `$attrs`、`$listeners` 透传
   - `open()`、`close()` 和返回值
   - 异步成功、失败和销毁场景

3. 页面编排测试
   - 查询、重置和分页
   - 批量操作
   - 请求失败和重试
   - 权限按钮
   - 文件导入导出
   - 快速切换条件引发的请求竞态

#### 标准测试矩阵

- 默认配置
- 显式配置
- 空数据
- 边界值
- 异常数据
- 事件顺序
- 异步成功与失败
- 组件销毁后的回调
- 属性与监听器透传
- 键盘和焦点
- 大数据量

### 4.3 `dscp-large-sfc-refactoring`

#### 目标

治理超过 1000 行的 Vue 单文件组件，在保持业务行为不变的前提下逐步降低复杂度。

#### 强制工作流

1. 使用 Graphify 分析目标目录依赖关系
2. 识别页面的状态域、请求域、配置域、渲染域和副作用域
3. 列出对外契约、事件顺序和关键响应式依赖
4. 补充 characterization tests 或明确人工回归基线
5. 优先拆固定配置和纯函数
6. 再拆页面私有组件和复杂渲染区域
7. 最后处理共享逻辑和跨页面复用
8. 每一步执行局部差异、影响范围和缺陷风险检查

#### 推荐拆分归属

- 固定枚举、选项、按钮和基础配置：`const.js`
- 纯转换、校验和动态配置构建：`helper.js`
- 表格列：`tableColumns.js`
- 页面私有弹窗、抽屉和区域组件：`components/`
- 跨页面通用纯逻辑：`src/components/utils/` 或 `src/utils/`
- API 请求：`src/api/`

#### 特殊风险检查

- 定时器和事件监听器释放
- iframe、图片和字体加载等待
- Object URL 释放
- 大数组复制和深度监听
- 事件触发顺序
- ref 命令式 API
- 异步请求竞态
- Element UI 弹层焦点与层级

### 4.4 `dscp-type-contracts`

#### 目标

不强制全面迁移 TypeScript，先建立结构化数据契约和渐进式类型检查能力。

#### 三阶段方案

第一阶段：JSDoc 类型契约

- DSCP API 标准响应
- 分页响应
- 查询参数
- 表格行模型
- 表单配置
- 组件 props 和事件负载
- Dialog/Drawer `open(payload)`

第二阶段：局部类型检查

- 对纯 JavaScript helper 使用 `// @ts-check`
- 为共享数据模型建立集中 typedef
- 校验 API 参数和响应的字段一致性
- 校验配置工厂的输入和输出

第三阶段：按收益引入 TypeScript

- 优先用于新建的复杂纯逻辑模块
- 优先用于高复用数据模型
- 不优先改造大型 `.vue` 页面
- 不在业务改造中顺带全面迁移

### 4.5 `dscp-async-error-handling`

#### 目标

统一异步请求、错误提示和状态生命周期，降低查询、保存和批量操作中的竞态风险。

#### 应覆盖的能力

- `try/catch/finally` 与 loading 闭合
- 最新请求生效策略
- 请求取消或忽略旧响应
- 页面销毁后的回调保护
- 重复提交控制
- 防抖和节流适用边界
- 批量操作的部分成功反馈
- Blob 和文件下载错误解析
- 权限错误、登录失效、业务错误和网络错误分层
- API 层与页面层异常职责
- 日志与用户提示的分工

### 4.6 `dscp-frontend-security`

#### 目标

针对 DSCP 中的上传、导出、富文本、OSS、PDF、Excel 和权限功能建立专项安全审查能力。

#### 应覆盖的能力

- 文件类型、MIME、大小和内容一致性检查
- 文件名和响应头解析安全
- XSS、`v-html` 和富文本清洗
- iframe 与外部 URL 白名单
- OSS 凭证和上传策略生命周期
- Excel 公式注入
- Object URL 和内存释放
- 敏感信息日志和导出检查
- 权限按钮与后端鉴权边界
- 第三方依赖已知风险和升级影响分析

## 五、现有组件 Skill 的增强建议

现有 `dscp-admin-component-usage` 已覆盖 props、事件、slots、ref API、helper/const 拆分、属性和监听器透传、README、主题、性能和内存风险。

因此不建议再创建职责重叠的“组件规范 Skill”，而应增强现有 Skill：

- 增加组件契约测试模板
- 增加键盘和焦点检查
- 增加异步竞态与销毁检查
- 增加性能预算和大数据量验证
- 增加受控与非受控状态规范
- 增加默认配置和显式配置兼容性检查
- 增加 README 与真实 props、events、slots 的一致性校验
- 增加公共组件变更的兼容性分级

## 六、建议的 Skill 路由

```text
新增业务页面
└─ dscp-bussiness-dev
   ├─ API：dscp-api-builder
   ├─ 公共组件：dscp-admin-component-usage
   ├─ 样式：dscp-ui-theme-css-spec
   ├─ 类型契约：dscp-type-contracts
   └─ 测试：dscp-vue2-testing

大型存量页面重构
└─ graphify
   └─ dscp-large-sfc-refactoring
      ├─ dscp-eslint-quality-guard
      ├─ dscp-vue2-testing
      ├─ dscp-async-error-handling
      └─ vue-admin-change-reviewer

文件、富文本、OSS、PDF、Excel 需求
└─ dscp-bussiness-dev
   ├─ dscp-frontend-security
   ├─ dscp-async-error-handling
   └─ dscp-vue2-testing
```

## 七、推荐实施顺序

### 第一批：建立质量底座

1. `dscp-eslint-quality-guard`
2. `dscp-vue2-testing`
3. `dscp-large-sfc-refactoring`

这三个 Skill 先解决“怎么发现问题、怎么证明行为没变、怎么安全拆分复杂页面”。

### 第二批：补足可靠性

4. `dscp-type-contracts`
5. `dscp-async-error-handling`
6. `dscp-frontend-security`

### 第三批：体验与长期升级

7. `dscp-accessibility`
8. `dscp-dependency-modernization`

## 八、为什么 TypeScript 不排在第一位

当前最大风险不是“代码没有写成 TypeScript”，而是：

- 复杂存量代码缺少可验证的行为边界
- 超大型 Vue 文件职责过多
- 公共组件缺少契约测试
- 异步和异常处理缺少统一模型
- ESLint 没有形成项目级质量门禁

如果先全面迁移 TypeScript，会同时产生类型、构建和业务代码变化，却不能直接证明现有行为正确。

更合理的顺序是：

```text
ESLint 基线
→ 测试与行为保护
→ 大型页面拆分
→ JSDoc/@ts-check 类型契约
→ 新模块按收益使用 TypeScript
```

## 九、Graphify 结论的使用边界

这份图谱来自代码 AST，导入关系和显式调用关系可信度较高，但缺少语义提取，因此需要注意：

- 没有 `accessibility` 节点不等于页面完全没有无障碍处理
- 同名函数可能来自不同页面，不能直接认定为重复代码
- 580 个社区且多数聚合度较低，适合发现热点，不适合直接作为组件拆分边界
- 是否抽取共享逻辑仍需回到源码比较参数、状态、事件和返回值契约
- PDF、Excel、上传和报价社区应优先作为专项审查范围，而不是一次性整体重构

## 十、影响范围

本分析只涉及 Graphify 输出、工程配置和源码结构，没有修改项目代码、依赖、构建配置或现有 Skill。

后续落地新 Skill 时，主要影响：

- `.eslintrc.js` 和 lint-staged 质量策略
- 公共组件开发与审查流程
- 业务页面拆分和回归验证流程
- API、表格、表单和弹窗的数据契约
- 上传、导出、PDF、Excel、富文本和权限功能的风险检查

## 十一、回归验证建议

创建上述 Skill 后，建议选择以下三类样本进行试运行：

1. 一个公共组件
   - 验证 props、事件、slots、ref API、属性透传和 README 检查是否准确

2. 一个 API 模块
   - 验证请求参数、标准响应、分页结构、异常职责和 JSDoc 类型是否完整

3. 一个超过 2000 行的业务页面
   - 验证 Graphify 分析、配置归属、拆分顺序、状态流转和回归清单是否可执行

试运行至少覆盖：

- 主流程
- 相关既有功能
- 默认配置与显式配置
- 空数据和边界值
- 请求成功、失败和竞态
- 权限差异
- 组件销毁与资源释放
- 文件上传和导出异常
- 已识别的高风险交互

