# 状态管理转化 Skill (State Management Conversion)

## 概述

本 Skill 封装了 Vuex 3.x → Pinia 的所有转化规则，供状态管理专家在迁移任务中调用。

---

## 一、核心迁移对照

| 概念 | Vuex 3.x | Pinia |
|------|----------|-------|
| 创建 Store | `new Vuex.Store({...})` | `defineStore({...})` |
| 安装插件 | `Vue.use(Vuex)` | `app.use(createPinia())` |
| 状态 | `state` 对象 | `state` 箭头函数 |
| 获取器 | `getters` 对象 | `getters` 对象 |
| 变更 | `mutations` 对象 | **移除**，直接修改 state |
| 操作 | `actions` 对象 | `actions` 对象 |
| 模块 | `modules` 嵌套 | 多个独立 Store |
| 命名空间 | `namespaced: true` | **移除**，Store 名自动隔离 |
| 辅助函数 | `mapState/mapGetters/mapActions/mapMutations` | **移除**，直接 import Store |

---

## 二、Store 迁移

### 2.1 模块化 Store 迁移

#### Vuex 模块 → Pinia Store

```javascript
// Vue2 Vuex 模块
// store/modules/user.js
export default {
  namespaced: true,
  state: {
    token: null,
    userInfo: {}
  },
  getters: {
    isLoggedIn: state => !!state.token,
    userName: state => state.userInfo.name || ''
  },
  mutations: {
    SET_TOKEN(state, token) {
      state.token = token
    },
    SET_USER_INFO(state, info) {
      state.userInfo = info
    }
  },
  actions: {
    async login({ commit }, credentials) {
      const { token, user } = await api.login(credentials)
      commit('SET_TOKEN', token)
      commit('SET_USER_INFO', user)
      return token
    },
    logout({ commit }) {
      commit('SET_TOKEN', null)
      commit('SET_USER_INFO', {})
    }
  }
}
```

```typescript
// Vue3 Pinia Store
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { api } from '@/utils/api'

export const useUserStore = defineStore('user', () => {
  // state → refs
  const token = ref<string | null>(null)
  const userInfo = ref<Record<string, any>>({})

  // getters → computed
  const isLoggedIn = computed(() => !!token.value)
  const userName = computed(() => userInfo.value.name || '')

  // mutations 移除，直接修改 state
  function setToken(newToken: string | null) {
    token.value = newToken
  }
  function setUserInfo(info: Record<string, any>) {
    userInfo.value = info
  }

  // actions
  async function login(credentials: { username: string; password: string }) {
    const { token: t, user } = await api.login(credentials)
    token.value = t
    userInfo.value = user
    return t
  }

  function logout() {
    token.value = null
    userInfo.value = {}
  }

  return { token, userInfo, isLoggedIn, userName, setToken, setUserInfo, login, logout }
})
```

### 2.2 Options API 风格 Store

```typescript
// Pinia Options API 风格（适合从 Vuex 直接迁移）
export const useUserStore = defineStore('user', {
  state: () => ({
    token: null as string | null,
    userInfo: {} as Record<string, any>
  }),
  getters: {
    isLoggedIn: (state) => !!state.token,
    userName: (state) => state.userInfo.name || ''
  },
  actions: {
    async login(credentials: { username: string; password: string }) {
      const { token, user } = await api.login(credentials)
      this.token = token  // 直接修改，不需要 mutation
      this.userInfo = user
      return token
    },
    logout() {
      this.token = null
      this.userInfo = {}
    }
  }
})
```

---

## 三、组件中使用

### 3.1 Composition API（推荐）

```vue
<!-- Vue3 Composition API -->
<script setup lang="ts">
import { useUserStore } from '@/stores/user'
import { useProductStore } from '@/stores/product'
import { storeToRefs } from 'pinia'

const userStore = useUserStore()
const productStore = useProductStore()

// 解构保持响应性
const { token, isLoggedIn, userName } = storeToRefs(userStore)

// 直接调用 actions（无需 dispatch）
function handleLogin() {
  userStore.login({ username: 'admin', password: '123456' })
}

// 直接修改 state
function updateName(name: string) {
  userStore.userInfo.name = name
}

// 多个 Store 协同
async function purchase(productId: string) {
  await productStore.buy(productId)
  userStore.setToken(await userStore.login({ ... }))
}
</script>
```

### 3.2 Options API 兼容

```vue
<script>
import { mapState, mapActions } from 'pinia'  // 来自 pinia，不是 vuex
import { useUserStore } from '@/stores/user'

export default {
  computed: {
    ...mapState(useUserStore, ['token', 'isLoggedIn']),
    ...mapState(useUserStore, { userName: 'userName' })
  },
  methods: {
    ...mapActions(useUserStore, ['login', 'logout'])
  }
}
</script>
```

---

## 四、Mutations → 直接修改

| Vuex | Pinia |
|------|-------|
| `commit('SET_TOKEN', token)` | `store.token = token` |
| `commit('SET_USER_INFO', info)` | `store.userInfo = info` |
| `commit('increment')` | `store.counter++` |
| `commit('addTodo', todo)` | `store.todos.push(todo)` |

**$reset 方法**：Pinia Store 内置 `$reset()` 方法可将 state 重置为初始值。

---

## 五、Store 间引用

```typescript
// stores/cart.ts
import { defineStore } from 'pinia'
import { useUserStore } from './user'

export const useCartStore = defineStore('cart', {
  state: () => ({
    items: [] as Array<{ id: string; name: string; price: number }>
  }),
  getters: {
    totalPrice: (state) => state.items.reduce((sum, item) => sum + item.price, 0)
  },
  actions: {
    async checkout() {
      const userStore = useUserStore()
      if (!userStore.isLoggedIn) {
        throw new Error('请先登录')
      }
      // ...执行购买逻辑
    }
  }
})
```

---

## 六、安装与入口文件

```typescript
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
const pinia = createPinia()

// Pinia 插件（可选）
pinia.use(({ store }) => {
  store.$subscribe((mutation, state) => {
    console.log(`[Pinia] ${store.$id} 状态已变更`)
  })
})

app.use(pinia)
app.mount('#app')
```

---

## 七、插件处理

### 7.1 Vuex 插件 → Pinia 插件

```typescript
// Vuex 插件示例
const myPlugin = (store: any) => {
  store.subscribe((mutation: any, state: any) => {
    localStorage.setItem('app-state', JSON.stringify(state))
  })
}

// Pinia 插件
const piniaPlugin = ({ store }: { store: any }) => {
  store.$subscribe((mutation: any, state: any) => {
    localStorage.setItem(store.$id, JSON.stringify(state))
  })
}
```

---

## 八、辅助函数替换

| Vuex 辅助函数 | Pinia 替换方式 |
|--------------|---------------|
| `mapState` | `mapState(useStore, keys)`（来自 pinia） |
| `mapGetters` | `mapState(useStore, keys)`（getter 统一为 state） |
| `mapActions` | `mapActions(useStore, keys)` |
| `mapMutations` | **移除**，直接调用 store 方法 |

---

## 九、常见迁移模式

### 9.1 本地存储持久化

```typescript
// stores/auth.ts
export const useAuthStore = defineStore('auth', {
  state: () => ({
    token: localStorage.getItem('token') || null
  }),
  actions: {
    setToken(token: string | null) {
      this.token = token
      if (token) {
        localStorage.setItem('token', token)
      } else {
        localStorage.removeItem('token')
      }
    }
  }
})
```

### 9.2 批量修改

```typescript
// 使用 $patch 批量修改
store.$patch({
  token: null,
  userInfo: {}
})

// 函数形式
store.$patch((state) => {
  state.token = null
  state.userInfo = {}
  state.permissions = []
})