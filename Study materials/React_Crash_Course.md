# React Crash Course — Zero to Interview-Ready

> You've used ReactJS at Rippling — this builds on that. Focus is on hooks, patterns, and TypeScript integration.

---

## Level 0: The Core Mental Model

React is a **component tree** where data flows **one way: down** (parent to child via props). When state changes, React re-renders the affected component and its children.

```
App
├── Navbar
├── Dashboard
│   ├── CompanyList
│   │   ├── CompanyCard
│   │   └── CompanyCard
│   └── EvidencePanel
│       └── EvidenceItem
└── Footer
```

**Key rules:**
1. Components are just functions that return JSX
2. State is private to a component — share it by lifting up or using context
3. Don't mutate state directly — always use the setter function
4. Props are read-only

---

## Level 1: Components and JSX

### Functional component (the only kind used today)
```tsx
// Basic component
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>
}

// Arrow function style (same thing)
const Greeting = ({ name }: { name: string }) => <h1>Hello, {name}</h1>

// With typed props using interface
interface Props {
  name: string
  role?: string
  onClick: () => void
}

function UserCard({ name, role = 'member', onClick }: Props) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{role}</p>
      <button onClick={onClick}>View</button>
    </div>
  )
}
```

### JSX rules
```tsx
// 1. Must return a single root element — wrap with <> if needed
function Component() {
  return (
    <>
      <h1>Title</h1>
      <p>Subtitle</p>
    </>
  )
}

// 2. className not class (class is reserved in JS)
<div className="container">

// 3. JavaScript in JSX — use curly braces
<p>{user.name}</p>
<p>{isAdmin ? 'Admin' : 'Member'}</p>

// 4. Lists — always need a key
{users.map(user => (
  <UserCard key={user.id} name={user.name} />
))}

// 5. Conditional rendering
{isLoading && <Spinner />}
{error && <ErrorMessage message={error} />}
{user ? <UserCard user={user} /> : <LoginPrompt />}
```

---

## Level 2: Hooks (the most important thing to know)

### useState — local component state
```tsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)         // initial value = 0
  const [user, setUser] = useState<User | null>(null)  // typed with TS

  // Updating state
  setCount(5)                      // set to value
  setCount(prev => prev + 1)       // use previous value (safer for async)

  // Updating object state — must spread, not mutate
  setUser(prev => ({ ...prev!, role: 'admin' }))  // NOT: prev.role = 'admin'

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  )
}
```

### useEffect — side effects (fetch data, subscriptions, timers)
```tsx
import { useEffect, useState } from 'react'

function EvidenceList({ companyId }: { companyId: number }) {
  const [evidence, setEvidence] = useState<Evidence[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // This runs after the component renders
    async function fetchEvidence() {
      setLoading(true)
      const res = await fetch(`/api/companies/${companyId}/evidence`)
      const data = await res.json()
      setEvidence(data.evidence)
      setLoading(false)
    }

    fetchEvidence()
  }, [companyId])  // <-- dependency array

  if (loading) return <p>Loading...</p>
  return <ul>{evidence.map(e => <li key={e.id}>{e.status}</li>)}</ul>
}
```

### Dependency array — the most common source of bugs
```tsx
useEffect(() => { ... })           // runs after EVERY render — almost never what you want
useEffect(() => { ... }, [])       // runs ONCE after initial render (like componentDidMount)
useEffect(() => { ... }, [id])     // runs when `id` changes
useEffect(() => { ... }, [a, b])   // runs when `a` OR `b` changes
```

**Rules:**
- Include everything the effect uses from the component scope
- If you're fetching based on an ID, include that ID in deps
- Empty `[]` = "set up once and never re-run" (data fetching on mount, subscriptions)

### Cleanup in useEffect
```tsx
useEffect(() => {
  const interval = setInterval(() => {
    refreshEvidence()
  }, 5000)

  // Return cleanup function — runs before next effect or unmount
  return () => clearInterval(interval)
}, [])

// Same pattern for event listeners, WebSockets, subscriptions
useEffect(() => {
  const ws = new WebSocket('ws://...')
  ws.onmessage = (event) => setAlerts(prev => [...prev, event.data])

  return () => ws.close()
}, [])
```

### useRef — persist a value without triggering re-render
```tsx
import { useRef } from 'react'

// 1. Access DOM element
function SearchInput() {
  const inputRef = useRef<HTMLInputElement>(null)

  const focusInput = () => inputRef.current?.focus()

  return <input ref={inputRef} type="text" />
}

// 2. Store a mutable value that doesn't trigger re-render
function Timer() {
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null)

  const start = () => {
    intervalRef.current = setInterval(() => tick(), 1000)
  }

  const stop = () => {
    if (intervalRef.current) clearInterval(intervalRef.current)
  }
}
```

### useCallback — memoize a function
```tsx
import { useCallback } from 'react'

function Parent({ companyId }: { companyId: number }) {
  // Without useCallback: new function reference on every render
  // -> child re-renders every time parent re-renders (even if id didn't change)
  const handleDelete = useCallback(async (evidenceId: number) => {
    await fetch(`/api/evidence/${evidenceId}`, { method: 'DELETE' })
  }, [])  // no deps — function never changes

  return <EvidenceList onDelete={handleDelete} />
}
```

**When to use:** When passing a function to a child that's wrapped in `React.memo`, or as a dep in another hook's dependency array.

### useMemo — memoize a computed value
```tsx
import { useMemo } from 'react'

function ComplianceDashboard({ evidence }: { evidence: Evidence[] }) {
  // Recomputes ONLY when evidence changes — not on every render
  const stats = useMemo(() => ({
    total: evidence.length,
    passing: evidence.filter(e => e.status === 'passing').length,
    failing: evidence.filter(e => e.status === 'failing').length,
    passRate: evidence.filter(e => e.status === 'passing').length / evidence.length * 100
  }), [evidence])

  return <div>Pass rate: {stats.passRate.toFixed(1)}%</div>
}
```

**When to use:** Expensive calculations, filtering/sorting large lists. Don't use for simple values — the memoization overhead isn't worth it.

---

## Level 3: Data Fetching Patterns

### Basic pattern (useState + useEffect)
```tsx
function useEvidence(companyId: number) {
  const [data, setData] = useState<Evidence[] | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    let cancelled = false  // prevent state update after unmount

    async function fetch() {
      setLoading(true)
      setError(null)
      try {
        const res = await fetchEvidence(companyId)
        if (!cancelled) setData(res)
      } catch (e) {
        if (!cancelled) setError('Failed to load evidence')
      } finally {
        if (!cancelled) setLoading(false)
      }
    }

    fetch()
    return () => { cancelled = true }
  }, [companyId])

  return { data, loading, error }
}

// Usage
function EvidencePanel({ companyId }: { companyId: number }) {
  const { data, loading, error } = useEvidence(companyId)

  if (loading) return <Spinner />
  if (error) return <ErrorMessage message={error} />
  return <EvidenceList evidence={data!} />
}
```

### Custom hooks — extract reusable logic
```tsx
// Any function starting with "use" that calls other hooks
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key)
    return stored ? JSON.parse(stored) : initialValue
  })

  const setStoredValue = (newValue: T) => {
    setValue(newValue)
    localStorage.setItem(key, JSON.stringify(newValue))
  }

  return [value, setStoredValue] as const
}

// Usage
const [theme, setTheme] = useLocalStorage('theme', 'light')
```

---

## Level 4: State Management Patterns

### Lifting state up — share state between siblings
```tsx
// Both CompanyList and EvidencePanel need selectedCompany
// -> lift state up to their common parent

function Dashboard() {
  const [selectedCompanyId, setSelectedCompanyId] = useState<number | null>(null)

  return (
    <div>
      <CompanyList
        onSelect={(id) => setSelectedCompanyId(id)}
        selectedId={selectedCompanyId}
      />
      {selectedCompanyId && (
        <EvidencePanel companyId={selectedCompanyId} />
      )}
    </div>
  )
}
```

### Context — avoid prop drilling for global state
```tsx
import { createContext, useContext, useState } from 'react'

interface AuthContextType {
  user: User | null
  login: (user: User) => void
  logout: () => void
}

const AuthContext = createContext<AuthContextType | null>(null)

// Provider — wrap your app with this
export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null)

  return (
    <AuthContext.Provider value={{
      user,
      login: (u) => setUser(u),
      logout: () => setUser(null)
    }}>
      {children}
    </AuthContext.Provider>
  )
}

// Custom hook to use context (safer than raw useContext)
export function useAuth() {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth must be used inside AuthProvider')
  return ctx
}

// Usage anywhere in the tree
function NavBar() {
  const { user, logout } = useAuth()
  return <button onClick={logout}>{user?.name}</button>
}
```

---

## Level 5: Controlled vs Uncontrolled Components

### Controlled — React owns the value
```tsx
function SearchBar() {
  const [query, setQuery] = useState('')

  return (
    <input
      value={query}                          // value controlled by state
      onChange={e => setQuery(e.target.value)}
      placeholder="Search..."
    />
  )
}
```

### Uncontrolled — DOM owns the value (use useRef)
```tsx
function LoginForm() {
  const emailRef = useRef<HTMLInputElement>(null)

  const handleSubmit = () => {
    const email = emailRef.current?.value  // read value on submit
  }

  return <input ref={emailRef} type="email" />
}
```

**Use controlled when:** You need to validate, transform, or react to every keystroke.
**Use uncontrolled when:** You only need the value on submit (simpler forms).

---

## Level 6: Performance Patterns

### React.memo — skip re-render if props haven't changed
```tsx
// Without memo: re-renders every time parent re-renders
// With memo: only re-renders when its props change

const EvidenceItem = React.memo(function EvidenceItem({ evidence }: { evidence: Evidence }) {
  return <div>{evidence.status}</div>
})
```

### Key prop — helps React identify list items
```tsx
// BAD: using index as key — causes bugs when list reorders
{evidence.map((e, index) => <EvidenceItem key={index} evidence={e} />)}

// GOOD: use stable, unique ID
{evidence.map(e => <EvidenceItem key={e.id} evidence={e} />)}
```

### Lazy loading — split bundle, load on demand
```tsx
import { lazy, Suspense } from 'react'

const Dashboard = lazy(() => import('./Dashboard'))

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Dashboard />
    </Suspense>
  )
}
```

---

## Common Patterns at Delve's Level

### Error boundary (class component — only place you still need classes)
```tsx
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  render() {
    if (this.state.hasError) return this.props.fallback
    return this.props.children
  }
}

// Usage
<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <EvidencePanel />
</ErrorBoundary>
```

### Optimistic updates — update UI before server confirms
```tsx
function EvidenceList({ evidence, onStatusChange }: Props) {
  const handleMarkPassing = async (id: number) => {
    // 1. Update UI immediately (optimistic)
    setEvidence(prev => prev.map(e => e.id === id ? { ...e, status: 'passing' } : e))

    try {
      // 2. Send to server
      await updateEvidence(id, { status: 'passing' })
    } catch {
      // 3. Revert if server fails
      setEvidence(prev => prev.map(e => e.id === id ? { ...e, status: 'failing' } : e))
    }
  }
}
```

---

## Interview Tips

1. **Always handle loading and error states** — "I'd also track loading and error state for this fetch."
2. **Explain the dependency array** if you write useEffect — interviewers love asking about it.
3. **Mention useCallback when passing functions to children** — shows you understand React re-renders.
4. **`useMemo` and `useCallback` are optimizations** — don't use them by default, only when there's a measurable perf issue.
5. **Context is not a state manager** — for complex state, mention you'd use Redux or Zustand.
6. **Bridge your Rippling experience** — "At Rippling I worked with React — this is the same model, just with TypeScript types on props and state."
