# Vue 语法转化 Skill - Vue Syntax Conversion

## 功能描述
将 Vue2 组件的 JavaScript/TypeScript 代码转化为 Vue3 兼容语法。支持 Options API 保持、Composition API 升级、TypeScript 类型标注等转化。

## 触发条件
当需要迁移 Vue2 组件语法时自动调用。包括：组件定义转化、生命周期钩子映射、API 调用替换、过滤器处理、事件总线迁移等。

---

## 1. 组件定义转化

### 1.1 基础组件定义

| Vue2 | Vue3 Options API | Vue3 Composition API (推荐) |
|------|-----------------|---------------------------|
| `export default { ... }` | `export default defineComponent({ ... })` | `<script setup lang="ts">` |
| `data() { return { x: 1 } }` | `data() { return { x: 1 } }` | `const x = ref(1)` |
| `computed: { double() { ... } }` | `computed: { double() { ... } }` | `const double = computed(() => ...)` |
| `watch: { x(newVal) { ... } }` | `watch: { x(newVal) { ... } }` | `watch(x, (newVal) => { ... })` |
| `methods: { foo() { ... } }` | `methods: { foo() { ... } }` | `function foo() { ... }` |
| `props: ['name']` | `props: { name: String }` | `defineProps({ name: String })` |
| `created() { ... }` | `created() { ... }` | `onMounted(() => { ... })` (近似) |
| `mounted() { ... }` | `mounted() { ... }` | `onMounted(() => { ... })` |
| `beforeDestroy() { ... }` | `beforeUnmount() { ... }` | `onBeforeUnmount(() => { ... })` |
| `destroyed() { ... }` | `unmounted() { ... }` | `onUnmounted(() => { ... })` |

### 1.2 生命周期钩子映射

| Vue2 | Vue3 | 说明 |
|------|------|------|
| `beforeCreate` | `setup()` / `<script setup>` | setup 中直接执行 |
| `created` | `setup()` / `<script setup>` | setup 中直接执行 |
| `beforeMount` | `onBeforeMount` | - |
| `mounted` | `onMounted` | - |
| `beforeUpdate` | `onBeforeUpdate` | - |
| `updated` | `onUpdated` | - |
| `beforeDestroy` | `onBeforeUnmount` | **名称变化** |
| `destroyed` | `onUnmounted` | **名称变化** |
| `errorCaptured` | `onErrorCaptured` | - |
| `activated` | `onActivated` | keep-alive |
| `deactivated` | `onDeactivated` | keep-alive |
| - | `onRenderTracked` | 新增（调试用） |
| - | `onRenderTriggered` | 新增（调试用） |
| `serverPrefetch` | `onServerPrefetch` | SSR |

### 1.3 异步组件变化

```javascript
// Vue2
const AsyncComponent = () => import('./MyComponent.vue')

// Vue3
import { defineAsyncComponent } from 'vue'
const AsyncComponent = defineAsyncComponent(() => import('./MyComponent.vue'))

// 带选项
const AsyncComponent = defineAsyncComponent({
  loader: () => import('./MyComponent.vue'),
  loadingComponent: LoadingComponent,
  errorComponent: ErrorComponent,
  delay: 200,
  timeout: 3000
})
```

### 1.4 函数式组件变化

```javascript
// Vue2 函数式组件
export default {
  functional: true,
  props: ['message'],
  render(h, { props }) {
    return h('div', props.message)
  }
}
```

```vue
<!-- Vue3 函数式组件 -->
<script setup lang="ts">
defineProps<{ message: string }>()
</script>
<template>
  <div>{{ message }}</div>
</template>
```

或者纯函数形式：
```typescript
import type { FunctionalComponent } from 'vue'
interface Props { message: string }
const MyComponent: FunctionalComponent<Props> = (props) => {
  return <div>{props.message}</div>
}
MyComponent.props = ['message']
```

### 1.5 异步组件使用

使用 `defineAsyncComponent` 包裹动态导入。

### 1.6 Vue.extend 组件定义转化

```javascript
// Vue2 - Vue.extend 定义组件（常见于 .js / .mjs 文件）
import Vue from 'vue'
const MyButton = Vue.extend({
  name: 'MyButton',
  props: { type: String, disabled: Boolean },
  data() {
    return { count: 0 }
  },
  methods: {
    handleClick() { this.$emit('click') }
  },
  template: '<button :class="type" @click="handleClick"><slot/></button>'
})

// 用法：作为组件注册或通过 Vue.use 安装
export default MyButton
```

```vue
<!-- Vue3 - 转化为 defineComponent / <script setup> -->

<!-- 方式一：Options API（保持非 .vue 文件的注册用途） -->
<script lang="ts">
import { defineComponent, PropType } from 'vue'
interface Props { type?: string; disabled?: boolean }
export default defineComponent({
  name: 'MyButton',
  props: { type: String, disabled: Boolean },
  data() { return { count: 0 } },
  methods: { handleClick() { this.$emit('click') } }
})
</script>

<!-- 方式二：Composition API（推荐，.vue 文件） -->
<script setup lang="ts">
defineOptions({ name: 'MyButton' })
const props = defineProps<{ type?: string; disabled?: boolean }>()
const count = ref(0)
const emit = defineEmits<{ (e: 'click'): void }>()
function handleClick() { emit('click') }
</script>
```

```javascript
// Vue3 - 纯 JS 文件中的组件定义（用于注册到 app）
import { defineComponent, h } from 'vue'
const MyButton = defineComponent({
  name: 'MyButton',
  props: { type: String, disabled: Boolean },
  setup(props, { emit, slots }) {
    const count = ref(0)
    function handleClick() { emit('click') }
    return () => h('button', { class: props.type, onClick: handleClick }, slots.default?.())
  }
})
export default MyButton
```

**关键变化说明：**
- `Vue.extend({...})` → `defineComponent({...})`（3.x 中等价替换，类型推导更完善）
- 如果原组件使用了 `template` 属性，需要改为 `render` 函数或转为 `.vue` 文件
- 如果原组件是通过 `Vue.use(ComponentExtend)` 安装的，见 §2.1 中的 `Vue.extend + Vue.use 组合` 处理

### 1.7 非 .vue 组件文件（.js / .mjs）处理

迁移工程中除了 `.vue` 文件，还存在大量 `.js` / `.mjs` 文件中声明组件、导出供其他组件注册使用的场景。以下是典型模式及迁移方式：

#### 1.7.1 Render 函数导出组件

```javascript
// Vue2 - .mjs / .js 文件中通过 render 导出组件
import { h } from 'vue'
export default {
  name: 'MyRenderComp',
  props: { message: String },
  render(h) { return h('div', { class: 'msg' }, this.message) }
}
```

```vue
<!-- Vue3 - 转化为 .vue 文件 -->
<script setup lang="ts">
defineProps<{ message: string }>()
</script>
<template>
  <div class="msg">{{ message }}</div>
</template>
```

或保持 JS 形式：
```javascript
// Vue3 - 保持 JS 形式（用于动态注册场景）
import { defineComponent, h } from 'vue'
const MyRenderComp = defineComponent({
  name: 'MyRenderComp',
  props: { message: String },
  setup(props, { slots }) {
    return () => h('div', { class: 'msg' }, props.message)
  }
})
export default MyRenderComp
```

#### 1.7.2 工厂函数/高阶组件导出

```javascript
// Vue2 - 工厂函数导出组件
export function createTable(options) {
  return Vue.extend({ 
    props: { data: Array },
    template: '<table>...</table>',
    methods: { /* 使用 options */ }
  })
}
```

```javascript
// Vue3 - 工厂函数导出
import { defineComponent, h } from 'vue'
export function createTable(options) {
  return defineComponent({
    props: { data: Array },
    setup(props) {
      // 使用 options
      return () => h('table', /* ... */)
    }
  })
}
```

#### 1.7.3 纯 JS 配置对象组件（无 Vue.extend 包装）

```javascript
// Vue2 - 直接导出配置对象
export default {
  name: 'PlainComponent',
  props: ['value'],
  template: '<div>{{ value }}</div>'
}
```

```javascript
// Vue3 - 用 defineComponent 包装（获得类型推导）
import { defineComponent } from 'vue'
export default defineComponent({
  name: 'PlainComponent',
  props: ['value'],
  template: '<div>{{ value }}</div>'
})
```

#### 1.7.4 VNode 节点导出（非组件）

```javascript
// Vue2 - 导出 VNode 供其他组件渲染
export function renderHeader(h) {
  return h('div', { class: 'header' }, 'Title')
}
// 其他组件中：<component :is="renderHeader" />
```

```vue
<!-- Vue3 - 导出为函数式组件 -->
<script setup lang="ts">
const renderHeader = () => h('div', { class: 'header' }, 'Title')
</script>
<template>
  <renderHeader />
</template>
```

---

## 2. 全局 API 迁移

### 2.1 入口文件转化

| Vue2 | Vue3 |
|------|------|
| `Vue.prototype.$http = axios` | `app.config.globalProperties.$http = axios` |
| `new Vue({...}).$mount('#app')` | `createApp(App).use(router).mount('#app')` |
| `Vue.component('name', comp)` | `app.component('name', comp)` |
| `Vue.directive('name', dir)` | `app.directive('name', dir)` |
| `Vue.mixin(mixin)` | `app.mixin(mixin)` |
| `Vue.use(plugin)` | `app.use(plugin)` |
| `Vue.filter('name', fn)` | `app.config.globalProperties.$filters = { name: fn }` |

**Vue.extend + Vue.use 组合特殊处理：**

Vue2 中常出现 `Vue.extend` 定义组件后通过 `Vue.use` 安装的模式：

```javascript
// Vue2 - Vue.extend + Vue.use 组合
// utils/my-plugin.js
import Vue from 'vue'
const MyComponent = Vue.extend({
  name: 'MyComponent',
  template: '<div>{{ message }}</div>',
  props: { message: String }
})
MyComponent.install = function(Vue) {
  Vue.component('MyComponent', MyComponent)
}
export default MyComponent

// main.js
import MyComponent from './utils/my-plugin'
Vue.use(MyComponent)
```

```javascript
// Vue3 - 拆分为组件定义 + app.component 注册
// utils/my-component.js
import { defineComponent } from 'vue'
const MyComponent = defineComponent({
  name: 'MyComponent',
  props: { message: String },
  template: '<div>{{ message }}</div>'
})
export default MyComponent

// main.js
import { createApp } from 'vue'
import MyComponent from './utils/my-component'
const app = createApp(App)
app.component('MyComponent', MyComponent)  // 替代 Vue.use(MyComponent)
app.mount('#app')
```

**处理规则：**
1. 识别 `.js`/`.mjs` 文件中 `Vue.extend({...})` + `Component.install` 或 `Vue.use(ComponentExtend)` 的模式
2. 移除 `install` 方法（Vue3 中不需要）
3. `Vue.use(ComponentExtend)` → `app.component('Name', Component)`（如果目标是注册为全局组件）
4. 如果 `Vue.extend` 的对象本身只是一个普通组件（无 `install` 方法），直接 `Vue.extend` → `defineComponent`

### 2.2 全局属性类型声明

```typescript
// src/types/global.d.ts
declare module 'vue' {
  interface ComponentCustomProperties {
    $http: typeof axios
    $message: any
    $filters: { currency: (value: number) => string }
  }
}
export {}
```

### 2.3 全局API映射表结构

| Vue2 表达式 | Vue3 替换方式 |
|------------|--------------|
| `this.$http` | `app.config.globalProperties.$http` 或 import 直接使用 |
| `this.$store` | Pinia store (由状态管理专家处理) |
| `this.$router` | `useRouter()` |
| `this.$route` | `useRoute()` |
| `this.$t` | `useI18n().t` (vue-i18n v9) |
| `this.$emit` | `defineEmits()` / `emits` |

---

## 3. v-model 迁移

### 3.1 组件 v-model 迁移

```vue
<!-- Vue2 组件 v-model -->
<script>
export default {
  model: { prop: 'value', event: 'change' },
  props: { value: String },
  methods: { updateValue(newVal) { this.$emit('change', newVal) } }
}
</script>
```

```vue
<!-- Vue3 Options API -->
<script>
export default {
  props: { modelValue: String },  // 默认 prop 改为 modelValue
  emits: ['update:modelValue'],   // 需要声明 emits
  methods: {
    updateValue(newVal) { this.$emit('update:modelValue', newVal) }
  }
}
</script>
```

```vue
<!-- Vue3 Composition API -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ (e: 'update:modelValue', value: string): void }>()
const updateValue = (newVal: string) => { emit('update:modelValue', newVal) }
</script>
```

```vue
<!-- Vue3.4+ defineModel -->
<script setup lang="ts">
const model = defineModel<string>()
// 自动生成 modelValue prop 和 update:modelValue emit
</script>
<template><input v-model="model" /></template>
```

### 3.2 .sync 修饰符迁移

```vue
<!-- Vue2 -->
<ChildComponent :visible.sync="isVisible" />

<!-- Vue3 -->
<ChildComponent v-model:visible="isVisible" />
```

---

## 4. 事件 & 监听器变化

### 4.1 $listeners 合并

Vue3 中 `$listeners` 移除，事件监听器合入 `$attrs`：

```vue
<!-- Vue2 -->
<child v-bind="$attrs" v-on="$listeners" />

<!-- Vue3 -->
<child v-bind="$attrs" />  <!-- $listeners 自动合并到 $attrs -->
```

### 4.2 .native 修饰符移除

```vue
<!-- Vue2 -->
<ChildComponent @click.native="handleClick" />

<!-- Vue3（移除 .native，所有事件默认为原生事件） -->
<ChildComponent @click="handleClick" />
```

### 4.3 事件总线迁移

```javascript
// Vue2 - event-bus.js
export const eventBus = new Vue()
eventBus.$on('event-name', handler)
eventBus.$emit('event-name', data)
```

```typescript
// Vue3 - 使用 mitt 库
// utils/event-bus.ts
import mitt from 'mitt'
type Events = { 'event-name': string }
export const emitter = mitt<Events>()

// 组件中
onMounted(() => { emitter.on('event-name', handler) })
onUnmounted(() => { emitter.off('event-name', handler) })
emitter.emit('event-name', 'data')
```

---

## 5. 过滤器 (filter) 迁移

Vue3 **移除了过滤器**，按以下方式替换：

```vue
<!-- Vue2 -->
<template><div>{{ price | currency('¥') }}</div></template>
<script>
export default {
  filters: { currency(value, symbol = '¥') { return symbol + value.toFixed(2) } }
}
</script>
```

```vue
<!-- Vue3 Composition API -->
<script setup>
const currency = (value, symbol = '¥') => symbol + value.toFixed(2)
</script>
<template><div>{{ currency(price) }}</div></template>
```

或全局注册：`app.config.globalProperties.$filters = { currency: (v) => ... }`

---

## 6. 混入 (Mixin) → 组合式函数 (Composable)

```javascript
// Vue2 Mixin
export default {
  data() { return { windowWidth: 0 } },
  mounted() { window.addEventListener('resize', this.onResize) },
  beforeDestroy() { window.removeEventListener('resize', this.onResize) },
  methods: { onResize() { this.windowWidth = window.innerWidth } }
}
```

```typescript
// Vue3 Composable - composables/useResize.ts
import { ref, onMounted, onUnmounted } from 'vue'
export function useResize() {
  const windowWidth = ref(0)
  function onResize() { windowWidth.value = window.innerWidth }
  onMounted(() => window.addEventListener('resize', onResize))
  onUnmounted(() => window.removeEventListener('resize', onResize))
  return { windowWidth }
}
```

---

## 7. TypeScript 类型标注

### 7.1 Options API 类型标注

```typescript
import { defineComponent, PropType } from 'vue'
interface User { id: number; name: string; email: string }

export default defineComponent({
  props: {
    user: { type: Object as PropType<User>, required: true },
    title: { type: String, default: '用户信息' }
  },
  emits: { 'update': (user: User) => true, 'delete': (id: number) => true },
  data() { return { loading: false as boolean, error: null as string | null } },
  methods: { async fetchData(): Promise<void> { /* ... */ } }
})
```

### 7.2 Composition API 类型标注

```vue
<script setup lang="ts">
interface User { id: number; name: string; email: string }
const props = defineProps<{ user: User; title?: string }>()
const emit = defineEmits<{ (e: 'update', user: User): void; (e: 'delete', id: number): void }>()
const loading = ref(false)
const error = ref<string | null>(null)
async function fetchData(): Promise<void> { /* ... */ }
</script>
```

---

### 7.3 纯 JS 工具文件 → TS 转化（非组件文件）

对于 `src/utils/*.js`、`src/api/*.js` 等不包含 Vue 组件定义、仅导出普通函数/常量的纯 JS 工具文件，需要将其转化为 TypeScript 文件（`.js` → `.ts`）并添加类型标注。

#### 7.3.1 工具函数文件（src/utils/*.js）

```javascript
// Vue2 - JS 工具函数
import moment from 'moment'
export function formatDate(date, format = 'YYYY-MM-DD') {
  return moment(date).format(format)
}
export function debounce(fn, delay) {
  let timer = null
  return function(...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}
export const API_BASE_URL = '/api/v1'
```

```typescript
// Vue3 - TS 工具函数
import moment from 'moment'
export function formatDate(date: string | Date, format: string = 'YYYY-MM-DD'): string {
  return moment(date).format(format)
}
export function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout> | null = null
  return function(this: any, ...args: Parameters<T>): void {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}
export const API_BASE_URL: string = '/api/v1'
```

#### 7.3.2 API 接口文件（src/api/*.js）

```javascript
// Vue2 - JS API 文件
import request from '@/utils/request'
export function getUsers(params) {
  return request({ url: '/users', method: 'get', params })
}
export function createUser(data) {
  return request({ url: '/users', method: 'post', data })
}
export function deleteUser(id) {
  return request({ url: `/users/${id}`, method: 'delete' })
}
```

```typescript
// Vue3 - TS API 文件
import request from '@/utils/request'

// 定义请求/响应类型
export interface UserQuery {
  page?: number
  pageSize?: number
  keyword?: string
  status?: string | number
}
export interface UserInfo {
  id: number
  name: string
  email: string
  role: string
  createdAt: string
}
export interface PaginatedResponse<T> {
  code: number
  data: { list: T[]; total: number }
  message: string
}

export function getUsers(params: UserQuery): Promise<PaginatedResponse<UserInfo>> {
  return request({ url: '/users', method: 'get', params })
}
export function createUser(data: Partial<UserInfo>): Promise<{ code: number; data: UserInfo; message: string }> {
  return request({ url: '/users', method: 'post', data })
}
export function deleteUser(id: number): Promise<{ code: number; message: string }> {
  return request({ url: `/users/${id}`, method: 'delete' })
}
```

#### 7.3.3 转化规则

| 原始 JS 模式 | 目标 TS 模式 | 说明 |
|-------------|-------------|------|
| `function fn(a, b)` | `function fn(a: type, b: type): ReturnType` | 为所有参数和返回值添加类型 |
| `const CONST = value` | `const CONST: type = value` | 为常量添加类型标注 |
| 未使用泛型 | 使用泛型约束 | 如 `debounce<T>(fn, delay)` |
| `any` 类型参数 | 尽可能推断精确类型 | 如 `params: UserQuery` 而非 `params: any` |
| 无接口定义 | 定义 Request/Response 接口 | 如 `UserQuery`、`UserInfo`、`PaginatedResponse<T>` |
| 无类型导出 | 导出接口供组件使用 | 其他文件可 `import type { UserInfo } from '@/api/user'` |
| `config.js` | `config.ts` | 配置文件也转为 TS，使用 `const` + `as const` |
| `router.js` | `router.ts` | 路由文件转为 TS，使用 `RouteRecordRaw` 类型 |

#### 7.3.4 文件扩展名变更

所有纯 JS 工具文件在迁移后需按以下规则变更文件扩展名：

| 原始路径 | 迁移后路径 | 说明 |
|---------|-----------|------|
| `src/utils/format.js` | `src/utils/format.ts` | 工具函数 |
| `src/utils/request.js` | `src/utils/request.ts` | 网络请求封装 |
| `src/utils/validate.js` | `src/utils/validate.ts` | 校验函数 |
| `src/utils/index.js` | `src/utils/index.ts` | 工具函数入口 |
| `src/api/user.js` | `src/api/user.ts` | API 接口 |
| `src/api/role.js` | `src/api/role.ts` | API 接口 |
| `src/config/index.js` | `src/config/index.ts` | 配置文件 |
| `src/router/index.js` | `src/router/index.ts` | 路由配置 |
| `src/store/*.js` | `src/store/*.ts` | 状态管理文件 |
| `src/plugins/*.js` | `src/plugins/*.ts` | 插件文件 |

#### 7.3.5 注意事项

1. **`import` 路径更新**：其他文件 `import` 这些工具文件时，需同步更新路径中的 `.js` → `.ts`（或直接去掉扩展名）
2. **`.mjs` 文件处理**：`.mjs` 文件同样转为 `.ts`，注意 ESM 模块语法保持（`import`/`export` 保留）
3. **默认导出**：JS 文件中的 `module.exports = { ... }` 转为 `export default { ... }` 或具名导出
4. **`require` 替换**：`const xxx = require('xxx')` 转为 `import xxx from 'xxx'`
5. **全局变量**：如果使用 `window.xxx` 全局变量，需在 `.d.ts` 文件中声明类型
6. **工具函数保留纯函数性**：不改变函数逻辑，仅添加类型信息
7. **`ts-ignore` 使用**：对于无法确定类型的第三方库调用，可使用 `// @ts-ignore` 暂时绕过，但需同时添加 `// TODO: 补充类型定义` 注释

## 8. 自定义指令钩子转化

| Vue2 | Vue3 | 说明 |
|------|------|------|
| `bind` | `beforeMount` | 绑定到元素，挂载前 |
| `inserted` | `mounted` | 插入父节点时 |
| `update` | `updated` | 组件更新时 |
| `componentUpdated` | `beforeUpdate` | 组件更新前 |
| `unbind` | `unmounted` | 解绑时 |

```javascript
// Vue2 指令
Vue.directive('highlight', {
  bind(el, binding) { /* ... */ },
  inserted(el) { el.focus() },
  unbind(el) { /* ... */ }
})

// Vue3 指令
app.directive('highlight', {
  beforeMount(el, binding) { /* ... */ },  // 原 bind
  mounted(el) { el.focus() },              // 原 inserted
  unmounted(el) { /* ... */ }              // 原 unbind
})
```

---

## 9. Vuex 调用标记

在 `this.$store` 调用处添加标记，供状态管理专家处理：

```vue
<script setup lang="ts">
// TODO: 状态管理专家 - 以下代码需要在 Pinia 迁移后调整
const store = useStore() // 需要状态管理专家实现
const userInfo = computed(() => store.state.user.info)
function login() { store.dispatch('user/login', payload) }
</script>
```

## 10. 注意事项

- 转化过程中，如果未匹配到上述转化规则，查阅vue3官方文档查找解决方案，不要自己猜；