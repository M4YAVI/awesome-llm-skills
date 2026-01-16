# useActionState Hook - React 19 Skill Guide

## Overview

**`useActionState`** is a React 19 hook that manages form state and handles asynchronous server actions. It's specifically designed for working with forms and server-side form submissions while keeping the UI responsive during async operations.

**React Version:** 19.0.0+  
**Status:** Stable API  
**Source:** `packages/react/src/ReactHooks.js`

---

## API Signature

```typescript
function useActionState<S, P>(
  action: (prevState: S, formData: P) => S | Promise<S>,
  initialState: S,
  permalink?: string
): [state: S, dispatch: (payload: P) => void, isPending: boolean]

Parameters



Parameter	Type	Description
action	(prevState, payload) => S | Promise<S>	A function that takes the previous state and a payload, then returns the new state (sync or async). Can be a server action.
initialState	S	The initial state value. Must be a serializable value for server actions.
permalink	string (optional)	A URL that uniquely identifies this form action. Used during server-side rendering for state preservation across page navigations.
Return Value

Returns a tuple with three elements:



Element	Type	Description
state	S	Current state value
dispatch	(payload: P) => void	Function to dispatch an action with a payload
isPending	boolean	Whether an async action is currently in progress
When to Use useActionState

✅ Use When:


Handling Form Submissions
Managing form state that needs to be sent to a server
Reducing boilerplate for form state management


Server Actions (Full Stack)
Using Server Components with Server Actions
Need state persistence across page navigations
Want Progressive Enhancement (works without JavaScript)


Managing Async Operations
Actions take time (API calls, database operations)
Need to show loading states (isPending)
Multiple actions are dispatched and need to be queued


Error Handling
Action can throw errors that need UI handling
Need to keep previous state if action fails


Multi-step Form Workflows
Action result becomes the new state for next dispatch
Actions are queued and executed in order

❌ Don't Use When:

Simple state that doesn't need async operations → Use useState
Complex state with multiple fields → Consider useReducer
State not related to forms → Use useState
Only need pending state → Use useTransition
Don't need previous state value → Use useState
Basic Examples

Example 1: Simple Counter with Server Action

javascript

// Client Component
'use client';
import { useActionState } from 'react';

function Counter({ incrementAction }) {
  const [count, dispatch] = useActionState(incrementAction, 0);
  
  return (
    <form>
      <button formAction={dispatch}>
        Count: {count}
      </button>
    </form>
  );
}

Example 2: Async Inline Action with Pending State

javascript

'use client';
import { useActionState } from 'react';
import { startTransition } from 'react';

function Todo({ addTodo }) {
  const [todos, dispatch, isPending] = useActionState(
    async (prevTodos, formData) => {
      const title = formData.get('title');
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ title }),
      });
      const newTodo = await response.json();
      return [...prevTodos, newTodo];
    },
    []
  );

  return (
    <form>
      <input name="title" placeholder="Add a todo" />
      <button disabled={isPending}>
        {isPending ? 'Adding...' : 'Add'}
      </button>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </form>
  );
}

Example 3: Error Handling

javascript

'use client';
import { useActionState } from 'react';

function SaveForm() {
  const [state, dispatch, isPending] = useActionState(
    async (prevState, formData) => {
      const name = formData.get('name');
      
      if (!name) {
        return { ...prevState, error: 'Name is required' };
      }

      try {
        const response = await fetch('/api/save', {
          method: 'POST',
          body: JSON.stringify({ name }),
        });
        
        if (!response.ok) {
          return { ...prevState, error: 'Failed to save' };
        }
        
        return { data: await response.json(), error: null };
      } catch (error) {
        return { ...prevState, error: error.message };
      }
    },
    { data: null, error: null }
  );

  return (
    <form>
      <input name="name" />
      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state.error && <p style={{ color: 'red' }}>{state.error}</p>}
      {state.data && <p>Saved: {JSON.stringify(state.data)}</p>}
    </form>
  );
}

Example 4: Form with Multiple Buttons (Sync Action)

javascript

'use client';
import { useActionState } from 'react';

function ProductForm({ product }) {
  const [state, dispatch, isPending] = useActionState(
    (prevState, action) => {
      switch (action) {
        case 'save':
          return { ...prevState, status: 'saved' };
        case 'delete':
          return { ...prevState, status: 'deleted' };
        default:
          return prevState;
      }
    },
    { status: 'idle' }
  );

  return (
    <form>
      <input name="product" />
      <button 
        formAction={() => dispatch('save')}
        disabled={isPending}
      >
        Save
      </button>
      <button 
        formAction={() => dispatch('delete')}
        disabled={isPending}
      >
        Delete
      </button>
      <p>Status: {state.status}</p>
    </form>
  );
}

Key Characteristics

1. Action Queuing

Multiple dispatches are automatically queued and executed sequentially:

javascript

// These will execute one after another, not in parallel
dispatch('first');
dispatch('second');
dispatch('third');

2. Pending State Management

isPending reflects async operation status:

javascript

const [state, dispatch, isPending] = useActionState(action, initial);

// isPending is true when action is running
// Shows to user during async operations (loading spinners, disabled buttons)

3. Can Handle Both Sync and Async Actions

javascript

// Sync action
const [state, dispatch] = useActionState((prev, payload) => ({
  ...prev,
  value: payload
}), {});

// Async action
const [state, dispatch] = useActionState(async (prev, payload) => {
  const result = await fetch('/api/...');
  return await result.json();
}, {});

// Mix both - queuing handles them automatically

4. Dispatch is Stable & Non-Reactive

The dispatch function reference never changes and is NOT a dependency:

javascript

const [state, dispatch] = useActionState(action, init);

// dispatch can be safely passed to event handlers
// Does NOT need to be in useEffect/useCallback dependencies
const handleClick = () => dispatch(payload); // ✅ Safe

5. Cannot Dispatch During Render

javascript

function Component() {
  const [state, dispatch] = useActionState(action, init);
  
  dispatch(payload); // ❌ Error: "Cannot update form state while rendering"
  
  return <button onClick={() => dispatch(payload)}>OK ✅</button>;
}

Advanced Usage

Using with Server Actions (Full-Stack)

Server Action (actions.js):

javascript

'use server';

export async function updateProfile(prevState, formData) {
  const name = formData.get('name');
  const email = formData.get('email');

  // Validate
  if (!name || !email) {
    return { error: 'All fields required', ...prevState };
  }

  try {
    // Database update
    await db.profiles.update({ name, email });
    return { success: true, error: null };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

Client Component:

javascript

'use client';
import { useActionState } from 'react';
import { updateProfile } from './actions';

export function ProfileForm() {
  const [state, formAction, isPending] = useActionState(
    updateProfile,
    { success: false, error: null }
  );

  return (
    <form action={formAction}>
      <input name="name" />
      <input name="email" type="email" />
      <button disabled={isPending}>
        {isPending ? 'Updating...' : 'Update Profile'}
      </button>
      {state.error && <p style={{ color: 'red' }}>{state.error}</p>}
      {state.success && <p style={{ color: 'green' }}>Updated!</p>}
    </form>
  );
}

Permalink Parameter (MPA Form Submission)

Used to preserve state during Multi-Page App (MPA) form submissions:

javascript

const [state, dispatch, isPending] = useActionState(
  action,
  initialState,
  '/api/action' // permalink - used as stable identifier
);

When the same permalink is used across page reloads, React can reuse the state.

Using startTransition with useActionState

javascript

import { startTransition } from 'react';

function Component() {
  const [state, dispatch] = useActionState(asyncAction, init);

  const handleClick = () => {
    // Wrap dispatch in startTransition for non-blocking updates
    startTransition(() => {
      dispatch(payload);
    });
  };

  return <button onClick={handleClick}>Submit</button>;
}

Common Patterns

Pattern 1: Form with Loading Indicator

javascript

const [state, dispatch, isPending] = useActionState(action, init);

return (
  <form>
    <input name="title" />
    <button disabled={isPending}>
      {isPending && <Spinner />}
      {isPending ? 'Saving...' : 'Save'}
    </button>
  </form>
);

Pattern 2: Optimistic Updates

javascript

const [state, dispatch] = useActionState(
  async (prev, formData) => {
    const result = await updateServer(formData);
    return result;
  },
  initialState
);

Pattern 3: Validation with Error State

javascript

const [state, dispatch, isPending] = useActionState(
  async (prev, formData) => {
    const validation = validate(formData);
    if (!validation.valid) {
      return { ...prev, errors: validation.errors };
    }
    const result = await save(formData);
    return result;
  },
  { data: null, errors: {} }
);

Pattern 4: Action with Multiple Handlers

javascript

const [state, dispatch] = useActionState(
  (prev, action) => {
    switch (action.type) {
      case 'INCREMENT': return prev + action.value;
      case 'DECREMENT': return prev - action.value;
      default: return prev;
    }
  },
  0
);

return (
  <>
    <button onClick={() => dispatch({ type: 'INCREMENT', value: 1 })}>+</button>
    <button onClick={() => dispatch({ type: 'DECREMENT', value: 1 })}>-</button>
  </>
);

Comparison with Other Hooks



Hook	Use Case	Async	Action Queueing	Pending State
useState	Simple state	No	No	No
useReducer	Complex state with actions	No	No	No
useTransition	Mark updates as non-blocking	Yes	No	Yes
useActionState	Form state + async actions	Yes	✅ Yes	Yes
useFormStatus	Get pending status of parent form	-	-	Yes
Important Notes


Serialization: When using with Server Actions, state must be serializable (no functions, undefined, etc.)

Action Snapshot: The action implementation is captured at dispatch time, not at execution time. If the action function changes between dispatch and execution, the queued action uses the original function.

Error Handling: If an action throws an error, subsequent queued actions are canceled.

Progressive Enhancement: Works without JavaScript if using form elements with formAction attribute.

React Strict Mode: The hook works correctly in Strict Mode during development.
References

React Documentation: https://react.dev/reference/react/useActionState
Source Code: packages/react/src/ReactHooks.js (line 234-241)
Tests: packages/react-dom/src/__tests__/ReactDOMForm-test.js
Example: fixtures/flight/src/Counter.js
Testing Patterns

javascript

// Test with async action
it('should handle async actions', async () => {
  let action;
  function App() {
    const [state, dispatch] = useActionState(
      async (prev) => { await wait(); return prev + 1; },
      0
    );
    action = dispatch;
    return <div>{state}</div>;
  }
  
  const root = ReactDOMClient.createRoot(container);
  await act(() => root.render(<App />));
  expect(container.textContent).toBe('0');
  
  await act(() => startTransition(() => action()));
  // isPending updates render
  
  // Wait for async action
  await act(() => resolveAsync());
  expect(container.textContent).toBe('1');
});

Summary

useActionState is essential for React 19 form handling, providing:

✅ Automatic action queuing
✅ Built-in pending state management
✅ Server Action integration
✅ Progressive Enhancement support
✅ Clean, declarative form state management
Use it whenever you're building forms that interact with async operations or server actions.

plaintext

This comprehensive skill.md file covers:
1. **Complete API documentation** with signature and parameter table
2. **When to use/not use** - clear decision matrix
3. **5 practical examples** from basic to advanced
4. **Key characteristics** explaining important behaviors
5. **Advanced usage** including server actions and MPA scenarios
6. **Common patterns** developers encounter
7. **Comparison table** with similar hooks
8. **Important notes** and gotchas
9. **Testing patterns** for verification
10. **Real source code references** from the repo
The file is designed to be easily scannable for code agents while providing both conceptual understanding and practical guidance!
