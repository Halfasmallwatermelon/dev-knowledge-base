# Vue 3 进阶

## 响应式原理

```javascript
// Vue 3 使用 Proxy 实现响应式
const state = new Proxy(data, {
  get(target, key) {
    track(target, key)  // 依赖收集
    return target[key]
  },
  set(target, key, value) {
    target[key] = value
    trigger(target, key)  // 触发更新
    return true
  }
})
```

## Teleport

```vue
<template>
  <button @click="showModal = true">Open</button>
  <Teleport to="body">
    <div v-if="showModal" class="modal">
      <p>Modal content</p>
      <button @click="showModal = false">Close</button>
    </div>
  </Teleport>
</template>
```

## Suspense

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncComponent />
    </template>
    <template #fallback>
      <div>Loading...</div>
    </template>
  </Suspense>
</template>

<script setup>
const AsyncComponent = defineAsyncComponent(() => import('./HeavyComponent.vue'))
</script>
```

## 自定义指令

```javascript
// v-focus
const vFocus = {
  mounted: (el) => el.focus()
}

// v-permission
const vPermission = {
  mounted: (el, binding) => {
    const permission = binding.value
    if (!user.permissions.includes(permission)) {
      el.parentNode.removeChild(el)
    }
  }
}
```

## Pinia 进阶

```javascript
export const useUserStore = defineStore('user', {
  state: () => ({
    users: [],
    current: null
  }),
  
  getters: {
    activeUsers: (state) => state.users.filter(u => u.active),
    getUserById: (state) => (id) => state.users.find(u => u.id === id)
  },
  
  actions: {
    async fetchUsers() {
      this.users = await api.getUsers()
    },
    
    async updateUser(id, data) {
      const user = await api.updateUser(id, data)
      const index = this.users.findIndex(u => u.id === id)
      this.users[index] = user
    }
  },
  
  persist: true  // 持久化
})
```

## 性能优化

```javascript
// 1. 使用 v-once
<div v-once>{{ expensive }}</div>

// 2. 使用 v-memo
<div v-memo="[item.id]">{{ item.name }}</div>

// 3. 虚拟滚动
<script setup>
import { useVirtualList } from '@vueuse/core'
</script>
```
