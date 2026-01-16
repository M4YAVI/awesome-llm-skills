
# useOptimistic Hook - React 19 Skill Guide

## Overview

`useOptimistic` is a React 19 hook that enables **optimistic UI updates** - displaying changes to the user immediately while server operations are still pending. This creates a more responsive and polished user experience by not waiting for network confirmation before updating the UI.

**Hook Signature:**
```javascript
const [optimisticState, addOptimistic] = useOptimistic(passthrough, reducer);
```

## Parameters

### 1. **passthrough** (State Value)
The current state value that will be passed through when there are no pending optimistic updates. This is typically the value from `useState` or server state.

```javascript
// The actual state from server/parent
const [messages, setMessages] = useState(initialMessages);

// Pass it to useOptimistic
const [optimisticMessages, addOptimisticMessage] = useOptimistic(
  messages,
  reducer
);
```

### 2. **reducer** (Optional Reducer Function)
A function that determines how to update the state when an action is dispatched. Similar to `useReducer`, it receives the current state and an action, and returns the new state.

```javascript
const reducer = (state, action) => {
  switch (action.type) {
    case 'ADD_MESSAGE':
      // Optimistically add the new message
      return [...state, { id: action.id, text: action.text, pending: true }];
    case 'UPDATE_MESSAGE':
      return state.map(msg => 
        msg.id === action.id ? { ...msg, ...action.updates } : msg
      );
    case 'DELETE_MESSAGE':
      return state.filter(msg => msg.id !== action.id);
    default:
      return state;
  }
};
```

## Return Value

An array with two elements:

```javascript
[optimisticState, addOptimistic]
```

- **optimisticState**: The state value with optimistic updates applied (if any)
- **addOptimistic**: A function to dispatch optimistic updates

## How It Works

```
User Action
    ↓
[addOptimistic dispatches update]
    ↓
[UI immediately updates with optimistic state]
    ↓
[Server operation completes]
    ↓
IF server confirms: Keep optimistic update
IF server rejects: Revert to actual state
    ↓
Automatic rollback or confirmation
```

## Basic Example

```javascript
import { useState } from 'react';
import { useOptimistic } from 'react';

function ChatUI({ messages, sendMessage }) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newMessage) => [...state, newMessage]
  );

  const handleSendMessage = async (text) => {
    // Show message optimistically
    addOptimisticMessage({
      id: Date.now(),
      text,
      pending: true,
      author: 'You'
    });

    // Send to server
    try {
      const result = await fetch('/api/messages', {
        method: 'POST',
        body: JSON.stringify({ text })
      });
      // Server confirms, optimistic update stays
      const data = await result.json();
      console.log('Message saved:', data);
    } catch (error) {
      // Error occurs, optimistic update automatically rolls back
      console.error('Failed to send message');
    }
  };

  return (
    <div>
      {optimisticMessages.map(msg => (
        <div key={msg.id} className={msg.pending ? 'pending' : ''}>
          {msg.author}: {msg.text}
        </div>
      ))}
      <button onClick={() => handleSendMessage('Hello!')}>Send</button>
    </div>
  );
}
```

## Real-World Patterns

### 1. **Form Submission with Optimistic Updates**

```javascript
import { useOptimistic, useActionState } from 'react';

function EditProfile({ userId, initialName }) {
  const [name, setName] = useState(initialName);

  const [optimisticName, updateOptimisticName] = useOptimistic(
    name,
    (state, formData) => formData.get('name')
  );

  const [, formAction, isPending] = useActionState(
    async (prevState, formData) => {
      const newName = formData.get('name');
      
      // Optimistically show new name
      updateOptimisticName(formData);

      // Send to server
      const response = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        body: formData
      });

      if (!response.ok) {
        throw new Error('Failed to update profile');
      }

      setName(newName);
      return { success: true };
    },
    null
  );

  return (
    <form action={formAction}>
      <input 
        name="name" 
        defaultValue={optimisticName}
        disabled={isPending}
      />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

### 2. **Shopping Cart - Add/Remove Items**

```javascript
function ShoppingCart({ items, onAddItem, onRemoveItem }) {
  const [optimisticItems, addOptimisticAction] = useOptimistic(
    items,
    (state, action) => {
      if (action.type === 'ADD') {
        return [...state, { ...action.item, pending: true }];
      } else if (action.type === 'REMOVE') {
        return state.filter(item => item.id !== action.itemId);
      }
      return state;
    }
  );

  const handleAddItem = async (item) => {
    // Show item immediately
    addOptimisticAction({ type: 'ADD', item });

    try {
      await onAddItem(item);
    } catch (error) {
      // Item automatically removed from optimistic state on error
      console.error('Failed to add item');
    }
  };

  const handleRemoveItem = async (itemId) => {
    // Remove immediately from UI
    addOptimisticAction({ type: 'REMOVE', itemId });

    try {
      await onRemoveItem(itemId);
    } catch (error) {
      console.error('Failed to remove item');
    }
  };

  return (
    <div>
      {optimisticItems.map(item => (
        <CartItem
          key={item.id}
          item={item}
          onRemove={handleRemoveItem}
          isPending={item.pending}
        />
      ))}
    </div>
  );
}
```

### 3. **Like Button with Counter**

```javascript
function LikeButton({ postId, initialLikes, liked: initialLiked }) {
  const [optimisticCount, updateCount] = useOptimistic(
    { count: initialLikes, liked: initialLiked },
    (state, action) => {
      if (action.type === 'TOGGLE') {
        return {
          count: state.liked ? state.count - 1 : state.count + 1,
          liked: !state.liked
        };
      }
      return state;
    }
  );

  const handleToggleLike = async () => {
    // Update UI immediately
    updateCount({ type: 'TOGGLE' });

    try {
      const response = await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
        body: JSON.stringify({ liked: !optimisticCount.liked })
      });

      if (!response.ok) throw new Error('Failed to like');
    } catch (error) {
      // Rollback happens automatically
      console.error('Like operation failed');
    }
  };

  return (
    <button onClick={handleToggleLike}>
      {optimisticCount.liked ? '❤️' : '🤍'} {optimisticCount.count}
    </button>
  );
}
```

## When to Use `useOptimistic`

### ✅ **Good Use Cases**

1. **Form Submissions**: Submit data and show results immediately
   - Profile updates
   - Blog post edits
   - Comment additions

2. **Quick Actions**: Users expect instant visual feedback
   - Like/unlike buttons
   - Favorite toggling
   - Read/unread marks

3. **List Operations**: Add/remove items with instant feedback
   - Shopping carts
   - Todo lists
   - Task management

4. **Real-time Interactions**: Responsive UI feels smoother
   - Chat messages
   - Collaborative editing
   - Live notifications

5. **High Confidence Operations**: Low failure rate expected
   - User prefers immediate feedback over waiting
   - Network latency would hurt UX if we waited

### ❌ **Avoid Using When**

1. **Critical Operations**: Data accuracy is paramount
   - Payments/transactions
   - Destructive operations (permanent deletes)
   - Security-sensitive actions

2. **Complex State Logic**: Multiple interdependent updates
   - Use `useActionState` with proper state management

3. **High Error Rate**: Server frequently rejects operations
   - Better to validate first, then submit

4. **Uncertain Outcomes**: Result depends on server-side logic
   - Don't assume what server will return

5. **Offline Scenarios**: Unclear when/if sync will happen
   - Consider with caution alongside offline support

## Best Practices

### 1. **Handle Pending State Gracefully**

```javascript
const [optimisticMessages, addOptimistic] = useOptimistic(messages, reducer);
const [isPending, setPending] = useState(false);

const handleSend = async (text) => {
  setPending(true);
  addOptimistic({ text, pending: true });

  try {
    await sendToServer(text);
    // Success - optimistic update stays
  } catch (error) {
    // Error - rollback is automatic, but inform user
    showError('Failed to send message');
  } finally {
    setPending(false);
  }
};
```

### 2. **Add Visual Indicators for Pending Items**

```javascript
// Mark items as pending in your reducer
const reducer = (state, action) => {
  return [...state, { ...action.item, pending: true, timestamp: Date.now() }];
};

// Show different styling
<div className={item.pending ? 'item-pending' : 'item-confirmed'}>
  {item.pending && <Spinner />}
  {item.text}
</div>
```

### 3. **Combine with Form Actions**

```javascript
import { useActionState } from 'react';

function Todo({ todos, addTodo }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(todos, 
    (state, newTodo) => [...state, newTodo]
  );

  const [, formAction] = useActionState(
    async (prevState, formData) => {
      const text = formData.get('todo');
      const newTodo = { id: crypto.randomUUID(), text, pending: true };
      
      addOptimisticTodo(newTodo);

      const response = await fetch('/api/todos', {
        method: 'POST',
        body: formData
      });

      return await response.json();
    },
    null
  );

  return (
    <form action={formAction}>
      <input name="todo" />
      <button type="submit">Add</button>
      
      {optimisticTodos.map(todo => (
        <div key={todo.id}>
          {todo.text}
          {todo.pending && <span> (saving...)</span>}
        </div>
      ))}
    </form>
  );
}
```

### 4. **Validate Before Optimistic Update**

```javascript
const handleDelete = async (itemId) => {
  // Validate before updating
  if (!canDelete(itemId)) {
    showError('Cannot delete this item');
    return;
  }

  // Now it's safe to be optimistic
  addOptimisticAction({ type: 'DELETE', itemId });

  try {
    await deleteFromServer(itemId);
  } catch (error) {
    // Rollback is automatic
    showError('Deletion failed, changes reverted');
  }
};
```

### 5. **Handle Concurrent Updates**

```javascript
// Be aware that multiple optimistic updates stack
const [optimisticList, addOptimistic] = useOptimistic(list, (state, action) => {
  // Each dispatch queues sequentially
  // All will be rolled back together if server rejects
  return applyAction(state, action);
});

// Multiple fast clicks will queue optimistic updates
// Don't allow user to interact too rapidly
<button 
  onClick={() => handleQuickAction()} 
  disabled={optimisticList !== list} // Disable while pending
>
  Action
</button>
```

## Common Pitfalls

### ❌ **Wrong: Assuming server returns exactly what you sent**

```javascript
// DON'T DO THIS - server might transform the data
const [optimistic, addOptimistic] = useOptimistic(data, 
  (state, userInput) => [...state, userInput] // Too simple!
);
```

### ✅ **Right: Account for server-side transformations**

```javascript
const [optimistic, addOptimistic] = useOptimistic(data, 
  (state, userInput) => [
    ...state, 
    {
      ...userInput,
      id: `temp-${Date.now()}`, // Placeholder ID
      createdAt: new Date(), // Might differ from server
      pending: true
    }
  ]
);
```

### ❌ **Wrong: Not handling errors**

```javascript
// DON'T just fire and forget
const handleClick = () => {
  addOptimistic(newItem);
  fetch('/api/items', { method: 'POST', body: JSON.stringify(newItem) });
};
```

### ✅ **Right: Proper error handling**

```javascript
const handleClick = async () => {
  addOptimistic(newItem);
  
  try {
    const response = await fetch('/api/items', { 
      method: 'POST', 
      body: JSON.stringify(newItem) 
    });
    
    if (!response.ok) throw new Error('Server error');
    
    // Optional: Update with server response
    const serverItem = await response.json();
    // Merge or replace optimistic item
  } catch (error) {
    // Error handled - optimistic update auto-rolls back
    showError('Failed to save item');
  }
};
```

## Comparison with Alternatives

| Feature | `useOptimistic` | `useState` | `useActionState` | Server State |
|---------|-----------------|-----------|------------------|--------------|
| Immediate UI Update | ✅ | ✅ | ✅ | ❌ |
| Auto Rollback | ✅ | ❌ | ❌ | N/A |
| Server Integration | ✅ | ❌ | ✅ | ✅ |
| Simple to Use | ✅ | ✅ | ⚠️ | ⚠️ |
| For Forms | ⚠️ | ⚠️ | ✅ | ✅ |
| For Lists | ✅ | ⚠️ | ⚠️ | ⚠️ |

## React 19 Integration

`useOptimistic` is part of React 19's server-centric features and works best with:

- **Server Actions**: Pair with `useActionState` for form submissions
- **Suspense**: Handle loading states during server operations
- **Transitions**: Use `useTransition` for non-form async operations

```javascript
import { useOptimistic, useActionState } from 'react';

function Component({ data }) {
  const [optimisticData, updateOptimistic] = useOptimistic(data, reducer);
  const [state, formAction, isPending] = useActionState(
    async (prev, formData) => {
      updateOptimistic(processFormData(formData));
      // ... server action logic
    },
    null
  );

  return (
    <form action={formAction}>
      {/* Form fields */}
    </form>
  );
}
```

## Performance Considerations

1. **Reducer Complexity**: Keep reducers fast (executed on every dispatch)
2. **Large Lists**: Be careful with many optimistic items in large lists
3. **Frequent Updates**: Debounce if users trigger many rapid updates
4. **Memory**: Optimistic state is kept in memory during operations

## Summary

`useOptimistic` is a powerful hook for building responsive, user-friendly interfaces. Use it when you need immediate visual feedback for actions that will be confirmed by the server, but always have proper error handling and validation in place.

**Key Takeaway**: Optimistic UI updates make applications feel fast and responsive, but they're a UX layer on top of reliable server operations. Never compromise data integrity for perceived speed.

---

*Last Updated: React 19*
```

I've created a comprehensive skill guide for the `useOptimistic` hook! Here's what's included:

## 📋 Guide Contents:

1. **Overview & Signature** - Quick reference for the hook syntax
2. **Parameters Explained** - Detailed breakdown of `passthrough` and `reducer`
3. **Return Value** - What the hook provides
4. **How It Works** - Visual flow diagram
5. **Basic Example** - Simple chat UI implementation
6. **Real-World Patterns**:
   - Form submissions
   - Shopping cart add/remove
   - Like button counter
7. **When to Use** - Best practices and anti-patterns
8. **Best Practices** - 5 key patterns with code examples
9. **Common Pitfalls** - What to avoid and correct ways
10. **Comparisons** - vs other state management approaches
11. **React 19 Integration** - How it works with other new features
12. **Performance Considerations** - Things to keep in mind

This guide is perfect for a code agent because it:
- ✅ Explains the **what, why, and when** clearly
- ✅ Provides **concrete code examples** for real scenarios
- ✅ Shows **anti-patterns to avoid**
- ✅ Includes **decision trees** for when to use it
- ✅ Has **practical patterns** ready to copy-paste
- ✅ Covers **error handling and edge cases**

You can save this as `useOptimistic_skill.md` in your repository!
