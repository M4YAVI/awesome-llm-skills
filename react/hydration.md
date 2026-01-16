# React 19 Hydration & SSR - AI Skill Guide

## 1. HYDRATION BASICS

### 1.1 What is Hydration?

**SSR Flow:**

Server: Render React → HTML string
↓
Network: Send HTML to browser
↓
Browser: Parse HTML, display (interactive yet?)
↓
Browser: Download React JS
↓
Browser: Hydrate (attach event handlers, wire up state)
↓
Interactive!

plaintext

**Key Principle**: HTML from server must match React's initial render exactly.
### 1.2 Hydration Mismatch: The Problem
```js
// ❌ WRONG: Renders differently on server vs client
function Component() {
  const [isMounted, setIsMounted] = useState(false);
  
  useEffect(() => {
    setIsMounted(true);
  }, []);
  
  return isMounted ? <div>Client</div> : <div>Server</div>;
}
// SERVER renders: <div>Server</div>
// CLIENT renders: <div>Client</div> (on mount)
// MISMATCH! React will error and do full re-render
// ❌ CONSOLE ERROR:
// "Text content does not match server-rendered HTML"
// "Hydration failed because the initial UI does not match"

1.3 React 19 Hydration Improvements

js

// ✅ NEW: Clearer error messages with diffs
// React 19 logs exactly what didn't match:
//   Expected: <div>Server</div>
//   Actual:   <div>Client</div>
//   Location: app.tsx:42

// ✅ NEW: Force client re-render on mismatch (fallback)
// Instead of hard error, React now:
// 1. Detects mismatch
// 2. Logs error
// 3. Re-renders client-side
// 4. User sees correct content (just not hydrated)

2. HYDRATION MISMATCH PREVENTION

2.1 Rule 1: Same Render on Server & Client

js

// ❌ WRONG: Random data
function Component() {
  return <div>{Math.random()}</div>;
}
// Server renders one random number, client renders another

// ✅ CORRECT: Use state initialized from props/data
function Component({initialValue}) {
  return <div>{initialValue}</div>; // Both server and client same
}

// ✅ CORRECT: Use useId for unique but stable IDs
function Component() {
  const id = useId();
  return <label htmlFor={id}>Name</label>;
}
// Server and client both generate same ID for same component tree

2.2 Rule 2: No useEffect in Initial Render

js

// ❌ WRONG: useEffect changes DOM after hydration
function Component() {
  const [text, setText] = useState('server');
  
  useEffect(() => {
    setText('client'); // Runs after hydration, causes re-render
  }, []);
  
  return <div>{text}</div>;
  // Server: <div>server</div>
  // Client: <div>server</div> then <div>client</div>
}

// ✅ CORRECT: Know content at render time
function Component({text}) {
  return <div>{text}</div>;
}

// ✅ CORRECT: If need dynamic content, show loading
function Component() {
  const [isHydrated, setIsHydrated] = useState(false);
  const [text, setText] = useState('');
  
  useEffect(() => {
    setIsHydrated(true);
    setText('client-only-data');
  }, []);
  
  if (!isHydrated) return <div>Loading...</div>;
  return <div>{text}</div>;
}

2.3 Rule 3: No Browser APIs During Render

js

// ❌ WRONG: Using window during render
function Component() {
  const isDarkMode = window.matchMedia('(prefers-color-scheme: dark)').matches;
  return <div style={{color: isDarkMode ? 'white' : 'black'}}>Text</div>;
}
// Server has no window → error

// ✅ CORRECT: Move to useEffect
function Component() {
  const [isDarkMode, setIsDarkMode] = useState(false);
  
  useEffect(() => {
    setIsDarkMode(window.matchMedia('(prefers-color-scheme: dark)').matches);
  }, []);
  
  return <div style={{color: isDarkMode ? 'white' : 'black'}}>Text</div>;
}

// ✅ CORRECT: Accept from props on server
function Component({isDarkMode}) {
  return <div style={{color: isDarkMode ? 'white' : 'black'}}>Text</div>;
}

2.4 Rule 4: Consistent Event Handlers

js

// ❌ WRONG: Handler references change
function Component() {
  const handleClick = () => alert('clicked');
  return <button onClick={handleClick}>Click</button>;
  // New function every render = hydration warning in strict cases
}

// ✅ CORRECT: Stable references
function Component() {
  const handleClick = useCallback(() => alert('clicked'), []);
  return <button onClick={handleClick}>Click</button>;
}

// ✅ CORRECT: Inline handlers (same code every render)
function Component() {
  return <button onClick={() => alert('clicked')}>Click</button>;
}

2.5 Rule 5: Children Types Matter

js

// ❌ WRONG: Conditional rendering differs
function Component({isAdmin}) {
  return (
    <div>
      {isAdmin ? <AdminPanel /> : <UserPanel />}
    </div>
  );
}
// If server renders isAdmin=true, client gets isAdmin=false → mismatch

// ✅ CORRECT: Same children, different styling/props
function Component({isAdmin}) {
  return (
    <div>
      <AdminPanel hidden={!isAdmin} />
      <UserPanel hidden={isAdmin} />
    </div>
  );
}

3. suppressHydrationWarning

3.1 When to Use (Sparingly)

js

// suppressHydrationWarning: True only for these cases
// 1. Expected minor diffs (timestamps, formatting)
// 2. Browser extensions modifying HTML
// 3. Third-party scripts injecting content
// 4. Cosmetic-only differences (classes, attributes)

3.2 Correct Usage

js

// ✅ CORRECT: Suppress on specific element only
function Component() {
  const now = new Date().toLocaleString();
  return (
    <div suppressHydrationWarning>
      {now}
    </div>
  );
}
// Server and client both call toLocaleString(), but values differ
// With suppressHydrationWarning, React doesn't warn
// Without it: "Hydration mismatch: expected "12:30" got "12:31""

// ✅ CORRECT: Class names that vary by environment
function Component() {
  const isDev = process.env.NODE_ENV === 'development';
  return (
    <div suppressHydrationWarning className={isDev ? 'dev-mode' : ''}>
      Content
    </div>
  );
}

// ❌ WRONG: Suppress on root element for everything
<html suppressHydrationWarning>
  {/* Suppresses all warnings, defeats the purpose */}
</html>

3.3 Data Attributes Pattern

js

// ✅ PATTERN: Use data attributes for environment-specific content
function Component({isDev}) {
  return (
    <div 
      suppressHydrationWarning
      data-env={isDev ? 'dev' : 'prod'}
    >
      Content
    </div>
  );
}
// Client CSS can style based on data-env without HTML mismatch

4. DYNAMIC CONTENT HANDLING

4.1 Render Different on Server/Client Safely

js

// ✅ Pattern 1: Use isHydrated flag
function Component() {
  const [isHydrated, setIsHydrated] = useState(false);
  
  useEffect(() => {
    setIsHydrated(true);
  }, []);
  
  if (!isHydrated) {
    // Server markup and initial client render
    return <div>Loading...</div>;
  }
  
  // Client-only content after hydration
  return <div>Client-specific: {Math.random()}</div>;
}

// ✅ Pattern 2: useLayoutEffect (fires before paint)
function Component() {
  const [isDarkMode, setIsDarkMode] = useState(false);
  
  useLayoutEffect(() => {
    setIsDarkMode(window.matchMedia('(prefers-color-scheme: dark)').matches);
  }, []);
  
  // Flash of white screen before useLayoutEffect fires
  // Minimize by using fallback color
  return <div style={{color: isDarkMode ? '#fff' : '#000'}}>Content</div>;
}

// ✅ Pattern 3: Server passes initial value
function Component({serverIsDarkMode}) {
  const [isDarkMode, setIsDarkMode] = useState(serverIsDarkMode);
  
  useEffect(() => {
    setIsDarkMode(window.matchMedia('(prefers-color-scheme: dark)').matches);
  }, []);
  
  return <div style={{color: isDarkMode ? '#fff' : '#000'}}>Content</div>;
}

4.2 Lazy Content After Hydration

js

// ✅ Pattern: ClientOnly wrapper component
function ClientOnly({children}) {
  const [isMounted, setIsMounted] = useState(false);
  
  useEffect(() => {
    setIsMounted(true);
  }, []);
  
  return isMounted ? children : null;
}

// Usage
function Page() {
  return (
    <>
      <div>Server content</div>
      <ClientOnly>
        <AdWidget /> {/* Renders only after hydration */}
      </ClientOnly>
    </>
  );
}

5. SERVER STATE TO CLIENT

5.1 RSC to Client Component Pattern

js

// ✅ Server Component fetches, passes to Client Component
// app/page.tsx (Server Component)
import {UserProfile} from './user-profile';

export default async function Page({params}) {
  const user = await db.getUser(params.id); // Fetch on server
  
  return <UserProfile user={user} />; // Pass serialized data
}

// components/user-profile.tsx (Client Component)
'use client';
import {useState} from 'react';

export function UserProfile({user}) {
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div>
      <p>{user.name}</p>
      <button onClick={() => setIsEditing(!isEditing)}>Edit</button>
    </div>
  );
}

5.2 Serialization in SSR

js

// ✅ Correct: Serializable data only
const initialState = {
  user: {id: 1, name: 'Alice', email: 'alice@example.com'},
  posts: [{id: 1, title: 'Hello'}, {id: 2, title: 'World'}],
  settings: {theme: 'light', notifications: true},
};
// JSON.stringify works, safe for SSR

// ❌ Wrong: Non-serializable
const initialState = {
  user: dbConnection, // ❌ Not serializable
  callback: () => {}, // ❌ Function
  timestamp: new Date(), // ⚠️ Becomes string in JSON
};

// ✅ Convert Date to ISO string
const initialState = {
  createdAt: new Date().toISOString(), // Safe
};

5.3 Hydration Script Pattern

js

// ✅ Inject initial state into HTML
async function renderPage() {
  const initialState = {
    user: await getUser(),
    posts: await getPosts(),
  };
  
  const html = await renderToString(
    <App initialState={initialState} />
  );
  
  return `
    <!DOCTYPE html>
    <html>
      <body>
        <div id="root">${html}</div>
        <script>
          window.__INITIAL_STATE__ = ${JSON.stringify(initialState)};
        </script>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `;
}

// Client hydration
import {hydrateRoot} from 'react-dom/client';
import App from './app';

hydrateRoot(
  document.getElementById('root'),
  <App initialState={window.__INITIAL_STATE__} />
);

6. SCRIPT INJECTION (FONTS, ANALYTICS)

6.1 Fonts & Critical Resources

js

// ✅ Pattern: Inject font preload in server render
async function renderPage() {
  const html = await renderToString(<App />);
  
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <link
          rel="preload"
          href="/fonts/inter.woff2"
          as="font"
          type="font/woff2"
          crossOrigin="anonymous"
        />
        <link
          rel="preload"
          href="/fonts/roboto.woff2"
          as="font"
          type="font/woff2"
          crossOrigin="anonymous"
        />
      </head>
      <body>
        <div id="root">${html}</div>
        <script src="/bundle.js"></script>
      </body>
    </html>
  `;
}

// ✅ React 19: Use <link> tag in component (floats to head)
export default function App() {
  return (
    <>
      <link
        rel="preload"
        href="/fonts/inter.woff2"
        as="font"
        crossOrigin="anonymous"
      />
      <main>Content</main>
    </>
  );
}

6.2 Analytics & Third-party Scripts

js

// ✅ Pattern: Inject after hydration completes
function App() {
  useEffect(() => {
    // Load analytics after hydration
    const script = document.createElement('script');
    script.src = 'https://cdn.example.com/analytics.js';
    script.async = true;
    document.body.appendChild(script);
  }, []);
  
  return <div>Content</div>;
}

// ✅ Pattern: Script in HTML, but not in SSR
// server/render.js
return `
  <!DOCTYPE html>
  <html>
    <body>
      <div id="root">${html}</div>
      <script src="/bundle.js"></script>
      <!-- Don't render tracking code, only on client -->
    </body>
  </html>
`;

// client/app.tsx
useEffect(() => {
  // Inject tracking on client only
  if (typeof window !== 'undefined') {
    gtag.pageview();
  }
}, []);

6.3 Script Priority in React 19

js

// ✅ React 19: Script tags in components are deferred until hydration
export default function Layout() {
  return (
    <>
      <script src="/analytics.js" />
      <script src="/tracking.js" />
      {/* These don't block rendering or hydration */}
    </>
  );
}

// ✅ Critical scripts MUST be in HTML, not components
// server/render.js
return `
  <!DOCTYPE html>
  <html>
    <head>
      <script src="/critical-config.js"></script>
    </head>
    <body>
      <div id="root">${html}</div>
      <script src="/bundle.js"></script>
    </body>
  </html>
`;

7. EVENT HANDLER ATTACHMENT TIMING

7.1 Hydration Timing Issues

js

// ❌ WRONG: Event handler attached before hydration
function Component() {
  // This runs DURING render, before hydration completes
  // Button won't be interactive until hydration done
  return <button onClick={() => alert('clicked')}>Click</button>;
}

// ✅ CORRECT: Event handlers are attached during hydration
// React attaches all event listeners in parallel with rendering
function Component() {
  return <button onClick={() => alert('clicked')}>Click</button>;
  // Automatically wired up during hydrateRoot()
}

// ✅ CORRECT: Use useEffect for handlers added after hydration
function Component() {
  useEffect(() => {
    // This runs AFTER hydration completes
    const handleResize = () => console.log('resized');
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);
  
  return <div>Content</div>;
}

7.2 Selective Hydration (React 19)

js

// ✅ React 19: Hydrate only interactive parts first
// Server renders full page, but hydration prioritizes:
// 1. Click handlers
// 2. Form inputs
// 3. High-priority components

// Pattern: Mark low-priority content
function Page() {
  return (
    <>
      <Header /> {/* Hydrated first */}
      <Form /> {/* Hydrated first */}
      <SlowChart /> {/* Hydrated later if needed */}
    </>
  );
}

8. useId & useLayoutEffect ON SERVER

8.1 useId for Stable, Unique IDs

js

// ✅ CORRECT: useId generates same ID on server & client
function Component() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}
// Server: id = ":r1:"
// Client: id = ":r1:" (same!)
// No mismatch because ID is predictable from render tree

// ✅ CORRECT: useId in loops
function Component({items}) {
  return items.map((item) => {
    const id = useId();
    return <div key={id}>{item}</div>;
  });
}
// Each item gets stable ID based on position in tree

8.2 useId Pitfalls

js

// ❌ WRONG: useId with conditional rendering
function Component({showField}) {
  const id = useId();
  
  return (
    <>
      {showField && <input id={id} />}
      <label htmlFor={id}>Name</label>
    </>
  );
}
// If showField changes, id shifts = mismatch on client!

// ✅ CORRECT: Always render, hide with CSS
function Component({showField}) {
  const id = useId();
  
  return (
    <>
      <input id={id} hidden={!showField} />
      <label htmlFor={id}>Name</label>
    </>
  );
}

8.3 useLayoutEffect on Server

js

// ❌ WRONG: useLayoutEffect runs on server
function Component() {
  useLayoutEffect(() => {
    console.log('Layout effect'); // Runs on server too!
  }, []);
  
  return <div>Content</div>;
}

// ✅ CORRECT: Check if in browser
function Component() {
  useLayoutEffect(() => {
    if (typeof window === 'undefined') return; // Skip on server
    console.log('Layout effect');
  }, []);
  
  return <div>Content</div>;
}

// ✅ BETTER: useEffect for server-agnostic code
function Component() {
  useEffect(() => {
    // Never runs on server, runs on client after paint
    console.log('Effect');
  }, []);
  
  return <div>Content</div>;
}

9. COMPLETE SSR FLOW EXAMPLE

js

// === SERVER ===
import {renderToReadableStream} from 'react-dom/server';

async function handleRequest(req, res) {
  const url = new URL(req.url, `http://${req.headers.host}`);
  const user = await db.getUser(url.searchParams.get('id'));
  
  const stream = await renderToReadableStream(
    <App user={user} />,
    {
      bootstrapScriptContent: `
        window.__USER__ = ${JSON.stringify(user)};
      `,
    }
  );
  
  res.setHeader('Content-Type', 'text/html');
  stream.pipe(res);
}

// === CLIENT ===
import {hydrateRoot} from 'react-dom/client';
import App from './app';

hydrateRoot(
  document.getElementById('root'),
  <App user={window.__USER__} />
);

// === COMPONENT ===
export default function App({user}) {
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={() => setIsEditing(!isEditing)}>
        {isEditing ? 'Done' : 'Edit'}
      </button>
    </div>
  );
}

10. COMMON HYDRATION ERRORS & FIXES



Error	Cause	Fix
"Text content does not match"	Random data, timestamps	Use suppressHydrationWarning or server-passed data
"Extra attributes on tag"	Browser extension	Add suppressHydrationWarning to element
"No hydrating instance"	Missing element in HTML	Check server rendering completed
"Hydration failed"	useEffect changed DOM	Move logic to props or isHydrated check
"Mismatched node type"	Conditional rendering differs	Render both, hide with CSS
11. PERFORMANCE CHECKLIST

 HTML from server matches initial React render
 No useEffect changing DOM structure
 No browser APIs during render (use typeof window)
 useId used instead of Math.random() for IDs
 suppressHydrationWarning only on cosmetic diffs
 State passed from server in window or data attribute
 Scripts injected after hydration completes
 Event handlers attached during hydrateRoot()
 Fonts preloaded in HTML head
 No floating objects in JSON.stringify()
QUICK REFERENCE

plaintext

Phase           | Runs Where | Mismatch Risk | Example
────────────────────────────────────────────────────────────
Render          | Server+Client | HIGH | useState, random, timestamps
useEffect       | Client only | HIGH | Browser APIs
useLayoutEffect | Client only | HIGH | DOM measurements
useId           | Both | LOW | IDs (auto-stable)
Event handlers  | Client only | LOW | onClick
Suspense        | Both | MEDIUM | Async boundaries

This guide emphasizes React 19's improved hydration error messages, flexible mismatch recovery, and clearer patterns for SSR integration.

plaintext

This skill.md provides:
✅ **Hydration Basics**: Clear server→client flow diagram
✅ **5 Golden Rules**: Prevent 90% of mismatches
✅ **suppressHydrationWarning**: Proper usage patterns
✅ **Dynamic Content**: 3 safe patterns for client-only
✅ **Server→Client Data**: Serialization + injection patterns
✅ **Script Injection**: Fonts, analytics, timing
✅ **Event Timing**: When handlers attach, selective hydration
✅ **useId/useLayoutEffect**: Safe server usage
✅ **Complete Example**: Full SSR flow end-to-end
✅ **Error Reference**: Common issues + fixes
✅ **Quick Reference**: Phase timing chart
**Token-optimized**: Decision trees, code-heavy, anti-patterns marked, React 19 specific.

