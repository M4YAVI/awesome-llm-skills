# useFormStatus Hook - React 19

## Overview

`useFormStatus` is a React 19 hook that provides information about the current status of a form submission within a component. It allows you to track whether a form is currently pending and access metadata about the form action being executed.

**Introduced in:** React 19  
**Package:** `react-dom`  
**Import:** `import { useFormStatus } from 'react-dom';`

---

## Signature

```javascript
const { pending, data, method, action } = useFormStatus();

Return Value

Returns a FormStatus object with the following properties:

typescript

type FormStatus = {
  // When the form is not pending
  pending: false,
  data: null,
  method: null,
  action: null,
} | {
  // When the form IS pending
  pending: true,
  data: FormData,
  method: string,
  action: string | (FormData => void | Promise<void>) | null,
}

Properties Explained



Property	Type	Description
pending	boolean	Whether the form is currently being submitted. true when a form action is in progress.
data	FormData | null	The form data being submitted. Available as a FormData object when pending is true, otherwise null. Access fields with data.get('fieldName').
method	string | null	The HTTP method of the form submission (e.g., 'get', 'post'). Extracted from the form's method attribute. null when not pending.
action	Function | string | null	The action being invoked. Can be a function, URL string, or null when not pending.
Core Concepts

1. Form Context Dependency

useFormStatus ONLY works within components that are descendants of a <form> element. The hook must be called inside the component tree of the form it's tracking.

javascript

// ✅ CORRECT - Component is inside the form
function MyForm() {
  return (
    <form action={handleSubmit}>
      <SubmitStatus />
      <input name="email" />
    </form>
  );
}

function SubmitStatus() {
  const { pending } = useFormStatus(); // Works!
  return pending ? <p>Submitting...</p> : null;
}

// ❌ WRONG - Component is outside the form
function App() {
  return (
    <div>
      <MyForm />
      <SubmitStatus /> {/* useFormStatus returns "not pending" always */}
    </div>
  );
}

2. Implicit Transitions

Form actions are automatically wrapped in startTransition, which means:

Updates caused by form actions are non-blocking
Slower updates can be interrupted by more urgent ones
UI can show pending states while waiting for results
Suspense boundaries are supported
javascript

function Form() {
  const { pending } = useFormStatus();
  
  return (
    <form action={asyncAction}>
      <Suspense fallback={<LoadingSpinner />}>
        <FormContent />
      </Suspense>
      {pending && <p>Processing...</p>}
    </form>
  );
}

3. Server vs. Client Actions

Works with both server actions and client-side async functions:

javascript

// With server action
'use server'
export async function submitData(formData) {
  // Server-side logic
}

// In client component
function Form() {
  const { pending, data } = useFormStatus();
  
  return (
    <form action={submitData}>
      {pending && <p>Saving to server...</p>}
    </form>
  );
}

Common Use Cases

1. Display Loading State During Form Submission

javascript

function SubscribeForm() {
  const { pending } = useFormStatus();

  return (
    <form action={subscribe}>
      <input type="email" name="email" placeholder="Enter email" />
      <button type="submit" disabled={pending}>
        {pending ? "Subscribing..." : "Subscribe"}
      </button>
    </form>
  );
}

When to use: Display spinner, change button text, disable inputs during submission.

2. Show Form Data Being Submitted

javascript

function CheckoutForm() {
  const { pending, data, method } = useFormStatus();

  return (
    <form action={processPayment} method="post">
      <input type="email" name="email" />
      <input type="text" name="amount" />
      
      {pending && data && (
        <div className="preview">
          <p>Submitting: {data.get('email')}</p>
          <p>Amount: ${data.get('amount')}</p>
        </div>
      )}
      
      <button type="submit">{pending ? "Processing..." : "Pay"}</button>
    </form>
  );
}

When to use: Preview or confirm data before final submission, show what's being sent.

3. Display Action Name for Debugging

javascript

function MultiActionForm() {
  const { pending, action } = useFormStatus();

  return (
    <form>
      <input type="text" name="title" />
      
      {pending && (
        <p>Executing: {action?.name || 'Unknown action'}</p>
      )}
      
      <button formAction={saveItem}>Save</button>
      <button formAction={deleteItem}>Delete</button>
    </form>
  );
}

When to use: Distinguish between multiple form actions on the same form.

4. Optimistic UI Updates

javascript

function TodoItem({ id, text }) {
  const { pending, data } = useFormStatus();
  
  // Optimistically show the updated text while submitting
  const isUpdating = pending && data?.get('id') === id;
  const displayText = isUpdating ? data.get('text') : text;

  return (
    <form action={updateTodo}>
      <input 
        type="hidden" 
        name="id" 
        value={id} 
      />
      <span style={{ opacity: isUpdating ? 0.6 : 1 }}>
        {displayText}
      </span>
    </form>
  );
}

When to use: Immediately reflect changes in the UI before server confirmation.

5. Async Form with Error Boundary

javascript

function ProfileForm() {
  const { pending } = useFormStatus();

  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <form action={updateProfile}>
        <input type="text" name="name" />
        <input type="email" name="email" />
        
        {pending && <LoadingOverlay />}
        
        <button type="submit" disabled={pending}>
          {pending ? "Updating..." : "Update Profile"}
        </button>
      </form>
    </ErrorBoundary>
  );
}

When to use: Handle errors gracefully while showing loading state.

Implementation Example

Complete Example: Newsletter Signup

javascript

import { useFormStatus } from 'react-dom';

async function subscribeAction(formData) {
  const email = formData.get('email');
  
  // Simulate API call
  await new Promise(resolve => setTimeout(resolve, 1000));
  
  console.log('Subscribed:', email);
}

function SubscribeStatus() {
  const { pending, data } = useFormStatus();
  
  if (!pending) {
    return null;
  }

  const email = data?.get('email');
  return (
    <div className="status">
      <p>Subscribing {email}...</p>
      <div className="spinner" />
    </div>
  );
}

export default function Newsletter() {
  return (
    <div className="newsletter">
      <h2>Join Our Newsletter</h2>
      
      <form action={subscribeAction}>
        <input
          type="email"
          name="email"
          placeholder="your@email.com"
          required
        />
        <button type="submit">Subscribe</button>
      </form>
      
      <SubscribeStatus />
    </div>
  );
}

Complete Example: Multi-Action Form

javascript

import { useFormStatus } from 'react-dom';

async function savePost(formData) {
  const title = formData.get('title');
  console.log('Saving:', title);
  // API call
}

async function publishPost(formData) {
  const title = formData.get('title');
  console.log('Publishing:', title);
  // API call
}

function FormStatus() {
  const { pending, action, data } = useFormStatus();
  
  if (!pending) return null;

  return (
    <div className="form-status">
      <p>
        {action === savePost && 'Saving draft...'}
        {action === publishPost && 'Publishing...'}
      </p>
      <p>Title: {data?.get('title')}</p>
    </div>
  );
}

export default function PostEditor() {
  return (
    <form className="editor">
      <input
        type="text"
        name="title"
        placeholder="Post title"
        required
      />
      <textarea
        name="content"
        placeholder="Post content"
        required
      />
      
      <div className="actions">
        <button formAction={savePost}>Save Draft</button>
        <button formAction={publishPost}>Publish</button>
      </div>
      
      <FormStatus />
    </form>
  );
}

When to Use useFormStatus

✅ Use useFormStatus When:


Need to show pending state during form submission
Loading spinners, disabled buttons, changing text
Example: Show "Saving..." while form action runs


Want to display form data being submitted
Preview what user is sending
Show confirmation before final action
Example: Display email address being subscribed


Have multiple actions on one form
Distinguish which action is being executed
Example: Save, Delete, Archive buttons on same form


Implementing optimistic updates
Reflect UI changes before server responds
Revert on error
Example: Check off todo before server confirms


Building forms with Suspense
Handle async dependencies during submission
Show loading fallback
Example: Load related data while processing


Form requires method or action inspection
Know the HTTP method being used
Identify which action is executing
Example: Log request details for analytics

❌ Don't Use useFormStatus When:


Component is NOT inside a form
Hook won't work; will always return "not pending"
Use useTransition instead for manual state management


Need to manage state outside form context
Use useState + useCallback for manual control
Use useTransition for custom transition handling


Handling non-form async operations
File uploads, API calls not tied to forms
Use useTransition or async/await with state


Legacy form handling with preventDefault
If you're manually handling form submission
Won't integrate with useFormStatus

Rules and Constraints

Rule 1: Must Be Inside a Form

javascript

// ❌ Won't work - outside form
const status = useFormStatus(); 

// ✅ Works - inside form component
function Form() {
  return (
    <form action={handleSubmit}>
      <FormChild /> {/* useFormStatus works here */}
    </form>
  );
}

Rule 2: Only Works with Actual Form Submission

javascript

// ❌ Won't track this - not using form submission
function Component() {
  const handleClick = async () => {
    const formData = new FormData();
    // manual action
  };
  return <button onClick={handleClick}>Click</button>;
}

// ✅ Works - using actual form submission
function Component() {
  return (
    <form action={action}>
      <input />
      <button type="submit">Submit</button>
    </form>
  );
}

Rule 3: Tracks Form Submission, Not Manual Calls

javascript

function Component() {
  const { pending } = useFormStatus();
  
  // ✅ Works - triggered by form submission
  const handleSubmit = async (formData) => {
    // pending will be true during this
  };

  // ❌ Won't update pending
  const manualAction = async () => {
    // pending stays false even if you call async code
  };

  return (
    <form action={handleSubmit}>
      <button type="submit">Submit</button>
    </form>
  );
}

Comparison with Related APIs



Hook	Purpose	Location	Best For
useFormStatus	Track form submission state	Inside form	Form loading states, progress
useTransition	Track custom async transitions	Anywhere	Manual async tracking
useActionState	Manage form action state	Inside/outside form	Form state + pending
useState	Manual state management	Anywhere	Simple state
Integration Points

Works With:

✅ Server actions ('use server')
✅ Client-side async functions
✅ useActionState hook
✅ Suspense boundaries
✅ Error boundaries
✅ startTransition
✅ requestFormReset
Does Not Work With:

❌ Non-form async operations
❌ Manual form submission (e.preventDefault + manual action)
❌ Components outside form context
❌ Event handlers that aren't form submission
Browser Support

useFormStatus is a React 19+ feature. Ensure:

React version 19 or higher
react-dom version 19 or higher
Modern browser (ES2020+)
Performance Considerations

Minimal Re-renders: Only components using useFormStatus re-render when form status changes
Memoization: Wrap child components with React.memo to prevent unnecessary re-renders
Move Down the Tree: Place useFormStatus calls as deep in the component tree as possible
javascript

// ✅ Better - only Status component re-renders
function Form() {
  return (
    <form action={handleSubmit}>
      <InputFields /> {/* No unnecessary re-renders */}
      <SubmitStatus /> {/* Only this re-renders */}
    </form>
  );
}

// ❌ Less efficient - entire form re-renders
function Form() {
  const { pending } = useFormStatus();
  return (
    <form action={handleSubmit}>
      {/* Everything re-renders when pending changes */}
      <InputFields />
      {pending && <Status />}
    </form>
  );
}

Common Pitfalls

Pitfall 1: Using Outside Form Context

javascript

// ❌ Doesn't work
function App() {
  return (
    <div>
      <MyForm />
      <Status /> {/* useFormStatus here won't work */}
    </div>
  );
}

// ✅ Correct
function App() {
  return (
    <form action={handleSubmit}>
      <MyForm />
      <Status /> {/* useFormStatus works */}
    </form>
  );
}

Pitfall 2: Trying to Track Non-Form Actions

javascript

// ❌ Won't work
function Component() {
  const { pending } = useFormStatus();
  
  const handleClick = async () => {
    // pending won't update here
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// ✅ Use useTransition instead
function Component() {
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    startTransition(async () => {
      // isPending will track this
    });
  };
  
  return <button onClick={handleClick}>Click</button>;
}

Pitfall 3: Accessing FormData Incorrectly

javascript

// ❌ Won't work - FormData is not plain object
const email = data.email;

// ✅ Correct - use .get() method
const email = data.get('email');

// ✅ Or iterate
for (const [key, value] of data) {
  console.log(key, value);
}

Advanced Patterns

Pattern 1: Async Form with Optimistic Updates

javascript

function TodoList() {
  const [todos, setTodos] = useState([]);
  
  async function addTodo(formData) {
    const text = formData.get('text');
    const newTodo = { id: Date.now(), text };
    
    // Optimistically add
    setTodos(prev => [...prev, newTodo]);
    
    // Send to server
    await fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify(newTodo)
    });
  }

  function TodoStatus() {
    const { pending, data } = useFormStatus();
    if (!pending) return null;
    
    return <p>Adding: {data.get('text')}</p>;
  }

  return (
    <form action={addTodo}>
      <input name="text" />
      <button type="submit">Add Todo</button>
      <TodoStatus />
    </form>
  );
}

Pattern 2: Form with Method Detection

javascript

function SmartForm() {
  const { pending, method, data } = useFormStatus();

  return (
    <form method="post" action={handleAction}>
      {pending && (
        <div className="notice">
          {method === 'post' && 'Creating...'}
          {method === 'put' && 'Updating...'}
          {method === 'delete' && 'Deleting...'}
        </div>
      )}
    </form>
  );
}

Testing with useFormStatus

javascript

import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('shows pending state during form submission', async () => {
  const user = userEvent.setup();
  
  render(<SubscribeForm />);
  
  const input = screen.getByPlaceholderText('Enter email');
  const button = screen.getByRole('button', { name: /subscribe/i });
  
  await user.type(input, 'test@example.com');
  await user.click(button);
  
  // Check for pending UI
  expect(screen.getByText('Subscribing...')).toBeInTheDocument();
  
  // Wait for completion
  await screen.findByText('Subscribe');
});

TypeScript Support

typescript

import { useFormStatus } from 'react-dom';

interface FormStatusValue {
  pending: boolean;
  data: FormData | null;
  method: string | null;
  action: ((formData: FormData) => void | Promise<void>) | string | null;
}

function MyForm(): JSX.Element {
  const status: FormStatusValue = useFormStatus();
  
  if (status.pending && status.data) {
    const email = status.data.get('email') as string;
    return <p>Subscribing {email}...</p>;
  }
  
  return null;
}

Key Takeaways

Track form submission state with pending, data, method, and action
Must be inside a form component to work
Use for loading states and optimistic UI updates
Works with server actions and async functions
Automatically integrates with Suspense and Error Boundaries
Place deep in the tree for better performance
Not for manual async operations (use useTransition instead)
Related Resources

React 19 Announcement
Server Components Guide
useActionState Hook
useTransition Hook
Form Element MDN Docs
Last Updated: React 19+

Difficulty Level: Intermediate

Use Frequency: Common (in form-heavy applications)

plaintext

Now let me save this to a file in the repository:
Let me create the file in a logical location. Based on the repository structure, I'll create it in the root:
I've created a comprehensive `useFormStatus` skill.md file! Here's what it covers:
## 📋 File Structure
The file includes:
1. **Overview** - What `useFormStatus` is, introduced in React 19
2. **Signature & Return Value** - Full type definition with all properties
3. **Core Concepts** - 3 key concepts with code examples:
   - Form context dependency
   - Implicit transitions
   - Server vs. client actions
4. **Common Use Cases** - 5 practical examples:
   - Loading states during submission
   - Show form data being submitted
   - Display action name for debugging
   - Optimistic UI updates
   - Async forms with error boundaries
5. **Complete Examples** - 2 full implementations:
   - Newsletter signup
   - Multi-action form (Save/Publish)
6. **When to Use** - Clear guidelines on when and when NOT to use it
7. **Rules & Constraints** - 3 important rules with code examples
8. **Comparison Table** - How it relates to `useTransition`, `useActionState`, `useState`
9. **Integration Points** - What it works/doesn't work with
10. **Performance Considerations** - Optimization tips
11. **Common Pitfalls** - 3 frequent mistakes with solutions
12. **Advanced Patterns** - Optimistic updates and method detection
13. **Testing Guide** - How to test components using `useFormStatus`
14. **TypeScript Support** - Type definitions and usage
15. **Key Takeaways** - Summary of main points
## 🎯 Perfect for Code Agents
This skill.md file is designed to help code agents by:
- ✅ **Clear structure** with headings and sections for easy navigation
- ✅ **Code examples** for every concept (both ✅ correct and ❌ incorrect patterns)
- ✅ **Use case matrix** showing when to use vs. when not to use
- ✅ **Complete implementations** they can copy/adapt
- ✅ **Common pitfalls** section helps avoid mistakes
- ✅ **Comparison tables** for understanding relationships with other APIs
- ✅ **TypeScript examples** for type-safe implementations
- ✅ **Testing examples** for integration with test suites
- ✅ **Performance tips** for optimization
- ✅ **Integration points** showing what works together
You can now use this file as a reference guide for understanding and implementing `useFormStatus` in React 19 applications!
