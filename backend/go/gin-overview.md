# Gin 框架概述

## 什么是 Gin？

Gin 是一个用 Go 编写的高性能 Web 框架。

## 快速开始

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    
    r.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "Hello World",
        })
    })
    
    r.Run(":8080")
}
```

## 路由

```go
r.GET("/users", getUsers)
r.POST("/users", createUser)
r.GET("/users/:id", getUser)
```