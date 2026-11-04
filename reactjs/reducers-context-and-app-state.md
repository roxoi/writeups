---
title: "Module 4: Reducers, Context & App-Wide State"
description: "Learn useReducer for complex state, Context for sharing data without prop drilling, why Context causes re-renders, and when to choose Zustand, Redux Toolkit or TanStack Query - from first idea to interview-level detail."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/react-module-4.png"
tags: [React, useReducer, Context, Zustand, Redux-Toolkit, State-Management, Interview-Prep]
keywords: ["React useReducer explained", "React Context API tutorial", "Context vs Redux vs Zustand", "Avoid prop drilling React", "React state management 2026", "Server state vs client state"]
---

# Module 4: Reducers, Context & App-Wide State

![React Module 4](/images/react-module-4.png)

## Words you will meet in this module

| Word              | Simple meaning                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------------- |
| **Reducer**       | A pure function `(state, action) => newState` that describes _every_ way the state can change.     |
| **Action**        | A small object describing **what happened**, e.g. `{ type: "added", id: 3 }`.                      |
| **Dispatch**      | The function you call with an action to ask the reducer to update state.                           |
| **Prop drilling** | Passing props through many middle components that do not use them, only to reach a deep child.     |
| **Context**       | A way to make a value available to a whole subtree without passing it through props.               |
| **Provider**      | The component that supplies a context value to everything inside it.                               |
| **Consumer**      | Any component that reads a context value (with `useContext` or `use`).                             |
| **Selector**      | A function that picks a _slice_ of a store so a component re-renders only when that slice changes. |
| **Client state**  | Data that only exists in the browser UI: modal open, theme, form draft, selected tab.              |
| **Server state**  | Data that lives on a server and is _borrowed_ by the UI: users, products, orders.                  |
| **Store**         | A central place (outside components) that holds state and the ways to change it.                   |

## 4.1 `useReducer`: State with Rules

`useState` is perfect for simple state. When one component has **many state values changed by many different handlers** (a cart, a form wizard, a game), the update logic spreads across the file and becomes hard to follow. **`useReducer`** moves all update logic into **one function** that you can read top to bottom.

Think of a bank. With `useState`, every customer walks up to the vault and changes the balance directly. With `useReducer`, customers fill in a **slip** ("deposit 500") and hand it to a **teller** who follows fixed rules. The slip is the **action**. The teller is the **reducer**.

### The shape

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

```
   event handler            React                     your reducer
 ───────────────────   ───────────────   ─────────────────────────────────
 dispatch(action)  ─►  calls reducer(state, action) ─► returns NEW state
                       stores it and re-renders
```

### Example: a task list

```jsx
import { useReducer } from 'react';

const initialState = [{ id: 1, text: 'Learn reducers', done: false }];

function tasksReducer(tasks, action) {
  switch (action.type) {
    case 'added':
      return [...tasks, { id: action.id, text: action.text, done: false }];

    case 'toggled':
      return tasks.map((t) =>
        t.id === action.id ? { ...t, done: !t.done } : t,
      );

    case 'deleted':
      return tasks.filter((t) => t.id !== action.id);

    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

let nextId = 2;

export default function TaskApp() {
  const [tasks, dispatch] = useReducer(tasksReducer, initialState);

  function handleAdd(text) {
    dispatch({ type: 'added', id: nextId++, text });
  }

  return (
    <>
      <AddTask onAdd={handleAdd} />
      <ul>
        {tasks.map((t) => (
          <li key={t.id}>
            <input
              type="checkbox"
              checked={t.done}
              onChange={() => dispatch({ type: 'toggled', id: t.id })}
            />
            {t.text}
            <button onClick={() => dispatch({ type: 'deleted', id: t.id })}>
              Delete
            </button>
          </li>
        ))}
      </ul>
    </>
  );
}
```

Event handlers now only say **what happened**. The reducer decides **how the state changes**. Adding a new behavior means adding one `case`, in one place.

### Rules for reducers

| Rule                                                      | Why                                                                                           |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Must be **pure**: same inputs, same output                | React may call it more than once. In development, StrictMode calls it **twice** to check.     |
| **Do not mutate** state. Return a new value               | Same reason as Module 2: change detection uses `Object.is`.                                   |
| **No side effects** (fetch, random, `Date.now()`, timers) | A reducer only calculates the next state. Put side effects in handlers or effects.            |
| Name actions by **what happened**, not "set X"            | `"item_added"` is a user intent. `"set_items"` just moves your old `setState` somewhere else. |
| Each action = one user interaction                        | A single `"reset_form"` action can change many fields at once.                                |

Generate IDs and random values **in the handler** and put them in the action (like `id: nextId++` above), never inside the reducer.

### Lazy initialization

```jsx
const [state, dispatch] = useReducer(reducer, userId, (id) =>
  createInitialState(id),
);
// the third argument is run once; createInitialState is not re-run on every render
```

### `useState` vs. `useReducer`

| Question                              | `useState`                       | `useReducer`                                   |
| ------------------------------------- | -------------------------------- | ---------------------------------------------- |
| Amount of code for simple state       | Less                             | More (reducer + actions)                       |
| Many handlers touching the same state | Update logic scattered           | Update logic **centralized**                   |
| Next state depends on complex rules   | Hard to read                     | Easy: one `switch`                             |
| Debugging                             | Add `console.log` at each setter | Log **every action** in one place              |
| Testing                               | Test via the component           | Test the reducer as a plain function, no React |
| Passing update logic to deep children | Pass many setters                | Pass **one stable `dispatch`**                 |

Rule of thumb: start with `useState`. Move to `useReducer` when several handlers change related state or when the logic gets hard to follow. Both are equally "correct."

#### Interview traps

- "Why is `dispatch` stable?" React guarantees `dispatch` (like `setState`) keeps the same identity across renders, so passing it to children or putting it in an effect's dependencies does not cause extra work.
- "Can a reducer fetch data?" No. It must be pure. Fetch in a handler or effect and `dispatch` the result.
- "Why does my reducer log twice?" StrictMode double-invokes reducers in development to catch impurity.

#### Tricky question and answer

###### Q1 [SDE-2]: What is wrong with this reducer?

```jsx
function reducer(state, action) {
  if (action.type === 'added') {
    state.items.push({ id: Date.now(), ...action.item });
    return state;
  }
}
```

**Answer:** Three problems. It **mutates** state (`push`) and returns the same reference, so React may not re-render. It calls **`Date.now()`**, which is impure, so the double invocation in StrictMode creates different IDs. It has **no default branch**, so unknown actions return `undefined` and wipe the state. Correct version: build the ID in the handler, return `{ ...state, items: [...state.items, action.item] }`, and throw or return `state` in `default`.

## 4.2 Context: Skipping the Middle Components

### The problem: prop drilling

```
App (has `user`)
 └─ Layout           ← does not use user, just passes it
     └─ Sidebar      ← does not use user, just passes it
         └─ Menu     ← does not use user, just passes it
             └─ Avatar   ← finally uses user
```

Three components carry a prop they do not care about. If `Avatar` later needs another value, all of them change.

**Context** is like a broadcast. A component high in the tree **provides** a value, and any component below can **read** it directly without it passing through the layers between.

### The three steps (React 19 syntax)

```jsx
import { createContext, useContext, useState } from 'react';

// 1. CREATE the context (the argument is a default used only if there's no provider)
const ThemeContext = createContext('light');

// 2. PROVIDE a value to a subtree
function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext value={theme}>
      {' '}
      {/* React 19: no .Provider needed */}
      <Page />
      <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
        Toggle
      </button>
    </ThemeContext>
  );
}

// 3. CONSUME the value anywhere below
function Button({ children }) {
  const theme = useContext(ThemeContext);
  return <button className={`btn-${theme}`}>{children}</button>;
}
```

> **React 19 change:** you can render `<ThemeContext value={...}>` directly. The older `<ThemeContext.Provider value={...}>` still works, and you will see it in most existing code.

### `use(Context)`: a more flexible read (React 19)

The `use` API can read context too, and, unlike hooks, it **can be called conditionally**:

```jsx
import { use } from 'react';

function Heading({ children }) {
  if (children == null) return null; // early return is fine before `use`
  const theme = use(ThemeContext); // allowed after a condition
  return <h1 className={theme}>{children}</h1>;
}
```

### The production pattern: provider component + custom hook

Do not export the raw context. Export a **Provider** component and a **hook**. This keeps the implementation private and catches mistakes early.

```jsx
// src/context/AuthContext.jsx
import {
  createContext,
  useContext,
  useState,
  useMemo,
  useCallback,
} from 'react';

const AuthContext = createContext(null); // null = "no provider found"

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = useCallback((u) => setUser(u), []);
  const logout = useCallback(() => setUser(null), []);

  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

  return <AuthContext value={value}>{children}</AuthContext>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (ctx === null) {
    throw new Error('useAuth must be used inside <AuthProvider>');
  }
  return ctx;
}
```

```jsx
// main.jsx
<AuthProvider>
  <App />
</AuthProvider>;

// anywhere
const { user, logout } = useAuth();
```

### What context is good for

| Good fit (changes rarely, needed widely) | Poor fit                                            |
| ---------------------------------------- | --------------------------------------------------- |
| Current theme, locale                    | Fast-changing values (mouse position, a timer)      |
| Logged-in user                           | Large, frequently updated objects (a big store)     |
| Feature flags, router info               | Server data that needs caching (use TanStack Query) |

### The big catch: every consumer re-renders

When a context value changes (by `Object.is`), **every component that reads that context re-renders**, even if it only uses one field of an object. There is no built-in selector.

```jsx
// This creates a NEW object each time AppProvider renders,
// so EVERY consumer re-renders whenever AppProvider re-renders:
<AppContext value={{ user, theme, setUser, setTheme }}>
```

Three ways to reduce the damage:

1. **Memoize the value** with `useMemo` (as in `AuthProvider` above), and keep updaters stable with `useCallback`. (The React Compiler can do this automatically, Module 5.)
2. **Split contexts** by update frequency: state in one, dispatch (which never changes) in another.
3. **Split by domain:** `ThemeContext`, `AuthContext`, `CartContext`, not one giant `AppContext`.

### Reducer + Context together (a mini "Redux")

```jsx
// src/context/TasksContext.jsx
import { createContext, useContext, useReducer } from 'react';

const TasksContext = createContext(null);
const TasksDispatchContext = createContext(null);

export function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, []);
  return (
    <TasksContext value={tasks}>
      <TasksDispatchContext value={dispatch}>
        {' '}
        {/* dispatch is stable */}
        {children}
      </TasksDispatchContext>
    </TasksContext>
  );
}

export const useTasks = () => useContext(TasksContext);
export const useTasksDispatch = () => useContext(TasksDispatchContext);
```

Why two contexts? A component that only **dispatches** (an "Add" button) reads only `TasksDispatchContext`. Since `dispatch` never changes, that component **does not re-render when tasks change**. Only components that read the task list do.

### Context scope and nesting

Context is looked up from the nearest provider **above** the consumer, so you can **override** it for a subtree:

```jsx
<ThemeContext value="dark">
  <Page /> {/* dark */}
  <ThemeContext value="light">
    <Footer /> {/* light: nearest provider wins */}
  </ThemeContext>
</ThemeContext>
```

#### Interview traps

- "Is Context a state management library?" No. It is a **transport**: a way to pass a value down without props. The state still comes from `useState` or `useReducer`.
- "Why do all consumers re-render when one field changes?" Context compares the **whole value** by reference. It has no selector. Split contexts, memoize, or use a store with selectors.
- "What is the default value of `createContext(x)` for?" It is used **only when no provider is above** the consumer. It is not the initial state.
- "Where do you place a Provider?" As **low** as is practical. Wrapping the whole app with everything makes every update expensive.

#### Tricky questions and answers

###### Q2 [SDE-2]: `<Provider value={{ user, setUser }}>` is in `App`. `App` re-renders because of an unrelated state change, and every consumer re-renders. Why, and how do you stop it?

**Answer:** The object literal `{ user, setUser }` is a **new object on every render**, so context sees a changed value and notifies all consumers. Fix: wrap it in `useMemo(() => ({ user, setUser }), [user])`, or hold the provider in its **own component** that owns the state so unrelated parents do not re-render it. With the React Compiler enabled, the memoization is added automatically if your code follows the Rules of React.

###### Q3 [SDE-3]: Components wrapped in `React.memo` still re-render when context changes. Why?

**Answer:** `memo` compares **props**. Context is not a prop. A component that calls `useContext` subscribes directly to that context, and React re-renders it when the value changes, regardless of `memo`. To avoid it, split the context so the component only reads the part that does not change, or lift the context read into a parent and pass down only the needed piece as a prop to a memoized child.

## 4.3 Composition: Often Better Than Context

Before reaching for Context, try restructuring. Prop drilling is often a sign that you passed **data** when you could pass **components**.

```jsx
// DRILLING: Layout and Sidebar must carry `user` just for Avatar
<Layout user={user} />;
// Layout → <Sidebar user={user} /> → <Avatar user={user} />

// COMPOSITION: build the Avatar at the top, pass it down as children / a prop
function App() {
  const user = useUser();
  return (
    <Layout
      sidebar={
        <Sidebar>
          <Avatar user={user} />
        </Sidebar>
      }
    >
      <Dashboard />
    </Layout>
  );
}
// Layout and Sidebar no longer know `user` exists.
```

This also helps performance: elements passed as `children` are created by the parent, so when `Layout` re-renders because of its own state, those child elements keep the **same reference** and React can skip re-rendering them (Module 5).

| Situation                                                 | Try first                                  |
| --------------------------------------------------------- | ------------------------------------------ |
| Data needed 2 to 3 levels down                            | Just pass props                            |
| Middle components only forward data                       | **Composition** (`children` or slot props) |
| Many distant components need the same slow-changing value | **Context**                                |
| Many components need fast-changing shared state           | **External store** (Topic 4.5)             |
| Data from a server                                        | **TanStack Query** (Module 6)              |

## 4.4 What State Belongs Where?

A common mistake is treating all state as "global state." Sort by **kind** first:

| Kind of state           | Examples                                        | Where it lives                                                                 |
| ----------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------ |
| **Local UI state**      | Input text, is dropdown open, hovered item      | `useState` in the component                                                    |
| **Shared UI state**     | Selected tab shared by two siblings             | Lift to the common parent                                                      |
| **URL state**           | Page number, search filter, selected product ID | The **URL** (search params, route params) so links and refresh work (Module 6) |
| **Global client state** | Theme, auth user, cart, notifications           | Context, Zustand, or Redux Toolkit                                             |
| **Server state**        | Products, profile, orders, any API data         | **TanStack Query** (or framework loaders)                                      |
| **Form state**          | Field values, errors, dirty flags               | Local state or React Hook Form                                                 |

The single most useful insight of the last few years: **most "global state" in old Redux apps was really server state** (cached API responses). Once you move that to a server-state library, the remaining client state is small, and `useState` plus a little Context is often enough.

### Server state is different

| Client state                         | Server state                                           |
| ------------------------------------ | ------------------------------------------------------ |
| You own it. It is always up to date. | Someone else owns it. Your copy can be **stale**.      |
| Synchronous                          | **Asynchronous**: loading, error, success              |
| No caching concerns                  | Needs caching, deduplication, refetching, invalidation |

Hand-writing that in `useState` + `useEffect` is how apps end up with loading-flag spaghetti (Module 3, Topic 3.3). Module 6 covers TanStack Query.

## 4.5 External State Libraries: Zustand and Redux Toolkit

When many components across the app share **frequently changing client state**, Context's "re-render everyone" behavior hurts. A **store** outside React, with **selectors**, fixes that. Both libraries below are built on `useSyncExternalStore` (Module 3, Topic 3.7).

### Zustand: small, simple, popular

```bash
npm install zustand
```

```jsx
// src/store/cartStore.js
import { create } from 'zustand';

export const useCartStore = create((set, get) => ({
  items: [],

  add: (product) =>
    set((state) => {
      const existing = state.items.find((i) => i.id === product.id);
      return existing
        ? {
            items: state.items.map((i) =>
              i.id === product.id ? { ...i, qty: i.qty + 1 } : i,
            ),
          }
        : { items: [...state.items, { ...product, qty: 1 }] };
    }),

  remove: (id) =>
    set((state) => ({ items: state.items.filter((i) => i.id !== id) })),

  clear: () => set({ items: [] }),

  total: () => get().items.reduce((sum, i) => sum + i.price * i.qty, 0),
}));
```

```jsx
// A component re-renders ONLY when the selected slice changes
function CartBadge() {
  const count = useCartStore((s) => s.items.length); // selector
  return <span>{count}</span>;
}

function AddButton({ product }) {
  const add = useCartStore((s) => s.add); // the action never changes
  return <button onClick={() => add(product)}>Add</button>;
}
```

No provider is needed. Persistence is a middleware: `persist` from `zustand/middleware` saves to `localStorage`.

> **Selector trap:** `useCartStore((s) => ({ a: s.a, b: s.b }))` returns a **new object each time**. Select one primitive per call, or use the library's shallow-compare helper, otherwise the component re-renders on every store change.

### Redux Toolkit (RTK): structured, great tooling, common in large codebases

```bash
npm install @reduxjs/toolkit react-redux
```

```jsx
// src/store/cartSlice.js
import { createSlice } from '@reduxjs/toolkit';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    added(state, action) {
      // RTK uses Immer: this "mutation" is safe and produces a new state
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) existing.qty += 1;
      else state.items.push({ ...action.payload, qty: 1 });
    },
    removed(state, action) {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
  },
});

export const { added, removed } = cartSlice.actions;
export default cartSlice.reducer;
```

```jsx
// src/store/index.js
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';

export const store = configureStore({ reducer: { cart: cartReducer } });
```

```jsx
// main.jsx
import { Provider } from 'react-redux';
<Provider store={store}>
  <App />
</Provider>;

// component
import { useSelector, useDispatch } from 'react-redux';
const count = useSelector((s) => s.cart.items.length);
const dispatch = useDispatch();
dispatch(added(product));
```

RTK Query (part of RTK) also handles server state, as an alternative to TanStack Query.

### Choosing between them

| Option                         | Best for                                                        | Cost                                              |
| ------------------------------ | --------------------------------------------------------------- | ------------------------------------------------- |
| `useState` + lifting           | Small trees                                                     | Drilling if the tree grows                        |
| **Context + reducer**          | Low-frequency global values (theme, auth, locale), small apps   | Re-render-all behavior, no selectors, no devtools |
| **Zustand**                    | Medium apps, fast-changing client state, minimal boilerplate    | Less enforced structure                           |
| **Redux Toolkit**              | Large teams, strict patterns, time-travel debugging, middleware | More boilerplate and concepts                     |
| **Jotai / Recoil-style atoms** | Fine-grained derived state                                      | Different mental model                            |
| **TanStack Query**             | **All server data**                                             | Another dependency, worth it                      |

Do not add a global store on day one. Start local, lift when needed, then escalate.

### Persisting state

Use a custom hook (Module 3's `useLocalStorage`) for simple local state, `persist` middleware for Zustand, or `redux-persist` for Redux. Never persist secrets or tokens in `localStorage` (Module 7, Topic 7.4).

#### Interview traps

- "Redux vs. Context?" Context passes a value. Redux (and Zustand) are **stores** with selectors, middleware, devtools and predictable update rules. Context re-renders all consumers on change. A store re-renders only components whose selected slice changed.
- "Why is Redux Toolkit allowed to 'mutate'?" It uses **Immer**, which gives you a draft proxy and produces a brand-new immutable object from your changes.
- "Do I need Redux in 2026?" Often not. Server state belongs in a query library, local state in components, and small global state in Context or Zustand. Redux Toolkit still makes sense for large apps and teams that want strict conventions.

#### Tricky questions and answers

###### Q4 [SDE-2]: Your cart badge in the header re-renders every time any item's quantity changes, but it only shows the item count. How do you fix it with Zustand?

**Answer:** Select only what you need: `useCartStore((s) => s.items.length)`. The component re-renders only when that _number_ changes. Changing a quantity does not change the length, so the badge stays still. Selecting the whole store (`useCartStore()`) subscribes to everything.

###### Q5 [SDE-3]: Design where state should live for an e-commerce page: product list, filters, cart, logged-in user, "add to cart" toast.

**Answer:** Product list: **server state** (TanStack Query, keyed by filters). Filters (category, price, page): **URL search params**, so the page is shareable and survives refresh. Cart: **global client state** (Zustand or reducer + context), persisted. Logged-in user: a small **Context** (changes rarely), backed by server state for the profile. Toast: local UI state in a provider that exposes `showToast`. Each kind of state lives in the tool built for it, instead of forcing everything into one store.

---

## Hands-On Lab: Shopping Cart with Reducer, Context and Persistence

```
1. Create CartContext.jsx with a cartReducer handling:
     "added" (increase qty if present), "removed", "quantity_changed", "cleared"
   - Pure reducer. No mutation. Throw on unknown actions.
2. Split into CartContext (state) and CartDispatchContext (dispatch).
   Export <CartProvider>, useCart() and useCartDispatch().
   useCart should throw a helpful error outside the provider.
3. Build ProductList (add-to-cart buttons), CartBadge (item count),
   and CartPanel (list, quantity inputs, total).
   - Compute total and count in render (derived values, not state).
4. Persist the cart: initialize the reducer lazily from localStorage
   and write back in a useEffect when the cart changes.
5. Open React DevTools Profiler (Module 5). Click "Add" on a product.
   Which components re-render? Does AddButton re-render if it only
   uses useCartDispatch?
6. Re-implement the same cart with Zustand using selectors.
   Compare: lines of code, re-renders, and ease of testing.
7. Write a unit test for cartReducer as a plain function (no React).
```

Questions to answer for yourself:

- Which of the two implementations re-renders fewer components, and why?
- What breaks if you move `Date.now()` for item IDs into the reducer?
- What would you move into the URL if you added a "sort by" option?

---

## Module 4 Cheat Sheet

| Concept                     | One-line interview answer                                                                             |
| --------------------------- | ----------------------------------------------------------------------------------------------------- |
| `useReducer`                | `[state, dispatch] = useReducer(reducer, init)`. Centralizes complex update logic in a pure function. |
| Reducer rules               | Pure, no mutation, no side effects, name actions by what happened.                                    |
| `useState` vs. `useReducer` | Same engine. Reducer wins when many handlers change related state or logic is complex.                |
| `dispatch` identity         | Stable across renders. Safe to pass down or use in deps.                                              |
| Prop drilling               | Passing props through components that do not use them. Fine for 2 to 3 levels.                        |
| Context                     | A broadcast channel for a value to a subtree. Not a state manager.                                    |
| React 19 context            | `<Ctx value={x}>` replaces `<Ctx.Provider>`. `use(Ctx)` can be conditional.                           |
| Context re-render rule      | Value change (by `Object.is`) re-renders **every** consumer. No selectors.                            |
| Fixing context churn        | Memoize the value, split contexts (state vs. dispatch, by domain), keep providers low.                |
| Provider + hook pattern     | Export `<XProvider>` and `useX()` that throws outside the provider.                                   |
| Composition                 | Pass components as `children` or slots to avoid drilling and extra re-renders.                        |
| Server vs. client state     | Server state is borrowed and can be stale. Use a query library. Keep client state small.              |
| URL as state                | Filters, pages and selected IDs belong in the URL so links and refresh work.                          |
| Zustand                     | Small store, selectors, no provider. Select slices to avoid re-renders.                               |
| Redux Toolkit               | Structured store, Immer inside, great devtools. For large apps and teams.                             |
| `useSyncExternalStore`      | The primitive that stores use to subscribe safely to non-React state.                                 |

---

## Browse Series

| #   | Module                                        | Read more                                                       |
| --- | --------------------------------------------- | --------------------------------------------------------------- |
| 1   | **Getting Started, JSX & Components**         | [Open Module 1](reactjs/getting-started-jsx-and-components)       |
| 2   | **State, Events & Forms**                     | [Open Module 2](reactjs/state-events-and-forms)                   |
| 3   | **Effects, Refs & Custom Hooks**              | [Open Module 3](reactjs/effects-refs-and-custom-hooks)            |
| 4   | **How React Renders & Performance**           | [Open Module 5](reactjs/rendering-performance-and-suspense)       |
| 5   | **Routing, Data Fetching & React 19 Actions** | [Open Module 6](reactjs/routing-data-fetching-and-actions)        |
| 6   | **Testing, TypeScript, Security & Capstone**  | [Open Module 7](reactjs/testing-typescript-security-and-capstone) |
