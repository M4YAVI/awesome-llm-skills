# React 19 Rendering & Performance - AI Skill Guide

## 1. RECONCILIATION (THE DIFF ALGORITHM)

### 1.1 Core Rules
React uses a **fiber-based reconciliation engine** to diff and update UI efficiently.

**Key Principles:**
- Components with same key/type at same position → Update
- Different type/key → Unmount old, mount new
- Lists without keys → Position-based matching (dangerous)

**Reconciliation Phases:**
1. **Render Phase**: Build new fiber tree (pausable, no side effects)
2. **Commit Phase**: Apply to DOM (unpausable, side effects happen here)

### 1.2 When Reconciliation Happens
```js
// Reconciliation triggered by:
1. State change (setState, hooks)
2. Props change (parent re-render)
3. Context value change
4. Force re-render (deprecated)

// NOT triggered by:
- Event handler definitions (same reference)
- Computed values that aren't memoized
- Parent component didn't actually re-render

1.3 The Diff Algorithm (Simplified)

plaintext

Old: <div><Button /><Input /></div>
New: <div><Input /><Button /></div>
❌ Without keys: Both components unmount/remount (loses state, refocus)
✅ With keys: Components swap position (preserves state)
<div>
  <Button key="btn" />     <Button key="btn" />
  <Input key="inp" />  →   <Input key="inp" />
</div>
Correct:
<div>
  <Input key="inp" />
  <Button key="btn" />
</div>

Critical Rule: Keys must be:

Unique per sibling
Stable (same for same item across renders)
❌ NOT index if list can reorder/filter/add items
1.4 Type Changes Cause Unmount

js

// ❌ WRONG: Component type changes = full unmount
function MyComponent({type}) {
  if (type === 'client') return <ClientComponent />;
  return <ServerComponent />;
}

// When type toggles, both components fully unmount/remount
// State is lost, refs reset, DOM recreated

// ✅ CORRECT: Both render, one display:none
function MyComponent({type}) {
  return (
    <>
      <ClientComponent style={{display: type === 'client' ? 'block' : 'none'}} />
      <ServerComponent style={{display: type === 'server' ? 'block' : 'none'}} />
    </>
  );
}
// OR use Suspense + lazy

2. MEMOIZATION STRATEGY

2.1 When to Memoize: Decision Tree

plaintext

Does component receive new objects/arrays as props EVERY render?
  ↓ YES
Does component do expensive rendering (1ms+)?
  ↓ YES → Memoize with React.memo()
  ↓ NO → Skip memoization (overhead > benefit)
Does parent re-render frequently?
  ↓ YES → Consider memoization
  ↓ NO → Skip memoization
Is component slow on DevTools Profiler?
  ↓ YES → Memoize
  ↓ NO → Don't memoize (premature optimization)

2.2 React.memo() - When to Use

js

// ✅ MEMOIZE: Heavy computation, re-renders often
const HeavyList = React.memo(({items}) => {
  return items.map(item => (
    <ExpensiveItemComponent key={item.id} item={item} />
  ));
});

// ❌ DON'T MEMOIZE: Lightweight, receives inline objects
const SimpleDisplay = React.memo(({data}) => {
  return <div>{data.name}</div>;
});
// Parent passes: <SimpleDisplay data={{name: 'foo'}} />
// Props always different (new object), memo does nothing

Memo Cost: Adds prop comparison overhead

Only worth if skipping 1ms+ of render
Shallow comparison: O(n) where n = prop count
Don't memoize if props are primitives only
2.3 useMemo - When to Use

js

// ❌ DON'T use useMemo for primitives
function Component({a, b}) {
  const sum = useMemo(() => a + b, [a, b]); // No benefit!
  return <div>{sum}</div>;
}

// ✅ USE useMemo for:
// 1. Expensive computations (sorting, filtering)
const filtered = useMemo(() => {
  return expensiveSort(items, filter); // 10ms+
}, [items, filter]);

// 2. Stable object references (prevent child re-renders)
const config = useMemo(() => ({
  apiUrl: API_URL,
  timeout: 5000,
}), []); // Never changes

// 3. Avoiding cascading re-renders
const memorizedValue = useMemo(() => {
  return transformData(data); // Used by memoized child
}, [data]);

useMemo Cost:

Comparison hook: deps.length * 4 bytes
Function call overhead: ~0.1ms
Only worth if computation > 1ms or prevents child re-render
2.4 useCallback - When to Use

js

// ❌ DON'T use useCallback unnecessarily
function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []); // Callback memoization overhead > benefit
  
  return <Child onClick={handleClick} />;
}

// ✅ USE useCallback when:
// 1. Callback passed to memoized child that compares it
const handleSubmit = useCallback(async (data) => {
  await api.save(data);
}, [api]); // Dependency: api client

return <Form onSubmit={handleSubmit} />; // Form is memoized

// 2. Callback in dependency array
const handleRender = useCallback(() => {
  setData(prevData => prevData + 1);
}, []);

useEffect(() => {
  window.addEventListener('click', handleRender);
  return () => window.removeEventListener('click', handleRender);
}, [handleRender]); // Dependency

// 3. Callback used as Map/Set key
const callbackMap = useMemo(() => new Map([
  [callback1, result1],
  [callback2, result2],
]), [callback1, callback2]);

Best Practice: Use useCallback sparingly. It's often misused.

2.5 React 19: Automatic Memoization in Strict Mode

js

// React 19 Strict Mode: useMemo and useCallback
// now reuse the memoized result from first render,
// during the second render (double-invoke test)

function Component() {
  const value = useMemo(() => {
    console.log('computing'); // Logs ONCE, not twice
    return expensive();
  }, []);
}
// In Strict Mode: value is memoized across both invocations

3. AVOIDING UNNECESSARY RE-RENDERS

3.1 Root Causes of Re-renders

js

// 1. STATE CHANGE
function Component() {
  const [count, setCount] = useState(0);
  return <>
    <div>{count}</div>
    <Child /> {/* Re-renders because parent re-rendered */}
  </>;
}

// 2. PROPS CHANGE
function Component({user}) {
  return <div>{user.name}</div>;
  // Re-renders whenever parent re-renders (props shallowly compared)
}

// 3. CONTEXT CHANGE
const ThemeContext = createContext();
function Component() {
  const theme = useContext(ThemeContext);
  return <div style={{color: theme.color}}>Text</div>;
  // Re-renders whenever ThemeContext value changes
}

// 4. PARENT RE-RENDER (cascading)
function Parent() {
  return <>
    <Parent.Child /> {/* Always re-renders when Parent re-renders */}
  </>;
}

3.2 Prevention Strategies

Strategy 1: Memoize Child Component

js

// ❌ WRONG: Child re-renders with parent
function Parent() {
  const [count, setCount] = useState(0);
  return <>
    <button onClick={() => setCount(count + 1)}>Count: {count}</button>
    <Child /> {/* Re-renders every parent render */}
  </>;
}

// ✅ CORRECT: Memoize child if props unchanged
const Child = React.memo(() => {
  console.log('Child rendered'); // Logs once on mount
  return <div>Expensive child</div>;
});

// Or pass children as props (prevents re-render):
function Parent() {
  const [count, setCount] = useState(0);
  return <>
    <button onClick={() => setCount(count + 1)}>Count: {count}</button>
    {children} {/* Only re-renders if parent's children prop changes */}
  </>;
}

Strategy 2: State Colocation (Move State Down)

js

// ❌ WRONG: Counter state at top, causes full tree re-render
export default function Page() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <Counter count={count} setCount={setCount} />
      <ExpensiveList /> {/* Re-renders when count changes! */}
      <ExpensiveSidebar /> {/* Re-renders when count changes! */}
    </div>
  );
}

// ✅ CORRECT: Move state to Counter component (colocation)
export default function Page() {
  return (
    <div>
      <Counter /> {/* State isolated here */}
      <ExpensiveList /> {/* Does NOT re-render */}
      <ExpensiveSidebar /> {/* Does NOT re-render */}
    </div>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}

Strategy 3: Pass Children as Props (Composition)

js

// ❌ WRONG: Defining JSX inside render causes re-mount
function Parent() {
  const [toggle, setToggle] = useState(false);
  
  // ❌ This component remounts every parent render
  function Child() {
    const [childCount, setChildCount] = useState(0);
    return <div>{childCount}</div>;
  }
  
  return <>
    <button onClick={() => setToggle(!toggle)}>Toggle</button>
    <Child /> {/* Remounts on parent re-render, loses state */}
  </>;
}

// ✅ CORRECT: Define component outside or pass as children
const Child = React.memo(() => {
  const [childCount, setChildCount] = useState(0);
  return <div>{childCount}</div>;
});

function Parent({children}) {
  const [toggle, setToggle] = useState(false);
  return <>
    <button onClick={() => setToggle(!toggle)}>Toggle</button>
    {children} {/* Doesn't re-render, only parent wrapper */}
  </>;
}

3.3 Anti-patterns to Avoid



Anti-pattern	Problem	Solution
New object props	style={{}} or config={{}} every render	Use const or useMemo
New function props	onClick={() => {}} inline	Use useCallback
Inline components	<div>{(() => <C />)()}</div>	Extract component
Large context	Context wraps entire app	Split contexts by concern
Array as deps	useEffect(..., [items]) with inline array	Reference the array, not copy
4. COMPONENT GRANULARITY

4.1 Finding Right Component Size

plaintext

TOO LARGE:
- 500+ lines
- Multiple concerns (fetch + render + mutations)
- Hard to memoize (too many props)
- Slow on profiler
TOO SMALL:
- Only renders HTML, no logic
- Created dynamically inside parent
- Prop drilling 3+ levels
✅ OPTIMAL:
- 50-150 lines
- Single responsibility
- Memoizable (limited props)
- Clear input/output

4.2 Granularity Strategy

js

// ❌ WRONG: Monolithic component
function UserProfile({userId, isAdmin}) {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  const [theme, setTheme] = useState('light');
  
  useEffect(() => { /* fetch user */ }, [userId]);
  useEffect(() => { /* fetch posts */ }, [userId]);
  useEffect(() => { /* fetch comments */ }, [userId]);
  
  return (
    <div>
      <header>{/* user info */}</header>
      <aside>{/* theme selector */}</aside>
      <section>{/* posts grid */}</section>
      <footer>{/* comments */}</footer>
    </div>
  );
}

// ✅ CORRECT: Decomposed, memoizable
function UserProfile({userId, isAdmin}) {
  return (
    <div>
      <UserHeader userId={userId} />
      <ThemeSelector />
      <PostsGrid userId={userId} />
      <CommentsFeed userId={userId} />
    </div>
  );
}

// Each component independently memoizable
const UserHeader = React.memo(({userId}) => {
  const [user] = useUser(userId);
  return <header>{user?.name}</header>;
});

const ThemeSelector = React.memo(() => {
  const [theme, setTheme] = useTheme();
  return <aside>Theme: {theme}</aside>;
});

const PostsGrid = React.memo(({userId}) => {
  const [posts] = usePosts(userId);
  return <section>{posts.map(p => <Post key={p.id} {...p} />)}</section>;
});

const CommentsFeed = React.memo(({userId}) => {
  const [comments] = useComments(userId);
  return <footer>{comments.map(c => <Comment key={c.id} {...c} />)}</footer>;
});

4.3 Splitting by Concerns

js

// ✅ Pattern: One component = one job

// 1. Container (logic, state)
function TodoListContainer() {
  const [todos, setTodos] = useState([]);
  const [filter, setFilter] = useState('all');
  
  const filtered = useMemo(() => {
    return todos.filter(t => {
      if (filter === 'done') return t.completed;
      if (filter === 'pending') return !t.completed;
      return true;
    });
  }, [todos, filter]);
  
  return <TodoListView todos={filtered} filter={filter} onFilterChange={setFilter} />;
}

// 2. Presenter (render only, memoizable)
const TodoListView = React.memo(({todos, filter, onFilterChange}) => (
  <div>
    <FilterButtons filter={filter} onChange={onFilterChange} />
    <ul>
      {todos.map(t => <TodoItem key={t.id} todo={t} />)}
    </ul>
  </div>
));

5. STATE COLOCATION

5.1 Principle: Store State Where It's Used

js

// ❌ WRONG: State at root, passed down
function App() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  
  return (
    <Header />
    <Sidebar />
    <Main isModalOpen={isModalOpen} setIsModalOpen={setIsModalOpen} />
    {/* All sibling components re-render when modal opens/closes */}
  );
}

// ✅ CORRECT: State in component that uses it
function Main() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  
  return (
    <div>
      <Button onClick={() => setIsModalOpen(true)}>Open Modal</Button>
      {isModalOpen && <Modal onClose={() => setIsModalOpen(false)} />}
    </div>
  );
}
// Only Main re-renders, not Header or Sidebar

5.2 Colocation Patterns

js

// Pattern 1: Use custom hook for related state
function useModal(initialState = false) {
  const [isOpen, setIsOpen] = useState(initialState);
  return {
    isOpen,
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
    toggle: () => setIsOpen(p => !p),
  };
}

function Modal() {
  const modal = useModal();
  
  return (
    <>
      <button onClick={modal.open}>Open</button>
      {modal.isOpen && <div>{/* modal content */}</div>}
    </>
  );
}

// Pattern 2: Move state to smallest common ancestor
function TodoApp() {
  const [todos, setTodos] = useState([]);
  
  return (
    <div>
      <TodoInput onAdd={t => setTodos([...todos, t])} />
      <TodoList todos={todos} onChange={setTodos} />
    </div>
  );
}
// Only TodoApp re-renders when todos change, not siblings

// Pattern 3: Use context only for deeply nested shared state
const PermissionsContext = createContext();

function PermissionProvider({children}) {
  const [permissions, setPermissions] = useState(null);
  useEffect(() => { /* fetch perms */ }, []);
  
  return (
    <PermissionsContext.Provider value={permissions}>
      {children}
    </PermissionsContext.Provider>
  );
}
// Only use Context when prop drilling 4+ levels

6. RENDER-AS-YOU-FETCH PATTERN

6.1 Anti-pattern: Fetch-on-Mount

js

// ❌ WRONG: Fetch in useEffect
function UserDetail({userId}) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser)
      .finally(() => setLoading(false));
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}

// PROBLEM:
// 1. Component mounts
// 2. Browser renders loading state
// 3. useEffect runs (waterfall)
// 4. Data fetches
// 5. Browser updates

6.2 Correct: Render-as-You-Fetch (Server Component or Suspense)

js

// ✅ CORRECT: Server Components (React 19)
async function UserDetail({userId}) {
  const user = await fetch(`/api/users/${userId}`).then(r => r.json());
  return <div>{user.name}</div>;
}

// ✅ CORRECT: Suspense + lazy fetch
function UserDetail({userId}) {
  const userPromise = useMemo(() => {
    return fetch(`/api/users/${userId}`).then(r => r.json());
  }, [userId]);
  
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserContent userPromise={userPromise} />
    </Suspense>
  );
}

function UserContent({userPromise}) {
  const user = use(userPromise); // Wait for promise
  return <div>{user.name}</div>;
}

// BENEFIT:
// 1. Fetch starts immediately (not in useEffect)
// 2. Parent renders, sees promise
// 3. Fallback shown while fetching
// 4. No waterfall, parallel fetches

6.3 Implementation

js

// Pattern: Render-as-you-fetch with startTransition
function Router({path}) {
  const [component, setComponent] = useState(null);
  
  useEffect(() => {
    startTransition(() => {
      const promise = import(`./${path}`).then(m => m.default);
      setComponent(() => promise); // Render promise
    });
  }, [path]);
  
  return (
    <Suspense fallback={<div>Loading page...</div>}>
      {component && <component />}
    </Suspense>
  );
}

7. AVOIDING PROP DRILLING WITHOUT ABUSING CONTEXT

7.1 Prop Drilling vs Context Trade-off



Depth	Solution	Reason
1-2 levels	Props	Direct, explicit, fast
3-4 levels	Props	Still manageable
5+ levels	Context	Avoid intermediate pass-through
Frequently changing	Props	Context updates all consumers
Rarely changing	Context	Set-once, many consumers
7.2 Anti-pattern: Over-contextualization

js

// ❌ WRONG: Context for everything
const AppContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState([]);
  const [modals, setModals] = useState({});
  
  return (
    <AppContext.Provider value={{
      user, setUser,
      theme, setTheme,
      notifications, setNotifications,
      modals, setModals,
    }}>
      <App />
    </AppContext.Provider>
  );
}

// PROBLEM:
// 1. Any consumer re-renders when ANY value changes
// 2. Entire app re-renders on notification change
// 3. Hard to optimize (can't memoize)
// 4. Tightly coupled

7.3 Correct: Split Contexts by Concern

js

// ✅ CORRECT: Separate contexts
const UserContext = createContext();
const ThemeContext = createContext();
const NotificationContext = createContext();

function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <NotificationProvider>
          <Routes />
        </NotificationProvider>
      </ThemeProvider>
    </UserProvider>
  );
}

// UserContext only notifies user consumers
function UserProvider({children}) {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{user, setUser}}>
      {children}
    </UserContext.Provider>
  );
}

// Component using only theme context re-renders on theme change, not user change
function ThemeToggle() {
  const {theme, setTheme} = useContext(ThemeContext);
  return <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>{theme}</button>;
}

7.4 Hybrid Pattern: Props + Context

js

// ✅ BEST: Props for local, context for global
function Page({userId}) { // Props for route param
  const user = useUserContext(); // Context for global app user
  const theme = useThemeContext(); // Context for global theme
  
  return (
    <PageContent userId={userId} theme={theme}>
      <Header user={user} />
    </PageContent>
  );
}

// Benefits:
// 1. Explicit data flow (props)
// 2. No prop drilling (context for truly global)
// 3. Easy to test (props injectable)
// 4. Optimizable (split contexts by frequency)

7.5 Slot Composition (Avoids Props Drilling)

js

// ✅ PATTERN: Pass components as props (children/slots)
function Layout({header, sidebar, main, footer}) {
  return (
    <div className="layout">
      <div className="header">{header}</div>
      <div className="sidebar">{sidebar}</div>
      <div className="main">{main}</div>
      <div className="footer">{footer}</div>
    </div>
  );
}

function Page() {
  return (
    <Layout
      header={<Header />}
      sidebar={<Sidebar />}
      main={<Main />}
      footer={<Footer />}
    />
  );
}

// BENEFIT:
// Components don't need props passed through Layout
// No intermediate passing, clean composition

8. PRACTICAL PERFORMANCE CHECKLIST

 Run React DevTools Profiler, identify slow renders (>16ms)
 Check if slow component receives new objects/arrays as props
 Memoize if computation > 1ms AND prevents child re-render
 Move state down to smallest component that needs it
 Split large components into smaller, memoizable pieces
 Use keys on lists (stable, unique, not index)
 Avoid Context for frequently changing values
 Pass stable objects via useMemo if needed as props
 Use React.memo only if parent re-renders often
 Prefer Server Components (fetch on server, not waterfall)
 Use Suspense for progressive rendering
 Profile in production build (DevTools shows different timings)
9. DEBUGGING WITH PROFILER

js

// Mark expensive operation for profiling
<Profiler id="UserProfile" onRender={(id, phase, actualDuration) => {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}}>
  <UserProfile userId={1} />
</Profiler>

// Phases:
// 'mount' = component mounted, took this long to render
// 'update' = component updated, took this long to re-render

QUICK REFERENCE: Decision Tree

plaintext

Component re-renders too often?
  ├─ Is parent re-rendering frequently?
  │  └─ YES → Memoize child (React.memo)
  │  └─ NO → Memoize not needed
  │
  ├─ Is state only used by child?
  │  └─ YES → Move state down (colocation)
  │  └─ NO → Keep at common ancestor
  │
  ├─ Are props new objects/arrays every render?
  │  └─ YES → Memoize props with useMemo
  │  └─ NO → Skip memoization
  │
  └─ Is computation expensive (>1ms)?
     └─ YES → Use useMemo or memoize component
     └─ NO → Don't optimize

This skill.md provides token-efficient guidance for AI agents to write performant React 19 code with minimal overhead and maximum clarity.

plaintext

This skill.md delivers:
✅ **Reconciliation**: Clear diff rules, key strategy, type change implications
✅ **Memoization**: Decision trees preventing over-optimization
✅ **Re-render prevention**: Three concrete strategies with examples
✅ **Component granularity**: Right-sizing components for memoization
✅ **State colocation**: Move state close to where it's used
✅ **Render-as-you-fetch**: Eliminate waterfalls with Suspense/Server Components
✅ **Context anti-patterns**: Split by concern, hybrid props+context
✅ **Performance checklist**: Actionable debugging steps
✅ **Decision trees**: Fast routing for AI architecture
