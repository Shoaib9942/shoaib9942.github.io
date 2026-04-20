---
title: Threads, Async, and I/O - What Actually Happens Under the Hood
date: 2026-04-07 00:00:00 +/-0000
categories: [Concurrency, Distributed Systems]
tags: [python, fastapi, async, threads]
---

> A ground-up explanation of how modern web frameworks handle concurrency — using FastAPI, asyncio, and Go as the lens.

---

## The Problem: Waiting is Expensive

Every web server spends most of its life *waiting* — waiting for a database to respond, waiting for a file to be read off disk, waiting for an external API to reply. The question that defines a framework's concurrency model is simple:

> **What does your thread do while it waits?**

The answer to that question separates blocking servers from non-blocking ones, threads from coroutines, and Python asyncio from Go goroutines.

---

## 1. Processes and Threads — The Foundation

Before diving into async, it's worth being precise about the primitives underneath everything.

### Processes

A **process** is an independent OS-level execution unit. It has its own memory space, its own file descriptors, and its own Python interpreter (with its own GIL). When you run:

```bash
uvicorn main:app --workers 4
```

You spawn 4 completely isolated Python processes. They share no memory. Each one independently handles incoming requests. This is how you use multiple CPU cores with Python — not threads, but processes.

### Threads

A **thread** lives inside a process and shares its memory. Multiple threads in the same process can read and write the same data. In theory, they can also run in parallel — but Python's **GIL (Global Interpreter Lock)** prevents this. Only one thread runs Python bytecode at any given moment.

This means Python threads are useful for **I/O-bound concurrency** (where threads spend most time waiting, not computing), but not for **CPU-bound parallelism** (where you need two threads genuinely running code simultaneously on two cores).

---

## 2. The Event Loop — One Thread, Many Tasks

FastAPI is built on **Starlette** and served by **Uvicorn**, an ASGI server. At its core is Python's `asyncio` event loop — a single-threaded scheduler that runs many coroutines concurrently.

The full stack looks like this:

```
Your FastAPI route handler
         ↓
    Starlette (ASGI framework)
         ↓
    Uvicorn (ASGI server)
         ↓
    asyncio event loop
         ↓
    OS (epoll / kqueue / IOCP)
```

The event loop runs on the **main thread** of each worker process. It doesn't spin up new threads for each request. Instead, it runs one tight loop, constantly asking the OS: *"Which of my pending I/O operations are ready?"*

```python
# Simplified pseudocode of asyncio's event loop
while True:
    ready_events = epoll.poll(timeout)   # ask the OS

    for event in ready_events:
        coroutine = waiting_on[event]
        coroutine.send(result)           # resume the paused coroutine
```

This loop is the engine. Everything else — your route handlers, middleware, database calls — runs as tasks inside it.

---

## 3. `async def` vs `def` — Where Does Your Code Run?

FastAPI makes a critical distinction based on how you define your route handler.

### `async def` — Runs on the event loop (main thread)

```python
@app.get("/users/{id}")
async def get_user(id: int):
    user = await db.fetch_one("SELECT * FROM users WHERE id = $1", id)
    return user
```

This runs directly on the main thread, inside the event loop. No thread switching occurs. The handler is a **coroutine** — a suspendable function that can pause at `await` points and let other coroutines run.

### `def` — Offloaded to a thread pool

```python
@app.get("/report")
def generate_report():
    time.sleep(5)   # blocking call
    return {"status": "done"}
```

FastAPI detects that this is a regular synchronous function and automatically offloads it to a **thread pool** via `asyncio.run_in_executor`. This prevents it from blocking the event loop. The main thread remains free to handle other requests while this runs on a worker thread.

The decision tree is:

```
Route defined with async def?
        │
       YES ──► Runs on event loop (main thread)
        │
        NO ──► Offloaded to thread pool automatically
```

---

## 4. What `await` Actually Does

This is where most confusion lives. `await` does **not** trigger the event loop. The event loop was already running — it's what called your handler in the first place.

`await` is a **cooperative yield point**. It says:

> *"I'm waiting for something. Go do other work. Come back when this is ready."*

Here's the sequence when your handler hits `await db.fetch_one(...)`:

```
1. Handler calls db.fetch_one()
2. asyncpg writes query bytes to a socket
3. `await` suspends the coroutine — saves its stack frame, local variables, resume point
4. Control returns to the event loop
5. Event loop picks up another pending request and runs it
6. Meanwhile: OS kernel manages TCP, sends packets to DB server
7. DB server executes the query on its own CPU
8. Response arrives → kernel puts bytes in socket buffer → notifies epoll
9. Event loop resumes the coroutine exactly where it left off
10. Handler reads the result, returns response
```

Your process's CPU is only active in steps 1–2 and 9–10. Everything in between — the network, the database, the TCP stack — happens **without your process doing anything**.

### The suspended coroutine is not running anywhere

This is the key insight that async programmers need to internalize:

> A suspended coroutine is **just a Python object in RAM**. It is not executing on any thread. It is not being polled. It is paused, frozen, waiting. No CPU. No thread. Just memory.

```
await db.fetch_one(...)
         │
         ▼
Coroutine state saved:
{
  local variables: { id: 42, ... },
  resume at: line 3,
  waiting for: socket fd 7
}
         │
         ▼
      IDLE. Sitting in a dictionary.
      Zero CPU cost.
```

The event loop maintains a mapping of `{fd → coroutine}`. When the OS signals that fd 7 has data, the event loop looks up this dictionary and resumes the coroutine. That's the entire mechanism.

---

## 5. I/O Operations Don't Need Your CPU

This is the conceptual leap that makes async make sense.

When you query a database, you're not doing computation — you're **waiting for bytes to arrive over a network**. The actual work is happening elsewhere:

```
Your Process         OS Kernel              DB Server
────────────────────────────────────────────────────────

Writes query →
Suspends (await)  →  Manages TCP stack  →  Executes SQL on its CPU
                     Sends packets
                  ←  Receives response  ←  Returns results
Resumes ←
Reads bytes
```

Between suspending and resuming, your process CPU involvement is **zero**. The kernel handles all the network mechanics. The database server does all the query work.

There are exactly two types of work a program does:

| Type | Needs Your CPU? | Async helps? |
|---|---|---|
| **I/O-bound** (network, disk, DB) | No — you're just waiting | ✅ Yes — suspend and do other work |
| **CPU-bound** (parsing, sorting, ML) | Yes — you're computing | ❌ No — need real threads or processes |

Async is the right tool when your bottleneck is *waiting*, not *computing*.

---

## 6. The Cost of Getting It Wrong

The event loop runs on a single thread. If anything blocks that thread, **all requests freeze**.

### Mistake 1: Calling a sync library inside `async def`

```python
# ❌ DANGEROUS — blocks the event loop for 2 seconds
async def bad_handler():
    time.sleep(2)    # blocks the main thread entirely
    return {"status": "done"}

# ✅ CORRECT — yields control for 2 seconds
async def good_handler():
    await asyncio.sleep(2)   # suspends, event loop runs other requests
    return {"status": "done"}
```

`time.sleep()` doesn't return a coroutine — it blocks the OS thread synchronously. Inside `async def`, this poisons the event loop.

### Mistake 2: Using a sync database driver in an async handler

```python
# ❌ DANGEROUS — psycopg2 is a blocking driver
async def get_user(id: int):
    conn = psycopg2.connect(...)
    result = conn.execute("SELECT ...")   # blocks the event loop!
    return result

# ✅ CORRECT — asyncpg is a non-blocking async driver
async def get_user(id: int):
    user = await db.fetch_one("SELECT ...")   # suspends, loop stays free
    return user
```

The async-native version suspends while waiting for the DB. The sync version freezes the entire server.

### Mistake 3: CPU-heavy work on the event loop

```python
# ❌ DANGEROUS — CPU work burns the main thread
async def classify_image(data: bytes):
    result = run_ml_model(data)   # takes 500ms of CPU — all requests blocked
    return result

# ✅ CORRECT — offload to a process pool
async def classify_image(data: bytes):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(process_pool, run_ml_model, data)
    return result
```

CPU-bound work needs genuine parallelism — a separate process or thread. Wrapping it in `async def` doesn't help if you don't yield.

---

## 7. Why Threads Don't Scale for I/O

Before async, the standard approach was thread-per-request. Each incoming connection got its own OS thread, which blocked while waiting for I/O.

```
Thread 1 → waiting on DB    (blocked, idle, ~1MB stack memory)
Thread 2 → waiting on DB    (blocked, idle, ~1MB stack memory)
Thread 3 → waiting on API   (blocked, idle, ~1MB stack memory)
...
Thread 10,000 → blocked     (10GB of RAM for idle stacks)
```

These threads are doing no useful work. They're just sitting in memory, with the OS scheduler cycling through them unnecessarily.

Async replaces 10,000 idle threads with 10,000 **paused Python objects** (~a few KB each):

```
Event loop (1 thread)
├── coroutine A — paused, waiting for DB (few KB)
├── coroutine B — paused, waiting for API (few KB)
├── coroutine C — currently executing
└── coroutine D — paused, waiting for disk (few KB)
```

Same concurrency. A fraction of the memory. No OS thread scheduling overhead.

---

## 8. Go Goroutines — The Same Idea, Done Better

Go uses the same fundamental insight — don't block threads on I/O — but its implementation is more powerful.

### Python asyncio is cooperative and single-threaded

```
Event Loop (1 thread)
│
├── coroutine A  →  hits await  →  yields voluntarily
├── coroutine B  →  hits await  →  yields voluntarily
└── coroutine C  →  hits await  →  yields voluntarily

Rule: YOU must write `await` to yield control.
      If you forget, the event loop is blocked.
```

### Go uses M:N scheduling across real threads

```
Go Runtime Scheduler
│
├── OS Thread 1  →  goroutine A
├── OS Thread 2  →  goroutine B
├── OS Thread 3  →  goroutine C
└── OS Thread 4  →  goroutine D

Rule: The runtime moves goroutines automatically.
      Write normal blocking-style code.
```

Go maps **M goroutines** onto **N OS threads** (where N typically equals the number of CPU cores). When a goroutine blocks on I/O, the runtime parks it and moves the OS thread to another runnable goroutine — automatically, without the programmer doing anything.

### The practical difference

In Python, you must use async-native libraries everywhere:

```python
# ❌ Breaks the async model — use of sync library
async def handler():
    resp = requests.get(url)   # blocks the event loop

# ✅ Must explicitly use async library
async def handler():
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
```

In Go, you just write normal code:

```go
// ✅ Looks synchronous — Go runtime handles the rest
func handler() {
    resp, err := http.Get(url)  // goroutine is suspended by runtime if needed
    // ...
}
```

### Go also handles CPU-bound work naturally

| | Python asyncio | Go goroutines |
|---|---|---|
| Scheduling model | Cooperative (requires `await`) | Preemptive (runtime decides) |
| Threads | 1 event loop thread per worker | M goroutines on N OS threads |
| I/O blocking | Must use async libraries | Normal code works |
| CPU-bound work | Blocks event loop (need ProcessPool) | Runs in parallel across cores |
| Goroutine/coroutine size | ~few KB (heap frames) | ~2KB initial stack, grows dynamically |
| Underlying I/O mechanism | epoll via asyncio | netpoller (epoll/kqueue) |

Go goroutines solve the same I/O problem as asyncio but without requiring you to restructure your entire codebase. And because goroutines spread across real OS threads, CPU-heavy work runs in parallel automatically — something Python cannot do due to the GIL.

---

<p align="center">
  <i>Thanks for reading! If you found this useful or have corrections/additions, feel free to reach out or drop a comment.</i>
</p>
