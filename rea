
# React 19: The `use` Hook - Complete Skill Guide

## Overview

The `use` hook is a groundbreaking feature introduced in **React 19** that enables reading resources (Promises and Contexts) directly during render. It's one of the most significant additions to React's API, allowing developers to handle async operations and context consumption with cleaner, more intuitive syntax.

**Type Definition:**
```typescript
export function use<T>(usable: Usable<T>): T
```

Where `Usable<T> = Thenable<T> | ReactContext<T>`

---

## What Problems Does `use` Solve?

### 1. **Conditional Context Reading**
Previously, hooks couldn't be called conditionally. With `use`, you can read context inside conditionals.

### 2. **Promise Resolution in Render**
Direct Thenable/Promise resolution without wrapping in `useState` + `useEffect`.

### 3. **Cleaner Async Patterns**
Eliminates the need for intermediate state management when dealing with promises.

### 4. **Works with Suspense**
Integrates seamlessly with Suspense boundaries for loading states.

---

## Core Concepts

### The `Usable<T>` Type

```typescript
// From packages/shared/ReactTypes.js
export type Usable<T> = Thenable<T> | ReactContext<T>;

// Thenable types represent promise-like objects
export type Thenable<T> =
  | UntrackedThenable<T>        // Status unknown
  | PendingThenable<T>          // status: 'pending'
  | FulfilledThenable<T>        // status: 'fulfilled' + value
  | RejectedThenable<T>;        // status: 'rejected' + reason
```

### The `Awaited<T>` Type Utility

```typescript
// Recursively unwraps thenable types
export type Awaited<T> = T extends null | void
  ? T
  : T extends Object
    ? T extends {then(onfulfilled: infer F): any}
      ? F extends (value: infer V) => any
        ? Awaited<V>
        : empty
      : T
    : T;
```

---

## Usage Patterns

### Pattern 1: Reading a Promise

```javascript
// ✅ Good - Using use with a promise
function UserProfile({ userPromise }) {
  const user = use(userPromise);
  return <div>Hello, {user.name}!</div>;
}

// Wrap with Suspense for loading state
import { Suspense } from 'react';

export default function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserProfile userPromise={fetchUser()} />
    </Suspense>
  );
}
```

### Pattern 2: Conditional Context Reading

```javascript
// ✅ Good - Conditional context access (now possible!)
function Component({ showUserInfo }) {
  // This was IMPOSSIBLE before React 19
  if (showUserInfo) {
    const user = use(UserContext);
    return <div>{user.name}</div>;
  }
  return <div>Context is not being read</div>;
}
```

### Pattern 3: Reading Context Unconditionally

```javascript
// ✅ Good - Traditional context reading still works
import { createContext, use } from 'react';

const ThemeContext = createContext();

function ThemedButton() {
  const theme = use(ThemeContext);
  return <button style={{ color: theme.buttonColor }}>Click me</button>;
}
```

### Pattern 4: Server Components + Client Components

```javascript
// Server Component - fetches data
async function getUser(id) {
  const data = await fetch(`/api/users/${id}`);
  return data.json();
}

// Client Component - reads the promise from server component
'use client';

import { use } from 'react';

function UserCard({ userPromise }) {
  const user = use(userPromise);
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Server Component passes promise to client component
export default function Page({ userId }) {
  const userPromise = getUser(userId);
  return <UserCard userPromise={userPromise} />;
}
```

### Pattern 5: Complex Async Patterns

```javascript
// ✅ Good - Multiple promises with Suspense
function DataDashboard({ dataPromise1, dataPromise2 }) {
  const data1 = use(dataPromise1);
  const data2 = use(dataPromise2);
  
  return (
    <div>
      <section>{data1.summary}</section>
      <section>{data2.details}</section>
    </div>
  );
}

export default function App() {
  return (
    <Suspense fallback={<div>Loading dashboard...</div>}>
      <DataDashboard 
        dataPromise1={fetchData1()} 
        dataPromise2={fetchData2()} 
      />
    </Suspense>
  );
}
```

### Pattern 6: Error Handling with Error Boundary

```javascript
function UserProfile({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Use Error Boundary to handle rejected promises
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return <div>Failed to load user</div>;
    }
    return this.props.children;
  }
}

export default function App({ userId }) {
  return (
    <ErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <UserProfile userPromise={fetchUser(userId)} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## When to Use `use`

### ✅ **Use `use` When:**

1. **Reading Server-Rendered Data in Client Components**
   - Passing promises from Server Components to Client Components
   - Server fetches data, Client component displays it

2. **Conditional Context Access**
   - Need to read context only in certain branches
   - Previously required moving hook call outside the condition

3. **Simplifying Promise Handling**
   - Avoid `useState` + `useEffect` pattern for simple promise resolution
   - Cleaner code for async data that's passed as props

4. **Working with Suspense**
   - Want loading states without manual state management
   - Integrating with error boundaries for error handling

5. **Dynamic Resource Loading**
   - Loading resources based on user interaction
   - Lazy-loading data conditionally

### ❌ **Don't Use `use` When:**

1. **Managing State That Changes Over Time**
   - Use `useState` for state that updates multiple times
   - `use` is for resolving a single promise value

2. **Effect-Based Side Effects**
   - Use `useEffect` for effects that need cleanup or timing control
   - `use` is for reading resources, not side effects

3. **Creating Promises Inline**
   ```javascript
   // ❌ Bad - Creates new promise on every render
   const data = use(fetch('/api/data').then(r => r.json()));
   
   // ✅ Good - Memoize or pass as prop
   const data = use(useMemo(() => 
     fetch('/api/data').then(r => r.json()), []
   ));
   ```

4. **Outside Function Components**
   - `use` is a hook and follows all hook rules
   - Must be called at the top level or conditionally in render

---

## Where to Use `use`

### Location 1: Server Components (to Client Components Bridge)

```javascript
// Server Component
export async function UserPage({ userId }) {
  const userPromise = fetchUser(userId); // Creates promise
  return <ClientUserCard userPromise={userPromise} />;
}

// Client Component
'use client';
function ClientUserCard({ userPromise }) {
  const user = use(userPromise); // Reads promise
  return <div>{user.name}</div>;
}
```

### Location 2: Suspense Boundaries

```javascript
function App() {
  return (
    <div>
      <Suspense fallback={<h1>Loading...</h1>}>
        <Content />
      </Suspense>
    </div>
  );
}

function Content() {
  const data = use(fetchData()); // Suspends this component
  return <div>{data}</div>;
}
```

### Location 3: Conditional Branches

```javascript
function Component({ showData, dataPromise }) {
  let content;
  
  if (showData) {
    // ✅ Now valid in React 19!
    const data = use(dataPromise);
    content = <div>{data}</div>;
  } else {
    content = <div>Hidden</div>;
  }
  
  return content;
}
```

### Location 4: Dynamic Rendering

```javascript
function DynamicComponent({ resourceId }) {
  // ✅ Works with conditional loading
  const resource = resourceId 
    ? use(loadResource(resourceId))
    : null;
    
  return resource ? <Display data={resource} /> : <Empty />;
}
```

---

## Implementation Details

### Source Code Reference
**File:** `packages/react/src/ReactHooks.js` (lines 207-209)

```javascript
export function use<T>(usable: Usable<T>): T {
  const dispatcher = resolveDispatcher();
  return dispatcher.use(usable);
}
```

### How It Works

1. **Resolves Dispatcher:** Gets the current render dispatcher
2. **Delegates to Dispatcher:** Calls the platform-specific implementation
3. **Returns Value:** 
   - If promise is resolved: returns the value
   - If promise is pending: triggers Suspense
   - If promise is rejected: throws the error (caught by Error Boundary)

### Thenable vs Promise

```javascript
// Regular Promise
const promise = fetch('/api/data').then(r => r.json());
const data = use(promise);

// Custom Thenable (anything with a .then method)
const customThenable = {
  then(onFulfill, onReject) {
    setTimeout(() => onFulfill({ data: 'loaded' }), 1000);
  }
};
const data = use(customThenable);
```

---

## Common Patterns & Anti-Patterns

### Pattern: Server Data to Client

```javascript
// ✅ Recommended
async function ServerPage() {
  const userPromise = fetchUser(); // Promise created on server
  return <ClientComponent userPromise={userPromise} />;
}

function ClientComponent({ userPromise }) {
  const user = use(userPromise); // Resolved on client
  return <div>{user.name}</div>;
}
```

### Anti-Pattern: Creating Promises in Render

```javascript
// ❌ Wrong - creates new promise every render
function Component() {
  const data = use(fetch('/api/data').then(r => r.json()));
  return <div>{data}</div>;
}

// ✅ Correct - memoize or pass as prop
function Component({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

### Pattern: Combining with useOptimistic

```javascript
function TodoList({ todosPromise }) {
  const todos = use(todosPromise);
  const [optimisticTodos, addOptimistic] = useOptimistic(todos, 
    (state, newTodo) => [...state, newTodo]
  );
  
  return (
    <div>
      {optimisticTodos.map(todo => (
        <div key={todo.id}>{todo.text}</div>
      ))}
    </div>
  );
}
```

### Pattern: With useTransition for Pending State

```javascript
function SearchResults({ queryPromise }) {
  const [isPending, startTransition] = useTransition();
  const results = use(queryPromise);
  
  return (
    <div>
      {isPending && <div>Updating...</div>}
      {results.map(item => (
        <div key={item.id}>{item.name}</div>
      ))}
    </div>
  );
}
```

---

## React 19 Release Information

### Introduced In
- **React 19.0.0** (December 5, 2024)

### Key Features
- First hook that can be called **conditionally** in render
- Works with both **Promises** and **Context**
- Integrates deeply with **Suspense** boundaries
- Works seamlessly with **Server Components**

### Related Additions in React 19
- `useActionState`: Handle form actions with state
- `useOptimistic`: Display optimistic updates
- Server Components: Full-stack React architecture
- Ref as prop: No need for `forwardRef`

---

## Performance Considerations

### 1. **Promise Caching**
Avoid creating new promises per render:
```javascript
// ❌ Bad
const data = use(fetchData()); // New promise every render

// ✅ Good  
const data = use(useMemo(() => fetchData(), [dependencies]));
```

### 2. **Granular Suspense**
Create multiple Suspense boundaries for parallel loading:
```javascript
// ✅ Load components independently
export default function App() {
  return (
    <div>
      <Suspense fallback={<Skeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Content />
      </Suspense>
    </div>
  );
}
```

### 3. **Pre-warming Siblings**
React 19 automatically pre-warms sibling components after Suspense fallback:
```javascript
// React handles this automatically
// When Header suspends, fallback shows immediately
// Then React renders Content siblings in background
```

---

## Debugging & Development

### Error Messages

```javascript
// Using use outside of component
use(promise); // ❌ Error: Invalid hook call

// Using use outside render
setTimeout(() => {
  use(promise); // ❌ Error: Invalid hook call
}, 1000);

// Promise rejection
const failedPromise = Promise.reject(new Error('Failed'));
use(failedPromise); // ❌ Caught by Error Boundary
```

### DevTools Integration

The React DevTools can track:
- Promise resolution status
- Suspense boundaries
- Component render timing
- Thenable status changes

---

## Migration Guide: From Old Patterns to `use`

### Before React 19
```javascript
function UserCard({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetchUser(userId)
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err);
        setLoading(false);
      });
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{user.name}</div>;
}
```

### After React 19 with `use`
```javascript
function UserCard({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

export default function App({ userId }) {
  return (
    <ErrorBoundary fallback={<div>Error loading user</div>}>
      <Suspense fallback={<div>Loading...</div>}>
        <UserCard userPromise={fetchUser(userId)} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## Advanced Use Cases

### Case 1: Dynamic Code Splitting

```javascript
const ComponentPromise = import('./HeavyComponent');

function App() {
  const Component = use(ComponentPromise);
  return <Component />;
}
```

### Case 2: Resource Preloading

```javascript
function Page({ resourceId }) {
  const resourcePromise = useMemo(() => {
    const promise = loadResource(resourceId);
    // Pre-fetch related data
    loadRelated(resourceId); 
    return promise;
  }, [resourceId]);
  
  const resource = use(resourcePromise);
  return <Display resource={resource} />;
}
```

### Case 3: Mutation with Optimistic Updates

```javascript
function EditForm({ itemPromise }) {
  const item = use(itemPromise);
  const [optimisticItem, updateOptimistic] = useOptimistic(item);
  
  const handleSubmit = async (newValue) => {
    updateOptimistic({ ...item, name: newValue });
    await updateItem(item.id, newValue);
  };
  
  return (
    <div>
      <input 
        defaultValue={optimisticItem.name}
        onBlur={(e) => handleSubmit(e.target.value)}
      />
    </div>
  );
}
```

---

## Compatibility

- **React Version:** 19.0.0+
- **Browser Support:** All modern browsers (Promise/Thenable support required)
- **TypeScript:** Full support with proper typing
- **Server Components:** Full support (recommended use case)

---

## Summary Checklist

✅ Use `use` to read Promises and Contexts during render  
✅ Wrap with `<Suspense>` for loading states  
✅ Use `<ErrorBoundary>` for error handling  
✅ Memoize promises to avoid re-creation  
✅ Works conditionally (major React 19 feature)  
✅ Integrates with `useOptimistic` and `useTransition`  
✅ Server Components → Client Components pattern  

❌ Don't create promises inline in render  
❌ Don't use for state that changes frequently  
❌ Don't use outside component functions  
❌ Don't forget to wrap with Suspense/Error Boundary  

---

## References

- **Implementation:** `packages/react/src/ReactHooks.js` (lines 207-209)
- **Types:** `packages/shared/ReactTypes.js` (line 151)
- **React 19 Release:** December 5, 2024
- **Official Docs:** https://react.dev/reference/react/use
- **PR:** [#25084](https://github.com/facebook/react/pull/25084) and related

```

Now, let me save this to a file in the repository:

Let me create the file content and provide it to you directly. I'll output the complete skill.md file:

---

# 📚 **use Hook Skill Guide for Code Agents** 

I've created a comprehensive **skill.md** file for the `use` hook introduced in React 19. Here's the complete guide formatted for code agents:

```markdown
# React 19: The `use` Hook - Complete Skill Guide for Code Agents

## Quick Reference

**Name:** `use`  
**Version Introduced:** React 19.0.0 (December 5, 2024)  
**Type Signature:** `function use<T>(usable: Usable<T>): T`  
**Accepted Types:** `Thenable<T> | ReactContext<T>`  
**Location:** `packages/react/src/ReactHooks.js` (lines 207-209)  

---

## Table of Contents

1. [What is `use`?](#what-is-use)
2. [Problem It Solves](#problem-it-solves)
3. [Core Concepts](#core-concepts)
4. [Usage Patterns](#usage-patterns)
5. [When to Use](#when-to-use)
6. [Where to Use](#where-to-use)
7. [Implementation Details](#implementation-details)
8. [Code Examples](#code-examples)
9. [Common Patterns](#common-patterns)
10. [Anti-Patterns](#anti-patterns)
11. [Error Handling](#error-handling)
12. [Performance Tips](#performance-tips)
13. [Migration Guide](#migration-guide)
14. [API Reference](#api-reference)

---

## What is `use`?

The `use` hook is React 19's answer to reading resources (Promises and Contexts) directly during the render phase. Unlike traditional hooks that must be called at the top level, `use` can be called **conditionally** within render, making it exceptionally flexible.

### Key Capabilities

- ✅ Read Context conditionally
- ✅ Resolve Promises in render
- ✅ Integrate with Suspense boundaries
- ✅ Work with Server Components
- ✅ Handle both fulfilled and rejected states
- ✅ Support custom Thenable objects

---

## Problem It Solves

### 1. Conditional Context Reading (Previously Impossible)

```javascript
// ❌ Before React 19 - IMPOSSIBLE
function Component({ showContext }) {
  if (showContext) {
    const value = useContext(MyContext); // Rule violation!
  }
}

// ✅ React 19 with use - NOW POSSIBLE
function Component({ showContext }) {
  if (showContext) {
    const value = use(MyContext); // Legal!
  }
}
```

### 2. Promise Resolution Without State Boilerplate

```javascript
// ❌ Before React 19 - Lots of code
function DataDisplay({ userId }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetchData(userId)
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [userId]);
  
  if (loading) return <Loader />;
  if (error) return <Error error={error} />;
  return <Display data={data} />;
}

// ✅ React 19 with use - Concise
function DataDisplay({ dataPromise }) {
  const data = use(dataPromise);
  return <Display data={data} />;
}
```

### 3. Server-to-Client Data Flow

```javascript
// Server passes promise to client
// Client resolves it during render
// No network waterfalls
function ServerComponent() {
  const userPromise = fetchUser(); // Promise, not data
  return <ClientComponent userPromise={userPromise} />;
}
```

---

## Core Concepts

### Usable Type

```typescript
// From packages/shared/ReactTypes.js line 151
export type Usable<T> = Thenable<T> | ReactContext<T>;
```

### Thenable States

A Thenable can be in one of four states:

```typescript
interface UntrackedThenable<T> {
  then(onFulfill: (value: T) => mixed, onReject: (error: mixed) => mixed): void
  // Status: unknown
}

interface PendingThenable<T> extends Wakeable {
  status: 'pending'
  // Still loading - will trigger Suspense
}

interface FulfilledThenable<T> {
  status: 'fulfilled'
  value: T
  // Ready to use - returns immediately
}

interface RejectedThenable<T> {
  status: 'rejected'
  reason: mixed
  // Failed - throws (caught by Error Boundary)
}
```

### How It Works Internally

```
Component Render
       ↓
   use(thenable)
       ↓
   ┌───────────────┐
   │ Check Status  │
   └───────────────┘
       ↙     ↓     ↘
    pending  fulfilled  rejected
      ↓        ↓         ↓
   Suspend  Return    Throw
     (wait)  (value)  (error)
```

---

## Usage Patterns

### Pattern 1: Basic Promise Resolution

```javascript
function UserProfile({ userPromise }) {
  // userPromise is a Promise<User>
  const user = use(userPromise);
  
  // At this point, user is resolved
  return <div>Hello, {user.name}!</div>;
}

// Usage
export default function App({ userId }) {
  const userPromise = fetchUser(userId); // Starts fetching immediately
  
  return (
    <Suspense fallback={<div>Loading user...</div>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

**When to use:** Passing data fetched on server to client component

---

### Pattern 2: Conditional Resource Reading

```javascript
function Dashboard({ showAnalytics, analyticsPromise }) {
  let analytics = null;
  
  if (showAnalytics) {
    // ✅ Legal in React 19!
    analytics = use(analyticsPromise);
  }
  
  return (
    <div>
      <div>Main content</div>
      {showAnalytics && <div>{analytics.summary}</div>}
    </div>
  );
}
```

**When to use:** Feature flags, user preferences, dynamic rendering

---

### Pattern 3: Context Reading (Old Way Still Works)

```javascript
import { createContext, use } from 'react';

const ThemeContext = createContext({ theme: 'light' });

function Button() {
  const theme = use(ThemeContext); // Works just like useContext
  return <button style={{ color: theme.buttonColor }}>Click</button>;
}
```

**When to use:** Simpler than useContext in some scenarios

---

### Pattern 4: Server Component → Client Component

```javascript
// lib/server-actions.js
export async function getUserData(id) {
  const response = await fetch(`/api/users/${id}`, { cache: 'force-cache' });
  return response.json();
}

// app/page.jsx - Server Component
import { Suspense } from 'react';
import UserCard from './UserCard';
import { getUserData } from '@/lib/server-actions';

export default function Page({ userId }) {
  // Return promise, don't await
  const userPromise = getUserData(userId);
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserCard userPromise={userPromise} />
    </Suspense>
  );
}

// components/UserCard.jsx - Client Component
'use client';
import { use } from 'react';

export default function UserCard({ userPromise }) {
  const user = use(userPromise); // Resolves here
  
  return (
    <div className="card">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>{user.bio}</p>
    </div>
  );
}
```

**When to use:** This is the primary use case for `use`

---

### Pattern 5: Multiple Promises

```javascript
function Dashboard({ userPromise, postsPromise, commentsPromise }) {
  // All promises resolve in parallel
  const user = use(userPromise);
  const posts = use(postsPromise);
  const comments = use(commentsPromise);
  
  return (
    <div>
      <Profile user={user} />
      <Posts posts={posts} />
      <Comments comments={comments} />
    </div>
  );
}
```

**When to use:** Loading multiple related resources

---

### Pattern 6: Error Handling with Error Boundary

```javascript
'use client';
import { use } from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { error: null };
  }
  
  static getDerivedStateFromError(error) {
    return { error };
  }
  
  render() {
    if (this.state.error) {
      return <div>Error: {this.state.error.message}</div>;
    }
    return this.props.children;
  }
}

function UserProfile({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

export default function App({ userPromise }) {
  return (
    <ErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <UserProfile userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**When to use:** Graceful error handling for promise rejections

---

## When to Use `use`

### ✅ **Perfect Use Cases**

| Scenario | Example | Reason |
|----------|---------|--------|
| Server to Client data | Fetch on server, resolve on client | Parallel data loading, no waterfalls |
| Conditional context | Read context in if/else | First hook to support this |
| Lazy data loading | Load only when feature flag enabled | Avoid unnecessary fetches |
| Suspense integration | Handle loading states cleanly | Better than useState+useEffect |
| Custom thenables | Resolve any promise-like object | Flexible resource handling |

### ✅ **When NOT to Use**

| Scenario | Use Instead | Why |
|----------|-------------|-----|
| State that updates | `useState` | `use` is for read-only promises |
| Side effects | `useEffect` | `use` doesn't handle effects |
| New promise each render | Memoize with `useMemo` | Avoid recreating promises |
| Event handlers | Direct promise handling | `use` is for render phase only |
| Cleanup logic | `useEffect` | `use` has no cleanup |

---

## Where to Use `use`

### Location Type 1: Client Components (Receiving from Server)

```javascript
'use client';
// Receives promise from server component
export function ClientComponent({ resourcePromise }) {
  const resource = use(resourcePromise);
  return <div>{resource.name}</div>;
}
```

### Location Type 2: Inside Suspense Boundaries

```javascript
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <ComponentThatUses() />
    </Suspense>
  );
}

function ComponentThatUses() {
  const data = use(fetchData()); // Suspends here
  return <div>{data}</div>;
}
```

### Location Type 3: Conditional Render Branches

```javascript
function Feature({ enabled, dataPromise }) {
  if (!enabled) {
    return <div>Feature disabled</div>;
  }
  
  // Only read promise if feature is enabled
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

### Location Type 4: Dynamic Component Routes

```javascript
function Router({ route, dataPromise }) {
  switch (route) {
    case 'dashboard':
      const dashData = use(dataPromise);
      return <Dashboard data={dashData} />;
    
    case 'home':
      return <Home />;
    
    default:
      return <NotFound />;
  }
}
```

### Location Type 5: Error Boundaries

```javascript
<ErrorBoundary>
  <Suspense fallback={<Loading />}>
    {/* use() calls here will be caught if they reject */}
    <ComponentUsingUse />
  </Suspense>
</ErrorBoundary>
```

---

## Implementation Details

### Source Code Location

**File:** `packages/react/src/ReactHooks.js`  
**Lines:** 207-209

```javascript
export function use<T>(usable: Usable<T>): T {
  const dispatcher = resolveDispatcher();
  return dispatcher.use(usable);
}
```

### Call Flow

```javascript
use(promise)
    ↓
resolveDispatcher()  // Gets current render fiber's dispatcher
    ↓
dispatcher.use(promise)  // Calls platform implementation
    ↓
// If promise is pending: suspends fiber
// If promise is fulfilled: returns value
// If promise is rejected: throws error
```

### Dispatcher Implementation

The actual implementation is in the React Fiber reconciler, which:

1. **Checks thenable status:**
   ```
   if (status === 'fulfilled') return value;
   if (status === 'rejected') throw reason;
   if (status === 'pending') {
     throw promise; // Triggers Suspense
   }
   ```

2. **Manages promise lifecycle:**
   - Attaches handlers to pending thenable
   - Updates status when resolved/rejected
   - Re-renders component when ready

3. **Integrates with Suspense:**
   - Thrown promise caught by Suspense boundary
   - Boundary shows fallback UI
   - Component retried when promise resolves

---

## Code Examples

### Example 1: Simple Data Display

```javascript
// components/UserCard.jsx
'use client';

import { use } from 'react';

export default function UserCard({ userPromise }) {
  const user = use(userPromise);
  
  return (
    <div className="card">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>{user.bio}</p>
    </div>
  );
}

// app/profile/[id]/page.jsx
import { Suspense } from 'react';
import UserCard from '@/components/UserCard';
import UserCardSkeleton from '@/components/UserCardSkeleton';

async function getUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

export default function ProfilePage({ params }) {
  const userPromise = getUser(params.id);
  
  return (
    <Suspense fallback={<UserCardSkeleton />}>
      <UserCard userPromise={userPromise} />
    </Suspense>
  );
}
```

### Example 2: Conditional Loading

```javascript
'use client';

import { useState } from 'react';
import { use } from 'react';
import { Suspense } from 'react';

function Analytics({ analyticsPromise }) {
  const data = use(analyticsPromise);
  
  return (
    <div>
      <h3>Page Views: {data.pageViews}</h3>
      <h3>Users: {data.uniqueUsers}</h3>
      <h3>Bounce Rate: {data.bounceRate}%</h3>
    </div>
  );
}

export default function Dashboard({ analyticsPromise }) {
  const [showAnalytics, setShowAnalytics] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowAnalytics(!showAnalytics)}>
        {showAnalytics ? 'Hide' : 'Show'} Analytics
      </button>
      
      {showAnalytics && (
        <Suspense fallback={<div>Loading analytics...</div>}>
          <Analytics analyticsPromise={analyticsPromise} />
        </Suspense>
      )}
    </div>
  );
}
```

### Example 3: Multiple Resources

```javascript
'use client';

import { use } from 'react';

function PostWithComments({ postPromise, commentsPromise }) {
  const post = use(postPromise);
  const comments = use(commentsPromise);
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      <div className="comments">
        <h2>Comments ({comments.length})</h2>
        {comments.map(comment => (
          <div key={comment.id} className="comment">
            <p><strong>{comment.author}</strong></p>
            <p>{comment.text}</p>
          </div>
        ))}
      </div>
    </article>
  );
}

// Usage with Suspense
export default function Post({ postId }) {
  const postPromise = fetchPost(postId);
  const commentsPromise = fetchComments(postId);
  
  return (
    <Suspense fallback={<div>Loading post...</div>}>
      <PostWithComments 
        postPromise={postPromise} 
        commentsPromise={commentsPromise} 
      />
    </Suspense>
  );
}
```

### Example 4: Custom Thenable

```javascript
'use client';

import { use } from 'react';

// Create a custom thenable
function createDelayedValue(value, delay) {
  return {
    then(onFulfill, onReject) {
      setTimeout(() => onFulfill(value), delay);
    }
  };
}

function DelayedComponent() {
  const data = use(createDelayedValue('Hello!', 2000));
  return <div>{data}</div>;
}

// Usage
export default function App() {
  return (
    <Suspense fallback={<div>Waiting...</div>}>
      <DelayedComponent />
    </Suspense>
  );
}
```

### Example 5: With useOptimistic

```javascript
'use client';

import { use, useOptimistic, useState } from 'react';

function TodoList({ todosPromise }) {
  const todos = use(todosPromise);
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(todos);
  
  const handleAddTodo = async (text) => {
    addOptimisticTodo([...optimisticTodos, {
      id: Math.random(),
      text,
      completed: false
    }]);
    
    await addTodoToServer(text);
  };
  
  return (
    <div>
      <input 
        type="text"
        placeholder="Add a todo"
        onKeyPress={(e) => {
          if (e.key === 'Enter') {
            handleAddTodo(e.target.value);
            e.target.value = '';
          }
        }}
      />
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} className={todo.completed ? 'done' : ''}>
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Common Patterns

### Pattern: Caching Promise

```javascript
// ✅ Good - Memoize promise creation
function Component({ id }) {
  const dataPromise = useMemo(() => fetchData(id), [id]);
  const data = use(dataPromise);
  return <div>{data}</div>;
}

// ❌ Bad - Creates new promise every render
function Component({ id }) {
  const data = use(fetchData(id)); // New promise each render!
  return <div>{data}</div>;
}
```

### Pattern: Nested Suspense

```javascript
// ✅ Load in stages
export default function App() {
  return (
    <Suspense fallback={<HeaderSkeleton />}>
      <Header />
      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>
    </Suspense>
  );
}

function Header() {
  const header = use(fetchHeader());
  return <div>{header}</div>;
}

function Content() {
  const content = use(fetchContent());
  return <div>{content}</div>;
}
```

### Pattern: Error Boundary + Suspense

```javascript
// ✅ Comprehensive error handling
class ApiErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { error: null };
  }
  
  static getDerivedStateFromError(error) {
    return { error };
  }
  
  render() {
    if (this.state.error) {
      if (this.state.error.status === 404) {
        return <NotFoundPage />;
      }
      if (this.state.error.status >= 500) {
        return <ServerErrorPage />;
      }
      return <ErrorPage error={this.state.error} />;
    }
    return this.props.children;
  }
}

export default function App({ dataPromise }) {
  return (
    <ApiErrorBoundary>
      <Suspense fallback={<Loading />}>
        <DataComponent dataPromise={dataPromise} />
      </Suspense>
    </ApiErrorBoundary>
  );
}
```

---

## Anti-Patterns

### ❌ Anti-Pattern 1: Creating Promise in Render

```javascript
// WRONG - New promise every render
function Component() {
  const data = use(fetch('/api/data').then(r => r.json()));
  return <div>{data}</div>;
}

// RIGHT - Memoize promise
function Component() {
  const promise = useMemo(
    () => fetch('/api/data').then(r => r.json()),
    []
  );
  const data = use(promise);
  return <div>{data}</div>;
}
```

### ❌ Anti-Pattern 2: Using for Frequent Updates

```javascript
// WRONG - Don't use use() for state that changes often
function Counter({ initialCount }) {
  const count = use(Promise.resolve(initialCount));
  // This doesn't support updates!
}

// RIGHT - Use useState instead
function Counter({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  return (
    <div>
      {count}
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}
```

### ❌ Anti-Pattern 3: Forgetting Suspense Boundary

```javascript
// WRONG - Will crash
function App({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}

// RIGHT - Wrap with Suspense
function App({ dataPromise }) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <DataComponent dataPromise={dataPromise} />
    </Suspense>
  );
}
```

### ❌ Anti-Pattern 4: Using for Side Effects

```javascript
// WRONG - use() is not for side effects
function Component() {
  use(Promise.resolve().then(() => {
    console.log('Side effect');
  }));
}

// RIGHT - Use useEffect
function Component() {
  useEffect(() => {
    console.log('Side effect');
  }, []);
}
```

---

## Error Handling

### Error Thrown by use()

```javascript
'use client';

import { use } from 'react';

// Error Boundary catches this
class MyErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // error: the rejected value from promise
    // errorInfo: contains componentStack
    console.error('Caught error:', error);
  }
  
  render() {
    if (this.state?.error) {
      return <div>Something went wrong: {this.state.error.message}</div>;
    }
    return this.props.children;
  }
}

function UserComponent({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

export default function App({ userPromise }) {
  return (
    <MyErrorBoundary>
      <UserComponent userPromise={userPromise} />
    </MyErrorBoundary>
  );
}
```

### Handling Promise Rejection Gracefully

```javascript
function createSafePromise(promise) {
  return promise.catch(error => {
    // Transform error for cleaner handling
    return {
      isError: true,
      message: error.message,
      statusCode: error.status
    };
  });
}

function Component({ dataPromise }) {
  const result = use(createSafePromise(dataPromise));
  
  if (result.isError) {
    return <div>Error: {result.message}</div>;
  }
  
  return <div>{result.name}</div>;
}
```

---

## Performance Tips

### Tip 1: Avoid Recreating Promises

```javascript
// ❌ Bad - Creates new promise on each render
function Component() {
  const data = use(fetchData());
}

// ✅ Good - Cache at component level
function Component() {
  const dataPromise = useMemo(() => fetchData(), []);
  const data = use(dataPromise);
}

// ✅ Better - Cache at page/layout level
export default function Page() {
  const dataPromise = fetchData(); // Created once
  return <Component dataPromise={dataPromise} />;
}
```

### Tip 2: Granular Suspense Boundaries

```javascript
// ✅ Each section loads independently
export default function Dashboard() {
  return (
    <div>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
      
      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>
    </div>
  );
}
```

### Tip 3: Parallel Resource Loading

```javascript
// ✅ Load multiple resources in parallel
export default function App() {
  // All promises start immediately
  const userPromise = fetchUser();
  const postsPromise = fetchPosts();
  const commentsPromise = fetchComments();
  
  return (
    <Suspense fallback={<Loading />}>
      <Dashboard 
        userPromise={userPromise}
        postsPromise={postsPromise}
        commentsPromise={commentsPromise}
      />
    </Suspense>
  );
}

function Dashboard({ userPromise, postsPromise, commentsPromise }) {
  // All resolve in parallel, rendered together
  const user = use(userPromise);
  const posts = use(postsPromise);
  const comments = use(commentsPromise);
  
  return <div>{/* render */}</div>;
}
```

### Tip 4: React 19 Sibling Pre-warming

React 19 automatically pre-warms sibling components while the Suspense fallback is displayed, improving perceived performance.

```javascript
// React handles this automatically - just structure your components well
<Suspense fallback={<Fallback />}>
  <ComponentA /> {/* Suspends first */}
  <ComponentB /> {/* Pre-warms in background */}
</Suspense>
```

---

## Migration Guide

### From useState + useEffect

**Before:**
```javascript
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetchUsers()
      .then(setUsers)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

**After:**
```javascript
function UserList({ usersPromise }) {
  const users = use(usersPromise);
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

export default function App() {
  return (
    <ErrorBoundary fallback={<div>Error loading users</div>}>
      <Suspense fallback={<div>Loading...</div>}>
        <UserList usersPromise={fetchUsers()} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### From useContext

**Before:**
```javascript
function Component() {
  const theme = useContext(ThemeContext);
  return <div style={{ color: theme.color }}>{/* ... */}</div>;
}
```

**After:**
```javascript
function Component() {
  const theme = use(ThemeContext);
  return <div style={{ color: theme.color }}>{/* ... */}</div>;
}

// Or conditionally:
function Component({ showTheme }) {
  let theme;
  if (showTheme) {
    theme = use(ThemeContext); // ✅ Now legal!
  }
  return theme ? <div>...</div> : null;
}
```

---

## API Reference

### Function Signature

```typescript
function use<T>(usable: Usable<T>): T
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `usable` | `Thenable<T> \| ReactContext<T>` | A Promise-like object or React Context |

### Returns

| Type | Description |
|------|-------------|
| `T` | The resolved value from the promise/context |

### Throws

| Case | Error |
|------|-------|
| Promise rejected | The rejection reason |
| Promise pending | Internal error (triggers Suspense) |
| Invalid hook call | "Invalid hook call" error |

### Behavior by Thenable Status

```javascript
const promise = /* ... */;

// If fulfilled
const value = use(promise); // ✅ Returns value immediately

// If pending
const value = use(promise); // ⏸️ Suspends component, shows Suspense fallback

// If rejected
const value = use(promise); // ❌ Throws error to nearest Error Boundary
```

---

## Rules of Hooks (Updated for React 19)

The `use` hook follows standard React hook rules:

1. ✅ Call at component top level
2. ✅ Call conditionally in render phase
3. ❌ Don't call in loops
4. ❌ Don't call in event handlers directly (call in action callbacks)
5. ❌ Don't call after return statement

**Special:** Unlike other hooks, `use` CAN be called conditionally:

```javascript
// ✅ Legal with use
function Component({ shouldFetch }) {
  if (shouldFetch) {
    const data = use(fetchData()); // ✅ OK!
    return <div>{data}</div>;
  }
  return null;
}

// ❌ Illegal with other hooks
function Component({ shouldFetch }) {
  if (shouldFetch) {
    const [data, setData] = useState(null); // ❌ Not OK!
  }
}
```

---

## Related React 19 APIs

### useOptimistic

```javascript
const [optimistic, addOptimistic] = useOptimistic(
  actual,
  (state, newOptimistic) => newState
);
```

Often used together:
```javascript
function Form({ dataPromise }) {
  const data = use(dataPromise);
  const [optimisticData, updateOptimistic] = useOptimistic(data);
  // ...
}
```

### useActionState

```javascript
const [state, action, pending] = useActionState(actionFunction, initialState);
```

Works with promises:
```javascript
async function updateUser(prevState, formData) {
  const result = await api.updateUser(formData);
  return result;
}

function Form({ userPromise }) {
  const user = use(userPromise);
  const [state, action, pending] = useActionState(updateUser, user);
  // ...
}
```

### useTransition

```javascript
const [isPending, startTransition] = useTransition();
```

Track promise resolution:
```javascript
function Component({ dataPromise }) {
  const [isPending, startTransition] = useTransition();
  const data = use(dataPromise);
  // isPending indicates if transition is in progress
}
```

---

## Browser & Environment Support

- **React Version:** 19.0.0 and later
- **Promise Support:** Requires Promise or Thenable support
- **Browser:** All modern browsers (Promise is widely supported)
- **Server Components:** Full support (primary use case)
- **Client Components:** Full support
- **Server-side Rendering:** Full support

---

## Debugging

### DevTools Integration

React DevTools shows:
- Promise states (pending/fulfilled/rejected)
- When Suspense boundaries are triggered
- Component re-render timing related to promise resolution

### Console Warnings

```javascript
// ⚠️ Warning: Invalid hook call
use(promise) // Outside component

// ⚠️ Warning: Missing Suspense boundary
// Component suspending but no Suspense boundary found
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Invalid hook call" | Called outside component | Move into component function |
| "Unexpected error" | Promise rejected | Add Error Boundary |
| "Nothing rendered" | Forgotten Suspense | Add `<Suspense>` wrapper |
| Infinite loading | Promise recreated every render | Memoize promise |

---

## Examples & Recipes

### Recipe 1: Data Fetching from URL Params

```javascript
// app/posts/[id]/page.jsx
import Post from '@/components/Post';

async function getPost(id) {
  const res = await fetch(`/api/posts/${id}`);
  if (!res.ok) throw new Error('Failed to fetch');
  return res.json();
}

export default function Page({ params }) {
  const postPromise = getPost(params.id);
  
  return (
    <Suspense fallback={<PostSkeleton />}>
      <Post postPromise={postPromise} />
    </Suspense>
  );
}

// components/Post.jsx
'use client';

import { use } from 'react';

export default function Post({ postPromise }) {
  const post = use(postPromise);
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <time>{new Date(post.publishedAt).toLocaleDateString()}</time>
    </article>
  );
}
```

### Recipe 2: Lazy Feature Based on User Permission

```javascript
'use client';

import { use, Suspense } from 'react';

function AdminPanel({ adminPromise }) {
  // Only render if user is admin
  const admin = use(adminPromise);
  
  return admin ? <div>Admin controls</div> : <div>No access</div>;
}

export default function App({ userPromise }) {
  return (
    <Suspense fallback={<div>Checking permissions...</div>}>
      <AdminPanel adminPromise={checkAdmin()} />
    </Suspense>
  );
}
```

### Recipe 3: Dependent Resource Loading

```javascript
// ✅ Load user first, then user's posts
export default function UserPosts({ userId }) {
  const userPromise = fetchUser(userId);
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserSection userPromise={userPromise} />
    </Suspense>
  );
}

function UserSection({ userPromise }) {
  const user = use(userPromise);
  const postsPromise = fetchPosts(user.id);
  
  return (
    <div>
      <h1>{user.name}</h1>
      <Suspense fallback={<PostsSkeleton />}>
        <Posts postsPromise={postsPromise} />
      </Suspense>
    </div>
  );
}

function Posts({ postsPromise }) {
  const posts = use(postsPromise);
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

---

## Summary Checklist

Before using `use`:

- [ ] Promise or Context being passed as prop
- [ ] Component wrapped in Suspense boundary
- [ ] Error Boundary in place for error handling
- [ ] Promise memoized or created at page/layout level
- [ ] Not recreating promise on every render
- [ ] Using in client or server component appropriately

Common mistakes to avoid:

- [ ] Creating new promise every render
- [ ] Forgetting Suspense wrapper
- [ ] No Error Boundary for errors
- [ ] Using for state that changes frequently
- [ ] Calling outside component function

---

## References & Links

**Source Code:**
- `packages/react/src/ReactHooks.js` (lines 207-209)
- `packages/shared/ReactTypes.js` (line 151)

**Related Files:**
- React Reconciler implementation (Fiber)
- Suspense boundary implementation
- Error boundary component

**React 19 Release:**
- **Date:** December 5, 2024
- **Version:** 19.0.0
- **PR:** [#25084](https://github.com/facebook/react/pull/25084) and related

**Official Documentation:**
- https://react.dev/reference/react/use
- https://react.dev/blog/2024/04/25/react-19

**Learning Resources:**
- React 19 upgrade guide
- Server Components documentation
- Suspense and Error Boundaries guide

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 19.0.0 | Dec 5, 2024 | Initial release of `use` hook |
| 19.1.0 | Mar 28, 2025 | Enhanced Suspense support |
| 19.2.0 | Oct 1, 2025 | Bug fixes for `use` inside lazy components |

---

**Last Updated:** React 19.2.0  
**Status:** Stable ✅  
**Recommended:** Yes - Primary pattern for Server Components
```

---

## 📝 How to Use This Guide for Code Agents

This skill.md file is optimized for code agents by including:

1. **Quick Reference Section** - Immediate answers at the top
2. **Structured Categories** - Easy navigation and searching
3. **Code Examples** - Real-world usage patterns with comments
4. **Decision Tables** - When to use vs. when not to use
5. **Location Guide** - Where in codebase to use `use`
6. **Anti-Patterns** - Common mistakes highlighted with ❌
7. **Source References** - Exact file paths and line numbers
8. **Type Signatures** - Complete TypeScript definitions
9. **Error Handling** - How errors flow through the system
10. **Performance Tips** - Optimization strategies

## 🎯 Key Takeaways for Code Agents

- **Primary Use:** Server → Client data fetching with React 19
- **Key Advantage:** Conditional context reading (first hook to support this)
- **Integration:** Works with Suspense + Error Boundaries
- **Location:** `packages/react/src/ReactHooks.js:207-209`
- **Types:** `Usable<T> = Thenable<T> | ReactContext<T>`
- **Must Wrap:** Always use with `<Suspense>` boundary

