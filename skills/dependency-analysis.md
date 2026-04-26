# 依赖分析 Skill - Dependency Analysis

## 功能描述
对 Vue2 项目的所有依赖项进行全面分析，判断哪些依赖需要升级、替换或移除，并提供兼容的版本对照表和依赖安装脚本。

## 触发条件
当依赖分析专家（AGENT-DEP-003）启动时自动调用，用于执行所有依赖分析的具体工作。

---

## 1. 依赖分类体系

### 1.1 运行时依赖分类

| 类别 | 说明 | 处理方式 |
|------|------|----------|
| **Vue 核心生态** | vue, vue-router, vuex, vue-i18n 等 | 需升级到 Vue3 版本 |
| **UI 组件库** | Element UI, Ant Design Vue, Huadesign 等 | 需升级到 Vue3 版本或替换 |
| **工具库** | lodash, axios, dayjs, moment 等 | 一般无需变更，版本按需升级 |
| **Vue 插件** | vue-xxx 形式的插件 | 需检查 Vue3 兼容性 |
| **隐式依赖（事件总线等）** | 通过 `Vue.prototype.xxx = new Vue()` 定义的运行时模式 | 需先分析 `main.js`/`main.ts` 识别隐式依赖，然后引入 `mitt` 等替代库 |
| **构建工具** | webpack, babel 等 | 替换为 Vite 生态 |
| **样式工具** | less, sass, postcss 等 | 可保留，版本按需升级 |

### 1.2 Vue 核心依赖版本对照

| 依赖名 | Vue2 版本 | Vue3 兼容版本 | 说明 |
|--------|----------|--------------|------|
| `vue` | ^2.6.x / ^2.7.x | ^3.4.x | 核心框架升级 |
| `vue-router` | ^3.x | ^4.x | API 彻底改变 |
| `vuex` | ^3.x | ❌ 废弃 | 建议使用 Pinia 替代 |
| `pinia` | - | ^2.x | Vue3 官方状态管理 |
| `vue-template-compiler` | ^2.x | ❌ 废弃 | Vue3 内置模板编译 |
| `@vue/compiler-sfc` | - | 内置 Vite 插件 | 替换 Vue2 的模板编译器 |
| `vue-i18n` | ^8.x | ^9.x | API 有变化 |
| `vue-meta` | ^2.x | ❌ 废弃 | 使用 Vue Router meta 或组合式 API |
| `mitt` | - | ^3.x | 事件总线替代方案。检测到 `Vue.prototype.$eventBus = new Vue()` 时需引入 |

### 1.3 常见 UI 组件库版本对照

| 组件库 | Vue2 版本 | Vue3 版本 | 说明 |
|--------|----------|-----------|------|
| Element UI | element-ui ^2.x | element-plus ^2.x | 包名、API 均改变 |
| Ant Design Vue | ant-design-vue ^1.x | ant-design-vue ^4.x | 兼容 Vue3 |
| Huadesign Vue | huadesign-vue ^1.x | huadesign-vue ^2.x (需确认) | 按 `huadesignVue转化规则.md` 处理 |
| Vant | vant ^2.x | vant ^4.x | 移动端组件库 |
| View Design (iView) | view-design ^4.x | ❌ 已停止维护 | 建议替换 |

### 1.4 构建工具版本对照

| 工具名 | Vue2 版本 | Vue3 版本 | 说明 |
|--------|----------|-----------|------|
| webpack | ^4.x / ^5.x | ❌ 废弃 | Vite 取代 |
| vite | - | ^5.x | Vue3 推荐构建工具 |
| babel | @babel/core ^7.x | ❌ 可选 | esbuild 取代大部分功能 |
| @vitejs/plugin-vue | - | ^5.x | Vite 的 Vue 插件 |
| vue-loader | ^15.x / ^17.x | ❌ 废弃 | @vitejs/plugin-vue 取代 |
| babel-plugin-import | ^1.x | ❌ 废弃 | Vite 使用 unplugin-auto-import/unplugin-vue-components |

### 1.5 测试工具版本对照

| 工具名 | Vue2 版本 | Vue3 版本 | 说明 |
|--------|----------|-----------|------|
| @vue/test-utils | ^1.x | ^2.x | API 变化 |
| vue-jest | ^4.x / ^5.x | ❌ 废弃 | 使用 vitest 或 @vue/vue3-jest |
| vitest | - | ^1.x | Vue3 推荐测试框架 |
| jest | jest ^27.x | jest ^29.x / vitest | 可保留或迁移到 vitest |

---

## 2. 依赖扫描与识别

### 2.1 扫描源

| 扫描源 | 识别目标 | 优先级 |
|--------|----------|--------|
| `package.json` | dependencies, devDependencies, peerDependencies | P0 |
| `index.html` | `<script>` 标签外部库、`<link>` 外部样式 | P0 |
| `main.js` / `main.ts` | Vue.prototype 注入、事件总线、全局组件注册 | P0 |
| `vue.config.js` | webpack 插件引用、构建配置 | P1 |
| `babel.config.js` | babel 插件 | P1 |
| `.browserslistrc` | 浏览器兼容目标 | P2 |
| `.env.*` | 环境变量配置 | P2 |

### 2.2 main.js/main.ts 隐式依赖识别

扫描并识别以下 Vue.prototype 注入模式：

```javascript
// 模式1：事件总线
Vue.prototype.$eventBus = new Vue()
Vue.prototype.EventBus = new Vue()
Vue.prototype.eventBus = new Vue()
Vue.prototype.$bus = new Vue()

// 模式2：全局 API 注入
Vue.prototype.$http = axios
Vue.prototype.$api = apiInstance
Vue.prototype.$utils = utils

// 模式3：全局注册
Vue.component('MyButton', MyButton)
Vue.directive('focus', { ... })
Vue.mixin({ ... })
Vue.use(ElementUI)
Vue.filter('currency', fn)
```

**识别结果记录格式：**
```json
{
  "implicit_dependencies": [
    {
      "type": "event_bus",
      "pattern": "Vue.prototype.$eventBus = new Vue()",
      "source_file": "/path/to/main.js",
      "line": 10,
      "action": "install_mitt",
      "notes": "需引入 mitt 库替代"
    },
    {
      "type": "global_property",
      "name": "$http",
      "pattern": "Vue.prototype.$http = axios",
      "action": "migrate_to_global_properties",
      "notes": "迁移到 app.config.globalProperties.$http"
    }
  ]
}
```

---

## 3. 兼容性分析

### 3.1 分析维度

| 维度 | 说明 | 判断依据 |
|------|------|----------|
| **版本号分析** | 对比 npm registry 中版本号与 Vue3 的兼容性 | npm 版本号、semver 范围 |
| **社区活跃度** | 检查库的维护状态 | GitHub 最近更新、issue 回复 |
| **官方声明** | 检查官方文档是否声明支持 Vue3 | 官方 README、官网 |
| **替代方案** | 对不支持 Vue3 的库寻找替代方案 | npm 趋势、社区推荐 |

### 3.2 依赖处理动作

| 动作 | 说明 | 条件 |
|------|------|------|
| `upgrade` | 升级到兼容 Vue3 的版本 | 库支持 Vue3，仅需升级版本 |
| `replace` | 替换为其他库 | 库不支持 Vue3，有替代方案 |
| `keep` | 保持现有版本不变 | 库与 Vue3 兼容 |
| `remove` | 移除依赖 | Vue3 不再需要该依赖 |
| `install` | 新增依赖 | 隐式依赖需要显式安装（如 mitt） |
| `unknown` | 需要人工确认 | 无法判断兼容性 |

---

## 4. 特殊依赖处理

### 4.1 Vue.prototype 全局 API

Vue2 → Vue3 转换：

```javascript
// Vue2
Vue.prototype.$http = axios

// Vue3
import { createApp } from 'vue'
const app = createApp(App)
app.config.globalProperties.$http = axios
```

**需记录：**
- 全局 API 的名称和类型
- 被使用的组件范围（通过代码搜索 `this.xxx`）
- Vue3 中的实现方式

### 4.2 过滤器（filters）

Vue3 移除了过滤器，处理方式：

| 场景 | 处理方式 |
|------|----------|
| 全局过滤器 | 注册为 `app.config.globalProperties.$filters` |
| 局部过滤器 | 转为普通函数或计算属性 |
| 管道格式化 | 使用组合式函数替代 |

### 4.3 混入（mixins）

| Mixin 类型 | 处理建议 |
|------------|----------|
| 简单 mixin（无状态） | 直接迁移，Options API 兼容 |
| 复杂 mixin（有状态/逻辑） | 重构为组合式函数（Composable） |

### 4.4 插件（Vue.use）

```javascript
// Vue2
Vue.use(MyPlugin)

// Vue3
const app = createApp(App)
app.use(MyPlugin)
```

**检查清单：**
- [ ] 插件是否支持 Vue3（检查 npm 包名和版本）
- [ ] 插件 API 是否变化（如 Vue Router 从 new Router() → createRouter()）
- [ ] 插件是否需要额外配置

### 4.5 自定义指令

钩子映射：

| Vue2 | Vue3 |
|------|------|
| `bind` | `beforeMount` |
| `inserted` | `mounted` |
| `update` | `updated` |
| `componentUpdated` | `beforeUpdate` |
| `unbind` | `unmounted` |

### 4.6 事件总线（Event Bus）

**识别方式：** 扫描 `main.js`/`main.ts` 中 `Vue.prototype.xxx = new Vue()` 模式。

```javascript
// Vue2 事件总线
Vue.prototype.$eventBus = new Vue()
// 使用：this.$eventBus.$emit() / this.$eventBus.$on()

// Vue3 使用 mitt 替代
import mitt from 'mitt'
const eventBus = mitt()
app.config.globalProperties.$eventBus = eventBus
// 使用：eventBus.emit() / eventBus.on()
```

**TypeScript 类型声明：**
```typescript
// src/types/global.d.ts
import { Emitter } from 'mitt'
declare module 'vue' {
  interface ComponentCustomProperties {
    $eventBus: Emitter<{ 'event-name': string }>
  }
}
export {}
```

**内存泄漏注意：** 所有 `$eventBus.on()` 需在 `onUnmounted` 时配对 `$eventBus.off()`。

---

## 5. 外部依赖处理

### 5.1 外部 Script 标签处理策略

| 情况 | 处理方式 |
|------|----------|
| 兼容 Vue3 的 CDN 资源 | 保留并升级版本 |
| 不兼容 Vue3 的 CDN 资源 | 替换为兼容版本或 npm 包 |
| 项目内部脚本 | 迁移到 Vue3 项目中 |
| 已弃用的外部资源 | 移除 |
| 需要 npm 安装的 | 改为 npm 包引入 |

---

## 6. 报告生成

### 6.1 报告结构

```json
{
  "report_version": "1.0",
  "generated_at": "2024-12-30T14:30:00Z",
  "project_name": "old_project",
  "summary": {
    "total_dependencies": 50,
    "need_upgrade": 15,
    "need_replace": 3,
    "compatible_no_change": 25,
    "can_remove": 5,
    "unknown": 2
  },
  "categories": {
    "vue_core": [
      {
        "name": "vue",
        "current_version": "^2.6.14",
        "target_version": "^3.4.0",
        "action": "upgrade",
        "priority": "critical",
        "notes": "Vue 核心框架升级"
      }
    ],
    "ui_library": [
      {
        "name": "element-ui",
        "current_version": "^2.15.6",
        "action": "replace",
        "replacement": "element-plus",
        "target_version": "^2.5.0",
        "priority": "high",
        "notes": "element-ui 不支持 Vue3，需替换为 element-plus"
      }
    ],
    "regular_libraries": [
      {
        "name": "axios",
        "current_version": "^0.24.0",
        "target_version": "^1.6.0",
        "action": "upgrade",
        "priority": "low",
        "notes": "常规升级"
      },
      {
        "name": "lodash",
        "current_version": "^4.17.21",
        "target_version": "^4.17.21",
        "action": "keep",
        "priority": "none",
        "notes": "无需变更"
      },
      {
        "name": "mitt",
        "current_version": "（未安装，隐式依赖）",
        "target_version": "^3.0.0",
        "action": "install",
        "priority": "high",
        "condition": "检测到 Vue.prototype.$eventBus = new Vue() 模式",
        "notes": "替代 Vue2 事件总线。Vue3 移除了 $on/$off/$once 方法"
      }
    ]
  },
  "removed_dependencies": [
    {
      "name": "vue-template-compiler",
      "reason": "Vue3 不需要，已内置",
      "notes": "移除该依赖"
    },
    {
      "name": "vue-loader",
      "reason": "Vite 使用 @vitejs/plugin-vue 代替",
      "notes": "移除该依赖，替换为 @vitejs/plugin-vue"
    }
  ],
  "external_scripts": [
    {
      "source": "index.html",
      "script_src": "https://cdn.example.com/some-lib.js",
      "action": "keep | upgrade | remove | replace",
      "notes": "处理说明"
    }
  ],
  "conflicts": [
    {
      "dependencies": ["package-a", "package-b"],
      "conflict_type": "version_conflict",
      "solution": "升级 package-a 到 x.x.x 以兼容 package-b"
    }
  ]
}
```

### 6.2 依赖安装脚本

```bash
# ===== 运行时依赖安装 =====
npm install vue@^3.4.0
npm install vue-router@^4.3.0
npm install pinia@^2.1.0
npm install element-plus@^2.5.0
npm install axios@^1.6.0

# 事件总线（条件性安装：检测到 Vue.prototype.$eventBus = new Vue() 时执行）
npm install mitt@^3.0.0

# ===== 开发依赖安装 =====
npm install -D vite@^5.0.0
npm install -D @vitejs/plugin-vue@^5.0.0
npm install -D typescript@^5.3.0
npm install -D vue-tsc@^2.0.0

# ===== 移除不兼容依赖 =====
npm uninstall vue-template-compiler vue-loader vuex