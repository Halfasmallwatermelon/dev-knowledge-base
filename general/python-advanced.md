# Python 高级特性

## 装饰器

```python
# 基础装饰器
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before function call")
        result = func(*args, **kwargs)
        print("After function call")
        return result
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

# 带参数的装饰器
def repeat(times):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet():
    print("Hello!")
```

## 生成器

```python
# 基础生成器
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1

for num in count_up_to(5):
    print(num)

# 生成器表达式
squares = (x**2 for x in range(10))
```

## 上下文管理器

```python
# 类实现
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
    
    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.file:
            self.file.close()

with FileManager('test.txt', 'w') as f:
    f.write("Hello!")

# contextlib
from contextlib import contextmanager

@contextmanager
def managed_resource():
    resource = acquire_resource()
    try:
        yield resource
    finally:
        release_resource(resource)
```

## 异步编程

```python
import asyncio

async def fetch_data():
    print("Start fetching")
    await asyncio.sleep(2)
    print("Done fetching")
    return {"data": 123}

async def main():
    task1 = asyncio.create_task(fetch_data())
    task2 = asyncio.create_task(fetch_data())
    await asyncio.gather(task1, task2)

asyncio.run(main())
```

## 元类

```python
class MyMeta(type):
    def __new__(cls, name, bases, dict):
        # 添加新属性
        dict['class_id'] = id(cls)
        return super().__new__(cls, name, bases, dict)

class MyClass(metaclass=MyMeta):
    pass
```

## 描述符

```python
class Property:
    def __init__(self, fget, fset=None):
        self.fget = fget
        self.fset = fset
    
    def __get__(self, obj, objtype=None):
        return self.fget(obj)
    
    def __set__(self, obj, value):
        if self.fset:
            self.fset(obj, value)

class Temperature:
    def __init__(self):
        self._celsius = 0
    
    @Property
    def celsius(self):
        return self._celsius
    
    @celsius.setter
    def celsius(self, value):
        self._celsius = value
    
    @Property
    def fahrenheit(self):
        return self._celsius * 9/5 + 32
    
    @fahrenheit.setter
    def fahrenheit(self, value):
        self._celsius = (value - 32) * 5/9
```