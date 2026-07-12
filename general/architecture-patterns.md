# 架构模式

## MVC

```
Model (模型) → 数据和业务逻辑
View (视图) → 用户界面
Controller (控制器) → 处理用户输入
```

```python
# Django 示例
class User(models.Model):  # Model
    name = models.CharField(max_length=100)

def user_list(request):  # Controller
    users = User.objects.all()
    return render(request, 'users/list.html', {'users': users})  # View
```

## MVVM

```
Model → 数据模型
View → 用户界面
ViewModel → 视图逻辑
```

```javascript
// Vue 3 示例
<template>
  <div>{{ message }}</div>
  <button @click="changeMessage">Change</button>
</template>

<script setup>
import { ref } from 'vue'

const message = ref('Hello')

function changeMessage() {
  message.value = 'World'
}
</script>
```

## Clean Architecture

```
┌─────────────────────────────────┐
│        Presentation             │
├─────────────────────────────────┤
│        Application              │
├─────────────────────────────────┤
│        Domain                   │
├─────────────────────────────────┤
│        Infrastructure           │
└─────────────────────────────────┘
```

```python
# Domain 层
class Order:
    def __init__(self, items):
        self.items = items
    
    def total(self):
        return sum(item.price for item in self.items)

# Application 层
class CreateOrderUseCase:
    def __init__(self, order_repo):
        self.order_repo = order_repo
    
    def execute(self, items):
        order = Order(items)
        self.order_repo.save(order)
        return order

# Infrastructure 层
class SqlOrderRepository:
    def save(self, order):
        # 数据库操作
        pass
```

## Event-Driven

```python
# 事件发布
class EventBus:
    def __init__(self):
        self.handlers = {}
    
    def subscribe(self, event_type, handler):
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)
    
    def publish(self, event):
        for handler in self.handlers.get(type(event), []):
            handler(event)

# 使用
event_bus = EventBus()
event_bus.subscribe(OrderCreatedEvent, send_confirmation_email)
event_bus.publish(OrderCreatedEvent(order))
```

## Serverless

```python
# AWS Lambda
import json

def handler(event, context):
    body = json.loads(event['body'])
    
    # 处理请求
    result = process_data(body)
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }
```
