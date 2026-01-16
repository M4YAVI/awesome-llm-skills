# React 19 Scalable Architecture & Component Design Skill Guide

## 1. Folder Structure That Scales

### Recommended Structure: Feature-Based with Clear Boundaries

src/
├── components/           # Shared UI primitives (dumb components)
│   ├── Button/
│   │   ├── Button.jsx
│   │   ├── Button.module.css
│   │   └── Button.test.jsx
│   ├── Card/
│   ├── Input/
│   └── Modal/
│
├── features/             # Feature-specific, domain-driven
│   ├── auth/
│   │   ├── components/   # Feature-specific components
│   │   │   ├── LoginForm.jsx
│   │   │   ├── SignupForm.jsx
│   │   │   └── ProfileCard.jsx
│   │   ├── hooks/        # Feature-specific hooks
│   │   │   ├── useAuth.js
│   │   │   └── useLogin.js
│   │   ├── api/          # API calls for this feature
│   │   │   └── authAPI.js
│   │   ├── contexts/     # Feature-specific context
│   │   │   └── AuthContext.js
│   │   └── index.js      # Public API (only export what's needed)
│   │
│   ├── products/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── index.js
│   │
│   └── dashboard/
│       ├── components/
│       ├── hooks/
│       ├── api/
│       └── index.js
│
├── contexts/             # Global contexts (auth, theme, user)
│   ├── AuthContext.js
│   └── ThemeContext.js
│
├── hooks/                # Global custom hooks
│   ├── useLocalStorage.js
│   ├── useFetch.js
│   └── useDebounce.js
│
├── utils/                # Pure utility functions
│   ├── formatters.js
│   ├── validators.js
│   └── constants.js
│
├── types/                # TypeScript types (if using TS)
│   ├── auth.d.ts
│   └── product.d.ts
│
├── styles/               # Global styles
│   ├── global.css
│   └── variables.css
│
└── App.jsx

plaintext

**Key Principles:**
- ✅ Each feature is **self-contained** with its own hooks, API, components
- ✅ Shared primitives live in `components/`
- ✅ Global concerns live at root level (`contexts/`, `hooks/`, `utils/`)
- ✅ Features export via `index.js` **only what's public**
- ✅ Easy to delete/move features without breaking other code
---
## 2. Component Responsibility Boundaries
### Single Responsibility Principle (SRP)
Each component should have **ONE reason to change**.
```jsx
// ✅ GOOD: Each component has ONE job
// 1. Presentation only
export function UserCard({ name, avatar, status }) {
  return (
    <div className="card">
      <img src={avatar} alt={name} />
      <h3>{name}</h3>
      <p className="status">{status}</p>
    </div>
  );
}
// 2. Logic only (custom hook)
export function useUserProfile(userId) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchUser(userId).then(setUser).finally(() => setLoading(false));
  }, [userId]);
  
  return { user, loading };
}
// 3. Container (composes logic + presentation)
export function UserProfileContainer({ userId }) {
  const { user, loading } = useUserProfile(userId);
  
  if (loading) return <Skeleton />;
  if (!user) return <Error />;
  
  return <UserCard {...user} />;
}
// ❌ BAD: Mixing too many concerns
export function UserProfileBad({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [expanded, setExpanded] = useState(false);
  const [editMode, setEditMode] = useState(false);
  
  // 50 more lines of logic...
  // Fetching, validation, editing, UI logic all mixed
}

Boundary Checklist



Concern	Component Type	Responsibility
State management	Container	Fetch data, manage logic
UI rendering	Presentational	Display props only
Event handling	Either	Container handles complex logic, Presentational delegates
Styling	Presentational	CSS modules or styled-components
API calls	Custom hook	Wrap in useEffect, memoize
3. Smart vs Dumb Components (Still Relevant in React 19)

Presentational (Dumb) Components

Characteristics:

Pure functions
Accept only props
No hooks (except forwardRef for refs)
Fully reusable
Easy to test
jsx

// ✅ Dumb component
export function Button({ 
  children, 
  variant = 'primary',
  size = 'md',
  disabled = false,
  onClick,
  className,
  ...props 
}) {
  return (
    <button
      className={`btn btn--${variant} btn--${size} ${className || ''}`}
      disabled={disabled}
      onClick={onClick}
      {...props}
    >
      {children}
    </button>
  );
}

// Test is trivial
test('Button renders with correct variant', () => {
  render(<Button variant="secondary">Click</Button>);
  expect(screen.getByRole('button')).toHaveClass('btn--secondary');
});

Container (Smart) Components

Characteristics:

Manage state & effects
Fetch data
Use custom hooks
Not directly tested with render
Handle business logic
jsx

// ✅ Smart component
export function UsersList({ teamId }) {
  const { users, loading, error } = useTeamUsers(teamId);
  const [searchTerm, setSearchTerm] = useState('');
  
  const filtered = users.filter(u => 
    u.name.toLowerCase().includes(searchTerm.toLowerCase())
  );
  
  if (loading) return <LoadingSkeleton />;
  if (error) return <ErrorMessage error={error} />;
  
  return (
    <>
      <SearchInput value={searchTerm} onChange={setSearchTerm} />
      <UserGrid users={filtered} />
    </>
  );
}

React 19 Update: Hybrid Approach with use() Hook

React 19 allows Presentational components to handle async data with use():

jsx

// Modern React 19 - Presentational with async capability
export function UserProfile({ userPromise }) {
  const user = use(userPromise); // Unwraps Promise, integrates with Suspense
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.bio}</p>
    </div>
  );
}

// Usage with Suspense (the logic layer)
function UserProfileContainer({ userId }) {
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  
  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

4. Avoiding God Components

God Component Symptoms

jsx

// ❌ BAD: 500+ line component doing everything
export function Dashboard() {
  // State for 10 different features
  const [users, setUsers] = useState([]);
  const [products, setProducts] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [filters, setFilters] = useState({});
  const [sortBy, setSortBy] = useState('date');
  const [selectedTab, setSelectedTab] = useState('overview');
  
  // Effects for 8 different data fetches
  useEffect(() => { /*...*/ }, []);
  useEffect(() => { /*...*/ }, []);
  useEffect(() => { /*...*/ }, []);
  
  // 200 lines of render logic mixed together
  return (
    <div>
      {/* Header, sidebar, tabs, tables, charts all here */}
      {/* 300 lines of JSX */}
    </div>
  );
}

Solution: Decompose Into Focused Components

jsx

// ✅ GOOD: Dashboard composes smaller, focused components

export function Dashboard() {
  const [selectedTab, setSelectedTab] = useState('overview');
  
  return (
    <div className="dashboard">
      <DashboardHeader />
      
      <div className="dashboard__content">
        <DashboardSidebar />
        
        <main className="dashboard__main">
          <Tabs 
            value={selectedTab} 
            onChange={setSelectedTab}
            tabs={[
              { id: 'overview', label: 'Overview', component: OverviewTab },
              { id: 'users', label: 'Users', component: UsersTab },
              { id: 'analytics', label: 'Analytics', component: AnalyticsTab },
            ]}
          />
        </main>
      </div>
    </div>
  );
}

// OverviewTab: Composes Overview-specific components
function OverviewTab() {
  return (
    <div className="tab-content">
      <Suspense fallback={<CardSkeleton />}>
        <StatsCards />
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>
    </div>
  );
}

// UsersTab: Independent, handles user list
function UsersTab() {
  const [search, setSearch] = useState('');
  const { users, loading } = useUserList(search);
  
  return (
    <div>
      <SearchInput value={search} onChange={setSearch} />
      {loading ? <Skeleton /> : <UserTable users={users} />}
    </div>
  );
}

God Component Prevention Checklist

❌ Component file > 300 lines?
❌ More than 5 useState calls?
❌ More than 3 useEffect calls?
❌ Multiple unrelated data fetches?
❌ Component handles 3+ major features?
Action: Extract into separate components.

5. Reusable UI Primitives

Location: src/components/

Build a component library of unstyled, composable primitives.

jsx

// src/components/Button/Button.jsx
export function Button({
  children,
  variant = 'primary',
  size = 'md',
  disabled = false,
  isLoading = false,
  startIcon,
  endIcon,
  type = 'button',
  onClick,
  asChild = false, // Radix pattern: render as different element
  ...props
}) {
  const Comp = asChild ? 'a' : 'button';
  
  return (
    <Comp
      className={`btn btn--${variant} btn--${size}`}
      disabled={disabled || isLoading}
      type={type}
      onClick={onClick}
      {...props}
    >
      {startIcon && <span className="btn__icon-start">{startIcon}</span>}
      {isLoading ? <Spinner /> : children}
      {endIcon && <span className="btn__icon-end">{endIcon}</span>}
    </Comp>
  );
}

jsx

// src/components/Card/Card.jsx
export function Card({ 
  children, 
  variant = 'default',
  padding = 'md',
  shadow = true,
  className = '',
  ...props 
}) {
  return (
    <div
      className={`card card--${variant} card--p-${padding} ${shadow ? 'card--shadow' : ''} ${className}`}
      {...props}
    >
      {children}
    </div>
  );
}

export function CardHeader({ children, className }) {
  return <div className={`card__header ${className || ''}`}>{children}</div>;
}

export function CardBody({ children, className }) {
  return <div className={`card__body ${className || ''}`}>{children}</div>;
}

export function CardFooter({ children, className }) {
  return <div className={`card__footer ${className || ''}`}>{children}</div>;
}

jsx

// src/components/Input/Input.jsx
export const Input = forwardRef(({
  label,
  error,
  hint,
  size = 'md',
  disabled = false,
  type = 'text',
  isRequired = false,
  ...props
}, ref) => {
  return (
    <div className="input-group">
      {label && (
        <label className="input-label">
          {label}
          {isRequired && <span className="required">*</span>}
        </label>
      )}
      <input
        ref={ref}
        type={type}
        disabled={disabled}
        className={`input input--${size} ${error ? 'input--error' : ''}`}
        aria-invalid={!!error}
        aria-describedby={error ? `${props.id}-error` : undefined}
        {...props}
      />
      {error && <div id={`${props.id}-error`} className="input-error">{error}</div>}
      {hint && <div className="input-hint">{hint}</div>}
    </div>
  );
});

Primitive Composition

jsx

// Compose primitives into domain components
export function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  return (
    <Card>
      <CardHeader>
        <h2>Sign In</h2>
      </CardHeader>
      <CardBody>
        <Input
          label="Email"
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          isRequired
        />
        <Input
          label="Password"
          type="password"
          value={password}
          onChange={e => setPassword(e.target.value)}
          isRequired
        />
      </CardBody>
      <CardFooter>
        <Button onClick={() => onSubmit({ email, password })}>
          Sign In
        </Button>
      </CardFooter>
    </Card>
  );
}

6. Domain-Driven Component Design

Structure Components by Domain (Business Logic)

plaintext

features/
├── auth/
│   ├── components/
│   │   ├── LoginForm.jsx
│   │   ├── SignupForm.jsx
│   │   ├── PasswordReset.jsx
│   │   ├── ProfileCard.jsx
│   │   └── index.js (public API)
│   ├── hooks/
│   │   ├── useAuth.js
│   │   ├── useLogin.js
│   │   └── usePasswordReset.js
│   ├── api/
│   │   ├── login.js
│   │   ├── signup.js
│   │   ├── logout.js
│   │   └── resetPassword.js
│   ├── contexts/
│   │   └── AuthContext.js
│   ├── types/
│   │   └── auth.d.ts
│   └── index.js (feature entry point)
│
├── products/
│   ├── components/
│   │   ├── ProductList.jsx
│   │   ├── ProductCard.jsx
│   │   ├── ProductDetail.jsx
│   │   ├── ProductFilters.jsx
│   │   └── index.js
│   ├── hooks/
│   │   ├── useProducts.js
│   │   ├── useProductDetail.js
│   │   └── useProductFilters.js
│   ├── api/
│   │   ├── getProducts.js
│   │   ├── getProductById.js
│   │   └── searchProducts.js
│   ├── types/
│   │   └── product.d.ts
│   └── index.js

Public API Pattern (index.js)

jsx

// features/auth/index.js - Only export what's needed
export { useAuth } from './hooks/useAuth';
export { AuthContext, AuthProvider } from './contexts/AuthContext';
export { LoginForm, SignupForm } from './components';

// ✅ Prevents: import from 'features/auth/hooks/useLogin'
// Users MUST: import { useAuth } from 'features/auth'

Domain Component Example

jsx

// features/products/components/ProductDetail.jsx
'use client';

import { use } from 'react';
import { useProductDetail } from '../hooks/useProductDetail';
import { ProductCard } from './ProductCard';

export function ProductDetail({ productId }) {
  const { product, related, loading, error } = useProductDetail(productId);
  
  if (loading) return <ProductDetailSkeleton />;
  if (error) return <ErrorMessage error={error} />;
  
  return (
    <div>
      <ProductCard product={product} />
      
      <section>
        <h3>Related Products</h3>
        <Suspense fallback={<RelatedSkeleton />}>
          <RelatedProductsList products={related} />
        </Suspense>
      </section>
    </div>
  );
}

jsx

// features/products/hooks/useProductDetail.js
export function useProductDetail(productId) {
  const [state, setState] = useState({
    product: null,
    related: [],
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    let mounted = true;
    
    Promise.all([
      api.getProductById(productId),
      api.getRelatedProducts(productId),
    ])
      .then(([product, related]) => {
        if (mounted) {
          setState({
            product,
            related,
            loading: false,
            error: null,
          });
        }
      })
      .catch(error => {
        if (mounted) {
          setState(prev => ({ ...prev, error, loading: false }));
        }
      });
    
    return () => { mounted = false; };
  }, [productId]);
  
  return state;
}

7. Component Organization Rules

What Goes Where?



Artifact	Location	When
Button, Card, Input	components/	Generic, reusable across features
LoginForm, UserCard	features/auth/components/	Auth-specific UI
useAuth	features/auth/hooks/	Auth business logic
formatDate, validateEmail	utils/	Pure functions, no React
AuthContext	features/auth/contexts/	Auth state management
API calls	features/auth/api/	HTTP requests for auth
Import Rules

jsx

// ✅ GOOD
import { Button, Card } from 'components';
import { useAuth } from 'features/auth';
import { formatDate } from 'utils';
import { AuthProvider } from 'features/auth/contexts/AuthContext';

// ❌ AVOID (breaks abstraction)
import UserCard from 'features/auth/components/UserCard';
import { useAuthInternals } from 'features/auth/hooks/internal';
import { validateEmail } from 'features/auth/api/validators';

8. Avoiding Common Mistakes

❌ Don't: Mix Presentation & Logic

jsx

// BAD
export function UserList() {
  const [users, setUsers] = useState([]);
  useEffect(() => { fetch('/api/users').then(setUsers); }, []);
  
  return (
    <div>
      {users.map(u => (
        <div key={u.id}>
          <h3 style={{color: 'red', fontSize: '20px'}}>
            {u.name}
          </h3>
          <button onClick={() => { /* delete */ }}>
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}

✅ Do: Separate Concerns

jsx

// Logic layer
function useUsers() {
  const [users, setUsers] = useState([]);
  useEffect(() => {
    api.getUsers().then(setUsers);
  }, []);
  return users;
}

// Presentation layer
function UserListItem({ user, onDelete }) {
  return (
    <UserCard user={user}>
      <DeleteButton onClick={() => onDelete(user.id)} />
    </UserCard>
  );
}

// Container
export function UserList() {
  const users = useUsers();
  return (
    <ul>
      {users.map(u => <UserListItem key={u.id} user={u} />)}
    </ul>
  );
}

9. File Naming Conventions

plaintext

✅ Components: PascalCase
  - UserCard.jsx
  - LoginForm.jsx
  - ProductDetail.jsx
✅ Hooks: camelCase starting with 'use'
  - useAuth.js
  - useProducts.js
  - useLocalStorage.js
✅ Utilities: camelCase
  - formatDate.js
  - validators.js
  - constants.js
✅ Tests: Same name + .test.js
  - Button.test.jsx
  - useAuth.test.js
✅ Styles: Component name + .module.css
  - Button.module.css
  - UserCard.module.css

10. React 19 Best Practices for Architecture

Use Server Components Where Possible

jsx

// ✅ Server Component (server-only, no hooks)
export async function ProductList({ category }) {
  const products = await db.products.find({ category });
  
  return (
    <div>
      {products.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  );
}

// ✅ Client Component (for interactivity)
'use client';
export function ProductFilters({ onFilterChange }) {
  const [filters, setFilters] = useState({});
  // Interactive logic here
}

Leverage use() for Async Data

jsx

'use client';

export function UserData({ userPromise }) {
  const user = use(userPromise); // React 19 magic
  return <div>{user.name}</div>;
}

Context Isolation by Feature

jsx

// ✅ Each feature has its own context
// features/auth/contexts/AuthContext.js
export const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
}

// features/products/contexts/ProductContext.js
// Separate, independent context

Quick Reference Checklist

✅ Folder structure is feature-based with clear boundaries
✅ Each component has ONE responsibility
✅ Presentational components are pure & dumb
✅ Logic lives in custom hooks or containers
✅ Primitives are in components/, domain-specific in features/
✅ No component file exceeds 300 lines
✅ Features export via index.js public API
✅ Utilities are pure functions (no React)
✅ Contexts are feature-scoped (not global)
✅ Tests live next to components (.test.js)
plaintext

This skill guide provides:
- ✅ **Practical folder structures** with clear rationale
- ✅ **Concrete patterns** (not abstract concepts)
- ✅ **Direct code examples** for each concept
- ✅ **React 19 specific** guidance (Server Components, `use()` hook)
- ✅ **Token-efficient** (focused, no fluff)
- ✅ **AI-friendly** (structured for code generation)
Use this as your coding agent's architecture reference!__*
