# Interpreted Language and the role of V8
### Machine Code:
- RISC vs CISC => CISC are complex and are not usually done in one cpu cycle. RISC are somewhat simpler and thats why are used in arm which is in mobile device.

### ⚙️ **CISC (Complex Instruction Set Computing)**

* **Philosophy:** Make instructions *powerful* so one instruction can do a lot (e.g., load from memory + add + store back in one step).
* **Examples:** Intel x86, AMD64.
* **Characteristics:**

  * Large, complex instruction set.
  * Some instructions take **multiple CPU cycles**.
  * Easier for **assembly programmers** (back in the day), but more work for hardware to decode/execute.
  * High transistor cost for the decoder and microcode.
  
* **Pros:** Fewer instructions per program (since one instruction can do more).
* **Cons:** Harder to pipeline, more power-hungry, variable instruction length.

---

### ⚙️ **RISC (Reduced Instruction Set Computing)**

* **Philosophy:** Keep instructions *simple and uniform* so they can execute **in 1 CPU cycle** (most of the time).
* **Examples:** ARM, RISC-V, MIPS, SPARC.
* **Characteristics:**

  * Smaller, simpler instruction set.
  * Each instruction does a small task (load, add, store are separate).
  * Fixed instruction size (e.g., 32-bit), easy to pipeline.
  * Needs more instructions per program, but each runs fast.
* **Pros:** High performance per watt, easier parallelism, simpler hardware.
* **Cons:** Compiler must do more work to generate efficient code.

---

### 📱 Why **ARM (RISC) dominates in mobile devices**

* **Power efficiency** is critical (battery life).
* RISC design = simpler, predictable, lower power per instruction.
* Easier to scale cores and integrate in System-on-Chip (SoC).

Meanwhile, **CISC (x86)** dominates desktops/servers because:

* Legacy software ecosystem (decades of x86 code).
* Heavy workloads that benefit from powerful instructions + aggressive micro-optimizations.

---

### 🔎 Fun fact

Modern CPUs blur the line:

* Intel/AMD **x86 (CISC)** chips **internally translate complex CISC instructions into RISC-like micro-ops** before execution.
* So under the hood, even CISC CPUs are “RISC-ish”!

---

👉 So your summary is on point:

* **CISC** = complex instructions, often multi-cycle, used in PCs/servers.
* **RISC** = simpler, efficient instructions, often single-cycle, used in ARM (mobiles, embedded).

---
High Level languages are more convenient than assembly. There are a lot of abstractions to hide complexity. Need to compiile for CPU. comiles turns code to machine code.
translate whatever you have written in an language into something that cpu understands is called compile. we have to compile against an architecture.
intels instructions won't run on mac. or android arm device. 

### Linking
The linker takes all these object files + libraries and connects them into one executable. The file is called "executable".

### INTERPRETD LANGUAGES: 
- A compiled program does not work everywhere.
- Can i write a code that runs everywhere? -> INTERPRETED LANGUAGE
- An interpreted language is one where the code you write (source code) is not directly translated into machine code beforehand. Instead, another program called an interpreter reads and executes it line by line (or statement by statement) at runtime.

Example:
Python, JavaScript, Ruby, PHP are commonly interpreted.

| Feature             | Compiled Languages (C, C++, Go)               | Interpreted Languages (Python, JS, Ruby)           |
| ------------------- | --------------------------------------------- | -------------------------------------------------- |
| **Execution**       | Translated to machine code **before** running | Executed **line by line / via VM**                 |
| **Speed**           | Faster (no runtime translation)               | Slower (translation happens at runtime)            |
| **Portability**     | Needs recompilation on each platform          | Highly portable (interpreter available everywhere) |
| **Error Detection** | Errors caught at compile-time                 | Errors appear at runtime                           |
| **Flexibility**     | Less dynamic (fixed types, structures)        | More dynamic (can eval code at runtime)            |

Your code is NOT compiled to a native .exe.
Instead, it’s fed into an interpreter program (which is a compiled executable, written in C/C++ usually).

Example:

Python → python.exe (Windows) / python3 (Linux)

Node.js (for JavaScript) → node.exe

Great follow-up 👍 Let’s break down **JIT (Just-In-Time compilation)**.

---

### 🔑 What is JIT?

* **JIT = Just-In-Time Compiler**
* It’s a hybrid approach between **compiled** and **interpreted** execution.
* Instead of compiling everything **ahead of time** (like C/C++) or interpreting **line by line** (like old Python), JIT compiles **parts of the code into machine code at runtime** (just in time to run it).

So:
➡️ You get **speed like compiled languages** + **flexibility like interpreted languages**.

---

### 🔨 How JIT Works (Step by Step)

1. **Source Code → Bytecode**

   * Language first compiles your code into bytecode (portable, platform-independent).
   * Example: Java → `.class` file (bytecode).

2. **Interpreter starts execution**

   * Interpreter/VM starts running bytecode instruction by instruction.

3. **Hotspot Detection**

   * The JIT compiler watches which parts of the code run **frequently (hot paths)**.
   * Example: a loop that runs 1 million times.

4. **On-the-fly Compilation**

   * JIT takes those hot parts of bytecode and **compiles them into native machine code**.
   * Stores that machine code in memory.

5. **Direct Execution**

   * Next time that code runs, the VM skips interpretation and **executes the cached machine code directly**.
   * → Much faster execution.

---

### 🔨 Example

### Without JIT (pure interpretation)

```python
for i in range(1_000_000):
    x = i * 2
```

* Each iteration: interpreter reads the loop, parses, runs instruction by instruction.
* Very slow.

### With JIT

* After a few iterations, JIT detects this loop runs many times.
* It compiles the loop into **native CPU code once**.
* The rest of the million iterations run at near-C speed.

---

### 🔑 Languages that use JIT

* **Java** (HotSpot JVM → compiles bytecode into native code at runtime)
* **.NET (C#, F#)** CLR uses JIT
* **JavaScript** (V8 in Chrome/Node.js, SpiderMonkey in Firefox)
* **PyPy** (a JIT-enabled Python implementation)

---

### ⚡ Comparison

| Method             | Description                                       | Speed                                 |
| ------------------ | ------------------------------------------------- | ------------------------------------- |
| **Compiled (AOT)** | Compile everything to machine code before running | Fast startup + fast runtime           |
| **Interpreted**    | Execute line by line with interpreter             | Slow                                  |
| **JIT**            | Compile *hot* code to machine code at runtime     | Startup slower, but runtime very fast |


---

✅ **In short:**
JIT = **Compiles parts of interpreted code into machine code at runtime**, giving a balance of **performance + flexibility**.

---

## **Garbage Collection in Node.js (V8)**

#### What is Memory Management?

Memory management = allocating memory when needed, freeing it when done. Forgetting to free memory = memory leak → program consumes ever-more RAM until crash.

#### Who manages memory?

| **Language**         | **Memory Management** | **Notes**                                                      |
| -------------------- | --------------------- | -------------------------------------------------------------- |
| C, C++               | Manual                | You call malloc() / free() — full control, full responsibility |
| Go, Python, Java     | Automatic (GC)        | Runtime handles allocation and freeing                         |
| JavaScript / Node.js | Automatic (V8 GC)     | V8's garbage collector — built into the runtime                |
## V8 Heap Layout

V8 splits its heap into two main generations:
-  Young Generation (New Space): newly created objects land here — small, collected frequently and fast
- Old Generation (Old Space): objects that survive multiple GC cycles get promoted here — larger, collected less often

```
┌─────────────────────────────────────────────┐
│               V8 HEAP                       │
│                                             │
│  ┌─────────────────┐  ┌─────────────────┐   │
│  │  Young Gen      │  │   Old Gen       │   │
│  │  (New Space)    │  │   (Old Space)   │   │
│  │                 │  │                 │   │
│  │  Short-lived ───┼─►│  Long-lived     │   │
│  │  objects        │  │  objects        │   │
│  └─────────────────┘  └─────────────────┘   │
└─────────────────────────────────────────────┘
```

### How V8 Tracks Objects — Reachability

V8 does NOT use reference counting. Instead it uses reachability:

> **_"Is this object reachable from a root?"_**

Roots include: global variables, currently executing function's local variables, the call stack.
If an object cannot be reached from any root → it is garbage → V8 collects it.

```js
function example() {
  let user = { name: 'Alice' };  // allocated on heap
  // user is reachable here
  user = null;   // no more references
  // { name: 'Alice' } is now unreachable → garbage
}
```

## V8's Two GC Algorithms
### 1. Scavenger (Minor GC) — Young Generation
•       Runs very frequently, very fast (milliseconds)
•       Uses semi-space copying: surviving objects copied from one half of New Space to the other
•       Objects surviving 2 scavenges → promoted to Old Generation
•       Dead objects are simply left behind and overwritten

### 2. Mark-Sweep & Mark-Compact (Major GC) — Old Generation
•       Runs less frequently, takes longer
◦       Mark phase: V8 traverses all reachable objects from roots and marks them
◦       Sweep/Compact phase: unmarked objects are freed; survivors are compacted to reduce fragmentation

```
MARK PHASE:

  Root → Object A (marked ✓) → Object B (marked ✓)

                             → Object C (marked ✓)

  Object D — unreachable (not marked) → collected

SWEEP PHASE:

  Object D's memory is reclaimed.
```

## Stop-The-World and Incremental GC
A naive GC pauses all JS execution while it runs (stop-the-world) → visible freezes / lag spikes.

V8 avoids this with:
•       Incremental Marking: marking done in small increments interleaved with JS execution
•       Concurrent Marking: marking runs on a background thread while JS continues
•       Lazy Sweeping: sweeping spread out over time, not all at once

## When GC Causes Slowdowns

| **Situation**                       | **Effect**                                     |
| ----------------------------------- | ---------------------------------------------- |
| Many short-lived object allocations | Scavenger (Minor GC) runs more often           |
| Many long-lived objects             | Old Gen grows → Major GC is expensive          |
| Memory leak (objects never freed)   | Heap grows → eventual crash or severe slowdown |
| GC pause during an HTTP request     | Increased latency / tail latency spikes        |

## Common Memory Leak Patterns in Node.js

•       Event listeners never removed
```js
emitter.on('data', handler);  // added but never removed
```

•       Growing cache with no eviction
```js
const cache = {};

function store(key, val) { cache[key] = val; }  // grows forever
```

•       Closures holding large references
```js
function outer() {
  const bigArray = new Array(1_000_000).fill('x');
  return function inner() {
    console.log(bigArray[0]);  // bigArray never freed
  };
}
```

Monitor GC in production with:
```
node --trace-gc app.js.
```

# Key Takeaways

| **Topic**               | **One-Line Summary**                                                                         |
| ----------------------- | -------------------------------------------------------------------------------------------- |
| CISC vs RISC            | CISC = complex multi-cycle instructions (PC/server); RISC = simple single-cycle (mobile/ARM) |
| Compiled vs Interpreted | Compiled = fast, platform-locked; Interpreted = slower, portable                             |
| JIT                     | Compiles hot code at runtime → near-compiled speed + interpreted flexibility                 |
| GC — Reachability       | If an object has no path from root → it is garbage                                           |
| GC — Minor (Scavenger)  | Fast, frequent — cleans young, short-lived objects                                           |
| GC — Major (Mark-Sweep) | Slower, infrequent — cleans old, long-lived objects                                          |
| GC — Incremental        | V8 breaks GC into small steps to avoid pausing your app                                      |
| Memory Leaks            | Unremoved listeners, unbounded caches, closures = common culprits                            |

## 1. Introduction: How Node.js Executes Code

- **What is Node.js?** At its core, Node.js is a C++ program built around Google's V8 engine that takes a JavaScript file as input.
- **Initial Execution:** When you start a Node process, it reads the provided JS file and interprets it.
- **Synchronous Processing:** The top-level JavaScript code is executed synchronously, line by line, from top to bottom.
- **Termination (The default behavior):** If a script contains only synchronous code (like basic math or variable declarations), the program simply ends once the last line is executed.

## 2. Why We Need the Event Loop (The Asynchronous Problem)

Serial, synchronous execution is not enough for modern applications. There are many situations where a program cannot simply run top-to-bottom and exit. It needs to wait for external events to occur.
**Common Asynchronous Situations:**
- **Timers:** "Execute this specific block of code after _X_ seconds." (e.g., `setTimeout`).

- **File I/O (Input/Output):** "Ask the Operating System to read a file from the hard drive, and execute this code _only_ when the OS is done."

- **Network I/O:** * "Listen on Port 8080 and run this code whenever a new client connects."
    - "Connect to a database server and wait for the requested data to arrive."

Because the Operating System handles things like reading files or making network requests, Node.js needs a way to "wait" for the OS to finish without freezing the entire application. It needs a mechanism to repeatedly check: _"Is the work done yet?"_

## 3. Blocking vs Non-Blocking

### ❌ Blocking (bad for Node)

```js
const data = readFileSync("file.txt"); // blocks everything
```

### ✅ Non-Blocking (Node way)

```js
fs.readFile("file.txt", callback);
```

👉 Node delegates this work to:
- OS
- Thread pool (via libuv)

## 4. Core Concepts: Callbacks and Events

- **Events:** Something that happens at a specific instance in time (e.g., a file finishes reading, a timer expires, a new network connection is established).

- **Callbacks:** A callback is simply a JavaScript function (a piece of code) that you want to execute _after_ a specific event is satisfied.
    Examples:
	- File read complete → run callback
	- Timer expires → run callback
	- Network request received → run callback
```js
fs.readFile("a.txt", function(err, data) {
    console.log(data);
});
```
👉 You are **registering** a callback, not executing it immediately.

- **Wiring:** Node.js "wires" (links) your callback function to the specific event.

- **Registering vs. Executing:** This is a crucial distinction. When you call `setTimeout(callback, 5000)`, you are **registering** the callback immediately. You are _not_ executing it. The execution happens 5 seconds later when the timer event triggers.

- _Note on Promises/Async-Await:_ While syntactically different, Promises and `async/await` are fundamentally built on top of this same callback/queueing architecture under the hood.

## 4. Meet the Event Loop
The Event Loop is the secret behind Node's ability to handle thousands of concurrent operations.

>  The Event Loop is a mechanism that keeps Node running and executes callbacks when events are ready.

**Key Characteristics:**
1. **Single Threaded (Mostly):** The Event Loop and your JavaScript code run on a single thread. _(Note: Node uses a background worker pool in C++ [libuv] to handle the heavy lifting of File I/O and crypto, but the JS execution remains single-threaded)._

2. **Asynchronous Non-Blocking I/O:** Node doesn't wait for I/O operations to finish. It offloads them to the OS, registers a callback, and moves on to the next line of code.

3. **Phases:** The loop is divided into multiple phases (e.g., Timers phase, I/O callbacks phase, Poll phase). Each phase handles specific types of operations.

4. **Callback Queues:** Each phase has its own queue of callbacks waiting to be executed.

5. **Termination Condition:** The Event Loop continually spins, checking the queues. The Node process only terminates when there is absolutely nothing left to do (no active timers, no pending I/O, and all callback queues are empty).

## 5. Summary of the Execution Lifecycle

1. **Read and Execute:** Node takes the JS file and executes the top-level code synchronously.

2. **Register:** During this initial execution, Node might register callbacks for asynchronous tasks (timers, file reads).

3. **Enter the Loop:** Once the synchronous code is done, Node enters the Event Loop.

4. **Check and Execute:** The loop checks the callback queues phase by phase. If a queue has callbacks (because an event finished), it executes them.

5. **Chain Reactions:** Executing a callback might register _more_ callbacks (e.g., reading a file, and then setting a timer based on the data).

6. **Exit:** The loop continues spinning until all queues are entirely empty.

## 6. Code Exercise Breakdown

Here is a step-by-step analysis of how the Event Loop handles your provided code block.

```js
const fs = require("fs")
const x = 1;
const y = 2;
const z = x + y;

function timer1Callback() { console.log("timeout elapsed 1ms") }
function timer2Callback() { fs.readFile("c.txt", readFileCCallback) }
function readFileCCallback() { console.log("read c after a second") }

function writeFileBCallback() {
    console.log("write b.txt");
    setTimeout(timer2Callback, 1000);
}

function readFileACallback(err, data) {
    if (err) console.error(err)
    console.log("read a.txt" + data);
    fs.writeFile("b.txt", "test", writeFileBCallback);
}

fs.readFile("a.txt", readFileACallback);
setTimeout(timer1Callback, 1);

```

### Execution Flow:
1. **Synchronous Phase:**
	- Node requires `fs`. Sets `x=1`, `y=2`, `z=3`. Registers all function definitions in memory.
	- It hits `fs.readFile("a.txt", ...)` -> It **registers** `readFileACallback` with the OS File System and tells it to start reading. _It does not wait._
	-  It hits `setTimeout(timer1Callback, 1)` -> It **registers** `timer1Callback` to run in 1 millisecond.
    - Main file execution finishes. Node enters the Event Loop.

2. **Event Loop Starts:**
	- **(Approx 1ms later):** The timer finishes. `timer1Callback` is pushed to the Timer queue. The Event Loop executes it: **Prints `"timeout elapsed 1ms"`**.
	- **(Sometime later):** The OS finishes reading `a.txt`. `readFileACallback` is pushed to the I/O queue.
	- The Event Loop executes `readFileACallback`: **Prints `"read a.txt [data]"`**.
	- Inside that callback, it hits `fs.writeFile(...)`. It **registers** `writeFileBCallback` with the OS and tells it to start writing to `b.txt`.
	- **(Sometime later):** The OS finishes writing `b.txt`. `writeFileBCallback` goes to the queue.
	- The Event Loop executes `writeFileBCallback`: **Prints `"write b.txt"`**.
	- Inside that callback, it hits `setTimeout(...)`. It **registers** `timer2Callback` to run in 1000ms (1 second).
	- **(1 second later):** The timer finishes. `timer2Callback` goes to the queue.
	- The Event Loop executes `timer2Callback`. Inside it, it hits `fs.readFile("c.txt", ...)`. It **registers** `readFileCCallback` and tells the OS to read `c.txt`.
	-  **(Sometime later):** The OS finishes reading `c.txt`. `readFileCCallback` goes to the queue.
    - The Event Loop executes `readFileCCallback`: **Prints `"read c after a second"`**.
    - **Termination:** The Event Loop checks. No timers are active. No file I/O is pending. All callback queues are empty. The Node process cleanly exits.

### Key Takeaways (Clean Summary)

- Node executes JS **synchronously first**
- Async work is **delegated to OS / libuv**
- Callbacks are **registered**, not executed immediately
- Event loop:
    - Picks callbacks from queues
    - Executes them phase-by-phase
- Node exits when **no work remains**

# The Node.js Main Module & Initial Phase

Before the Node.js Event Loop even initializes, the application goes through a crucial step known as the **Initial Phase**. This phase focuses primarily on loading modules and executing the Main Module.

## 1. Module Loading

Before your main file runs, Node.js must set up the environment and load all dependencies.

- **Modules load first:** Modules are resolved and load before the main module.
    
- **Code execution:** The code within those modules is executed first.
    
- **All imports loaded:** All `require()` or `import` statements are completely loaded before running any code in your main file.
    
- **Recursive loading:** If modules load other modules, that chain is fully resolved beforehand.

## 2. The Main Module Execution

Once all modules are loaded, Node.js executes your primary JavaScript file (the Main Module).

- **Synchronous Execution:** All main code executes synchronously.
    
- **No Callbacks Executed:** **No callbacks can get executed** during this phase. Even if a timer (like `setTimeout`) expires while this code is running, its callback is blocked and must wait.
    
- **Loop Not Initialized:** The main Event Loop is _not yet initialized_.
    
- **Runs Once:** This initial phase runs exactly once.

## 3. Performance Tip: Keep it Short!

> **🚀 Keep the initial phase as short as possible!**
> 
> - This code is executed first.
>     
> - Keeping it short allows for greater performance.
>     
> - It spins up the Node process faster.
>     
> - **Know your modules:** Be careful about importing heavy, synchronous modules that might block this initial phase.


## 4. Code Example: Blocking the Initial Phase
This example visually proves that the Main Module executes completely before the Event Loop (and timers) can start.

```js

const x = 1;
const y = x + 1;

setTimeout(() => console.log("Should run in 1ms"), 1);

// A massive, synchronous loop blocking the initial phase
for (let i = 0; i < 1000000000; i++);

console.log("Will this be printed first?");

```

**Output:**

```
Will this be printed first?
Should run in 1ms
```

**What happens here?**
1. `x` and `y` are assigned synchronously.
    
2. `setTimeout` registers the callback to run in 1ms, but _does not execute it_. Node hands this off to the background timer.
    
3. Node hits the `for` loop. This takes considerable time to count to a billion.
    
4. Even though 1ms passes quickly, **no callbacks can execute** because we are still in the Initial Phase and the Event Loop has not started.
    
5. The synchronous code finishes by printing `"Will this be printed first?"`.
    
6. Only now is the Initial Phase done. The Event Loop initializes, checks the Timers phase, sees the 1ms timer expired long ago, and finally executes the callback.

# THE TIMER PHASE (Node.js Event Loop)

## Overview

- Runs **after the initial phase**.
- First phase inside the **event loop**.
- Managed internally by **Libuv** (Node.js underlying C library).
- Responsible for executing callbacks from:
    - `setTimeout()`
    - `setInterval()`

# Important Concept

Timers are **NOT exact**.

A timer means:

> “Execute this callback **after at least** X milliseconds.”

Actual execution can be delayed because:
- synchronous code is blocking the thread
- other event loop phases are running
- CPU is busy
- long-running callbacks exist

# How Timers Work Internally
## Step-by-step Example

Suppose at `T0` we schedule:
```js
setTimeout(..., 100);
setTimeout(..., 500);
setTimeout(..., 950);
```

## What Libuv Does

### 1. Store timers

Libuv stores:
- callback
- timeout duration
- target end time

Example:

| Timer | End Time   |
| ----- | ---------- |
| T100  | T0 + 100ms |
| T500  | T0 + 500ms |
| T950  | T0 + 950ms |
### 2. Find the smallest timer

Smallest timer = `100ms`

Libuv asks the OS:

> “Wake me after 100ms.”

### 3. Event loop continues

Meanwhile:
- main thread keeps running
- other phases continue normally

### 4. OS wakes Libuv thread

After ~100ms:

- OS wakes Libuv
- timer becomes **ready**

### 5. Timer phase executes callback

When the event loop enters the **timer phase**:
- ready timer callbacks execute

Example:
```
T100 callback executes
```

### 6. Repeat for remaining timers

Next smallest remaining timer:

```
500 - 100 = 400ms
```

OS sleeps again for 400ms.

Then:
- T500 becomes ready
- callback executes

Then same for T950.

# Important Notes

## Timers are sorted by duration

Shortest timer is checked first.

## Timers persist end times

Node remembers:

```
current time + delay
```

NOT:

```
“execute exactly after delay”
```

## Delays can happen

Even if a timer becomes ready:

```
callback waits until:
- current sync code finishes
- current event loop phase ends
```

Example:
```js
const timerCallback = (a,b) => console.log(
  `Timer callback ${a} delayed by ${Date.now() - start - b}`
);

const start = Date.now();

setTimeout(timerCallback, 100, '100 ms',100);
setTimeout(timerCallback, 0, '0 ms',0);
setTimeout(timerCallback, 1, '1 ms', 1);
setTimeout(timerCallback, 300, '300 ms', 300);

// Blocks thread for ~380ms
for (let i = 1; i <= 1_000_000_000; i++);
```

# What Happens Here?

## Initial Phase

All synchronous code executes first.

That includes:

```
for (let i = 1; i <= 1_000_000_000; i++);
```

This blocks the thread for ~380ms.

During this time:
- event loop cannot continue
- timer phase cannot run
- callbacks cannot execute

# After Blocking Finishes

By the time sync code ends:
- 0ms timer is ready
- 1ms timer is ready
- 100ms timer is ready
- 300ms timer is also ready

# Timer Phase Starts

Node enters timer phase and executes all ready callbacks.

Approx output:

```
Timer callback 0 ms delayed by ~380ms
Timer callback 1 ms delayed by ~379ms
Timer callback 100 ms delayed by ~280ms
Timer callback 300 ms delayed by ~80ms
```

# Key Learning

## `setTimeout(fn, 0)` does NOT mean:

```
Run immediately
```

It means:

```
Run after current synchronous code finishes AND when timer phase gets a chance
```

# Visual Timeline
```
T0
│
├── Initial phase starts
│
├── Timers scheduled
│
├── Long synchronous loop blocks thread (~380ms)
│
└── Initial phase ends

T380
│
├── Event loop enters timer phase
│
├── 0ms timer executes
├── 1ms timer executes
├── 100ms timer executes
└── 300ms timer executes
```

# Summary

## Timer Phase

- First phase of the event loop
- Managed by Libuv
- Executes timer callbacks

## Timers

- Stored with target end times
- Sorted by duration
- Not perfectly accurate

## Delays Happen Because

- synchronous code blocks thread
- other phases are running
- callbacks take time

# Core Rule

```
Timers specify MINIMUM delay,
NOT exact execution time.
```


# Pending Callbacks Phase
---
After the timers phase, the event loop moves to the **pending callbacks phase**. This phase executes I/O callbacks that were **deferred from the previous loop iteration** — they weren't ready to run when they were generated, so Node.js queued them up to be handled here, one tick later.

This phase executes certain system-level callbacks that were deferred from the previous loop iteration.

Mainly:
- TCP errors
- UDP errors
- Some network-related system callbacks

# Why does this phase exist?

Not every callback needs to run immediately.

Some operations are:

- low priority
- temporary failures
- recoverable after retry

Instead of interrupting important work, Node.js defers them to the **Pending Callbacks phase**.

This helps:

- keep the event loop responsive
- prioritize important callbacks
- avoid unnecessary immediate retries

# Important Idea

Pending callbacks are usually:

- network-related
- deferred system callbacks
- callbacks that failed during async operations

Especially:

- `ECONNREFUSED`
- TCP connection failures
- certain socket errors

# Internal Flow

Suppose a TCP connection fails.

Instead of immediately executing:

```js
client.on('error', ...)
```

Node/libuv may queue it into:

```
Pending Callbacks Queue
```

Then on the **next loop iteration**, Node executes those callbacks during the Pending Callbacks phase.

### Example
```js
const net = require('net');

console.log("START");

const clientFail = new net.Socket();

clientFail.connect(9999, '192.168.4.21', () => {
    console.log("Connected");
});

clientFail.on('error', (err) => {
    console.log("Connection error:", err.message);
});

for (let i = 0; i < 500000000; i++);

setTimeout(() => console.log("timer!"), 0);

console.log("END");
```

Possible output:
```
START
END
timer!
Connection error: connect ECONNREFUSED 192.168.4.21:9999
```

### What happened internally?
#### 1. Initial execution phase runs

```js
console.log("START")
```

#### 2. TCP connection starts asynchronously

```js
clientFail.connect(...)
```

Node asks the OS:

```
"Try connecting to this server"
```

---

#### 3. Heavy synchronous loop blocks the thread

```js
for (...)
```

Event loop cannot start yet.

---

#### 4. Connection fails in background

OS detects:

```
ECONNREFUSED
```

But Node does NOT immediately run the callback.

Instead:

```
Error callback gets queued into Pending Callbacks phase
```

---

#### 5. Main script finishes

```js
console.log("END")
```

Now event loop begins.

---

#### 6. Timers phase runs first

```js
setTimeout(..., 0)
```

Output:

```
timer!
```

---

#### 7. Pending Callbacks phase runs

Node executes deferred TCP error callback:

```js
clientFail.on('error', ...)
```

Output:

```
Connection error: connect ECONNREFUSED ...
```

---

#### Successful TCP Connection Example

```js
clientSuccess.connect(80, '93.184.215.14', ...)
```

If connection succeeds:

- no TCP error exists
- nothing enters Pending Callbacks queue
- success callback usually executes later during Poll phase

Possible output:

```
START
END
timer!
Connected to server at 93.184.215.14:80
```

# Special Exception — Loopback Address

Loopback:

```
127.0.0.1
```

behaves differently.

Example:

```js
clientFail.connect(9999, '127.0.0.1')
```

If nothing is listening on that port:

```
ECONNREFUSED
```

often happens immediately.

Why?

Because:

- loopback never leaves your machine
- OS instantly knows the port is closed
- no real network traversal is needed

So the error may appear immediately instead of being deferred.

### Difference Between Normal IP vs Loopback

| Type        | Behavior                                   |
| ----------- | ------------------------------------------ |
| External IP | Error may be deferred to Pending Callbacks |
| 127.0.0.1   | Error often detected immediately           |

### Role of libuv

libuv:
- manages socket operations
- receives OS-level network errors
- decides whether callback executes immediately or later
- queues deferred callbacks into Pending Callbacks phase

### Important Interview Point

Pending callbacks are:
- NOT regular timers
- NOT I/O callbacks
- mostly deferred system/network callbacks

This phase mainly exists for:
- TCP socket errors
- UDP socket errors
- deferred networking operations

## Simple One-Line Definition

> Pending Callbacks phase executes deferred system-level callbacks, mainly TCP/UDP networking errors from previous loop iterations.


# Idle & Prepare Phase
---
After the **Pending Callbacks phase**, Node.js enters:
```
Idle → Prepare
```

This is often called the **invisible phase** — it's an internal, bookkeeping phase that Node.js and libuv use for their own housekeeping. As a JavaScript developer, you will **never directly interact with this phase**. It exists entirely below the JS layer.

It's technically two sub-phases bundled together, both mapped from libuv internals into Node's event loop.

#### Why "Invisible"?

- Not exposed to JavaScript modules at all.
- You cannot hook into it via any Node.js API (`setTimeout`, `setImmediate`, `process.nextTick` — none of those land here).
- The only way to interact with it is through **C++ Node addons** (native extensions), which is deep internals territory.
# Important Idea

This phase exists mainly for:

- internal housekeeping
- preparing for I/O
- setting up polling structures
- preparing the event loop before Poll phase

#### Idle Sub-phase

Despite the name, "idle" doesn't mean the loop is doing nothing — it means this runs when the loop has a free moment, but **it still executes on every single iteration** regardless.

Used for **internal tasks** that Node needs to do periodically — things like internal state maintenance that libuv needs kept up to date as the loop runs.

Think of it like a background thread doing quiet work every tick, invisible to your JS code.

# What happens here?

Mostly:

- internal maintenance
- bookkeeping
- cleanup
- state synchronization

Not user callbacks.

---
#### Prepare Sub-phase

Also runs on **every iteration**, specifically **right before the poll phase**.
Its job:
```
"Prepare the system for incoming I/O"
```

# What happens here?

Examples:

- registering sockets
- updating epoll/kqueue/select structures
- preparing file descriptors
- syncing internal watcher lists

# Important Linux Concept — epoll
On Linux, Node/libuv commonly uses:  

#epoll

`epoll` allows the OS to efficiently monitor:

- sockets
- file descriptors
- network events

# Example Idea

Suppose:

```js
server.listen(3000)
```

Node creates:
- socket
- file descriptor

Before Poll phase can wait for incoming connections:

```
that socket must be registered with epoll
```

This preparation often happens during:

```
Prepare phase
```

# What is epoll_ctl?

`epoll_ctl` is used to:

- add file descriptors to epoll
- remove them
- modify them

Basically:

```
"Tell Linux what sockets/files we want to monitor"
```

# Why is this relevant?

Because before Poll phase begins:

```
libuv must prepare what the OS should watch
```

That’s exactly why Prepare phase exists.

```js
//we run this code with this command

//strace -e trace=epoll_ctl -f -o test.txt node 04-idle-prepare/040-idleprepare.js
//this will run an strace (linux) and will listen to epoll_ctl system calls
//these are specifcally called to setup new file desctriptors to prepare 
//for the poll phase 

const net = require('net');

// Create a server object
const server = net.createServer((socket) => {
    console.log('Client connected');

    // Handle incoming data from the client
    socket.on('data', (data) => {
        console.log(`Received data: ${data}`);
        socket.write('Echo: ' + data); // Echo back the data
    });

    // Handle client disconnection
    socket.on('end', () => {
        console.log('Client disconnected');
    });

    // Handle errors
    socket.on('error', (err) => {
        console.error('Socket error:', err);
    });
});

// Server listens on port 3000
server.listen(3000 ,'0.0.0.0',  () => {
    console.log('Server listening on port 3000');
});

// Handle server errors
server.on('error', (err) => {
    console.error('Server error:', err);
});

setTimeout( ()=> server.close(), 30000); 

function test(){
console.log(Date.now() + "test");
	setTimeout(test, 4000);
}

//test();
console.log("start")
//for (let i = 0; i < 10000000000; i++);
console.log("end")
```

# Internal Flow

## 1. Main script executes

```
console.log("start")console.log("end")
```

---

## 2. Server socket created

```
server.listen(...)
```

Node creates:

- socket FD
- listening socket

---

## 3. Idle/Prepare phase runs

libuv prepares:

- epoll watcher
- socket registration
- polling structures

Internally something similar happens:

```
epoll_ctl(ADD, server_socket_fd)
```

meaning:

```
"Linux, notify me when this socket receives connections"
```

# Then Poll Phase Can Work

After Prepare phase finishes:

Poll phase can now safely do:

```
Wait for incoming connections/events
```

because epoll is fully configured.

---

# Why “Follows a Wait”?

Prepare phase comes immediately before:

```
Poll waiting
```

Meaning:

1. prepare all I/O structures
2. THEN block/wait for events

---

# Important Observation

Prepare phase itself:

```
does NOT execute your JavaScript callbacks
```

It only sets up infrastructure.

# Why This Matters

This phase is one of the reasons Node.js scales well.

Instead of constantly checking sockets manually:

- Node delegates monitoring to OS
- epoll efficiently tracks events
- Prepare phase updates those watchers

# Idle vs Prepare

|Idle|Prepare|
|---|---|
|Internal housekeeping|Setup before Poll|
|Runs every iteration|Runs before Poll|
|Uses spare loop time|Prepares I/O structures|
|Mostly maintenance|Mostly epoll/socket prep|

# Mental Model

Think of it like:

```
Idle Phase
	→ "Do internal cleanup/maintenance"

Prepare Phase
	→ "Setup everything needed before waiting for I/O"
```

# Key Takeaways

## Idle Phase

- internal libuv housekeeping
- runs every loop iteration
- not exposed to JavaScript

---

## Prepare Phase

- runs before Poll
- prepares sockets/file descriptors
- updates epoll watchers
- prepares OS for async I/O

---
## epoll_ctl

Used by Linux to:
- add/remove/update watched file descriptors

Node/libuv uses it heavily before Poll phase.


### Simple One-Line Definition

> Idle/Prepare phase is an internal libuv phase where Node performs housekeeping and prepares OS-level I/O monitoring structures before entering the Poll phase.


