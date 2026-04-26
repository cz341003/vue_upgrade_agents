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