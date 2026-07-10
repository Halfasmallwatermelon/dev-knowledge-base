# TypeScript 指南

## 基础类型

```typescript
// 原始类型
let name: string = "John"
let age: number = 25
let isActive: boolean = true

// 数组
let numbers: number[] = [1, 2, 3]
let names: Array<string> = ["a", "b", "c"]

// 元组
let tuple: [string, number] = ["John", 25]

// 枚举
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE"
}

// any（避免使用）
let anything: any = "hello"

// void
function log(message: string): void {
  console.log(message)
}
```

## 接口和类型

```typescript
// 接口
interface User {
  id: number
  name: string
  email: string
  age?: number  // 可选
}

// 类型别名
type ID = string | number
type Point = { x: number; y: number }

// 函数类型
type Callback = (data: any) => void
```

## 泛型

```typescript
// 泛型函数
function identity<T>(arg: T): T {
  return arg
}

// 泛型接口
interface Response<T> {
  data: T
  status: number
}

// 泛型类
class Box<T> {
  private value: T
  constructor(value: T) {
    this.value = value
  }
  getValue(): T {
    return this.value
  }
}
```

## 类

```typescript
class User {
  public name: string
  protected email: string
  private password: string

  constructor(name: string, email: string, password: string) {
    this.name = name
    this.email = email
    this.password = password
  }

  // 方法
  getName(): string {
    return this.name
  }

  // 静态方法
  static create(name: string): User {
    return new User(name, '', '')
  }
}

// 继承
class Admin extends User {
  role: string
  constructor(name: string, role: string) {
    super(name, '', '')
    this.role = role
  }
}
```

## 实用工具类型

```typescript
// Partial：所有属性可选
type PartialUser = Partial<User>

// Required：所有属性必填
type RequiredUser = Required<User>

// Pick：选取部分属性
type UserName = Pick<User, 'name' | 'email'>

// Omit：排除部分属性
type UserWithoutPassword = Omit<User, 'password'>

// Record：构造键值对类型
type Users = Record<string, User>
```

## 类型守卫

```typescript
// typeof
function isString(value: any): value is string {
  return typeof value === 'string'
}

// instanceof
function isError(value: any): value is Error {
  return value instanceof Error
}

// 自定义类型守卫
interface Cat {
  meow(): void
}
interface Dog {
  bark(): void
}

function isCat(animal: Cat | Dog): animal is Cat {
  return (animal as Cat).meow !== undefined
}
```