# React / Next.js / TypeScript Interview Questions

**Your Self-Assessment:** 8/10  
**Focus:** Advanced concepts, edge cases, performance, and JavaScript internals

---

## Question 1: React Concurrent Rendering

**Difficulty:** Advanced  
**Category:** Strength Validation

Explain how React's concurrent rendering works under the hood. How does React decide what to render, and what role does the scheduler play?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

React's concurrent rendering is built on three core concepts:

1. **Time Slicing:** React breaks rendering work into chunks and can pause, yield, or resume work based on priority. This is achieved through the `MessageChannel` API (not `setTimeout`).

2. **Fiber Architecture:** Each React element is represented as a Fiber node (a JavaScript object with `return`, `child`, `sibling` pointers). This creates a linked list tree that React can traverse incrementally.

3. **Scheduler & Lanes:** React uses a priority lane system (not a simple queue). Each update gets a priority lane, and the scheduler picks the highest-priority work. Low-priority work can be interrupted.

**Key Mechanisms:**
- `requestIdleCallback` polyfill via `MessageChannel`
- Double-buffering (current tree vs. work-in-progress tree)
- Reconciliation is interruptible; commit phase is not
- `useTransition` and `useDeferredValue` hook into this system

**Key Concepts:**
- Concurrency ≠ Parallelism (still single-threaded JS)
- Cooperative scheduling
- Priority lanes (SyncLane, InputContinuousLane, DefaultLane, TransitionLane, etc.)

**Common Mistakes:**
- Thinking React is multi-threaded (it's not)
- Believing `useTransition` makes code run faster (it keeps UI responsive)
- Forgetting that state updates during render are restricted

**Interview Tip:**
> "React's concurrent rendering uses a Fiber architecture with priority lanes. The scheduler can interrupt low-priority work for high-priority updates like user input. This isn't parallelism—it's cooperative multitasking on a single thread, achieved through `MessageChannel` for time slicing."

</details>

---

## Question 2: Server Components vs. Client Components

**Difficulty:** Advanced  
**Category:** Strength Validation

When would you use a Server Component vs. a Client Component in Next.js 13+? What are the trade-offs?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Server Components (Default in App Router):**
- Render on the server, never ship JavaScript for them
- Can directly access backend resources (databases, file systems, APIs)
- Cannot use hooks, event handlers, or browser APIs
- Reduce bundle size, improve initial page load
- Better for SEO and data-heavy pages

**Client Components (`"use client"` directive):**
- Render on client and server (hydration)
- Can use state, effects, event handlers, browser APIs
- Ship JavaScript to the browser
- Interactive UI elements

**Trade-offs:**

| Aspect | Server Components | Client Components |
|--------|-------------------|-------------------|
| Bundle size | Smaller | Larger |
| Interactivity | None | Full |
| Data access | Direct backend | API calls only |
| SEO | Excellent | Requires SSR/SSG |
| Hydration cost | None | Yes |

**Decision Framework:**
- Static content, data fetching → Server Component
- Forms, clicks, state → Client Component
- Data + interactivity → Server Component with Client Component child

**Key Concepts:**
- RSC payload (React Server Component serialization)
- Boundary placement (keep client boundaries low in the tree)
- Server Actions for mutations from server components

**Common Mistakes:**
- Making entire pages client components ("use client" at the top)
- Trying to use hooks in server components
- Not understanding that client components are still SSR'd

**Interview Tip:**
> "I default to Server Components for everything. I only add 'use client' when I need interactivity, and I keep the client boundary as low in the component tree as possible to minimize the hydration footprint."

</details>

---

## Question 3: TypeScript Conditional Types

**Difficulty:** Advanced  
**Category:** Learning

What is a TypeScript conditional type? Implement a type that extracts the return type of an async function.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Conditional types** have the form `T extends U ? X : Y`. They distribute over union types when `T` is a naked type parameter.

**Example 1: Extract Async Return Type**

```typescript
type MyAwaited<T> = T extends Promise<infer U> 
  ? U extends Promise<any> 
    ? MyAwaited<U> 
    : U
  : T;

// Usage
type A = MyAwaited<Promise<string>>;  // string
type B = MyAwaited<Promise<Promise<number>>>;  // number
```

**Example 2: Filter Union Types**

```typescript
type NonFunction<T> = T extends Function ? never : T;

type Result = NonFunction<string | number | (() => void)>; 
// string | number
```

**Example 3: Distributive vs. Non-Distributive**

```typescript
type ToArray<T> = T extends any ? T[] : never;  // Distributive: string[] | number[]
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;  // Non-distributive: (string | number)[]
```

**Key Concepts:**
- `infer` keyword for type extraction
- Distributive behavior over unions
- `never` for filtering
- Recursive conditional types
- Built-in: `ReturnType`, `Parameters`, `Awaited`

**Common Mistakes:**
- Forgetting that `T extends U` distributes over unions
- Not wrapping in brackets `[T] extends [U]` to prevent distribution when needed
- Over-engineering simple type problems

**Interview Tip:**
> "Conditional types let you create type-level logic. The `infer` keyword is powerful for extracting types from generic structures. I use distributive types to filter unions and non-distributive (with brackets) when I need to treat the union as a whole."

</details>

---

## Question 4: JavaScript Event Loop

**Difficulty:** Intermediate  
**Category:** Strength Validation

Explain the JavaScript event loop. What's the difference between microtasks and macrotasks? Provide an example where this distinction matters.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**The Event Loop Process:**

1. **Call Stack:** Synchronous code executes here
2. **Web APIs/Node APIs:** Async operations (`setTimeout`, `fetch`, `Promise`) are delegated here
3. **Task Queue (Macrotasks):** Callbacks from `setTimeout`, `setInterval`, I/O
4. **Microtask Queue (Job Queue):** `Promise.then()`, `queueMicrotask()`, `MutationObserver`
5. **Render Queue:** Browser repaints (before next paint)

**Execution Order:**
1. Execute all synchronous code (call stack)
2. Drain the **microtask queue** completely
3. Take **one macrotask** from the task queue
4. Render (if needed)
5. Repeat

**Example: Output Order**

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

queueMicrotask(() => console.log('4'));

console.log('5');

// Output: 1, 5, 3, 4, 2
```

**Why It Matters:**

```javascript
// Problem: UI freezes with many promises
for (let i = 0; i < 10000; i++) {
  Promise.resolve().then(() => heavyWork()); // Blocks render
}

// Solution: Yield to event loop
async function processBatch(items) {
  for (const item of items) {
    processItem(item);
    await new Promise(r => setTimeout(r, 0)); // Yield
  }
}
```

**Key Concepts:**
- Microtasks have **higher priority** than macrotasks
- All microtasks run before the next macrotask
- `process.nextTick` (Node.js) runs **before** other microtasks
- `requestAnimationFrame` runs before the next paint

**Common Mistakes:**
- Assuming `setTimeout(fn, 0)` runs immediately
- Not understanding why Promise chains can block rendering
- Confusing Node.js vs. browser event loop (Node has additional phases)

**Interview Tip:**
> "JavaScript is single-threaded, but the event loop enables async behavior. Microtasks (Promises) always run before the next macrotask (setTimeout). This is why Promise chains can block rendering if they run in a tight loop—you need to explicitly yield with `await setTimeout(r, 0)`."

</details>

---

## Question 5: Custom Hooks Pattern

**Difficulty:** Intermediate  
**Category:** Learning

Design a `useDebounce` hook that returns a debounced value. What edge cases should you handle?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Implementation:**

```typescript
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(timer);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

**Edge Cases to Handle:**

1. **Unmount cleanup:** Always clear timeout in cleanup function ✓
2. **Rapid value changes:** Cleanup cancels previous timer ✓
3. **Delay changes:** Dependency array includes `delay` ✓
4. **Initial value:** Should return immediately, not after delay
5. **Leading vs. trailing edge:** This is trailing (fires after delay)

**Enhanced Version with Leading Edge:**

```typescript
export function useDebounce<T>(
  value: T, 
  delay: number, 
  options: { leading?: boolean; trailing?: boolean } = {}
): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  const leadingRef = useRef(true);

  useEffect(() => {
    if (options.leading && leadingRef.current) {
      setDebouncedValue(value);
      leadingRef.current = false;
      return;
    }

    const timer = setTimeout(() => {
      setDebouncedValue(value);
      leadingRef.current = true;
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

**Advanced: Debounced Callback Hook**

```typescript
export function useDebouncedCallback<T extends (...args: any[]) => any>(
  callback: T,
  delay: number
): T {
  const callbackRef = useRef(callback);
  const timerRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
    callbackRef.current = callback;
  }, [callback]);

  return useCallback(
    ((...args) => {
      if (timerRef.current) clearTimeout(timerRef.current);
      timerRef.current = setTimeout(() => {
        callbackRef.current(...args);
      }, delay);
    }) as T,
    [delay]
  );
}
```

**Key Concepts:**
- `useRef` for mutable values that don't trigger re-renders
- `useCallback` for stable function references
- Cleanup functions prevent memory leaks
- Stale closure problem (solved with refs)

**Common Mistakes:**
- Forgetting cleanup function (memory leak)
- Not handling the initial value correctly
- Using `setTimeout` without tracking the timer ID
- Creating a new callback on every render (breaks memoization)

**Interview Tip:**
> "I use `useRef` to track the latest callback, preventing stale closures. The cleanup function is critical—if the component unmounts or the value changes, I clear the pending timer to avoid setting state on an unmounted component."

</details>

---

## Question 6: Next.js Caching Strategies

**Difficulty:** Advanced  
**Category:** Strength Validation

Explain the four caching layers in Next.js App Router. When does each get invalidated?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**The Four Caching Layers:**

1. **Request Memoization** (Per-Request, Server-Side)
   - Deduplicates `fetch()` calls within a single request
   - Lifetime: Single server request
   - Invalidated: When request ends

2. **Data Cache** (Persistent, Server-Side)
   - Stores `fetch()` results across requests
   - Lifetime: Configurable (default: indefinite for static, no cache for dynamic)
   - Invalidated: `revalidatePath()`, `revalidateTag()`, time-based `revalidate`

3. **Full Route Cache** (Persistent, Server-Side)
   - Caches the rendered HTML and RSC payload
   - Lifetime: Indefinite for static routes
   - Invalidated: Build time, `revalidatePath()`, or route segment config

4. **Router Cache** (Client-Side)
   - Stores RSC payload in browser memory
   - Lifetime: Session-based (30s for dynamic, 5min for static)
   - Invalidated: User refresh, navigation, or `router.refresh()`

**Configuration:**

```typescript
// Per-fetch caching
fetch('https://api.example.com/data', {
  next: { 
    revalidate: 3600,      // Time-based
    tags: ['products']     // Tag-based
  }
});

// Per-route caching
export const revalidate = 60;  // ISR
export const dynamic = 'force-dynamic';  // No caching
```

**Opting Out:**

```typescript
// Server Actions (default: no cache)
'use server';
export async function createPost() { }

// Cookies/Headers (forces dynamic rendering)
import { cookies } from 'next/headers';
const cookieStore = cookies(); // Makes route dynamic
```

**Key Concepts:**
- `revalidateTag()` for granular invalidation
- `unstable_cache` for non-fetch data
- Static vs. dynamic rendering
- PPR (Partial Prerendering) - static shell + dynamic holes

**Common Mistakes:**
- Assuming all `fetch()` calls are cached
- Not using tags for targeted invalidation
- Confusing client-side Router Cache with server-side Data Cache
- Forgetting that `cookies()` or `headers()` makes routes dynamic

**Interview Tip:**
> "Next.js has four caching layers: request memoization (per-request dedup), data cache (fetch results), full route cache (HTML/RSC), and router cache (client-side). I use `revalidateTag()` for targeted invalidation and tag my fetches from the start to avoid stale data issues."

</details>

---

## Question 7: Prototypal Inheritance

**Difficulty:** Intermediate  
**Category:** Learning

Explain JavaScript's prototypal inheritance. How does the prototype chain work? What's the difference between `__proto__` and `prototype`?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Core Concepts:**

Every JavaScript object has a hidden property `[[Prototype]]` (accessed via `__proto__` or `Object.getPrototypeOf()`). When accessing a property, JavaScript walks up the prototype chain until it finds the property or reaches `null`.

**Function `prototype` vs. Object `__proto__`:**

```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function() {
  return `Hello, I'm ${this.name}`;
};

const alice = new Person('Alice');

console.log(alice.greet()); // "Hello, I'm Alice"
console.log(alice.__proto__ === Person.prototype); // true
console.log(Person.__proto__ === Function.prototype); // true
```

**Key Differences:**
- `prototype`: Property on **functions**, used when creating instances with `new`
- `__proto__`: Property on **objects**, points to the prototype it inherits from
- `Object.getPrototypeOf()` is the modern, safe way to access `__proto__`

**Prototype Chain Lookup:**

```javascript
const obj = { a: 1 };
const child = Object.create(obj);
child.b = 2;

// child -> obj -> Object.prototype -> null
console.log(child.a); // 1 (found in obj via prototype chain)
console.log(child.hasOwnProperty('a')); // false (inherited)
```

**ES6 Classes are Syntactic Sugar:**

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  speak() {
    return `${this.name} barks`;
  }
}
```

**Key Concepts:**
- `Object.create(proto)` creates object with specified prototype
- `instanceof` checks prototype chain
- `hasOwnProperty()` checks own properties only
- Property shadowing (own property hides prototype property)

**Common Mistakes:**
- Confusing `prototype` (function property) with `__proto__` (object property)
- Modifying built-in prototypes (e.g., `Array.prototype`) - bad practice
- Not understanding that `Object.create(null)` creates object with no prototype
- Forgetting that arrow functions don't have a `prototype` property

**Interview Tip:**
> "Every object has a `[[Prototype]]`. Functions have a `prototype` property used when creating instances with `new`. The prototype chain is how JavaScript achieves inheritance. ES6 classes are syntactic sugar over this prototypal system—`class` methods go on `Class.prototype`."

</details>

---

## Question 8: Performance Optimization

**Difficulty:** Advanced  
**Category:** Strength Validation

Your Next.js app has a 500KB initial bundle and slow Time to Interactive. Walk through your optimization strategy.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Diagnosis Phase:**

1. **Analyze the bundle:**
   ```bash
   ANALYZE=true npm run build
   ```
   Use `@next/bundle-analyzer` to visualize.

2. **Profile in browser:**
   - Chrome DevTools → Performance tab
   - React DevTools Profiler
   - Lighthouse audit

**Optimization Strategies:**

**1. Code Splitting**

```typescript
// Route-level (automatic in Next.js)
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <Skeleton />,
  ssr: false
});
```

**2. Tree Shaking & Dead Code Elimination**

```typescript
// Bad: imports entire library
import _ from 'lodash';

// Good: imports only what's needed
import debounce from 'lodash/debounce';
```

**3. Optimize Dependencies**

```typescript
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', 'date-fns']
  }
};
```

**4. Server Components & Streaming**

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <Header />
      <Suspense fallback={<Skeleton />}>
        <SlowDataSection /> {/* Streams when ready */}
      </Suspense>
    </div>
  );
}
```

**5. Image & Font Optimization**

```typescript
import Image from 'next/image';
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] }); // Self-hosted, no FOUT
```

**6. Memoization**

```typescript
const ExpensiveComponent = React.memo(({ data }) => {
  return <div>{data.map(...)}</div>;
});

const handleClick = useCallback(() => {
  // ...
}, [dependency]);
```

**7. State Management**

- Avoid Context for high-frequency updates (causes re-renders)
- Use external stores (Zustand, Jotai) for global state
- Co-locate state to reduce re-render scope

**Key Concepts:**
- FCP (First Contentful Paint), LCP (Largest Contentful Paint), TTI (Time to Interactive)
- Hydration cost vs. bundle size
- Critical rendering path
- Resource hints (`preload`, `prefetch`, `preconnect`)

**Common Mistakes:**
- Over-memoizing (adds overhead for simple components)
- Using `useMemo` for everything (premature optimization)
- Not measuring before optimizing
- Loading polyfills unnecessarily

**Interview Tip:**
> "I start with measurement—bundle analyzer and Lighthouse. Then I apply the 80/20 rule: route-level code splitting and converting client components to server components usually give the biggest wins. I avoid premature optimization and always measure the impact of each change."

</details>

---

## Question 9: TypeScript Mapped Types

**Difficulty:** Advanced  
**Category:** Learning

Create a `DeepReadonly<T>` type that makes all properties (including nested) readonly. How do you handle edge cases like arrays and functions?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Basic Implementation:**

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object 
    ? T[K] extends Function 
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};
```

**Edge Case Handling:**

```typescript
// Handle arrays (arrays are objects but should remain mutable in some cases)
type DeepReadonly<T> = T extends Function
  ? T
  : T extends Array<infer U>
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

**Usage Examples:**

```typescript
interface User {
  name: string;
  address: {
    city: string;
    zip: number;
  };
  hobbies: string[];
}

type ReadonlyUser = DeepReadonly<User>;
// {
//   readonly name: string;
//   readonly address: {
//     readonly city: string;
//     readonly zip: number;
//   };
//   readonly hobbies: readonly string[];
// }

const user: ReadonlyUser = {
  name: 'Alice',
  address: { city: 'NYC', zip: 10001 },
  hobbies: ['reading']
};

user.address.city = 'LA'; // ❌ TypeError
user.hobbies.push('coding'); // ❌ TypeError
```

**Advanced: Handling Date, Map, Set**

```typescript
type DeepReadonly<T> = T extends BuiltIn
  ? T
  : T extends (...args: any[]) => any
  ? T
  : T extends Array<infer U>
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends ReadonlyArray<infer U>
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

type BuiltIn = Date | Map<any, any> | Set<any> | RegExp | Error;
```

**Key Concepts:**
- Mapped types iterate over keys: `{ [K in keyof T]: ... }`
- `infer` for extracting generic parameters
- Conditional types for type-level logic
- Recursive types for nested structures
- Built-in: `Readonly<T>`, `Partial<T>`, `Required<T>`

**Common Mistakes:**
- Not handling the recursion base case (causes infinite recursion)
- Forgetting to preserve function types
- Not considering that `Array` and `Map` need special handling
- Over-constraining types (making everything readonly including methods)

**Interview Tip:**
> "DeepReadonly needs three checks: is it a function (preserve as-is), is it an array (use ReadonlyArray), is it a plain object (recurse). Without these checks, you either get infinite recursion or break the type. TypeScript's built-in `Readonly` only goes one level deep."

</details>

---

## Question 10: Closures and Memory

**Difficulty:** Intermediate  
**Category:** Learning

What is a closure? Provide an example where a closure can cause a memory leak, and how to prevent it.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

A **closure** is a function that remembers variables from its lexical scope even when the function is executed outside that scope.

**Basic Example:**

```javascript
function createCounter() {
  let count = 0; // Closed over variable
  return function() {
    return ++count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
// `count` is still alive in memory
```

**Memory Leak Example:**

```javascript
function attachListeners() {
  const hugeData = new Array(1000000).fill('data');
  
  document.getElementById('button').addEventListener('click', function() {
    console.log(hugeData.length); // Closure captures hugeData
  });
}

attachListeners(); // hugeData is now retained in memory
// Even if hugeData is not needed elsewhere, it can't be GC'd
// because the event listener holds a reference to it
```

**Solutions:**

**1. Extract Only What You Need**

```javascript
function attachListeners() {
  const hugeData = new Array(1000000).fill('data');
  const length = hugeData.length; // Extract primitive
  
  document.getElementById('button').addEventListener('click', function() {
    console.log(length); // Only captures `length`, not hugeData
  });
}
```

**2. Clean Up Event Listeners**

```javascript
function attachListeners() {
  const hugeData = new Array(1000000).fill('data');
  const button = document.getElementById('button');
  
  const handler = function() {
    console.log(hugeData.length);
  };
  
  button.addEventListener('click', handler);
  
  // Provide cleanup
  return () => button.removeEventListener('click', handler);
}

const cleanup = attachListeners();
// Later: cleanup();
```

**3. Use WeakMap/WeakSet**

```javascript
const listeners = new WeakMap();

function attachListeners(element) {
  const hugeData = new Array(1000000).fill('data');
  
  const handler = function() {
    console.log(hugeData.length);
  };
  
  element.addEventListener('click', handler);
  listeners.set(element, handler);
}

// When element is GC'd, handler is also GC'd
```

**Key Concepts:**
- Lexical scoping
- Garbage collection and reachability
- `WeakMap`/`WeakSet` for weak references
- Event listener cleanup in React (`useEffect` cleanup)

**Common Mistakes:**
- Not removing event listeners in React components
- Capturing large objects in timers/intervals
- Thinking closures are "bad" (they're essential for functional programming)

**Interview Tip:**
> "Closures are powerful but can cause memory leaks if you capture large objects in long-lived callbacks. The fix is to either extract what you need, clean up references, or use `WeakMap` for automatic garbage collection. In React, the `useEffect` cleanup function is critical for this."

</details>

---

## Question 11: React State Batching

**Difficulty:** Intermediate  
**Category:** Strength Validation

How does React batch state updates? How did this change in React 18?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Pre-React 18 (Legacy Mode):**

State updates inside event handlers were batched, but updates in promises, timeouts, or native event handlers were **not**:

```javascript
// React 17: Batched
function handleClick() {
  setCount(c => c + 1);
  setName('Alice');
  // Re-render happens once
}

// React 17: NOT batched
setTimeout(() => {
  setCount(c => c + 1);
  setName('Alice');
  // Re-render happens twice ❌
}, 1000);
```

**React 18 (Automatic Batching):**

All state updates are batched, regardless of where they occur:

```javascript
// React 18: Batched everywhere
setTimeout(() => {
  setCount(c => c + 1);
  setName('Alice');
  // Re-render happens once ✓
}, 1000);

fetch('/api').then(() => {
  setCount(c => c + 1);
  setName('Alice');
  // Re-render happens once ✓
});
```

**How It Works:**

1. Updates are queued in a "render queue"
2. React schedules a single re-render
3. All updates are applied atomically
4. `flushSync()` can opt out (for legacy use cases)

**Opt-Out with `flushSync`:**

```javascript
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setCount(c => c + 1);
  });
  // DOM is updated here
  
  flushSync(() => {
    setName('Alice');
  });
  // DOM is updated here
  // Two separate renders (usually not what you want)
```

**Key Concepts:**
- Automatic batching reduces re-renders
- Improves performance in async code
- `flushSync` is an escape hatch
- Concurrent features rely on batching

**Common Mistakes:**
- Using `flushSync` unnecessarily (defeats the purpose)
- Assuming `useState` updates are synchronous (they're not)
- Not understanding that multiple `setState` calls in the same function are batched

**Interview Tip:**
> "React 18 introduced automatic batching for all state updates, including those in promises, timeouts, and native event handlers. This was a breaking change in behavior—code that worked in React 17 might re-render fewer times in React 18, which is usually a performance win."

</details>

---

## Question 12: Discriminated Unions

**Difficulty:** Intermediate  
**Category:** Learning

What is a discriminated union in TypeScript? Design a type-safe state machine for a fetch operation.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

A **discriminated union** (tagged union) is a union type where each member has a common property (the "discriminant") with a literal type, allowing TypeScript to narrow the type based on that property.

**State Machine Example:**

```typescript
type FetchState<T> = 
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function handleState<T>(state: FetchState<T>) {
  switch (state.status) {
    case 'idle':
      // state is { status: 'idle' }
      return 'Not started';
    
    case 'loading':
      // state is { status: 'loading' }
      return 'Loading...';
    
    case 'success':
      // state is { status: 'success'; data: T }
      return `Data: ${state.data}`;
    
    case 'error':
      // state is { status: 'error'; error: Error }
      return `Error: ${state.error.message}`;
  }
}
```

**Real-World Use Case:**

```typescript
type AsyncResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

function processResult<T>(result: AsyncResult<T>) {
  if (result.ok) {
    // TypeScript knows result.value exists
    console.log(result.value);
  } else {
    // TypeScript knows result.error exists
    console.error(result.error);
  }
}
```

**Component Example:**

```typescript
type ButtonProps = 
  | { variant: 'primary'; onClick: () => void }
  | { variant: 'link'; href: string }
  | { variant: 'disabled' };

function Button(props: ButtonProps) {
  switch (props.variant) {
    case 'primary':
      return <button onClick={props.onClick}>Click</button>;
    case 'link':
      return <a href={props.href}>Link</a>;
    case 'disabled':
      return <button disabled>Disabled</button>;
  }
}
```

**Key Concepts:**
- Common discriminant property (usually `type`, `kind`, `status`)
- TypeScript narrows the union in conditional blocks
- Exhaustiveness checking with `never` type
- Better than optional properties (makes invalid states unrepresentable)

**Exhaustiveness Check:**

```typescript
type Status = 'idle' | 'loading' | 'success' | 'error';

function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${x}`);
}

function handleStatus(status: Status) {
  switch (status) {
    case 'idle': return 'Idle';
    case 'loading': return 'Loading';
    // Missing 'success' and 'error' cases
    default: return assertNever(status); // ❌ Type error
  }
}
```

**Common Mistakes:**
- Using optional properties instead of unions (allows invalid states)
- Forgetting the discriminant property
- Not using exhaustiveness checks in switch statements

**Interview Tip:**
> "Discriminated unions make invalid states unrepresentable. Instead of `isLoading: boolean, data?: T, error?: Error`, I use `{ status: 'loading' } | { status: 'success'; data: T }`. TypeScript's type narrowing ensures you handle each case correctly."

</details>

---

## Question 13: Next.js Middleware

**Difficulty:** Advanced  
**Category:** Strength Validation

What is Next.js Middleware? What are its limitations, and when should you use it vs. API routes?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

Next.js Middleware is code that runs **before** a request is completed. It can modify the response, redirect, rewrite, or set headers.

**Basic Example:**

```typescript
// middleware.ts (at project root)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token');
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*']
};
```

**Use Cases:**
- Authentication & authorization
- A/B testing
- Geolocation-based redirects
- Rate limiting
- Header manipulation
- Bot detection

**Limitations:**

| Aspect | Limitation |
|--------|-----------|
| Runtime | Edge runtime only (no Node.js APIs) |
| Bundle size | 1MB limit |
| Database | No direct DB access (use HTTP) |
| APIs | Limited Web APIs available |
| Execution time | Must be fast (runs on every matched request) |

**Middleware vs. API Routes:**

| Use Middleware When | Use API Routes When |
|---------------------|---------------------|
| Modifying request/response | Business logic |
| Authentication checks | Database operations |
| Redirects/rewrites | Complex computations |
| Setting headers | Third-party API calls |
| Bot detection | File uploads/downloads |

**Advanced: A/B Testing**

```typescript
export function middleware(request: NextRequest) {
  const bucket = Math.random() < 0.5 ? 'a' : 'b';
  const response = NextResponse.next();
  response.cookies.set('bucket', bucket);
  return response;
}
```

**Key Concepts:**
- Runs on Edge Runtime (V8 isolates, not Node.js)
- Executes before pages and API routes
- Can rewrite, redirect, or return responses
- `matcher` config controls which routes it runs on

**Common Mistakes:**
- Trying to use Node.js APIs in middleware
- Doing heavy computation (slows every request)
- Using middleware for complex business logic (use API routes)
- Not testing edge cases (matcher patterns can be tricky)

**Interview Tip:**
> "Middleware runs on the Edge runtime before requests complete. I use it for cross-cutting concerns like auth, redirects, and A/B testing. For business logic, I use API routes which have access to Node.js APIs and databases. Middleware is fast and lightweight—it shouldn't do heavy work."

</details>

---

## Question 14: React Suspense and Error Boundaries

**Difficulty:** Advanced  
**Category:** Learning

How do Suspense and Error Boundaries work together? Design a data-fetching component that handles loading and errors gracefully.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Concepts:**

- **Suspense:** Catches promises thrown during render, shows fallback UI
- **Error Boundary:** Catches errors thrown during render, shows error UI
- **Together:** Handle loading and error states declaratively

**Implementation:**

```typescript
// ErrorBoundary.tsx
import { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error caught:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

**Data Fetching with Suspense:**

```typescript
// Modern: Using React Query or SWR with Suspense
import { useSuspenseQuery } from '@tanstack/react-query';

function UserProfile({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId)
  });
  
  return <div>{user.name}</div>;
}

// Custom hook with Suspense
function useSuspenseFetch<T>(url: string): T {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);

  if (error) throw error;
  if (!data) {
    throw fetch(url)
      .then(r => r.json())
      .then(setData)
      .catch(setError);
  }

  return data;
}
```

**Combining Both:**

```typescript
function App() {
  return (
    <ErrorBoundary fallback={<ErrorUI />}>
      <Suspense fallback={<LoadingUI />}>
        <UserProfile userId="123" />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**Nested Boundaries:**

```typescript
function Dashboard() {
  return (
    <div>
      <Header />
      <ErrorBoundary fallback={<HeaderError />}>
        <Suspense fallback={<HeaderSkeleton />}>
          <UserHeader />
        </Suspense>
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<ContentError />}>
        <Suspense fallback={<ContentSkeleton />}>
          <MainContent />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

**Key Concepts:**
- Suspense only catches **promises** (not async/await)
- Error boundaries only catch **render errors** (not event handlers)
- Can nest both for granular error handling
- Use `useTransition` for non-urgent updates

**Common Mistakes:**
- Throwing promises outside Suspense boundaries
- Trying to catch event handler errors with error boundaries
- Not providing fallback UI
- Over-using error boundaries (creates confusing UX)

**Interview Tip:**
> "Suspense handles loading states declaratively—I throw a promise, and React shows the fallback until it's resolved. Error boundaries catch render errors. Together, they replace try/catch in components. I nest them for granular control: one boundary per feature, not one for the whole app."

</details>

---

## Question 15: TypeScript `unknown` vs. `any`

**Difficulty:** Beginner  
**Category:** Learning

What's the difference between `any` and `unknown`? When should you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`any` - The Escape Hatch:**

```typescript
let value: any = 5;
value.toUpperCase(); // ❌ Runtime error, but TypeScript doesn't care
value.foo.bar.baz; // ❌ Runtime error, but no compile error
value(); // ❌ Runtime error, but no compile error
```

- Disables type checking entirely
- Can be assigned to anything, and anything can be assigned to it
- Use when migrating from JavaScript or when you truly don't know the type

**`unknown` - The Type-Safe Alternative:**

```typescript
let value: unknown = 5;
value.toUpperCase(); // ❌ Compile error: Object is of type 'unknown'
value.foo.bar.baz; // ❌ Compile error

// Must narrow the type first
if (typeof value === 'string') {
  value.toUpperCase(); // ✓ Now it's a string
}

// Or use type guards
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

if (isString(value)) {
  value.toUpperCase(); // ✓ Type-safe
}
```

**Key Differences:**

| Aspect | `any` | `unknown` |
|--------|-------|-----------|
| Assignability | Anywhere | Anywhere (but must narrow before use) |
| Type safety | None | Enforced |
| Operations | Allowed without checks | Requires narrowing |
| Use case | Escape hatch | External data, error handling |

**When to Use Each:**

```typescript
// Use `unknown` for:
// 1. User input
function handleInput(input: unknown) {
  if (typeof input === 'string') {
    // ...
  }
}

// 2. JSON.parse return
const data: unknown = JSON.parse(jsonString);

// 3. Catch clause variables
try {
  // ...
} catch (error: unknown) {
  if (error instanceof Error) {
    console.error(error.message);
  }
}

// Use `any` for:
// 1. Third-party libraries without types
declare const legacyLib: any;

// 2. Gradual migration from JavaScript
let legacy: any = getOldCode();

// 3. Truly dynamic data (use sparingly)
```

**Key Concepts:**
- `unknown` is the type-safe counterpart to `any`
- `never` represents values that never occur
- Type narrowing with `typeof`, `instanceof`, or custom type guards
- Discriminated unions work well with `unknown`

**Common Mistakes:**
- Using `any` to silence TypeScript errors (defeats the purpose)
- Not narrowing `unknown` before use
- Using `as any` casts when you should define proper types
- Confusing `unknown` with `never` (never is the bottom type)

**Interview Tip:**
> "I use `unknown` for external data—API responses, user input, catch blocks. It forces me to narrow the type before using it, which catches bugs at compile time. I use `any` only as a last resort for untyped third-party libraries. `any` is an escape hatch, not a default."

</details>

---

## Question 16: React 19 New Features

**Difficulty:** Advanced  
**Category:** Strength Validation

What are the key new features in React 19? How do `use()`, Server Actions, and the `useFormStatus` hook work?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Key New Features in React 19:**

**1. `use()` Hook (Resource Reading)**

```typescript
import { use } from 'react';

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // Suspends until resolved
  return <div>{user.name}</div>;
}

// Can also unwrap Context
function Component() {
  const theme = use(ThemeContext); // Works with conditional contexts
  return <div style={{ color: theme.primary }}>Hello</div>;
}
```

**2. Server Actions**

```typescript
// app/actions.ts
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title');
  await db.posts.create({ title });
  revalidatePath('/posts');
}

// app/page.tsx
import { createPost } from './actions';

export default function Page() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

**3. `useFormStatus()` Hook**

```typescript
'use client';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}
```

**4. `useFormState()` Hook**

```typescript
'use client';
import { useFormState } from 'react-dom';

async function action(prevState: any, formData: FormData) {
  // Server-side validation
  const error = await validate(formData);
  return error ? { error } : { success: true };
}

function Form() {
  const [state, formAction] = useFormState(action, { error: null });
  return (
    <form action={formAction}>
      {state.error && <p>{state.error}</p>}
      <input name="email" />
    </form>
  );
}
```

**5. `useOptimistic()` Hook**

```typescript
function Comments({ comments, addComment }: Props) {
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (state, newComment) => [...state, { ...newComment, pending: true }]
  );

  async function handleSubmit(formData: FormData) {
    addOptimistic({ text: formData.get('text') });
    await addComment(formData);
  }
}
```

**6. Other Improvements:**
- `ref` is now a prop (no more `forwardRef`)
- `<Context>` provider as JSX
- Automatic preload/prefetch
- Improved error reporting

**Key Concepts:**
- Server Actions reduce client-side state management
- `use()` can read promises and context conditionally
- Optimistic updates improve UX
- Server-side validation with `useFormState`

**Common Mistakes:**
- Forgetting `'use server'` directive
- Not handling pending states in forms
- Mixing server and client component patterns incorrectly
- Not using `useOptimistic` for better UX

**Interview Tip:**
> "React 19 simplifies data fetching with `use()` and Server Actions, which let me call server functions from client components without API routes. I use `useOptimistic` for instant UI feedback, and `useFormStatus` for loading states. The `ref` as a prop change eliminates the need for `forwardRef`."

</details>

---

## Question 17: Bundle Splitting Strategies

**Difficulty:** Intermediate  
**Category:** Strength Validation

Compare route-based, component-based, and vendor splitting. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Three Main Strategies:**

**1. Route-Based Splitting (Automatic in Next.js)**

```typescript
// Next.js does this automatically
// Each page in app/ or pages/ becomes a separate chunk
// Loaded on navigation

// Manual with React.lazy
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

**When to use:** Different pages with different dependencies

**2. Component-Based Splitting**

```typescript
// Split heavy components
const Chart = dynamic(() => import('./Chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false
});

const RichTextEditor = dynamic(() => import('./RichTextEditor'), {
  loading: () => <EditorSkeleton />
});

function ArticlePage() {
  return (
    <article>
      <h1>Title</h1>
      <RichTextEditor />
      <Chart data={data} />
    </article>
  );
}
```

**When to use:** Heavy components not needed on initial render

**3. Vendor Splitting**

```typescript
// Webpack config
module.exports = {
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

**Next.js handles this automatically** by separating framework code (`framework-*.js`), lib code (`lib-*.js`), and shared code (`shared-*.js`).

**When to use:** Large third-party libraries

**Comparison Table:**

| Strategy | Granularity | Use Case | Impact |
|----------|-------------|----------|--------|
| Route-based | Page-level | Multi-page apps | High (initial load) |
| Component-based | Component-level | Heavy UI components | Medium (after navigation) |
| Vendor | Library-level | Large dependencies | Medium (caching) |

**Advanced: Conditional Loading**

```typescript
// Load only when needed
const AdminPanel = dynamic(() => import('./AdminPanel'), {
  loading: () => <Skeleton />,
  ssr: false // Don't render on server
});

function App() {
  const [isAdmin, setIsAdmin] = useState(false);
  
  return (
    <div>
      {isAdmin && <AdminPanel />}
    </div>
  );
}
```

**Key Concepts:**
- `React.lazy()` for code splitting
- `Suspense` boundary required
- `dynamic()` in Next.js with options
- Webpack `splitChunks` for vendor code
- Prefetching strategies

**Common Mistakes:**
- Over-splitting (too many small chunks = more HTTP requests)
- Not using `Suspense` with `React.lazy`
- Splitting small components (overhead > benefit)
- Not considering server-side rendering impact

**Interview Tip:**
> "I use route-based splitting for different pages (Next.js does this automatically). For heavy components like charts or editors, I use `dynamic()` with `ssr: false`. Vendor splitting is handled by Next.js's default webpack config. I avoid over-splitting—each chunk should be at least 30-50KB to be worth the HTTP request."

</details>

---

## Question 18: TypeScript Decorators

**Difficulty:** Advanced  
**Category:** Learning

What are TypeScript decorators? Implement a custom `@debounce` decorator for methods.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

**Decorators** are functions that modify classes, methods, properties, or parameters. They're an experimental feature (Stage 3) but widely used in frameworks like NestJS, TypeORM, and Angular.

**Types of Decorators:**

```typescript
// 1. Class Decorator
function Logger(constructor: Function) {
  console.log(`Class created: ${constructor.name}`);
}

@Logger
class User { }

// 2. Method Decorator
function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${propertyKey} with`, args);
    return original.apply(this, args);
  };
}

class Calculator {
  @Log
  add(a: number, b: number) {
    return a + b;
  }
}

// 3. Property Decorator
function Readonly(target: any, propertyKey: string) {
  Object.defineProperty(target, propertyKey, {
    writable: false
  });
}

// 4. Parameter Decorator
function Param(target: any, propertyKey: string, parameterIndex: number) {
  console.log(`Parameter ${parameterIndex} in ${propertyKey}`);
}
```

**Implementation: `@Debounce` Decorator**

```typescript
function Debounce(ms: number) {
  return function(
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    let timeoutId: NodeJS.Timeout;

    descriptor.value = function(...args: any[]) {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => {
        originalMethod.apply(this, args);
      }, ms);
    };

    return descriptor;
  };
}

// Usage
class SearchComponent {
  @Debounce(300)
  handleInput(query: string) {
    console.log('Searching:', query);
    // API call
  }
}

const search = new SearchComponent();
search.handleInput('a'); // Cancelled
search.handleInput('ab'); // Cancelled
search.handleInput('abc'); // Executes after 300ms
```

**Advanced: `@Memoize` Decorator**

```typescript
function Memoize() {
  return function(
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    const cache = new WeakMap();

    descriptor.value = function(...args: any[]) {
      const key = JSON.stringify(args);
      if (!cache.has(this)) {
        cache.set(this, new Map());
      }
      const methodCache = cache.get(this)!;
      
      if (methodCache.has(key)) {
        return methodCache.get(key);
      }
      
      const result = originalMethod.apply(this, args);
      methodCache.set(key, result);
      return result;
    };

    return descriptor;
  };
}
```

**Configuration:**

```json
// tsconfig.json
{
  "experimentalDecorators": true,
  "emitDecoratorMetadata": true
}
```

**Key Concepts:**
- Decorator factories (functions that return decorators)
- `PropertyDescriptor` for method decorators
- Metadata reflection API
- Stage 3 ECMAScript proposal (not standard yet)

**Common Mistakes:**
- Forgetting to enable `experimentalDecorators`
- Not handling `this` binding correctly
- Using decorators without understanding the order of execution
- Over-using decorators (hard to debug)

**Interview Tip:**
> "Decorators are metadata functions that wrap classes, methods, or properties. I've used them in Angular and NestJS for dependency injection and validation. For the `@Debounce` pattern, I store the timeout ID and clear it on each call. Decorators are powerful but should be used sparingly—they can make code harder to debug."

</details>

---

## Question 19: React Hydration

**Difficulty:** Intermediate  
**Category:** Learning

What is hydration? What are common hydration errors, and how do you prevent them?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

**Hydration** is the process where React takes server-rendered HTML and "attaches" event listeners and state to make it interactive. The server-rendered HTML becomes a "hydrated" React tree.

**The Process:**

1. **Server:** React renders components to HTML string
2. **Browser:** Browser displays HTML immediately (fast FCP)
3. **Client:** React loads, runs components again to build virtual DOM
4. **Reconciliation:** React compares virtual DOM with server HTML
5. **Hydration:** React attaches event listeners to existing DOM

**Common Hydration Errors:**

**1. Date/Time Mismatches**

```typescript
// ❌ Server renders "Jan 1, 2024" but client renders "Jan 2, 2024"
function Timestamp() {
  return <div>{new Date().toLocaleString()}</div>;
}

// ✓ Solution: Use suppressHydrationWarning or client-only
function Timestamp() {
  const [time, setTime] = useState<string>('');
  
  useEffect(() => {
    setTime(new Date().toLocaleString());
  }, []);
  
  return <div suppressHydrationWarning>{time}</div>;
}
```

**2. Conditional Rendering Based on Client State**

```typescript
// ❌ Server doesn't know about localStorage
function Component() {
  const isLoggedIn = localStorage.getItem('token'); // ❌ Server crash
  return <div>{isLoggedIn ? 'Welcome' : 'Login'}</div>;
}

// ✓ Solution: Use useEffect for client-only logic
function Component() {
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  
  useEffect(() => {
    setIsLoggedIn(!!localStorage.getItem('token'));
  }, []);
  
  return <div>{isLoggedIn ? 'Welcome' : 'Login'}</div>;
}
```

**3. Random Values**

```typescript
// ❌ Server and client generate different random values
function Component() {
  return <div>{Math.random()}</div>;
}

// ✓ Solution: Use a seed or generate on client only
function Component() {
  const [value, setValue] = useState(0);
  
  useEffect(() => {
    setValue(Math.random());
  }, []);
  
  return <div>{value}</div>;
}
```

**4. Browser-Only APIs**

```typescript
// ❌ window is not defined on server
function Component() {
  return <div>{window.innerWidth}</div>;
}

// ✓ Solution: Check for window or use useEffect
function Component() {
  const [width, setWidth] = useState(0);
  
  useEffect(() => {
    setWidth(window.innerWidth);
  }, []);
  
  return <div>{width}</div>;
}
```

**Prevention Strategies:**

```typescript
// 1. Use 'use client' for interactive components
'use client';
function InteractiveComponent() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// 2. Use Next.js dynamic with ssr: false
const ClientOnlyComponent = dynamic(() => import('./Component'), {
  ssr: false
});

// 3. Use suppressHydrationWarning sparingly
<div suppressHydrationWarning>
  {new Date().toISOString()}
</div>

// 4. Render identical content on server and client
// Avoid: Math.random(), Date.now(), localStorage, window
```

**Key Concepts:**
- Hydration mismatch = server and client render different HTML
- React 18: can recover from mismatches (re-renders entire tree)
- React 19: improved error messages
- SSR vs. SSG vs. CSR trade-offs

**Common Mistakes:**
- Using browser APIs during render
- Rendering time-dependent content
- Not testing with JavaScript disabled
- Ignoring hydration warnings

**Interview Tip:**
> "Hydration mismatches happen when server and client render different HTML. Common causes: Date, Math.random, localStorage, window. I prevent them by using `useEffect` for client-only logic, `'use client'` directive for interactive components, and `dynamic()` with `ssr: false` for browser-only features. React 18+ can recover, but it's still a performance hit."

</details>

---

## Question 20: Web Performance Metrics

**Difficulty:** Intermediate  
**Category:** Strength Validation

Explain Core Web Vitals (LCP, FID/INP, CLS). How do you optimize each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**The Three Core Web Vitals:**

**1. Largest Contentful Paint (LCP)**
- **What:** Time until the largest content element is rendered
- **Good:** < 2.5 seconds
- **Measures:** Loading performance

**Optimization:**
```typescript
// Preload critical images
<link rel="preload" as="image" href="hero.webp" />

// Use Next.js Image component
import Image from 'next/image';
<Image
  src="/hero.webp"
  alt="Hero"
  width={1920}
  height={1080}
  priority // Preload LCP image
  sizes="100vw"
/>

// Optimize server response time
// - Use CDN
// - Enable caching
// - Optimize database queries
// - Use SSR for fast initial render
```

**2. Interaction to Next Paint (INP) - Replaced FID in 2024**
- **What:** Time from user interaction to next visual update
- **Good:** < 200 milliseconds
- **Measures:** Responsiveness

**Optimization:**
```typescript
// Break up long tasks
async function processLargeDataset(data: any[]) {
  const chunks = chunk(data, 100);
  for (const chunk of chunks) {
    processChunk(chunk);
    await new Promise(r => setTimeout(r, 0)); // Yield to main thread
  }
}

// Use Web Workers for heavy computation
const worker = new Worker('./worker.js');
worker.postMessage(largeData);

// Defer non-critical work
import { startTransition } from 'react';
startTransition(() => {
  // Non-urgent update
});
```

**3. Cumulative Layout Shift (CLS)**
- **What:** Sum of layout shift scores for unexpected shifts
- **Good:** < 0.1
- **Measures:** Visual stability

**Optimization:**
```typescript
// Always specify image dimensions
<Image src="/photo.jpg" width={800} height={600} alt="..." />

// Reserve space for dynamic content
<div style={{ minHeight: '200px' }}>
  {data && <ExpensiveComponent />}
</div>

// Avoid inserting content above existing content
// ❌ Bad: Banner appears and pushes content down
useEffect(() => {
  setShowBanner(true);
}, []);

// ✓ Good: Reserve space or use portal
<div style={{ minHeight: showBanner ? '100px' : '0' }}>
  {showBanner && <Banner />}
</div>

// Use font-display: swap with fallback metrics
@font-face {
  font-family: 'Custom';
  src: url('font.woff2');
  font-display: swap;
  size-adjust: 100%;
  ascent-override: 90%;
}
```

**Measurement Tools:**
- Lighthouse (Chrome DevTools)
- Web Vitals Chrome extension
- PageSpeed Insights
- Chrome User Experience Report (field data)
- Vercel Analytics (for Next.js)

**Key Concepts:**
- LCP: Loading performance
- INP: Responsiveness (replaced FID)
- CLS: Visual stability
- Field data vs. lab data
- Real User Monitoring (RUM)

**Common Mistakes:**
- Optimizing for lab data but ignoring field data
- Not measuring INP (just optimizing for FID)
- Ignoring mobile performance
- Not setting image dimensions

**Interview Tip:**
> "Core Web Vitals are LCP (loading), INP (responsiveness), and CLS (stability). I optimize LCP with image preloading and CDN, INP by breaking up long tasks and using web workers, and CLS by reserving space for dynamic content. I measure with Lighthouse and Vercel Analytics for real user data."

</details>

---

## Question 21: React 19 `use()` Hook Deep Dive

**Difficulty:** Advanced  
**Category:** Learning

How does the `use()` hook differ from `useContext()`? Can it be used outside of components?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`use()` vs. `useContext()`:**

| Aspect | `useContext()` | `use()` |
|--------|----------------|---------|
| Conditional usage | ❌ No | ✅ Yes |
| Resources | Context only | Context + Promises |
| Suspense integration | ❌ No | ✅ Yes |
| Server Components | ❌ No | ✅ Yes |

**Key Difference: Conditional Usage**

```typescript
// ❌ useContext cannot be used conditionally
function Component({ showTheme }: { showTheme: boolean }) {
  if (showTheme) {
    const theme = useContext(ThemeContext); // ❌ Rules of Hooks violation
  }
}

// ✓ use() can be used conditionally
function Component({ showTheme }: { showTheme: boolean }) {
  if (showTheme) {
    const theme = use(ThemeContext); // ✅ Works
  }
}
```

**Reading Promises:**

```typescript
// Server Component
async function fetchUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // Suspends until resolved
  return <div>{user.name}</div>;
}

// Parent
function App() {
  const userPromise = fetchUser('123');
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

**Reading Context Conditionally:**

```typescript
function Button({ variant }: { variant: 'primary' | 'secondary' }) {
  if (variant === 'primary') {
    const theme = use(ThemeContext);
    return <button style={{ background: theme.primary }}>Click</button>;
  }
  return <button>Click</button>;
}
```

**Can `use()` Be Used Outside Components?**

**No, with exceptions:**

```typescript
// ❌ Outside component - error
const theme = use(ThemeContext);

// ❌ In regular function - error
function getTheme() {
  return use(ThemeContext);
}

// ✅ In component or custom hook
function useTheme() {
  return use(ThemeContext); // ✅ Works in custom hook
}

// ✅ In if blocks within component
function Component({ condition }) {
  if (condition) {
    const theme = use(ThemeContext); // ✅ Works
  }
}
```

**Server Components Support:**

```typescript
// Server Component
async function ServerComponent() {
  const user = await fetchUser('123'); // No use() needed
  return <div>{user.name}</div>;
}

// Client Component with promise
'use client';
function ClientComponent({ userPromise }: Props) {
  const user = use(userPromise); // ✅ Works
  return <div>{user.name}</div>;
}
```

**Error Handling:**

```typescript
function Component({ dataPromise }: Props) {
  const data = use(dataPromise);
  return <div>{data.name}</div>;
}

// Wrap in Error Boundary
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Loading />}>
    <Component dataPromise={fetchData()} />
  </Suspense>
</ErrorBoundary>
```

**Key Concepts:**
- `use()` is a "universal" resource reader
- Can read context conditionally
- Integrates with Suspense for promises
- Works in Server and Client Components
- More flexible than `useContext()`

**Common Mistakes:**
- Using `use()` outside React components
- Not wrapping promise reads in Suspense
- Confusing `use()` with `useContext()`
- Using `use()` for non-React resources

**Interview Tip:**
> "`use()` is a new resource reader that can read both context and promises. Unlike `useContext()`, it can be used conditionally and integrates with Suspense. I use it for conditional context access and reading promises in client components. It's not a replacement for `useContext()`—it's more flexible."

</details>

---

## Question 22: TypeScript `satisfies` Operator

**Difficulty:** Intermediate  
**Category:** Learning

What does the `satisfies` operator do? How is it different from type annotations?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

The `satisfies` operator (TypeScript 4.9+) checks that a value matches a type **without changing the inferred type**. It validates while preserving the most specific type.

**Type Annotation (Changes Type):**

```typescript
type Config = {
  endpoint: string;
  retries?: number;
};

const config: Config = {
  endpoint: 'https://api.example.com',
  retries: 3
};

// Type of `config.retries` is `number | undefined`
// We lost the literal type information
```

**`satisfies` (Preserves Type):**

```typescript
const config = {
  endpoint: 'https://api.example.com',
  retries: 3
} satisfies Config;

// Type of `config.retries` is `3` (literal type preserved!)
// Still type-checked against Config
```

**Practical Use Cases:**

**1. Exhaustive Object Keys**

```typescript
const routes = {
  home: '/',
  about: '/about',
  contact: '/contact'
} satisfies Record<string, string>;

// Autocomplete works for 'home' | 'about' | 'contact'
type RouteKey = keyof typeof routes;
// 'home' | 'about' | 'contact'
```

**2. Color Palette Validation**

```typescript
const colors = {
  primary: '#007bff',
  secondary: '#6c757d',
  danger: '#dc3545'
} satisfies Record<string, `#${string}`>;

// Each color is still a literal string type
colors.primary.toUpperCase(); // Type: string (preserved)
```

**3. Environment Variables**

```typescript
const env = {
  API_URL: 'https://api.example.com',
  PORT: '3000',
  NODE_ENV: 'development'
} satisfies Record<string, string>;

// env.PORT is still '3000' literal type
const port: 3000 = env.PORT as 3000;
```

**4. API Response Validation**

```typescript
const response = {
  status: 200,
  data: { name: 'Alice' }
} satisfies {
  status: number;
  data: { name: string };
};

// response.status is 200 (literal), not just number
```

**Comparison:**

| Aspect | Type Annotation | `satisfies` |
|--------|----------------|-------------|
| Changes type | ✅ Yes | ❌ No |
| Preserves literals | ❌ No | ✅ Yes |
| Type checking | ✅ Yes | ✅ Yes |
| Autocomplete | Limited | Full |

**Key Concepts:**
- Validates type compatibility
- Preserves inferred type information
- Works with literal types
- Useful for configuration objects
- Different from `as` (which forces type)

**Common Mistakes:**
- Using `satisfies` when you want to widen the type
- Confusing with `as` (type assertion)
- Not using it for config objects (loses literal info)
- Over-using (sometimes plain annotation is clearer)

**Interview Tip:**
> "I use `satisfies` for configuration objects and color palettes where I want type validation but need to preserve literal types. With a type annotation, `'#007bff'` becomes `string`. With `satisfies`, it stays as a literal. It's especially useful for exhaustive object keys and API responses."

</details>

---

## Question 23: Next.js Parallel Routes

**Difficulty:** Advanced  
**Category:** Strength Validation

What are Parallel Routes in Next.js 13+? When would you use them vs. nested layouts?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

**Parallel Routes** allow rendering multiple pages in the same layout simultaneously, each with its own loading and error states. They're defined using named slots prefixed with `@`.

**Example Structure:**

```
app/
├── layout.tsx
├── page.tsx
├── @analytics/
│   └── page.tsx
├── @team/
│   └── page.tsx
└── @notifications/
    └── page.tsx
```

**Layout:**

```typescript
// app/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
  notifications
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
  notifications: React.ReactNode;
}) {
  return (
    <div>
      <div>{children}</div>
      <div className="dashboard">
        <div>{analytics}</div>
        <div>{team}</div>
        <div>{notifications}</div>
      </div>
    </div>
  );
}
```

**Use Cases:**

**1. Dashboard with Independent Sections**

```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({
  user,
  revenue,
  notifications
}: {
  user: React.ReactNode;
  revenue: React.ReactNode;
  notifications: React.ReactNode;
}) {
  return (
    <div className="grid">
      <section>{user}</section>
      <section>{revenue}</section>
      <section>{notifications}</section>
    </div>
  );
}
```

Each slot has its own loading and error states:

```
app/
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── @user/
│   │   ├── page.tsx
│   │   ├── loading.tsx
│   │   └── error.tsx
│   ├── @revenue/
│   │   ├── page.tsx
│   │   └── loading.tsx
│   └── @notifications/
│       ├── page.tsx
│       └── error.tsx
```

**2. Modals with Parallel Routes**

```typescript
// app/@modal/default.tsx
export default function Default() {
  return null;
}

// app/photos/[id]/page.tsx
export default function PhotoPage() {
  return <div>Photo {params.id}</div>;
}

// app/photos/@modal/(.)[id]/page.tsx
export default function PhotoModal({ params }: { params: { id: string } }) {
  return (
    <Modal>
      <Photo id={params.id} />
    </Modal>
  );
}
```

**Parallel Routes vs. Nested Layouts:**

| Aspect | Parallel Routes | Nested Layouts |
|--------|-----------------|----------------|
| Structure | Same level, named slots | Parent-child hierarchy |
| Loading states | Independent per slot | Shared in nested layout |
| Error boundaries | Independent per slot | Shared in nested layout |
| URL structure | Single URL | Nested URLs |
| Use case | Dashboard sections | Multi-page apps |

**Intercepting Routes:**

Combine with intercepting routes for modal-like behavior:

```
app/
├── photos/
│   ├── page.tsx
│   ├── [id]/
│   │   └── page.tsx
│   └── @modal/
│       ├── default.tsx
│       ├── (.)[id]/
│       │   └── page.tsx  // Intercept same level
│       └── (..)[id]/
│           └── page.tsx  // Intercept parent level
```

**Key Concepts:**
- Slots defined with `@` prefix
- Each slot can have its own `loading.tsx`, `error.tsx`, `not-found.tsx`
- Must include `default.tsx` to handle unmatched routes
- Works with intercepting routes for modals
- Independent sub-navigation per slot

**Common Mistakes:**
- Forgetting `default.tsx` files
- Not understanding the URL structure
- Using parallel routes for simple layouts
- Confusing with route groups `(folder)`

**Interview Tip:**
> "Parallel Routes render multiple pages in the same layout, each with independent loading and error states. I use them for dashboards where different sections load at different speeds. For modals, I combine them with intercepting routes. For most apps, nested layouts are simpler—parallel routes are for complex UIs with independent states."

</details>

---

## Question 24: React Server Components Data Fetching

**Difficulty:** Advanced  
**Category:** Learning

How do you fetch data in React Server Components? Compare with client-side fetching patterns.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Server Component Data Fetching (Next.js App Router):**

```typescript
// app/products/page.tsx (Server Component by default)
async function ProductsPage() {
  // Direct database access
  const products = await db.products.findMany();
  
  // Or API call
  const res = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 } // ISR
  });
  const products = await res.json();
  
  return (
    <div>
      {products.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  );
}
```

**No `useEffect` needed!** Data fetching happens on the server.

**Patterns:**

**1. Sequential Fetching (Slow)**

```typescript
async function Page() {
  const user = await fetchUser();
  const posts = await fetchPosts(user.id); // Waits for user
  return <div>{posts.length} posts</div>;
}
```

**2. Parallel Fetching (Fast)**

```typescript
async function Page() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ]);
  return <div>{posts.length} posts</div>;
}
```

**3. Streaming with Suspense**

```typescript
import { Suspense } from 'react';

async function Page() {
  return (
    <div>
      <Suspense fallback={<Skeleton />}>
        <SlowSection />
      </Suspense>
      <FastSection />
    </div>
  );
}

async function SlowSection() {
  const data = await fetchSlowData();
  return <div>{data}</div>;
}
```

**4. Preloading Pattern**

```typescript
import { cache } from 'react';

const getUser = cache(async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

async function Page({ params }: { params: { id: string } }) {
  getUser(params.id); // Preload, don't await
  const otherData = await fetchOther();
  const user = await getUser(params.id); // Already cached
  return <div>{user.name}</div>;
}
```

**Server vs. Client Fetching:**

| Aspect | Server Component | Client Component |
|--------|------------------|------------------|
| `useEffect` | ❌ No | ✅ Yes |
| Direct DB access | ✅ Yes | ❌ No |
| Caching | Automatic | Manual (React Query) |
| Bundle size | No JS shipped | JS shipped |
| SEO | Excellent | Requires SSR |
| Real-time | ❌ No | ✅ Yes (WebSocket) |
| User interaction | ❌ No | ✅ Yes |

**When to Use Each:**

```typescript
// Server: Initial data, SEO content, static data
async function BlogPost({ id }: { id: string }) {
  const post = await fetchPost(id);
  return <article>{post.content}</article>;
}

// Client: User interactions, real-time updates, form handling
'use client';
function LiveChat() {
  const [messages, setMessages] = useState([]);
  useEffect(() => {
    const ws = new WebSocket('ws://...');
    ws.onmessage = (e) => setMessages(m => [...m, e.data]);
  }, []);
  return <div>{messages.map(...)}</div>;
}
```

**Hybrid Pattern:**

```typescript
// Server fetches initial data
async function ProductPage({ id }: { id: string }) {
  const product = await fetchProduct(id);
  return <InteractiveProductView product={product} />;
}

// Client component receives data as prop
'use client';
function InteractiveProductView({ product }: { product: Product }) {
  const [quantity, setQuantity] = useState(1);
  return (
    <div>
      <h1>{product.name}</h1>
      <input 
        type="number" 
        value={quantity}
        onChange={e => setQuantity(+e.target.value)}
      />
    </div>
  );
}
```

**Key Concepts:**
- Server Components fetch data on the server
- No `useEffect` or `useState` needed
- Use `Promise.all` for parallel fetching
- `Suspense` for streaming
- `cache()` for request-scoped deduplication
- Pass data to Client Components as props

**Common Mistakes:**
- Using `useEffect` in Server Components
- Not parallelizing independent fetches
- Fetching in Client Components when Server Components would work
- Not using `Suspense` for slow data
- Forgetting that Client Components need API routes

**Interview Tip:**
> "In Server Components, I fetch data directly with `async/await`—no `useEffect` needed. I parallelize independent fetches with `Promise.all`, use `Suspense` for streaming, and `cache()` for deduplication within a request. I only fetch in Client Components when I need real-time updates or user interactions."

</details>

---

## Question 25: React 19 Concurrent Features

**Difficulty:** Advanced  
**Category:** Strength Validation

Explain `useTransition()` and `useDeferredValue()`. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definitions:**

Both hooks let you mark updates as **non-urgent** so they don't block urgent updates (like user input).

**`useTransition()` - Wrap State Updates**

```typescript
import { useTransition, useState } from 'react';

function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    // Urgent: update input immediately
    setQuery(e.target.value);
    
    // Non-urgent: can be interrupted
    startTransition(() => {
      const filtered = expensiveFilter(e.target.value);
      setResults(filtered);
    });
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Results data={results} />
    </div>
  );
}
```

**`useDeferredValue()` - Defer a Value**

```typescript
import { useDeferredValue, useState, useMemo } from 'react';

function SearchComponent() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  // Expensive computation uses deferred value
  const results = useMemo(
    () => expensiveFilter(deferredQuery),
    [deferredQuery]
  );

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <Results data={results} />
    </div>
  );
}
```

**Key Differences:**

| Aspect | `useTransition()` | `useDeferredValue()` |
|--------|-------------------|----------------------|
| What it wraps | State updates | A value |
| Granularity | Imperative (you choose what to defer) | Declarative (value is automatically deferred) |
| `isPending` state | ✅ Yes | ❌ No |
| Use case | Event handlers, async operations | Derived values, expensive computations |
| Control | Explicit | Automatic |

**When to Use Each:**

```typescript
// ✓ useTransition: When you control the update
function Component() {
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    startTransition(() => {
      setCount(c => c + 1);
      setExpensiveState(compute());
    });
  };
}

// ✓ useDeferredValue: When you want to defer a value
function Component({ query }: Props) {
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => search(deferredQuery), [deferredQuery]);
  return <List items={results} />;
}
```

**Real-World Example: Tab Switching**

```typescript
function Dashboard() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  function selectTab(nextTab: string) {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <div>
      <TabBar tab={tab} onSelect={selectTab} />
      {isPending && <div className="loading" />}
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        {tab === 'home' && <Home />}
        {tab === 'profile' && <Profile />}
        {tab === 'settings' && <Settings />}
      </div>
    </div>
  );
}
```

**How It Works Internally:**

1. **Priority Lanes:** React assigns priority to updates
   - `SyncLane` (user input) - highest
   - `TransitionLane` (startTransition) - lower
2. **Interruptible Rendering:** React can pause TransitionLane work for SyncLane
3. **Visual Feedback:** `isPending` lets you show loading states

**Performance Impact:**

```typescript
// ❌ Without useTransition: Input lags during heavy computation
function Component() {
  const [query, setQuery] = useState('');
  const results = expensiveFilter(query); // Blocks input
  return <div>{query} {results.length}</div>;
}

// ✓ With useTransition: Input stays responsive
function Component() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  function handleChange(e) {
    setQuery(e.target.value); // Urgent
    startTransition(() => {
      setResults(expensiveFilter(e.target.value)); // Non-urgent
    });
  }
}
```

**Key Concepts:**
- `useTransition` wraps state updates
- `useDeferredValue` defers a value
- Both use Concurrent React features
- `isPending` from `useTransition` shows transition state
- Don't wrap urgent updates
- Don't use for every state update (overhead)

**Common Mistakes:**
- Wrapping urgent updates (input changes)
- Using `useDeferredValue` for simple values (no benefit)
- Not using `isPending` for visual feedback
- Expecting instant results (it's a delay, not removal)
- Using in React 17 (concurrent features require React 18+)

**Interview Tip:**
> "I use `useTransition` when I control the state update in an event handler, especially for navigation or expensive computations. I use `useDeferredValue` when I receive a value from props and want to defer expensive derived computations. Both keep the UI responsive by marking updates as non-urgent. The `isPending` flag from `useTransition` is useful for loading states."

</details>

---

## Summary Checklist

- [x] Concurrent rendering internals
- [x] Server vs. Client Components
- [x] Advanced TypeScript types
- [x] JavaScript engine behavior
- [x] Custom hooks patterns
- [x] Next.js caching layers
- [x] Prototypal inheritance
- [x] Performance optimization
- [x] Mapped types
- [x] Closures and memory
- [x] State batching
- [x] Discriminated unions
- [x] Middleware
- [x] Suspense and Error Boundaries
- [x] `unknown` vs. `any`
- [x] React 19 features
- [x] Bundle splitting
- [x] Decorators
- [x] Hydration
- [x] Web Vitals
- [x] `use()` hook
- [x] `satisfies` operator
- [x] Parallel Routes
- [x] Server Component data fetching
- [x] Concurrent features

**Total: 25 questions** covering advanced React, Next.js, TypeScript, and JavaScript internals.