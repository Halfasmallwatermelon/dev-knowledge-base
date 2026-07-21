# React 进阶技术文档

本文档面向中高级前端开发者，系统梳理 React 在现代工程中的核心进阶能力。内容基于 React 18+ 生态，涵盖性能优化、架构模式、状态管理、服务端渲染、并发特性及工程化实践。

---

## 1. React 性能优化（React.memo、useMemo、useCallback、虚拟列表）

### 概念解释
React 默认采用单向数据流，状态变更会触发组件树重渲染。进阶优化核心在于**减少不必要的渲染**与**降低单次渲染成本**：
- `React.memo`：组件级浅比较，阻止 props 未变化时的重渲染。
- `useMemo`：缓存计算结果，避免每次渲染重复执行昂贵函数。
- `useCallback`：稳定函数引用，配合子组件 memo 或依赖数组使用。
- 虚拟列表：仅渲染可视区域 DOM 节点，适用于万级以上列表。

### 完整代码示例

```tsx
// 1. React.memo + useCallback 组合
import { memo, useCallback, useState } from 'react';

interface UserCardProps {
  user: { id: number; name: string };
  onToggle: (id: number) => void;
}

const UserCard = memo(function UserCard({ user, onToggle }: UserCardProps) {
  return (
    <div className="card">
      <span>{user.name}</span>
      <button onClick={() => onToggle(user.id)}>Toggle</button>
    </div>
  );
});

export function UserList() {
  const [selected, setSelected] = useState<Set<number>>(new Set());

  // 稳定回调引用
  const handleToggle = useCallback((id: number) => {
    setSelected(prev => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  }, []);

  return (
    <ul>
      {users.map(u => (
        <UserCard key={u.id} user={u} onToggle={handleToggle} />
      ))}
    </ul>
  );
}

// 2. useMemo 缓存复杂计算
function ExpensiveChart({ data }: { data: number[] }) {
  const stats = useMemo(() => {
    // 模拟 CPU 密集计算
    return data.reduce((acc, v) => ({
      sum: acc.sum + v,
      max: Math.max(acc.max, v),
      min: Math.min(acc.min, v),
    }), { sum: 0, max: -Infinity, min: Infinity });
  }, [data]);

  return <pre>{JSON.stringify(stats)}</pre>;
}

// 3. 虚拟列表（使用 react-window）
import { FixedSizeList as List } from 'react-window';

function VirtualizedList({ items }: { items: string[] }) {
  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={40}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index]}</div>
      )}
    </List>
  );
}
```

### 最佳实践
- `React.memo` 仅对纯展示组件有效；若组件内部有大量局部状态或上下文消费，收益有限。
- `useCallback`/`useMemo` 的依赖数组必须准确；空数组 `[]` 仅在函数不依赖外部值时使用。
- 虚拟列表需固定高度或可预测高度；动态高度可使用 `VariableSizeList` 或 `react-virtuoso`。
- 优先通过**状态拆分**和**组件边界下沉**优化，而非盲目加 memo。

### 常见问题与解决方案
- **问题**：加了 `React.memo` 后仍然频繁渲染。
  **解决**：检查父组件是否每次渲染创建新对象/函数作为 props。使用 `useMemo` 缓存对象，`useCallback` 缓存函数。
- **问题**：`useMemo` 反而拖慢性能。
  **解决**：浅比较本身有开销。仅对计算成本高于比较成本的场景使用。
- **问题**：虚拟列表滚动卡顿。
  **解决**：确保列表项样式固定；避免在列表项内使用复杂动画或异步加载未预热的图片。

---

## 2. 自定义 Hook 设计模式

### 概念解释
自定义 Hook 是 React 逻辑复用的核心机制。优秀设计应遵循：单一职责、副作用隔离、可组合、支持 SSR、提供明确的生命周期语义。

### 完整代码示例

```tsx
// useLocalStorage
import { useState, useEffect } from 'react';

export function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T | ((prev: T) => T)) => void] {
  const [stored, setStored] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue;
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch { return initialValue; }
  });

  useEffect(() => {
    try { window.localStorage.setItem(key, JSON.stringify(stored)); } catch {}
  }, [key, stored]);

  return [stored, setStored];
}

// useDebounce
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}

// useFetch（支持 AbortController、缓存、错误处理）
import { useState, useEffect, useRef, useCallback } from 'react';

type FetchState<T> = { data: T | null; loading: boolean; error: Error | null; refetch: () => void };

export function useFetch<T>(url: string, options?: RequestInit): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({ data: null, loading: true, error: null, refetch: () => {} });
  const controllerRef = useRef<AbortController | null>(null);

  const fetchFn = useCallback(async () => {
    controllerRef.current?.abort();
    const controller = new AbortController();
    controllerRef.current = controller;

    setState(s => ({ ...s, loading: true, error: null }));
    try {
      const res = await fetch(url, { ...options, signal: controller.signal });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const data = await res.json();
      setState({ data, loading: false, error: null, refetch: fetchFn });
    } catch (err: any) {
      if (err.name !== 'AbortError') setState(s => ({ ...s, loading: false, error: err }));
    }
  }, [url, JSON.stringify(options)]);

  useEffect(() => { fetchFn(); return () => controllerRef.current?.abort(); }, [fetchFn]);
  return state;
}

// useMediaQuery
import { useState, useEffect } from 'react';

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() =>
    typeof window !== 'undefined' ? window.matchMedia(query).matches : false
  );

  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, [query]);

  return matches;
}
```

### 最佳实践
- Hook 命名必须以 `use` 开头，且只能调用其他 Hook。
- 副作用必须返回清理函数，防止内存泄漏。
- 网络请求类 Hook 应内置 `AbortController` 避免竞态条件。
- 避免在 Hook 内部直接操作 DOM；将命令式需求交给 `ref` 或 `useImperativeHandle`。

### 常见问题与解决方案
- **问题**：Hook 内部状态在 SSR/CSR 切换时 hydration mismatch。
  **解决**：使用 `typeof window === 'undefined'` 或 `useEffect` 初始化客户端状态。
- **问题**：依赖数组遗漏导致闭包过期。
  **解决**：启用 ESLint `react-hooks/exhaustive-deps`；复杂依赖使用 `useRef` 存储最新值。
- **问题**：多个 Hook 共享全局状态冲突。
  **解决**：通过参数注入或组合 Hook 显式传递上下文，避免隐式单例。

---

## 3. 状态管理进阶（Context优化、Zustand、Jotai、Redux Toolkit）

### 概念解释
- **Context**：适合低频、跨层级共享状态；高频更新会导致所有消费者重渲染。
- **Zustand**：轻量、无 Provider 包裹，基于订阅/选择器实现细粒度更新。
- **Jotai**：原子化状态模型，天然支持派生状态与异步原子。
- **Redux Toolkit**：企业级可预测状态机，内置 Immer、RTK Query、DevTools。

### 完整代码示例

```tsx
// Context 优化：拆分 + 选择器模式
import { createContext, useContext, useMemo } from 'react';

interface ThemeCtx { theme: 'light' | 'dark'; setTheme: (t: any) => void; }
const ThemeContext = createContext<ThemeCtx | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState('light');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

// Zustand
import { create } from 'zustand';

interface CounterStore {
  count: number;
  increment: () => void;
  decrement: () => void;
}

export const useCounter = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set(s => ({ count: s.count + 1 })),
  decrement: () => set(s => ({ count: s.count - 1 })),
}));

// 组件中使用选择器避免无关渲染
function CountDisplay() {
  const count = useCounter(s => s.count);
  return <span>{count}</span>;
}

// Jotai
import { atom, useAtomValue, useSetAtom } from 'jotai';

const countAtom = atom(0);
const doubleCountAtom = atom(get => get(countAtom) * 2);

function DoubleCounter() {
  const doubled = useAtomValue(doubleCountAtom);
  const inc = useSetAtom(countAtom);
  return <button onClick={() => inc(c => c + 1)}>{doubled}</button>;
}

// Redux Toolkit
import { configureStore, createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { useDispatch, useSelector } from 'react-redux';

interface Todo { id: string; text: string; done: boolean; }
const todoSlice = createSlice({
  name: 'todos',
  initialState: { items: [] as Todo[], status: 'idle' as 'idle' | 'loading' | 'failed' },
  reducers: { addTodo: (s, a) => s.items.push({ id: crypto.randomUUID(), text: a.payload, done: false }) },
  extraReducers: (b) => b.addMatcher(/* 异步匹配 */),
});

export const store = configureStore({ reducer: { todos: todoSlice.reducer } });
export const useAppDispatch = useDispatch.withTypes<typeof store>();
export const useAppSelector = useSelector.withTypes<typeof store>();
```

### 最佳实践
- 优先使用局部状态；仅当状态需要跨组件树共享或持久化时才引入全局方案。
- 全局 Store 按领域拆分，避免单一大 Store。
- 使用 `createEntityAdapter`（RTK）管理关系型数据，保持扁平化。
- 网络请求状态优先使用 RTK Query 或 SWR/TanStack Query，而非手写 fetch。

### 常见问题与解决方案
- **问题**：Context 更新导致整棵树重渲染。
  **解决**：拆分多个 Context；使用 `useSyncExternalStore` 或 `useReducer` 限制订阅范围。
- **问题**：Zustand/Jotai 与 SSR 水合不一致。
  **解决**：在服务端初始化前设置默认值；客户端首次渲染后同步真实状态。
- **问题**：Redux 样板代码过多。
  **解决**：使用 Toolkit 的 `createSlice`、`createAsyncThunk`、`createEntityAdapter` 大幅缩减模板。

---

## 4. Server Components 与 RSC（服务端渲染、流式渲染、Suspense）

### 概念解释
React Server Components（RSC）允许组件在服务端执行，仅将序列化后的 UI 树发送给客户端。客户端组件（`'use client'`）负责交互。配合 Streaming SSR 与 Suspense，可实现渐进式渲染与零 JS 初始负载。

### 完整代码示例

```tsx
// app/page.tsx（Server Component）
import { UserProfile } from './UserProfile';
import { ProductGrid } from './ProductGrid';

export default async function HomePage() {
  const user = await fetch('https://api.example.com/user').then(r => r.json());
  const products = await fetch('https://api.example.com/products').then(r => r.json());

  return (
    <main>
      <h1>Welcome, {user.name}</h1>
      {/* Suspense 包裹客户端交互组件或耗时资源 */}
      <Suspense fallback={<ProductSkeleton />}>
        <ProductGrid items={products} />
      </Suspense>
      <UserProfile userId={user.id} />
    </main>
  );
}

// app/UserProfile.tsx
export async function UserProfile({ userId }: { userId: string }) {
  const profile = await db.users.find(userId);
  return <article>{profile.bio}</article>;
}

// app/ProductGrid.tsx（混合：服务端获取数据，客户端交互）
'use client';
import { useState } from 'react';

export function ProductGrid({ items }: { items: any[] }) {
  const [filter, setFilter] = useState('all');
  return (
    <section>
      <select onChange={e => setFilter(e.target.value)}>
        <option value="all">All</option>
        <option value="sale">Sale</option>
      </select>
      <ul>
        {items.filter(i => filter === 'all' || i.sale).map(p => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </section>
  );
}
```

### 最佳实践
- 默认编写 Server Components，仅在需要 `useState`/事件处理器/浏览器 API 时添加 `'use client'`。
- Suspense 边界应放在**可独立延迟的内容**上（如第三方组件、图表、实时数据）。
- 服务端组件可直接访问数据库、文件系统、密钥，无需暴露给客户端。
- 使用 `async/await` 在服务端并行请求数据，减少关键路径阻塞。

### 常见问题与解决方案
- **问题**：服务端组件无法使用 Hooks。
  **解决**：这是设计约束。将交互逻辑下沉到客户端子组件，通过 props 传递数据。
- **问题**：Streaming 导致页面闪烁。
  **解决**：为每个 Suspense 提供合理的 `fallback`；避免在首屏关键内容上滥用延迟。
- **问题**：调试困难。
  **解决**：Next.js App Router 提供 RSC Payload 查看器；使用 `console.log` 在服务端输出；区分 Client/Server 组件边界。

---

## 5. 并发特性（useTransition、useDeferredValue、并发模式）

### 概念解释
React 18 引入并发渲染能力，允许将 UI 更新分为**紧急**与**非紧急**优先级：
- `useTransition`：标记状态更新为非紧急，保持输入响应。
- `useDeferredValue`：延迟派生值更新，优先渲染当前值。
- 并发模式底层由 React Scheduler 调度，中断低优先级渲染以响应高优先级交互。

### 完整代码示例

```tsx
import { useState, useTransition, useDeferredValue } from 'react';

// 搜索框：输入期间保持流畅，过滤结果异步渲染
export function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    startTransition(() => setQuery(value)); // 非紧急更新
  };

  return (
    <div>
      <input value={query} onChange={handleChange} placeholder="Search..." />
      {isPending && <Spinner />}
      <ResultList query={deferredQuery} />
    </div>
  );
}

// Tab 切换：非活动 Tab 使用低优先级渲染
export function Dashboard() {
  const [tab, setTab] = useState('overview');
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <Tabs active={tab} onChange={(t) => startTransition(() => setTab(t))} />
      {isPending && <Overlay />}
      <TabContent tab={tab} />
    </>
  );
}
```

### 最佳实践
- 将 `useTransition` 包裹**用户输入触发的状态更新**，而非所有 `setState`。
- `useDeferredValue` 适合派生数据（过滤、排序、格式化），不要用于核心业务状态。
- 避免手动设置并发优先级；让 React 根据交互类型自动调度。
- 配合 Suspense 使用效果最佳，非紧急更新可在边界内降级展示。

### 常见问题与解决方案
- **问题**：过渡状态导致界面"卡住"。
  **解决**：确保非紧急更新不阻塞关键布局；为 Suspense 提供骨架屏。
- **问题**：并发更新顺序不可预测。
  **解决**：依赖 React 保证的提交顺序；避免在多个 setState 间假设同步结果，使用 `useEffect` 或函数式更新。
- **问题**：旧版浏览器不支持。
  **解决**：React 18 并发特性在主流现代浏览器已原生支持；Polyfill 仅针对极老环境，通常不建议。

---

## 6. 高阶组件与 Render Props 模式

### 概念解释
在 Hooks 普及前，HOC 与 Render Props 是逻辑复用的两大模式：
- **HOC**：函数接收组件，返回增强组件（`withAuth(User)`）。
- **Render Props**：通过 `render` 或函数子组件传递动态内容（`<Mouse render={mouse => ...} />`）。
两者本质都是**组合优于继承**，但存在嵌套过深、调试困难等问题。现代 React 推荐用自定义 Hook 替代。

### 完整代码示例

```tsx
// HOC：权限校验
function withAuth<P extends object>(
  WrappedComponent: React.ComponentType<P>
) {
  return function AuthWrapper(props: P & { user: User | null }) {
    if (!props.user) return <LoginPrompt />;
    return <WrappedComponent {...props} />;
  };
}

// Render Props：鼠标追踪
interface MouseProps {
  render: (pos: { x: number; y: number }) => React.ReactNode;
}
function Mouse({ render }: MouseProps) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  return <>{render(pos)}</>;
}

// 现代替代：自定义 Hook
function useMouse() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const h = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', h);
    return () => window.removeEventListener('mousemove', h);
  }, []);
  return pos;
}
```

### 最佳实践
- **新项目优先使用自定义 Hook**；HOC/Render Props 主要用于兼容遗留代码或库设计。
- HOC 应保留原组件 displayName，便于调试：`WrappedComponent.displayName = ...`。
- Render Props 避免嵌套过深，可拆分为多个小组件。
- 若必须复用，确保模式不混合使用，保持团队一致性。

### 常见问题与解决方案
- **问题**：HOC 包裹多层导致组件名混乱。
  **解决**：使用 `forwardRef` 透传 ref；设置静态属性与 displayName。
- **问题**：Render Props 造成"回调地狱"。
  **解决**：提取为独立组件或使用 Hook；将复杂逻辑拆分为多个 Render Props 组件。
- **问题**：与 Context/State 冲突。
  **解决**：HOC 不应修改被包裹组件的内部状态；通过 props 显式传递数据。

---

## 7. Ref 进阶（useRef、forwardRef、useImperativeHandle）

### 概念解释
`ref` 是 React 中绕过声明式渲染、进行命令式操作的通道：
- `useRef`：持有可变值，不触发重渲染；常用于定时器、DOM 引用、上一次状态。
- `forwardRef`：允许父组件访问子组件内部 DOM 或实例。
- `useImperativeHandle`：定制 ref 暴露的接口，隐藏内部实现。

### 完整代码示例

```tsx
import { useRef, forwardRef, useImperativeHandle, useEffect } from 'react';

// 1. useRef 存储上一次值
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => { ref.current = value; });
  return ref.current;
}

// 2. forwardRef + useImperativeHandle 封装聚焦输入框
interface FocusInputProps {
  placeholder: string;
  maxLength?: number;
}

export interface FocusInputRef {
  focus: () => void;
  clear: () => void;
}

const FocusInput = forwardRef<FocusInputRef, FocusInputProps>(function FocusInput(
  { placeholder, maxLength }, ref
) {
  const innerRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus: () => innerRef.current?.focus(),
    clear: () => {
      innerRef.current!.value = '';
      innerRef.current!.dispatchEvent(new Event('input', { bubbles: true }));
    },
  }));

  return <input ref={innerRef} placeholder={placeholder} maxLength={maxLength} />;
});

// 3. 父组件调用
function Parent() {
  const inputRef = useRef<FocusInputRef>(null);
  return (
    <>
      <FocusInput ref={inputRef} placeholder="Type..." />
      <button onClick={() => inputRef.current?.clear()}>Clear</button>
    </>
  );
}
```

### 最佳实践
- 能用声明式状态解决的，绝不使用 ref。
- ref 值变更不会触发渲染；若需响应变化，使用 `useEffect` 监听 ref 或改用状态。
- `useImperativeHandle` 应只暴露必要方法，避免暴露内部 DOM。
- TypeScript 中明确定义 ref 类型，避免 `any`。

### 常见问题与解决方案
- **问题**：`ref.current` 在首次渲染时为 `null`。
  **解决**：访问前做空值检查；或在 `useEffect` 中操作 DOM。
- **问题**：forwardRef 丢失组件 displayName。
  **解决**：React 18 自动推断；若需兼容旧版，手动设置 `displayName`。
- **问题**：ref 与状态不同步。
  **解决**：在状态更新后使用 `useEffect` 同步 ref；或使用 `flushSync`（谨慎使用）。

---

## 8. 表单处理（React Hook Form、Zod 验证）

### 概念解释
传统受控表单在每次按键时触发全组件重渲染。`React Hook Form` 基于 uncontrolled + ref 架构，将渲染次数降至最低。结合 `Zod` 进行 Schema 优先的类型安全验证，是现代表单的最佳实践。

### 完整代码示例

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('请输入有效邮箱'),
  password: z.string().min(8, '密码至少8位').regex(/[A-Z]/, '需包含大写字母'),
  age: z.number().min(18).max(120),
  role: z.enum(['admin', 'user']),
}).refine(data => data.password !== data.email, {
  message: '密码不能与邮箱相同',
  path: ['password'],
});

type FormData = z.infer<typeof schema>;

export function SignupForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
    mode: 'onTouched', // 失焦时验证
  });

  const onSubmit = async (data: FormData) => {
    await new Promise(r => setTimeout(r, 1000));
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input {...register('email')} placeholder="Email" />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input {...register('password')} type="password" placeholder="Password" />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      <input {...register('age', { valueAsNumber: true })} type="number" placeholder="Age" />
      {errors.age && <p role="alert">{errors.age.message}</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? '提交中...' : '注册'}
      </button>
    </form>
  );
}
```

### 最佳实践
- 使用 `uncontrolled` 模式；避免 `onChange` 中手动 `setState`。
- Schema 定义与组件分离，便于复用与单元测试。
- 文件上传、富文本等复杂字段使用 `Controller` 或 `useController` 桥接。
- 异步验证使用 `z.promise()` 或自定义 resolver。

### 常见问题与解决方案
- **问题**：动态表单项验证失效。
  **解决**：使用 `useFieldArray` 管理数组字段；Schema 使用 `z.array()` 并配置 `minLength`。
- **问题**：TypeScript 类型推断不完整。
  **解决**：始终通过 `z.infer<typeof schema>` 获取类型；避免手动声明接口。
- **问题**：提交后错误提示未清除。
  **解决**：在 `onSubmitSuccess` 中调用 `reset()`；或设置 `shouldUnregister: false` 保留历史值。

---

## 9. 测试进阶（React Testing Library、Mock Service Worker）

### 概念解释
- **React Testing Library (RTL)**：倡导测试用户行为而非实现细节。优先使用 `screen`、`waitFor`、`userEvent`。
- **Mock Service Worker (MSW)**：在浏览器网络层拦截请求，模拟真实 API 响应，避免 Jest `fetch` mock 带来的上下文失真。

### 完整代码示例

```tsx
// 1. RTL 组件测试
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SearchPage } from './SearchPage';

test('用户搜索后显示匹配结果', async () => {
  render(<SearchPage />);
  const input = screen.getByPlaceholderText('Search...');
  const user = userEvent.setup();

  await user.type(input, 'react');

  await waitFor(() => {
    expect(screen.getByText(/react/i)).toBeInTheDocument();
  });
});

// 2. MSW 网络拦截
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { render, screen, waitFor } from '@testing-library/react';
import { Profile } from './Profile';

const server = setupServer(
  http.get('/api/user/:id', ({ params }) => {
    if (params.id === 'invalid') return new HttpResponse(null, { status: 404 });
    return HttpResponse.json({ id: params.id, name: 'Alice', bio: 'Developer' });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('加载用户资料', async () => {
  render(<Profile userId="1" />);
  await waitFor(() => expect(screen.getByText('Alice')).toBeInTheDocument());
});
```

### 最佳实践
- 测试断言围绕**用户可见行为**：文本、按钮状态、路由跳转、网络请求结果。
- 避免 `findBy*` 与 `getBy*` 混用；同步内容用 `getBy*`，异步用 `findBy*` 或 `waitFor`。
- MSW 覆盖成功、失败、超时、分页等边界场景。
- 快照测试仅用于视觉回归；业务逻辑必须用行为测试验证。

### 常见问题与解决方案
- **问题**：测试不稳定（Flaky Tests）。
  **解决**：使用 `waitFor` 等待异步完成；避免固定 `setTimeout`；检查 MSW 响应顺序。
- **问题**：过度 Mock 导致测试脱离实际。
  **解决**：只在边界（API、第三方 SDK）使用 Mock；内部函数尽量真实调用。
- **问题**：RTL 找不到元素。
  **解决**：优先用 `getByText`/`getByRole`/`getByLabelText`；避免 `getByTestId` 除非必要。

---

## 10. 性能分析工具（React DevTools Profiler、Lighthouse）

### 概念解释
- **React DevTools Profiler**：记录组件渲染原因、耗时、更新频率，定位冗余渲染与瓶颈。
- **Lighthouse**：评估 Web 性能、可访问性、SEO、最佳实践，输出 Core Web Vitals（LCP、CLS、INP）。

### 完整代码示例与使用流程

```bash
# 1. React DevTools 分析步骤
# - Chrome 安装 React Developer Tools
# - 打开 Components 标签，勾选 "Highlight updates when components render"
# - 切换到 Profiler 标签，点击录制，执行目标操作，停止
# - 查看 Flamegraph / Why did render? / Commit 频率

# 2. Lighthouse 命令行
npx lighthouse https://your-app.com --output=json --output-path=./lhr.json --only-categories=performance,accessibility
```

### 最佳实践
- **先测量再优化**：使用 Profiler 定位具体组件，而非凭感觉加 `memo`。
- 在生产构建中测试性能；开发模式的警告与慢速渲染会干扰判断。
- Lighthouse 使用模拟设备与网络节流；关键指标应在真实设备与 4G/WiFi 下复测。
- 关注 **Render 原因**：是 props 变化、Context 更新、还是父组件重渲染导致？
- 结合 `web-vitals` 库在运行时采集真实用户数据（RUM）。

### 常见问题与解决方案
- **问题**：Profiler 显示大量组件渲染但页面流畅。
  **解决**：区分"渲染"与"绘制"。使用 Performance 面板查看主线程阻塞；DOM 节点少时渲染快不代表体验好。
- **问题**：Lighthouse 分数低但用户反馈正常。
  **解决**：检查是否为模拟器偏差；使用 Field Data（Chrome UX Report）对比真实 INP/LCP。
- **问题**：开启 Strict Mode 后性能数据异常。
  **解决**：Strict Mode 故意双重渲染以检测副作用；生产环境默认关闭，Profiler 录制时也应确认不在 Strict 模式下。

---

## 附录：进阶学习路径建议

| 阶段 | 重点 |
|------|------|
| 巩固基础 | 深入理解 Reconciliation、Fiber、批处理机制 |
| 架构选型 | 根据项目规模选择状态管理、SSR/ISR/SSG 策略 |
| 工程化 | Eslint 规则、Commit 规范、CI 性能门禁、监控告警 |
| 源码阅读 | 推荐 `react-reconciler`、`scheduler`、`use-sync-external-store` 实现 |

React 进阶的核心不是记住 API，而是理解**渲染时机、数据流向、边界划分**。掌握上述模式后，结合实际 Profiling 与用户反馈持续迭代，方能构建高性能、可维护的现代 React 应用。
