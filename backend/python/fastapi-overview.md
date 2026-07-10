# FastAPI 概述

## 什么是 FastAPI？

FastAPI 是一个现代、快速（高性能）的 Web 框架，用于构建 API。

## 核心特点

- **高性能**：基于 Starlette 和 Pydantic
- **快速开发**：开发速度提高 200% - 300%
- **更少 Bug**：减少 40% 的人为错误
- **直观**：强大的编辑器支持
- **简单**：易于学习和使用

## 快速开始

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```

## 请求体

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

@app.post("/users/")
async def create_user(user: UserCreate):
    return user
```