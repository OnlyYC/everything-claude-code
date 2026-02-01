---
name: frontend-patterns
description: 前端开发模式：React、Next.js、Vue.js、Vanilla JS、状态管理、性能优化、UI 最佳实践。
version: 1.1.0
tech_stack: [React, Next.js, Vue.js, TypeScript, JavaScript]
tools: Read, Write, Edit, Bash, Grep, Glob
related_skills: [coding-standards, verification-loop]
---

# 前端开发模式

现代前端开发模式，涵盖 React、Next.js、Vue.js 和 Vanilla JavaScript，以及状态管理、性能优化和 UI 最佳实践。

## 如何选择框架（决策树）

```
开始：新项目前端技术栈选择
│
├── 是否需要服务端渲染（SSR）？
│   ├── 是 → 是否需要 SEO 优化？
│   │   ├── 是 → Next.js (React) 或 Nuxt.js (Vue)
│   │   └── 否 → 考虑客户端渲染框架
│   │
│   └── 否 → 是否需要复杂状态管理？
│       ├── 是 → React + Redux/Zustand 或 Vue + Pinia
│       └── 否 → 简单项目考虑轻量级方案
│           ├── 团队熟悉 TypeScript？
│           │   ├── 是 → React/Vue 都可以
│           │   └── 否 → 考虑学习曲线
│           └── 是否需要渐进式集成？
│               ├── 是 → Vue.js（渐进式友好）
│               └── 否 → React（生态更丰富）
│
├── 特殊场景：
│   ├── 老旧项目维护 → Vanilla JavaScript 模块
│   ├── 简单页面增强 → Alpine.js / Vanilla JS
│   └── 移动端优先 → React Native / Flutter（非本技能范围）
│
└── 本技能各框架章节跳转：
    ├── React 模式 → 见"组件模式"、"自定义 Hooks"章节
    ├── Next.js 优化 → 见"性能优化"章节
    ├── Vue.js 模式 → 见"Vue.js 模式"章节
    └── Vanilla JS → 见"Vanilla JavaScript 模式"章节
```

## 技术栈导航

本技能涵盖多个前端框架和模式，请根据项目需求选择相应部分：

| 技术栈 | 章节 | 适用场景 |
|--------|------|---------|
| React | 组件模式、自定义 Hooks、性能优化 | 单页应用 (SPA) |
| Next.js | SSR 优化、代码分割 | 服务端渲染应用 |
| Vue.js | Composition API、Pinia | 渐进式应用 |
| Vanilla JS | 模块模式、DOM 操作 | 轻量级项目、老旧项目维护 |

---

## 组件模式

### 组合优于继承

```typescript
// ✅ 良好：组件组合
interface CardProps {
  children: React.ReactNode
  variant?: 'default' | 'outlined'
}

export function Card({ children, variant = 'default' }: CardProps) {
  return <div className={`card card-${variant}`}>{children}</div>
}

export function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>
}

export function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>
}

// 使用方式
<Card>
  <CardHeader>标题</CardHeader>
  <CardBody>内容</CardBody>
</Card>
```

### 复合组件

```typescript
interface TabsContextValue {
  activeTab: string
  setActiveTab: (tab: string) => void
}

const TabsContext = createContext<TabsContextValue | undefined>(undefined)

export function Tabs({ children, defaultTab }: {
  children: React.ReactNode
  defaultTab: string
}) {
  const [activeTab, setActiveTab] = useState(defaultTab)

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  )
}

export function TabList({ children }: { children: React.ReactNode }) {
  return <div className="tab-list">{children}</div>
}

export function Tab({ id, children }: { id: string, children: React.ReactNode }) {
  const context = useContext(TabsContext)
  if (!context) throw new Error('Tab must be used within Tabs')

  return (
    <button
      className={context.activeTab === id ? 'active' : ''}
      onClick={() => context.setActiveTab(id)}
    >
      {children}
    </button>
  )
}

// 使用方式
<Tabs defaultTab="overview">
  <TabList>
    <Tab id="overview">概览</Tab>
    <Tab id="details">详情</Tab>
  </TabList>
</Tabs>
```

### Render Props 模式

```typescript
interface DataLoaderProps<T> {
  url: string
  children: (data: T | null, loading: boolean, error: Error | null) => React.ReactNode
}

export function DataLoader<T>({ url, children }: DataLoaderProps<T>) {
  const [data, setData] = useState<T | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false))
  }, [url])

  return <>{children(data, loading, error)}</>
}

// 使用方式
<DataLoader<Market[]> url="/api/markets">
  {(markets, loading, error) => {
    if (loading) return <Spinner />
    if (error) return <Error error={error} />
    return <MarketList markets={markets!} />
  }}
</DataLoader>
```

## 自定义 Hooks 模式

### 状态管理 Hook

```typescript
export function useToggle(initialValue = false): [boolean, () => void] {
  const [value, setValue] = useState<boolean>(initialValue)

  const toggle = useCallback((): void => {
    setValue(v => !v)
  }, [])

  return [value, toggle]
}

// 使用方式
const [isOpen, toggleOpen] = useToggle()
```

### 异步数据获取 Hook

```typescript
interface UseQueryOptions<T> {
  onSuccess?: (data: T) => void
  onError?: (error: Error) => void
  enabled?: boolean
}

interface UseQueryResult<T> {
  data: T | null
  error: Error | null
  loading: boolean
  refetch: () => Promise<void>
}

export function useQuery<T>(
  key: string,
  fetcher: () => Promise<T>,
  options?: UseQueryOptions<T>
): UseQueryResult<T> {
  const [data, setData] = useState<T | null>(null)
  const [error, setError] = useState<Error | null>(null)
  const [loading, setLoading] = useState<boolean>(false)

  const refetch = useCallback(async (): Promise<void> => {
    setLoading(true)
    setError(null)

    try {
      const result = await fetcher()
      setData(result)
      options?.onSuccess?.(result)
    } catch (err) {
      const error = err as Error
      setError(error)
      options?.onError?.(error)
    } finally {
      setLoading(false)
    }
  }, [fetcher, options])

  useEffect(() => {
    if (options?.enabled !== false) {
      refetch()
    }
  }, [key, refetch, options?.enabled])

  return { data, error, loading, refetch }
}

// 使用方式
const { data: markets, loading, error, refetch } = useQuery<Market[]>(
  'markets',
  () => fetch('/api/markets').then((r: Response) => r.json()),
  {
    onSuccess: (data: Market[]) => console.log('Fetched', data.length, 'markets'),
    onError: (err: Error) => console.error('Failed:', err)
  }
)
```

### Debounce Hook

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler: NodeJS.Timeout = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// 使用方式
const [searchQuery, setSearchQuery] = useState<string>('')
const debouncedQuery = useDebounce<string>(searchQuery, 500)

useEffect(() => {
  if (debouncedQuery) {
    performSearch(debouncedQuery)
  }
}, [debouncedQuery])
```

## 状态管理模式

### Context + Reducer 模式

```typescript
interface State {
  markets: Market[]
  selectedMarket: Market | null
  loading: boolean
}

type Action =
  | { type: 'SET_MARKETS'; payload: Market[] }
  | { type: 'SELECT_MARKET'; payload: Market }
  | { type: 'SET_LOADING'; payload: boolean }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'SET_MARKETS':
      return { ...state, markets: action.payload }
    case 'SELECT_MARKET':
      return { ...state, selectedMarket: action.payload }
    case 'SET_LOADING':
      return { ...state, loading: action.payload }
    default:
      return state
  }
}

const MarketContext = createContext<{
  state: State
  dispatch: Dispatch<Action>
} | undefined>(undefined)

export function MarketProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, {
    markets: [],
    selectedMarket: null,
    loading: false
  })

  return (
    <MarketContext.Provider value={{ state, dispatch }}>
      {children}
    </MarketContext.Provider>
  )
}

export function useMarkets() {
  const context = useContext(MarketContext)
  if (!context) throw new Error('useMarkets must be used within MarketProvider')
  return context
}
```

## 性能优化

### 记忆化

```typescript
// ✅ useMemo 用于昂贵计算
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume)
}, [markets])

// ✅ useCallback 用于传递给子组件的函数
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])

// ✅ React.memo 用于纯组件
export const MarketCard = React.memo<MarketCardProps>(({ market }) => {
  return (
    <div className="market-card">
      <h3>{market.name}</h3>
      <p>{market.description}</p>
    </div>
  )
})
```

### 代码分割与延迟载入

```typescript
import { lazy, Suspense } from 'react'

// ✅ 延迟载入重型组件
const HeavyChart = lazy(() => import('./HeavyChart'))
const ThreeJsBackground = lazy(() => import('./ThreeJsBackground'))

export function Dashboard() {
  return (
    <div>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>

      <Suspense fallback={null}>
        <ThreeJsBackground />
      </Suspense>
    </div>
  )
}
```

### 长列表虚拟化

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

export function VirtualMarketList({ markets }: { markets: Market[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: markets.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100,  // 预估行高
    overscan: 5  // 额外渲染的项目数
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative'
        }}
      >
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`
            }}
          >
            <MarketCard market={markets[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

## 表单处理模式

### 带验证的受控表单

```typescript
interface FormData {
  name: string
  description: string
  endDate: string
}

interface FormErrors {
  name?: string
  description?: string
  endDate?: string
}

export function CreateMarketForm() {
  const [formData, setFormData] = useState<FormData>({
    name: '',
    description: '',
    endDate: ''
  })

  const [errors, setErrors] = useState<FormErrors>({})

  const validate = (): boolean => {
    const newErrors: FormErrors = {}

    if (!formData.name.trim()) {
      newErrors.name = '名称为必填'
    } else if (formData.name.length > 200) {
      newErrors.name = '名称必须少于 200 个字符'
    }

    if (!formData.description.trim()) {
      newErrors.description = '描述为必填'
    }

    if (!formData.endDate) {
      newErrors.endDate = '结束日期为必填'
    }

    setErrors(newErrors)
    return Object.keys(newErrors).length === 0
  }

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    if (!validate()) return

    try {
      await createMarket(formData)
      // 成功处理
    } catch (error) {
      // 错误处理
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name}
        onChange={e => setFormData(prev => ({ ...prev, name: e.target.value }))}
        placeholder="市场名称"
      />
      {errors.name && <span className="error">{errors.name}</span>}

      {/* 其他字段 */}

      <button type="submit">建立市场</button>
    </form>
  )
}
```

## Error Boundary 模式

```typescript
interface ErrorBoundaryState {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error boundary caught:', error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>发生错误</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            重试
          </button>
        </div>
      )
    }

    return this.props.children
  }
}

// 使用方式
<ErrorBoundary>
  <App />
</ErrorBoundary>
```

## 动画模式

### Framer Motion 动画

```typescript
import { motion, AnimatePresence } from 'framer-motion'

// ✅ 列表动画
export function AnimatedMarketList({ markets }: { markets: Market[] }) {
  return (
    <AnimatePresence>
      {markets.map(market => (
        <motion.div
          key={market.id}
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -20 }}
          transition={{ duration: 0.3 }}
        >
          <MarketCard market={market} />
        </motion.div>
      ))}
    </AnimatePresence>
  )
}

// ✅ Modal 动画
export function Modal({ isOpen, onClose, children }: ModalProps) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            className="modal-overlay"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          <motion.div
            className="modal-content"
            initial={{ opacity: 0, scale: 0.9, y: 20 }}
            animate={{ opacity: 1, scale: 1, y: 0 }}
            exit={{ opacity: 0, scale: 0.9, y: 20 }}
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  )
}
```

## 无障碍模式

### 键盘导航

```typescript
export function Dropdown({ options, onSelect }: DropdownProps) {
  const [isOpen, setIsOpen] = useState(false)
  const [activeIndex, setActiveIndex] = useState(0)

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault()
        setActiveIndex(i => Math.min(i + 1, options.length - 1))
        break
      case 'ArrowUp':
        e.preventDefault()
        setActiveIndex(i => Math.max(i - 1, 0))
        break
      case 'Enter':
        e.preventDefault()
        onSelect(options[activeIndex])
        setIsOpen(false)
        break
      case 'Escape':
        setIsOpen(false)
        break
    }
  }

  return (
    <div
      role="combobox"
      aria-expanded={isOpen}
      aria-haspopup="listbox"
      onKeyDown={handleKeyDown}
    >
      {/* 下拉选单实现 */}
    </div>
  )
}
```

### 焦点管理

```typescript
export function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)
  const previousFocusRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      // 储存目前聚焦的元素
      previousFocusRef.current = document.activeElement as HTMLElement

      // 聚焦 modal
      modalRef.current?.focus()
    } else {
      // 关闭时恢复焦点
      previousFocusRef.current?.focus()
    }
  }, [isOpen])

  return isOpen ? (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      tabIndex={-1}
      onKeyDown={e => e.key === 'Escape' && onClose()}
    >
      {children}
    </div>
  ) : null
}
```

**记住**：现代前端模式能实现可维护、高效能的使用者介面。选择符合你项目复杂度的模式。

---

*注意：现代前端模式能实现可维护、高效能的使用者介面。选择符合你项目复杂度的模式。*

---

## Vanilla JavaScript 模式

> 以下为轻量级项目或老旧项目维护时的参考模式。现代项目建议使用 React/Vue 等框架。

### 模块模式（Module Pattern）

```javascript
// ✅ 使用 ES6 模块
// utils.js
export function formatDate(date) {
  return new Intl.DateTimeFormat('zh-TW').format(date);
}

export function debounce(func, wait) {
  let timeout;
  return function executedFunction(...args) {
    const later = () => {
      clearTimeout(timeout);
      func(...args);
    };
    clearTimeout(timeout);
    timeout = setTimeout(later, wait);
  };
}

// main.js
import { formatDate, debounce } from './utils.js';

document.addEventListener('DOMContentLoaded', () => {
  const dateElement = document.getElementById('date');
  dateElement.textContent = formatDate(new Date());
});
```

### 事件委托

```javascript
// ✅ 使用事件委托处理动态元素
class ListManager {
  constructor(containerId) {
    this.container = document.getElementById(containerId);
    this.items = [];

    // 单一事件监听器处理所有项目
    this.container.addEventListener('click', this.handleClick.bind(this));
  }

  handleClick(event) {
    const item = event.target.closest('.list-item');
    if (!item) return;

    const id = parseInt(item.dataset.id);

    if (event.target.classList.contains('delete-btn')) {
      this.deleteItem(id);
    } else if (event.target.classList.contains('edit-btn')) {
      this.editItem(id);
    }
  }

  addItem(text) {
    const item = { id: Date.now(), text };
    this.items.push(item);
    this.render();
  }

  deleteItem(id) {
    this.items = this.items.filter(item => item.id !== id);
    this.render();
  }

  render() {
    this.container.innerHTML = this.items.map(item => `
      <div class="list-item" data-id="${item.id}">
        <span>${item.text}</span>
        <button class="edit-btn">编辑</button>
        <button class="delete-btn">删除</button>
      </div>
    `).join('');
  }
}

// 使用方式
const listManager = new ListManager('list-container');
listManager.addItem('第一项');
listManager.addItem('第二项');
```

### 状态管理（观察者模式）

```javascript
// ✅ 简单的状态管理
class Store {
  constructor(initialState = {}) {
    this.state = initialState;
    this.listeners = new Set();
  }

  getState() {
    return { ...this.state };
  }

  setState(partialState) {
    this.state = { ...this.state, ...partialState };
    this.notify();
  }

  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  notify() {
    this.listeners.forEach(listener => listener(this.state));
  }
}

// 使用方式
const store = new Store({ count: 0, user: null });

// 订阅状态变化
store.subscribe((state) => {
  document.getElementById('count').textContent = state.count;
});

// 更新状态
document.getElementById('increment').addEventListener('click', () => {
  const current = store.getState().count;
  store.setState({ count: current + 1 });
});
```

### DOM 操作最佳实践

```javascript
// ✅ 使用 DocumentFragment 批量操作 DOM
function renderList(items) {
  const fragment = document.createDocumentFragment();

  items.forEach(item => {
    const li = document.createElement('li');
    li.textContent = item.text;
    li.className = 'item';
    fragment.appendChild(li);
  });

  document.getElementById('list').appendChild(fragment);
}

// ✅ 缓存 DOM 查询
class Component {
  constructor(elementId) {
    // 缓存 DOM 引用
    this.element = document.getElementById(elementId);
    this.titleElement = this.element.querySelector('.title');
    this.contentElement = this.element.querySelector('.content');
  }

  updateTitle(title) {
    this.titleElement.textContent = title;
  }

  updateContent(content) {
    this.contentElement.innerHTML = content;
  }
}
```

### Fetch API 封装

```javascript
// ✅ 封装 HTTP 请求
class HttpClient {
  constructor(baseURL = '', headers = {}) {
    this.baseURL = baseURL;
    this.headers = {
      'Content-Type': 'application/json',
      ...headers
    };
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const config = {
      ...options,
      headers: {
        ...this.headers,
        ...options.headers
      }
    };

    try {
      const response = await fetch(url, config);

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      return await response.json();
    } catch (error) {
      console.error('Request failed:', error);
      throw error;
    }
  }

  get(endpoint, options) {
    return this.request(endpoint, { ...options, method: 'GET' });
  }

  post(endpoint, data, options) {
    return this.request(endpoint, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  put(endpoint, data, options) {
    return this.request(endpoint, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }

  delete(endpoint, options) {
    return this.request(endpoint, { ...options, method: 'DELETE' });
  }
}

// 使用方式
const api = new HttpClient('/api');

async function loadMarkets() {
  try {
    const markets = await api.get('/markets');
    console.log('加载的市场:', markets);
  } catch (error) {
    console.error('加载失败:', error);
  }
}
```

### 本地存储封装

```javascript
// ✅ 本地存储工具
class Storage {
  constructor(prefix = 'app_') {
    this.prefix = prefix;
  }

  getKey(key) {
    return `${this.prefix}${key}`;
  }

  set(key, value) {
    const serialized = JSON.stringify(value);
    localStorage.setItem(this.getKey(key), serialized);
  }

  get(key, defaultValue = null) {
    const serialized = localStorage.getItem(this.getKey(key));
    if (serialized === null) return defaultValue;
    try {
      return JSON.parse(serialized);
    } catch {
      return serialized;
    }
  }

  remove(key) {
    localStorage.removeItem(this.getKey(key));
  }

  clear() {
    const keys = Object.keys(localStorage);
    keys.forEach(key => {
      if (key.startsWith(this.prefix)) {
        localStorage.removeItem(key);
      }
    });
  }
}

// 使用方式
const storage = new Storage();
storage.set('user', { name: '张三', age: 25 });
const user = storage.get('user');
```

---

## Vue.js 模式

### Composition API 基础

```vue
<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue';

// 响应式状态
const count = ref(0);
const name = ref('');

// 计算属性
const doubleCount = computed(() => count.value * 2);

const greeting = computed(() => {
  return name.value ? `你好，${name.value}！` : '请输入你的名字';
});

// 方法
function increment() {
  count.value++;
}

// 生命周期
onMounted(() => {
  console.log('组件已挂载');
});

// 监听器
watch(name, (newValue, oldValue) => {
  console.log(`名字从 ${oldValue} 变为 ${newValue}`);
});

// 监听多个源
watch([count, name], ([newCount, newName]) => {
  console.log('count 或 name 改变');
});
</script>

<template>
  <div>
    <p>{{ greeting }}</p>
    <input v-model="name" placeholder="输入名字" />
    <p>计数: {{ count }}</p>
    <p>双倍: {{ doubleCount }}</p>
    <button @click="increment">增加</button>
  </div>
</template>
```

### Composables（可组合函数）

```typescript
// composables/useToggle.ts
import { ref } from 'vue';

export function useToggle(initialValue = false) {
  const value = ref(initialValue);

  const toggle = () => {
    value.value = !value.value;
  };

  const setTrue = () => {
    value.value = true;
  };

  const setFalse = () => {
    value.value = false;
  };

  return {
    value,
    toggle,
    setTrue,
    setFalse
  };
}

// composables/useFetch.ts
import { ref, type Ref } from 'vue';

export function useFetch<T>(url: string) {
  const data: Ref<T | null> = ref(null);
  const error: Ref<Error | null> = ref(null);
  const loading = ref(false);

  async function fetch() {
    loading.value = true;
    error.value = null;

    try {
      const response = await fetch(url);
      data.value = await response.json();
    } catch (err) {
      error.value = err as Error;
    } finally {
      loading.value = false;
    }
  }

  return { data, error, loading, fetch };
}

// 使用方式
<script setup lang="ts">
import { useToggle } from '@/composables/useToggle';
import { useFetch } from '@/composables/useFetch';

const { value: isOpen, toggle } = useToggle();
const { data: markets, loading, fetch: loadMarkets } = useFetch<Market[]>('/api/markets');

loadMarkets();
</script>
```

### Provide/Inject 模式

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue';
import { type SymbolKey } from './symbols';

const theme = ref('light');
const user = ref(null);

// 提供全局状态
provide(SymbolKey.Theme, theme);
provide(SymbolKey.User, user);
</script>

<template>
  <router-view />
</template>

<!-- ChildComponent.vue -->
<script setup lang="ts">
import { inject } from 'vue';
import { SymbolKey } from './symbols';

const theme = inject(SymbolKey.Theme);
const user = inject(SymbolKey.User);

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light';
}
</script>

<template>
  <div :class="`theme-${theme}`">
    <button @click="toggleTheme">切换主题</button>
  </div>
</template>
```

### Pinia 状态管理

```typescript
// stores/market.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useMarketStore = defineStore('market', () => {
  // 状态
  const markets = ref<Market[]>([]);
  const selectedMarket = ref<Market | null>(null);
  const loading = ref(false);

  // 计算属性
  const activeMarkets = computed(() =>
    markets.value.filter(m => m.status === 'active')
  );

  const totalVolume = computed(() =>
    markets.value.reduce((sum, m) => sum + m.volume, 0)
  );

  // 操作
  async function fetchMarkets() {
    loading.value = true;
    try {
      const response = await fetch('/api/markets');
      markets.value = await response.json();
    } finally {
      loading.value = false;
    }
  }

  function selectMarket(market: Market) {
    selectedMarket.value = market;
  }

  function updateMarket(id: number, updates: Partial<Market>) {
    const index = markets.value.findIndex(m => m.id === id);
    if (index !== -1) {
      markets.value[index] = { ...markets.value[index], ...updates };
    }
  }

  return {
    // 状态
    markets,
    selectedMarket,
    loading,
    // 计算属性
    activeMarkets,
    totalVolume,
    // 操作
    fetchMarkets,
    selectMarket,
    updateMarket
  };
});

// 使用方式
<script setup lang="ts">
import { useMarketStore } from '@/stores/market';

const marketStore = useMarketStore();

marketStore.fetchMarkets();
</script>

<template>
  <div>
    <div v-if="marketStore.loading">加载中...</div>
    <div v-else>
      <div v-for="market in marketStore.activeMarkets" :key="market.id">
        {{ market.name }} - {{ market.volume }}
      </div>
    </div>
  </div>
</template>
```

### 自定义指令

```typescript
// directives/clickOutside.ts
import type { Directive } from 'vue';

export const clickOutside: Directive = {
  mounted(el, binding) {
    el._clickOutside = (event: MouseEvent) => {
      if (!(el === event.target || el.contains(event.target as Node))) {
        binding.value(event);
      }
    };
    document.addEventListener('click', el._clickOutside);
  },
  unmounted(el) {
    document.removeEventListener('click', el._clickOutside);
  }
};

// main.ts
import { clickOutside } from './directives/clickOutside';

app.directive('click-outside', clickOutside);

// 使用方式
<template>
  <div v-click-outside="closeDropdown">
    <p>点击外部关闭</p>
  </div>
</template>
```

### 插槽（Slots）模式

```vue
<!-- BaseCard.vue -->
<script setup lang="ts">
interface Props {
  variant?: 'default' | 'outlined' | 'elevated';
}

withDefaults(defineProps<Props>(), {
  variant: 'default'
});
</script>

<template>
  <div :class="['card', `card-${variant}`]">
    <!-- 具名插槽 -->
    <div v-if="$slots.header" class="card-header">
      <slot name="header" />
    </div>

    <div v-if="$slots.default" class="card-body">
      <slot />
    </div>

    <div v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </div>
  </div>
</template>

<!-- 使用方式 -->
<BaseCard variant="outlined">
  <template #header>
    <h2>标题</h2>
  </template>

  <p>卡片内容</p>

  <template #footer>
    <button>操作</button>
  </template>
</BaseCard>
```

---

## 高级性能优化

### React 性能优化

```typescript
// ✅ 使用 React.memo 避免不必要的重渲染
export const ProductCard = React.memo<ProductCardProps>(
  ({ product, onAddToCart }) => {
    return (
      <div className="product-card">
        <h3>{product.name}</h3>
        <p>{product.price}</p>
        <button onClick={() => onAddToCart(product.id)}>加入购物车</button>
      </div>
    );
  },
  (prevProps, nextProps) => {
    // 自定义比较函数
    return (
      prevProps.product.id === nextProps.product.id &&
      prevProps.product.price === nextProps.product.price
    );
  }
);

// ✅ 使用 useMemo 缓存昂贵计算
const filteredProducts = useMemo(() => {
  console.log('过滤产品...');
  return products.filter(p =>
    p.name.toLowerCase().includes(searchQuery.toLowerCase())
  );
}, [products, searchQuery]);

// ✅ 使用 useCallback 缓存函数
const handleAddToCart = useCallback((productId: number) => {
  dispatch(addToCart(productId));
}, [dispatch]);

// ✅ 使用 useTransition 标记非紧急更新
import { useTransition } from 'react';

function SearchPage() {
  const [isPending, startTransition] = useTransition();
  const [filter, setFilter] = useState('');

  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;

    // 紧急更新：输入框
    setFilter(value);

    // 非紧急更新：搜索结果
    startTransition(() => {
      performSearch(value);
    });
  };

  return (
    <div>
      <input value={filter} onChange={handleChange} />
      {isPending ? <Spinner /> : <SearchResults />}
    </div>
  );
}

// ✅ 使用 useDeferredValue 延迟更新
const deferredQuery = useDeferredValue(searchQuery);

const results = useMemo(() => {
  return searchProducts(deferredQuery);
}, [deferredQuery]);
```

### Vue 性能优化

```vue
<script setup lang="ts">
import { defineAsyncComponent, shallowRef, markRaw } from 'vue';

// ✅ 异步组件
const HeavyChart = defineAsyncComponent(() =>
  import('./HeavyChart.vue')
);

// ✅ 使用 shallowRef 减少深度响应式开销
const largeData = shallowRef<LargeObject[]>([]);

// ✅ 使用 markRaw 防止对象被转换为响应式
const staticConfig = markRaw({
  // 大型静态配置
});

// ✅ v-once 只渲染一次
<template>
  <div v-once>
    <p>{{ staticTitle }}</p>
  </div>
</template>

// ✅ v-memo 条件记忆
<template>
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id, item.selected]"
  >
    {{ item.name }}
  </div>
</template>
</script>
```

### 图片优化

```typescript
// ✅ 响应式图片
<img
  src="image-800.jpg"
  srcSet="image-400.jpg 400w,
          image-800.jpg 800w,
          image-1200.jpg 1200w"
  sizes="(max-width: 600px) 400px,
         (max-width: 1200px) 800px,
         1200px"
  alt="描述"
  loading="lazy"
/>

// ✅ Next.js Image 组件
import Image from 'next/image';

<Image
  src="/market.jpg"
  alt="市场图片"
  width={800}
  height={600}
  priority={false}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
/>

// ✅ Vue 图片懒加载
<img
  v-lazy="imageUrl"
  :alt="imageAlt"
/>
```

### 代码分割策略

```typescript
// ✅ 路由级代码分割
const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}

// ✅ 组件级代码分割
const Chart = lazy(() =>
  import('./Chart').then(module => ({
    default: module.Chart
  }))
);

// ✅ 条件加载
function AdminPanel() {
  const [ChartComponent, setChartComponent] = useState(null);

  useEffect(() => {
    if (user.isAdmin) {
      import('./AdminChart').then(module => {
        setChartComponent(() => module.default);
      });
    }
  }, [user.isAdmin]);

  return ChartComponent ? <ChartComponent /> : null;
}
```

### 服务端渲染（SSR）优化

```typescript
// ✅ Next.js ISR（增量静态再生）
export async function getStaticProps() {
  const markets = await fetchMarkets();

  return {
    props: { markets },
    revalidate: 60 // 每 60 秒重新生成页面
  };
}

// ✅ Next.js SSR（服务端渲染）
export async function getServerSideProps() {
  const markets = await fetchMarkets();

  return {
    props: { markets }
  };
}

// ✅ Nuxt.js 数据获取
<script setup lang="ts">
const { data: markets } = await useFetch('/api/markets', {
  key: 'markets',
  // 缓存策略
  getCachedData: (key) => useNuxtData(key).data
});
</script>
```

### Web Worker 使用

```javascript
// ✅ 使用 Web Worker 处理密集计算
// worker.js
self.onmessage = function(e) {
  const { data } = e;
  const result = performHeavyCalculation(data);
  self.postMessage(result);
};

// main.js
const worker = new Worker('worker.js');

worker.onmessage = function(e) {
  const result = e.data;
  updateUI(result);
};

worker.postMessage(largeDataSet);

// ✅ React Hook 封装
function useWorker<T, R>(
  workerFn: (data: T) => R,
  dependencies: any[] = []
) {
  const [result, setResult] = useState<R | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState(false);

  const workerRef = useRef<Worker | null>(null);

  useEffect(() => {
    workerRef.current = new Worker(
      URL.createObjectURL(new Blob([`(${workerFn.toString()})`], {
        type: 'text/javascript'
      }))
    );

    workerRef.current.onmessage = (e) => {
      setResult(e.data);
      setLoading(false);
    };

    workerRef.current.onerror = (e) => {
      setError(new Error(e.message));
      setLoading(false);
    };

    return () => {
      workerRef.current?.terminate();
    };
  }, dependencies);

  const execute = useCallback((data: T) => {
    setLoading(true);
    setError(null);
    workerRef.current?.postMessage(data);
  }, []);

  return { result, error, loading, execute };
}
```

---

## 性能监控

### Core Web Vitals

```typescript
// ✅ CLS（累积布局偏移）监控
let clsValue = 0;
let clsEntries: any[] = [];

new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      clsValue += entry.value;
      clsEntries.push(entry);
    }
  }
  console.log('CLS:', clsValue);
}).observe({ entryTypes: ['layout-shift'] });

// ✅ FCP（首次内容绘制）监控
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('FCP:', entry.startTime);
  }
}).observe({ entryTypes: ['paint'] });

// ✅ LCP（最大内容绘制）监控
new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  console.log('LCP:', lastEntry.startTime);
}).observe({ entryTypes: ['largest-contentful-paint'] });

// ✅ FID（首次输入延迟）监控
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('FID:', entry.processingStart - entry.startTime);
  }
}).observe({ entryTypes: ['first-input'] });
```

### 自定义性能追踪

```typescript
// ✅ React 性能追踪
import { Profiler, ProfilerOnRenderCallback } from 'react';

const onRenderCallback: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  console.log('Component:', id);
  console.log('Phase:', phase);
  console.log('Duration:', actualDuration);
  console.log('Base Duration:', baseDuration);
};

<Profiler id="MarketList" onRender={onRenderCallback}>
  <MarketList />
</Profiler>

// ✅ Vue 性能追踪
// main.ts
app.config.performance = true;

// 在 Chrome DevTools 中查看 Performance 标签

// ✅ 自定义性能标记
function measurePerformance(name: string, fn: () => void) {
  performance.mark(`${name}-start`);
  fn();
  performance.mark(`${name}-end`);
  performance.measure(name, `${name}-start`, `${name}-end`);

  const measure = performance.getEntriesByName(name)[0];
  console.log(`${name}: ${measure.duration}ms`);

  performance.clearMarks();
  performance.clearMeasures();
}
```

---

## 相关技能

- `coding-standards` - 通用编码标准
- `verification-loop` - 项目验证流程
