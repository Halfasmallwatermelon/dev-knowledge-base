# TypeScript + React

## 组件类型

```typescript
// 函数组件
interface UserProps {
  name: string
  age?: number
  onClick: (id: number) => void
}

function UserCard({ name, age = 0, onClick }: UserProps) {
  return (
    <div onClick={() => onClick(1)}>
      <h2>{name}</h2>
      {age > 0 && <p>Age: {age}</p>}
    </div>
  )
}

// 泛型组件
interface ListProps<T> {
  items: T[]
  renderItem: (item: T) => React.ReactNode
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{renderItem(item)}</li>
      ))}
    </ul>
  )
}
```

## Hooks 类型

```typescript
// useState
const [count, setCount] = useState<number>(0)
const [user, setUser] = useState<User | null>(null)

// useEffect
useEffect(() => {
  fetchData()
}, [])

// useCallback
const handleClick = useCallback((id: number) => {
  console.log(id)
}, [])

// useMemo
const memoizedValue = useMemo(() => {
  return expensiveCalculation(data)
}, [data])

// useRef
const inputRef = useRef<HTMLInputElement>(null)

// 自定义 Hook
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    const item = localStorage.getItem(key)
    return item ? JSON.parse(item) : initialValue
  })
  
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))
  }, [key, value])
  
  return [value, setValue] as const
}
```

## Context 类型

```typescript
interface AuthContextType {
  user: User | null
  login: (credentials: Credentials) => Promise<void>
  logout: () => void
}

const AuthContext = React.createContext<AuthContextType | undefined>(undefined)

function useAuth() {
  const context = useContext(AuthContext)
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider')
  }
  return context
}
```

## API 类型

```typescript
// Axios
interface ApiResponse<T> {
  data: T
  status: number
}

async function fetchUsers(): Promise<ApiResponse<User[]>> {
  const response = await axios.get<User[]>('/api/users')
  return response
}

// React Query
function useUsers() {
  return useQuery<User[], Error>({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(res => res.json())
  })
}
```