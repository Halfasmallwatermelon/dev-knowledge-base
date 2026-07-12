# 系统设计

## 高可用设计

```python
# 健康检查
@app.get("/health")
def health_check():
    return {"status": "healthy"}

# 负载均衡
upstream backend {
    server backend1:3000
    server backend2:3000
    server backend3:3000
}
```

## 缓存策略

```python
# Cache Aside Pattern
def get_user(user_id):
    # 先查缓存
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # 缓存未命中，查数据库
    user = db.query(User).get(user_id)
    
    # 写入缓存
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user

# 写时更新
def update_user(user_id, data):
    # 先更新数据库
    db.update(User, user_id, data)
    
    # 再删除缓存
    redis.delete(f"user:{user_id}")
```

## 消息队列

```python
# RabbitMQ
import pika

# 生产者
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='task_queue')
channel.basic_publish(exchange='', routing_key='task_queue', body='Hello')

# 消费者
def callback(ch, method, properties, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='task_queue', on_message_callback=callback)
```

## 分布式锁

```python
import redis

def acquire_lock(lock_name, timeout=10):
    identifier = str(uuid.uuid4())
    lock_key = f"lock:{lock_name}"
    
    if redis.set(lock_key, identifier, nx=True, ex=timeout):
        return identifier
    return None

def release_lock(lock_name, identifier):
    lock_key = f"lock:{lock_name}"
    lua_script = '''
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    '''
    return redis.eval(lua_script, 1, lock_key, identifier)
```

## 限流

```python
# 令牌桶
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.last_update = time.time()
    
    def consume(self):
        now = time.time()
        elapsed = now - self.last_update
        self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
        self.last_update = now
        
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

## CAP 理论

```
C (Consistency): 一致性
A (Availability): 可用性
P (Partition tolerance): 分区容错性

三者只能满足两个：
- CP 系统：ZooKeeper, etcd
- AP 系统：Cassandra, DynamoDB
```
