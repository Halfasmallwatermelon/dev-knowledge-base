# NestJS 概述

## 什么是 NestJS？

NestJS 是一个用于构建高效、可靠和可扩展的服务端应用程序的渐进式 Node.js 框架。

## 核心概念

- **模块（Module）**：组织代码的基本单位
- **控制器（Controller）**：处理 HTTP 请求
- **服务（Service）**：业务逻辑
- **依赖注入（DI）**：解耦组件

## 快速开始

```bash
npm i -g @nestjs/cli
nest new my-app
cd my-app
npm run start:dev
```

## 模块

```typescript
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

## 控制器

```typescript
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll()
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id)
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto)
  }
}
```

## 服务

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  private users = []

  findAll() {
    return this.users
  }

  findOne(id: string) {
    return this.users.find(user => user.id === id)
  }

  create(user: CreateUserDto) {
    this.users.push(user)
    return user
  }
}
```

## 守卫

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest()
    return request.headers.authorization === 'valid-token'
  }
}

// 使用
@UseGuards(AuthGuard)
@Get('protected')
protectedRoute() {
  return 'This is protected'
}
```

## 拦截器

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now()
    return next.handle().pipe(
      tap(() => console.log(`Execution time: ${Date.now() - now}ms`))
    )
  }
}
```

## 与 Express 对比

| 特性 | NestJS | Express |
|------|--------|---------|
| 架构 | 模块化、依赖注入 | 灵活、无固定结构 |
| TypeScript | 原生支持 | 需要配置 |
| 企业级 | 适合 | 需要额外配置 |
| 学习曲线 | 较陡 | 平缓 |