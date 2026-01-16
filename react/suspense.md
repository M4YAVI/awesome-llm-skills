# React 19 Suspense Mastery Skill Guide

## Core Concepts

### What is Suspense?
Suspense is React's mechanism for handling async operations (data fetching, lazy loading) declaratively. It pauses component rendering until a Promise resolves, showing a fallback UI meanwhile.

**Key States:**
- **Pending**: Component is fetching data → show fallback
- **Resolved**: Data ready → render component
- **Rejected**: Error occurred → propagate to Error Boundary

---

## 1. Suspense Basics

### Pattern: Simple Suspense Boundary
```jsx
<Suspense fallback={<LoadingSpinner />}>
  <UserProfile userId={123} />
</Suspense>

Rules:

Wrap async components with <Suspense>
Provide fallback prop (required)
Fallback must be a valid React element
Only catches Promises thrown during render phase
2. Streaming Rendering

Pattern: Enable Streaming with React Server Components (RSC)

jsx

// Server Component (app.server.js)
export default async function App() {
  const data = await fetch('/api/data');
  return <Dashboard data={data} />;
}

// Client-side streaming
<Suspense fallback={<DashboardSkeleton />}>
  <App />
</Suspense>

When to Use:

SSR with Time-to-First-Byte optimization
Gradual page hydration
Large data sets delivered in chunks
Prioritize above-the-fold content
Best Practice: Wrap entire app with <Suspense> at high level for streaming to work effectively.

3. Suspense Boundaries Placement

Pattern: Progressive Enhancement with Multiple Boundaries

jsx

export function PageLayout() {
  return (
    <div>
      {/* Header loads immediately */}
      <Header />
      
      {/* Sidebar streams separately */}
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
      
      {/* Main content streams independently */}
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}

Strategic Placement:

Place boundaries at component tree leaves (deepest logical point)
Avoid wrapping entire page in single boundary
Separate concerns: header, sidebar, content each have own boundary
Allows independent streaming of different page sections
Anti-pattern:

jsx

// ❌ Single boundary loses granular loading
<Suspense fallback={<FullPageLoader />}>
  <Header />
  <Sidebar />
  <MainContent />
</Suspense>

4. Loading Fallbacks Without Flicker

Pattern: Instant Transitions with startTransition

jsx

function TabSwitcher() {
  const [tab, setTab] = useState('home');

  return (
    <>
      <button 
        onClick={() => startTransition(() => setTab('profile'))}
      >
        Profile
      </button>
      
      <Suspense fallback={<Skeleton />}>
        {tab === 'home' && <Home />}
        {tab === 'profile' && <Profile />}
      </Suspense>
    </>
  );
}

Key: startTransition prevents fallback if data loads fast (<500ms by default)

Pattern: Skeleton UI for Consistent UX

jsx

function UserCard({ userId }) {
  return (
    <Suspense fallback={<UserCardSkeleton />}>
      <ActualUserCard userId={userId} />
    </Suspense>
  );
}

function UserCardSkeleton() {
  return (
    <div className="card animate-pulse">
      <div className="h-4 bg-gray-200 rounded w-3/4"></div>
      <div className="h-4 bg-gray-200 rounded w-1/2 mt-2"></div>
    </div>
  );
}

Anti-flicker Techniques:

Match fallback layout dimensions to real content
Use skeleton screens, not generic spinners
Combine with useTransition() to keep old UI briefly
Set reasonable CSS min-height on boundaries
5. Error Boundaries with Suspense

Pattern: Combined Error Boundary + Suspense

jsx

class ErrorBoundary extends React.Component {
  state = { error: null };
  
  static getDerivedStateFromError(error) {
    return { error };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Caught error:', error, errorInfo);
  }
  
  render() {
    if (this.state.error) {
      return (
        <div className="error-box">
          <h2>Something went wrong</h2>
          <p>{this.state.error.message}</p>
          <button onClick={() => this.setState({ error: null })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <Suspense fallback={<Loader />}>
    <UserProfile />
  </Suspense>
</ErrorBoundary>

Error Propagation Flow:

Suspense catches Promise → shows fallback
Promise rejects → Error Boundary catches
Error Boundary renders error UI
Important: Error Boundaries MUST be class components. They don't catch errors thrown during Promise resolution (from Suspense), only render-time errors. For async errors, rely on Suspense + Promise rejection handling.

6. Data Fetching with Suspense

Pattern: Async Component with Use Hook (React 19)

jsx

function UserProfile({ userId }) {
  const user = use(fetchUser(userId));
  return <div>{user.name}</div>;
}

function fetchUser(id) {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

// In parent
<Suspense fallback={<UserSkeleton />}>
  <UserProfile userId={123} />
</Suspense>

vs. useEffect approach (outdated):

jsx

// ❌ Old pattern - no Suspense support
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);
  
  if (!user) return <UserSkeleton />;
  return <div>{user.name}</div>;
}

Pattern: Server Component Data Fetching

jsx

// app.server.js
export async function UserProfile({ userId }) {
  const user = await fetch(`/api/users/${userId}`);
  return <div>{user.name}</div>;
}

// Parent wraps with Suspense
<Suspense fallback={<UserSkeleton />}>
  <UserProfile userId={123} />
</Suspense>

Data Fetching Rules:

Promise must be thrown during render for Suspense to catch it
Use use() hook to unwrap Promises in Client Components
Server Components can await directly
Cache data requests to avoid refetching on re-render
7. Nested Suspense Patterns

Pattern: Multi-Level Streaming

jsx

export function Dashboard() {
  return (
    <div>
      {/* Level 1: Static content */}
      <Header />
      
      {/* Level 2: Immediate data */}
      <Suspense fallback={<SummarySkeletons />}>
        <SummaryCards />
        
        {/* Level 3: Nested Suspense for deep content */}
        <Suspense fallback={<ChartSkeleton />}>
          <AnalyticsChart />
        </Suspense>
      </Suspense>
      
      {/* Level 2: Independent section */}
      <Suspense fallback={<TableSkeleton />}>
        <DataTable />
      </Suspense>
    </div>
  );
}

Waterfall Prevention:

jsx

// ❌ WATERFALL: ChartData depends on SummaryData loading first
<Suspense fallback={<SummarySkeletons />}>
  <SummaryCards onComplete={() => <ChartData />} />
</Suspense>

// ✅ PARALLEL: Both load independently
<Suspense fallback={<SummarySkeletons />}>
  <SummaryCards />
</Suspense>
<Suspense fallback={<ChartSkeleton />}>
  <ChartData />
</Suspense>

Pattern: Suspense List for Ordered Reveal (React.unstable_SuspenseList)

jsx

import { unstable_SuspenseList as SuspenseList } from 'react';

<SuspenseList revealOrder="forwards" tail="collapsed">
  <Suspense fallback={<Skeleton />}>
    <Section1 />
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <Section2 />
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <Section3 />
  </Suspense>
</SuspenseList>

SuspenseList Options:

revealOrder="forwards" - reveal in DOM order
revealOrder="backwards" - reverse order
revealOrder="together" - wait for all
tail="collapsed" - show latest fallback only
tail="hidden" - hide fallbacks entirely
8. Advanced Patterns

Pattern: Conditional Suspense

jsx

function SearchResults({ query }) {
  // Only apply Suspense if query exists
  const content = query ? (
    <Suspense fallback={<ResultsSkeleton />}>
      <Results query={query} />
    </Suspense>
  ) : (
    <EmptyState />
  );
  
  return <div>{content}</div>;
}

Pattern: Retry Mechanism

jsx

function DataFetcher({ url, retries = 3 }) {
  const [attempt, setAttempt] = useState(0);
  
  if (attempt > retries) {
    return <Error message="Max retries exceeded" />;
  }
  
  return (
    <ErrorBoundary 
      onError={() => setAttempt(a => a + 1)}
    >
      <Suspense fallback={<Loader />}>
        <DataComponent url={url} key={attempt} />
      </Suspense>
    </ErrorBoundary>
  );
}

Pattern: Pre-loading Data

jsx

// Prefetch before render
function HomePage() {
  // Start loading immediately
  const dataPromise = fetchData();
  
  return (
    <Suspense fallback={<Skeleton />}>
      <Content dataPromise={dataPromise} />
    </Suspense>
  );
}

function Content({ dataPromise }) {
  const data = use(dataPromise); // Uses already-loading promise
  return <div>{data}</div>;
}

Key Takeaways for Code Generation

Always wrap async components with <Suspense fallback={...}>
Place boundaries strategically - not all in one place
Use startTransition to prevent fallback flicker
Combine Error Boundaries with Suspense for full error handling
Use use() hook for Client Components to handle Promises
Server Components can await directly (no hooks needed)
Avoid waterfalls - run independent Suspense boundaries in parallel
Match fallback dimensions to prevent layout shift
Leverage streaming for SSR performance
Test with network throttling to catch UX issues
Common Mistakes to Avoid



❌ Mistake	✅ Solution
Fallback causes jarring flicker	Use skeleton screens + startTransition
Single boundary wraps entire page	Use multiple focused boundaries
Missing Error Boundary	Combine with ErrorBoundary for Promise rejections
useEffect + setState for data	Use use() hook or Server Components
Waterfalls in nested Suspense	Place boundaries at same level, not nested
No key prop in list rendering	Add key={id} to prevent state leakage
plaintext

This skill guide is:
- **Token-efficient**: Focused on patterns, not verbose explanations
- **Practical**: Code-first with clear examples
- **React 19 focused**: Uses modern `use()` hook and RSC patterns
- **AI-friendly**: Structured for easy parsing by code generation agents
- **Actionable**: Clear rules and anti-patterns for decision making
Use this as a system prompt context or reference file for your code agent!
