# 微服务进阶

## 服务网格（Service Mesh）

```yaml
# Istio 配置
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
```

## Saga 模式

```python
# 编排式 Saga
class OrderSaga:
    def __init__(self):
        self.steps = [
            CreateOrder(),
            ReserveInventory(),
            ProcessPayment(),
            ShipOrder()
        ]
    
    def execute(self, order):
        completed = []
        try:
            for step in self.steps:
                step.execute(order)
                completed.append(step)
        except Exception as e:
            # 补偿已完成的步骤
            for step in reversed(completed):
                step.compensate(order)
            raise
```

## CQRS

```python
# 命令端
class CreateOrderCommand:
    def execute(self, data):
        order = Order(data)
        db.save(order)
        event_store.append(OrderCreatedEvent(order))

# 查询端
class OrderQueryService:
    def get_order(self, order_id):
        # 从读优化的数据库查询
        return read_db.query(order_id)
```

## 事件溯源

```python
class Order:
    def __init__(self):
        self.events = []
    
    def apply(self, event):
        self.events.append(event)
        self._apply_event(event)
    
    def _apply_event(self, event):
        if isinstance(event, OrderCreatedEvent):
            self.id = event.order_id
            self.status = 'created'
        elif isinstance(event, OrderPaidEvent):
            self.status = 'paid'
```

## API 网关

```python
# FastAPI 网关
from fastapi import FastAPI
import httpx

app = FastAPI()

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"http://user-service:3000/users/{user_id}")
        return response.json()
```

## 限流熔断

```python
import circuitbreaker

@circuitbreaker.circuit(failure_threshold=5, recovery_timeout=60)
def call_service():
    response = requests.get("http://service/api")
    return response.json()

# 限流
from fastapi import FastAPI
from slowapi import Limiter

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter

@app.get("/api/data")
@limiter.limit("100/minute")
async def get_data():
    return {"data": "value"}
```
