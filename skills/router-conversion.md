# 路由转化 Skill (Router Conversion)

## 概述

本 Skill 封装了 Vue Router 3.x → Vue Router 4.x 的所有转化规则，供路由专家在迁移任务中调用。

---

## 一、路由配置迁移

### 1.1 核心迁移对照

| 项目 | Vue2 (Vue Router 3) | Vue3 (Vue Router 4) | 说明 |
|------|---------------------|---------------------|------|
| 构造函数 | `new Router({...})` | `createRouter({...})` | 函数式创建 |
| 历史模式 | `mode: 'history'` | `history: createWebHistory()` | 独立函数 |
| 哈希模式 | `mode: 'hash'` | `history: createWebHashHistory()` | 独立函数 |
| 抽象模式 | `mode: 'abstract'` | `history: createMemoryHistory()` | SSR 使用 |
| base 选项 | `base: '/app/'` | `createWebHistory('/app/')` | 作为历史模式参数 |
| 通配符路由 | `path: '*'` | `path: '/:pathMatch(.*)*'` | 自定义正则参数 |
| 路由注册 | `Vue.use(Router)` | 无需调用，直接 `app.use(router)` | Vue3 插件机制 |

### 1.2 核心配置迁移示例

```typescript
// Vue3 router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'

const routes: Array<RouteRecordRaw> = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/views/Home.vue'),
    meta: { requiresAuth: false, title: '首页' }
  },
  {
    path: '/:pathMatch(.*)*',  // Vue3 通配符
    name: 'NotFound',
    redirect: '/404'
  }
]

const router = createRouter({
  history: createWebHistory('/app/'),  // base 移入这里
  linkActiveClass: 'active',
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition
    return { top: 0 }
  },
  routes
})

export default router
```

### 1.3 Vue.use 移除

```typescript
// main.ts
import { createApp } from 'vue'
import router from './router'

const app = createApp(App)
app.use(router)  // 替代 Vue.use(Router) + new Vue({ router })
app.mount('#app')
```

---

## 二、导航守卫迁移

### 2.1 关键变化

| 行为 | Vue2 | Vue3 |
|------|------|------|
| 第三个参数 | `next()` 必须调用 | `next()` 已废弃，使用返回值 |
| 放行 | `next()` | 不返回或 `return undefined` |
| 重定向 | `next('/login')` | `return '/login'` |
| 取消导航 | `next(false)` | `return false` |
| 错误 | `next(error)` | `throw error` |
| 异步守卫 | 需要 `next()` | 直接返回 Promise |

### 2.2 全局守卫迁移

```typescript
// Vue3 全局守卫
router.beforeEach(async (to, from): Promise<NavigationGuardReturn | void> => {
  if (to.meta.requiresAuth) {
    const isAuthenticated = await checkAuth()
    if (!isAuthenticated) {
      return { name: 'Login', query: { redirect: to.fullPath } }
    }
  }
  document.title = (to.meta.title as string) || '默认标题'
  // 不返回任何值表示放行
})
```

### 2.3 组件内守卫迁移

```vue
<!-- Vue3 Composition API -->
<script setup lang="ts">
import { onBeforeRouteEnter, onBeforeRouteUpdate, onBeforeRouteLeave } from 'vue-router'

// 路由参数变化时重新加载
onBeforeRouteUpdate(async (to, from) => {
  data.value = await loadData(to.params.id as string)
})

// 离开确认
onBeforeRouteLeave((to, from) => {
  return window.confirm('确定要离开吗？')  // false 取消导航
})

// onBeforeRouteEnter：通过 callback 访问组件实例
onBeforeRouteEnter((to, from) => {
  return (vm) => {
    console.log('组件实例已创建', vm)
  }
})
</script>
```

### 2.4 组件内守卫对照

| Vue2 (Vue Router 3) | Vue3 (Vue Router 4) |
|---------------------|---------------------|
| `beforeRouteEnter` | `onBeforeRouteEnter` |
| `beforeRouteUpdate` | `onBeforeRouteUpdate` |
| `beforeRouteLeave` | `onBeforeRouteLeave` |

---

## 三、动态路由迁移

### 3.1 动态添加

```typescript
// Vue3：addRoutes 已废弃，使用多次 addRoute
// 先添加父路由
router.addRoute({
  path: '/admin',
  name: 'Admin',
  component: () => import('@/views/admin/AdminLayout.vue'),
  children: []
})

// 再添加子路由（第一个参数为父路由 name）
router.addRoute('Admin', {
  path: 'dashboard',
  name: 'AdminDashboard',
  component: () => import('@/views/admin/Dashboard.vue')
})
```

### 3.2 动态删除与检查

```typescript
router.removeRoute('AdminDashboard')  // 按 name 删除
if (router.hasRoute('Admin')) { ... }     // 检查是否存在
const allRoutes = router.getRoutes()       // 获取所有路由
```

---

## 四、组件中使用 $router 和 $route

### 4.1 Options API

Vue3 Options API 中 `this.$router` 和 `this.$route` 仍然可用，但需用 `defineComponent` 包裹。

### 4.2 Composition API（推荐）

```vue
<script setup lang="ts">
import { useRouter, useRoute } from 'vue-router'
import { watch } from 'vue'

const router = useRouter()
const route = useRoute()

// 导航
function goToUser(id: number) {
  router.push({ name: 'User', params: { id } })
}

// 监听路由变化
watch(
  () => route.path,
  (newPath, oldPath) => {
    console.log(`路由变化: ${oldPath} → ${newPath}`)
  }
)
</script>
```

---

## 五、路由元信息类型增强

```typescript
// src/types/router.d.ts
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    title?: string
    icon?: string
    roles?: string[]
    keepAlive?: boolean
    showInSidebar?: boolean
    sortOrder?: number
    [key: string]: unknown
  }
}
```

---

## 六、特殊路由场景

### 6.1 通配符路由

```typescript
// Vue3 自定义正则参数替代通配符
{ path: '/:pathMatch(.*)*', name: 'NotFound', redirect: '/404' }
{ path: '/user-:afterUser(.*)', component: () => import('@/views/UserWildcard.vue') }

// *: 匹配 0 个或多个路径片段
// +: 匹配 1 个或多个路径片段
```

### 6.2 router-link 变化

```vue
<!-- Vue3：tag 和 event props 已移除，使用 v-slot -->
<router-link :to="{ name: 'User' }" custom v-slot="{ navigate, href, isActive }">
  <li @click="navigate" :class="{ active: isActive }">
    <a :href="href">用户</a>
  </li>
</router-link>
```

### 6.3 重复导航

Vue Router 4 在重复导航到相同路由时不再抛出 `NavigationDuplicated` 错误，可移除 catch 处理逻辑。

---

## 七、输入/输出

### 输入
- Vue2 路由配置文件路径
- Vue3 项目目标目录

### 输出
- Vue3 路由配置文件迁移后内容
- 迁移变更清单 (JSON)
- 待处理事项列表
- 组件守卫修改建议

---

## 八、错误处理

| 错误类型 | 处理方式 |
|---------|----------|
| next() 未迁移 | 扫描检测所有守卫函数，标记未迁移的 next() |
| 通配符未替换 | 自动替换为 `/:pathMatch(.*)` |
| 重复路由名称 | 重命名并记录变更 |
| 缺失 meta 类型 | 自动生成 RouteMeta 类型扩展 |
| base 位置错误 | 自动移至历史模式函数参数 |