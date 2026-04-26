# 质量验证 Skill (QA Validation)

## 概述

本 Skill 封装了 Vue2→Vue3 迁移项目的质量验证规则和自动化检查逻辑，供质量保障专家在迁移验证任务中调用。包含构建验证、运行时验证、代码质量扫描、构建产物检查等全面的验证规则。

---

## 一、项目结构检查规则

### 1.1 Vue3 项目必需文件清单

```json
{
  "required_files": [
    "package.json",
    "vite.config.ts",
    "tsconfig.json",
    "index.html",
    "src/main.ts",
    "src/App.vue",
    "src/env.d.ts",
    "src/router/index.ts",
    "src/stores/"
  ]
}
```

### 1.2 package.json 依赖检查规则

```json
{
  "dependencies_check": {
    "must_have": [
      "vue@^3.x",
      "vue-router@^4.x",
      "pinia@^2.x",
      "@vitejs/plugin-vue",
      "vite@^5.x"
    ],
    "must_not_have": [
      "vue@^2.x",
      "vue-router@^3.x",
      "vuex@^3.x",
      "vue-template-compiler",
      "webpack@^4.x || webpack@^5.x"
    ]
  }
}
```

### 1.3 vite.config.ts 配置检查

| 检查项 | 规则 | 示例 |
|-------|------|------|
| vue 插件 | 必须包含 `@vitejs/plugin-vue` | `plugins: [vue()]` |
| 路径别名 | 必须配置 `@` 指向 `src` | `resolve: { alias: { '@': '/src' } }` |
| 代理配置 | 必须包含原 Vue2 `devServer.proxy` 配置 | `server: { proxy: { '/api': ... } }` |
| 构建输出 | 确认 `outDir` 配置 | `build: { outDir: 'dist' }` |

---

## 二、构建验证规则

### 2.1 构建命令执行序列

```bash
# Step 1: 依赖安装
npm install --legacy-peer-deps

# Step 2: TypeScript 类型检查
npx vue-tsc --noEmit

# Step 3: ESLint 检查
npm run lint

# Step 4: 构建
npm run build

# Step 5: 检查构建产物
npx vite build --outDir ./dist-check
```

### 2.2 构建错误分类与处理

| 错误类型 | 严重程度 | 处理策略 |
|---------|---------|---------|
| 编译错误 | P0-致命 | **必须修复**，否则无法构建 |
| TypeScript 类型错误 | P1-严重 | **必须修复**，确保类型安全 |
| ESLint 错误 | P1-严重 | **必须修复**，保持代码规范 |
| ESLint 警告 | P2-中等 | 优先修复，部分可忽略 |
| 控制台警告 | P3-低 | 记录并尝试修复 |

### 2.3 构建产物检查规则

```typescript
interface BuildOutputCheck {
  // 检查构建产物是否存在
  checkFilesExist({ distDir: string }): {
    htmlExists: boolean,
    jsExists: boolean,
    cssExists: boolean,
    assetsDirExists: boolean,
    missingFiles: string[]
  }

  // 检查构建产物大小是否合理
  checkBundleSize({ distDir: string }): {
    totalSize: string,
    mainJsSize: string,
    mainCssSize: string,
    isAcceptable: boolean  // max: 3MB total
  }

  // 检查 sourcemap 是否正确
  checkSourceMap({ distDir: string }): {
    hasSourceMap: boolean,
    missingSourceFiles: string[]
  }
}
```

---

## 三、运行时验证规则

### 3.1 页面加载验证清单

```json
{
  "basic_checks": {
    "dev_server_startup": { "expected": "成功启动，无错误" },
    "home_page_load": { "expected": "首页正常渲染，无 404" },
    "route_navigation": { "expected": "路由切换正常，无错误" },
    "lazy_loading": { "expected": "懒加载路由正常" },
    "404_page": { "expected": "未匹配路由显示 404" }
  },
  "state_checks": {
    "store_initialization": { "expected": "Pinia store 正常初始化" },
    "state_reactivity": { "expected": "状态变更触发表单更新" },
    "async_actions": { "expected": "异步 action 正常执行" },
    "cross_store_calls": { "expected": "跨 store 调用正常" }
  },
  "feature_checks": {
    "login_flow": { "expected": "登录成功，跳转到首页" },
    "permission_control": { "expected": "权限控制正常" },
    "data_crud": { "expected": "数据增删改查正常" },
    "file_upload": { "expected": "文件上传正常" }
  }
}
```

---

## 四、代码质量扫描规则

### 4.1 Vue2 遗留语法检测

```javascript
// 正则扫描规则
const VUE2_DEPRECATED_PATTERNS = [
  // Vue 实例方法
  { pattern: /(?:^|\s)Vue\.(set|delete|observable|nextTick|directive|filter|component)\(/, severity: "error", message: "Vue3 API 变化，需使用 app.xxx() 替代" },
  { pattern: /new Vue\(/, severity: "error", message: "Vue3 使用 createApp() 替代" },
  
  // Vue 实例属性
  { pattern: /\$children/, severity: "error", message: "Vue3 移除 $children，使用 ref + defineExpose" },
  { pattern: /\$listeners/, severity: "error", message: "Vue3 合并到 $attrs" },
  { pattern: /\$scopedSlots/, severity: "error", message: "Vue3 统一为 $slots" },
  { pattern: /\$on\b/, severity: "error", message: "Vue3 移除 $on，使用 mitt 或 provide/inject" },
  { pattern: /\$once\b/, severity: "error", message: "Vue3 移除 $once，使用 mitt" },
  { pattern: /\$off\b/, severity: "error", message: "Vue3 移除 $off，使用 mitt" },
  { pattern: /\$destroy\b/, severity: "error", message: "Vue3 移除 $destroy，Vue 自动卸载" },
  
  // Vuex 遗留引用
  { pattern: /this\.\$store/, severity: "error", message: "Vuex 已迁移到 Pinia，使用 useXxxStore()" },
  { pattern: /Vue\.use\(Vuex\)/, severity: "error", message: "使用 createPinia() 替代" },
  
  // Vue Router 2.x
  { pattern: /beforeRouteEnter\(/, severity: "warning", message: "Vue Router 4 建议使用组合式 API" },
  { pattern: /beforeRouteUpdate\(/, severity: "warning", message: "Vue Router 4 使用 onBeforeRouteUpdate" },
  { pattern: /beforeRouteLeave\(/, severity: "warning", message: "Vue Router 4 使用 onBeforeRouteLeave" },
  
  // Vue2 filter
  { pattern: /\{\{[^}]*\|[^}]+\}\}/, severity: "error", message: "Vue3 移除 filter，转为 computed 或方法" },
  
  // Vue2 .native
  { pattern: /\.native\b/, severity: "error", message: "Vue3 移除 .native，使用 emits 声明" },
  
  // Vue2 v-model
  { pattern: /\.sync\b/, severity: "warning", message: "Vue3 使用 v-model:propName" },
]
```

### 4.2 ESLint + TypeScript 验证

```bash
# TypeScript 严格模式检查
npx vue-tsc --noEmit --strict

# ESLint 检查（使用 Vue3 规则）
npx eslint src/ --ext .vue,.ts,.tsx --rule '{
  "vue/no-deprecated-v-bind-style": "warn",
  "vue/no-deprecated-v-on-native-modifier": "error",
  "vue/no-deprecated-slot-attribute": "error",
  "vue/no-deprecated-slot-scope-attribute": "error",
  "vue/no-deprecated-filter": "error",
  "vue/no-deprecated-destroyed-lifecycle": "error",
  "vue/no-deprecated-dollar-listeners-api": "error",
  "vue/no-deprecated-dollar-scopedslots-api": "error"
}'
```

### 4.3 构建后代码扫描

```bash
# 搜索残留的 Vue2 特有代码
grep -r "Vue\.set\|Vue\.delete\|Vue\.observable\|Vue\.filter" src/ --include="*.ts" --include="*.vue"

# 搜索残留的 vuex 引用
grep -r "vuex" src/ --include="*.ts" --include="*.vue"

# 搜索未迁移的路由守卫（使用了 next 参数）
grep -r "next(" src/router/ --include="*.ts"

# 搜索未按 pinia 规范的操作
grep -r "\$store" src/ --include="*.ts" --include="*.vue"
```

---

## 五、错误修复规则

### 5.1 通用错误修复策略

```typescript
const ERROR_FIX_STRATEGIES: Record<string, ErrorFixStrategy> = {
  // 1. 组件未注册错误
  "Failed to resolve component": {
    severity: "P0",
    cause: "组件未在 Vue3 中正确注册或导入路径错误",
    steps: [
      "检查组件 import 路径是否正确（注意大小写）",
      "检查是否使用 app.component() 全局注册",
      "检查局部注册 components 选项是否正确"
    ]
  },

  // 2. 响应式错误
  "TypeError: Cannot read properties of undefined": {
    severity: "P1",
    cause: "响应式数据初始化不完整",
    steps: [
      "确保所有响应式数据有初始值",
      "使用 ref(null as Type | null) 确保类型",
      "使用可选链 ?. 操作符"
    ]
  },

  // 3. this 上下文错误
  "Cannot access 'this' before initialization": {
    severity: "P1",
    cause: "setup() 中使用 this，Vue3 setup 中 this 指向不同",
    steps: [
      "在 setup 中使用 getCurrentInstance() 获取实例",
      "或在 setup 外部使用 Options API"
    ]
  },

  // 4. v-model 错误
  "v-model is not supported on this element": {
    severity: "P1",
    cause: "Vue3 v-model 行为变化",
    steps: [
      "自定义组件 model 选项改为 modelValue + update:modelValue",
      "使用 defineModel() 宏定义",
      "多个 v-model 使用 v-model:xxx 语法"
    ]
  },

  // 5. slot 错误
  "slot attribute is deprecated": {
    severity: "P2",
    cause: "Vue3 废弃了 slot 属性",
    steps: [
      "slot=\"xxx\" → v-slot:xxx",
      "slot-scope → v-slot=\"props\"",
      "或使用 #xxx 缩写"
    ]
  },

  // 6. 异步组件错误
  "Async component is not a function": {
    severity: "P1",
    cause: "Vue3 异步组件需要使用 defineAsyncComponent",
    steps: [
      "import defineAsyncComponent from 'vue'",
      "defineAsyncComponent(() => import('xxx.vue'))",
      "或直接 () => import() 但仅支持 Composition API"
    ]
  },

  // 7. 过滤器错误
  "filter is not a function": {
    severity: "P1",
    cause: "Vue3 移除了 filter",
    steps: [
      "将 filter 转为 computed 属性",
      "或转为方法调用",
      "或使用全局函数挂在 globalProperties"
    ]
  },

  // 8. 事件总线错误
  "$on / $emit / $off is not a function": {
    severity: "P1",
    cause: "Vue3 移除了实例事件 API",
    steps: [
      "安装 mitt 库: npm install mitt",
      "创建 eventBus: export const emitter = mitt()",
      "替换 this.$on → emitter.on, this.$emit → emitter.emit"
    ]
  },
}
```

### 5.2 错误处理流程

```
[发现错误] → [查询 commonIssue.md] → [存在已知方案？]
  ├─ 是 → [按文档规则修复] → [验证修复]
  └─ 否 → [查询 ERROR_FIX_STRATEGIES]
           ├─ 匹配 → [按策略修复] → [验证修复]
           └─ 不匹配 → [分析错误根因] → [制定修复策略] → [修复并记录]
                                                  ↓
                                             [将新方案补充到知识库]
```

### 5.3 错误优先级

| 优先级 | 错误类型 | 处理时限 | 处理方式 |
|--------|---------|---------|---------|
| **P0** | 构建失败、白屏、路由不通 | 立即 | 停止所有工作，优先修复 |
| **P1** | 功能异常、API 调用失败 | 高优 | 尽快修复，影响用户体验 |
| **P2** | 样式错乱、UI 不一致 | 正常 | 按计划修复 |
| **P3** | 控制台警告、性能问题 | 低优 | 记录并批量处理 |

---

## 六、测试执行规则

### 6.1 测试类型明细

| 测试类型 | 工具 | 执行方式 | 验证目标 |
|---------|------|---------|---------|
| 单元测试 | Vitest | 自动化 | 组件、工具函数、store |
| 组件测试 | @vue/test-utils | 自动化 | 组件渲染、交互 |
| E2E 测试 | Playwright / Cypress | 自动化 + 手动 | 完整业务流程 |
| 视觉回归测试 | 截图对比 | 手动 | UI 一致性 |

### 6.2 核心测试用例模板

```typescript
// 1. 组件挂载测试
import { mount } from '@vue/test-utils'
import { createRouter, createWebHistory } from 'vue-router'
import { createPinia, setActivePinia } from 'pinia'
import { describe, it, expect, beforeEach } from 'vitest'

describe('Component Mount Test', () => {
  let wrapper: any
  
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('组件正常挂载', async () => {
    const router = createRouter({
      history: createWebHistory(),
      routes: [{ path: '/', component: () => import('@/views/Home.vue') }]
    })
    
    wrapper = mount(Component, {
      global: {
        plugins: [router, pinia]
      }
    })
    
    await router.isReady()
    expect(wrapper.exists()).toBe(true)
    expect(wrapper.html()).toContain('expected-text')
  })
})

// 2. Pinia Store 测试
import { setActivePinia, createPinia } from 'pinia'
import { useUserStore } from '@/stores/user'

describe('Pinia Store Test', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('store 初始化状态正确', () => {
    const store = useUserStore()
    expect(store.token).toBeNull()
    expect(store.isLoggedIn).toBe(false)
  })

  it('action 更新状态', () => {
    const store = useUserStore()
    store.setToken('test-token')
    expect(store.token).toBe('test-token')
    expect(store.isLoggedIn).toBe(true)
  })
})
```

---

## 七、性能评估指标

### 7.1 Bundle 大小对比

| 指标 | 说明 | 可接受范围 |
|------|------|-----------|
| 总 JS 体积 | 所有 JS chunk 之和 | < 3MB（优化目标 < 1.5MB）|
| 总 CSS 体积 | 所有 CSS chunk 之和 | < 500KB |
| 首屏 JS 体积 | 入口 chunk 大小 | < 200KB |
| 首屏 CSS 体积 | 首屏 CSS 大小 | < 100KB |

### 7.2 构建性能对比

| 指标 | Vue2 (Webpack) | Vue3 (Vite) | 期望变化 |
|------|---------------|-------------|---------|
| 首次构建 | ~180s | ~60s | -66% |
| 增量构建 | ~30s | < 1s | -97% |
| HMR 更新 | ~5s | < 100ms | -98% |

### 7.3 性能检查清单

- [ ] bundle 体积是否比 Vue2 版本更小
- [ ] 是否有未使用的依赖需要移除
- [ ] 是否有大文件需要懒加载
- [ ] 路由是否正确配置了懒加载
- [ ] 组件是否合理使用了异步加载（defineAsyncComponent）
- [ ] 样式是否存在重复加载或未加载的情况
- [ ] 图片等静态资源是否正确加载