# Node.js Interview Notes — Mid-Level Backend (2–5 yrs)

> Target: mid-level Node.js backend interviews. Covers JavaScript language traps, the event loop and
> runtime internals, TypeScript, and production backend Node. Every section has **what they ask**,
> **the answer**, and **runnable examples**.
>
> **Node versions as of Aug 2026:** Node **26** is Current (released May 2026, becomes LTS Oct 2026),
> Node **24 "Krypton"** is Active LTS, Node **22 "Jod"** is Maintenance LTS. Code here runs on Node 22+;
> anything needing 24/26 is flagged. See §0 for the release-schedule change — it's a good talking point.

## How to use this doc

1. Read a section, then **close it and re-explain the "Interview answer" out loud.**
2. ⚠️ blocks are the classic traps — that's where mid-level interviews are decided.
3. 🎯 blocks are the exact phrasings that score well.
4. Node interviews are unusually **output-prediction heavy** ("what does this log, and in what order?").
   Every ordering example here has verified output.

---

# Table of Contents

**Part 0 — Release schedule (quick context)**

**Part 1 — JavaScript Core & Traps**
1. Types, coercion, `==` vs `===`
2. Scope, hoisting, TDZ, `var`/`let`/`const`
3. Closures
4. `this` — the five rules
5. Prototypes, classes, inheritance
6. Objects: destructuring, spread, optional chaining
7. Arrays: the methods that matter
8. Copying, equality, immutability
9. Errors in JavaScript

**Part 2 — Async & the Event Loop**
10. Callbacks → Promises → async/await
11. **The event loop: phases and libuv**
12. **Microtasks vs macrotasks, `process.nextTick`**
13. Promise combinators
14. Async patterns & pitfalls
15. EventEmitter
16. Concurrency control

**Part 3 — Node Runtime Deep Dive**
17. Architecture: V8, libuv, bindings
18. Modules: CommonJS vs ESM
19. **Streams & backpressure**
20. Buffers & binary data
21. `fs`, `path`, `process`, env
22. **worker_threads vs cluster vs child_process**
23. Memory, GC, and leaks
24. Performance & profiling

**Part 4 — TypeScript for Node**
25. Setup & `tsconfig.json`
26. The type system you actually need
27. Generics & utility types
28. Narrowing & type guards
29. TypeScript + Node patterns

**Part 5 — Backend / Production Node**
30. HTTP server & frameworks
31. Middleware & error handling
32. Validation
33. Databases & connection pools
34. Security
35. Testing
36. Graceful shutdown & health checks
37. Observability
38. Production war stories

**Part 6 — Question Bank**
39. Rapid-fire Q&A (100+)
40. Output-prediction drills
41. 15-minute cram sheet

---

# PART 0 — Release schedule (know this, it's current)

Node.js is changing how it ships, and mentioning it signals you follow the ecosystem.

**Old model:** two majors a year. Odd versions (21, 23, 25) were "Current" only and never became LTS;
even versions (20, 22, 24) became LTS. Confusing, and nobody used the odd ones.

**New model, starting with Node 27 (April 2027):**
- **One major per year**, every April.
- **Every release becomes LTS** — the odd/even distinction is gone.
- Version numbers align with the calendar year (27 in 2027, 28 in 2028).
- Lifecycle: **Alpha 6mo → Current 6mo → LTS 30mo → EOL** (36 months total support).

**Where things stand right now (Aug 2026):**

| Version | Codename | Status | Notes |
|---|---|---|---|
| **26** | — | **Current** | Released May 2026; enters LTS Oct 2026 |
| 25 | — | EOL | Odd-numbered, never went LTS |
| **24** | Krypton | **LTS** | Leaving Active LTS around now |
| **22** | Jod | **Maintenance LTS** | The conservative production choice |

🎯 **What to say:** "For production I target the Active LTS line and upgrade one major behind Current.
From Node 27 onward that's simpler — one release a year, every one becomes LTS, so the odd/even
rule goes away."

**Recent features worth name-dropping** (they signal you're current, not stuck on Node 14):
- **Built-in test runner** (`node:test`, stable since 20) — no Jest needed for many projects.
- **`fetch`, `FormData`, `Headers`, `Response`** global since 18 — `axios`/`node-fetch` often unnecessary.
- **`--watch`** flag — no more `nodemon`.
- **`--env-file=.env`** — no more `dotenv`.
- **`node:sqlite`** built in (22+).
- **Stable `require(esm)`** (22.12+/23+) — CommonJS can now `require()` a synchronous ES module.
- **Type stripping** for TypeScript (`--experimental-strip-types` in 22, on by default in 23+) —
  run `.ts` files directly, no build step for types-only TS.
- **`AsyncLocalStorage`** — request context without passing it everywhere.
- **`glob`/`globSync`** in `node:fs` (22+).

---

# PART 1 — JAVASCRIPT CORE & TRAPS

## 1. Types, coercion, `==` vs `===`

### The seven primitives + object
`string`, `number`, `bigint`, `boolean`, `undefined`, `symbol`, `null` — everything else is an `object`
(including arrays, functions, dates, regexes).

```js
typeof "a"          // 'string'
typeof 1            // 'number'
typeof 1n           // 'bigint'
typeof true         // 'boolean'
typeof undefined    // 'undefined'
typeof Symbol()     // 'symbol'
typeof {}           // 'object'
typeof []           // 'object'    ← not 'array'
typeof function(){} // 'function'  ← the one non-primitive typeof
typeof null         // 'object'    ← THE famous bug, kept for backwards compatibility
```
⚠️ **`typeof null === 'object'`** is a 1995 bug that can never be fixed without breaking the web.
To test for null: `x === null`. To check arrays: `Array.isArray(x)`.

### `undefined` vs `null`
- `undefined` — "this was never assigned." The engine produces it (missing variable, missing
  property, missing argument, function with no return).
- `null` — "intentionally empty." **You** produce it.

```js
let a;                    // undefined
const obj = {};
obj.missing;              // undefined
function f(x) { return x; }
f();                      // undefined
JSON.stringify({a: undefined, b: null});   // '{"b":null}'  ← undefined is DROPPED
```
🎯 In APIs this matters: `undefined` disappears from JSON, `null` survives. That's the difference
between "field not sent" and "field explicitly cleared" in a PATCH endpoint.

### Truthy / falsy — memorize the falsy list
**Exactly eight falsy values:** `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`.
**Everything else is truthy** — including `[]`, `{}`, `"0"`, `"false"`, and `function(){}`.

```js
if ([])  console.log("empty array is truthy");   // runs!
if ({})  console.log("empty object is truthy");  // runs!
Boolean("0")     // true
Boolean([])      // true
[] == false      // true   ← but see coercion below 😱
```

### `==` vs `===`
- `===` — strict. No coercion. Types must match.
- `==` — loose. Coerces, following a genuinely baroque algorithm.

```js
1 == "1"          // true
0 == false        // true
0 == ""           // true
null == undefined // true   ← the ONE useful == case
null == 0         // false  ← null only loosely equals undefined
NaN == NaN        // false  ← NaN equals nothing, including itself
[] == false       // true   ([] -> "" -> 0, false -> 0)
[] == ![]         // true   😱  (![] is false, so [] == false)
"" == 0           // true
"0" == 0          // true
"" == "0"         // false  ← == is NOT transitive
```
🎯 **Interview answer:** "Always `===`. The single exception is `x == null`, which is a concise way
to check for null-or-undefined in one comparison — and even that I'd usually write explicitly."

### `Object.is` and NaN
```js
Object.is(NaN, NaN)   // true   ← the only reliable NaN equality
Object.is(0, -0)      // false  ← distinguishes signed zero
NaN === NaN           // false
Number.isNaN(NaN)     // true   ← use this, NOT the global isNaN
isNaN("foo")          // true   😱 global isNaN coerces first
Number.isNaN("foo")   // false  ← correct: "foo" is not the NaN value
```

### Number gotchas
```js
0.1 + 0.2 === 0.3          // false — IEEE 754 binary floating point
0.1 + 0.2                  // 0.30000000000000004
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON   // true — the correct comparison

Number.MAX_SAFE_INTEGER    // 9007199254740991 (2^53 - 1)
9007199254740992 === 9007199254740993   // true 😱 — beyond safe integer range
BigInt(9007199254740993)   // 9007199254740993n — use BigInt for large IDs

parseInt("08")             // 8
parseInt("1.9")            // 1     (truncates)
parseInt("12px")           // 12    (stops at first non-digit)
Number("12px")             // NaN   (stricter)
parseFloat("1.5e3")        // 1500
+"42"                      // 42    (unary plus — fast coercion)

(1234.5678).toFixed(2)     // '1234.57'  ← returns a STRING
```
🎯 **Money in Node:** never use floats. Store **integer cents** (or minor units), or use
`decimal.js` / `BigInt`. This comes up constantly in fintech interviews.

### String vs number coercion in `+`
```js
1 + 2      // 3
"1" + 2    // '12'   ← + with any string means CONCATENATION
1 + "2"    // '12'
1 - "2"    // -1     ← - has no string meaning, so it coerces to number
"5" * "2"  // 10
[] + {}    // '[object Object]'
[] + []    // ''
{} + []    // 0 in some REPL contexts (parsed as a block), '[object Object]' as an expression
```
🎯 If asked about `{} + []`: "It depends on parse position — as a statement, `{}` is an empty block
and `+[]` is unary plus on an empty array, giving `0`. As an expression it's string concatenation.
It's a parser trivia question, not something you'd ever write."

### Nullish coalescing vs `||`
```js
const port = process.env.PORT || 3000;    // 🐛 if PORT="" you silently get 3000
const port2 = process.env.PORT ?? 3000;   // ✅ only null/undefined fall through

0 || "default"      // 'default'   ← falsy
0 ?? "default"      // 0           ← not nullish
"" || "default"     // 'default'
"" ?? "default"     // ''
false ?? true       // false
```
⚠️ **This exact bug** — `||` swallowing a legitimate `0`, `""`, or `false` — is one of the most
common real-world Node bugs. Counts, retry limits, timeouts, and boolean flags are all legitimately
falsy.

⚠️ **The subtlety worth getting right:** `process.env` values are **always strings**, and `"0"` is
**truthy**. So `process.env.PORT || 3000` with `PORT="0"` gives `"0"`, not `3000`. The `||` bug bites
env vars only when the value is the **empty string**. It bites much harder *after* you coerce:
```js
const retries = Number(process.env.RETRIES) || 3;   // 🐛 RETRIES="0" → Number is 0 → falsy → 3
const retries2 = Number(process.env.RETRIES ?? 3);  // ✅ 0 survives
```
If an interviewer claims `PORT="0"` yields `3000` with `||`, they've skipped the string/number step.
Being precise here reads as very senior.

```js
// logical assignment operators
let a = null;  a ??= 5;    // 5   — assign if nullish
let b = 0;     b ||= 5;    // 5   — assign if falsy
let c = 1;     c &&= 5;    // 5   — assign if truthy
```

---

## 2. Scope, hoisting, TDZ, `var`/`let`/`const`

### The comparison table

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | **function** | block | block |
| Hoisted | yes, initialized to `undefined` | yes, but **TDZ** | yes, but **TDZ** |
| Redeclare in same scope | yes | no | no |
| Reassign | yes | yes | **no** |
| Creates global property (`globalThis`) | yes (in scripts) | no | no |

```js
console.log(x);   // undefined  ← var is hoisted and initialized
var x = 1;

console.log(y);   // ReferenceError: Cannot access 'y' before initialization
let y = 1;

console.log(z);   // ReferenceError: z is not defined
```
🧠 **TDZ (Temporal Dead Zone):** `let`/`const` ARE hoisted — the binding exists from the top of the
block — but accessing it before the declaration throws. This is deliberate: it turns a silent
`undefined` bug into a loud error.

🎯 **Interview answer:** "All three are hoisted. `var` is initialized to `undefined` at hoist time;
`let` and `const` stay uninitialized in the temporal dead zone until the declaration executes."

### `const` does NOT mean immutable
```js
const arr = [1, 2];
arr.push(3);        // ✅ fine — the BINDING is constant, not the value
// arr = [];        // ❌ TypeError: Assignment to constant variable

const obj = { a: 1 };
obj.a = 2;          // ✅ fine
Object.freeze(obj); // shallow freeze
obj.a = 3;          // silently ignored (throws in strict mode / ESM)
```
🎯 "`const` prevents rebinding the identifier. For actual immutability you need `Object.freeze`,
and even that is shallow — `structuredClone` + freeze recursively, or use a library."

### Function vs block scope — the classic loop question
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3 3 3  ← ONE `i`, function-scoped, already 3 by the time the callbacks run

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 0 1 2  ← `let` creates a NEW binding per iteration
```
🧠 This is *the* most-asked JavaScript interview question. The pre-ES6 fix was an IIFE:
```js
for (var i = 0; i < 3; i++) {
  (function (j) { setTimeout(() => console.log(j), 0); })(i);
}
```
🎯 Bonus: "`let` in a `for` loop gets a fresh binding each iteration, and the spec explicitly copies
the value forward — that's why closures capture distinct values."

### Function declarations vs expressions
```js
hoisted();            // ✅ works — declarations are fully hoisted
function hoisted() {}

// notHoisted();      // ❌ TypeError: notHoisted is not a function
var notHoisted = function () {};

// arrow();           // ❌ ReferenceError (TDZ)
const arrow = () => {};
```

### Strict mode
ESM and class bodies are **always** strict. CommonJS is sloppy unless you add `'use strict'`.
Strict mode: no implicit globals, `this` is `undefined` in plain functions, duplicate params are
errors, silent assignment failures throw.

---

## 3. Closures

🧠 **A closure is a function plus the lexical environment it was created in.** The inner function
keeps its outer scope alive even after the outer function returns.

```js
function counter() {
  let count = 0;                       // captured
  return {
    inc: () => ++count,
    get: () => count,
  };
}
const c = counter();
c.inc(); c.inc();
console.log(c.get());     // 2 — `count` is private, unreachable from outside
```

### What they're really testing
```js
function once(fn) {
  let called = false, result;
  return function (...args) {
    if (!called) { called = true; result = fn.apply(this, args); }
    return result;
  };
}

function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);      // ⚠️ naive; fails on functions/undefined/key order
    if (!cache.has(key)) cache.set(key, fn.apply(this, args));
    return cache.get(key);
  };
}

function debounce(fn, ms) {
  let t;
  return function (...args) {
    clearTimeout(t);
    t = setTimeout(() => fn.apply(this, args), ms);
  };
}

function throttle(fn, ms) {
  let last = 0;
  return function (...args) {
    const now = Date.now();
    if (now - last >= ms) { last = now; fn.apply(this, args); }
  };
}
```
🎯 **Debounce vs throttle** is a guaranteed question: "Debounce waits for a quiet period — it fires
once after the calls stop, good for search-as-you-type. Throttle fires at most once per interval
regardless — good for scroll handlers and rate limiting."

### Closures and memory leaks
```js
function leaky() {
  const bigBuffer = Buffer.alloc(50 * 1024 * 1024);   // 50MB
  return () => bigBuffer.length;    // closure pins the whole 50MB alive forever
}
const fn = leaky();   // 50MB retained as long as `fn` is reachable

function fixed() {
  const bigBuffer = Buffer.alloc(50 * 1024 * 1024);
  const len = bigBuffer.length;     // capture only what you need
  return () => len;
}
```
⚠️ This is a real Node leak pattern: an event listener or a cached callback closing over a large
request body, keeping it alive for the process lifetime. §23 covers finding it.

---

## 4. `this` — the five rules

🧠 In a regular function, **`this` is determined by HOW the function is called**, not where it's
defined. Resolve it in this order:

**1. `new` binding** — `this` is the new object.
```js
function User(name) { this.name = name; }
const u = new User("a");     // this === u
```

**2. Explicit binding** — `call` / `apply` / `bind`.
```js
function greet(greeting) { return `${greeting}, ${this.name}`; }
greet.call({name: "a"}, "hi");     // 'hi, a'   — args listed
greet.apply({name: "a"}, ["hi"]);  // 'hi, a'   — args as an Array
const bound = greet.bind({name: "a"});
bound("hi");                        // 'hi, a'  — returns a NEW permanently-bound function
```
🎯 Mnemonic: **C**all = **C**omma-separated, **A**pply = **A**rray, **B**ind = **B**ound later.

**3. Implicit binding** — the object left of the dot.
```js
const obj = { name: "a", greet() { return this.name; } };
obj.greet();      // 'a'
```

**4. Default binding** — no context: `undefined` in strict mode, `globalThis` in sloppy mode.

**5. Arrow functions** — **no own `this`.** They inherit it lexically from the enclosing scope and
`call`/`apply`/`bind` cannot change it.

### ⚠️ The lost-`this` trap (guaranteed question)
```js
const obj = {
  name: "a",
  greet() { return this.name; },
};
const fn = obj.greet;
fn();                      // undefined (or TypeError in strict mode) — context lost on extraction

setTimeout(obj.greet, 0);  // same problem
```
**Four fixes:**
```js
setTimeout(() => obj.greet(), 0);          // arrow wrapper — preferred
setTimeout(obj.greet.bind(obj), 0);        // bind
const fn2 = obj.greet; fn2.call(obj);      // explicit
// or define the method as an arrow-valued class field (below)
```

### Arrow functions in classes
```js
class Service {
  name = "svc";
  regular() { return this.name; }
  arrow = () => this.name;       // class field: `this` bound at construction, per instance
}
const s = new Service();
const { regular, arrow } = s;
// regular();     // ❌ TypeError — `this` is undefined
arrow();          // ✅ 'svc'
```
⚠️ Trade-off: an arrow class field is a **new function per instance** (more memory, not on the
prototype, harder to stub in tests). Use it for callbacks you'll pass around; use regular methods
otherwise.

⚠️ **Never use an arrow function as an object method that needs `this`:**
```js
const bad = { name: "a", greet: () => this.name };   // `this` is module scope, not `bad`
```
⚠️ **Never use an arrow as an EventEmitter/Mocha handler that relies on `this`** — arrows can't
receive the emitter as `this`.

---

## 5. Prototypes, classes, inheritance

🧠 **Every object has a hidden link (`[[Prototype]]`) to another object.** Property lookup walks
that chain until it finds the key or hits `null`. That's the entire inheritance model — classes are
syntax over it.

```js
const animal = { speak() { return "generic sound"; } };
const dog = Object.create(animal);
dog.bark = () => "woof";

dog.speak();                          // 'generic sound' — found on the prototype
Object.getPrototypeOf(dog) === animal; // true
dog.hasOwnProperty("speak");           // false
"speak" in dog;                        // true  ← `in` walks the chain, hasOwnProperty doesn't
Object.hasOwn(dog, "speak");           // false (modern replacement, ES2022)
```

### Classes
```js
class Animal {
  static count = 0;               // static field
  #secret = "hidden";             // TRUE private field (ES2022) — not just a convention
  constructor(name) {
    this.name = name;
    Animal.count++;
  }
  speak() { return `${this.name} makes a sound`; }
  get label() { return `<${this.name}>`; }          // getter
  set label(v) { this.name = v.replace(/[<>]/g, ""); }
  static create(name) { return new Animal(name); }  // static method
  reveal() { return this.#secret; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);                  // ⚠️ MUST call super() before touching `this`
    this.breed = breed;
  }
  speak() { return `${super.speak()} — a bark`; }   // super call
}

const d = new Dog("Rex", "Lab");
d.speak();                        // 'Rex makes a sound — a bark'
d instanceof Dog;                 // true
d instanceof Animal;              // true
// d.#secret                      // SyntaxError — genuinely inaccessible
```

🎯 **"Are JS classes real classes?"** → "They're syntactic sugar over prototypal inheritance —
`class` creates a constructor function whose `.prototype` object holds the methods, and `extends`
sets up the prototype chain. But they're not *purely* sugar: class bodies are always strict mode,
methods are non-enumerable, they can't be called without `new`, and `#private` fields have no
pre-class equivalent."

### `__proto__` vs `prototype`
```js
function F() {}
const f = new F();
f.__proto__ === F.prototype;              // true
Object.getPrototypeOf(f) === F.prototype; // true (preferred — __proto__ is legacy)
F.prototype.constructor === F;            // true
```
🎯 "`prototype` is a property **on constructor functions** — the object that instances will
delegate to. `__proto__` is the actual link **on an instance**. Only functions have `.prototype`;
every object has a `[[Prototype]]`."

⚠️ **Prototype pollution** — a real security vulnerability in Node:
```js
// a naive deep-merge on untrusted JSON
merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'));
// now ({}).isAdmin === true across the whole process 😱
```
Defenses: reject `__proto__`/`constructor`/`prototype` keys, use `Object.create(null)` for maps,
`Object.freeze(Object.prototype)`, or a vetted library (`lodash.merge` is patched; hand-rolled
merges usually aren't). Expect this in security-minded interviews.

---

## 6. Objects: destructuring, spread, optional chaining

```js
const user = { id: 1, name: "a", address: { city: "X" }, tags: ["p"] };

// destructuring with rename, default, and nesting
const { id, name: userName, missing = "fallback", address: { city } } = user;

// rest
const { id: _, ...withoutId } = user;

// function params — the idiomatic Node signature
function connect({ host = "localhost", port = 5432, ssl = false } = {}) {
  return `${host}:${port}`;
}
connect();                       // works — note the `= {}` default
connect({ port: 6543 });

// arrays
const [first, second = 0, ...rest] = [1];        // first=1, second=0, rest=[]
let a = 1, b = 2;
[a, b] = [b, a];                                  // swap

// spread / merge (SHALLOW)
const merged = { ...user, name: "b" };            // later wins
const copy = [...user.tags];

// optional chaining & nullish
user?.address?.city;              // 'X'
user?.missing?.deep;              // undefined — no TypeError
user.getName?.();                 // undefined if getName isn't a function
user.tags?.[0];                   // 'p'
const city = user?.address?.city ?? "unknown";
```
🔑 `?.` short-circuits the *whole* chain on the first nullish value.
⚠️ `?.` guards `null`/`undefined` only — it does **not** guard against a property being `0` or `""`.

### Useful `Object` statics
```js
Object.keys(user);           // own enumerable string keys
Object.values(user);
Object.entries(user);        // [[k, v], ...]
Object.fromEntries(pairs);   // inverse — great with .map/.filter over entries
Object.assign({}, a, b);     // shallow merge (mutates the target!)
Object.freeze(obj);          // shallow
Object.hasOwn(obj, "k");     // ES2022, replaces hasOwnProperty
structuredClone(obj);        // ✅ TRUE deep clone, built into Node 17+
```

### `Map`/`Set` vs object/array — know when
```js
const m = new Map();
m.set("a", 1).set({id: 1}, "object key");   // ANY key type, including objects
m.get("a"); m.has("a"); m.delete("a"); m.size;
for (const [k, v] of m) {}                   // insertion-ordered, directly iterable

const s = new Set([1, 2, 2, 3]);             // {1,2,3}
[...new Set(array)];                          // 🔑 the idiomatic dedupe

new WeakMap();  // object keys only, NOT enumerable, entries GC'd when the key dies
```
🎯 **When to use Map over a plain object:** non-string keys, frequent add/delete (Maps are optimized
for it), you need `.size`, you need guaranteed insertion order for all key types, or you want to
avoid prototype-pollution/key-collision risk from untrusted input.
🎯 **WeakMap** is the answer to "how do you attach metadata to objects without leaking memory?" —
per-object caches and private data keyed by instance.

---

## 7. Arrays: the methods that matter

```js
const xs = [1, 2, 3, 4, 5];

// non-mutating
xs.map(x => x * 2);                       // [2,4,6,8,10]
xs.filter(x => x % 2);                    // [1,3,5]
xs.reduce((acc, x) => acc + x, 0);        // 15
xs.find(x => x > 3);                      // 4      (value)
xs.findIndex(x => x > 3);                 // 3      (index)
xs.findLast(x => x < 4);                  // 3      (ES2023)
xs.some(x => x > 4);                      // true   (short-circuits)
xs.every(x => x > 0);                     // true   (short-circuits)
xs.includes(3);                           // true   (uses SameValueZero — finds NaN)
xs.slice(1, 3);                           // [2,3]  — copy
xs.flat(2); xs.flatMap(x => [x, x]);
xs.at(-1);                                // 5      (ES2022 — negative indexing)
xs.join("-"); xs.concat([6]);
xs.toSorted(); xs.toReversed(); xs.with(0, 9);   // ES2023 non-mutating versions ✅

// MUTATING (⚠️ these change the original)
xs.push(6); xs.pop(); xs.shift(); xs.unshift(0);
xs.splice(1, 2, "a");                     // remove 2 at index 1, insert "a"
xs.sort(); xs.reverse(); xs.fill(0);
```
⚠️ **`sort()` is lexicographic by default and mutates:**
```js
[10, 9, 1].sort();                     // [1, 10, 9]  😱 string comparison
[10, 9, 1].sort((a, b) => a - b);      // [1, 9, 10]  ✅
[...arr].sort((a, b) => a - b);        // ✅ copy first if you need the original
arr.toSorted((a, b) => a - b);         // ✅ ES2023, no copy needed
```

### `reduce` patterns worth knowing
```js
// group by
const byRole = users.reduce((acc, u) => {
  (acc[u.role] ??= []).push(u);
  return acc;
}, {});
// Node 21+ / ES2024 built-in:
const grouped = Object.groupBy(users, u => u.role);
const mapGrouped = Map.groupBy(users, u => u.role);

// index by id
const byId = new Map(users.map(u => [u.id, u]));

// sum / count / max
const total = items.reduce((s, i) => s + i.price, 0);
```
⚠️ `reduce` **without an initial value** throws on an empty array and uses element 0 as the seed.
Always pass the initial value.

### Creating & iterating
```js
Array.from({ length: 5 }, (_, i) => i);   // [0,1,2,3,4]
Array.from("abc");                         // ['a','b','c']
Array.from(new Set([1,1,2]));              // [1,2]
Array(3).fill(0);                          // [0,0,0]
// Array(3).map(...)                       // ⚠️ NO-OP — holes are skipped by map/forEach

for (const x of xs) {}          // values ✅ works with break/continue/await
for (const i in xs) {}          // ⚠️ KEYS as strings, includes inherited — avoid on arrays
xs.forEach(x => {});            // ⚠️ can't break, and IGNORES async callbacks (§14)
for (const [i, x] of xs.entries()) {}
```
🎯 **`for...in` vs `for...of`:** "`for...in` iterates enumerable string keys including the prototype
chain — it's for objects, and even then `Object.keys` is safer. `for...of` iterates values of any
iterable and is what you want for arrays, Maps, Sets, and async iteration."

---

## 8. Copying, equality, immutability

```js
const original = { a: 1, nested: { b: 2 }, list: [1, 2] };

// SHALLOW — nested references are shared
const s1 = { ...original };
const s2 = Object.assign({}, original);
s1.nested.b = 99;
original.nested.b;          // 99 😱

// DEEP
const deep = structuredClone(original);       // ✅ Node 17+, handles Map/Set/Date/RegExp/cycles
const jsonClone = JSON.parse(JSON.stringify(original));   // ⚠️ lossy
```
⚠️ **`JSON.parse(JSON.stringify(x))` silently destroys:** `undefined` values, functions, `Symbol`s,
`Date` → string, `Map`/`Set` → `{}`, `BigInt` → **throws**, `NaN`/`Infinity` → `null`, and it
**throws on circular references**. `structuredClone` handles all of these correctly (it can't clone
functions, but it throws loudly instead of silently dropping them).

```js
// deep equality — Node has this built in, no lodash needed
import assert from "node:assert";
import { isDeepStrictEqual } from "node:util";
isDeepStrictEqual({a:[1]}, {a:[1]});   // true
assert.deepStrictEqual({a:1}, {a:1});  // throws on mismatch
```

---

## 9. Errors in JavaScript

```js
class AppError extends Error {
  constructor(message, { status = 500, code = "INTERNAL", cause } = {}) {
    super(message, { cause });          // ⚠️ `cause` (ES2022) preserves the original
    this.name = this.constructor.name;  // otherwise it's just "Error"
    this.status = status;
    this.code = code;
    Error.captureStackTrace?.(this, this.constructor);   // V8: trim this frame from the stack
  }
}
class NotFoundError extends AppError {
  constructor(what) { super(`${what} not found`, { status: 404, code: "NOT_FOUND" }); }
}

try {
  throw new NotFoundError("user");
} catch (err) {
  if (err instanceof NotFoundError) console.log(err.status, err.code, err.name);
}
```
⚠️ **Setting `this.name = this.constructor.name` is not optional** — without it, subclassed errors
serialize and log as `"Error"`, which makes production triage miserable.

⚠️ **You can throw anything**, so never assume a caught value is an Error:
```js
try { throw "a string"; } catch (e) { e.message; }   // undefined
// defensive:
catch (e) { const err = e instanceof Error ? e : new Error(String(e)); }
```

```js
// optional catch binding (ES2019)
try { JSON.parse(s); } catch { return null; }

// AggregateError — from Promise.any and errors.Join-style grouping
const agg = new AggregateError([new Error("a"), new Error("b")], "all failed");
agg.errors.length;   // 2

// error.cause chain
const wrapped = new Error("could not load config", { cause: originalErr });
wrapped.cause;
```
🎯 **`finally` always runs** — including on `return` and on `throw`. ⚠️ A `return` inside `finally`
**overrides** the try/catch return value and swallows in-flight exceptions:
```js
function f() { try { return 1; } finally { return 2; } }   // 2 😱
```
---

# PART 2 — ASYNC & THE EVENT LOOP

> This is where Node interviews are won or lost. Expect 40%+ of the interview here, and expect at
> least one "what does this print, in what order?" question.

## 10. Callbacks → Promises → async/await

### Callback style and its problems
```js
fs.readFile("a.txt", (err, data) => {         // ⚠️ error-first callback convention
  if (err) return cb(err);                     // ALWAYS `return` — no automatic exit
  parse(data, (err2, parsed) => {
    if (err2) return cb(err2);
    save(parsed, (err3, saved) => {            // "callback hell" / pyramid of doom
      if (err3) return cb(err3);
      cb(null, saved);
    });
  });
});
```
Problems: no composability, error handling repeated at every level, **`try/catch` cannot catch an
async callback's throw**, and it's easy to call the callback twice or never.

### Promises
```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 100);
});
p.then(v => v.toUpperCase())
 .then(v => console.log(v))
 .catch(err => console.error(err))
 .finally(() => console.log("cleanup"));
```
**Three states, and the transition is one-way and permanent:** `pending` → `fulfilled` **or**
`rejected`. Calling `resolve` twice is a silent no-op.

🎯 **"Settled" vs "resolved":** settled = fulfilled or rejected. "Resolved" technically means the
promise's fate is locked in — which can mean it's been resolved *to another promise* and is still
pending. Nitpicky, but senior interviewers use it precisely.

```js
// promisifying a callback API
import { promisify } from "node:util";
const readFileAsync = promisify(fs.readFile);
// or just use the promise APIs Node already ships:
import fs from "node:fs/promises";
```

### async/await — syntax over promises
```js
async function load(id) {
  try {
    const raw = await fs.readFile(`${id}.json`, "utf8");
    const parsed = JSON.parse(raw);
    return await save(parsed);     // `await` here so a rejection is caught by THIS try/catch
  } catch (err) {
    throw new AppError("load failed", { cause: err });
  } finally {
    // always runs
  }
}
```
🧠 **Key facts:**
- An `async` function **always returns a promise**, whatever you return.
- `await` unwraps a promise (or a thenable, or a plain value wrapped via `Promise.resolve`).
- `throw` inside async → the returned promise rejects.
- **`await` only pauses the enclosing async function**, never the process or the event loop.

⚠️ **`return await` vs `return`:** inside a `try`, `return somePromise` **escapes the try/catch**
(you return the promise before it rejects). `return await somePromise` keeps the rejection inside.
Outside a try/catch they're equivalent (and `return await` costs one extra microtask tick).

---

## 11. The event loop: phases and libuv 🔑

🧠 **The one-sentence model:** Node is single-threaded for *your JavaScript*, but delegates I/O to
libuv, which uses the OS's async primitives (epoll/kqueue/IOCP) plus a small thread pool. The event
loop is what picks up completed work and runs your callbacks.

### The six phases, in order

```
   ┌───────────────────────────┐
┌─>│         timers            │  setTimeout / setInterval callbacks whose time has come
│  ├───────────────────────────┤
│  │    pending callbacks      │  deferred system-level callbacks (e.g. some TCP errors)
│  ├───────────────────────────┤
│  │    idle, prepare          │  internal only
│  ├───────────────────────────┤     ┌──────────────────┐
│  │         poll              │<────│  incoming I/O    │  ← retrieve I/O events; BLOCKS here
│  ├───────────────────────────┤     └──────────────────┘    when there's nothing else to do
│  │         check             │  setImmediate callbacks
│  ├───────────────────────────┤
│  │    close callbacks        │  socket.on('close'), etc.
└──┤───────────────────────────┘
```

**Between EVERY phase — and between every individual callback — Node drains the
`process.nextTick` queue and then the microtask (promise) queue completely.**

### What each phase actually does
- **timers** — runs callbacks scheduled by `setTimeout`/`setInterval`. The delay is a **minimum**,
  not a guarantee; a busy loop delays it.
- **poll** — the heart. Calculates how long to block, then waits for I/O. If there are `setImmediate`
  callbacks pending, it doesn't block; otherwise it blocks until the next timer is due or I/O arrives.
- **check** — `setImmediate` callbacks. The name exists to run things "immediately after poll."
- **close** — `'close'` event handlers.

### `setTimeout(fn, 0)` vs `setImmediate(fn)` — the classic question

```js
// At the TOP LEVEL of a module: NONDETERMINISTIC
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
```
Verified over 20 runs on Node 22 — the output genuinely alternates:
`I T T I I T I T I T T I I T I T I T I T ...`

🎯 **Why:** `setTimeout(fn, 0)` is clamped to 1ms. Whether that 1ms has elapsed by the time the
loop reaches the timers phase depends on how long process startup took — so it's a race.

```js
// INSIDE an I/O callback: setImmediate ALWAYS wins — guaranteed
fs.readFile(__filename, () => {
  setTimeout(() => console.log("  timeout inside io"), 0);
  setImmediate(() => console.log("  immediate inside io"));
});
// io callback
//   immediate inside io      ← always
//   timeout inside io
```
🎯 **Why it's guaranteed:** you're already *in* the poll phase. `check` comes immediately after
poll, so `setImmediate` runs on this same loop iteration. The timers phase is at the top of the
*next* iteration.

🎯 **The takeaway to say out loud:** "At the top level it's a race and you shouldn't depend on it.
Inside an I/O cycle, `setImmediate` always fires first because `check` follows `poll` in the same
tick. When I need to defer work until after the current poll phase, I use `setImmediate`; I never
use `setTimeout(fn, 0)` for ordering."

### libuv and the thread pool
- **Network I/O** (TCP, HTTP, DNS via `dns.lookup` is the exception) uses the OS event notification
  system — **no threads involved**, which is why Node handles tens of thousands of sockets cheaply.
- **The thread pool (default 4 threads, `UV_THREADPOOL_SIZE`, max 1024)** handles:
  **file system operations**, **DNS `lookup()`**, **`crypto.pbkdf2`/`scrypt`/`randomBytes`**, and
  **zlib compression**.

⚠️ **A classic production bug:** heavy `bcrypt`/`zlib`/`fs` work saturates all 4 pool threads and
every other pool-backed operation queues behind it — including DNS lookups, so *unrelated* outbound
HTTP calls start timing out.
```bash
UV_THREADPOOL_SIZE=16 node server.js      # rule of thumb: match your core count, measure
```
🎯 Note the asymmetry: "File I/O is thread-pool-backed; network I/O is not. That surprises people
who assume Node is 'async all the way down' — it's async, but file work costs a pool thread."

---

## 12. Microtasks vs macrotasks, `process.nextTick` 🔑🔑

### Two queues, both drained between everything
1. **`process.nextTick` queue** — Node-specific. Highest priority.
2. **Microtask queue** — promises, `queueMicrotask`, `await` continuations, `async` function resumptions.

Both are drained **completely** (including anything newly added while draining) before the loop
moves on. **Macrotasks** are the phase callbacks: timers, I/O, `setImmediate`.

### ⚠️ nextTick vs promises: the ordering DEPENDS ON MODULE SYSTEM

Most blog posts say "`nextTick` always beats promises." **That is only true inside callbacks.**
Verified on Node 22:

```js
// ---- t.cjs (CommonJS) ----
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("promise"));
queueMicrotask(() => console.log("queueMicrotask"));
```
```
nextTick
promise
queueMicrotask
```

```js
// ---- t.mjs (ESM) — SAME CODE ----
```
```
promise
queueMicrotask
nextTick          ← 😱 reversed!
```

```js
// ---- inside ANY callback (timer, I/O, immediate) — classic order restored ----
setTimeout(() => {
  process.nextTick(() => console.log("nextTick"));
  Promise.resolve().then(() => console.log("promise"));
}, 0);
```
```
nextTick
promise
```

🧠 **Why:** ES module evaluation is itself performed as a promise job. The top-level body of an
`.mjs` file runs *inside* a microtask, so the surrounding microtask drain finishes before Node gets
back to processing the nextTick queue. In CommonJS the module body runs synchronously, so the normal
"nextTick first" rule holds.

🎯 **How to answer this in an interview:** "Inside any callback, the nextTick queue drains before
the promise microtask queue. At the top level of an ES module it's inverted, because module
evaluation is itself a promise job. In practice I never write code that depends on the difference —
if ordering matters between two async operations, I make the dependency explicit."

That answer is better than the textbook one, and it's correct.

### The full ordering drill — verified output

```js
// order.mjs
console.log('1 sync start');
setTimeout(() => console.log('2 timeout 0'), 0);
setImmediate(() => console.log('3 immediate'));
process.nextTick(() => console.log('4 nextTick'));
Promise.resolve().then(() => console.log('5 promise'));
queueMicrotask(() => console.log('6 queueMicrotask'));
(async () => { console.log('7 async fn sync part'); await null; console.log('8 after await'); })();
console.log('9 sync end');
```
```
1 sync start
7 async fn sync part      ← an async function runs SYNCHRONOUSLY until its first await
9 sync end
5 promise
6 queueMicrotask
8 after await             ← the await continuation is just another microtask
4 nextTick                ← ESM ordering (in CJS this line moves to the top of this group)
3 immediate
2 timeout 0
```
🔑 **The two lessons that matter most here:** (a) an `async` function body runs synchronously until
the first `await` — it does not "start in the background"; (b) `await null` still costs a microtask
tick, so everything after it is deferred.

### Nested queue draining
```js
process.nextTick(() => { console.log('tick1'); process.nextTick(() => console.log('tick1.1')); });
Promise.resolve().then(() => {
  console.log('p1');
  process.nextTick(() => console.log('tick-in-p1'));
  Promise.resolve().then(() => console.log('p1.1'));
});
process.nextTick(() => console.log('tick2'));
Promise.resolve().then(() => console.log('p2'));
```
```
p1
p2
p1.1              ← newly-queued microtasks are drained in the SAME pass
tick1
tick2
tick-in-p1
tick1.1           ← and so are newly-queued ticks
```

⚠️ **Recursive `process.nextTick` starves the event loop completely** — the loop never advances to
the timers or poll phase, so your server stops responding while CPU sits at 100%:
```js
function starve() { process.nextTick(starve); }   // 🐛 total I/O starvation
function safe()   { setImmediate(safe); }          // ✅ yields between loop iterations
```
🎯 **When to actually use `process.nextTick`:** almost never in application code. Its legitimate use
is library-level — guaranteeing a callback is always async ("don't release Zalgo"), and letting a
caller attach event listeners before you emit:
```js
class Thing extends EventEmitter {
  constructor() {
    super();
    // this.emit('ready');                  // 🐛 nobody has subscribed yet
    process.nextTick(() => this.emit('ready'));   // ✅ caller gets a chance to .on()
  }
}
```

---

## 13. Promise combinators

| Method | Resolves when | Rejects when | Result |
|---|---|---|---|
| `Promise.all` | **all** fulfil | **first** rejection (fail-fast) | array of values |
| `Promise.allSettled` | **all** settle | **never** | array of `{status, value\|reason}` |
| `Promise.race` | **first** to settle (either way) | first settles as a rejection | that one value/reason |
| `Promise.any` | **first** fulfilment | **all** reject → `AggregateError` | that value |

```js
// all — fail fast; ⚠️ the other promises KEEP RUNNING, they are not cancelled
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);

// allSettled — the one you usually want for batch work
const results = await Promise.allSettled(ids.map(fetchUser));
const ok     = results.filter(r => r.status === "fulfilled").map(r => r.value);
const failed = results.filter(r => r.status === "rejected").map(r => r.reason);

// race — timeouts
const withTimeout = (p, ms) => Promise.race([
  p,
  new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), ms)),
]);
// ⚠️ leaks the timer until it fires; better:
async function timeout(p, ms) {
  const ac = new AbortController();
  const t = setTimeout(() => ac.abort(), ms);
  try { return await Promise.race([p, rejectOnAbort(ac.signal)]); }
  finally { clearTimeout(t); }
}
// ✅ or just use the built-in (Node 17+):
// await fetch(url, { signal: AbortSignal.timeout(5000) });

// any — first success wins (fallback endpoints)
const fastest = await Promise.any([fetchFromA(), fetchFromB()]);
```
🎯 **`Promise.all` vs `allSettled` is a guaranteed question.** "`all` is fail-fast — I use it when
every result is required and one failure makes the whole operation meaningless. `allSettled` never
rejects, so I use it for batch jobs where partial success is useful — indexing 10,000 documents,
one bad file shouldn't kill the run."

⚠️ **`Promise.all` does not cancel siblings.** Rejection just means *you* stop waiting; the other
requests still complete and can still throw unhandled rejections. Use `AbortController` for real
cancellation.

### AbortController — the standard cancellation primitive
```js
const ac = new AbortController();
setTimeout(() => ac.abort(new Error("too slow")), 5000);

const res = await fetch(url, { signal: ac.signal });
const data = await fs.readFile(path, { signal: ac.signal });

// combine signals (Node 20+)
const signal = AbortSignal.any([ac.signal, AbortSignal.timeout(3000)]);

// honour it in your own code
async function work(signal) {
  for (const item of items) {
    signal.throwIfAborted();          // throws AbortError
    await process(item);
  }
}
```
🔑 `signal` is accepted by `fetch`, `fs/promises`, `stream.pipeline`, `events.once`, `setTimeout`
(the timers/promises version), and most modern libraries. It's the Node equivalent of Go's `context`.

---

## 14. Async patterns & pitfalls

### ⚠️ Sequential when you meant parallel — the #1 perf bug
```js
// 🐛 SEQUENTIAL: 3 × 200ms = 600ms
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();

// ✅ PARALLEL: ~200ms
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// ✅ start early, await late (when you need `a` before starting `c`)
const pA = fetchA(), pB = fetchB();       // both start NOW
const a = await pA;
const b = await pB;
```
🧠 The moment you call an async function, the work **starts**. `await` only decides when you block
on the result.

### ⚠️ `await` inside `forEach` does nothing
```js
// 🐛 forEach ignores the returned promise — this logs "done" FIRST, and errors vanish
items.forEach(async (item) => { await save(item); });
console.log("done");

// ✅ sequential
for (const item of items) { await save(item); }

// ✅ parallel
await Promise.all(items.map(item => save(item)));

// ✅ bounded parallel (see §16)
```
Same trap applies to `map`/`filter`/`reduce` with async callbacks — `filter(async x => ...)` keeps
**everything**, because every promise is truthy.

### ⚠️ Unhandled rejections crash the process (Node 15+)
```js
// since Node 15 the default is --unhandled-rejections=throw
Promise.reject(new Error("boom"));    // process exits with a non-zero code

// safety nets — LOG AND EXIT, do not swallow
process.on("unhandledRejection", (reason) => {
  logger.fatal({ err: reason }, "unhandled rejection");
  process.exit(1);
});
process.on("uncaughtException", (err) => {
  logger.fatal({ err }, "uncaught exception");
  process.exit(1);
});
```
🎯 **The senior answer:** "These handlers are for logging and a clean exit, not for recovery. After
an uncaught exception the process is in an undefined state — you can't know which invariants were
broken. Log it, flush telemetry, exit, and let your supervisor (Kubernetes, systemd, pm2) restart
a clean process."

### ⚠️ `try/catch` cannot catch async callback throws
```js
try {
  setTimeout(() => { throw new Error("boom"); }, 0);   // 🐛 uncaught — different stack
} catch (e) { /* never runs */ }
```

### Async iteration
```js
for await (const chunk of readableStream) { process(chunk); }

async function* paginate(url) {
  let next = url;
  while (next) {
    const page = await fetch(next).then(r => r.json());
    yield* page.items;             // yield each item
    next = page.nextUrl;
  }
}
for await (const item of paginate("/api/items")) { console.log(item); }
```
🔑 Async generators are the clean way to model paginated APIs and streaming responses.

### Other traps
```js
// creating a promise around something already promise-based (the "explicit construction antipattern")
// 🐛
function bad() { return new Promise((res, rej) => { fetchIt().then(res).catch(rej); }); }
// ✅
function good() { return fetchIt(); }

// mixing callback + promise — double resolution
// forgetting to return in a .then chain — breaks the chain, loses errors
p.then(v => { doSomethingAsync(v); })   // 🐛 not returned → not awaited, errors escape
p.then(v => doSomethingAsync(v));       // ✅

// await in a loop over a huge array with no concurrency limit → see §16
```

---

## 15. EventEmitter

```js
import { EventEmitter, once } from "node:events";

class Job extends EventEmitter {
  async run() {
    this.emit("start", { at: Date.now() });
    try {
      const result = await work();
      this.emit("done", result);
    } catch (err) {
      this.emit("error", err);        // ⚠️ see below
    }
  }
}

const job = new Job();
job.on("done", (r) => console.log("done", r));      // every time
job.once("start", () => console.log("started"));    // first time only
job.off("done", handler);                            // remove (alias: removeListener)
job.listenerCount("done");
job.emit("done", 1);                                 // SYNCHRONOUS — listeners run inline
```

⚠️ **An `'error'` event with no listener throws and crashes the process.** This is unique to
`'error'` and it is deliberate.
```js
emitter.on("error", (err) => logger.error({ err }));   // always attach one
```

⚠️ **`emit` is synchronous.** Listeners run in registration order, on the caller's stack — a slow
listener blocks the emitter. A throwing listener propagates to the `emit()` call site.

⚠️ **`MaxListenersExceededWarning`** at 11 listeners on one event is a **leak detector**, not a
limit. If you see it, you're almost certainly adding listeners per request and never removing them.
```js
emitter.setMaxListeners(20);      // only if you genuinely need more
```

```js
// promise-friendly helpers
const [result] = await once(job, "done");                 // wait for one event
for await (const [msg] of on(emitter, "message")) {}      // async-iterate events
```
🎯 **EventEmitter vs Streams vs Observables:** "EventEmitter is push-based with no backpressure and
no completion signal. Streams add backpressure and lifecycle. If I need backpressure I use a stream,
not an emitter."

---

## 16. Concurrency control 🔑

Node's single thread means you're not limited by CPU when doing I/O — but you **must** bound
concurrency or you'll exhaust sockets, hit rate limits, and blow memory.

```js
// ⚠️ unbounded — 10,000 simultaneous requests
await Promise.all(urls.map(fetchOne));
```

### A dependency-free semaphore pool
```js
async function pool(items, limit, worker) {
  const results = new Array(items.length);
  let index = 0;
  const runners = Array.from({ length: Math.min(limit, items.length) }, async () => {
    while (index < items.length) {
      const i = index++;                       // safe: single-threaded, no await between read & write
      results[i] = await worker(items[i], i);
    }
  });
  await Promise.all(runners);
  return results;
}

const out = await pool(urls, 10, url => fetch(url).then(r => r.json()));
```
🎯 The `const i = index++` line is worth explaining: "There's no `await` between reading and
incrementing, so no other task can interleave. That's a real property of the single-threaded model —
you get atomicity for free as long as you don't yield mid-update."

### With error isolation
```js
async function poolSettled(items, limit, worker) {
  return pool(items, limit, async (item, i) => {
    try { return { status: "fulfilled", value: await worker(item, i) }; }
    catch (reason) { return { status: "rejected", reason }; }
  });
}
```

### Libraries to name-drop
`p-limit` / `p-map` / `p-queue` (Sindre Sorhus), `bottleneck` (rate limiting), `async` (legacy
callback-era). For real job queues: **BullMQ** (Redis) — the standard answer for "how do you run
background work in Node."

### Retry with exponential backoff + jitter
```js
async function retry(fn, { attempts = 5, base = 100, max = 10_000, signal } = {}) {
  let lastErr;
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (err) {
      lastErr = err;
      if (!isRetryable(err) || i === attempts - 1) break;
      const backoff = Math.min(max, base * 2 ** i);
      const delay = backoff / 2 + Math.random() * (backoff / 2);   // 🔑 jitter
      await setTimeoutPromise(delay, undefined, { signal });
    }
  }
  throw lastErr;
}
// import { setTimeout as setTimeoutPromise } from "node:timers/promises";
```
🔑 `node:timers/promises` gives you `setTimeout`, `setImmediate`, and `scheduler.wait` as
awaitable, abortable functions — no more `new Promise(r => setTimeout(r, ms))`.

🎯 **Retry only what's retryable:** 429, 5xx, network errors, timeouts — yes. 4xx client errors —
no. Respect `Retry-After`. Cap total attempts. Mention **circuit breakers** (`opossum`) for the
"how do you prevent a cascading failure" follow-up.
---

# PART 3 — NODE RUNTIME DEEP DIVE

## 17. Architecture: V8, libuv, bindings

```
┌──────────────────────────────────────────────┐
│              Your JavaScript                  │
├──────────────────────────────────────────────┤
│   Node core JS (fs, http, streams, ...)      │
├──────────────────────────────────────────────┤
│   Node C++ bindings  (node_file.cc, ...)     │
├──────────────────┬───────────────────────────┤
│       V8         │          libuv            │
│  JS engine:      │  event loop, thread pool, │
│  parse, JIT, GC  │  async I/O, TCP/UDP, fs   │
├──────────────────┴───────────────────────────┤
│  c-ares (DNS) · OpenSSL · zlib · llhttp      │
└──────────────────────────────────────────────┘
```

**V8** compiles and runs JS: Ignition (bytecode interpreter) → Sparkplug (baseline) → Maglev
(mid-tier, newer) → TurboFan (optimizing JIT). It also owns the heap and GC.
**libuv** is a C library providing the event loop, the thread pool, and a cross-platform async I/O
abstraction (epoll on Linux, kqueue on macOS/BSD, IOCP on Windows).

🎯 **"Is Node single-threaded?"** — the question they're really asking:
> "Your JavaScript runs on one thread, so there's one call stack and no shared-memory data races in
> user code. But the *process* is multi-threaded: libuv runs a 4-thread pool for file I/O, DNS
> lookup, crypto and zlib; V8 has its own GC and compiler threads; and I can add real parallelism
> with `worker_threads`. So: single-threaded event loop, multi-threaded runtime."

🎯 **"Why is Node good at I/O and bad at CPU work?"** — "Non-blocking I/O means one thread can
have thousands of sockets in flight, because waiting costs nothing. But any CPU-bound JavaScript
occupies the only thread that can run callbacks, so every other request stalls. CPU work goes to
`worker_threads`, a child process, or off the box entirely."

---

## 18. Modules: CommonJS vs ESM 🔑

### The comparison

| | CommonJS (`require`) | ESM (`import`) |
|---|---|---|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | **synchronous**, at runtime | **asynchronous**, statically analysed |
| Resolution | dynamic — can be in an `if` | static — hoisted to the top |
| Bindings | **copy of the value** at require time | **live binding** to the original |
| `this` at top level | `module.exports` | `undefined` |
| File extension | `.cjs`, or `.js` by default | `.mjs`, or `.js` with `"type":"module"` |
| `__dirname`/`__filename` | ✅ available | ❌ — use `import.meta.dirname` (20.11+) |
| Top-level `await` | ❌ | ✅ |
| Tree-shakeable | no | yes |
| Circular deps | partial exports (often `undefined`) | handled via hoisting + TDZ errors |

```js
// ---- CommonJS ----
const fs = require("node:fs");
const { join } = require("node:path");
module.exports = { a, b };
module.exports.c = c;
exports.d = d;                     // ⚠️ `exports = {...}` does NOT work — it breaks the alias

// ---- ESM ----
import fs from "node:fs";
import { join } from "node:path";
import * as path from "node:path";
export const a = 1;
export default class Service {}
export { b as renamed };
const mod = await import("./dynamic.js");     // dynamic import — works in BOTH systems
```

### ESM specifics that trip people up
```js
// package.json
{ "type": "module" }               // makes .js files ESM

// __dirname replacements
import.meta.dirname                 // Node 20.11+  ✅ simplest
import.meta.filename
import { fileURLToPath } from "node:url";
const __dirname = path.dirname(fileURLToPath(import.meta.url));   // older Node

// import a JSON file
import pkg from "./package.json" with { type: "json" };   // Node 20.10+ / 22 stable
```
⚠️ **ESM requires file extensions in relative imports.** `import "./util"` fails; use `"./util.js"`.
⚠️ **ESM has no `require`.** Build one with `createRequire` if you must:
```js
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
```
⚠️ **You cannot `import` a named export from a CommonJS module reliably** — CJS has no static
export list, so you usually get only the default: `import pkg from "cjs-lib"; const { x } = pkg;`

🔑 **`require(esm)` is now stable** (Node 22.12+ / 23+): CommonJS can synchronously `require()` an
ES module, as long as that module has no top-level `await`. This removed the single biggest pain
point of the dual-module era.

### Live bindings — the conceptual difference
```js
// counter.mjs
export let count = 0;
export function inc() { count++; }

// main.mjs
import { count, inc } from "./counter.mjs";
console.log(count);   // 0
inc();
console.log(count);   // 1  ← ESM binding is LIVE

// CommonJS equivalent would still print 0 — you got a copy of the value
```

### Module resolution & `package.json` exports
```json
{
  "name": "my-lib",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./utils": "./dist/utils.mjs"
  },
  "imports": { "#config": "./src/config.js" }
}
```
🔑 **`exports` is an encapsulation boundary** — consumers can only import the paths you list, and
deep imports into your internals stop working. `imports` (with the `#` prefix) gives you internal
path aliases with no build step.

🎯 **Module caching:** `require` caches by resolved path in `require.cache` — a module body runs
**once** per process. ESM caches in the module map, and there's no supported way to clear it (which
is why ESM test frameworks use worker isolation or loader hooks for mocking).

---

## 19. Streams & backpressure 🔑🔑

🧠 **Why streams exist:** `fs.readFile` on a 4 GB file loads 4 GB into memory. A stream processes it
in ~64 KB chunks with flat memory usage. Streams are Node's answer to "process more data than you
have RAM," and **backpressure** is the mechanism that keeps a fast producer from drowning a slow
consumer.

### Four types
| Type | Does | Example |
|---|---|---|
| **Readable** | produces data | `fs.createReadStream`, HTTP request (server) |
| **Writable** | consumes data | `fs.createWriteStream`, HTTP response |
| **Duplex** | both, independently | TCP socket |
| **Transform** | Duplex where output derives from input | `zlib.createGzip`, `crypto.createCipheriv` |

### The right way: `pipeline`
```js
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

await pipeline(
  createReadStream("big.log"),
  createGzip(),
  createWriteStream("big.log.gz"),
);
```
⚠️ **Never use `.pipe()` in production code.** `pipe` does not forward errors and does not clean up
the other streams when one fails — you leak file descriptors and get unhandled `'error'` events.
`pipeline` propagates errors and destroys every stream in the chain. This is a **guaranteed
interview question**; the one-line answer is *"pipe doesn't handle errors, pipeline does."*

### Backpressure, explained properly
```js
// 🐛 IGNORING BACKPRESSURE — unbounded memory growth
readable.on("data", (chunk) => {
  writable.write(chunk);          // ignores the return value
});

// ✅ HONOURING IT MANUALLY
readable.on("data", (chunk) => {
  if (!writable.write(chunk)) {   // false = internal buffer is above highWaterMark
    readable.pause();             // stop reading
    writable.once("drain", () => readable.resume());   // resume when flushed
  }
});

// ✅✅ or just let pipeline/for-await do it for you
for await (const chunk of readable) {
  if (!writable.write(chunk)) await once(writable, "drain");
}
```
🎯 **Say this:** "`writable.write()` returns `false` when the internal buffer exceeds
`highWaterMark` — 64 KB for byte streams, 16 objects for object mode. That's the signal to stop
producing until the `'drain'` event. `pipe` and `pipeline` implement this automatically; the classic
bug is a hand-rolled `'data'` handler that ignores the return value and grows the heap until OOM."

### Writing your own streams
```js
import { Readable, Writable, Transform } from "node:stream";

// Readable from any (async) iterable — by far the easiest way
const rs = Readable.from(asyncGenerator());

class Upper extends Transform {
  constructor() { super({ objectMode: false }); }
  _transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  }
  _flush(callback) { callback(); }        // emit any trailing state
}

class Collect extends Writable {
  constructor() { super({ objectMode: true }); this.items = []; }
  _write(chunk, encoding, callback) { this.items.push(chunk); callback(); }
}
```
⚠️ **Always call `callback()`** — exactly once, and with the error if there was one. Forgetting it
hangs the stream forever with no error; calling it twice throws.

### Object mode & interop
```js
new Transform({ objectMode: true, transform(obj, enc, cb) { cb(null, {...obj, seen: true}); } });

// Node streams ⟷ Web streams (Node 17+)
const webReadable = Readable.toWeb(nodeReadable);
const nodeReadable2 = Readable.fromWeb(webReadable);
```

### A realistic pipeline: NDJSON → transform → gzip → disk
```js
import { pipeline } from "node:stream/promises";
import { Transform } from "node:stream";
import readline from "node:readline";

const parse = new Transform({
  readableObjectMode: true,
  writableObjectMode: true,
  transform(line, _enc, cb) {
    try { cb(null, JSON.parse(line)); }
    catch { cb(); }                       // skip bad lines instead of killing the run
  },
});

await pipeline(
  readline.createInterface({ input: createReadStream("in.ndjson"), crlfDelay: Infinity }),
  parse,
  enrich,
  new Transform({ writableObjectMode: true, transform(o,_e,cb){ cb(null, JSON.stringify(o)+"\n"); } }),
  createGzip(),
  createWriteStream("out.ndjson.gz"),
);
```

🎯 **When NOT to use streams:** small payloads. The per-chunk overhead and complexity aren't worth
it below a few MB — `readFile`/`writeFile` are clearer and faster there.

---

## 20. Buffers & binary data

```js
Buffer.alloc(10);                       // ✅ zero-filled
Buffer.allocUnsafe(10);                 // ⚠️ FAST but contains old memory — must fully overwrite
Buffer.from("hello", "utf8");
Buffer.from([0x68, 0x69]);
Buffer.from("aGVsbG8=", "base64");
Buffer.concat([b1, b2]);

buf.toString("utf8");
buf.toString("hex");
buf.toString("base64url");              // URL-safe base64
buf.length;                             // BYTES, not characters
buf.slice(0, 4);                        // ⚠️ legacy: shares memory (like subarray)
buf.subarray(0, 4);                     // shares the underlying memory — no copy
Buffer.copyBytesFrom(view);             // explicit copy
```
⚠️ **`Buffer.allocUnsafe` is a real security footgun** — it hands you uninitialized memory that may
contain fragments of previous requests (passwords, tokens). Only use it when you overwrite every
byte immediately.

⚠️ **Multi-byte characters split across chunk boundaries** produce mojibake:
```js
// 🐛 a UTF-8 character can span two chunks
stream.on("data", chunk => result += chunk.toString("utf8"));

// ✅ use a StringDecoder, or set the stream encoding
import { StringDecoder } from "node:string_decoder";
const decoder = new StringDecoder("utf8");
stream.on("data", chunk => result += decoder.write(chunk));
stream.on("end", () => result += decoder.end());
// or: stream.setEncoding("utf8")
```

```js
// Buffers vs TypedArrays: Buffer IS a Uint8Array subclass
Buffer.from([1,2]) instanceof Uint8Array;   // true

// reading structured binary
buf.readUInt32BE(0); buf.readInt16LE(4); buf.readBigUInt64BE(0);
buf.writeUInt32BE(value, 0);
new DataView(buf.buffer, buf.byteOffset, buf.byteLength);
```
🔑 Buffers live **outside the V8 heap** (in external memory), so a buffer-heavy process shows a
small `heapUsed` but large RSS. Know this for §23 — it's why "my heap looks fine but the container
OOMs" happens.

---

## 21. `fs`, `path`, `process`, env

```js
import fs from "node:fs/promises";              // ✅ promise API — prefer this
import { createReadStream } from "node:fs";
import path from "node:path";

await fs.readFile("a.txt", "utf8");
await fs.writeFile("a.txt", data);
await fs.appendFile("log.txt", line);
await fs.mkdir("a/b", { recursive: true });
await fs.rm("dir", { recursive: true, force: true });
await fs.readdir(".", { withFileTypes: true, recursive: true });
await fs.stat(p); await fs.access(p, fs.constants.R_OK);
for await (const entry of await fs.opendir(".")) {}
const files = await Array.fromAsync(fs.glob("**/*.js"));   // Node 22+
```
⚠️ **Never use `fs.existsSync` + then open** — that's a TOCTOU race. Just try to open it and catch
`ENOENT`. ⚠️ **Never use sync fs calls in a request path** — they block the entire event loop.
Sync is fine at startup (loading config) and in CLI scripts.

```js
path.join("a", "b", "../c");     // 'a/c'         — normalizes
path.resolve("a", "b");           // absolute from cwd
path.basename(p, ".js"); path.extname(p); path.dirname(p);
path.sep;                          // '/' or '\\'
```
⚠️ **Path traversal:** never concatenate user input into a path.
```js
const requested = path.resolve(UPLOAD_DIR, path.basename(userInput));
if (!requested.startsWith(UPLOAD_DIR + path.sep)) throw new Error("invalid path");
```

```js
process.argv;              // [node, script, ...args]
process.env.NODE_ENV;
process.cwd();
process.exit(1);           // ⚠️ immediate — pending async writes may be lost
process.exitCode = 1;      // ✅ preferred: let the loop drain naturally
process.pid;
process.memoryUsage();     // { rss, heapTotal, heapUsed, external, arrayBuffers }
process.hrtime.bigint();   // monotonic nanoseconds — use for measuring durations
process.on("SIGTERM", shutdown);

import { parseArgs } from "node:util";     // built-in CLI arg parsing (18.3+)
const { values } = parseArgs({ options: { port: { type: "string", short: "p" } } });
```

```bash
node --env-file=.env server.js       # built-in .env loading (20.6+) — no dotenv
node --watch server.js               # built-in restart-on-change — no nodemon
node --test                          # built-in test runner — no Jest
```
🔑 Validate env at startup and **fail fast** — a missing `DATABASE_URL` should crash on boot, not
produce a 500 an hour into production. (§32 shows this with zod.)

---

## 22. worker_threads vs cluster vs child_process 🔑

| | `cluster` | `worker_threads` | `child_process` |
|---|---|---|---|
| Unit | separate **process** | **thread** in the same process | separate **process** |
| Memory | isolated (~40MB each) | **shared heap possible** via `SharedArrayBuffer` | isolated |
| Startup | ~50ms | **~5ms** | ~50ms |
| Use for | **scaling an HTTP server across cores** | **CPU-bound JS** | running other programs / scripts |
| Comms | IPC (JSON serialized) | `postMessage` (structured clone), `SharedArrayBuffer` | IPC / stdio |

### cluster — use all your cores for HTTP
```js
import cluster from "node:cluster";
import { availableParallelism } from "node:os";

if (cluster.isPrimary) {
  const n = availableParallelism();          // ✅ container-aware, unlike os.cpus().length
  for (let i = 0; i < n; i++) cluster.fork();
  cluster.on("exit", (worker, code, signal) => {
    logger.warn({ pid: worker.process.pid, code, signal }, "worker died, restarting");
    cluster.fork();
  });
} else {
  startServer();                              // the OS load-balances accept() across workers
}
```
🎯 **In containers, prefer horizontal scaling over `cluster`.** "If I'm on Kubernetes I run one
Node process per container and let the orchestrator scale replicas — it gives better isolation,
simpler observability, and rolling restarts. `cluster` is for a single VM where you own all the
cores. `pm2` in cluster mode is the classic non-container answer."
⚠️ **Workers share nothing.** In-memory caches, rate-limit counters, and sessions must move to Redis
the moment you fork.

### worker_threads — CPU-bound work
```js
// main.js
import { Worker } from "node:worker_threads";

function runTask(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./worker.js", { workerData: data });
    worker.on("message", resolve);
    worker.on("error", reject);
    worker.on("exit", (code) => { if (code !== 0) reject(new Error(`exit ${code}`)); });
  });
}

// worker.js
import { parentPort, workerData } from "node:worker_threads";
const result = expensiveComputation(workerData);   // blocks only THIS thread
parentPort.postMessage(result);
```
⚠️ **Spawning a worker per request is an antipattern** — ~5ms startup plus a fresh V8 isolate each
time. Use a **pool** (`piscina` is the standard library for this).
```js
import Piscina from "piscina";
const pool = new Piscina({ filename: "./worker.js", maxThreads: 4 });
const result = await pool.run(data);
```
🔑 Transfer large buffers instead of copying: `postMessage(buf, [buf.buffer])` transfers ownership
(zero-copy); `SharedArrayBuffer` + `Atomics` gives genuinely shared memory.

### child_process
```js
import { execFile, spawn } from "node:child_process";
const child = spawn("ffmpeg", ["-i", input, output]);   // streams stdout/stderr
child.stdout.pipe(process.stdout);
```
⚠️ **Never use `exec`/`execSync` with interpolated user input** — it runs through a shell, so
`"; rm -rf /"` is command injection. Use `execFile`/`spawn` with an **argument array** and no shell.

🎯 **The decision answer:** "CPU-bound JS → `worker_threads` with a pool. Scaling HTTP across cores
on one machine → `cluster`, or container replicas if I'm orchestrated. Running an external binary →
`spawn`. And before any of it, I'd check whether the CPU work belongs in Node at all."

---

## 23. Memory, GC, and leaks 🔑

### V8's generational GC
- **Young generation (new space, small, ~1–8MB)** — Scavenger, a semi-space copying collector.
  Most objects die here. Collections are frequent and **very fast (<1ms)**.
- **Old generation** — objects that survived two scavenges get promoted. Collected by
  **mark-sweep-compact**, mostly **concurrent and incremental** (Orinoco), with short STW pauses.
- **Orinoco** = the umbrella name for parallel/incremental/concurrent GC work in V8.

```bash
node --max-old-space-size=4096 server.js     # heap cap in MB (default ~2–4GB on 64-bit)
node --expose-gc -e "global.gc()"            # force GC (testing only)
node --trace-gc server.js                    # log every collection
```
🎯 **The container answer:** "Set `--max-old-space-size` to roughly 75–80% of the container memory
limit. Node doesn't read cgroup limits for the heap by default, so in a 512MB container it will
happily try to grow to 2GB and get OOM-killed with no JS error — just exit code 137."

### `process.memoryUsage()` — what each number means
```js
{
  rss: 50_000_000,          // Resident Set Size — TOTAL process memory (what the OOM killer sees)
  heapTotal: 20_000_000,    // V8 heap allocated
  heapUsed: 15_000_000,     // V8 heap in use  ← "is my JS leaking?"
  external: 8_000_000,      // C++ objects bound to JS (Buffers live here)
  arrayBuffers: 5_000_000,  // ArrayBuffer/Buffer allocations
}
```
🔑 **Rising `heapUsed` = a JavaScript leak. Rising `external`/`arrayBuffers` with flat `heapUsed` =
a Buffer leak.** Rising RSS with both flat = native module or fragmentation. Knowing which number
to look at is the difference between a 20-minute and a 2-day investigation.

### The five classic Node leaks
1. **Event listeners added per request, never removed** — watch for `MaxListenersExceededWarning`.
   ```js
   // 🐛
   app.get("/x", (req, res) => { emitter.on("tick", () => res.write("x")); });
   // ✅ emitter.off in a finally / res.on("close"), or use `once`
   ```
2. **An unbounded module-scope cache/Map** — `const cache = new Map()` with no TTL or size cap.
   Use `lru-cache`, or a `WeakMap` if keys are objects.
3. **Closures capturing large objects** (§3) — a callback holding a whole request body forever.
4. **Timers never cleared** — `setInterval` on a per-connection object keeps it alive.
   Always `clearInterval`; consider `.unref()` for background timers.
5. **Global arrays that only grow** — an in-memory log/metrics buffer with no flush.

### Finding a leak — the actual workflow
```bash
node --inspect server.js         # then chrome://inspect → Memory → Heap snapshot
```
1. Snapshot after warmup. 2. Run load. 3. Snapshot again. 4. **Compare** the two and sort by
   "Delta" / "objects allocated between snapshots." 5. Look at **Retainers** to see what's holding
   the objects alive — that path *is* your bug.
```js
import v8 from "node:v8";
v8.writeHeapSnapshot(`/tmp/heap-${Date.now()}.heapsnapshot`);   // works in production
```
```js
// cheap production canary
setInterval(() => {
  const { heapUsed, rss, external } = process.memoryUsage();
  logger.info({ heapUsed, rss, external }, "mem");
}, 60_000).unref();
```
🔑 `.unref()` tells Node "this timer shouldn't keep the process alive" — essential for background
timers, or your process refuses to exit.

⚠️ **Exit code 137** = SIGKILL, almost always the container OOM killer. There's no JS stack trace
because the process was killed from outside. **Exit code 134** = SIGABRT, typically V8's own
"JavaScript heap out of memory" abort — that one *does* print a stack.

---

## 24. Performance & profiling

### CPU profiling
```bash
node --cpu-prof --cpu-prof-dir=./prof server.js    # writes a .cpuprofile, open in Chrome DevTools
node --prof server.js && node --prof-process isolate-*.log > processed.txt
node --inspect server.js                            # live profiling via chrome://inspect
npx clinic doctor -- node server.js                 # clinic.js: doctor / flame / bubbleprof
npx 0x server.js                                    # flamegraph
```
🔑 **Read a flamegraph**: width = time spent, not call count. Wide plateaus at the top are where
the CPU actually is. Look for anything unexpectedly wide — usually JSON serialization, regex,
crypto, or a synchronous fs call.

### Event loop lag — the single most important Node metric
```js
import { monitorEventLoopDelay } from "node:perf_hooks";
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  logger.info({ p50: h.mean / 1e6, p99: h.percentile(99) / 1e6 }, "loop delay ms");
  h.reset();
}, 10_000).unref();
```
🎯 **"How do you know your event loop is blocked?"** → "I monitor event loop delay with
`perf_hooks.monitorEventLoopDelay` and alert on p99. Healthy is single-digit milliseconds. If p99
climbs, something synchronous is running too long — usually `JSON.parse` on a huge payload, a
sync fs call, a catastrophic regex, or crypto on the main thread."

### Measuring
```js
import { performance, PerformanceObserver } from "node:perf_hooks";
performance.mark("start");
await work();
performance.mark("end");
performance.measure("work", "start", "end");

// benchmarking
import { Bench } from "tinybench";        // or `node --test` + manual timing, or mitata
```

### Common Node performance wins
1. **Don't block the loop.** Move CPU work to a worker; chunk long loops with `setImmediate`.
2. **Stream large payloads** instead of buffering.
3. **Reuse connections** — a keep-alive HTTP agent, and a database pool (§33).
4. **Faster JSON** — for very hot paths, `fast-json-stringify` (schema-based, ~2× faster than
   `JSON.stringify`); Fastify uses it by default.
5. **Avoid `JSON.parse` on huge bodies** — cap request size, or stream-parse.
6. **Cache** — in-memory LRU for hot reads, Redis for shared state.
7. **Watch out for catastrophic regex backtracking (ReDoS)** — `/(a+)+$/` against a long string
   hangs the whole server. Validate lengths first, prefer non-backtracking patterns, and consider
   `re2`.
8. **Use `Buffer` end to end** rather than converting to/from strings repeatedly.
9. **`Object.freeze` doesn't speed things up**; **monomorphic shapes do** — don't add properties to
   objects after creation in hot paths.
10. **Measure first.** `--cpu-prof` before you optimize anything.
---

# PART 4 — TYPESCRIPT FOR NODE

> TypeScript is the default for new Node backends in 2026. Expect to be asked about it even for a
> "Node" role. This part covers what you need to be productive and to answer confidently — not the
> type-level gymnastics.

## 25. Setup & `tsconfig.json`

### `tsconfig.json` for a Node service
```jsonc
{
  "compilerOptions": {
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "NodeNext",              // ✅ correct dual CJS/ESM resolution for Node
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",

    "strict": true,                    // 🔑 turn this on. It's the whole point of TypeScript.
    "noUncheckedIndexedAccess": true,  // arr[i] is T | undefined — catches real bugs
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,      // `import type` is erased predictably

    "esModuleInterop": true,
    "skipLibCheck": true,              // skip type-checking node_modules — big speedup
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,

    "declaration": true,
    "sourceMap": true,
    "isolatedModules": true            // required if you use esbuild/swc for transpilation
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```
```bash
npm i -D typescript @types/node tsx
npx tsc --noEmit          # type-check only (this is what runs in CI)
npx tsx src/server.ts     # run TS directly, fast (esbuild under the hood)
npx tsx watch src/server.ts
```

🔑 **`strict: true` is the answer to "how do you configure TypeScript?"** It enables
`strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, and more. Without `strictNullChecks`,
TypeScript lets `null` into every type and you get most of the annoyance with little of the benefit.

### Running TypeScript in Node — the four options
| Approach | Notes |
|---|---|
| `tsc` → run `dist/` | The safe default for production. Full type checking at build time. |
| **`tsx`** / `ts-node` | Run `.ts` directly in dev. `tsx` is much faster (esbuild). No type checking. |
| **Node native type stripping** | `--experimental-strip-types` (22), **on by default in 23+**. Erases types, no transpilation. Doesn't support enums/namespaces/decorators (use `--experimental-transform-types` for those). Zero dependencies. |
| Bundle (esbuild/tsup/swc) | For libraries, or when you want a single-file output. |

🎯 **Say this:** "In dev I run `tsx watch`, or Node's native type stripping now that it's on by
default. In CI I run `tsc --noEmit` for real type checking, and build with `tsc` or `tsup` for
production. The key point is that transpilers like tsx and esbuild **don't type-check** — you still
need `tsc --noEmit` in the pipeline or types become decoration."

---

## 26. The type system you actually need

```ts
// primitives & inference
let n: number = 1;
let s = "hello";                  // inferred as string — don't annotate what's obvious
const t = "hello";                // inferred as the LITERAL type "hello"

// arrays, tuples
const xs: string[] = [];
const pair: [string, number] = ["a", 1];
const named: [id: string, count: number] = ["a", 1];

// objects
interface User {
  readonly id: string;
  name: string;
  email?: string;                 // optional → string | undefined
  [key: string]: unknown;         // index signature
}

// type alias vs interface
type Point = { x: number; y: number };
interface Shape { area(): number }
```
🎯 **`interface` vs `type`:** "Interfaces can be declaration-merged and are conventional for object
shapes and public API contracts. Type aliases can express unions, intersections, tuples, mapped and
conditional types. I default to `interface` for object shapes and `type` for everything else — but
they're interchangeable for plain objects."

### Unions, intersections, literals
```ts
type Status = "pending" | "active" | "closed";       // string literal union — use these constantly
type Id = string | number;
type Admin = User & { role: "admin"; permissions: string[] };

// discriminated union — THE most useful TS pattern for backend code
type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: Error };

function handle(r: Result<number>) {
  if (r.ok) return r.value;        // TS narrows to the success branch
  return r.error.message;          // and to the failure branch here
}
```
🔑 A discriminated union with a literal tag is how you model "success or failure," "which event
type," and "which payment method" — and it makes exhaustiveness checking work (§28).

### `any` vs `unknown` vs `never`
```ts
let a: any;        // ⚠️ disables ALL checking, infects everything it touches
let u: unknown;    // ✅ safe top type — you MUST narrow before using
let n: never;      // bottom type — no value is assignable

function assertNever(x: never): never { throw new Error(`unexpected: ${JSON.stringify(x)}`); }

// unknown forces you to check
function parse(raw: unknown) {
  // raw.foo                     // ❌ Object is of type 'unknown'
  if (typeof raw === "object" && raw !== null && "foo" in raw) return raw.foo;   // ✅
}
```
🎯 **"any vs unknown"** is asked in almost every TS screen. "`any` opts out of the type system —
anything is assignable to it and it's assignable to anything, so errors propagate silently.
`unknown` is the type-safe counterpart: everything is assignable *to* it, but you can't do anything
*with* it until you narrow. `catch (e: unknown)` and JSON parsing are where I use it."

### Functions
```ts
function f(a: string, b = 0, ...rest: number[]): void {}
type Handler = (req: Request, res: Response) => Promise<void>;

// overloads
function get(key: string): string;
function get(key: string, fallback: string): string;
function get(key: string, fallback?: string): string { return fallback ?? ""; }

// `this` typing
function handler(this: Emitter, ev: Event): void {}
```

### `satisfies` (TS 4.9+) — underused and worth knowing
```ts
const config = {
  port: 3000,
  host: "localhost",
} satisfies Record<string, string | number>;

config.port.toFixed();     // ✅ still known to be `number`, not widened to string|number
```
🎯 "`satisfies` validates a value against a type **without widening it**. `const x: T = ...` checks
and widens; `satisfies` checks and keeps the literal inference."

---

## 27. Generics & utility types

```ts
function first<T>(items: T[]): T | undefined { return items[0]; }

// constraints
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }
pluck({ a: 1, b: "x" }, "a");        // inferred number

// generic classes / interfaces
interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  save(entity: T): Promise<T>;
  findAll(filter?: Partial<T>): Promise<T[]>;
}

class InMemoryRepo<T extends { id: string }> implements Repository<T> {
  #items = new Map<string, T>();
  async findById(id: string) { return this.#items.get(id) ?? null; }
  async save(e: T) { this.#items.set(e.id, e); return e; }
  async findAll() { return [...this.#items.values()]; }
}

// default type params + conditional
type Unwrap<T> = T extends Promise<infer U> ? U : T;
type A = Unwrap<Promise<string>>;    // string
```

### Utility types — know these cold
```ts
interface User { id: string; name: string; email: string; age: number }

Partial<User>                        // all optional  → PATCH request bodies
Required<User>                       // all required
Readonly<User>                       // all readonly
Pick<User, "id" | "name">            // subset        → API response shapes
Omit<User, "email">                  // everything except
Record<string, User>                 // dictionary
Exclude<"a" | "b" | "c", "a">        // "b" | "c"
Extract<"a" | 1, string>             // "a"
NonNullable<string | null>           // string
ReturnType<typeof fn>                // the return type of a function
Parameters<typeof fn>                // its params as a tuple
Awaited<ReturnType<typeof asyncFn>>  // 🔑 unwrap a Promise return type
```

```ts
// real-world combinations you'll actually write
type CreateUserDto = Omit<User, "id">;
type UpdateUserDto = Partial<Omit<User, "id">>;
type PublicUser    = Omit<User, "email">;                        // strip PII from responses
type UserResponse  = Readonly<Pick<User, "id" | "name">>;

// mapped types
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
// -> { getId: () => string; getName: () => string; ... }
```

### `keyof`, `typeof`, `as const`
```ts
const ROLES = ["admin", "user", "guest"] as const;    // readonly ["admin","user","guest"]
type Role = typeof ROLES[number];                      // "admin" | "user" | "guest"  🔑

const CONFIG = { port: 3000, env: "dev" } as const;
type ConfigKey = keyof typeof CONFIG;                  // "port" | "env"
```
🔑 The `as const` + `typeof X[number]` combination turns a runtime array into a compile-time union —
one source of truth for both validation and types. You'll use it constantly.

---

## 28. Narrowing & type guards

```ts
function process(v: string | number | string[] | null) {
  if (v === null) return "null";
  if (typeof v === "string") return v.toUpperCase();      // typeof narrowing
  if (Array.isArray(v)) return v.join(",");                // Array.isArray narrows
  return v.toFixed(2);                                     // must be number
}

// instanceof
if (err instanceof AppError) err.status;

// `in` narrowing
type Cat = { meow(): void }; type Dog = { bark(): void };
function speak(a: Cat | Dog) { return "meow" in a ? a.meow() : a.bark(); }

// discriminated union + exhaustiveness ✅ the pattern to demonstrate
type Event =
  | { type: "created"; id: string }
  | { type: "updated"; id: string; changes: Record<string, unknown> }
  | { type: "deleted"; id: string };

function reduce(e: Event): string {
  switch (e.type) {
    case "created": return `+${e.id}`;
    case "updated": return `~${e.id} ${Object.keys(e.changes).length}`;
    case "deleted": return `-${e.id}`;
    default: return assertNever(e);      // 🔑 compile error if a case is ever added and missed
  }
}
```
🎯 **`assertNever` is worth showing unprompted.** "Adding a fourth event type turns the missing case
into a *compile* error rather than a runtime bug. That's the payoff of discriminated unions."

### Custom type guards & assertions
```ts
function isUser(v: unknown): v is User {                 // type predicate
  return typeof v === "object" && v !== null && "id" in v && typeof (v as User).id === "string";
}

function assertIsDefined<T>(v: T): asserts v is NonNullable<T> {   // assertion function
  if (v == null) throw new Error("expected a value");
}

const maybe = users.find(u => u.id === id);
assertIsDefined(maybe);
maybe.name;                                               // narrowed to User
```

⚠️ **`as` is a lie to the compiler, not a runtime check.**
```ts
const user = JSON.parse(body) as User;    // 🐛 zero validation — body could be anything
```
🎯 **This is the most important thing to say about TypeScript in a Node interview:**
> "TypeScript is erased at runtime. It gives me zero guarantees about data crossing a boundary —
> HTTP bodies, env vars, database rows, third-party APIs. At every boundary I validate with a
> runtime schema library like zod and *derive* the type from the schema, so there's one source of
> truth and the type can't drift from the check."

That's §32.

### Typing `catch`
```ts
try { risky(); }
catch (err) {                       // `unknown` under `useUnknownInCatchVariables` (part of strict)
  if (err instanceof AppError) return err.status;
  const e = err instanceof Error ? err : new Error(String(err));
  logger.error({ err: e });
}
```

---

## 29. TypeScript + Node patterns

### Typing Express
```ts
import type { Request, Response, NextFunction, RequestHandler } from "express";

interface AuthedRequest extends Request {
  user?: { id: string; role: Role };
}

// generic params: <Params, ResBody, ReqBody, Query>
const createUser: RequestHandler<{}, UserResponse, CreateUserDto> = async (req, res) => {
  const dto = req.body;             // typed as CreateUserDto
  res.json(toPublic(await service.create(dto)));
};

// augment the global type instead of casting everywhere
declare global {
  namespace Express {
    interface Request { user?: { id: string; role: Role }; requestId: string }
  }
}
```
🎯 Fastify does this better out of the box — it infers request/response types from a schema, so you
get validation and types from one declaration. Worth mentioning as a reason to prefer it.

### Typed environment config
```ts
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export type Env = z.infer<typeof EnvSchema>;      // 🔑 type DERIVED from the schema

export const env: Env = (() => {
  const parsed = EnvSchema.safeParse(process.env);
  if (!parsed.success) {
    console.error("invalid environment:", z.treeifyError(parsed.error));
    process.exit(1);                               // 🔑 fail fast at boot
  }
  return parsed.data;
})();
```

### Branded types — cheap, high value
```ts
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId  = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

const asUserId = (s: string): UserId => s as UserId;

function getUser(id: UserId) {}
// getUser(orderId);       // ❌ compile error — can't pass an OrderId where a UserId is expected
```
🔑 This eliminates an entire class of "passed the wrong ID" bugs with no runtime cost. Great thing
to volunteer in a design discussion.

### Typing `node:test`
```ts
import test from "node:test";
import assert from "node:assert/strict";

test("adds", () => { assert.equal(1 + 1, 2); });
```

### Declaration files for untyped packages
```ts
// src/types/some-lib.d.ts
declare module "some-untyped-lib" {
  export function doThing(x: string): Promise<number>;
}
```

### Common TS mistakes in Node code
| Mistake | Fix |
|---|---|
| `as any` to silence an error | narrow properly, or `unknown` + a guard |
| `as User` on parsed JSON | validate with zod, derive the type |
| Not enabling `strict` | enable it; migrate file by file if needed |
| `enum` | prefer `as const` unions — enums emit runtime code and don't work with type stripping |
| Types duplicated from a schema | derive with `z.infer` |
| Relying on `tsx`/esbuild to catch errors | they don't type-check; run `tsc --noEmit` in CI |
| `@ts-ignore` | `@ts-expect-error` — it errors if the problem is ever fixed |
---

# PART 5 — BACKEND / PRODUCTION NODE

## 30. HTTP server & frameworks

### Raw `node:http` — know it, they sometimes ask
```js
import http from "node:http";

const server = http.createServer(async (req, res) => {
  if (req.method === "GET" && req.url === "/health") {
    res.writeHead(200, { "content-type": "application/json" });
    return res.end(JSON.stringify({ status: "ok" }));
  }
  res.writeHead(404).end();
});

server.headersTimeout   = 10_000;    // time to receive headers (Slowloris defense)
server.requestTimeout   = 30_000;    // whole request
server.keepAliveTimeout = 65_000;    // ⚠️ must EXCEED your load balancer's idle timeout
server.listen(3000);
```
⚠️ **The keep-alive race:** if your ALB idles at 60s and Node closes at 5s (the old default), the LB
sends a request down a socket Node is closing → intermittent 502s with no error in your logs.
Set `keepAliveTimeout` **above** the LB's, and `headersTimeout` above that. This is a classic
"we get random 502s" production story — great to have ready.

### Express vs Fastify vs Nest vs Hono

| | Express | Fastify | NestJS | Hono |
|---|---|---|---|---|
| Style | minimal, middleware | minimal, schema-first | opinionated, DI, decorators | minimal, web-standard |
| Speed | baseline | **~2–3× faster** | Express/Fastify under the hood | very fast, edge-friendly |
| Validation | bring your own | **built-in JSON Schema** | class-validator | bring your own |
| Async errors | ⚠️ v4 doesn't catch them; **v5 does** | handled | handled | handled |
| Best for | anything, huge ecosystem | performance, typed APIs | large teams, enterprise | edge/serverless, portability |

🎯 **"Express or Fastify?"** — "Express for ecosystem and familiarity; Fastify when I want speed and
schema-driven validation and serialization, since it gives me request validation, response
serialization (via `fast-json-stringify`), and TypeScript types from one schema. Express 5 finally
handles async errors natively, which removes its biggest footgun."

```js
// Express 5
import express from "express";
const app = express();
app.use(express.json({ limit: "1mb" }));       // ⚠️ ALWAYS cap the body size
app.get("/users/:id", async (req, res) => {
  const user = await service.get(req.params.id);
  if (!user) throw new NotFoundError("user");   // ✅ Express 5 forwards async throws
  res.json(user);
});

// Fastify
import Fastify from "fastify";
const fastify = Fastify({ logger: true, bodyLimit: 1_048_576 });
fastify.post("/users", {
  schema: {
    body:     { type: "object", required: ["name"], properties: { name: { type: "string" } } },
    response: { 201: { type: "object", properties: { id: { type: "string" } } } },
  },
}, async (req, reply) => reply.code(201).send(await service.create(req.body)));
```

---

## 31. Middleware & error handling

```js
// middleware signature: (req, res, next)
const requestId = (req, res, next) => {
  req.id = req.headers["x-request-id"] ?? crypto.randomUUID();
  res.setHeader("x-request-id", req.id);
  next();
};

const requestLogger = (req, res, next) => {
  const start = process.hrtime.bigint();
  res.on("finish", () => {                   // 'finish' fires when the response is sent
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    logger.info({ id: req.id, method: req.method, url: req.originalUrl,
                  status: res.statusCode, ms }, "request");
  });
  next();
};

// ERROR middleware has FOUR arguments — that's how Express identifies it
const errorHandler = (err, req, res, next) => {
  if (res.headersSent) return next(err);      // ⚠️ can't change a response already streaming
  const status = err.status ?? 500;
  if (status >= 500) logger.error({ err, id: req.id }, "server error");
  res.status(status).json({
    error: {
      code: err.code ?? "INTERNAL",
      // ⚠️ NEVER leak internals or stack traces to clients
      message: status >= 500 ? "internal server error" : err.message,
      requestId: req.id,
    },
  });
};

app.use(requestId, requestLogger);
app.use("/api", routes);
app.use((req, res) => res.status(404).json({ error: { code: "NOT_FOUND" } }));
app.use(errorHandler);        // 🔑 registered LAST
```
⚠️ **Order matters absolutely.** Middleware runs in registration order; the error handler must be
last; the 404 handler goes after all routes but before the error handler.

⚠️ **Express 4 does NOT catch async errors** — an `async` handler that throws produces a hung
request, not a 500. Fixes: upgrade to **Express 5**, use `express-async-errors`, or wrap:
```js
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
app.get("/x", asyncHandler(async (req, res) => { ... }));
```
This is one of the most commonly asked Express questions.

---

## 32. Validation

🔑 **Validate everything crossing a boundary.** TypeScript types vanish at runtime; a request body
is `unknown` until you check it.

```ts
import { z } from "zod";

const CreateUser = z.object({
  name: z.string().min(1).max(100).trim(),
  email: z.email(),                 // ⚠️ zod 4 moved these to top level: z.email(), z.url(),
                                    //    z.uuid(). The old z.string().email() still works but
                                    //    is deprecated — know both, tutorials use the old form.
  age: z.number().int().min(13).max(120).optional(),
  role: z.enum(["admin", "user"]).default("user"),
  tags: z.array(z.string()).max(10).default([]),
});

type CreateUserDto = z.infer<typeof CreateUser>;      // 🔑 ONE source of truth

const validate = (schema, source = "body") => (req, res, next) => {
  const parsed = schema.safeParse(req[source]);
  if (!parsed.success) {
    return res.status(400).json({
      error: { code: "VALIDATION_FAILED", details: z.treeifyError(parsed.error) },
    });
  }
  req[source] = parsed.data;      // ✅ use the PARSED value — coerced, defaulted, stripped
  next();
};

app.post("/users", validate(CreateUser), async (req, res) => {
  res.status(201).json(await service.create(req.body));   // req.body is CreateUserDto
});
```
🔑 Zod **strips unknown keys by default** — that alone prevents mass-assignment attacks
(a client sending `{"role":"admin"}` to a signup endpoint). Use `.strict()` to reject instead.

Alternatives to name: **valibot** (much smaller bundle), **typebox** (JSON Schema + types, pairs
perfectly with Fastify), **ArkType**, **class-validator** (NestJS), **joi** (older, no TS inference),
and **Fastify's built-in JSON Schema**.

---

## 33. Databases & connection pools

```js
import pg from "pg";
const pool = new pg.Pool({
  connectionString: env.DATABASE_URL,
  max: 20,                          // 🔑 tune: total connections across ALL instances < DB limit
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,   // fail fast when the pool is exhausted
  allowExitOnIdle: false,
});
pool.on("error", (err) => logger.error({ err }, "idle client error"));

// ✅ parameterized — NEVER string-concatenate user input
const { rows } = await pool.query("SELECT id, name FROM users WHERE id = $1", [id]);

// transaction — the correct shape
async function transfer(from, to, amount) {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    await client.query("UPDATE acct SET bal = bal - $1 WHERE id = $2", [amount, from]);
    await client.query("UPDATE acct SET bal = bal + $1 WHERE id = $2", [amount, to]);
    await client.query("COMMIT");
  } catch (err) {
    await client.query("ROLLBACK");
    throw err;
  } finally {
    client.release();               // ⚠️ MANDATORY — in `finally`, or the pool leaks
  }
}
```
⚠️ **The #1 Node database bug: not releasing the client.** An early `return` inside the try without
a `finally` leaks a connection; ~20 of those and every request hangs on `connectionTimeoutMillis`.

🎯 **Pool sizing:** "Total connections = instances × pool size, and that has to stay under the
database's `max_connections` with headroom. Ten pods with `max: 20` is 200 connections — that will
exhaust a default Postgres. For serverless or high replica counts I put **PgBouncer** in front."

⚠️ **N+1 queries** — the classic ORM trap:
```js
// 🐛 1 + N queries
const users = await User.findAll();
for (const u of users) u.posts = await Post.findAll({ where: { userId: u.id } });

// ✅ one query, or two
const users = await User.findAll({ include: [Post] });          // eager load / JOIN
const posts = await Post.findAll({ where: { userId: { in: ids } } });   // then group in memory
```

**Ecosystem:** `pg`/`mysql2` (drivers), **Prisma** (great DX, generated types, heavier),
**Drizzle** (SQL-like, TS-first, lightweight — the trendy answer), **Kysely** (typed query builder),
**TypeORM**/**Sequelize** (older ORMs), **Knex** (query builder + migrations).
Migrations: whatever your ORM provides, or `node-pg-migrate`.

---

## 34. Security

```js
import helmet from "helmet";
import rateLimit from "express-rate-limit";
import cors from "cors";

app.use(helmet());                                   // sets ~15 security headers
app.use(cors({ origin: env.ALLOWED_ORIGINS.split(","), credentials: true }));
app.use(rateLimit({ windowMs: 60_000, limit: 100, standardHeaders: "draft-7" }));
app.use(express.json({ limit: "100kb" }));           // cap body size
app.disable("x-powered-by");
```

### The checklist they expect
| Risk | Defense |
|---|---|
| **SQL injection** | parameterized queries, always. Never template literals into SQL. |
| **NoSQL injection** | validate types — `{"$ne": null}` as a password field is the classic Mongo attack |
| **Command injection** | `execFile`/`spawn` with an args array; never `exec` with interpolation |
| **Path traversal** | `path.resolve` + verify the result is inside the allowed root |
| **Prototype pollution** | reject `__proto__`/`constructor` keys; `Object.create(null)` for maps |
| **ReDoS** | bound input length; avoid nested quantifiers; consider `re2` |
| **XSS** (if rendering) | escape output; CSP via helmet |
| **CSRF** | SameSite cookies + CSRF tokens for cookie-authed state changes |
| **Secrets in git** | `.env` gitignored; a secret manager in prod; rotate |
| **Dependency CVEs** | `npm audit`, Dependabot/Renovate, `npm ci` with a committed lockfile |
| **Supply chain** | `npm ci --ignore-scripts` where possible; pin versions; review new deps |

```js
// passwords
import argon2 from "argon2";                    // preferred; bcrypt is also fine
const hash = await argon2.hash(password);
await argon2.verify(hash, attempt);
// ⚠️ bcrypt/argon2 are CPU-heavy and run on the libuv thread pool — they can saturate it (§11)

// tokens & comparison
import crypto from "node:crypto";
crypto.randomBytes(32).toString("base64url");    // ✅ CSPRNG — never Math.random() for secrets
crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));   // constant-time compare
crypto.randomUUID();
```

🎯 **JWT questions come up constantly:**
- Header.Payload.Signature, base64url-encoded. **The payload is signed, not encrypted** — never put
  secrets in it.
- Always pin the algorithm on verify (`{ algorithms: ["HS256"] }`) — the `alg: none` and
  RS256→HS256 confusion attacks both exploit trusting the header.
- Short-lived access token (5–15 min) + a long-lived, **rotating, revocable** refresh token.
- **You cannot revoke a stateless JWT** — that's the core trade-off. Mitigate with short TTLs plus a
  denylist for logout, or use server-side sessions when revocation matters more than statelessness.
- Store refresh tokens in `httpOnly; Secure; SameSite=Strict` cookies, not `localStorage` (XSS).

---

## 35. Testing

### The built-in runner (no Jest required)
```js
import test, { describe, it, before, after, mock } from "node:test";
import assert from "node:assert/strict";

describe("UserService", () => {
  let service, repo;
  before(() => { repo = { findById: mock.fn(async () => ({ id: "1", name: "a" })) };
                 service = new UserService(repo); });

  it("returns the user", async () => {
    const u = await service.get("1");
    assert.deepEqual(u, { id: "1", name: "a" });
    assert.equal(repo.findById.mock.callCount(), 1);
    assert.deepEqual(repo.findById.mock.calls[0].arguments, ["1"]);
  });

  it("throws when missing", async () => {
    repo.findById = mock.fn(async () => null);
    await assert.rejects(() => service.get("nope"), NotFoundError);
  });
});
```
```bash
node --test                          # discovers *.test.js, test/**
node --test --watch
node --test --experimental-test-coverage
node --test --test-name-pattern="user"
```

### HTTP tests with supertest
```js
import request from "supertest";
import { app } from "../src/app.js";

it("validates the body", async () => {
  const res = await request(app).post("/users").send({ name: "" });
  assert.equal(res.status, 400);
  assert.equal(res.body.error.code, "VALIDATION_FAILED");
});
```

### Timers and time
```js
mock.timers.enable({ apis: ["setTimeout", "Date"] });
mock.timers.tick(5000);              // no more sleeping in tests
mock.timers.reset();
```

🎯 **Test strategy answer:** "Unit tests for pure logic with fakes at the boundaries. Integration
tests for the HTTP layer with **testcontainers** running a real Postgres/Redis — mocking a database
tests your mock, not your SQL. A thin end-to-end layer on the critical paths. I mock the *network*
(`nock`/`msw`), not my own modules, because module mocking couples tests to implementation."

**Ecosystem:** `node:test` (built in, zero deps), **Vitest** (fast, great TS/ESM support — the
modern default), **Jest** (huge ecosystem, slower, ESM support still awkward), `supertest`, `nock`,
`msw`, `testcontainers`, `fast-check` (property-based).

---

## 36. Graceful shutdown & health checks 🔑

Near-guaranteed interview question, and one most candidates get partly wrong.

```js
const server = app.listen(env.PORT);
let shuttingDown = false;

async function shutdown(signal) {
  if (shuttingDown) return;                     // ⚠️ idempotent — SIGTERM can arrive twice
  shuttingDown = true;
  logger.info({ signal }, "shutting down");

  // 1. fail readiness FIRST so the load balancer stops sending new traffic
  //    (see the /ready handler below), then give it time to notice
  await sleep(5_000);

  // 2. stop accepting new connections; the callback fires when in-flight requests finish
  await new Promise((resolve) => server.close(resolve));

  // 3. close dependencies in reverse dependency order
  await queue.close();
  await pool.end();
  await redis.quit();

  // 4. flush telemetry
  await logger.flush?.();

  process.exit(0);
}

process.on("SIGTERM", () => shutdown("SIGTERM"));   // ← what Kubernetes/Docker send
process.on("SIGINT",  () => shutdown("SIGINT"));    // ← Ctrl-C

// hard deadline — never hang forever
const forceTimer = setTimeout(() => {
  logger.fatal("shutdown timed out, forcing exit");
  process.exit(1);
}, 30_000);
forceTimer.unref();
```

```js
app.get("/health", (req, res) => res.json({ status: "ok" }));   // liveness: am I alive?

app.get("/ready", async (req, res) => {                          // readiness: can I serve?
  if (shuttingDown) return res.status(503).json({ status: "shutting_down" });
  try {
    await pool.query("SELECT 1");
    res.json({ status: "ready" });
  } catch (err) {
    res.status(503).json({ status: "not_ready" });
  }
});
```
🎯 **The points that score:**
- **Liveness vs readiness are different.** Liveness failing → restart me. Readiness failing → stop
  routing to me, but don't kill me. Putting a DB check in the *liveness* probe means a brief DB
  blip restarts your entire fleet — a classic self-inflicted outage.
- **The 5-second sleep before `server.close()` is the part everyone forgets.** Kubernetes removes
  the pod from endpoints *asynchronously*, so traffic keeps arriving for a second or two after
  SIGTERM. Close too fast and you serve connection resets.
- `server.close()` waits for in-flight requests but **not for idle keep-alive sockets** — use
  `server.closeIdleConnections()` (Node 18.2+) or the `http-terminator` package.
- Always set a **force-exit deadline**.

---

## 37. Observability

```js
import pino from "pino";
const logger = pino({
  level: env.LOG_LEVEL,
  redact: ["req.headers.authorization", "req.headers.cookie", "*.password", "*.token"],
  formatters: { level: (label) => ({ level: label }) },
});
logger.info({ userId, durationMs }, "request completed");
```
🔑 **Structured JSON logs to stdout, one line per event.** Let the platform handle shipping and
rotation. `pino` is the standard (very fast, low overhead); `winston` is the older alternative.
⚠️ Redact secrets. ⚠️ `console.log` is synchronous to a file/pipe in some cases and has no levels —
don't use it in a server.

### Request context without prop-drilling
```js
import { AsyncLocalStorage } from "node:async_hooks";
const als = new AsyncLocalStorage();

app.use((req, res, next) => als.run({ requestId: req.id, userId: req.user?.id }, next));

// anywhere deeper in the call stack, no parameter passing:
function log(msg) { logger.info({ ...als.getStore(), msg }); }
```
🔑 `AsyncLocalStorage` is Node's answer to thread-local storage — it survives across `await` points.
It's how request IDs and trace context reach your logs. Mention the small perf cost.

### Metrics & tracing
```js
import client from "prom-client";
client.collectDefaultMetrics();                 // gives you event loop lag, GC, heap for free
const httpDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  labelNames: ["method", "route", "status"],    // ⚠️ NEVER label with a raw URL or user id
});
app.get("/metrics", async (req, res) => res.type(client.contentType).send(await client.register.metrics()));
```
🎯 Alert on: **event loop lag p99**, error rate, request duration p95/p99, `heapUsed` trend,
active handles, and RSS vs container limit. OpenTelemetry (`@opentelemetry/auto-instrumentations-node`)
gives distributed tracing with almost no code changes — worth naming.

---

## 38. Production war stories they ask about

**"The service is slow but CPU is low."**
Event loop blocked (check loop lag), connection pool exhausted, a missing timeout on an upstream
call, thread pool saturation from crypto/fs/zlib, or DNS latency. Check loop lag first — it
separates "Node is stuck" from "we're waiting on someone else."

**"Memory grows until the pod restarts."**
§23: is it `heapUsed`, `external`, or RSS? Two heap snapshots, compare, follow the retainers.
Usual causes: per-request event listeners, an unbounded cache, closures over big buffers, timers
never cleared.

**"We get random 502s."**
`keepAliveTimeout` shorter than the load balancer's idle timeout (§30). Or shutdown closing the
server before the LB deregisters the pod (§36).

**"One endpoint takes down the whole service."**
Synchronous work on the event loop — a huge `JSON.parse`, a sync fs call, catastrophic regex
backtracking, or unbounded `Promise.all`. Everything on one thread means one bad endpoint is a
full outage. Fix: bound input size, move CPU work to a worker, add timeouts everywhere.

**"How do you handle a background job?"**
`setImmediate`/`setTimeout` is not a job queue — it dies with the process. **BullMQ on Redis** with
retries, backoff, a dead-letter queue, and idempotent handlers. Make every handler idempotent via an
idempotency key.

**"How do you deploy safely?"**
Health + readiness probes, graceful shutdown with a pre-stop delay, rolling or canary deploys,
feature flags, backward-compatible migrations (expand → migrate → contract), and a rollback plan.

**"How do you debug production?"**
Structured logs with a request ID, distributed tracing, `--inspect` on a canary instance (never
publicly exposed), heap snapshots on demand, `--cpu-prof` for a sampled window, and core-dump
analysis with `llnode` as a last resort.

---

# PART 6 — QUESTION BANK

## 39. Rapid-fire Q&A

### JavaScript
1. **`==` vs `===`?** §1. Always `===`, except `x == null`.
2. **`var`/`let`/`const`?** §2 table. All hoisted; `let`/`const` have a TDZ.
3. **What is a closure?** A function plus its captured lexical scope. §3.
4. **How does `this` work?** Five rules, §4. Arrows have no own `this`.
5. **`call` vs `apply` vs `bind`?** Comma / Array / Bound-later.
6. **Prototypal inheritance?** §5. Classes are sugar over it — but not purely.
7. **`null` vs `undefined`?** Intentionally empty vs never assigned; `undefined` drops out of JSON.
8. **Falsy values?** `false 0 -0 0n "" null undefined NaN` — eight, and `[]`/`{}` are truthy.
9. **Deep vs shallow copy?** `structuredClone` vs spread. `JSON.parse(JSON.stringify())` is lossy.
10. **`map` vs `forEach`?** `map` returns a new array and is chainable; `forEach` returns undefined,
    can't break, and ignores async callbacks.
11. **Debounce vs throttle?** §3.
12. **What is a Symbol for?** Unique property keys that won't collide; well-known symbols like
    `Symbol.iterator` and `Symbol.asyncIterator` customize language behavior.
13. **What makes something iterable?** A `[Symbol.iterator]()` method returning `{next()}`.
14. **`Object.freeze` deep?** No, shallow.
15. **What is the temporal dead zone?** §2.
16. **`Map` vs object?** §6.
17. **What's a `WeakMap` for?** Object-keyed metadata that doesn't prevent GC.
18. **Generator functions?** `function*` + `yield` — lazy sequences, and the basis of async iteration.
19. **What is currying?** `f(a)(b)(c)` — partial application via closures.
20. **Prototype pollution?** §5. A real Node CVE class.

### Async & event loop
21. **Is Node single-threaded?** §17 — the nuanced answer.
22. **Explain the event loop phases.** §11 — timers, pending, idle/prepare, poll, check, close.
23. **`setTimeout(fn,0)` vs `setImmediate`?** §11 — race at top level; immediate always wins inside I/O.
24. **`process.nextTick` vs `Promise.then`?** §12 — and the CJS/ESM twist.
25. **What is the microtask queue?** Promises + `queueMicrotask` + await continuations; drained
    fully between every macrotask.
26. **Can you starve the event loop?** Yes — recursive `nextTick`, or any long sync block.
27. **`Promise.all` vs `allSettled` vs `race` vs `any`?** §13 table.
28. **Does `Promise.all` cancel on failure?** No. Use `AbortController`.
29. **What happens on an unhandled rejection?** Since Node 15, the process exits. §14.
30. **`await` in a `forEach`?** Does nothing. §14.
31. **How do you limit concurrency?** §16 — a semaphore pool or `p-limit`.
32. **What is `AbortController`?** §13 — the standard cancellation signal.
33. **Why is `emit` synchronous?** §15 — and why a slow listener blocks the emitter.
34. **What crashes on an `'error'` event with no listener?** The process. §15.
35. **`for await...of`?** Async iteration — streams, paginated APIs.
36. **What's the libuv thread pool for?** fs, DNS `lookup`, crypto, zlib. Default 4. §11.
37. **Does network I/O use the thread pool?** No — epoll/kqueue/IOCP.
38. **Is `async` faster than callbacks?** No — same event loop, slightly more overhead. It's about
    readability and error handling.
39. **What does an `async` function return?** Always a promise.
40. **When does an async function's body start running?** Immediately, synchronously, until the
    first `await`.

### Node runtime
41. **CJS vs ESM?** §18 table.
42. **What's `__dirname` in ESM?** Doesn't exist — `import.meta.dirname`.
43. **Is a module cached?** Yes, per resolved path, once per process.
44. **Circular dependencies?** CJS gives partial exports; ESM handles it with hoisting but can throw
    a TDZ error. Fix the design.
45. **What is `package.json` `exports`?** An encapsulation boundary + conditional resolution. §18.
46. **Streams: why?** Constant memory over unbounded data + backpressure. §19.
47. **`pipe` vs `pipeline`?** **Pipe doesn't propagate errors or clean up; pipeline does.** §19.
48. **What is backpressure?** §19 — `write()` returning `false`, then `'drain'`.
49. **Four stream types?** Readable, Writable, Duplex, Transform.
50. **What's `highWaterMark`?** The buffer threshold — 64KB bytes / 16 objects.
51. **`Buffer.alloc` vs `allocUnsafe`?** Zero-filled vs uninitialized (fast, security risk). §20.
52. **Do Buffers live in the V8 heap?** No — external memory. That's why RSS ≫ heapUsed. §20.
53. **cluster vs worker_threads?** §22 table.
54. **How do you use all CPU cores?** `cluster`, or container replicas.
55. **How do you do CPU-heavy work?** `worker_threads` with a pool (`piscina`).
56. **`exec` vs `spawn` vs `execFile`?** `exec` uses a shell (injection risk) and buffers output;
    `spawn` streams; `execFile` skips the shell.
57. **How does V8 GC work?** §23 — generational, scavenger + mark-sweep-compact, mostly concurrent.
58. **How do you find a memory leak?** §23 — two heap snapshots, compare, follow retainers.
59. **Exit code 137?** SIGKILL — container OOM killer.
60. **How do you set the heap limit?** `--max-old-space-size`, ~75–80% of the container limit.
61. **How do you profile CPU?** `--cpu-prof`, clinic, 0x. §24.
62. **How do you detect a blocked event loop?** `perf_hooks.monitorEventLoopDelay`. §24.
63. **What is `AsyncLocalStorage`?** §37 — request context across awaits.
64. **`process.exit()` vs `process.exitCode`?** Immediate vs graceful drain.

### TypeScript
65. **`any` vs `unknown`?** §26 — the one they always ask.
66. **`interface` vs `type`?** §26.
67. **What does `strict` enable?** `strictNullChecks`, `noImplicitAny`, and friends.
68. **Do types exist at runtime?** **No.** Hence runtime validation at boundaries. §28.
69. **What's a discriminated union?** §26 — plus `assertNever` for exhaustiveness.
70. **`Partial`/`Pick`/`Omit`/`Record`?** §27.
71. **What does `as` do?** Silences the compiler; performs no runtime check.
72. **`satisfies`?** Check without widening. §26.
73. **What's `keyof typeof X[number]` for?** Turning a const array into a union. §27.
74. **Should you use `enum`?** Prefer `as const` unions — enums emit runtime code.
75. **Does `tsx`/esbuild type-check?** No. Run `tsc --noEmit` in CI.

### Backend
76. **Express 4 and async errors?** Not caught — hangs. Express 5 fixes it. §31.
77. **How does error middleware get recognized?** By its **four** parameters.
78. **Why Fastify over Express?** Speed + schema-driven validation/serialization/types. §30.
79. **How do you validate input?** zod/typebox at the boundary, derive types. §32.
80. **How do you prevent SQL injection?** Parameterized queries.
81. **How do you size a connection pool?** instances × pool < DB max_connections. §33.
82. **What's an N+1 query?** §33.
83. **How do you store passwords?** argon2/bcrypt — never plain, never fast hashes.
84. **JWT vs sessions?** §34 — statelessness vs revocability.
85. **Can you revoke a JWT?** Not by itself. Short TTL + refresh rotation + denylist.
86. **How do you rate limit?** Token bucket in Redis for multi-instance; `express-rate-limit` for one.
87. **Graceful shutdown steps?** §36 — fail readiness, wait, close server, close deps, force-exit timer.
88. **Liveness vs readiness?** §36 — and why a DB check belongs only in readiness.
89. **Why random 502s behind a load balancer?** `keepAliveTimeout` too low. §30.
90. **How do you run background jobs?** BullMQ + Redis, idempotent handlers, DLQ.
91. **How do you handle file uploads?** Stream to disk/S3 with a size cap; `busboy`/`multer`;
    sanitize the filename; validate the content type by magic bytes, not the extension.
92. **`npm ci` vs `npm install`?** `ci` is lockfile-exact, deletes `node_modules`, fails on drift —
    the correct choice in CI and Docker.
93. **dependencies vs devDependencies vs peerDependencies?** Runtime / build-and-test / "the host
    must provide this."
94. **What's in a good Dockerfile for Node?** Multi-stage build, `npm ci --omit=dev`, non-root user,
    `NODE_ENV=production`, a proper init (`--init` or tini) so SIGTERM reaches Node, `.dockerignore`.
95. **Semver?** major.minor.patch; `^` allows minor+patch, `~` allows patch only.
96. **How do you test HTTP endpoints?** supertest against the app object — no port binding needed.
97. **How do you mock an external API?** `nock` or `msw` — mock the network, not your modules.
98. **What logging library?** `pino`, structured JSON to stdout, redact secrets.
99. **How do you pass a request ID down the stack?** `AsyncLocalStorage`.
100. **Node vs Go for a backend?** Node: I/O-heavy, fast iteration, shared language with the
     frontend, huge ecosystem. Go: CPU-bound work, true parallelism, predictable latency, single
     static binary, lower memory. Be honest about the trade-off rather than picking a favourite.

---

## 40. Output-prediction drills

Cover the answers and work through each one.

**1.**
```js
console.log(typeof null, typeof [], typeof NaN, typeof function(){});
```
→ `object object number function`

**2.**
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log(j), 0);
```
→ `3 3 3` then `0 1 2`

**3.**
```js
const obj = { name: "a", greet() { return this.name; } };
const fn = obj.greet;
console.log(obj.greet(), fn());
```
→ `a` then `undefined` (or a TypeError in strict mode)

**4.**
```js
async function f() { console.log(1); await null; console.log(3); }
f(); console.log(2);
```
→ `1 2 3` — the body runs synchronously to the first `await`

**5.**
```js
setTimeout(() => console.log("timeout"));
Promise.resolve().then(() => console.log("promise"));
console.log("sync");
```
→ `sync promise timeout`

**6.**
```js
console.log([1, 10, 9].sort());
```
→ `[1, 10, 9]` — lexicographic

**7.**
```js
console.log(0.1 + 0.2 === 0.3, [] == false, "" == 0, null == undefined, NaN === NaN);
```
→ `false true true true false`

**8.**
```js
function f() { try { return 1; } finally { return 2; } }
console.log(f());
```
→ `2`

**9.**
```js
const p = Promise.reject(new Error("x"));
setTimeout(() => p.catch(() => console.log("caught")), 100);
```
→ In Node 15+, the unhandled rejection is detected before the late `.catch` attaches → the process
crashes. Attach handlers synchronously.

**10.**
```js
process.env.RETRIES = "0";
console.log(process.env.RETRIES || 3);
console.log(Number(process.env.RETRIES) || 3);
console.log(Number(process.env.RETRIES ?? 3));
```
→ `"0"`, then `3`, then `0`.
🎯 The trap inside the trap: env values are **strings**, and `"0"` is truthy — so the first line is
fine. The bug appears only *after* coercion, when `Number("0")` becomes falsy `0` and `||` swallows
it. Most people get this half-right; getting it fully right reads as very senior. §1.

**11.**
```js
const a = { x: { y: 1 } };
const b = { ...a };
b.x.y = 2;
console.log(a.x.y);
```
→ `2` — spread is shallow

**12.**
```js
[1, 2, 3].forEach(async (n) => { await sleep(10); console.log(n); });
console.log("done");
```
→ `done` first, then `1 2 3` in ~10ms — `forEach` doesn't await

---

## 41. 15-minute cram sheet

**Say these and you'll sound like a Node engineer:**

- "Node runs your JavaScript on one thread with an event loop, and delegates I/O to libuv. Network
  I/O uses epoll/kqueue with no threads; file I/O, DNS lookup, crypto, and zlib use a 4-thread pool."
- "The phases are timers, pending callbacks, idle/prepare, poll, check, close — and the nextTick and
  microtask queues drain completely between every callback."
- "`setImmediate` always beats `setTimeout(fn, 0)` inside an I/O callback because check follows poll.
  At the top level it's a race — don't depend on it."
- "An async function body runs synchronously until its first `await`."
- "`Promise.all` is fail-fast and doesn't cancel siblings; `allSettled` never rejects, which is what
  I want for batch work."
- "`await` inside `forEach` does nothing — `for...of` for sequential, `Promise.all(map)` for
  parallel, and a bounded pool for anything large."
- "Unhandled rejections crash the process since Node 15. My handlers log and exit; I don't try to
  recover, because the process state is undefined."
- "`pipe` doesn't forward errors or clean up — I always use `stream.pipeline`."
- "Backpressure is `write()` returning false and waiting for `'drain'`. Ignoring it is how you OOM."
- "`Buffer` lives outside the V8 heap, so a buffer leak shows in `external` and RSS, not `heapUsed`."
- "For CPU work I use `worker_threads` with a pool; for scaling HTTP across cores, `cluster` on a VM
  or container replicas under an orchestrator."
- "I monitor event loop delay p99 — that's the single best signal that something synchronous is
  blocking the server."
- "Set `--max-old-space-size` to about 80% of the container limit, or Node will get OOM-killed with
  exit code 137 and no stack trace."
- "TypeScript is erased at runtime, so I validate every boundary with zod and derive the type from
  the schema."
- "Express 4 doesn't catch async errors; Express 5 does. Error middleware is identified by having
  four parameters."
- "Graceful shutdown: fail readiness, wait ~5s for the LB to deregister, `server.close()`, close the
  pool and queues, and a force-exit timer as a backstop."
- "Liveness failing restarts the pod; readiness failing just stops traffic. A DB check belongs only
  in readiness."
- "`keepAliveTimeout` must exceed the load balancer's idle timeout, or you get random 502s."
- "`npm ci` in CI and Docker — lockfile-exact and it fails on drift."

**Commands to have at your fingertips**
```bash
node --test --experimental-test-coverage
node --watch --env-file=.env src/server.js
node --inspect server.js            # chrome://inspect
node --cpu-prof --cpu-prof-dir=./prof server.js
node --trace-gc --max-old-space-size=1024 server.js
npx clinic doctor -- node server.js
npm ci --omit=dev && npm audit --audit-level=high
npx tsc --noEmit
UV_THREADPOOL_SIZE=16 node server.js
```

**Three questions to ask THEM (shows seniority)**
- "How do you handle graceful shutdown and in-flight requests during deploys?"
- "Are you on CommonJS or ESM, and how did that migration go?"
- "What's your story for CPU-bound work — workers, a separate service, or does it not come up?"

---

## Practice plan before the mock

| Day | Focus |
|---|---|
| 1 | Part 1 §1–4. Re-derive the falsy list and the five `this` rules from memory. |
| 2 | Part 1 §5–9 + Part 2 §10. Write `once`, `memoize`, `debounce`, `throttle` from scratch. |
| 3 | **Part 2 §11–12.** Run every ordering example yourself. Predict before you run. |
| 4 | Part 2 §13–16. Write the concurrency pool and the retry-with-jitter without looking. |
| 5 | **Part 3 §19.** Build a real pipeline: read → transform → gzip → write, with error handling. |
| 6 | Part 3 §17–18, §20–22. Explain GMP-style architecture and cluster vs workers out loud. |
| 7 | Part 3 §23–24. Deliberately leak memory, then find it with two heap snapshots. |
| 8 | Part 4. Get a small service to pass `tsc --strict --noEmit`. |
| 9 | Part 5 §30–33. Build a server with middleware, zod validation, and a pooled DB. |
| 10 | Part 5 §34–38. Add graceful shutdown, health/ready, pino, and three tests. |
| 11 | §39 rapid-fire out loud. |
| 12 | §40 drills — predict every output before running it. Then §41. |

---

*Notes prepared for a mid-level Node.js backend interview loop. Event loop ordering, timer races,
and module-system differences were verified by execution on Node v22.22.2; version-specific behavior
is flagged inline. Current as of Aug 2026: Node 26 Current, Node 24 LTS, Node 22 Maintenance LTS.*
