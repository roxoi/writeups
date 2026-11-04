---
title: "Module 1: Getting Started, JSX & Components"
description: "Learn what React is and why it exists, set up a project with Vite, understand JSX rules, build components with props and children, and render lists with keys - from first idea to interview-level detail."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/react-module-1.png"
tags: [React, JSX, Components, Props, Vite, Interview-Prep]
keywords: ["What is React", "React JSX explained", "React props and children", "React keys in lists", "Create React app with Vite", "React declarative vs imperative"]
---

# Module 1: Getting Started, JSX & Components

![React Module 1](/images/react-module-1.png)

## Words you will meet in this module

| Word            | Simple meaning                                                                                 |
| --------------- | ---------------------------------------------------------------------------------------------- |
| **Component**   | A function that returns UI. The basic building block of React.                                 |
| **JSX**         | HTML-like syntax written inside JavaScript. A compiler turns it into function calls.           |
| **Element**     | A small plain object describing one piece of UI, such as `{ type: "h1", props: {...} }`.       |
| **Props**       | Read-only inputs a parent passes to a child component.                                         |
| **Children**    | Whatever you put between a component's opening and closing tags.                               |
| **Fragment**    | An invisible wrapper (`<>...</>`) that lets you return several elements without an extra node. |
| **Key**         | A special prop that gives each list item a stable identity.                                    |
| **Declarative** | You describe _what_ the UI should be. The library works out _how_ to get there.                |
| **Bundler**     | A tool that combines your many files into a few files the browser can load (Vite, webpack).    |
| **HMR**         | Hot Module Replacement: the browser updates your code instantly without a full reload.         |
| **StrictMode**  | A development-only wrapper that runs extra checks to expose bugs early.                        |

## 1.1 What Is React and Why Does It Exist?

**React** is a JavaScript library for building user interfaces. You build the screen out of small, reusable pieces called **components**: a button, a search box, a product card, a whole page. You then combine them like building blocks.

To see _why_ React exists, look at the problem it solves.

### The problem: updating the screen by hand

Say you want a "Like" button that shows how many likes it has. In plain JavaScript you write the instructions yourself:

```js
const button = document.querySelector('#like');
const label = document.querySelector('#count');
let count = 0;

button.addEventListener('click', () => {
  count += 1;
  label.textContent = count; // YOU must remember to update the screen
});
```

This style is **imperative**: you tell the browser each step. It is fine for one button. Now imagine 200 pieces of data that appear in 30 places on the screen, and any of them can change at any time. You must remember _every_ place to update. Miss one and the screen shows stale data. That is where bugs breed.

### The React way: describe the result

```jsx
import { useState } from 'react';

function LikeButton() {
  const [count, setCount] = useState(0);

  return <button onClick={() => setCount(count + 1)}>Likes: {count}</button>;
}
```

This style is **declarative**. You do not touch the DOM. You say: "when `count` is 5, the button reads _Likes: 5_." When `count` changes, React re-runs your function, sees the new description, and updates only what changed.

```
Imperative (vanilla JS)               Declarative (React)
─────────────────────────             ─────────────────────────
"Find the label.                      "The label shows {count}."
 Set its text to 5.
 Find the badge.                      React keeps the screen in sync.
 Hide it. ..."
You manage every change.              You manage the data. React manages the screen.
```

### Three core ideas

| Idea                  | Meaning                                                                                       |
| --------------------- | --------------------------------------------------------------------------------------------- |
| **Components**        | UI is a tree of small functions. Each one owns one piece of the screen.                       |
| **Declarative UI**    | You write what the UI looks like for the current data, not the steps to change it.            |
| **One-way data flow** | Data flows **down** from parent to child through props. Events flow **up** through callbacks. |

### Library, not framework

React is only the **view layer**. It does not include a router, an HTTP client, a form library or a build tool. You pick those yourself (this series uses Vite, React Router and TanStack Query). A **framework** such as Next.js bundles React with all those choices. Part 3 of this path covers Next.js.

| Question                        | React (library)        | Next.js / React Router framework mode |
| ------------------------------- | ---------------------- | ------------------------------------- |
| Who decides how routing works?  | You (add React Router) | The framework                         |
| Who decides how data is loaded? | You                    | The framework                         |
| Server rendering built in?      | No                     | Yes                                   |
| Flexibility vs. convenience     | Flexible, more setup   | Convenient, more rules                |

### Where React runs

- **React DOM:** web browsers (this series).
- **React Native:** iOS and Android apps, using the same component ideas.
- **The server:** Server-Side Rendering and React Server Components (Module 7 previews them, Next.js covers them).

### The virtual DOM in one paragraph

When your components run, they return **elements**: lightweight JavaScript objects that describe the UI. Together they form a tree, often called the **virtual DOM**. On each update React builds a new tree, compares it with the previous tree (this is called **diffing**), and applies only the differences to the real DOM. Module 5 covers this fully.

#### Interview traps

- "Is React a framework?" No. It is a **library** for building UI. Say what it does _not_ include (routing, data fetching, bundling).
- "Is the virtual DOM faster than the real DOM?" Not by itself. Touching the real DOM directly can be faster for one tiny change. The virtual DOM's value is that it makes **declarative updates affordable**, so you never write the update logic by hand. Do not claim it is "always faster."
- "What does declarative mean?" You describe the desired result for a given state. You do not write the steps to transition between states.

#### Tricky question and answer

###### Q1 [SDE-1]: Why can't I just use `document.getElementById` inside a React component?

**Answer:** You can, but you should not for normal UI. React owns the DOM nodes it creates. If you change them behind its back, React's tree and the real DOM disagree, and your changes may be overwritten on the next render. Use state and let React update the screen. For the rare cases that truly need the DOM (focus, scroll position, measuring), use a **ref** (Module 3).

## 1.2 The JavaScript You Need Before React

React is "just JavaScript." Most beginner confusion is JavaScript confusion in disguise. These are the features you will use in almost every file. If any of them look new, practice them in the browser console first.

```js
// 1. const / let, arrow functions, template strings
const greet = (name) => `Hello, ${name}!`;

// 2. Object destructuring (with a default value)
const user = { name: 'Asha', age: 24 };
const { name, role = 'student' } = user; // role is "student"

// 3. Array destructuring (useState returns an array)
const [first, second] = ['a', 'b'];

// 4. Spread: copy and extend, without changing the original
const older = { ...user, age: 25 }; // new object
const more = [...[1, 2, 3], 4]; // new array [1, 2, 3, 4]

// 5. map / filter: turn arrays into new arrays
const nums = [1, 2, 3, 4];
const doubled = nums.map((n) => n * 2); // [2, 4, 6, 8]
const evens = nums.filter((n) => n % 2 === 0); // [2, 4]
const total = nums.reduce((sum, n) => sum + n, 0); // 10

// 6. Optional chaining and nullish coalescing
const city = user.address?.city ?? 'Unknown'; // no crash if address is missing

// 7. ES modules
// utils.js:  export const add = (a, b) => a + b;  export default function main() {}
// app.js:    import main, { add } from "./utils.js";

// 8. async / await
async function loadUser() {
  const res = await fetch('/api/user');
  const data = await res.json();
  return data;
}
```

### Truthy and falsy (needed for conditional rendering)

These values are **falsy**: `false`, `0`, `""`, `null`, `undefined`, `NaN`. Everything else is truthy, including `"0"`, `[]` and `{}`. The `&&` operator returns the first falsy value or the last value. This matters in JSX, as Topic 1.6 shows.

### Closures (the one to really understand)

A **closure** is a function that remembers the variables around it when it was created. Every React event handler is a closure over the values from the render in which it was created. This is the key to understanding why state behaves as a "snapshot" in Module 2.

```js
function makeCounter() {
  let count = 0;
  return () => ++count; // remembers `count`
}
const next = makeCounter();
next(); // 1
next(); // 2
```

## 1.3 Setting Up a Project with Vite

### What you need first

React runs in the browser, but the tools that build a React app run on **Node.js**. Install the current LTS release of Node.js. The current Vite (version 8) needs **Node.js 20.19 or later (22.12 or later on the 22 line)**. Check your version:

```bash
node -v    # should be v20.19+ or v22.12+
npm -v
```

### Create the project

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

Open the address it prints (usually `http://localhost:5173`). Newer versions of `create-vite` may ask interactive questions instead; choose **React** and **JavaScript** (or **TypeScript** if you prefer, Module 7 covers it). Bun users can run `bun create vite my-app` and `bun install`.

> **Note on Create React App:** older tutorials start with `npx create-react-app`. The React team **deprecated Create React App in February 2025**. Do not use it for new projects. Use Vite for a client-side app, or a framework such as Next.js when you need server rendering.

### What Vite creates

```
my-app/
├── index.html          ← the ONE html page; has <div id="root">
├── package.json        ← dependencies and scripts
├── vite.config.js      ← build tool configuration
├── public/             ← files served as-is (favicon, robots.txt)
└── src/
    ├── main.jsx        ← entry point: mounts React into #root
    ├── App.jsx         ← your first component
    ├── App.css
    └── index.css
```

### The entry point: `main.jsx`

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';
import './index.css';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Read it line by line:

- `createRoot(domNode)` tells React "you own this DOM node and everything inside it."
- `.render(<App />)` draws your root component into it. Everything else on the page is built from `App`.
- `<StrictMode>` adds development-only checks. It **renders every component twice** and re-runs effects once extra, to expose impure code and missing cleanup. It does nothing in production builds.

### The scripts in `package.json`

| Command           | What it does                                                          |
| ----------------- | --------------------------------------------------------------------- |
| `npm run dev`     | Starts the dev server with HMR. Use this while coding.                |
| `npm run build`   | Creates an optimized production build in `dist/`.                     |
| `npm run preview` | Serves the `dist/` build locally so you can test it before deploying. |
| `npm run lint`    | Runs ESLint.                                                          |

### Environment variables

Vite exposes only variables that start with `VITE_`, through `import.meta.env`:

```
# .env
VITE_API_URL=https://api.example.com
```

```js
const apiUrl = import.meta.env.VITE_API_URL;
```

> **Security warning:** anything with the `VITE_` prefix is **baked into the JavaScript that every visitor downloads**. Never put a secret key in it. Module 7, Topic 7.4 explains this.

### Choosing a starting point

| You want...                                            | Use                               |
| ------------------------------------------------------ | --------------------------------- |
| Learn React, build a dashboard or internal tool (SPA)  | **Vite + React** (this series)    |
| SEO, fast first load, server rendering, full-stack app | **Next.js** (Part 3 of this path) |
| Server rendering with React Router                     | React Router framework mode       |

## 1.4 JSX: HTML-Like Syntax Inside JavaScript

**JSX** lets you write markup next to your logic. It looks like HTML, but it is JavaScript. A compiler (Babel, SWC or esbuild inside Vite) converts it before the browser sees it.

```jsx
const element = <h1 className="title">Hello, React</h1>;
```

becomes, roughly:

```js
import { jsx } from 'react/jsx-runtime'; // added automatically
const element = jsx('h1', { className: 'title', children: 'Hello, React' });
```

and `jsx(...)` returns a plain object:

```js
{ type: "h1", props: { className: "title", children: "Hello, React" } }
```

So JSX is **syntax sugar**. With the modern "automatic runtime" you do **not** need `import React from "react"` at the top of every file.

### The rules of JSX

| #   | Rule                                           | Wrong                 | Right                                    |
| --- | ---------------------------------------------- | --------------------- | ---------------------------------------- |
| 1   | Return **one root** (use a Fragment if needed) | `<h1/><p/>`           | `<><h1/><p/></>`                         |
| 2   | **Close every tag**                            | `<img src="a.png">`   | `<img src="a.png" />`                    |
| 3   | Attributes are **camelCase**                   | `onclick`, `tabindex` | `onClick`, `tabIndex`                    |
| 4   | Use `className` and `htmlFor`                  | `class`, `for`        | `className`, `htmlFor`                   |
| 5   | `style` takes an **object**                    | `style="color: red"`  | `style={{ color: "red", fontSize: 16 }}` |
| 6   | `{}` holds a JavaScript **expression**         | `{if (x) ...}`        | `{x ? "yes" : "no"}`                     |
| 7   | Comments inside JSX need braces                | `<!-- note -->`       | `{/* note */}`                           |

The double braces in `style={{ color: "red" }}` are not special syntax. The outer `{}` says "JavaScript here", and the inner `{}` is an object literal.

### Expressions vs. statements

Inside `{}` you can put anything that **produces a value**:

```jsx
const user = { name: "Asha", score: 82 };

<p>Name: {user.name}</p>                       {/* variable */}
<p>Double: {user.score * 2}</p>                {/* arithmetic */}
<p>{user.score >= 50 ? "Pass" : "Fail"}</p>    {/* ternary */}
<p>{greet(user.name)}</p>                      {/* function call */}
<ul>{["a", "b"].map((x) => <li key={x}>{x}</li>)}</ul>  {/* array of elements */}
```

You **cannot** put statements (`if`, `for`, `switch`) directly inside `{}`. Compute the value above the `return`, or use a ternary or `&&`.

### JSX is escaped by default (good for security)

Values inside `{}` are converted to text, not HTML. If `name` is `"<script>alert(1)</script>"`, the user sees that text literally. It does not run. This is why React is safe against most XSS by default. The exception is `dangerouslySetInnerHTML`, covered in Module 7.

### Spreading props

```jsx
const buttonProps = { type: 'submit', disabled: false, className: 'btn' };
<button {...buttonProps}>Save</button>;
```

#### Interview traps

- "Can browsers read JSX?" No. A build tool compiles it into `jsx()` calls first.
- "Why `className` and not `class`?" `class` is a reserved word in JavaScript, and JSX attributes map to DOM property names such as `className`.
- "Why can't a component return two sibling elements?" A function returns one value, and JSX compiles to one object. Wrap them in a parent or a Fragment.

## 1.5 Components and Props

A **component** is a JavaScript function that returns JSX. Props are its inputs, just like function arguments.

```jsx
function Greeting(props) {
  return <h2>Hello, {props.name}!</h2>;
}

<Greeting name="Asha" />;
```

### Rules for components

1. **Name starts with a capital letter.** React treats lowercase tags (`<div>`) as built-in HTML elements and capitalized tags (`<Greeting />`) as your components. A lowercase name will not work.
2. **Return JSX, `null` (render nothing), or any renderable value** (string, number, array of elements).
3. **Define components at the top level of a file**, never inside another component (see the trap below).
4. **One component per responsibility.** If you can describe it with "and", split it.

### Props: destructuring, defaults, children

Destructure in the parameter list. It is cleaner, and it shows what the component needs:

```jsx
function Avatar({ src, name, size = 48 }) {      // size defaults to 48
  return (
    <img
      className="avatar"
      src={src}
      alt={`Photo of ${name}`}
      width={size}
      height={size}
    />
  );
}

<Avatar src="/asha.png" name="Asha" />            // size = 48
<Avatar src="/ravi.png" name="Ravi" size={96} />
```

Strings can be written as `name="Asha"`. Everything else (numbers, booleans, objects, functions) needs braces: `size={96}`, `isAdmin={true}`, or simply `isAdmin`.

### `children`: components that wrap other content

```jsx
function Card({ title, children }) {
  return (
    <section className="card">
      <h3>{title}</h3>
      <div className="card-body">{children}</div>
    </section>
  );
}

<Card title="Profile">
  <Avatar src="/asha.png" name="Asha" />
  <p>Frontend developer</p>
</Card>;
```

Whatever sits between `<Card>` and `</Card>` arrives as `props.children`. This is how layouts, modals, and buttons with icons are built.

### Passing functions (events go up)

Data flows **down** as props. To let a child tell its parent something happened, the parent passes a **function** as a prop and the child calls it:

```jsx
function Parent() {
  const handleDelete = (id) => console.log('delete', id);
  return <Item id={7} onDelete={handleDelete} />;
}

function Item({ id, onDelete }) {
  return <button onClick={() => onDelete(id)}>Delete</button>;
}
```

The convention is to name the prop `onSomething` and the handler in the parent `handleSomething`.

### Props are read-only

A component must **never modify its own props**. Think of props as function arguments: you do not change what the caller passed you. If a component needs to change something over time, it needs **state** (Module 2).

```jsx
function Bad({ user }) {
  user.name = 'Hacked'; // WRONG: mutating a prop
  return <p>{user.name}</p>;
}
```

### Composition over inheritance

React has no component inheritance. You build bigger things by **putting components inside components** and passing content through `children` or other props:

```jsx
function Layout({ header, sidebar, children }) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </div>
  );
}

<Layout header={<Navbar />} sidebar={<Menu />}>
  <Dashboard />
</Layout>;
```

Props can hold JSX, so you can create "slots." This is the same principle explained in [Inheritance vs. Composition](object-oriented-programing/inheritance-composition): prefer assembling small parts over deep class hierarchies.

### Component trap: defining a component inside a component

```jsx
function Page() {
  function Row() {
    return <li>Row</li>;
  } // WRONG: new function every render
  return (
    <ul>
      <Row />
    </ul>
  );
}
```

Each time `Page` renders it creates a _new_ `Row` function. React sees a different component type, throws away the old one (losing its state and DOM) and builds a new one. Always define `Row` at the top level and pass what it needs as props.

### Prop drilling (a preview)

If `App` has a `user` and a deeply nested `Avatar` needs it, passing `user` through five components that do not use it is called **prop drilling**. It is fine for two or three levels. For more, Module 4 shows Context and other solutions.

#### Interview traps

- "Props vs. state?" Props are **inputs from the parent** and read-only. State is **memory owned by the component** and changes over time.
- "How does a child talk to its parent?" The parent passes a callback function as a prop, and the child calls it.
- "Can a component change its props?" No. Treat props as immutable.
- "Why capital letters for components?" So JSX can tell `<div>` (HTML tag) from `<Div>` (your component).

## 1.6 Conditional Rendering, Lists and Keys

### Conditional rendering

JSX has no `if` attribute. You use plain JavaScript:

```jsx
function Status({ isOnline, unread }) {
  // 1. Early return: good for "nothing to show"
  if (unread === null) return null;

  return (
    <div>
      {/* 2. Ternary: choose one of two things */}
      <span>{isOnline ? '🟢 Online' : '⚪ Offline'}</span>

      {/* 3. Logical AND: show something or nothing */}
      {unread > 0 && <Badge count={unread} />}
    </div>
  );
}
```

> **The `0 &&` trap:** `{count && <Badge />}` shows a literal **`0`** on screen when `count` is `0`, because `0 && x` evaluates to `0`, and React renders numbers. Write `{count > 0 && <Badge />}` instead. Only `false`, `null`, `undefined` and `true` render nothing.

| Pattern           | Use when                                            |
| ----------------- | --------------------------------------------------- |
| `if (...) return` | Whole component has an "empty" or "loading" version |
| `cond ? A : B`    | Two alternatives                                    |
| `cond && A`       | Show A or nothing (make sure `cond` is a boolean)   |
| A variable        | Many branches. Compute above the `return`.          |

### Rendering lists with `map`

```jsx
const people = [
  { id: 1, name: 'Asha', role: 'Designer' },
  { id: 2, name: 'Ravi', role: 'Engineer' },
  { id: 3, name: 'Meera', role: 'Engineer' },
];

function TeamList() {
  const engineers = people.filter((p) => p.role === 'Engineer'); // filter first
  return (
    <ul>
      {engineers.map((p) => (
        <li key={p.id}>
          {p.name} ({p.role})
        </li>
      ))}
    </ul>
  );
}
```

### Keys: why React needs them

When a list changes (item added, removed, reordered), React must work out **which old item matches which new item**, so it can keep each item's state and DOM. The `key` prop gives each item an identity.

**Rules for keys:**

| Rule                                                                 | Why                                                                             |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Unique **among siblings** (not globally)                             | React only compares siblings.                                                   |
| **Stable**: the same item has the same key every render              | Changing keys makes React destroy and recreate the item.                        |
| Use an **ID from your data** (database ID, slug)                     | It travels with the item.                                                       |
| **Never** use `Math.random()` or `crypto.randomUUID()` during render | New key every render means everything is remounted.                             |
| Index (`i`) only for **static** lists                                | If the list can be reordered, filtered or inserted into, index keys cause bugs. |
| Put the key on the **outermost element in the `map`**                | Not on an inner child.                                                          |

When one item must produce **several** sibling elements, use the long Fragment form, because `<>` cannot take a key:

```jsx
import { Fragment } from 'react';

{
  items.map((item) => (
    <Fragment key={item.id}>
      <dt>{item.term}</dt>
      <dd>{item.definition}</dd>
    </Fragment>
  ));
}
```

> **A bug you will see in interviews:** a list of inputs using `key={index}`. Type something in the first input, delete the first item, and the typed text appears to "move" to the wrong row. The key (index 0) still exists, so React keeps the _old DOM node and its typed text_, but gives it the _new item's data_. Module 5 explains this with a diagram.

#### Interview traps

- "Why do lists need keys?" To identify items between renders so React updates, moves or removes the right ones and preserves their state. The warning is not just cosmetic.
- "Is using the index as a key always wrong?" No. It is fine for a static list that is never reordered or filtered. It is wrong for dynamic lists.
- "Do keys get passed to the component as a prop?" No. `key` is consumed by React. If the child needs the ID, pass it as a separate prop such as `id`.

## 1.7 Purity and the Rendering Mental Model

### Components must be pure

A **pure** function gives the same output for the same input and has no side effects. React expects your component **render** to be pure:

- Same props and state in → same JSX out.
- Do not modify anything that existed before the render started (outer variables, props, state, the DOM).
- Creating and changing variables _inside_ the render is fine. That is called **local mutation**.

```jsx
// IMPURE: changes a variable outside the component
let guestCount = 0;
function Cup() {
  guestCount = guestCount + 1; // side effect during render
  return <h2>Guest #{guestCount}</h2>;
}
// Rendering twice shows #1 then #2. Same props, different output.

// PURE: depends only on props
function CupPure({ guest }) {
  return <h2>Guest #{guest}</h2>;
}

// ALSO FINE: local mutation, created and used within this render
function List({ names }) {
  const items = [];
  for (const n of names) items.push(<li key={n}>{n}</li>);
  return <ul>{items}</ul>;
}
```

### Why purity matters

React may render a component **more than once**, pause a render, throw it away, or render it in the background (Module 5). That is only safe if rendering has no side effects. In development, `<StrictMode>` renders twice on purpose. If your component prints different results or has side effects, the double render exposes it.

### Where do side effects go?

- **Responding to a user action:** inside an **event handler** (Module 2).
- **Syncing with something outside React** (timers, network, browser APIs): inside **`useEffect`** (Module 3).
- **Never** during the render itself.

### The component tree

Your app is a tree of components. Data flows down, and React can reason about each branch separately.

```
            App
         ┌───┴────────┐
      Navbar        Main
     ┌──┴──┐      ┌───┴─────┐
   Logo  Menu   Sidebar   TeamList
                          ┌──┴───────┐
                      ProfileCard  ProfileCard
```

#### Tricky question and answer

###### Q2 [SDE-2]: My component logs "rendered" twice on every update in development. Is this a bug?

**Answer:** No. This is `<StrictMode>` rendering each component twice in development to check that your render is pure, and re-running effects once extra to check that you clean up properly. It does not happen in production. If you see different output on the two renders, you have a side effect in the render body: fix that, do not remove StrictMode.

## 1.8 Styling Components

| Approach                | How                                           | Good for                             | Watch out for                                    |
| ----------------------- | --------------------------------------------- | ------------------------------------ | ------------------------------------------------ |
| **Plain CSS file**      | `import "./App.css"` and use `className`      | Small apps, learning                 | Class names are **global**, so collisions happen |
| **CSS Modules**         | File named `Card.module.css`                  | Scoped styles with zero runtime cost | A bit more typing                                |
| **Inline `style`**      | `style={{ color: "red" }}`                    | Dynamic one-off values               | No hover, media queries or pseudo-classes        |
| **Tailwind CSS**        | Utility classes: `className="p-4 rounded-lg"` | Fast, consistent UI. Very popular.   | Long class strings. Learn the utility names.     |
| **CSS-in-JS (runtime)** | styled-components, Emotion                    | Dynamic theming                      | Runtime cost. Poor fit with Server Components.   |

CSS Modules example:

```css
/* Card.module.css */
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 16px;
}
.title {
  font-weight: 600;
}
```

```jsx
import styles from './Card.module.css';

function Card({ title, children }) {
  return (
    <section className={styles.card}>
      <h3 className={styles.title}>{title}</h3>
      {children}
    </section>
  );
}
```

Vite renames `.card` to something unique like `_card_x7a2b_1`, so it cannot clash with another file's `.card`.

Combining classes conditionally:

```jsx
<button className={`btn ${isPrimary ? 'btn-primary' : ''}`}>Go</button>
// or use the tiny `clsx` package: className={clsx("btn", { "btn-primary": isPrimary })}
```

### Interview traps

- "Why not mutate `style` strings like in HTML?" JSX `style` is an object with camelCase keys (`backgroundColor`), because it maps to the DOM `element.style` properties.
- "CSS Modules vs. Tailwind?" Modules scope hand-written CSS per file. Tailwind removes hand-written CSS by composing small utility classes. Both avoid global collisions, in different ways.

### Tricky questions and answers

#### Q3 [SDE-1]: What is the difference between a component and an element?

**Answer:** A **component** is a function (or class) that you write, such as `Avatar`. An **element** is the plain object that describes one thing on screen, such as `{ type: Avatar, props: { size: 48 } }`. `<Avatar size={48} />` creates an _element_ that _refers to_ the `Avatar` component. React calls your component function to turn elements into more elements, until only built-in HTML elements remain.

#### Q4 [SDE-2]: Why does `{items.length && <List />}` show a `0`?

**Answer:** `0 && x` evaluates to `0`, and React renders numbers as text. Use `items.length > 0 && <List />` or a ternary with `null`. Only `false`, `null`, `undefined` and `true` render nothing.

#### Q5 [SDE-2]: What happens if two siblings in a list share the same key?

**Answer:** React warns in development, and its behavior becomes unpredictable. It cannot tell the two items apart, so on updates it may duplicate, drop or wrongly reuse items and their state. Keys must be unique among siblings.

---

## Hands-On Lab: Build a Team Directory

Create a new Vite project and build this by hand. Do not copy-paste, type it.

```
1. npm create vite@latest team-directory -- --template react
2. Make a file src/data.js that exports an array of 6 people:
   { id, name, role, city, photo, isOnline }
3. Create src/components/ProfileCard.jsx
   - props: name, role, city, photo, isOnline, size (default 64)
   - shows an <img> with a proper alt, the name, role and city
   - shows "Online" or "Offline" with a ternary
4. Create src/components/Card.jsx that accepts `title` and `children`,
   and wrap each ProfileCard in it.
5. In App.jsx, render the list with .map and the correct key (the person's id).
6. Show only people where role === "Engineer" (use .filter before .map).
7. Add a heading that shows "No results" when the filtered list is empty
   (use an early return or a ternary, and avoid the `0 &&` trap).
8. Style it with a CSS Module.
```

Questions to answer for yourself:

- What happens if you change `key={person.id}` to `key={index}` here? Does anything visibly break? (Probably not yet. Module 2 adds state and shows the bug.)
- Open React DevTools, the **Components** tab. Can you see your tree and each component's props?
- Wrap the app in `<StrictMode>` and add `console.log("render", name)` inside `ProfileCard`. How many logs per card? Why?

## Module 1 Cheat Sheet

| Concept               | One-line interview answer                                                                      |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| React                 | A JavaScript **library** for building UI from components. Not a framework.                     |
| Declarative           | Describe the UI for a given state. React works out the DOM changes.                            |
| Virtual DOM           | A tree of plain-object elements. React diffs old vs. new and updates only the differences.     |
| JSX                   | Syntax sugar for `jsx()` calls. Compiled by the build tool. One root, close tags, `className`. |
| `{}` in JSX           | Holds an **expression** (value), never a statement.                                            |
| Component             | A function with a **capital-letter name** that takes props and returns JSX.                    |
| Props                 | Read-only inputs, flowing **down**. Never mutate them.                                         |
| `children`            | The content between a component's tags. The basis of composition.                              |
| Events go up          | The parent passes a callback in props. The child calls it.                                     |
| Conditional rendering | Early return, ternary, `&&` (beware `0 &&`).                                                   |
| Lists                 | `array.map(...)` with a stable, unique **key** from your data.                                 |
| Key rules             | Unique among siblings, stable across renders, never random, index only for static lists.       |
| Purity                | Same props and state → same JSX. No side effects during render.                                |
| StrictMode            | Dev-only. Double-renders to expose impure code and missing cleanup.                            |
| Nested component trap | Never define a component inside another. It is remounted on every render.                      |
| Vite vs. CRA          | Vite is the standard for new SPAs. Create React App is deprecated (Feb 2025).                  |
| `VITE_` variables     | Exposed to the browser. Never put secrets in them.                                             |

---

## Browse Series

| #   | Module                                        | Read more                                                       |
| --- | --------------------------------------------- | --------------------------------------------------------------- |
| 1   | **State, Events & Forms**                     | [Open Module 2](reactjs/state-events-and-forms)                   |
| 2   | **Effects, Refs & Custom Hooks**              | [Open Module 3](reactjs/effects-refs-and-custom-hooks)            |
| 3   | **Reducers, Context & App-Wide State**        | [Open Module 4](reactjs/reducers-context-and-app-state)           |
| 4   | **How React Renders & Performance**           | [Open Module 5](reactjs/rendering-performance-and-suspense)       |
| 5   | **Routing, Data Fetching & React 19 Actions** | [Open Module 6](reactjs/routing-data-fetching-and-actions)        |
| 6   | **Testing, TypeScript, Security & Capstone**  | [Open Module 7](reactjs/testing-typescript-security-and-capstone) |
