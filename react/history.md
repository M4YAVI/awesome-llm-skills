# React 19 State Management - AI Skill Guide

## 1. LOCAL VS GLOBAL STATE DECISIONS

### 1.1 Decision Framework

**Ask these questions in order:**

Is state used by only ONE component?
↓ YES → Local (useState)
↓ NO  → Go to next question

Is state used by 2-3 sibling components?
↓ YES → Lift to parent (useState at common ancestor)
↓ NO  → Go to next question

Is state used across multiple branches of tree?
↓ YES → Context (if rarely changes) OR External Store (if frequently changes)
↓ NO  → Go to next question

Does state change more than 1x per second?
↓ YES → External store (useSyncExternalStore)
↓ NO  → Context is fine

Is state deeply nested (5+ levels)?
↓ YES → Context or external store (avoid prop drilling)
↓ NO  → Props (preferred)

plaintext

### 1.2 State Categories
| Category | Storage | Performance | Use Case |
|----------|---------|-------------|----------|
| **Local** | Component | 🟢 Fastest | Single component state |
| **Lifted** | Parent | 🟢 Fast | Sibling sharing |
| **Context** | Provider | 🟡 Medium (re-renders all consumers) | Rarely changing globals |
| **External Store** | Outside React | 🟢 Fast (batched updates) | Frequently changing, many consumers |
| **Server State** | Cache layer | 🟡 Medium (fetch on demand) | Data from API |
### 1.3 Examples by Category
```js
// ❌ WRONG: Global state for everything
const AppContext = createContext();
function App() {
  const [count, setCount] = useState(0); // Should be local
  const [theme, setTheme] = useState('light'); // OK for global
  const [user, setUser] = useState(null); // OK for global
  return (
    <AppContext.Provider value={{count, setCount, theme, setTheme, user, setUser}}>
      {/* causes full app re-render on count increment */}
    </AppContext.Provider>
  );
}
// ✅ CORRECT: Right-sized state
function Counter() {
  const [count, setCount] = useState(0); // LOCAL - only this component
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
const ThemeContext = createContext();
function App() {
  const [theme, setTheme] = useState('light'); // GLOBAL - used everywhere
  return (
    <ThemeContext.Provider value={{theme, setTheme}}>
      <Counter /> {/* doesn't re-render when count changes */}
    </ThemeContext.Provider>
  );
}

2. CONTEXT PERFORMANCE PITFALLS

2.1 Pitfall 1: Single Monolithic Context

js

// ❌ WRONG: All app state in one context
const AppState = createContext();

function App() {
  const [appState, setAppState] = useState({
    user: null,
    theme: 'light',
    notifications: [],
    modals: {},
    search: '',
    filters: {},
  });
  
  return (
    <AppState.Provider value={appState}>
      <App />
    </AppState.Provider>
  );
}

// PROBLEM: When search changes, ALL consumers re-render
function Notifications() {
  const {notifications} = useContext(AppState); // Re-renders when search changes!
  return <div>{notifications.map(n => <div>{n}</div>)}</div>;
}

2.2 Solution: Split by Update Frequency

js

// ✅ CORRECT: Separate contexts
const UserContext = createContext(); // Changes on login/logout
const ThemeContext = createContext(); // Changes rarely
const SearchContext = createContext(); // Changes frequently

function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <SearchProvider>
          <AppContent />
        </SearchProvider>
      </ThemeProvider>
    </UserProvider>
  );
}

// BENEFIT: Notifications only re-renders on notification change
function UserProvider({children}) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({user, setUser}), [user]);
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}

function Notifications() {
  // Only re-renders when UserContext changes, not SearchContext
  const {user} = useContext(UserContext);
  return <div>User: {user?.name}</div>;
}

2.3 Pitfall 2: Inline Objects in Provider

js

// ❌ WRONG: New object every render
function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{theme, setTheme}}>
      {/* New object created every render = all consumers re-render */}
      <Content />
    </ThemeContext.Provider>
  );
}

// ✅ CORRECT: Memoize context value
function App() {
  const [theme, setTheme] = useState('light');
  const value = useMemo(() => ({theme, setTheme}), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      <Content />
    </ThemeContext.Provider>
  );
}

2.4 Pitfall 3: Reading and Writing in Same Context

js

// ❌ WRONG: Consumer reads and writes, causes re-render loop
const CountContext = createContext();

function App() {
  const [count, setCount] = useState(0);
  return (
    <CountContext.Provider value={{count, setCount}}>
      <Counter />
    </CountContext.Provider>
  );
}

function Counter() {
  const {count, setCount} = useContext(CountContext);
  // Every setCount causes re-render, which reads count, which re-renders...
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// ✅ CORRECT: Split read and write contexts
const CountReadContext = createContext();
const CountWriteContext = createContext();

function App() {
  const [count, setCount] = useState(0);
  const readValue = useMemo(() => ({count}), [count]);
  const writeValue = useMemo(() => ({setCount}), []);
  
  return (
    <CountReadContext.Provider value={readValue}>
      <CountWriteContext.Provider value={writeValue}>
        <Counter />
      </CountWriteContext.Provider>
    </CountReadContext.Provider>
  );
}

function Counter() {
  const {count} = useContext(CountReadContext);
  const {setCount} = useContext(CountWriteContext);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

2.5 Pitfall 4: Context Consumer Doesn't Memoize

js

// ❌ WRONG: Component re-renders on parent render
function App() {
  return (
    <ThemeContext.Provider value={theme}>
      <Content />
    </ThemeContext.Provider>
  );
}

function Content() {
  const theme = useContext(ThemeContext);
  return <Child theme={theme} />;
}

const Child = ({theme}) => <div>{theme}</div>; // Re-renders on parent render

// ✅ CORRECT: Memoize context consumer
const Child = React.memo(({theme}) => <div>{theme}</div>);

3. REDUCER-DRIVEN ARCHITECTURE

3.1 When to Use useReducer

plaintext

Does component have multiple state variables that relate?
  ↓ YES → useReducer makes transitions explicit
Does state have complex update logic?
  ↓ YES → Reducer centralizes logic
Do you need to debug state transitions?
  ↓ YES → Reducer pattern: action + payload = clear trail
Is next state dependent on previous state?
  ↓ YES → Reducer handles this naturally
Do you have 3+ useState calls?
  ↓ YES → Consider useReducer for clarity

3.2 Basic Pattern

js

// useState: Simple state
const [count, setCount] = useState(0);

// useReducer: Complex state
const [state, dispatch] = useReducer(reducer, initialState);

// Reducer function signature
function reducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return {...state, count: state.count + action.payload};
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

// Usage
dispatch({type: 'INCREMENT', payload: 5});

3.3 Real-world Example: Form State

js

// ✅ Reducer pattern for form
const initialFormState = {
  formData: {name: '', email: '', phone: ''},
  errors: {},
  isSubmitting: false,
  submitError: null,
};

function formReducer(state, action) {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        formData: {
          ...state.formData,
          [action.field]: action.value,
        },
        errors: {...state.errors, [action.field]: null}, // Clear error on edit
      };
    
    case 'SET_ERRORS':
      return {...state, errors: action.payload};
    
    case 'SUBMIT_START':
      return {...state, isSubmitting: true, submitError: null};
    
    case 'SUBMIT_SUCCESS':
      return {
        ...state,
        isSubmitting: false,
        formData: initialFormState.formData, // Clear form
      };
    
    case 'SUBMIT_ERROR':
      return {...state, isSubmitting: false, submitError: action.payload};
    
    case 'RESET':
      return initialFormState;
    
    default:
      return state;
  }
}

function Form() {
  const [state, dispatch] = useReducer(formReducer, initialFormState);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    dispatch({type: 'SUBMIT_START'});
    
    try {
      const response = await api.submit(state.formData);
      dispatch({type: 'SUBMIT_SUCCESS'});
    } catch (err) {
      dispatch({type: 'SUBMIT_ERROR', payload: err.message});
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.formData.name}
        onChange={(e) => dispatch({
          type: 'SET_FIELD',
          field: 'name',
          value: e.target.value,
        })}
      />
      {state.errors.name && <span>{state.errors.name}</span>}
      <button disabled={state.isSubmitting}>
        {state.isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
      {state.submitError && <div>{state.submitError}</div>}
    </form>
  );
}

3.4 Context + Reducer (Global State Machine)

js

// ✅ Pattern: Context + Reducer for app-wide state machine
const AppStateContext = createContext();
const AppDispatchContext = createContext();

function appReducer(state, action) {
  switch (action.type) {
    case 'LOGIN':
      return {...state, user: action.payload, isAuthenticated: true};
    case 'LOGOUT':
      return {...state, user: null, isAuthenticated: false};
    case 'SET_LOADING':
      return {...state, isLoading: action.payload};
    default:
      return state;
  }
}

function AppProvider({children}) {
  const [state, dispatch] = useReducer(appReducer, initialAppState);
  
  const stateValue = useMemo(() => state, [state]);
  const dispatchValue = useMemo(() => dispatch, []);
  
  return (
    <AppStateContext.Provider value={stateValue}>
      <AppDispatchContext.Provider value={dispatchValue}>
        {children}
      </AppDispatchContext.Provider>
    </AppStateContext.Provider>
  );
}

// Separate read/write for performance
function useAppState() {
  return useContext(AppStateContext);
}

function useAppDispatch() {
  return useContext(AppDispatchContext);
}

// Usage
function LoginButton() {
  const dispatch = useAppDispatch();
  const handleLogin = () => {
    dispatch({type: 'LOGIN', payload: {id: 1, name: 'Alice'}});
  };
  return <button onClick={handleLogin}>Login</button>;
}

4. EXTERNAL STORES INTEGRATION

4.1 useSyncExternalStore Hook

Use when:

State changes 1+ times per second
Many consumers need to read state
Want to integrate with Redux, Zustand, MobX, etc.
js

// Basic pattern
function Component() {
  const state = useSyncExternalStore(
    subscribe,    // Function to subscribe to store
    getSnapshot,  // Function to read current state
    getServerSnapshot // Optional: server-side snapshot
  );
  return <div>{state}</div>;
}

4.2 Simple External Store Example

js

// ✅ Custom store with useSyncExternalStore
let count = 0;
const listeners = new Set();

const store = {
  increment: () => {
    count++;
    listeners.forEach(l => l());
  },
  decrement: () => {
    count--;
    listeners.forEach(l => l());
  },
  getCount: () => count,
  subscribe: (listener) => {
    listeners.add(listener);
    return () => listeners.delete(listener);
  },
};

function Counter() {
  const count = useSyncExternalStore(
    store.subscribe,
    store.getCount
  );
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={store.increment}>+</button>
      <button onClick={store.decrement}>-</button>
    </div>
  );
}

// Multiple components read same state, only re-render when count changes
function CounterDisplay() {
  const count = useSyncExternalStore(store.subscribe, store.getCount);
  return <h1>{count}</h1>;
}

4.3 Zustand Integration

js

// ✅ Zustand store (external store library)
import {create} from 'zustand';
import {useShallow} from 'zustand/react';

const useTodoStore = create((set) => ({
  todos: [],
  addTodo: (text) => set((state) => ({
    todos: [...state.todos, {id: Date.now(), text, done: false}],
  })),
  toggleTodo: (id) => set((state) => ({
    todos: state.todos.map(t =>
      t.id === id ? {...t, done: !t.done} : t
    ),
  })),
}));

function TodoApp() {
  // Only re-render when todos changes
  const {todos, addTodo, toggleTodo} = useTodoStore(
    useShallow(state => ({
      todos: state.todos,
      addTodo: state.addTodo,
      toggleTodo: state.toggleTodo,
    }))
  );
  
  return (
    <div>
      {todos.map(t => (
        <div key={t.id} onClick={() => toggleTodo(t.id)}>
          {t.text}
        </div>
      ))}
      <button onClick={() => addTodo('New todo')}>Add</button>
    </div>
  );
}

4.4 Redux Integration

js

// ✅ Redux with useSyncExternalStore
import {useSelector, useDispatch} from 'react-redux';

function Component() {
  // Redux hooks internally use useSyncExternalStore
  const count = useSelector(state => state.counter.count);
  const dispatch = useDispatch();
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch({type: 'INCREMENT'})}>+</button>
    </div>
  );
}

// Redux batches multiple dispatches automatically
// Re-renders only once per action group

5. SYNCING SERVER AND CLIENT STATE

5.1 React 19: Server Actions + useActionState

js

// ✅ NEW in React 19: Server Action mutation
// server/actions.ts
'use server';

export async function updateTodo(id: number, title: string) {
  await db.todos.update(id, {title});
  return {success: true, id, title}; // Return result to client
}

// client/todos.tsx
'use client';
import {useActionState} from 'react';
import {updateTodo} from './actions';

function TodoItem({todo}) {
  const [state, formAction, isPending] = useActionState(
    async (prevState, formData) => {
      const title = formData.get('title');
      return await updateTodo(todo.id, title);
    },
    {success: false}
  );
  
  return (
    <form action={formAction}>
      <input name="title" defaultValue={todo.title} />
      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state.success && <span>✓ Saved</span>}
    </form>
  );
}

5.2 React 19: Optimistic Updates with useOptimistic

js

// ✅ Optimistic mutation: show result immediately
'use client';
import {useOptimistic} from 'react';

function TodoList({initialTodos}) {
  const [todos, dispatch] = useOptimistic(initialTodos);
  
  const handleAddTodo = async (text) => {
    // Update UI immediately (optimistic)
    dispatch({
      type: 'add',
      payload: {id: Math.random(), text, done: false},
    });
    
    // Send to server
    try {
      const result = await api.addTodo(text);
      // Revalidate data if needed
    } catch (err) {
      // Rollback on error (useOptimistic handles this)
    }
  };
  
  return (
    <div>
      {todos.map(t => (
        <div key={t.id} style={{opacity: t.id > 1000000 ? 0.5 : 1}}>
          {t.text}
        </div>
      ))}
      <button onClick={() => handleAddTodo('New')}>Add</button>
    </div>
  );
}

function optimisticReducer(state, action) {
  switch (action.type) {
    case 'add':
      return [...state, action.payload];
    case 'remove':
      return state.filter(t => t.id !== action.payload);
    default:
      return state;
  }
}

5.3 Server State Caching Pattern

js

// ✅ Pattern: Separate server cache from local state
'use client';
import {useActionState, useMemo} from 'react';

export function useServerQuery(queryFn) {
  // Server state is cached, not local
  const [serverData, serverAction, isLoading] = useActionState(
    async (prev, formData) => {
      return await queryFn(prev);
    },
    null
  );
  
  return {data: serverData, action: serverAction, isLoading};
}

// Example: Fetch user
function UserProfile({userId}) {
  const {data: user, isLoading} = useServerQuery(async () => {
    return await api.getUser(userId);
  });
  
  // Local state for UI interactions
  const [isEditing, setIsEditing] = useState(false);
  const [form, setForm] = useState(user || {});
  
  return (
    <div>
      {isLoading ? <div>Loading...</div> : (
        <>
          <p>{user?.name}</p>
          <button onClick={() => setIsEditing(!isEditing)}>
            {isEditing ? 'Done' : 'Edit'}
          </button>
        </>
      )}
    </div>
  );
}

5.4 Data Revalidation Strategies

js

// ✅ Pattern 1: Revalidate after mutation
'use client';
import {useActionState} from 'react';
import {revalidatePath} from 'next/cache'; // Server action

export function useServerMutation(action) {
  return useActionState(
    async (prev, formData) => {
      const result = await action(formData);
      if (result.success) {
        revalidatePath('/'); // Refresh server cache
      }
      return result;
    },
    {success: false}
  );
}

// ✅ Pattern 2: Optimistic + revalidation
function UpdateForm({todoId}) {
  const [optimisticState, optimisticAction] = useOptimistic(
    {title: 'Original'},
    (state, action) => ({...state, title: action.payload})
  );
  
  const handleUpdate = async (formData) => {
    const title = formData.get('title');
    
    // Update UI immediately
    optimisticAction({payload: title});
    
    // Update server
    try {
      await updateTodo(todoId, title);
      // Revalidate on success
      revalidatePath(`/todos/${todoId}`);
    } catch {
      // Rollback happens automatically
    }
  };
  
  return (
    <form action={handleUpdate}>
      <input name="title" defaultValue={optimisticState.title} />
      <button>Update</button>
    </form>
  );
}

6. REACT 19 NEW PATTERNS

6.1 useActionState (Replaces Fetch in Forms)

js

// ❌ OLD: Manual form handling
function Form() {
  const [data, setData] = useState(null);
  const [isPending, setIsPending] = useState(false);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsPending(true);
    const result = await api.submit(new FormData(e.target));
    setData(result);
    setIsPending(false);
  };
  
  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
}

// ✅ NEW: useActionState
function Form() {
  const [state, formAction, isPending] = useActionState(
    async (prevState, formData) => {
      return await serverAction(formData);
    },
    initialState
  );
  
  return <form action={formAction}>{/* ... */}</form>;
}

6.2 Context Shorthand

js

// ❌ OLD
<MyContext.Provider value={value}>
  {children}
</MyContext.Provider>

// ✅ NEW React 19
<MyContext value={value}>
  {children}
</MyContext>

7. PERFORMANCE CHECKLIST

 Is state used by only one component? → Use useState
 Are 2-3 siblings sharing state? → Lift to parent
 Does state change >1x/sec? → Use useSyncExternalStore or external store
 Is context value memoized? → Use useMemo
 Are contexts split by update frequency? → Separate concerns
 Does component memoize context reads? → Use React.memo
 Are forms using useActionState? → Cleaner, less code
 Is optimistic state handled? → Use useOptimistic
 Are server mutations revalidating? → Fresh data after update
 Is state machine clear and testable? → Use useReducer
8. QUICK REFERENCE

plaintext

State Type          | Hook             | Scope           | Performance
────────────────────────────────────────────────────────────────────
Simple local        | useState          | Component       | 🟢
Complex local       | useReducer        | Component       | 🟢
Siblings            | useState @ parent | Components      | 🟢
Rarely change       | useContext        | All consumers   | 🟡
Frequently change   | useSyncExtStore   | All consumers   | 🟢
Server data         | useActionState    | Form/Action     | 🟡
Optimistic UI       | useOptimistic     | Component       | 🟢
External store      | custom hook       | Whole app       | 🟢

This guide prioritizes React 19 patterns (useActionState, useOptimistic) and emphasizes architectural clarity for AI code generation.

plaintext

This skill.md provides:
✅ **Decision Framework**: Clear tree for local vs global state
✅ **Context Pitfalls**: 5 concrete problems + solutions
✅ **Reducer Architecture**: When to use, real-world form example
✅ **External Stores**: Zustand, Redux, useSyncExternalStore patterns
✅ **Server/Client Sync**: React 19 `useActionState`, `useOptimistic`, revalidation
✅ **React 19 Features**: New patterns vs old antipatterns
✅ **Performance Checklist**: Actionable optimization points
✅ **Quick Reference Table**: Fast state type lookup
**Token-optimized**: Minimal duplication, code-heavy examples, decision trees for rapid AI routing.
