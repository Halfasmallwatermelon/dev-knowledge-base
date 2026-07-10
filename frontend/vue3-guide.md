# Vue 3 开发指南

## Composition API

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubleCount = computed(() => count.value * 2)

function increment() {
  count.value++
}

onMounted(() => {
  console.log('Component mounted')
})
</script>
```

## Pinia 状态管理

```javascript
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    name: '',
    email: ''
  }),
  actions: {
    async login(credentials) {
      const response = await api.login(credentials)
      this.name = response.name
    }
  }
})
```