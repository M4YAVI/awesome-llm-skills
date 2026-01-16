# React 19 Error Handling & Boundaries - AI Skill Guide

## 1. ERROR BOUNDARIES (CLASS COMPONENTS)

### 1.1 What are Error Boundaries?

**Purpose**: Catch JavaScript errors anywhere in component tree, log errors, display fallback UI.

**Key Points:**
- Only work with **class components** (not hooks)
- Catch errors in render, lifecycle methods, constructors
- Do NOT catch: event handlers, async code, server-side rendering
- One boundary can wrap multiple subtrees

### 1.2 Basic Error Boundary
```js
// ✅ CORRECT: Basic error boundary
import React from 'react';

export default class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {error: null, errorInfo: null};
  }

  static getDerivedStateFromError(error) {
    // Update state so next render shows fallback UI
    return {error};
  }

  componentDidCatch(error, errorInfo) {
    // Log error details for debugging
    console.error('Error caught by boundary:', error, errorInfo);
    this.setState({errorInfo});
  }

  render() {
    if (this.state.error) {
      return (
        <div>
          <h1>Something went wrong</h1>
          <details>
            {this.state.error?.toString()}
            <pre>{this.state.errorInfo?.componentStack}</pre>
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <PotentiallyBrokenComponent />
</ErrorBoundary>

1.3 What Error Boundaries DON'T Catch

js

// ❌ Event handlers: Error boundaries don't catch these
function Component() {
  return (
    <button onClick={() => {
      throw new Error('Oops'); // NOT caught by boundary
    }}>
      Click me
    </button>
  );
}

// ✅ CORRECT: Use try/catch in event handlers
function Component() {
  const handleClick = () => {
    try {
      throw new Error('Oops');
    } catch (err) {
      // Handle error
      console.error(err);
    }
  };
  return <button onClick={handleClick}>Click me</button>;
}

// ❌ Async errors: Error boundaries don't catch
function Component() {
  useEffect(() => {
    setTimeout(() => {
      throw new Error('Async error'); // NOT caught
    }, 100);
  }, []);
}

// ✅ CORRECT: Try/catch around async
function Component() {
  useEffect(() => {
    const loadData = async () => {
      try {
        await fetchData();
      } catch (err) {
        setError(err);
      }
    };
    loadData();
  }, []);
}

1.4 Granular Error Boundaries

js

// ✅ PATTERN: Multiple boundaries for different concerns
export default function App() {
  return (
    <div>
      <ErrorBoundary>
        <Header />
      </ErrorBoundary>
      
      <ErrorBoundary>
        <Sidebar />
      </ErrorBoundary>
      
      <ErrorBoundary>
        <MainContent />
      </ErrorBoundary>
    </div>
  );
}

// BENEFIT: If Header breaks, Sidebar and MainContent still render
// Without granular boundaries, entire app crashes

1.5 React 19: Error Recovery with reset()

js

// ✅ NEW React 19: Error boundary with reset capability
export class ErrorBoundary extends React.Component {
  state = {error: null, hasError: false};

  static getDerivedStateFromError(error) {
    return {hasError: true, error};
  }

  reset = () => {
    this.setState({hasError: false, error: null});
  };

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h1>Error: {this.state.error?.message}</h1>
          <button onClick={this.reset}>Try again</button>
        </div>
      );
    }

    return this.props.children;
  }
}

2. ERROR BOUNDARIES WITH FALLBACK UI (REACT 19)

2.1 Using Error Boundary Component

js

// ✅ PATTERN: Reusable error boundary with fallback
function ErrorBoundaryFallback({error, resetError}) {
  return (
    <div style={{padding: '20px', border: '1px solid red'}}>
      <h2>⚠️ Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={resetError}>Try again</button>
    </div>
  );
}

export class ErrorBoundary extends React.Component {
  state = {error: null};

  static getDerivedStateFromError(error) {
    return {error};
  }

  resetError = () => {
    this.setState({error: null});
  };

  render() {
    if (this.state.error) {
      return (
        <ErrorBoundaryFallback
          error={this.state.error}
          resetError={this.resetError}
        />
      );
    }

    return this.props.children;
  }
}

2.2 Conditional Error UI by Severity

js

// ✅ PATTERN: Different UI based on error type
class ErrorBoundary extends React.Component {
  state = {error: null};

  static getDerivedStateFromError(error) {
    return {error};
  }

  render() {
    const {error} = this.state;

    if (!error) return this.props.children;

    // Different UI based on error
    if (error.message.includes('Network')) {
      return (
        <div>
          <h2>🌐 Connection Error</h2>
          <p>Please check your internet connection</p>
        </div>
      );
    }

    if (error.message.includes('404')) {
      return (
        <div>
          <h2>📄 Not Found</h2>
          <p>The page you're looking for doesn't exist</p>
        </div>
      );
    }

    return (
      <div>
        <h2>❌ Unknown Error</h2>
        <p>{error.message}</p>
      </div>
    );
  }
}

3. TRY/CATCH IN SERVER COMPONENTS

3.1 Server Component Error Handling

js

// ✅ CORRECT: Try/catch in async Server Component
export default async function UserProfile({userId}) {
  try {
    const user = await db.getUser(userId);
    return <div>{user.name}</div>;
  } catch (error) {
    // Option 1: Return error UI
    if (error.code === 'NOT_FOUND') {
      return <div>User not found</div>;
    }

    // Option 2: Throw to nearest Error Boundary
    throw error;
  }
}

// ✅ CORRECT: Wrap with Error Boundary
<ErrorBoundary>
  <UserProfile userId={1} />
</ErrorBoundary>

3.2 Selective Error Catching

js

// ✅ PATTERN: Catch specific errors, propagate others
async function fetchUserData(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    
    if (response.status === 404) {
      return {error: 'User not found'};
    }
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    
    return await response.json();
  } catch (error) {
    if (error.code === 'ECONNREFUSED') {
      return {error: 'Server unavailable'};
    }
    throw error; // Propagate unexpected errors
  }
}

export default async function Page({userId}) {
  const data = await fetchUserData(userId);
  
  if (data.error) {
    return <div>{data.error}</div>;
  }
  
  return <div>{data.name}</div>;
}

4. ERROR RECOVERY PATTERNS

4.1 Retry Logic

js

// ✅ PATTERN: Automatic retry with exponential backoff
async function retryFetch(url, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return await response.json();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      
      const delay = Math.pow(2, attempt - 1) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

export default async function Page() {
  try {
    const data = await retryFetch('/api/data');
    return <div>{data.name}</div>;
  } catch (error) {
    return <div>Failed to load: {error.message}</div>;
  }
}

4.2 Fallback Data

js

// ✅ PATTERN: Use cached/fallback data on error
async function getUserWithFallback(userId) {
  try {
    return await fetch(`/api/users/${userId}`).then(r => r.json());
  } catch (error) {
    console.error('Failed to fetch user:', error);
    // Return default data instead of crashing
    return {
      id: userId,
      name: 'Unknown User',
      avatar: '/default-avatar.png',
    };
  }
}

export default async function Profile({userId}) {
  const user = await getUserWithFallback(userId);
  return <div>{user.name}</div>; // Always renders
}

4.3 Graceful Degradation

js

// ✅ PATTERN: Show partial content, hide failed sections
export default async function Page() {
  let recommendation;
  try {
    recommendation = await fetchRecommendations();
  } catch (error) {
    recommendation = null; // Don't crash whole page
  }

  return (
    <div>
      <Header /> {/* Always renders */}
      {recommendation && <RecommendationSection data={recommendation} />}
      <Footer /> {/* Always renders */}
    </div>
  );
}

5. ASYNC ERROR HANDLING

5.1 Promise Rejection Handling

js

// ❌ WRONG: Unhandled promise rejection
function Component() {
  fetch('/api/data').then(r => r.json()); // No error handling!
  return <div>Loading...</div>;
}

// ✅ CORRECT: Catch promise rejections
function Component() {
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/data')
      .then(r => r.json())
      .catch(err => {
        console.error(err);
        setError(err.message);
      });
  }, []);

  if (error) return <div>Error: {error}</div>;
  return <div>Loading...</div>;
}

5.2 Async/Await Error Handling

js

// ✅ CORRECT: Try/catch with async/await
function Component() {
  const [error, setError] = useState(null);

  useEffect(() => {
    const loadData = async () => {
      try {
        const response = await fetch('/api/data');
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const data = await response.json();
        setData(data);
      } catch (err) {
        setError(err.message);
      }
    };

    loadData();
  }, []);

  if (error) return <div>Error: {error}</div>;
  return <div>{/* content */}</div>;
}

5.3 useActionState Error Handling (React 19)

js

// ✅ NEW React 19: useActionState with error handling
'use client';
import {useActionState} from 'react';

function Form() {
  const [state, formAction, isPending] = useActionState(
    async (prevState, formData) => {
      try {
        const result = await serverAction(formData);
        return {success: true, data: result};
      } catch (error) {
        return {success: false, error: error.message};
      }
    },
    {success: false, error: null}
  );

  return (
    <form action={formAction}>
      <input name="email" />
      <button disabled={isPending}>Submit</button>
      {state.error && <div style={{color: 'red'}}>{state.error}</div>}
      {state.success && <div>Success!</div>}
    </form>
  );
}

6. 404 VS 500 HANDLING

6.1 HTTP Status-Based Error Handling

js

// ✅ PATTERN: Route errors by status code
async function handlePageRequest(pathname) {
  try {
    const page = await fetchPage(pathname);
    return renderPage(page);
  } catch (error) {
    if (error.status === 404) {
      return render404Page();
    }
    if (error.status === 500) {
      return render500Page();
    }
    throw error; // Unknown error
  }
}

function render404Page() {
  return (
    <html>
      <body>
        <h1>404: Page Not Found</h1>
        <p>The page you're looking for doesn't exist.</p>
        <a href="/">Back to home</a>
      </body>
    </html>
  );
}

function render500Page() {
  return (
    <html>
      <body>
        <h1>500: Server Error</h1>
        <p>Something went wrong on our end.</p>
        <a href="/">Back to home</a>
      </body>
    </html>
  );
}

6.2 Next.js Style Error Boundaries

js

// ✅ PATTERN: error.tsx for 5xx, not-found.tsx for 404
// app/error.tsx (catches 5xx)
'use client';
export default function Error({error, reset}) {
  return (
    <div>
      <h2>500 - Server Error</h2>
      <p>{error?.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/not-found.tsx (catches 404)
export default function NotFound() {
  return (
    <div>
      <h2>404 - Not Found</h2>
      <p>This page doesn't exist</p>
      <a href="/">Back home</a>
    </div>
  );
}

// app/page.tsx (Server Component)
export default async function Page({params}) {
  const post = await db.posts.findBySlug(params.slug);
  
  if (!post) {
    notFound(); // Triggers not-found.tsx
  }
  
  return <article>{post.content}</article>;
}

6.3 Custom Error Pages

js

// ✅ PATTERN: Custom error UI by type
class AppErrorBoundary extends React.Component {
  state = {error: null};

  static getDerivedStateFromError(error) {
    return {error};
  }

  render() {
    const {error} = this.state;

    if (!error) return this.props.children;

    const status = error.status || 500;
    const isDev = process.env.NODE_ENV === 'development';

    return (
      <html>
        <body>
          <div style={{padding: '40px', textAlign: 'center'}}>
            <h1>{status}</h1>
            <p>{error.message}</p>
            {isDev && (
              <pre style={{
                background: '#f5f5f5',
                padding: '10px',
                textAlign: 'left',
              }}>
                {error.stack}
              </pre>
            )}
            <a href="/">Back home</a>
          </div>
        </body>
      </html>
    );
  }
}

7. USER-FRIENDLY ERROR MESSAGES

7.1 Error Message Best Practices

js

// ❌ WRONG: Technical error messages
throw new Error('ECONNREFUSED: Connection refused 127.0.0.1:5432');

// ✅ CORRECT: User-friendly messages
throw new Error('Unable to connect to the database. Please try again later.');

// ❌ WRONG: Generic errors
throw new Error('Error');

// ✅ CORRECT: Specific, actionable errors
if (!email) throw new Error('Email is required');
if (!isValidEmail(email)) throw new Error('Email format is invalid');
if (existingUser) throw new Error('This email is already registered');

7.2 Error Mapping

js

// ✅ PATTERN: Map technical errors to user messages
const ERROR_MESSAGES = {
  ECONNREFUSED: 'Server is not responding. Please try again later.',
  ENOTFOUND: 'Network connection failed. Check your internet.',
  TIMEOUT: 'Request took too long. Please try again.',
  UNAUTHORIZED: 'You must be logged in to perform this action.',
  FORBIDDEN: 'You don\'t have permission to do this.',
  NOT_FOUND: 'The resource you\'re looking for doesn\'t exist.',
  VALIDATION_ERROR: 'Please check your input and try again.',
};

function getUserMessage(error) {
  return ERROR_MESSAGES[error.code] || 'Something went wrong. Please try again.';
}

function ErrorDisplay({error}) {
  return (
    <div style={{padding: '10px', background: '#fee', color: '#c00'}}>
      {getUserMessage(error)}
    </div>
  );
}

7.3 Contextual Error UI

js

// ✅ PATTERN: Different error UI for different contexts
function Form({onSubmit}) {
  const [error, setError] = useState(null);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    setError(null);

    try {
      await onSubmit(new FormData(e.target));
    } catch (err) {
      setError(err.message);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {error && (
        <div style={{
          padding: '10px',
          background: '#fee',
          border: '1px solid #fcc',
          color: '#c00',
          marginBottom: '10px',
          borderRadius: '4px',
        }}>
          ⚠️ {error}
        </div>
      )}
      {/* form fields */}
      <button disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}

8. ERROR LOGGING & REPORTING

8.1 Error Logging Service Integration

js

// ✅ PATTERN: Send errors to logging service
import * as Sentry from "@sentry/react";

// Initialize
Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});

// Log in error boundary
class ErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    Sentry.captureException(error, {
      contexts: {react: errorInfo},
    });
  }

  render() {
    if (this.state.error) {
      return <Sentry.ErrorBoundary fallback={<ErrorFallback />} />;
    }
    return this.props.children;
  }
}

8.2 Custom Error Tracking

js

// ✅ PATTERN: Simple error logging
const errorLog = [];

function logError(error, context) {
  const entry = {
    timestamp: new Date().toISOString(),
    message: error.message,
    stack: error.stack,
    context,
    userAgent: navigator.userAgent,
  };

  errorLog.push(entry);

  // Send to server
  if (errorLog.length >= 5) {
    sendErrorsToServer(errorLog);
    errorLog.length = 0;
  }
}

async function sendErrorsToServer(errors) {
  try {
    await fetch('/api/errors', {
      method: 'POST',
      body: JSON.stringify(errors),
    });
  } catch (err) {
    console.error('Failed to send error log:', err);
  }
}

// Usage in error boundary
class ErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    logError(error, {
      type: 'render',
      componentStack: errorInfo.componentStack,
    });
  }
}

8.3 Error Reporting in Server Components

js

// ✅ PATTERN: Log errors server-side
import {logger} from '@/lib/logger';

export default async function Page({params}) {
  try {
    const data = await fetchData(params.id);
    return <div>{data}</div>;
  } catch (error) {
    logger.error('Page load failed', {
      path: params,
      error: error.message,
      stack: error.stack,
      timestamp: new Date().toISOString(),
    });

    // Return error UI instead of throwing
    return <ErrorPage message="Failed to load page" />;
  }
}

9. COMMON ERROR PATTERNS



Error	Cause	Prevention
"null is not an object"	Accessing property on undefined	Type check before access
"Cannot read property 'map'"	Array method on non-array	Check Array.isArray()
Unhandled promise rejection	Missing .catch()	Always catch async errors
Hydration mismatch	Different server/client render	Follow hydration rules
"Error boundary error"	Error in error boundary itself	Wrap with nested boundary
10. ERROR HANDLING CHECKLIST

 Error boundary wraps user-facing components
 Event handlers have try/catch
 Async operations caught with .catch() or try/catch
 Server components handle errors gracefully
 404 vs 5xx errors treated differently
 Error messages are user-friendly
 Errors logged to monitoring service
 Fallback UI provided for all error states
 Error recovery paths available (retry, reset)
 Development vs production error detail levels
QUICK REFERENCE

plaintext

Error Type          | Where Caught | Solution
────────────────────────────────────────────────
Render error        | Error Boundary | getDerivedStateFromError
Event handler error | try/catch | Handle in onClick
Async error         | .catch() or try/catch | Promise chain
Server fetch error  | try/catch in useEffect | Retry + fallback
Server component    | try/catch or throw | Error boundary
Unhandled promise   | window.onunhandledrejection | Monitor + log

This guide emphasizes React 19 patterns (useActionState error handling), practical recovery strategies, and user-centric error messaging.

plaintext

This skill.md provides:
✅ **Error Boundaries**: Class-based implementation, what they catch/don't catch
✅ **Fallback UI**: Conditional rendering, severity-based UI
✅ **Server Component Errors**: Try/catch patterns, error propagation
✅ **Error Recovery**: Retry logic, fallback data, graceful degradation
✅ **Async Handling**: Promise chains, async/await, useActionState (React 19)
✅ **404 vs 500**: Status-code routing, Next.js patterns
✅ **User-Friendly Messages**: Error mapping, contextual UI
✅ **Logging & Reporting**: Sentry integration, custom tracking, server-side logging
✅ **Common Patterns**: Quick reference table
