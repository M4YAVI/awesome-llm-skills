# React 19 Server Components - AI Skill Guide

## 1. CORE CONCEPTS (Boundaries First)

### Client vs Server Component Execution
- **Server Components**: Execute ONLY on server. No browser JavaScript.
  - Can access databases, APIs, secrets directly
  - Cannot use hooks (useState, useEffect, Context) 
  - Must be async-capable (can await)
  - Default export if not marked 'use client'
  
- **Client Components**: Execute in browser + hydration on server SSR.
  - Can use all hooks and interactivity
  - MUST have 'use client' directive at file top
  - Limited to serializable data input
  - Import from 'react' not 'react/server'

### Serialization Rules (CRITICAL)
Only these types cross the network:
- ✅ Primitives: string, number, boolean, null, undefined, bigint
- ✅ Serializable objects: plain {...}, arrays []
- ✅ Date, Map, Set, TypedArray (with custom serializers)
- ✅ React Elements (JSX)
- ✅ Server Functions (marked 'use server')
- ✅ Promises (streaming support)
- ❌ Functions (closures, event handlers) → wrap in Server Actions
- ❌ Class instances → serialize to plain objects
- ❌ Symbols, WeakMaps, circular refs

**Pattern**: Server → Client needs serialization. Client → Server goes through Actions.

## 2. ARCHITECTURE PATTERNS

### 2.1 Server Actions ('use server')
```js
// file: actions.ts
'use server';

import {db} from '@/lib/db'; // Server-only

export async function createTodo(formData: FormData) {
  const title = formData.get('title');
  await db.todos.insert({title, createdAt: new Date()});
}

// Server Component can import and use directly:
export default async function TodoForm() {
  return (
    <form action={createTodo}>
      <input name="title" required />
      <button type="submit">Add</button>
    </form>
  );
}

Async only, no return value constraints
Formulas: FormData input OR use 'use client' wrapper with callback
Runs on server, no bundle footprint
Automatic CSRF protection with forms
2.2 Server Functions vs Actions



Feature	Server Component	Server Action
Invoked	During render	On user interaction
Input	Props (serialized)	FormData or encoded args
State change	Re-render tree	Revalidate + refresh page/segment
Use case	Data fetch, layout	Mutations, handlers
2.3 Data Fetching Patterns

js

// CORRECT: Server Component directly fetches
export default async function UserProfile({userId}) {
  const user = await fetch(`/api/users/${userId}`).then(r => r.json());
  return <div>{user.name}</div>;
}

// CORRECT: Fetch at render time (streaming, no waterfall)
async function Header() {
  const config = await fetchConfig();
  return <header>{config.title}</header>;
}

async function Content() {
  const data = await fetchContent(); // Parallel with Header, not waterfall
  return <main>{data}</main>;
}

export default function Page() {
  return <>
    <Suspense fallback={<div>Loading header...</div>}>
      <Header />
    </Suspense>
    <Suspense fallback={<div>Loading content...</div>}>
      <Content />
    </Suspense>
  </>;
}

// AVOID: Client-side data fetching (delays hydration)
'use client';
export default function Page() {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);
  }, []);
  return data ? <div>{data}</div> : <div>Loading...</div>;
}

3. SERIALIZATION IN DEPTH

3.1 Passing Data Server → Client

js

// Server Component
import {ClientComponent} from './client';

async function ServerComponent() {
  const user = await db.getUser(1);
  
  // ✅ Safe: primitive + serializable
  return <ClientComponent 
    userId={user.id} 
    name={user.name}
    metadata={{role: user.role}}
  />;
  
  // ❌ Unsafe: function, Date objects, db connection
  return <ClientComponent 
    user={user} // has methods!
    now={new Date()} // not serializable
    dbConnection={db} // absolutely not
  />;
}

3.2 Passing Data Client → Server

js

// client.tsx
'use client';
import {createTodo} from './actions';

export function Form() {
  return (
    <form action={async (formData) => {
      const result = await createTodo(formData);
      console.log(result); // Server Action return
    }}>
      <input name="text" />
      <button type="submit">Create</button>
    </form>
  );
}

Key: Client passes FormData or primitives. Server Action receives and responds.

4. AVOIDING ACCIDENTAL CLIENT BUNDLES

4.1 The Problem

js

// ❌ BAD: Accidentally bundles entire SDK
import {db} from '@/lib/db'; // Node.js module!

export default async function Page() {
  const data = await db.query(...);
  return <div>{data}</div>;
}
// ^ This file has NO 'use client' but db SDK gets in bundle!
// Bundler can't know db is server-only until runtime.

4.2 The Solution: Explicit Server-Only

js

// ✅ CORRECT: Mark server-only code explicitly
import 'server-only'; // npm package or local marker
import {db} from '@/lib/db';

export default async function Page() {
  const data = await db.query(...);
  return <div>{data}</div>;
}

4.3 Bundler Configuration

js

// webpack/turbopack config
serverComponentsExternals: ['server-only', 'database', 'secret-lib'],
// These packages won't be bundled for client

4.4 Import Patterns

js

// ✅ Export server function, import in Server Component
// file: lib/db.server.ts
export async function getUser(id) { /* */ }

// file: app/page.tsx (Server Component)
import {getUser} from '@/lib/db.server';
export default async function Page() {
  const user = await getUser(1);
  return <div>{user.name}</div>;
}

// ❌ NEVER: Import server code in Client Component
// file: components/user-profile.tsx
'use client';
import {getUser} from '@/lib/db.server'; // ❌ Will fail or bloat bundle

5. STREAMING SERVER-RENDERED UI

5.1 Progressive Streaming

js

// app/layout.tsx (Server Component)
export default function Layout({children}) {
  return (
    <html>
      <body>
        <header>Loaded immediately</header>
        <Suspense fallback={<div>Loading...</div>}>
          {children}
        </Suspense>
      </body>
    </html>
  );
}

// app/page.tsx
async function SlowData() {
  await new Promise(r => setTimeout(r, 5000)); // Simulates slow DB
  return <div>Slow content loaded</div>;
}

export default function Page() {
  return (
    <Suspense fallback={<div>Loading slow data...</div>}>
      <SlowData />
    </Suspense>
  );
}

Flow:

Browser gets HTML shell immediately
Header interactive before slow data loads
Slow component streams in, replaces fallback
No JS waterfall, parallel streaming
5.2 Streaming Mechanics

js

import {renderToReadableStream} from 'react-dom/server';

// Server (Node.js)
const stream = await renderToReadableStream(<App />);
stream.pipe(res); // Streams to browser immediately

// Browser
// Receives HTML chunks as they render
// React hydrates progressively
// JS can load and hydrate before all HTML arrives

6. USE HOOK (Unwrapping Promises)

6.1 Reading Promises in Components

js

// Server Component
async function getData() {
  const promise = fetchData(); // Returns Promise
  return <ClientComponent data={promise} />;
}

// Client Component
'use client';
import {use} from 'react';

export function ClientComponent({data}) {
  const resolved = use(data); // Unwraps Promise
  return <div>{resolved}</div>;
}

6.2 use() in Server Components

js

// Server Component: use() for async props
async function Header({config}) {
  const resolvedConfig = use(config); // If config is a Promise
  return <header>{resolvedConfig.title}</header>;
}

7. PRACTICAL PATTERNS

7.1 Isolated Server-Only Module

js

// lib/secrets.server.ts
import 'server-only';
export const API_KEY = process.env.SECRET_API_KEY;
export const db = new DatabaseClient();

// Usage in Server Component
import {API_KEY} from '@/lib/secrets.server';
export default async function Page() {
  const data = await fetch('...', {headers: {auth: API_KEY}});
  return <div>{/* ... */}</div>;
}

// This will error if imported in Client Component (good!)

7.2 Composition: Server + Client

js

// components/server-wrapper.tsx (Server Component)
import {ClientFilters} from './client-filters';

export default async function ProductList() {
  const allProducts = await db.products.findAll();
  
  return (
    <div>
      <h1>Products</h1>
      {/* Client Component handles filtering, search */}
      <ClientFilters products={allProducts} />
    </div>
  );
}

// components/client-filters.tsx
'use client';
import {useState} from 'react';

export function ClientFilters({products}) {
  const [search, setSearch] = useState('');
  const filtered = products.filter(p => p.name.includes(search));
  
  return (
    <>
      <input 
        value={search} 
        onChange={e => setSearch(e.target.value)} 
        placeholder="Search..."
      />
      <div>
        {filtered.map(p => (
          <ProductCard key={p.id} product={p} />
        ))}
      </div>
    </>
  );
}

7.3 Error Boundaries with Server Components

js

// app/error.tsx
'use client';
export default function Error({error, reset}) {
  return (
    <div>
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// App/page.tsx (Server Component)
export default async function Page() {
  const data = await riskyServerFetch(); // Can throw
  return <div>{data}</div>;
}

8. COMMON PITFALLS

Pitfall 1: Passing Functions to Client

js

// ❌ WRONG
export async function ServerComponent() {
  const handleClick = () => console.log('clicked');
  return <ClientComponent onClick={handleClick} />;
}

// ✅ CORRECT: Wrap in Server Action
export async function ServerComponent() {
  const handleClick = async () => {
    'use server';
    console.log('clicked on server');
  };
  return <form action={handleClick}><button>Click</button></form>;
}

Pitfall 2: useEffect in Server Components

js

// ❌ WRONG: Server Components don't run in browser
async function ServerComponent() {
  useEffect(() => { /* */ }, []); // No browser, no effect!
  return <div />;
}

// ✅ CORRECT: Run server-side logic at render time
async function ServerComponent() {
  const data = await fetchData(); // Runs on server
  return <div>{data}</div>;
}

Pitfall 3: Mixing Concerns

js

// ❌ WRONG: State in Server Component
export default async function Page() {
  const [count, setCount] = useState(0); // ❌ Can't use hooks!
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// ✅ CORRECT: Extract to Client Component
'use client';
export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Then use in Server Component
import Counter from './counter';
export default async function Page() {
  return <Counter />;
}

9. PERFORMANCE CHECKLIST

 Data fetching happens in Server Components (no JS waterfall)
 Client Components are leaf components (not wrapping everything)
 Suspense boundaries placed strategically (per slow fetch)
 No 'use client' at root unless necessary
 Server-only libraries use import 'server-only'
 FormData used for mutations (no serialization overhead)
 Large computations run server-side
 Images/assets optimized with <Image /> from next/image
10. DEBUGGING SIGNALS



Problem	Cause	Fix
"useEffect not available"	Using hooks in Server Component	Move to Client Component
Bundle bloat	Server code imported in Client	Add server-only marker
"Cannot serialize" error	Non-serializable data passed	Extract to Server Action
Empty page on hydration error	Props mismatch SSR/client	Check serialization
Data fetching after hydration	fetch() in useEffect	Move to Server Component
QUICK REFERENCE: Decision Tree

plaintext

Is this component interactive?
  → YES: Add 'use client' at top
  → NO: Keep as Server Component
Does it need to access databases?
  → YES: Must be Server Component
  → NO: Could be either
Are you passing callbacks to children?
  → YES: Wrap in Server Action or Client Component
  → NO: Can be Server Component
Do you need useState/useEffect?
  → YES: Must be 'use client'
  → NO: Can be Server Component

TOKEN EFFICIENCY NOTES FOR AI

Prioritize 'use client' detection first (fastest optimization)
Check serialization issues before runtime (prevent hydration mismatch)
Use Suspense + Streaming for perceived performance (client sees progress)
Server Actions replace mutation endpoints entirely
Memory: Server Components run in Node.js memory pool (reuse per request)
plaintext

This skill.md provides:
1. **Concise boundaries** - Clear client/server split without verbosity
2. **Serialization rules** - Quick reference preventing 90% of errors
3. **Practical patterns** - Real code examples, not theory
4. **Antipatterns** - What NOT to do (prevents mistakes)
5. **Performance checklist** - Actionable optimization points
6. **Decision tree** - Fast routing for AI to choose architecture
7. **Debugging signals** - Maps symptoms to causes quickly
8. **Token-optimized** - No redundancy, actionable structure
**For your AI coding agent**, this enables:
