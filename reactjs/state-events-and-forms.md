---
title: "Module 2: State, Events & Forms"
description: "Learn how React components remember data with useState, why state is a snapshot, how to update objects and arrays immutably, how to structure state well, and how to build controlled forms - from first idea to interview-level detail."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/react-module-2.png"
tags: [React, useState, State, Events, Forms, Interview-Prep]
keywords: ["React useState explained", "React state is a snapshot", "Immutable state updates in React", "Controlled vs uncontrolled components", "Lifting state up", "React forms tutorial"]
---

# Module 2: State, Events & Forms

![React Module 2](/images/react-module-2.png)

## Words you will meet in this module

| Word                   | Simple meaning                                                                                    |
| ---------------------- | ------------------------------------------------------------------------------------------------- |
| **State**              | Data a component remembers between renders. Changing it triggers a re-render.                     |
| **`useState`**         | The hook that adds a piece of state to a component.                                               |
| **Setter function**    | The function `useState` returns (`setCount`) to ask React to change the state.                    |
| **Snapshot**           | The fixed values of props and state for **one** render. Handlers see that snapshot.               |
| **Updater function**   | Passing a function to the setter: `setCount((c) => c + 1)`. It receives the latest pending value. |
| **Batching**           | React groups several state updates from one event into a single re-render.                        |
| **Immutable update**   | Making a **new** object or array instead of changing the existing one.                            |
| **Derived state**      | A value you can calculate from existing state or props. It should **not** be stored as state.     |
| **Lifting state up**   | Moving state to the closest common parent so several children can share it.                       |
| **Controlled input**   | A form field whose value is held in React state.                                                  |
| **Uncontrolled input** | A form field whose value is held by the DOM and read when needed.                                 |

## 2.1 Events: Responding to the User

### The idea in plain words

An **event handler** is a function that runs when the user does something: click, type, submit, hover. In React you attach it as a prop on a JSX element.

```jsx
function SaveButton() {
  function handleClick() {
    alert('Saved!');
  }
  return <button onClick={handleClick}>Save</button>;
}
```

### The most common beginner bug: calling instead of passing

```jsx
<button onClick={handleClick()}>Save</button>   // WRONG: runs during render
<button onClick={handleClick}>Save</button>     // RIGHT: passes the function
<button onClick={() => handleClick(7)}>Save</button>  // RIGHT: wrapper to pass an argument
```

`handleClick()` is a _call_. It runs immediately while React renders, and what it returns (usually `undefined`) becomes the handler. You must pass the **function itself**, not its result.

### Event handler rules

| Rule                                     | Example                                                        |
| ---------------------------------------- | -------------------------------------------------------------- |
| Props are camelCase                      | `onClick`, `onChange`, `onSubmit`, `onKeyDown`, `onMouseEnter` |
| Handlers receive an **event object**     | `function handle(e) { console.log(e.target.value); }`          |
| Name handlers `handleX`, and props `onX` | `<Item onDelete={handleDelete} />`                             |
| Handlers can have side effects           | Unlike render, an event handler is _the_ place for them.       |

### Event object, preventDefault and bubbling

```jsx
function SignupForm() {
  function handleSubmit(e) {
    e.preventDefault(); // stop the browser's full-page form submit
    console.log('submitting');
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">Sign up</button>
    </form>
  );
}
```

Events **bubble** up the tree: a click on a button also fires `onClick` on every parent that has one. Stop it with `e.stopPropagation()`.

```jsx
<div onClick={() => console.log('div')}>
  <button
    onClick={(e) => {
      e.stopPropagation();
      console.log('button');
    }}
  >
    Click
  </button>
</div>
// Without stopPropagation: logs "button" then "div".
```

> React wraps native events in a cross-browser object (a _synthetic event_). It works like the native one. Since React 17, React attaches its listeners to the **root container**, not to `document`.

#### Interview traps

- "Why does my `onClick={doThing()}` run on load?" You called the function. Pass `doThing` or `() => doThing()`.
- "Difference between `e.preventDefault()` and `e.stopPropagation()`?" The first cancels the browser's default action (such as form submit or link navigation). The second stops the event from bubbling to parent handlers.
- "Can I use `addEventListener` in React?" Yes, in an effect for things outside your JSX (like `window` scroll). For normal elements, use the `onX` props.

## 2.2 `useState` and Why State Is a Snapshot

### The idea in plain words

A normal variable inside a function is thrown away when the function ends. A component function runs again on every render, so a normal variable cannot remember anything. **State** is how React gives a component memory.

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0); // [current value, setter]

  return (
    <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>
  );
}
```

`useState(0)` returns a pair: the **current value** (`count`) and a **setter** (`setCount`). The `0` is only used on the **first** render.

### What really happens on a click

```
 first render:   count = 0          → button shows "Clicked 0 times"
        │
 click → setCount(0 + 1)            → React stores 1 and schedules a re-render
        │
 second render:  React calls Counter() again.
                 useState returns the stored value: count = 1
                                    → button shows "Clicked 1 times"
```

React keeps state **outside** your function, tied to the component's position in the tree. Each call to `useState` is matched by its **order**, which is why hooks have rules (Module 3).

### Why plain variables and mutation do not work

```jsx
function Broken() {
  let count = 0;
  return (
    <button
      onClick={() => {
        count++;
        console.log(count);
      }}
    >
      {count}
    </button>
  );
}
```

Two things go wrong. First, `count` is reset to `0` on every render. Second, changing a local variable does not **tell React** to re-render. A setter does both jobs: it stores the new value and triggers the re-render.

### State is a snapshot

This is the most-tested idea in this module. Each render gets its **own fixed copy** of the state. A setter does **not** change the variable in the running code. It asks React for a **new render** with a new value.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1); // count is 0 in this render → asks for 1
    setCount(count + 1); // count is STILL 0        → asks for 1
    setCount(count + 1); // count is STILL 0        → asks for 1
  }
  // After one click, count is 1, not 3.
  return <button onClick={handleClick}>{count}</button>;
}
```

Inside `handleClick`, `count` is a **constant `0`** for this render (it is a closure over this render's snapshot). The three calls all say "set it to 1."

The same reason explains this:

```jsx
function handleClick() {
  setCount(count + 5);
  alert(count); // shows 0, not 5. The snapshot has not changed.
}
```

### The updater function

To base the next state on the **latest pending** state, pass a function:

```jsx
function handleClick() {
  setCount((c) => c + 1); // c is the latest queued value
  setCount((c) => c + 1);
  setCount((c) => c + 1);
}
// After one click: count is 3.
```

React queues the updater functions and runs them in order, each receiving the result of the previous one.

| Use                   | When                                                                               |
| --------------------- | ---------------------------------------------------------------------------------- |
| `setX(value)`         | The new value does not depend on the old one (`setName("Asha")`).                  |
| `setX((prev) => ...)` | The new value is **calculated from** the old one (`prev + 1`, adding to an array). |

Use the updater form inside timers, async code and anything that might run after a render has become stale (Module 3).

### Batching

When one event triggers several state updates, React groups them into **one** re-render. Since React 18 this happens everywhere: event handlers, timeouts, promises. You do not need to do anything.

```jsx
function handleClick() {
  setName('Asha');
  setAge(24);
  setIsAdmin(true);
  // ONE re-render, not three.
}
```

### Lazy initial state

If the initial value is expensive to compute, pass a **function**. React calls it only on the first render:

```jsx
const [items, setItems] = useState(() =>
  JSON.parse(localStorage.getItem('items') ?? '[]'),
);
// useState(JSON.parse(...)) would run the parse on EVERY render (the result is just ignored after the first).
```

### State belongs to a component instance

Two copies of `<Counter />` have **separate** state. State is tied to _where_ a component sits in the tree, not to the function itself.

```jsx
<Counter />   {/* its own count */}
<Counter />   {/* a different count */}
```

### State vs. plain variable vs. ref

| Need                                                                               | Use                 |
| ---------------------------------------------------------------------------------- | ------------------- |
| Value that changes and **should update the screen**                                | `useState`          |
| Value you can **calculate** during render                                          | A normal `const`    |
| Value that changes but should **not** update the screen (timer ID, previous value) | `useRef` (Module 3) |

#### Interview traps

- "Why does `setCount(count + 1)` three times add only 1?" Each call uses the same snapshot value of `count`. Use `setCount((c) => c + 1)` to chain.
- "Does `setState` update the variable immediately?" No. It schedules a new render. The variable in the _current_ render never changes.
- "Is `setState` synchronous or asynchronous?" It is better to say: it **queues** an update that is applied on the next render. The current code keeps seeing the old snapshot.
- "What does `useState` return on re-render?" The **stored** value. The initial argument is ignored after the first render.

#### Tricky questions and answers

###### Q1 [SDE-2]: What does this log, and what does the screen show after one click?

```jsx
const [n, setN] = useState(0);
function handleClick() {
  setN(n + 1);
  setN(n + 1);
  setN((x) => x + 1);
  console.log(n);
}
```

**Answer:** It logs `0` (the snapshot). The queue is: "set to 1", "set to 1", then "take the latest value and add 1." The first two replace the value with `1` each, and the updater then takes that `1` and returns `2`. The screen shows **2**.

###### Q2 [SDE-2]: Why does this `setTimeout` show an old value?

```jsx
function handleClick() {
  setCount(count + 1);
  setTimeout(() => alert(count), 3000);
}
```

**Answer:** The timeout callback is a closure over the `count` of the render in which `handleClick` was created. Even if the state changes during the 3 seconds, that closure still holds the old snapshot. This is a feature: each render's handlers are consistent with what the user saw on screen.

## 2.3 Immutability: Updating Objects and Arrays

### The rule

**Never mutate state. Replace it with a new value.** React decides whether to re-render by comparing the new state with the old state using `Object.is` (a reference check for objects and arrays). If you mutate the same object and pass it back, the reference is unchanged and React may see "nothing changed."

```jsx
const [user, setUser] = useState({ name: 'Asha', age: 24 });

user.age = 25; // WRONG: mutation. Screen may not update, and it breaks other features.
setUser(user); // Same reference. React bails out.

setUser({ ...user, age: 25 }); // RIGHT: a new object
```

### Objects: copy with spread

```jsx
const [form, setForm] = useState({
  name: 'Asha',
  address: { city: 'Kolkata', pin: '700001' },
});

// Change a top-level field
setForm({ ...form, name: 'Ravi' });

// Change a nested field: copy EVERY level you touch
setForm({
  ...form,
  address: { ...form.address, city: 'Pune' },
});
```

### Arrays: use methods that return a new array

| Task                 | Do NOT use (mutates)         | Use (returns new array)                                             |
| -------------------- | ---------------------------- | ------------------------------------------------------------------- |
| Add to the end       | `push`, `unshift`            | `[...items, newItem]`                                               |
| Remove               | `splice`, `pop`, `shift`     | `items.filter((i) => i.id !== id)`                                  |
| Replace / update one | `items[i] = x`               | `items.map((i) => i.id === id ? { ...i, done: true } : i)`          |
| Insert in the middle | `splice`                     | `[...items.slice(0, i), newItem, ...items.slice(i)]`                |
| Sort / reverse       | `sort`, `reverse` (in place) | `[...items].sort(...)`, `items.toSorted(...)`, `items.toReversed()` |

```jsx
function TodoApp() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn React', done: false },
  ]);

  const add = (text) =>
    setTodos((prev) => [
      ...prev,
      { id: crypto.randomUUID(), text, done: false },
    ]);
  const remove = (id) => setTodos((prev) => prev.filter((t) => t.id !== id));
  const toggle = (id) =>
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t)),
    );
  // ...
}
```

Creating the ID inside the **event handler** (not during render) is correct: it runs once per click.

### Deep nesting is painful: Immer

If your state is deeply nested, spread chains get ugly. **Immer** lets you write "mutating" code on a safe draft copy:

```jsx
import { useImmer } from 'use-immer';

const [person, updatePerson] = useImmer({
  name: 'Asha',
  address: { city: 'Kolkata' },
});
updatePerson((draft) => {
  draft.address.city = 'Pune';
}); // Immer produces a new object
```

The better first step is usually to **flatten** the state (Topic 2.4).

### Why immutability is required, not just "style"

1. **Change detection:** `Object.is(old, new)` is cheap. Comparing deeply would be slow.
2. **Memoization:** `memo`, `useMemo` and the React Compiler (Module 5) skip work when references are unchanged. Mutation breaks them silently.
3. **Snapshots:** past renders' handlers keep pointing at old objects. Mutating them would change the "past."
4. **Features like undo:** keeping old states is trivial if you never change them.

#### Interview traps

- "Why can't I `push` onto a state array?" It mutates the existing array. React sees the same reference and may skip the update, and old snapshots get corrupted. Create a new array.
- "Does `{ ...obj }` deep-copy?" No. It is a **shallow** copy. Nested objects are shared, so copy each level you change.
- "Is `array.sort()` safe on state?" No, it sorts **in place**. Copy first: `[...arr].sort()`, or use `toSorted()`.

#### Tricky question and answer

###### Q3 [SDE-2]: A checkbox in a list does not update when clicked. The handler does `todo.done = !todo.done; setTodos(todos);`. Why?

**Answer:** The code mutates the existing object and passes the **same array reference** back to `setTodos`. React compares the new state with the old using `Object.is`, finds the same reference, and skips the re-render. Fix it by creating new objects: `setTodos(todos.map(t => t.id === id ? { ...t, done: !t.done } : t))`.

## 2.4 Choosing a Good State Structure

Bad state structure is a bigger source of bugs than bad code. Five principles:

| #   | Principle                           | Bad                                                                             | Good                                                        |
| --- | ----------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| 1   | **Group related state**             | `const [x, setX]`, `const [y, setY]` that always change together                | `const [pos, setPos] = useState({ x: 0, y: 0 })`            |
| 2   | **Avoid contradictions**            | `isLoading` and `isSuccess` and `isError` as three booleans (can all be `true`) | One `status`: `"idle" \| "loading" \| "success" \| "error"` |
| 3   | **Avoid redundant (derived) state** | `fullName` stored beside `firstName` and `lastName`                             | `const fullName = first + " " + last` during render         |
| 4   | **Avoid duplication**               | `selectedItem` (a copy of an item) beside `items`                               | `selectedId`, and look up `items.find(...)` when needed     |
| 5   | **Avoid deep nesting**              | Tree inside tree inside tree                                                    | A flat list/map keyed by ID                                 |

### Derived values: calculate, do not store

```jsx
function Cart({ items }) {
  // GOOD: always correct, nothing to keep in sync
  const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
  const count = items.length;

  // BAD: a second source of truth that can go out of date
  // const [total, setTotal] = useState(0);
  // useEffect(() => setTotal(...), [items]);   // extra render and extra bug surface

  return (
    <p>
      {count} items, total ₹{total}
    </p>
  );
}
```

The effect version causes an extra render with a **wrong value** for one frame, and every new field adds another thing to forget. Module 3, Topic 3.4 makes this a central rule.

### Mirroring props in state

```jsx
function Bad({ color }) {
  const [c, setC] = useState(color); // copies the prop ONCE. Later prop changes are ignored.
}
```

This is only right if you _mean_ "initial value, then I own it" (name the prop `initialColor`). Otherwise read `color` directly.

#### Interview traps

- "When should something be state?" When it **changes over time**, **cannot be computed** from other state or props, and **should update the screen**.
- "Why avoid derived state?" Two sources of truth go out of sync, and syncing needs extra effects and renders.

## 2.5 Sharing State: Lifting Up, Controlled Components and `key`

### Lifting state up

Two sibling components that must stay in sync cannot share state directly. Move the state to their **closest common parent** and pass it down as props plus a callback.

```jsx
function Accordion() {
  const [openIndex, setOpenIndex] = useState(0); // owned by the parent
  return (
    <>
      <Panel
        title="About"
        isOpen={openIndex === 0}
        onShow={() => setOpenIndex(0)}
      >
        ...
      </Panel>
      <Panel
        title="Details"
        isOpen={openIndex === 1}
        onShow={() => setOpenIndex(1)}
      >
        ...
      </Panel>
    </>
  );
}

function Panel({ title, isOpen, onShow, children }) {
  return (
    <section>
      <h3>{title}</h3>
      {isOpen ? <p>{children}</p> : <button onClick={onShow}>Show</button>}
    </section>
  );
}
```

`Panel` is now a **controlled component**: its behavior is driven by props from the parent. This keeps **one source of truth**. Because only one panel can be open, the parent stores _which_ is open, not a boolean per panel (principle #2 above).

```
       Accordion (state: openIndex)
        ┌────┴─────┐
     Panel       Panel
   (isOpen via   (isOpen via
     props)        props)
```

### How React tracks state: position in the tree

React matches state to a component by its **position and type** in the tree. Same position and same type → state is preserved. Different type or removed → state is destroyed.

```jsx
{
  isFancy ? <Counter isFancy /> : <Counter />;
} // same type, same position: STATE KEPT
{
  isFancy ? (
    <div>
      <Counter />
    </div>
  ) : (
    <section>
      <Counter />
    </section>
  );
} // different types: STATE RESET
```

### Resetting state with `key`

You can intentionally reset a component's state by changing its **key**. A new key means "this is a different component."

```jsx
function Messenger({ contactId }) {
  return <Chat key={contactId} contactId={contactId} />;
}
// Switching contact gives Chat a new key → fresh draft text, no leftover state.
```

This is cleaner than writing an effect that clears state whenever `contactId` changes.

#### Interview traps

- "Where should state live?" At the **lowest common ancestor** of every component that needs it. Not higher.
- "How do you reset a component's state?" Change its `key`. Do not write an effect for this.
- "Does re-rendering a component reset its state?" No. State is preserved as long as the component stays at the same position with the same type.

## 2.6 Forms: Controlled and Uncontrolled Inputs

### Controlled input: React owns the value

```jsx
function NameField() {
  const [name, setName] = useState('');

  return (
    <label>
      Name
      <input value={name} onChange={(e) => setName(e.target.value)} />
    </label>
  );
}
```

The loop: user types → `onChange` fires → state updates → re-render → input shows `value={name}`. React is the **single source of truth**. If you give `value` without `onChange`, the input becomes read-only (React warns).

### All the common input types

```jsx
// Text, email, number, textarea, select: use e.target.value
<textarea value={bio} onChange={(e) => setBio(e.target.value)} />

<select value={country} onChange={(e) => setCountry(e.target.value)}>
  <option value="in">India</option>
  <option value="us">USA</option>
</select>

// Checkbox: use `checked` and e.target.checked
<input type="checkbox" checked={agree} onChange={(e) => setAgree(e.target.checked)} />

// Radio: checked is a comparison
<input type="radio" value="free" checked={plan === "free"} onChange={(e) => setPlan(e.target.value)} />
```

> **`null` or `undefined` trap:** `value={undefined}` makes an input _uncontrolled_, then `value="x"` flips it to controlled, and React warns. Initialize text state to `""`, not `undefined` or `null`.

### One handler for many fields

Give each input a `name` and use a computed property key:

```jsx
function SignupForm() {
  const [form, setForm] = useState({ email: '', password: '', plan: 'free' });

  function handleChange(e) {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value })); // [name] is a computed key
  }

  function handleSubmit(e) {
    e.preventDefault();
    console.log('Submitting', form);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        type="email"
        value={form.email}
        onChange={handleChange}
        required
      />
      <input
        name="password"
        type="password"
        value={form.password}
        onChange={handleChange}
        required
        minLength={8}
      />
      <select name="plan" value={form.plan} onChange={handleChange}>
        <option value="free">Free</option>
        <option value="pro">Pro</option>
      </select>
      <button type="submit">Create account</button>
    </form>
  );
}
```

### Simple validation

Validation is **derived** from the current values, so compute it in render. Do not store `errors` as extra state unless you track "touched."

```jsx
const emailError = !form.email.includes('@') ? 'Enter a valid email' : '';
const canSubmit = !emailError && form.password.length >= 8;

<button type="submit" disabled={!canSubmit}>
  Create account
</button>;
{
  emailError && <p role="alert">{emailError}</p>;
}
```

Always validate again on the **server**. Client-side checks are for user experience, not security.

### Uncontrolled input: the DOM owns the value

When you only need the value at submit time, you can skip state. Read from `FormData`:

```jsx
function SearchForm({ onSearch }) {
  function handleSubmit(e) {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    onSearch(data.get('q'));
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="q" defaultValue="" /> {/* defaultValue, not value */}
      <button>Search</button>
    </form>
  );
}
```

Uncontrolled forms re-render nothing while typing, so they are simple and fast. React 19 builds on this style with `<form action={fn}>` (Module 6, Topic 6.3).

### Controlled vs. uncontrolled

| Question                                      | Controlled           | Uncontrolled                                  |
| --------------------------------------------- | -------------------- | --------------------------------------------- |
| Who holds the value?                          | React state          | The DOM                                       |
| Set by                                        | `value` + `onChange` | `defaultValue`, read with `FormData` or a ref |
| Live validation, formatting, dependent fields | Easy                 | Harder                                        |
| Re-renders on every keystroke                 | Yes                  | No                                            |
| Best for                                      | Interactive forms    | Simple submit-only forms, file inputs         |

For big forms (many fields, complex validation), use a library: **React Hook Form** (uncontrolled-based, minimal re-renders) with **Zod** for schemas.

#### Interview traps

- "Controlled vs. uncontrolled?" See the table. State the trade-off: control costs a re-render per keystroke but gives full power over the value.
- "Why does my input not let me type?" You set `value` without an `onChange` that updates it.
- "Why do I get a 'changing an uncontrolled input to be controlled' warning?" The state started as `undefined`. Use `""`.
- "Is `<input type="file">` controlled?" It cannot be. Its value is read-only. Always uncontrolled.

#### Tricky question and answer

###### Q4 [SDE-2]: You build a list of editable rows with `key={index}`. After deleting row 1, the typed text in the other rows looks wrong. Explain.

**Answer:** With index keys, after deleting the first item, the old second item now has key `0`. React sees key `0` still exists and **reuses the DOM node and its internal state** (including uncontrolled input text) but gives it the new item's props. Items shift but their DOM nodes do not. The fix is a stable key from the data: `key={item.id}`. Then React tracks each item by identity and moves or removes the correct nodes.

---

## Hands-On Lab: Build a Shopping List

```
1. Create a ShoppingList component with state:
   items: [{ id, name, qty, bought }]
2. A form with a name input and a quantity input (type="number").
   - Controlled inputs, one handler using [name]: value
   - Submit adds an item with an immutable update ([...prev, newItem])
   - Reset the form after submit
3. Each row has: a checkbox (toggle bought with .map), a delete button (.filter)
4. Show derived values, computed in render (NOT state):
   - total items, items bought, items left
5. Add a "Show only items left" checkbox (a boolean state) and filter the list
6. Persist to localStorage using the lazy-initial-state pattern
   (reading) and an effect (writing). We will formalize this in Module 3.
7. Reset the whole list by giving the component a `key` and changing it.
```

Questions to answer for yourself:

- Which of your values are real state, and which are derived? Can you delete any state variable by deriving it?
- Replace `key={item.id}` with `key={index}`, then add a text input inside each row and delete the first row. What do you see?
- Add `console.log("render")` in the component. Click the checkbox for one item. How many renders? Why only one?

## Module 2 Cheat Sheet

| Concept              | One-line interview answer                                                                              |
| -------------------- | ------------------------------------------------------------------------------------------------------ |
| Event handler        | Pass the function (`onClick={fn}`), do not call it (`onClick={fn()}`).                                 |
| State                | Component memory. Changing it via the setter schedules a re-render.                                    |
| State is a snapshot  | Each render has fixed values. Setters ask for a _new_ render. They do not change the current variable. |
| Updater function     | `setX((prev) => next)`. Use it when the new value depends on the old one.                              |
| Batching             | Multiple updates in one event cause one re-render (everywhere since React 18).                         |
| Lazy initial state   | `useState(() => expensive())` runs the function only on the first render.                              |
| Immutability         | Never mutate state. Use spread, `map`, `filter` to create new objects and arrays.                      |
| Shallow copy         | `{ ...obj }` copies one level. Copy every level you change.                                            |
| Good state structure | Group related, avoid contradictions, avoid redundancy, avoid duplication, avoid deep nesting.          |
| Derived state        | Compute it in render. Do not store what you can calculate.                                             |
| Lifting state up     | Move state to the closest common parent. Pass down values and callbacks.                               |
| State identity       | Matched by position and type in the tree. Change `key` to reset.                                       |
| Controlled input     | `value` + `onChange`. React state is the source of truth.                                              |
| Uncontrolled input   | `defaultValue` + `FormData` or a ref. The DOM holds the value.                                         |
| Common input bug     | `value={undefined}`, or `value` without `onChange`.                                                    |
| Index as key         | Breaks state for dynamic lists. Use a stable ID.                                                       |

---

## Browse Series

| #   | Module                                        | Read more                                                       |
| --- | --------------------------------------------- | --------------------------------------------------------------- |
| 1   | **Getting Started, JSX & Components**         | [Open Module 1](reactjs/getting-started-jsx-and-components)       |
| 2   | **Effects, Refs & Custom Hooks**              | [Open Module 3](reactjs/effects-refs-and-custom-hooks)            |
| 3   | **Reducers, Context & App-Wide State**        | [Open Module 4](reactjs/reducers-context-and-app-state)           |
| 4   | **How React Renders & Performance**           | [Open Module 5](reactjs/rendering-performance-and-suspense)       |
| 5   | **Routing, Data Fetching & React 19 Actions** | [Open Module 6](reactjs/routing-data-fetching-and-actions)        |
| 6   | **Testing, TypeScript, Security & Capstone**  | [Open Module 7](reactjs/testing-typescript-security-and-capstone) |
