# React 进阶

## React.memo

```javascript
const MemoizedComponent = React.memo(({ data }) => {
  return <div>{data.name}</div>
})

// 只有 data 变化时才重新渲染
```

## useMemo & useCallback

```javascript
function App({ items, filter }) {
  // 缓存计算结果
  const filteredItems = useMemo(() => {
    return items.filter(item => item.category === filter)
  }, [items, filter])
  
  // 缓存函数引用
  const handleClick = useCallback((id) => {
    console.log(id)
  }, [])
  
  return (
    <List items={filteredItems} onItemClick={handleClick} />
  )
}
```

## 虚拟列表

```javascript
import { FixedSizeList } from 'react-window'

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={500}
      itemCount={items.length}
      itemSize={50}
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  )
}
```

## 代码分割

```javascript
const LazyComponent = React.lazy(() => import('./LazyComponent'))

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LazyComponent />
    </Suspense>
  )
}
```

## 错误边界

```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false }
  
  static getDerivedStateFromError(error) {
    return { hasError: true }
  }
  
  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>
    }
    return this.props.children
  }
}
```

## 自定义 Hook

```javascript
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const item = localStorage.getItem(key)
    return item ? JSON.parse(item) : initialValue
  })
  
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))
  }, [key, value])
  
  return [value, setValue]
}
```
