# Multithreading in Android with Java — Cheat Sheet

## Threading Layers

```
Java Primitives        →  Thread, Runnable, Callable, Future, synchronized, volatile
Java Concurrency Utils →  Locks, Atomics, Executors, BlockingQueue
Android OS Layer       →  Looper, MessageQueue, Handler, HandlerThread, Choreographer
High-level             →  WorkManager, RxJava, Kotlin Coroutines
```

---

## Thread Fundamentals

### `Thread` Lifecycle

- **NEW** → `start()` → **RUNNABLE** → `run()` returns → **TERMINATED**
- **BLOCKED**: waiting to acquire a `synchronized` monitor lock — not voluntary.
- **WAITING**: voluntarily paused via `Object.wait()`, `Thread.join()`, `LockSupport.park()` — needs explicit wake.
- **TIMED_WAITING**: same as WAITING but with timeout — `Thread.sleep(n)`, `wait(n)`, `join(n)`.
- **BLOCKED ≠ WAITING**: BLOCKED = competing for a lock. WAITING = voluntarily released CPU.

### Runnable vs Callable vs Future

- **`Runnable`**: `void run()` — no return, no checked exceptions.
- **`Callable<V>`**: `V call() throws Exception` — returns result, can throw.
- **`Future<V>`**: handle to async result. `get()` **blocks**. Never call on Main Thread.

```java
Future<Bitmap> f = executor.submit(() -> BitmapFactory.decodeStream(url.openStream()));
Bitmap bmp = f.get(5, TimeUnit.SECONDS); // blocks — background thread only!
```

- `future.cancel(true)` → sends interrupt to running thread (cooperative, not forcible).
- `t.start()` spawns a new OS thread. `t.run()` runs on the **current** thread — common mistake.

---

## Handler / Looper / HandlerThread

### `Looper`

- One `Looper` per thread. Main Thread has one created automatically by `ActivityThread`.
- `Looper.prepare()` → creates Looper for current thread. `Looper.loop()` → **blocks forever** processing messages. Parked via OS **epoll** (zero CPU when idle).
- `looper.quit()` — discards ALL pending messages. `looper.quitSafely()` — processes past-due messages first.

| Method | Description |
|---|---|
| `Looper.prepare()` | Create Looper for current thread |
| `Looper.loop()` | Start message loop (blocks) |
| `Looper.myLooper()` | This thread's Looper (null if none) |
| `Looper.getMainLooper()` | Main Thread Looper — safe from any thread |

### `Handler`

- Bound permanently to one `Looper` at construction. Posts **Runnables** or **Messages** to that thread.

```java
Handler main = new Handler(Looper.getMainLooper());
main.post(() -> textView.setText(result));       // from any background thread
main.postDelayed(() -> animate(), 300);
```

- **Memory leak**: anonymous `Handler`/`Runnable` holds implicit reference to enclosing Activity.
- Fix: `handler.removeCallbacksAndMessages(null)` in `onDestroy()`.

### `HandlerThread`

- `Thread` subclass with built-in `Looper` setup. Ideal for serialized background tasks. Used internally by Android for `Camera` callbacks and `AsyncQueryHandler`. `IntentService` also used `HandlerThread` internally but was **deprecated in API 30** — migrate to `WorkManager`.

```java
HandlerThread ht = new HandlerThread("bg", Process.THREAD_PRIORITY_BACKGROUND);
ht.start();
Handler bgHandler = new Handler(ht.getLooper());
// cleanup:
ht.quitSafely();
```

### `Message`

- **Always use `Message.obtain()`** (pool of 50) — avoids GC pressure.
- `msg.what` (int id), `msg.arg1`, `msg.arg2`, `msg.obj`. `msg.callback` = Runnable if posted via `post()`.

### MessageQueue Internals

- Sorted **singly-linked list** by `when` (`SystemClock.uptimeMillis()` delivery timestamp).
- `enqueueMessage()` is `synchronized(this)` — any thread can enqueue safely.
- `next()` (Looper thread only) calls `nativePollOnce()` — sleeps via epoll until message is due.
- Looper does NOT hold Java lock while sleeping; background threads enqueue without blocking.

### Dispatch Priority in `Handler.dispatchMessage()`

1. `msg.callback != null` → `runnable.run()` (posted via `handler.post()`)
2. `handler.mCallback != null` → `mCallback.handleMessage()` (interception point)
3. `handler.handleMessage()` (subclass override)

### `ActivityThread` and Lifecycle Delivery

- `ActivityThread.main()` is the real app entry point (not `Application.onCreate()`).
- System Server sends IPC → `ApplicationThread` (Binder stub) → posts `EXECUTE_TRANSACTION` message → `ActivityThread.H.handleMessage()` on Main Thread → `Activity.onCreate()` etc.

---

## `volatile` and `synchronized`

### `volatile`

- Guarantees **visibility** across threads via memory barriers (cache coherence).
- Prevents compiler/CPU reordering around the access.
- **Does NOT guarantee atomicity** — `counter++` is still a 3-step read-modify-write.
- Use only for simple assignments where writes don't depend on current value.

```java
private volatile boolean running = true; // visible to all threads immediately
```

### Double-Checked Locking — requires `volatile`

```java
private static volatile Singleton instance; // volatile is mandatory
public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) instance = new Singleton();
        }
    }
    return instance;
}
```

### `synchronized`

- **Mutual exclusion** + **visibility** (changes made inside are visible to next thread that acquires the same lock).
- Prefer `synchronized (privateLock)` over `synchronized (this)` — prevents external locking on the same object.
- `synchronized (new Object())` is wrong — new instance every time, no mutual exclusion.
- `synchronized ("string")` is wrong — string literals may be shared unexpectedly.

---

## Atomic Classes

- **CAS (Compare-And-Swap)**: hardware-level atomic — reads and conditionally writes in one instruction. Optimistic, retries on contention.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();               // atomic ++counter
counter.compareAndSet(expected, update); // atomic conditional write
```

- **`AtomicBoolean`**: use `compareAndSet(false, true)` for safe one-time start/init.
- **`AtomicReference<V>`**: CAS on object references.
- **ABA Problem**: value A→B→A fools CAS. Fix: **`AtomicStampedReference<V>`** (adds integer stamp).
- **`LongAdder`**: higher throughput than `AtomicLong` under heavy contention (striped counters).

| Scenario | Use |
|---|---|
| Simple counter / flag | `AtomicInteger` / `AtomicBoolean` |
| Multi-variable invariant | `synchronized` |
| High-contention counter | `LongAdder` |
| Complex state transition | `ReentrantLock` |

---

## Thread Interruption

- `thread.interrupt()` sets the **interrupt flag** — cooperative signal, does NOT forcibly stop.
- `isInterrupted()` reads flag without clearing. `Thread.interrupted()` reads AND clears.
- Blocking methods (`sleep`, `wait`, `join`, `BlockingQueue.take`) throw **`InterruptedException`** and **clear the flag** when interrupted.

```java
while (!Thread.currentThread().isInterrupted()) { doWork(); }
```

**Handling `InterruptedException`:**
- ❌ Never swallow silently — the interrupt is lost.
- ✅ Re-interrupt: `Thread.currentThread().interrupt()` (restores the flag).
- ✅ Propagate: declare `throws InterruptedException`.

### Why You Cannot Cancel a Running Thread Directly

- `Thread.stop()` — **deprecated**, dangerous. Throws `ThreadDeath` at an arbitrary point, releases monitor locks with shared data in inconsistent state.
- `Thread.suspend()` — **deprecated**, deadlock risk (holds locks without releasing).
- **Correct approach**: cooperative cancellation via interrupt flag or `AtomicBoolean cancelled`.

---

## The Android Main (UI) Thread

- Handles: Activity/Fragment lifecycle, View measure/layout/draw, touch events, BroadcastReceiver, Service callbacks, Handler messages, Choreographer frame callbacks.
- **Frame budget**: 60 fps = **16 ms**, 90 fps = **11 ms**, 120 fps = **8 ms**.
- **ANR thresholds**: input unresponsive **5s** | `BroadcastReceiver.onReceive()` foreground **10s** (Android 13-), background **60s** (Android 13-) — Android 14+ dynamically scales to **10–20s**/**60–120s** based on CPU starvation | foreground Service **20s** | background Service **200s** | `ContentProvider` — **no fixed default timeout** (caller-specified via `ContentProviderClient.setDetectNotResponding()`).
- **Never on Main Thread**: network I/O (`NetworkOnMainThreadException` since API 11), disk I/O, `Thread.sleep()`, `future.get()`.

```java
// StrictMode for DEBUG builds
StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
    .detectDiskReads().detectDiskWrites().detectNetwork()
    .penaltyLog().build());
```

---

## VSYNC and the Display Pipeline

- **VSYNC**: hardware signal at display refresh rate. Prevents screen tearing by synchronizing frame production with display scan.
- Pipeline: `View.invalidate()` → `ViewRootImpl.scheduleTraversals()` → posts **sync barrier** + registers Choreographer callback → VSYNC fires → async frame message dispatched (skips sync barrier) → `performTraversals()` (measure/layout/draw) → barrier removed.
- **Sync Barrier**: `Message` with `target == null`. `MessageQueue.next()` skips all sync messages, dispatches only **async** messages. Ensures frame work jumps ahead of app messages.
- `msg.setAsynchronous(true)` marks a message to bypass sync barriers (API 22+).
- **`Choreographer`**: receives VSYNC signals, schedules frame callbacks. Used internally by `ValueAnimator`, `ObjectAnimator`, `RecyclerView`.
- **Jank**: frame not ready when SurfaceFlinger composites → previous frame shown again → dropped frame. Logged: `"Skipped N frames!"`.
- **Main Thread priority**: `THREAD_PRIORITY_DEFAULT` (nice 0) — NOT elevated. RenderThread runs at `THREAD_PRIORITY_DISPLAY` (nice -4). Frame priority comes from sync barrier, not OS priority.

---

## Inter-Thread Communication

- **`Handler.post()`**: most idiomatic Android pattern. `mainHandler.post(() -> ui())` from background.
- **`Activity.runOnUiThread()`**: posts to Main Thread, or runs immediately if already on Main Thread.
- **`View.post()`**: posts to View's attached Handler (Main Thread). Also useful to defer until after layout.
- **`CountDownLatch`**: one-shot. `countDown()` N times → `await()` unblocks. Not reusable.
- **`CyclicBarrier`**: reusable. N threads call `await()` → all proceed together. Optional barrier action.
- **`Semaphore`**: controls access to a resource pool. `acquire()` blocks if no permits; `release()` in `finally`.
- **`Exchanger<V>`**: two threads swap objects at a synchronization point.

---

## `BlockingQueue`

- Thread-safe queue with **blocking put** (when full) and **blocking take** (when empty). Backbone of producer-consumer pattern.
- **Always prefer bounded queues on Android.** Unbounded queues risk OOM silently.

| Method flavor | Throws | Returns value | Blocks | Timed block |
|---|---|---|---|---|
| Insert | `add(e)` | `offer(e)` | `put(e)` | `offer(e,t,u)` |
| Remove | `remove()` | `poll()` | `take()` | `poll(t,u)` |

```java
// bounded producer-consumer
BlockingQueue<Bitmap> q = new LinkedBlockingQueue<>(10);
// producer: q.put(frame);   // blocks when full — natural backpressure
// consumer: q.take();       // blocks when empty
```

**Never call `put()`/`take()` on the Main Thread.**

### Implementations

| Class | Bounded | Lock | Best For |
|---|---|---|---|
| `LinkedBlockingQueue` | Optional | 2 locks (head/tail — high throughput) | General-purpose, thread pools |
| `ArrayBlockingQueue` | Always | 1 lock (lower memory, worse throughput) | Memory-sensitive |
| `SynchronousQueue` | No buffer | — (direct hand-off) | `CachedThreadPool`, immediate dispatch |
| `PriorityBlockingQueue` | No | — | Priority task scheduling |
| `DelayQueue` | No | — | Scheduled/delayed tasks |
| `LinkedTransferQueue` | No | Non-blocking (highest throughput) | High-throughput pipelines |

---

## Locks and Synchronization Mechanisms

| Lock | Reentrant | Interruptible | Timed | Conditions | Fairness |
|---|---|---|---|---|---|
| `synchronized` | ✅ | ❌ | ❌ | ❌ (`wait`/`notify`) | ❌ |
| `ReentrantLock` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ReentrantReadWriteLock` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `StampedLock` | ❌ | ✅ | ✅ | ❌ | ❌ |

### `ReentrantLock`

```java
lock.lock();
try { count++; } finally { lock.unlock(); } // ALWAYS unlock in finally
lock.tryLock(100, TimeUnit.MILLISECONDS);   // timed acquisition
lock.lockInterruptibly();                   // cancellable acquisition
```

- **`Condition`**: replaces `wait()`/`notify()`. Multiple conditions per lock. `await()` releases lock; `signal()` wakes one waiter.

### `ReentrantReadWriteLock`

- Many concurrent readers OR one exclusive writer. Read-heavy caches/config maps.
- Lock downgrading (write → read) supported. Upgrading (read → write) is NOT.
- Risk: writer starvation under constant reads.

### `StampedLock` (API 24+ natively)

- Three modes: **write** (exclusive), **read** (shared), **optimistic read** (no lock — fastest).
- Optimistic read: `tryOptimisticRead()` returns stamp; read fields; `validate(stamp)` — if false, fall back to real read lock.
- **NOT reentrant** — acquiring write lock while holding write lock = deadlock.

### Deadlock Prevention

- **Lock ordering**: always acquire multiple locks in the same global order.
- **`tryLock(timeout)`**: back off and retry on failure.
- **Lock-free design**: use `Atomic` classes or `ConcurrentHashMap`.

---

## Executors and `ThreadPoolExecutor`

### Task Submission Flow

```
execute(task)
  → pool < corePoolSize?   YES → new core thread (even if idle threads exist)
  → queue.offer(task)?     YES → enqueue
  → pool < maxPoolSize?    YES → new non-core thread
  → RejectedExecutionHandler
```

### `RejectedExecutionHandler` Policies

| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | throws `RejectedExecutionException` |
| `CallerRunsPolicy` | **calling thread runs the task** — natural backpressure |
| `DiscardPolicy` | silently drops task |
| `DiscardOldestPolicy` | drops oldest queued task, retries |

### `Executors` Factory Pitfalls

- `newFixedThreadPool(n)` — **unbounded** `LinkedBlockingQueue` → OOM risk.
- `newCachedThreadPool()` — **unbounded** thread count → thousands of threads under load.
- **Prefer `ThreadPoolExecutor` directly** with explicit bounded parameters.

---

## CPU-Bound Thread Pool

- CPU-bound threads are **always runnable** — more threads than cores = context-switch overhead.
- `corePoolSize = maximumPoolSize = Runtime.getRuntime().availableProcessors()` (or `+1`).
- Small bounded queue: `new LinkedBlockingQueue<>(cpuCount * 2)`.
- Use `CallerRunsPolicy` for backpressure.
- Set `Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)` — yields to Main/RenderThread.

### Android Thread Priority (Linux nice values)

| Constant | Nice | Used By |
|---|---|---|
| `THREAD_PRIORITY_DISPLAY` | -4 | RenderThread |
| `THREAD_PRIORITY_DEFAULT` | 0 | Main Thread, normal threads |
| `THREAD_PRIORITY_BACKGROUND` | 10 | **Recommended for workers** |

### `ForkJoinPool` (API 21+)

- **Work-stealing**: idle threads steal tasks from busy threads' queues.
- Use `RecursiveAction` / `RecursiveTask` for divide-and-conquer (parallel sort, image processing).

---

## I/O-Bound Thread Pool

- I/O threads spend most time **blocked** — CPU is idle. More threads than cores is productive.
- Formula: `Optimal Threads = N_cpu × (1 + W/C)` where W = wait time, C = compute time.
- Mobile cap: `Math.min(cpuCount * 4, 32)` — each idle thread ~512 KB–1 MB stack.
- `corePoolSize = cpuCount * 2`, `maxPoolSize = cpuCount * 4`, `keepAlive = 60s`.
- Larger queue: `new LinkedBlockingQueue<>(128)`. Enable `allowCoreThreadTimeOut(true)`.

| Parameter | CPU Pool | I/O Pool |
|---|---|---|
| `corePoolSize` | `N_cpu` | `N_cpu × 2` |
| `maximumPoolSize` | `N_cpu` | `N_cpu × 4` (cap 32) |
| `keepAliveTime` | 0 | 30–60s |
| Queue capacity | Small (2×N_cpu) | Medium-large (64–256) |

---

## `ThreadLocal`

- **Per-thread storage**: each thread gets its own independent copy. Reads/writes invisible to other threads.
- Used internally by `Looper`: `static final ThreadLocal<Looper> sThreadLocal` — how `Looper.myLooper()` returns per-thread Looper.
- Common uses: non-thread-safe formatters (`SimpleDateFormat`), per-thread transaction context.
- **Memory leak with thread pools**: value persists in pool thread's `ThreadLocalMap` unless removed.

```java
executor.execute(() -> {
    try { threadLocal.set(obj); doWork(); }
    finally { threadLocal.remove(); } // CRITICAL in thread pools
});
```

- **`InheritableThreadLocal`**: copies value to child thread at creation time only. Does NOT work with thread pools (threads are reused, not created fresh).

---

## `CompletableFuture` (API 24+, or API 21+ with desugaring)

- Non-blocking async pipeline via chaining. `Future.get()` blocks; `CompletableFuture` chains callbacks.

```java
CompletableFuture.supplyAsync(() -> fetchJson(id), ioPool)
    .thenApply(json -> parseUser(json))
    .thenAcceptAsync(user -> {
        new Handler(Looper.getMainLooper()).post(() -> showUser(user));
    }, ioPool)
    .exceptionally(t -> { Log.e("TAG", "Error", t); return null; });
```

| Method | Signature | Notes |
|---|---|---|
| `thenApply(fn)` | `T → U` | same thread as prior stage |
| `thenApplyAsync(fn, exec)` | `T → U` | on given executor |
| `thenCompose(fn)` | `T → CF<U>` | flatMap for futures |
| `thenCombine(cf, fn)` | `(T,U) → V` | when both complete |
| `allOf(...)` | → `CF<Void>` | wait for all |
| `anyOf(...)` | → `CF<Object>` | first to complete wins |
| `exceptionally(fn)` | recovers from exception | |
| `handle(fn)` | `(T, Throwable) → U` | called on success or failure |

- **Always provide a custom executor** — default `ForkJoinPool.commonPool()` has no Android priority management.
- `CompletableFuture` has no Main Thread awareness — manually `post()` to `mainHandler` for UI updates.

---

## Concurrent Collections

| Collection | Thread-Safe | Best For |
|---|---|---|
| `ConcurrentHashMap` | ✅ (fine-grained CAS/lock-striping) | General-purpose thread-safe map |
| `ConcurrentSkipListMap` | ✅ (sorted) | Sorted map with concurrent writes |
| `CopyOnWriteArrayList` | ✅ (copy-on-write) | Read-heavy listener/observer lists |
| `CopyOnWriteArraySet` | ✅ | Read-heavy unique element set |
| `ConcurrentLinkedQueue` | ✅ (lock-free CAS) | Non-blocking thread-safe queue |

- **`ConcurrentHashMap`** over `Collections.synchronizedMap()`: `synchronizedMap` wraps each call but compound operations (`containsKey` + `put`) are still racy. `ConcurrentHashMap.putIfAbsent()` is atomic.
- **`CopyOnWriteArrayList`**: reads are lock-free (snapshot). Writes are O(n) — copies entire array. For frequently modified lists, use `ConcurrentLinkedQueue` or synchronized alternatives.

---

## `ScheduledExecutorService`

```java
ScheduledExecutorService s = Executors.newScheduledThreadPool(2);
s.schedule(task, 5, TimeUnit.SECONDS);               // one-shot after 5s
s.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS); // every 1s from START of previous
s.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS); // 1s after END of previous
```

- `scheduleAtFixedRate`: period = start-to-start. If task exceeds period, next starts immediately (no overlap). Good for polling at regular intervals.
- `scheduleWithFixedDelay`: delay = end-to-start. Frequency varies with task duration. Good for tasks needing breathing room.
- ⚠️ **Uncaught exception silently cancels all future executions.** Always `try/catch` inside scheduled tasks.

| Aspect | `ScheduledExecutorService` | `Handler.postDelayed()` |
|---|---|---|
| Thread | Pool thread | Handler's Looper thread |
| Periodic | Built-in | Manual re-posting |
| Use case | Background periodic work | UI delays, animations |

---

## Virtual Threads (Project Loom)

- **NOT available on Android.** Android uses **ART**, not desktop OpenJDK/HotSpot. Not supported as of Android 16 (API 36).
- JDK 21 feature: ~1 KB stack, millions concurrently, auto-unmounts on I/O blocking — no OS thread wasted.
- Android alternative for lightweight concurrency: **Kotlin Coroutines** (cooperative scheduling at library level).
- Java fallback on Android: `ThreadPoolExecutor` (I/O pool) + `CompletableFuture` + RxJava.
