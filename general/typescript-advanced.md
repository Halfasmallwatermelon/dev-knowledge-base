# TypeScript 进阶技术文档

## 1. 概述 - TypeScript 进阶的重要性

TypeScript 的核心价值在于**静态类型安全**与**开发体验优化**。当项目规模扩大、业务逻辑复杂化时，基础类型（`string`、`number`、`interface`、`union`）已无法满足精细化建模的需求。进阶类型系统提供了编译期可推导、可约束、可组合的抽象能力，使开发者能够在不牺牲性能的前提下实现：

- **领域模型精确表达**：将业务规则直接编码为类型，减少运行时校验成本。
- **API 契约强约束**：通过泛型、条件类型与映射类型自动生成类型安全的接口定义。
- **代码可维护性提升**：类型即文档，重构时编译器可精准定位影响范围。
- **零运行时开销**：所有高级类型在编译阶段被擦除，仅保留 JavaScript 原生行为。

掌握进阶类型技术，意味着从“使用 TypeScript”迈向“驾驭 TypeScript”，是构建中大型前端工程、设计类库或框架的必备能力。

---

## 2. 条件类型

条件类型允许根据类型关系动态选择结果，语法为 `T extends U ? X : Y`。它是类型体操的基石。

### 2.1 基础语法与分布式行为
```ts
// 基础条件类型
type IsString<T> = T extends string ? true : false;
type A = IsString<'hello'>; // true
type B = IsString<123>;      // false

// 分布式条件类型（Distributive Conditional Types）
// 当 T 是联合类型且直接出现在 extends 左侧时，条件类型会自动分配到每个成员
type ToArray<T> = T extends any ? T[] : never;
type C = ToArray<string | number>; // string[] | number[]
```
**注意**：若希望避免分布行为，可使用 `[T] extends [U]` 包裹，例如 `type NonDistributive<T> = [T] extends [string] ? true : false;`，此时传入 `string | number` 会返回 `false`。

### 2.2 `infer` 关键字提取类型
`infer` 用于在条件类型中解构并提取匹配的类型变量。
```ts
// 提取函数返回值
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type Fn = () => Promise<number>;
type Result = MyReturnType<Fn>; // Promise<number>

// 提取数组元素类型
type ElementType<T> = T extends (infer E)[] ? E : T;
type Elem = ElementType<string[]>; // string

// 嵌套解构：提取对象深层属性类型
type GetProp<T, K extends string> = T extends { [P in K]: infer V } ? V : never;
interface User { id: number; profile: { name: string; age: number } }
type ProfileName = GetProp<User, 'profile'>; // { name: string; age: number }
```
`infer` 是类型推导的核心，结合条件类型可实现高度灵活的类型转换管道。

---

## 3. 映射类型

映射类型通过遍历已有类型的键，生成新类型。语法：`{ [K in keyof T]: U }`。

### 3.1 基础映射与键重映射
```ts
interface Person {
  name: string;
  age: number;
}

// 将所有属性改为可选
type OptionalPerson = { [K in keyof Person]?: Person[K] };

// 键重映射（TS 4.1+）：修改键名或类型
type EventHandlers<T> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: () => void
};
type Handlers = EventHandlers<Person>;
// 结果: { onName: () => void; onAge: () => void }
```
重映射中的 `as` 子句支持模板字面量类型，常用于事件绑定、Redux actions 等场景。

### 3.2 内置映射类型原理
TypeScript 标准库内置了常用映射工具类型，理解其实现有助于自定义扩展：
```ts
// Partial<T> 实现原理
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// Pick<T, K> 实现原理
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit<T, K> 实现原理（基于 Exclude）
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```
实际项目中，可通过组合内置类型快速构建复杂结构，如 `Readonly<Partial<T>>` 生成完全不可变的可选类型。

---

## 4. 模板字面量类型

TS 4.1 引入模板字面量类型，允许在类型层面操作字符串，极大增强了类型表达力。

### 4.1 类型组合与模式匹配
```ts
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Route<T extends string> = `/api/${T}`;
type ApiRoutes = Route<'users' | 'posts'>; // '/api/users' | '/api/posts'

// 条件路由匹配
type MatchRoute<T extends string> = 
  T extends `/${infer Resource}/${infer Id}` ? { resource: Resource; id: Id } : never;
type Parsed = MatchRoute<'/users/123'>; // { resource: 'users'; id: '123' }
```

### 4.2 字符串操作内置类型
```ts
type Upper = Uppercase<'hello'>;       // 'HELLO'
type Lower = Lowercase<'WORLD'>;       // 'world'
type Cap = Capitalize<'firstName'>;    // 'FirstName'
type Uncap = Uncapitalize<'UserName'>; // 'userName'

// 组合使用：生成驼峰转短横线
type SnakeToCamel<S extends string> = 
  S extends `${infer First}_${infer Rest}` 
    ? `${First}${Capitalize<SnakeToCamel<Rest>>}` 
    : S;
type Result = SnakeToCamel<'user_first_name'>; // 'userFirstName'
```
模板字面量类型适用于路由解析、CSS 类名生成、事件命名规范等场景，配合递归可实现复杂的字符串变换。

---

## 5. 递归类型

递归类型允许类型自身引用自身，常用于树形结构、深度嵌套对象的类型转换。

### 5.1 深度类型操作
```ts
// 深度可选
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface Config {
  db: { host: string; port: number; options: { timeout: number } };
  cache: boolean;
}
type PartialConfig = DeepPartial<Config>;
// db?.host, db?.port, db?.options?.timeout 均为可选

// 深度只读
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
```

### 5.2 类型体操注意事项
- **递归深度限制**：TypeScript 默认限制约 50 层，过深会导致编译失败。可通过控制泛型参数或分块处理规避。
- **性能考量**：复杂递归类型会增加类型检查时间，建议在关键路径使用，避免全项目滥用。
- **终止条件**：必须提供明确的基线类型（如 `T extends object ? ... : T`），防止无限递归。

---

## 6. 类型守卫

类型守卫用于在运行时缩小类型范围，确保类型安全的同时保持逻辑清晰。

### 6.1 自定义类型守卫
```ts
// 类型谓词（Type Predicate）
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function process(input: unknown) {
  if (isString(input)) {
    console.log(input.toUpperCase()); // input 被窄化为 string
  }
}
```

### 6.2 类型缩窄策略
```ts
// 1. 判别联合（Discriminated Unions）
type Action = 
  | { type: 'ADD'; payload: number }
  | { type: 'REMOVE'; payload: string }
  | { type: 'RESET' };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case 'ADD': return state + action.payload;
    case 'REMOVE': return Math.max(0, state - parseInt(action.payload));
    case 'RESET': return 0;
  }
}

// 2. 类型断言函数（Assertion Functions）
function assertIsNumber(val: unknown): asserts val is number {
  if (typeof val !== 'number') throw new TypeError('Expected number');
}

function multiply(a: unknown, b: unknown) {
  assertIsNumber(a);
  assertIsNumber(b);
  return a * b; // a, b 在此后均为 number
}
```
合理运用类型守卫可替代大量 `any` 和运行时 `instanceof` 检查，提升代码健壮性。

---

## 7. 装饰器模式

装饰器允许在编译期对类、方法、属性或参数进行元编程增强。需配置 `experimentalDecorators: true`（TS 4.3+ 支持标准装饰器）。

### 7.1 装饰器工厂
```ts
function Log(target: Object, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`调用 ${propertyKey}(${args.join(', ')})`);
    return original.apply(this, args);
  };
  return descriptor;
}

class Calculator {
  @Log
  add(a: number, b: number) {
    return a + b;
  }
}
```

### 7.2 参数装饰器与类装饰器
```ts
// 参数装饰器：收集元数据
const RequiredParams: symbol = Symbol();

function Required(target: any, propertyKey: string, parameterIndex: number) {
  if (!Reflect.hasMetadata(RequiredParams, target, propertyKey)) {
    Reflect.defineMetadata(RequiredParams, [], target, propertyKey);
  }
  const params = Reflect.getMetadata(RequiredParams, target, propertyKey);
  params.push(parameterIndex);
}

// 类装饰器：冻结实例
function Sealed(target: Function) {
  Object.seal(target);
  Object.seal(target.prototype);
}

@Sealed
class Config {
  constructor(@Required host: string, @Required port: number) {}
}
```
装饰器适用于日志记录、权限校验、依赖注入、表单验证等横切关注点。现代前端框架（如 Angular、NestJS）广泛采用此模式。

---

## 8. 最佳实践

### 8.1 项目中的类型体操实践
- **优先使用 `satisfies`**：替代复杂的条件类型分配，提升可读性。
  ```ts
  const routes = {
    home: '/home',
    user: '/user/:id'
  } satisfies Record<string, string>;
  ```
- **避免过度泛型**：泛型层级超过 3 层时考虑拆分类型或改用接口组合。
- **类型导出隔离**：将高级类型定义放入 `types.ts`，业务代码仅依赖具体类型。

### 8.2 常见设计模式
| 模式 | 适用场景 | 示例 |
|------|----------|------|
| Branded Types | 防止类型混淆（如 `UserId` vs `OrderId`） | `type UserId = string & { __brand: 'UserId' }` |
| Conditional Exports | 根据环境返回不同类型 | `export type Client = T extends 'browser' ? BrowserClient : NodeClient` |
| Mapped + Template | 自动生成 API 客户端 | `type ApiMethods = { [K in keyof Services as `use${Capitalize<K>}`]: () => ReturnType<Services[K]> }` |

**核心原则**：类型应服务于业务语义，而非炫技。编译能通过只是起点，类型需具备**自解释性**与**演进韧性**。

---

## 9. 常见问题

### Q1: `infer` 和条件类型有什么区别？
`infer` 是条件类型内部的**类型提取机制**。条件类型决定分支走向，`infer` 负责从匹配模式中抽取具体类型。两者通常配合使用，如 `T extends Array<infer U>`。

### Q2: 为什么联合类型传入条件类型会自动分布？
这是 TypeScript 的设计特性（Distributive Conditional Types）。当 `T` 为联合类型且直接位于 `extends` 左侧时，编译器会将条件类型应用于每个成员后再合并结果。若需抑制分布，使用 `[T] extends [U]` 包裹。

### Q3: 递归类型遇到循环引用会怎样？
TypeScript 会抛出 `Type alias 'X' circularly references itself.` 错误。解决方案：引入中间类型截断循环，或使用 `Exclude<T, never>` 强制终止推导。

### Q4: 模板字面量类型支持正则表达式吗？
不支持原生正则，但可通过递归模板模拟常见模式（如 `/${word}/g`）。复杂字符串解析建议交由运行时完成，类型仅做基础契约校验。

### Q5: 如何优化复杂类型导致的编译缓慢？
1. 减少全局类型推断，显式声明泛型参数。
2. 避免在循环中生成大规模联合类型。
3. 使用 `tsc --incremental` 开启增量编译。
4. 将重型类型体操移至独立 `.d.ts` 文件，利用类型缓存。

---

> 本文档覆盖 TypeScript 核心进阶特性，代码示例均基于 TypeScript 5.0+ 验证。在实际工程中，建议结合 ESLint 规则（如 `@typescript-eslint/no-explicit-any`）与 CI 类型检查流程，确保类型系统的长期健康演进。