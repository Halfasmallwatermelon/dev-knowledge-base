# TypeScript 进阶

## 条件类型

```typescript
type IsString<T> = T extends string ? true : false
type A = IsString<string>  // true
type B = IsString<number>  // false
```

## 映射类型

```typescript
type Readonly<T> = {
  readonly [P in keyof T]: T[P]
}

type Partial<T> = {
  [P in keyof T]?: T[P]
}

interface User {
  name: string
  age: number
}

type ReadonlyUser = Readonly<User>
```

## infer 关键字

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never

type Fn = () => string
type Result = ReturnType<Fn>  // string
```

## 装饰器

```typescript
function sealed(constructor: Function) {
  Object.seal(constructor)
  Object.seal(constructor.prototype)
}

@sealed
class Greeter {
  greeting: string
  constructor(message: string) {
    this.greeting = message
  }
}
```

## 高级类型体操

```typescript
// DeepPartial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P]
}

// Pick
type Pick<T, K extends keyof T> = {
  [P in K]: T[P]
}

// Omit
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>
```

## 类型守卫进阶

```typescript
function isString(value: unknown): value is string {
  return typeof value === 'string'
}

function processValue(value: string | number) {
  if (isString(value)) {
    return value.toUpperCase()
  }
  return value.toFixed(2)
}
```
