# Multithreading in Android with Java

## Table of Contents

1. [Introduction](#1-introduction)
2. [Threading Fundamentals in Android with Java](#2-threading-fundamentals-in-android-with-java)
3. [Thread Primitives — `volatile` and `synchronized`](#3-thread-primitives--volatile-and-synchronized)
4. [Atomic Classes](#4-atomic-classes)
5. [Thread Interruption in Java](#5-thread-interruption-in-java)
6. [Why You Cannot Cancel a Running Thread Directly](#6-why-you-cannot-cancel-a-running-thread-directly)
7. [The Android Main (UI) Thread](#7-the-android-main-ui-thread)
8. [The Main Thread Message Queue](#8-the-main-thread-message-queue)
9. [VSYNC and the Display Pipeline](#9-vsync-and-the-display-pipeline)
10. [Inter-Thread Communication](#10-inter-thread-communication)
11. [Locks and Synchronization Mechanisms](#11-locks-and-synchronization-mechanisms)
12. [Executors and `ThreadPoolExecutor`](#12-executors-and-threadpoolexecutor)
13. [Efficient ThreadPool for CPU-Intensive Tasks](#13-efficient-threadpool-for-cpu-intensive-tasks)
14. [Efficient ThreadPool for IO-Intensive Tasks](#14-efficient-threadpool-for-io-intensive-tasks)

---

## 1. Introduction

Android apps run in a single-threaded world by default. Every UI update, every touch event, every `Activity` lifecycle callback executes on a single thread called the **Main Thread** (or UI Thread). This design simplifies UI programming because you never need to worry about concurrent modifications to Views, but it creates a critical constraint:

> **Any operation that blocks the Main Thread for more than ~5 seconds causes an ANR (Application Not Responding) dialog, and the OS may kill your app.**

### Why Multithreading Matters in Android

- **Responsiveness**: Network calls, disk I/O, and heavy computation must be offloaded to background threads so the Main Thread remains free to handle user input and render frames.
- **Performance**: Modern Android devices have 4–12 CPU cores. Without multithreading, you leave most of the hardware idle.
- **Frame budget**: At 60 fps the Main Thread has only **16 ms** per frame. At 120 fps that shrinks to **8 ms**. Any background work that leaks onto the Main Thread causes jank (dropped frames).

### Threading Layers in Android (Java)

```
Java Primitives          →  Thread, Runnable, Callable, Future, synchronized, volatile
Java Concurrency Utils   →  Locks, Atomics, Executors, BlockingQueue
Android OS Layer         →  Looper, MessageQueue, Handler, HandlerThread, Choreographer
High-level Abstractions  →  AsyncTask (deprecated), WorkManager, RxJava, Kotlin Coroutines
```

This document focuses on the **Java primitives** and **Android OS layer** — the foundation that every higher-level abstraction is built on.

---

## 2. Threading Fundamentals in Android with Java

### 2.1 `Thread`

`Thread` is the fundamental unit of concurrency in Java. Each `Thread` has its own **stack** (default ~512 KB on Android), a program counter, and local variables. Threads within the same process share the **heap** (objects, static fields).

#### Thread Lifecycle

```
                           ┌──────────────────────────────────────┐
                           │                                      │
   NEW ──start()──► RUNNABLE ──run() returns / exception──► TERMINATED
                      │  ▲
          ┌───────────┼──┴────────────────────────────┐
          │           │                               │
          │ (waiting  │ (lock acquired)               │ (Object.wait() /
          │  for      │                               │  Thread.sleep() /
          ▼  lock)    │                               │  LockSupport.park() /
        BLOCKED ──────┘                               │  Thread.join())
                                                      ▼
                                              WAITING / TIMED_WAITING
                                                      │
                                                      │ (notify/notifyAll,
                                                      │  timeout, unpark)
                                                      └──────────► RUNNABLE
```

- **NEW**: `Thread` object created but `start()` not yet called.
- **RUNNABLE**: Thread is eligible to run (may or may not be currently executing on a CPU core). Note: from the JVM's perspective, both "ready to run" and "currently executing" are RUNNABLE.
- **BLOCKED**: Waiting **exclusively** to acquire a monitor lock (`synchronized` block/method). Transitions back to RUNNABLE once the lock is acquired.
- **WAITING**: Indefinitely paused via `Object.wait()`, `Thread.join()` (no timeout), or `LockSupport.park()`. Requires an explicit `notify()` / `notifyAll()` / `unpark()` to wake up.
- **TIMED_WAITING**: Same as WAITING but with a timeout — via `Thread.sleep(n)`, `Object.wait(n)`, `Thread.join(n)`, `LockSupport.parkNanos()`. Automatically transitions back to RUNNABLE when the timeout expires.
- **TERMINATED**: `run()` method has returned normally or via an uncaught exception. Cannot be restarted.

> ⚠️ **BLOCKED vs WAITING**: These are different states. BLOCKED means the thread is competing for a `synchronized` monitor lock. WAITING/TIMED_WAITING means the thread voluntarily gave up the CPU (e.g., via `wait()` or `sleep()`) and is not competing for any lock.

#### `start()` vs `run()`

```java
Thread t = new Thread(() -> {
    // executes on a NEW thread
    Log.d("TAG", "Running on: " + Thread.currentThread().getName());
});

t.start(); // Correct: spawns a new OS thread, calls run() on it
t.run();   // WRONG:  calls run() on the CURRENT thread (no new thread created)
```

#### Creating Threads

```java
// Option 1: Subclass Thread
class MyThread extends Thread {
    @Override
    public void run() {
        // background work
    }
}
new MyThread().start();

// Option 2: Implement Runnable (preferred — separates task from thread management)
Thread t = new Thread(new Runnable() {
    @Override
    public void run() {
        // background work
    }
});
t.start();

// Option 3: Lambda (Java 8+)
new Thread(() -> doWork()).start();
```

**Prefer `Runnable` or `Callable` over subclassing `Thread`**. Subclassing ties the task definition to a specific thread management strategy and prevents reuse with Executors.

---

### 2.2 `Runnable`

`Runnable` is a **functional interface** with a single method `void run()`. It represents a task that:
- Takes no arguments
- Returns no result
- Cannot throw checked exceptions

```java
Runnable task = () -> {
    Log.d("TAG", "Task running on: " + Thread.currentThread().getName());
};

// Submit to a Thread
new Thread(task).start();

// Submit to a Handler (posts to a Looper's MessageQueue)
handler.post(task);

// Submit to an ExecutorService
executorService.execute(task);
```

---

### 2.3 `Callable<V>`

`Callable<V>` is similar to `Runnable` but it **returns a result** and **can throw checked exceptions**:

```java
Callable<Bitmap> loadBitmapTask = () -> {
    // can throw IOException, etc.
    return BitmapFactory.decodeStream(url.openStream());
};
```

`Callable` is submitted to an `ExecutorService` via `submit()`, which returns a `Future<V>`.

---

### 2.4 `Future<V>`

`Future<V>` is a **handle to an asynchronous computation**. It allows you to:
- **Retrieve the result** once computation completes
- **Block** the calling thread while waiting for the result
- **Cancel** the task (with limitations — see Section 6)
- **Check status** without blocking

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<Bitmap> future = executor.submit(() -> {
    return BitmapFactory.decodeStream(url.openStream());
});

// Later, on a background thread (never block the Main Thread!):
try {
    Bitmap bitmap = future.get();              // blocks until done
    Bitmap bitmap2 = future.get(5, TimeUnit.SECONDS); // with timeout
} catch (ExecutionException e) {
    // The Callable threw an exception
    Throwable cause = e.getCause();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore interrupt status
} catch (TimeoutException e) {
    future.cancel(true); // give up
}

// Non-blocking checks
boolean done      = future.isDone();
boolean cancelled = future.isCancelled();

// Cancel: sends interrupt signal if task is running AND mayInterruptIfRunning=true
future.cancel(true);
```

> ⚠️ **Never call `future.get()` on the Main Thread.** It blocks until the result is ready, which will cause ANR if the task takes longer than 5 seconds.

---

### 2.5 `Handler`

`Handler` is Android's mechanism for **sending messages and posting Runnables to a thread's `MessageQueue`**. It always targets a specific `Looper` (and therefore a specific thread).

```java
// Handler targeting the Main Thread
Handler mainHandler = new Handler(Looper.getMainLooper());

// Post a Runnable to run on the Main Thread from any background thread
mainHandler.post(() -> textView.setText("Updated from background!"));

// Post with a delay
mainHandler.postDelayed(() -> animate(), 300);

// Post at an absolute time
mainHandler.postAtTime(() -> doSomething(), SystemClock.uptimeMillis() + 1000);

// Send a Message object
Message msg = Message.obtain(mainHandler, MSG_LOAD_DONE);
msg.obj = bitmap;
mainHandler.sendMessage(msg);

// Handle messages in a custom Handler subclass
Handler handler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(@NonNull Message msg) {
        switch (msg.what) {
            case MSG_LOAD_DONE:
                imageView.setImageBitmap((Bitmap) msg.obj);
                break;
        }
    }
};
```

#### Handler Memory Leak Warning

Anonymous `Handler` subclasses and anonymous `Runnable` instances posted to a `Handler` hold an **implicit reference** to the enclosing class (e.g., `Activity`). If the Activity is destroyed before the message is processed, the Handler's reference keeps the Activity alive (memory leak).

**Fix**: Use a `WeakReference` to the Activity, or remove callbacks on destroy:

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null); // remove ALL pending messages/runnables
}
```

---

### 2.6 `Looper`

`Looper` runs an **infinite message-processing loop** for a thread. Without a `Looper`, a thread executes its `run()` method and terminates. With a `Looper`, the thread stays alive waiting for messages.

```
Thread ──has──► Looper ──reads──► MessageQueue ──dispatches to──► Handler.handleMessage()
```

Only **one Looper per thread** is allowed. The Main Thread has a Looper created automatically by the Android framework.

```java
// On a background thread — manual Looper setup:
class LooperThread extends Thread {
    public Handler handler;

    @Override
    public void run() {
        Looper.prepare();                     // 1. Create Looper for this thread
        handler = new Handler(Looper.myLooper()) {  // 2. Create Handler bound to this Looper
            @Override
            public void handleMessage(Message msg) {
                // process messages on this background thread
            }
        };
        Looper.loop();                        // 3. Start processing messages (blocks here)
    }
}
```

#### Key `Looper` Methods

| Method | Description |
|---|---|
| `Looper.prepare()` | Creates a `Looper` for the current thread. Must be called before `loop()`. |
| `Looper.loop()` | Starts the message loop. **Blocks the calling thread until `quit()` is called.** |
| `Looper.myLooper()` | Returns the `Looper` for the current thread, or `null` if none. |
| `Looper.getMainLooper()` | Returns the Main Thread's `Looper`. Safe to call from any thread. |
| `looper.quit()` | Stops the loop immediately. All pending messages (present and future) are **discarded**. |
| `looper.quitSafely()` | Stops the loop, but first processes all messages whose delivery time has **already passed** (`when ≤ now`). Messages scheduled for the **future** are still discarded. |

---

### 2.7 `HandlerThread`

`HandlerThread` is a convenient subclass of `Thread` that **automatically sets up a `Looper`** for you. Use it when you need a single dedicated background thread with a message queue.

```java
// Create and start the HandlerThread
HandlerThread handlerThread = new HandlerThread("MyBackgroundThread",
        Process.THREAD_PRIORITY_BACKGROUND);
handlerThread.start();

// Get a Handler that posts work to this background thread
Handler backgroundHandler = new Handler(handlerThread.getLooper());

// Post work to the background thread
backgroundHandler.post(() -> {
    // Runs on handlerThread, NOT the Main Thread
    Bitmap bitmap = loadLargeBitmap();

    // Switch back to Main Thread to update UI
    new Handler(Looper.getMainLooper()).post(() -> {
        imageView.setImageBitmap(bitmap);
    });
});

// Clean up when done (e.g., in onDestroy)
handlerThread.quitSafely();
```

`HandlerThread` is used internally by Android for things like `Camera` callbacks, `IntentService`, and `AsyncQueryHandler`. It is ideal for **serialized** background operations (one at a time, in order).

---

### 2.8 Interaction Diagram

```
  ┌─────────────────────────────────────────────────────────┐
  │                    BACKGROUND THREAD                    │
  │                                                         │
  │  handlerThread.start()                                  │
  │       └─► Looper.prepare() → MessageQueue created       │
  │           Looper.loop() → waiting for messages...       │
  │                 ▲                                       │
  └─────────────────│───────────────────────────────────────┘
                    │ enqueue Message/Runnable
  ┌─────────────────│───────────────────────────────────────┐
  │               Handler (bound to background Looper)      │
  │  handler.post(runnable)  ──────────────────────────►    │
  │  handler.sendMessage(msg) ─────────────────────────►    │
  └─────────────────────────────────────────────────────────┘
```

---

## 3. Thread Primitives — `volatile` and `synchronized`

### 3.1 The Java Memory Model (JMM)

Modern CPUs have **multi-level caches** (L1, L2, L3). Each CPU core may cache a copy of a shared variable. Without synchronization, one thread's write may never become visible to another thread — this is called a **visibility problem**.

The JMM defines **happens-before** relationships that guarantee when one thread's actions are visible to another. The key rules:
- A write to a `volatile` variable happens-before any subsequent read of that variable.
- An unlock of a monitor happens-before any subsequent lock of the same monitor.

---

### 3.2 `volatile`

`volatile` ensures that **all threads see the most up-to-date value** of a variable by:
1. Bypassing CPU caches — reads/writes go directly to main memory.
2. Preventing instruction reordering around the volatile access (memory barrier).

```java
// Without volatile — other threads may see stale value of 'running'
private boolean running = true;

// With volatile — guaranteed visibility across threads
private volatile boolean running = true;

// Producer thread
public void stop() {
    running = false; // immediately visible to all threads
}

// Consumer thread
public void run() {
    while (running) { // always reads fresh value
        doWork();
    }
}
```

#### ⚠️ `volatile` Does NOT Guarantee Atomicity

```java
private volatile int counter = 0;

// This is NOT thread-safe! It is a compound operation: read → increment → write
counter++; // Three steps that can interleave between threads
```

`volatile` is suitable only when:
1. Writes do not depend on the current value (simple assignment).
2. The variable is not part of an invariant with other variables.
3. No compound check-then-act operations.

---

### 3.3 `synchronized`

`synchronized` provides two guarantees:
1. **Mutual exclusion**: Only one thread executes the synchronized block at a time.
2. **Visibility**: Changes made inside a synchronized block are visible to all subsequent threads that acquire the same lock.

Every Java object has an intrinsic **monitor lock**. `synchronized` acquires this lock.

```java
// Synchronized method — locks on 'this'
public synchronized void increment() {
    counter++;
}

// Synchronized block — more granular, locks on an explicit object
private final Object lock = new Object();

public void increment() {
    synchronized (lock) {
        counter++;
    }
}

// Static synchronized method — locks on the Class object
public static synchronized void staticMethod() {
    // ...
}
```

#### `synchronized` Block vs Method

| Aspect | `synchronized` method | `synchronized` block |
|---|---|---|
| Lock object | `this` (instance) or `Class` (static) | Any object you choose |
| Granularity | Entire method | Only the critical section |
| Performance | Holds lock longer | Holds lock only as needed |
| Flexibility | Less control | Full control |

**Prefer synchronized blocks** with a private `final Object lock` — this prevents external code from accidentally synchronizing on the same object.

---

### 3.4 Common Pitfalls

#### Double-Checked Locking (Broken without `volatile`)

```java
// BROKEN — JVM may reorder instructions; other threads see partially-initialized object
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton(); // NOT atomic!
            }
        }
    }
    return instance;
}

// CORRECT — volatile prevents reordering
private static volatile Singleton instance;
```

#### Locking on the Wrong Object

```java
// WRONG — 'this' refers to a new anonymous Runnable, not shared
synchronized (new Object()) { ... }

// WRONG — Integer/String literals may be cached/shared unexpectedly
synchronized ("lock") { ... }

// CORRECT — private final field, always the same object
private final Object lock = new Object();
synchronized (lock) { ... }
```

---

## 4. Atomic Classes

### 4.1 The Problem: Compound Operations

Even with `volatile`, **compound operations** (read-modify-write) are not atomic:

```java
volatile int count = 0;

// Thread A and Thread B both executing concurrently:
// 1. Thread A reads count = 5
// 2. Thread B reads count = 5
// 3. Thread A writes count = 6
// 4. Thread B writes count = 6   ← lost update! Should be 7
count++;
```

Using `synchronized` works but has overhead. The `java.util.concurrent.atomic` package provides **lock-free** atomic operations using CPU-level **Compare-And-Swap (CAS)** instructions.

### 4.2 CAS (Compare-And-Swap)

CAS is a hardware-level atomic instruction:

```
CAS(variable, expected, newValue):
    if variable == expected:
        variable = newValue
        return true
    else:
        return false   // retry with fresh value
```

It is **optimistic** — assumes no contention, retries if another thread modified the value first. Under low-to-moderate contention, CAS outperforms `synchronized`.

---

### 4.3 `AtomicInteger` / `AtomicLong`

```java
AtomicInteger counter = new AtomicInteger(0);

counter.get()                    // read
counter.set(10)                  // write
counter.incrementAndGet()        // ++counter (returns new value)
counter.getAndIncrement()        // counter++ (returns old value)
counter.addAndGet(5)             // counter += 5 (returns new value)
counter.compareAndSet(5, 10)     // if counter==5 then counter=10; returns boolean

// Useful for thread-safe sequence numbers, reference counting, etc.
private final AtomicInteger requestId = new AtomicInteger(0);

public int nextRequestId() {
    return requestId.incrementAndGet();
}
```

---

### 4.4 `AtomicBoolean`

```java
AtomicBoolean isRunning = new AtomicBoolean(false);

// Safe start — only one thread can transition false→true
if (isRunning.compareAndSet(false, true)) {
    startWork(); // guaranteed to execute only once at a time
}

// Safe stop
isRunning.set(false);
```

---

### 4.5 `AtomicReference<V>`

```java
AtomicReference<Config> configRef = new AtomicReference<>(initialConfig);

// Thread-safe update of an immutable config object
Config current, updated;
do {
    current = configRef.get();
    updated = current.withNewSetting(value); // create new immutable config
} while (!configRef.compareAndSet(current, updated));
```

---

### 4.6 The ABA Problem

CAS can fail silently if a value changes from A → B → A between your read and your CAS:

```
Thread 1 reads: value = A
Thread 2 changes: A → B → A
Thread 1 CAS: expected=A, actual=A → succeeds, but state has changed!
```

**Fix**: `AtomicStampedReference<V>` adds an integer **stamp** (version number) alongside the reference:

```java
AtomicStampedReference<Node> ref = new AtomicStampedReference<>(node, 0);

int[] stampHolder = new int[1];
Node current = ref.get(stampHolder); // reads value AND stamp
int stamp = stampHolder[0];

// CAS checks BOTH value and stamp
ref.compareAndSet(current, newNode, stamp, stamp + 1);
```

---

### 4.7 Other Atomic Utilities

| Class | Use Case |
|---|---|
| `AtomicIntegerArray` | Lock-free operations on individual elements of an `int[]` |
| `AtomicLongArray` | Same for `long[]` |
| `AtomicReferenceArray<E>` | Same for object arrays |
| `AtomicIntegerFieldUpdater` | Apply atomic ops to a `volatile int` field without wrapping in `AtomicInteger` |
| `LongAdder` | High-throughput counter; better than `AtomicLong` under high contention |
| `LongAccumulator` | Generalized `LongAdder` with custom accumulation function |

---

### 4.8 When to Use Atomic Classes vs `synchronized`

| Scenario | Recommendation |
|---|---|
| Simple counter / flag | `AtomicInteger` / `AtomicBoolean` |
| Single variable, low contention | Atomic class |
| Multiple related variables (invariant) | `synchronized` (atomics can't protect multi-variable invariants) |
| High contention counter | `LongAdder` |
| Complex state transition | `synchronized` or `ReentrantLock` |

---

## 5. Thread Interruption in Java

### 5.1 What is an Interrupt?

An **interrupt** is a **cooperative cancellation signal** from one thread to another. It does NOT forcibly stop the target thread — it merely sets a flag that the target thread should check and decide what to do.

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        doUnitOfWork(); // check at each loop iteration
    }
    // cleanup
});
worker.start();

// From another thread, request cancellation:
worker.interrupt(); // sets the interrupt flag on 'worker'
```

---

### 5.2 The Interrupt Status Flag

Every thread has a boolean **interrupt status** flag.

| Method | Description |
|---|---|
| `thread.interrupt()` | Sets the interrupt flag on `thread`. |
| `Thread.currentThread().isInterrupted()` | Reads the flag. **Does NOT clear it.** |
| `Thread.interrupted()` | Static method. Reads AND **clears** the flag for the current thread. |

---

### 5.3 How Blocking Methods Respond

The key insight: methods that block (put a thread in WAITING/TIMED_WAITING state) **check the interrupt flag when entering or while waiting**, and if set, they throw `InterruptedException` and **clear the flag**.

```java
Thread.sleep(1000);    // throws InterruptedException if interrupted
Object.wait();         // throws InterruptedException if interrupted
Thread.join();         // throws InterruptedException if interrupted
BlockingQueue.take();  // throws InterruptedException if interrupted
```

```java
// Example: task that sleeps and is interruptible
public void run() {
    try {
        while (true) {
            processNextItem();
            Thread.sleep(100); // will throw if interrupted
        }
    } catch (InterruptedException e) {
        // Thread was interrupted while sleeping
        // Re-interrupt the thread (flag was cleared by the exception)
        Thread.currentThread().interrupt();
        // Now clean up...
        Log.d("TAG", "Worker interrupted, shutting down.");
    }
}
```

---

### 5.4 Best Practices for Handling `InterruptedException`

#### ❌ WRONG: Swallow the exception
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // Do nothing — the interrupt is lost! Caller cannot detect cancellation.
}
```

#### ✅ CORRECT Option 1: Re-interrupt
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore the flag
}
```

#### ✅ CORRECT Option 2: Propagate
```java
public void myMethod() throws InterruptedException {
    Thread.sleep(1000); // just let it propagate
}
```

#### ✅ CORRECT Option 3: Clean up and exit
```java
try {
    while (!Thread.currentThread().isInterrupted()) {
        doWork();
        Thread.sleep(100);
    }
} catch (InterruptedException e) {
    // Expected — time to stop
    cleanup();
    // Optionally re-interrupt
    Thread.currentThread().interrupt();
}
```

---

## 6. Why You Cannot Cancel a Running Thread Directly

### 6.1 `Thread.stop()` is Deprecated and Dangerous

Java originally had `Thread.stop()`, but it was **deprecated in Java 1.2** (formally marked with the `@Deprecated` annotation) and has never been safe to use. In **desktop JDK 21**, it was finally removed. On Android (which runs its own **ART runtime**, not desktop OpenJDK), `Thread.stop()` still exists in the API but must **never** be called because it is fundamentally unsafe:

When `Thread.stop()` is called:
1. It throws a `ThreadDeath` Error **at any arbitrary point** in the target thread's execution.
2. The thread unwinds its stack, releasing every monitor lock it holds.
3. But the **shared data protected by those locks may be in an inconsistent state** at that arbitrary point.
4. Other threads then acquire those locks and see corrupted data.

```java
// Example of why Thread.stop() is dangerous:
synchronized (account) {
    account.debit(amount);   // ← Thread.stop() here?
    account.credit(other);   // ← locks released, amount debited but not credited!
}
```

### 6.2 `Thread.suspend()` and `Thread.resume()` are Also Deprecated

`Thread.suspend()` pauses a thread **without releasing its locks**. If another thread tries to acquire those locks, it **deadlocks forever** (because only `Thread.resume()` can wake the suspended thread, but the thread holding the call to `resume()` may be the one that needs the lock).

---

### 6.3 The Cooperative Cancellation Pattern

The correct approach is **cooperative cancellation** — the running thread periodically checks a cancellation flag and stops itself cleanly.

#### Using the Interrupt Flag

```java
public class DownloadTask implements Runnable {
    @Override
    public void run() {
        try {
            for (int i = 0; i < 1000; i++) {
                if (Thread.currentThread().isInterrupted()) {
                    Log.d("TAG", "Cancelled at chunk " + i);
                    return; // exit cleanly
                }
                downloadChunk(i);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            Log.d("TAG", "Interrupted during blocking I/O");
        }
    }
}

Thread t = new Thread(new DownloadTask());
t.start();
t.interrupt(); // request cancellation
```

#### Using an `AtomicBoolean` Cancel Flag

```java
public class CancellableTask implements Runnable {
    private final AtomicBoolean cancelled = new AtomicBoolean(false);

    public void cancel() {
        cancelled.set(true);
    }

    @Override
    public void run() {
        while (!cancelled.get()) {
            doWork();
        }
    }
}
```

The interrupt flag approach is preferred because it also works with blocking methods (`sleep`, `wait`, `take`, etc.).

---

### 6.4 `Future.cancel(boolean mayInterruptIfRunning)`

`Future.cancel(true)` does the following:
- If the task **hasn't started**: marks it as cancelled, task will never execute.
- If the task **is running**: calls `thread.interrupt()` on the thread running the task (only if `mayInterruptIfRunning=true`). The task must cooperate by checking the interrupt flag.
- If the task **is done**: returns `false`, has no effect.

```java
Future<?> future = executor.submit(myTask);

// Later:
boolean cancelled = future.cancel(true); // attempt interruption
if (!cancelled) {
    Log.d("TAG", "Task already completed");
}
```

`Future.cancel()` does **not** forcibly stop the thread. The running task must be written to respond to interrupts.

---

## 7. The Android Main (UI) Thread

### 7.1 What Runs on the Main Thread

The Main Thread is responsible for **all UI-related work**:

| Category | Examples |
|---|---|
| **Activity/Fragment lifecycle** | `onCreate()`, `onStart()`, `onResume()`, `onPause()`, `onStop()`, `onDestroy()` |
| **View system** | Measuring, laying out, and drawing every View in the hierarchy |
| **Input events** | Touch events, key events, trackball events |
| **Broadcast Receivers** | `onReceive()` (runs on Main Thread by default) |
| **Service callbacks** | `onStartCommand()`, `onBind()` |
| **Handler messages** | Any `Handler(Looper.getMainLooper())` callbacks |
| **Choreographer callbacks** | Frame callbacks from `Choreographer` |

---

### 7.2 The Frame Budget

```
60 fps  →  1000 ms / 60  =  16.67 ms per frame
90 fps  →  1000 ms / 90  =  11.11 ms per frame
120 fps →  1000 ms / 120 =   8.33 ms per frame
```

Within each frame budget, the Main Thread must complete:
1. Handle input events
2. Run any pending `Message`/`Runnable` from the `MessageQueue`
3. Run Choreographer frame callbacks
4. Perform `View.measure()` / `View.layout()` / `View.draw()`
5. Submit drawing commands to `RenderThread`

If any of these steps takes too long, a frame is skipped → **jank**.

---

### 7.3 ANR (Application Not Responding)

| Trigger | Threshold |
|---|---|
| Activity not responding to input | **5 seconds** |
| `BroadcastReceiver.onReceive()` not completing (foreground app) | **10 seconds** |
| `BroadcastReceiver.onReceive()` not completing (background app) | **60 seconds** |
| Foreground `Service` not completing `onStartCommand()` / `onBind()` | **20 seconds** |
| Background `Service` not completing | **200 seconds** |
| `ContentProvider` not responding | **10 seconds** |

> **Note**: These thresholds are approximate and can vary by Android version and OEM customizations. Starting from Android 11, background process ANR enforcement became stricter. Always measure with real devices.

---

### 7.4 What Must NOT Run on the Main Thread

- **Network I/O**: Android enforces this since API 11 via `NetworkOnMainThreadException`
- **Disk I/O**: Reading/writing large files, database queries
- **Heavy computation**: Sorting large lists, image processing, cryptography
- **`Thread.sleep()`**: Never deliberately pause the Main Thread
- **`future.get()`**: Never block-wait on the Main Thread

---

### 7.5 Detecting Main Thread Violations with StrictMode

```java
// In Application.onCreate() or Activity.onCreate() for DEBUG builds
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
        .detectDiskReads()
        .detectDiskWrites()
        .detectNetwork()
        .detectCustomSlowCalls()   // annotate slow calls with @MainThread + StrictMode.noteSlowCall()
        .penaltyLog()              // log to Logcat
        .penaltyDialog()           // show a dialog
        // .penaltyDeath()         // crash the app (useful in CI)
        .build());
}
```

---

## 8. The Main Thread Message Queue

### 8.1 Architecture

The Main Thread's message processing system is one of the most important — and most misunderstood — parts of Android. It has **four interlocking components** that must all work together to process work on the Main Thread.

```
  ANY THREAD                          MAIN THREAD
  ──────────                          ───────────────────────────────────────────────────────────
                                      ┌──────────────────────────────────────────────────────┐
                                      │  ActivityThread.main()                               │
                                      │     Looper.prepareMainLooper()                       │
                                      │     Looper.loop()  ◄─── blocks here forever          │
                                      │         │                                            │
                                      │         │ polls MessageQueue.next()                  │
                                      │         ▼                                            │
                                      │    MessageQueue  (sorted linked list, by 'when')     │
                                      │    ┌──────────────────────────────────────────────┐  │
                                      │    │ [Sync Barrier? target=null]  ◄─ optional     │  │
                                      │    │ [Async Msg: frame traversal] ◄─ skips barrier│  │
                                      │    │ [Sync  Msg: what=1, T=100ms]                 │  │
                                      │    │ [Sync  Msg: what=2, T=200ms]                 │  │
                                      │    │ [Sync  Msg: callback=Runnable, T=300ms]      │  │
                                      │    └──────────────────────────────────────────────┘  │
                                      │         │                                            │
                                      │         ▼  (when T ≤ SystemClock.uptimeMillis())     │
                                      │    msg.target.dispatchMessage(msg)                   │
                                      │         │                                            │
                                      │         ├─ msg.callback != null → Runnable.run()     │
                                      │         ├─ handler.mCallback != null → callback()    │
                                      │         └─ handler.handleMessage(msg)                │
                                      └──────────────────────────────────────────────────────┘
        │                                       ▲
        │  handler.sendMessage(msg)              │
        │  handler.post(runnable)                │
        └───────── enqueue via ─────────────────►│
                   MessageQueue.enqueueMessage() │
                   (synchronized block on 'this')│
```

#### The Four Components in Depth

---

#### A. `ActivityThread` — The Entry Point

`ActivityThread` is the real entry point of every Android app process (not `Application.onCreate()` as many assume). Its `main()` method is called by the Android runtime (Zygote) when the process starts:

```java
// Simplified version of ActivityThread.main() (Android source)
public static void main(String[] args) {
    // 1. Prepare the Main Looper (creates MessageQueue for this thread)
    Looper.prepareMainLooper();

    // 2. Create the ActivityThread and its Handler
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    // 3. Start the message loop — this call NEVER returns (until process dies)
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

Every `Activity` launch, `onResume()`, touch event, and layout pass is delivered as a `Message` to this `ActivityThread`'s `Handler` (`mH`), which dispatches to the appropriate lifecycle method.

---

#### B. `Looper` — The Message Pump

`Looper` is the **engine** that drives the message queue. It owns a `MessageQueue` and continuously calls `MessageQueue.next()` to fetch and dispatch messages.

```java
// Simplified Looper.loop() (Android source)
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;

    for (;;) {
        // next() BLOCKS (via epoll/native poll) when the queue is empty
        // or when the next message's delivery time has not arrived yet
        Message msg = queue.next();

        if (msg == null) {
            // null means the Looper was quit — exit the loop
            return;
        }

        // Dispatch: calls msg.target.dispatchMessage(msg)
        // msg.target is the Handler that posted this message
        msg.target.dispatchMessage(msg);

        // Recycle Message back to the pool
        msg.recycleUnchecked();
    }
}
```

**Important**: `Looper.loop()` is an **infinite blocking loop**. It does NOT spin-wait the CPU. When the queue is empty or the next message is in the future, it parks the thread using the OS **epoll** mechanism (zero CPU usage while waiting).

---

#### C. `MessageQueue` — The Sorted Linked List

`MessageQueue` is **not** a standard Java queue — it is a **singly-linked list of `Message` objects sorted in ascending order by their `when` field** (`SystemClock.uptimeMillis()` timestamp of when the message should be delivered).

```
mMessages (head)
    │
    ▼
 Message(when=100) → Message(when=150) → Message(when=200) → Message(when=500) → null
```

##### How `enqueueMessage()` Works (Thread-Safe Insertion)

```java
// Simplified MessageQueue.enqueueMessage()
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {              // the queue itself is the lock object
        msg.when = when;
        Message p = mMessages;         // current head of the list
        boolean needWake;

        if (p == null || when == 0 || when < p.when) {
            // Insert at the HEAD (empty queue, or this msg is earliest)
            msg.next = p;
            mMessages = msg;           // new head
            needWake = mBlocked;       // wake up the blocked thread if sleeping
        } else {
            // Walk the list to find the correct insertion point (sorted by 'when')
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;             // found insertion point between prev and p
                }
            }
            msg.next = p;
            prev.next = msg;           // insert: prev → msg → p
            needWake = false;          // no need to wake; head didn't change
        }

        if (needWake) {
            nativeWake(mPtr);          // write to epoll fd → unblock nativePollOnce()
        }
    }
    return true;
}
```

Key observations:
- Insertion is **O(n)** in the worst case (must walk to find position), but in practice messages are usually appended near the end (delayed) or inserted at the front (immediate), making it fast.
- The `synchronized (this)` block makes insertion **thread-safe** — any thread can call `enqueueMessage()` concurrently without corrupting the list.
- When a new message is inserted **before** the currently-next-due message (i.e., at the head), `nativeWake()` is called to interrupt the sleeping `Looper` so it re-evaluates the new earliest delivery time.

##### How `next()` Works — Native Polling

```java
// Simplified MessageQueue.next()
Message next() {
    int nextPollTimeoutMillis = 0;

    for (;;) {
        // Block until either:
        //   nextPollTimeoutMillis =  0 → return immediately (don't sleep)
        //   nextPollTimeoutMillis >  0 → sleep for that many ms, then re-check
        //   nextPollTimeoutMillis = -1 → sleep indefinitely (queue is empty)
        nativePollOnce(mPtr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;   // start at head of the sorted linked list

            if (msg != null && msg.target == null) {
                // Hit a SYNC BARRIER (target == null is the barrier sentinel)
                // Skip all synchronous messages; find the next ASYNC message
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }

            if (msg != null) {
                if (now < msg.when) {
                    // Message is not yet due — calculate sleep duration
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Message is due — dequeue it from the linked list
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next; // remove from middle (async msg after barrier)
                    } else {
                        mMessages = msg.next;    // remove from head (normal case)
                    }
                    msg.next = null;             // detach from list
                    return msg;                  // hand to Looper.loop() for dispatch
                }
            } else {
                // No messages at all — sleep indefinitely until nativeWake() is called
                nextPollTimeoutMillis = -1;
            }

            // Run IdleHandlers when queue has nothing urgent to process
            // ... (see Section 8.4)
        }
    }
}
```

The native layer uses **Linux `epoll`** to block the thread with zero CPU cost. A file descriptor (`mPtr`) is written to by `nativeWake()` (called from `enqueueMessage()` on any thread) to unblock `nativePollOnce()` when a new message arrives or the queue state changes.

---

#### D. `Handler` — The Message Gateway

`Handler` is the **public API surface** for inserting and receiving messages. Every `Handler` is permanently bound to exactly **one `Looper`** (and therefore one thread) at construction time.

```java
// Handler dispatch logic (simplified)
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        // Message was posted via handler.post(runnable) — callback is the Runnable
        handleCallback(msg);           // → msg.callback.run()
    } else {
        if (mCallback != null) {
            // Handler was constructed with a Callback object
            if (mCallback.handleMessage(msg)) {
                return;               // Callback consumed the message — don't propagate
            }
        }
        // Fall through to the Handler subclass override
        handleMessage(msg);
    }
}
```

This three-tier dispatch (Runnable callback → Handler.Callback → handleMessage override) gives flexible interception points, which frameworks like `Fragment` use to intercept messages before they reach your code.

---

#### E. Full Message Lifecycle

```
1. Producer (any thread):
   handler.post(runnable)
       └─► Message.obtain()           (from pool)
       └─► msg.callback = runnable
       └─► msg.when = uptimeMillis() + delayMs
       └─► MessageQueue.enqueueMessage(msg, when)
               └─► insert into sorted linked list (synchronized)
               └─► nativeWake() if necessary

2. Looper (Main Thread):
   MessageQueue.next()
       └─► nativePollOnce() — sleep until msg.when ≤ now
       └─► dequeue msg from list (synchronized)
       └─► return msg to Looper.loop()

3. Dispatch (Main Thread):
   msg.target.dispatchMessage(msg)
       └─► msg.callback.run()  OR  handler.handleMessage(msg)

4. Recycling:
   msg.recycleUnchecked()
       └─► clear all fields
       └─► return to Message pool (up to MAX_POOL_SIZE = 50)
```

---

#### F. Thread Safety of the Message Queue

The `MessageQueue` is designed to be called from **any thread** for enqueueing, but **only from the Looper's thread** for dequeueing:

| Operation | Thread | Synchronization |
|---|---|---|
| `enqueueMessage()` | Any thread | `synchronized (this)` block on the queue |
| `removeMessages()` | Any thread | `synchronized (this)` block on the queue |
| `next()` (dequeue) | Looper thread only | `synchronized (this)` block inside `next()` |
| `nativePollOnce()` | Looper thread only | Native epoll — no Java lock held while sleeping |

> ⚠️ The Looper thread **does NOT hold the Java lock** while sleeping in `nativePollOnce()`. This means background threads can always enqueue messages without blocking on the sleeping Looper.

---

#### G. `ActivityThread.mH` — The Main Thread's Master Handler

`ActivityThread` has an inner class `H` that handles all framework-level messages on the Main Thread. Its `handleMessage()` is the single dispatch point for lifecycle events. Some key message constants (values vary by Android version):

```java
// Selected MSG constants from ActivityThread.H
// (Android 9+ / API 28+ unified lifecycle via ClientTransaction)
public static final int BIND_APPLICATION    = 110;  // Application.onCreate()
public static final int EXECUTE_TRANSACTION = 159;  // Activity/Fragment lifecycle (API 28+)
public static final int RELAUNCH_ACTIVITY   = 160;
public static final int CREATE_SERVICE      = 114;
public static final int STOP_SERVICE        = 116;
public static final int RECEIVER            = 113;  // BroadcastReceiver.onReceive()
public static final int CONFIGURATION_CHANGED = 118;

// Before Android 9 (API 28), separate messages existed for each lifecycle step:
// LAUNCH_ACTIVITY (100), RESUME_ACTIVITY (107), PAUSE_ACTIVITY (101), etc.
// These were replaced by EXECUTE_TRANSACTION + ClientTransaction in API 28.
```

When the Android system wants to launch an `Activity`, it sends an IPC call (via Binder) to your app process, which the `ApplicationThread` Binder stub receives, then **posts a message** to `ActivityThread.mH` to actually do the work on the Main Thread. This is why all lifecycle callbacks are guaranteed to run on the Main Thread.

```
System Server (remote process)
    │ Binder IPC
    ▼
ApplicationThread (Binder stub, runs on Binder thread pool in your process)
    │ mH.sendMessage(H.EXECUTE_TRANSACTION, ...)
    ▼
ActivityThread.H.handleMessage() — runs on MAIN THREAD
    │
    └─► Activity.onCreate() / onResume() / etc.
```

---

### 8.2 The `Message` Object

`Message` is a lightweight data container:

```java
Message msg = Message.obtain(); // use pool, avoid allocation
msg.what = MSG_TYPE;            // int identifier
msg.arg1 = 42;                  // int argument 1
msg.arg2 = 100;                 // int argument 2
msg.obj  = myObject;            // arbitrary Object payload
// msg.callback = runnable;     // if set, Looper calls this instead of handleMessage()

handler.sendMessage(msg);
```

**Always use `Message.obtain()`** instead of `new Message()`. Android maintains a pool of recycled `Message` objects (up to 50) to avoid GC pressure.

---

### 8.3 Posting Messages

```java
Handler h = new Handler(Looper.getMainLooper());

// Post a Runnable (sets msg.callback internally)
h.post(runnable);

// Post a Runnable with delay (relative to now)
h.postDelayed(runnable, 500 /* ms */);

// Post a Runnable at an absolute uptime
h.postAtTime(runnable, SystemClock.uptimeMillis() + 500);

// Post at the FRONT of the queue (bypasses waiting messages)
h.postAtFrontOfQueue(runnable); // use sparingly — can starve other messages

// Send a Message immediately
h.sendMessage(msg);

// Send with delay
h.sendMessageDelayed(msg, 500);

// Send at absolute time
h.sendMessageAtTime(msg, SystemClock.uptimeMillis() + 500);

// Convenience: send empty message (only 'what' field)
h.sendEmptyMessage(MSG_REFRESH);
h.sendEmptyMessageDelayed(MSG_REFRESH, 1000);
```

---

### 8.4 `MessageQueue.addIdleHandler()`

`IdleHandler` callbacks are invoked when the `MessageQueue` has **no more messages to process** (the queue is idle):

```java
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override
    public boolean queueIdle() {
        // Runs on the Main Thread when idle
        performLowPriorityWork();
        return false; // false = remove this handler; true = keep it for future idle periods
    }
});
```

Android uses this internally for things like `ActivityThread` garbage collection hints and Choreographer idle frame callbacks.

---

### 8.5 Synchronization Barriers

**Synchronization barriers** are a hidden internal mechanism used by the VSYNC pipeline. A barrier is a special `Message` with `target == null`. When `MessageQueue.next()` encounters a barrier, it **skips all synchronous messages** and only dispatches **asynchronous messages** (messages marked with `msg.setAsynchronous(true)`).

This allows VSYNC frame work (marked asynchronous) to **jump ahead of** regular application messages during a frame render.

```java
// Internal Android usage (not public API):
// int token = messageQueue.postSyncBarrier(); // posts a barrier
// ... VSYNC async messages are processed ...
// messageQueue.removeSyncBarrier(token);      // remove barrier, normal messages resume

// You can mark your own messages as async (API 22+):
Message msg = Message.obtain();
msg.setAsynchronous(true); // this message bypasses sync barriers
handler.sendMessage(msg);
```

> ⚠️ `MessageQueue.postSyncBarrier()` is a `@hide` API — do not use it directly in production apps.

---

## 9. VSYNC and the Display Pipeline

### 9.1 What is VSYNC?

**VSYNC** (Vertical Synchronization) is a **hardware signal** emitted by the display driver at a fixed rate (typically 60, 90, or 120 Hz) that marks the beginning of a new display refresh cycle.

Without VSYNC synchronization, the GPU might send a new frame while the display is halfway through scanning out the previous one — causing a **visual tear** (top half shows frame N, bottom half shows frame N+1).

VSYNC tells the rendering pipeline: **"Start preparing the next frame NOW so it's ready when I refresh."**

---

### 9.2 The Display Pipeline

```
Hardware VSYNC Signal
        │
        ▼
  SurfaceFlinger (system process)
        │  distributes VSYNC to apps via Choreographer
        ▼
  Choreographer (in your app process)
        │  posts a frame callback as an ASYNC message
        ▼
  MessageQueue (Main Thread)
        │  sync barrier blocks normal messages
        │  async frame message is dispatched immediately
        ▼
  ViewRootImpl.performTraversals()
        │
        ├─► View.measure()    ← calculate sizes
        ├─► View.layout()     ← position views
        └─► View.draw()       ← record drawing commands
                │
                ▼
        RenderThread (separate thread)
                │
                ▼
        GPU (execute drawing commands)
                │
                ▼
        SurfaceFlinger composites layers
                │
                ▼
        Display shows frame
```

---

### 9.3 `Choreographer`

`Choreographer` is the Android class that **receives VSYNC signals and schedules frame rendering**. It lives on the Main Thread.

```java
// Register a frame callback (runs at the next VSYNC)
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        // frameTimeNanos: the VSYNC timestamp in nanoseconds
        // Update animations, custom views, etc.
        updateAnimation(frameTimeNanos);

        // Re-register for next frame if animation is still running
        if (animationRunning) {
            Choreographer.getInstance().postFrameCallback(this);
        }
    }
});
```

`ValueAnimator`, `ObjectAnimator`, and `RecyclerView` animations all use `Choreographer` internally.

---

### 9.4 How VSYNC Uses the Sync Barrier to Lock the Message Queue

The VSYNC pipeline involves two separate mechanisms that work together:

**Part A — `ViewRootImpl.scheduleTraversals()` (called when a View is invalidated):**

1. Any `View.invalidate()` or `requestLayout()` call triggers `ViewRootImpl.scheduleTraversals()`.
2. `scheduleTraversals()` immediately calls `MessageQueue.postSyncBarrier()` to install a **sync barrier** in the queue.
3. It then calls `Choreographer.postCallback(CALLBACK_TRAVERSAL, mTraversalRunnable)` to register the traversal work as a frame callback.
4. The Choreographer callback is stored but does **not** run yet — it waits for the next VSYNC signal.

**Part B — VSYNC arrives from the display hardware:**

5. The display driver fires a VSYNC signal; Android's `DisplayEventReceiver` (native) receives it.
6. `Choreographer.onVsync()` is called, which posts the registered frame callbacks as **asynchronous messages** into the Main Thread's `MessageQueue`.
7. `Looper.loop()` sees the sync barrier → **skips all synchronous messages** → dispatches the async frame message immediately.
8. `ViewRootImpl.performTraversals()` executes: measure + layout + draw.
9. After the frame traversal completes, the sync barrier is **removed** → normal synchronous messages resume.

```
View.invalidate()
    │
    ▼
ViewRootImpl.scheduleTraversals()
    ├─► MessageQueue.postSyncBarrier()     ← barrier posted NOW
    └─► Choreographer.postCallback(TRAVERSAL, runnable)
                │
                │ (waiting for VSYNC)
                ▼
    DisplayEventReceiver.onVsync()         ← hardware VSYNC fires
                │
                ▼
    Choreographer posts ASYNC message into MessageQueue
                │
    MessageQueue state:
    ┌─────────────────────────────────────────────┐
    │ [SYNC BARRIER]          ← blocks sync msgs   │
    │ [ASYNC: performTraversals] ← runs first      │
    │ [SYNC: user message A]  ← skipped (blocked)  │
    │ [SYNC: user message B]  ← skipped (blocked)  │
    └─────────────────────────────────────────────┘
                │
                ▼
    ViewRootImpl.performTraversals()
    MessageQueue.removeSyncBarrier()       ← sync msgs resume
```

This ensures that **frame rendering always gets priority** over regular application messages, preventing jank caused by a busy message queue.

---

### 9.5 Jank: What Happens When a Frame Misses the Deadline

If the Main Thread takes longer than 16 ms (at 60 fps) to complete `performTraversals()`:

1. The frame is not ready when SurfaceFlinger composites.
2. SurfaceFlinger displays the **previous frame again**.
3. The user perceives a **stutter** (dropped frame = jank).
4. `Choreographer` logs: `"Skipped N frames! The application may be doing too much work on its main thread."`

#### Tools for Diagnosing Jank

- **Android Studio Profiler**: CPU/Frame timeline
- **Systrace / Perfetto**: See VSYNC signals, `performTraversals()` durations, and blocked messages
- **`adb shell dumpsys gfxinfo`**: Frame timing statistics
- **GPU Rendering bars** (Developer Options → Profile GPU Rendering)

---

## 10. Inter-Thread Communication

### 10.1 `Handler` + `Message` / `Runnable` (Android-Idiomatic)

The most common pattern in Android — post work to a specific thread's `Looper`:

```java
// Background thread → Main Thread
Handler mainHandler = new Handler(Looper.getMainLooper());
mainHandler.post(() -> textView.setText(result));

// Main Thread → Background HandlerThread
Handler bgHandler = new Handler(handlerThread.getLooper());
bgHandler.post(() -> processInBackground());
```

---

### 10.2 `Activity.runOnUiThread()`

Convenience method that posts a `Runnable` to the Main Thread, or runs it immediately if already on the Main Thread:

```java
// From a background thread:
activity.runOnUiThread(() -> {
    imageView.setImageBitmap(bitmap);
});
```

Internally: checks if current thread is Main Thread → if yes, calls `runnable.run()` directly; if no, calls `mainHandler.post(runnable)`.

---

### 10.3 `View.post()` / `View.postDelayed()`

```java
// Posts runnable to the View's attached Handler (which is the Main Thread Handler)
imageView.post(() -> imageView.setImageBitmap(bitmap));

// Also useful to delay work until after layout:
view.post(() -> {
    int width = view.getWidth(); // layout is complete at this point
});
```

---

### 10.4 `CountDownLatch`

Allows one or more threads to **wait until a set of operations** in other threads completes:

```java
int taskCount = 3;
CountDownLatch latch = new CountDownLatch(taskCount);

// Worker threads
for (int i = 0; i < taskCount; i++) {
    executor.execute(() -> {
        try {
            doWork();
        } finally {
            latch.countDown(); // decrement count
        }
    });
}

// Waiting thread blocks until count reaches 0
try {
    latch.await();                       // wait indefinitely
    latch.await(5, TimeUnit.SECONDS);    // wait with timeout
    Log.d("TAG", "All tasks complete!");
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

`CountDownLatch` is **not reusable** — once the count reaches zero it stays zero.

---

### 10.5 `CyclicBarrier`

A **reusable** synchronization point where N threads wait for each other before proceeding:

```java
int threadCount = 4;
CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> {
    // Optional runnable executed when all threads arrive
    Log.d("TAG", "All threads reached the barrier, proceeding...");
});

for (int i = 0; i < threadCount; i++) {
    executor.execute(() -> {
        phaseOne();
        try {
            barrier.await(); // wait for all threads
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
        phaseTwo(); // all threads start this simultaneously
    });
}
```

---

### 10.6 `Semaphore`

Controls access to a pool of resources by limiting the number of threads that can access it concurrently:

```java
// Allow max 3 concurrent database connections
Semaphore semaphore = new Semaphore(3);

executor.execute(() -> {
    try {
        semaphore.acquire();     // blocks if 3 permits are already taken
        useDatabase();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        semaphore.release();     // always release in finally
    }
});
```

---

### 10.7 `BlockingQueue` — Producer-Consumer Pattern

#### What is a `BlockingQueue`?

`BlockingQueue<E>` is a thread-safe queue (from `java.util.concurrent`) that adds two critical **blocking behaviours** on top of a regular queue:

1. **Block on put (full queue)**: If the queue has reached its capacity, the thread trying to add an element **blocks** until space becomes available (another thread consumes an element).
2. **Block on take (empty queue)**: If the queue is empty, the thread trying to remove an element **blocks** until an element becomes available (another thread produces one).

This makes `BlockingQueue` the **ideal backbone for the Producer-Consumer pattern** without requiring any manual `synchronized` / `wait()` / `notify()` code.

```
PRODUCER THREAD(S)                    CONSUMER THREAD(S)
──────────────────                    ─────────────────
      │                                       │
      │  queue.put(item)                      │  queue.take()
      │      │                                │      │
      │      ▼                                │      ▼
      │  ┌───────────────────────────────┐    │  dequeue from head
      └─►│  [item1][item2][item3][item4] │◄───┘
         │   ▲                           │
         │   HEAD                      TAIL
         └───────────────────────────────┘
              BlockingQueue (bounded or unbounded)

  If FULL  → put() BLOCKS producer until consumer takes
  If EMPTY → take() BLOCKS consumer until producer puts
```

---

#### Complete `BlockingQueue` Method Reference

Every `BlockingQueue` operation comes in **four flavours** depending on how you want to handle a full (for producers) or empty (for consumers) queue:

| Behaviour | Throws Exception | Returns Special Value | **Blocks** | Blocks with Timeout |
|---|---|---|---|---|
| **Insert** | `add(e)` → throws `IllegalStateException` | `offer(e)` → returns `false` | **`put(e)`** → waits forever | `offer(e, time, unit)` |
| **Remove** | `remove()` → throws `NoSuchElementException` | `poll()` → returns `null` | **`take()`** → waits forever | `poll(time, unit)` |
| **Examine** | `element()` → throws | `peek()` → returns `null` | — | — |

```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(5);

// ── INSERT ──────────────────────────────────────────────────────────────────
queue.add("a");                              // throws if full
queue.offer("b");                            // returns false if full (no throw)
queue.put("c");                              // BLOCKS until space available
queue.offer("d", 100, TimeUnit.MILLISECONDS); // blocks up to 100ms, then false

// ── REMOVE ──────────────────────────────────────────────────────────────────
String s1 = queue.remove();                   // throws if empty
String s2 = queue.poll();                     // returns null if empty (no throw)
String s3 = queue.take();                     // BLOCKS until element available
String s4 = queue.poll(100, TimeUnit.MILLISECONDS); // blocks up to 100ms, then null

// ── EXAMINE (peek without removing) ─────────────────────────────────────────
String head = queue.peek();                   // returns null if empty (no remove)

// ── BULK OPERATIONS ─────────────────────────────────────────────────────────
List<String> batch = new ArrayList<>();
int count = queue.drainTo(batch);             // move ALL available elements to list
int count2 = queue.drainTo(batch, 10);        // move AT MOST 10 elements

// ── QUERY ────────────────────────────────────────────────────────────────────
int size = queue.size();
int remaining = queue.remainingCapacity();     // Integer.MAX_VALUE if unbounded
boolean contains = queue.contains("a");
```

---

#### Basic Producer-Consumer Example

```java
// Bounded queue: max 10 items in flight at once
BlockingQueue<Bitmap> queue = new LinkedBlockingQueue<>(10);

// ── PRODUCER THREAD ──────────────────────────────────────────────────────────
Thread producer = new Thread(() -> {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            Bitmap frame = captureFrame();
            queue.put(frame);     // blocks if queue is full (natural backpressure)
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}, "producer-thread");

// ── CONSUMER THREAD ──────────────────────────────────────────────────────────
Thread consumer = new Thread(() -> {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            Bitmap frame = queue.take();   // blocks if queue is empty
            processFrame(frame);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}, "consumer-thread");

producer.start();
consumer.start();
```

---

#### Multiple Producers and Multiple Consumers

`BlockingQueue` is fully thread-safe for any number of producers and consumers simultaneously. No external synchronization is required:

```java
BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(50);
int producerCount = 3;
int consumerCount = Runtime.getRuntime().availableProcessors();

// Multiple producers
for (int i = 0; i < producerCount; i++) {
    final int id = i;
    new Thread(() -> {
        while (true) {
            Task t = generateTask(id);
            try {
                taskQueue.put(t); // all producers share the same queue safely
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }, "producer-" + i).start();
}

// Multiple consumers
for (int i = 0; i < consumerCount; i++) {
    new Thread(() -> {
        while (true) {
            try {
                Task t = taskQueue.take(); // all consumers share the same queue safely
                processTask(t);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }, "consumer-" + i).start();
}
```

---

#### Bounded vs Unbounded Queues

| Property | Bounded Queue | Unbounded Queue |
|---|---|---|
| Capacity | Fixed (e.g., `new LinkedBlockingQueue<>(100)`) | `Integer.MAX_VALUE` effectively |
| `put()` behaviour | **Blocks when full** — natural backpressure | Never blocks (until OOM) |
| Memory | Predictable, capped | Can grow without limit → `OutOfMemoryError` |
| Use case | Rate-limited pipelines, thread pools | Only when production rate is guaranteed ≤ consumption rate |

> ⚠️ **On Android, always prefer bounded queues.** An unbounded queue in the face of a slow consumer can silently consume all available heap memory, leading to OOM crashes that are extremely hard to diagnose.

---

#### `BlockingQueue` Implementations in Detail

---

##### 1. `LinkedBlockingQueue<E>`

The most commonly used implementation. Backed by a **singly-linked list of nodes**.

```java
// Unbounded (capacity = Integer.MAX_VALUE) — DANGEROUS on Android
BlockingQueue<Task> q1 = new LinkedBlockingQueue<>();

// Bounded — RECOMMENDED
BlockingQueue<Task> q2 = new LinkedBlockingQueue<>(100);
```

**Internal design**: Uses **two separate `ReentrantLock` instances** — one for the head (take lock) and one for the tail (put lock). This means **producers and consumers can operate concurrently** without blocking each other, which gives excellent throughput under high load.

```
  put()  ──────────────────────────────────────────────────►  take()
  inserts here                                          removes here
       │                                                       ▲
       ▼                                                       │
     TAIL                                                    HEAD
  [newest] → [node] → [node] → [node] → [oldest/next to take]
  (putLock                                           (takeLock
   guards TAIL)                                       guards HEAD)
```

- ✅ High throughput under concurrent load (two independent locks = producers and consumers never block each other)
- ✅ Can be bounded or unbounded
- ✅ Fair FIFO ordering
- ⚠️ Slightly higher memory overhead per element (each node is a heap-allocated object with a `next` pointer)

---

##### 2. `ArrayBlockingQueue<E>`

Backed by a **fixed-size circular array**. Always bounded (capacity given at construction).

```java
// Capacity is mandatory
BlockingQueue<Task> q = new ArrayBlockingQueue<>(50);

// Optional fairness: true = FIFO thread waiting order (less throughput, more predictable)
BlockingQueue<Task> fairQ = new ArrayBlockingQueue<>(50, true /* fair */);
```

**Internal design**: Uses a **single `ReentrantLock`** shared by both producers and consumers. This means only one thread (producer OR consumer) can touch the queue at a time. The array is circular — `putIndex` advances on each insert and wraps around; `takeIndex` advances on each remove.

```
Single ReentrantLock protects all access:

  items[] = [ ][ ][item2][item3][item4][ ][ ]
              ↑putIndex  ↑takeIndex
              (next      (oldest item,
               empty     removed next)
               slot)
  
  take() removes from takeIndex → takeIndex advances right (wraps)
  put()  inserts at  putIndex  → putIndex  advances right (wraps)
```

- ✅ **Lower memory overhead** — pre-allocated array, no per-node objects
- ✅ Better memory locality (contiguous array vs scattered linked nodes)
- ✅ Bounded by design — forces you to handle backpressure
- ⚠️ Lower throughput than `LinkedBlockingQueue` under high concurrent load (single lock)
- ✅ Optional fairness flag (useful for strict ordering guarantees)

**Use `ArrayBlockingQueue` when**:
- Memory footprint matters more than throughput
- You want strict capacity enforcement with pre-allocated memory
- Elements are small value types (array locality helps)

---

##### 3. `SynchronousQueue<E>`

A queue with **zero internal capacity**. Every `put()` blocks until another thread calls `take()`, and vice versa. It is effectively a **direct hand-off** between threads.

```java
BlockingQueue<Task> q = new SynchronousQueue<>();

// Producer — blocks until a consumer is ready
q.put(task);     // blocks here until a consumer calls take()

// Consumer — blocks until a producer is ready
Task t = q.take(); // blocks here until a producer calls put()
```

```
PRODUCER                    CONSUMER
   │                           │
   │  put(task) ─── BLOCKS     │
   │          ◄────────────────│ take() — rendezvous!
   │               handoff     │
   │  continues                │ continues with task
```

- ✅ **Zero buffering** — forces tight coupling between producer and consumer speed
- ✅ **Lowest latency** hand-off (no buffering delay)
- ✅ Used by `Executors.newCachedThreadPool()` internally to force immediate thread creation
- ⚠️ No buffering means any mismatch in producer/consumer speed causes one side to always block

**Use `SynchronousQueue` when**:
- You want `CallerRunsPolicy`-like backpressure at the queue level
- Each task must be handed off directly to an available worker (no queuing)
- Building a thread pool where tasks should spin up new threads immediately

---

##### 4. `PriorityBlockingQueue<E>`

An **unbounded, priority-ordered** blocking queue. Elements are ordered by their natural ordering (`Comparable`) or a `Comparator`.

```java
// Tasks with higher priority (lower int value) processed first
BlockingQueue<PriorityTask> q = new PriorityBlockingQueue<>(
    16,                                         // initial capacity (grows automatically)
    Comparator.comparingInt(PriorityTask::getPriority)
);

class PriorityTask implements Comparable<PriorityTask> {
    private final int priority; // lower = more urgent
    private final Runnable work;

    @Override
    public int compareTo(PriorityTask other) {
        return Integer.compare(this.priority, other.priority);
    }
}

// High-priority task jumps ahead of low-priority tasks
q.put(new PriorityTask(10, lowPriorityWork));
q.put(new PriorityTask(1,  urgentWork));   // this will be taken first

PriorityTask next = q.take(); // returns urgentWork (priority=1)
```

- ✅ Automatic task prioritization without manual reordering
- ⚠️ **Unbounded** — can grow without limit, risk of OOM
- ⚠️ `take()` never blocks (always available if non-empty) — use carefully with memory
- ⚠️ FIFO order is NOT guaranteed for elements with equal priority

**Use `PriorityBlockingQueue` when**:
- Tasks have different urgency (e.g., user-facing network requests vs background sync)
- You can bound the total task count through other means (rate limiting at the producer side)

---

##### 5. `DelayQueue<E extends Delayed>`

Elements can only be taken **after a specified delay has expired**. Elements implement the `Delayed` interface.

```java
class ScheduledTask implements Delayed {
    private final long executeAtMs;
    private final Runnable work;

    public ScheduledTask(long delayMs, Runnable work) {
        this.executeAtMs = SystemClock.uptimeMillis() + delayMs;
        this.work = work;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        long remaining = executeAtMs - SystemClock.uptimeMillis();
        return unit.convert(remaining, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed other) {
        return Long.compare(
            getDelay(TimeUnit.MILLISECONDS),
            other.getDelay(TimeUnit.MILLISECONDS)
        );
    }
}

DelayQueue<ScheduledTask> delayQueue = new DelayQueue<>();
delayQueue.put(new ScheduledTask(5000, () -> Log.d("TAG", "After 5s")));
delayQueue.put(new ScheduledTask(1000, () -> Log.d("TAG", "After 1s")));

// Consumer thread
Thread consumer = new Thread(() -> {
    while (true) {
        try {
            ScheduledTask task = delayQueue.take(); // blocks until earliest task is due
            task.work.run();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
});
```

- ✅ Built-in scheduling without `ScheduledExecutorService`
- ✅ Used internally by `ScheduledThreadPoolExecutor`
- ⚠️ **Unbounded** — no capacity limit

---

##### 6. `LinkedTransferQueue<E>` (API 21+)

An **unbounded**, highly concurrent queue that also supports a **transfer** operation — a producer can optionally wait until a consumer has actually received the element (stronger than `put()`):

```java
LinkedTransferQueue<Task> q = new LinkedTransferQueue<>();

// transfer() — blocks until a consumer ACTUALLY receives the task
// (stronger guarantee than put(), which just enqueues)
q.transfer(urgentTask); // blocks until a consumer thread calls take()

// tryTransfer() — returns false immediately if no consumer is waiting
boolean handed = q.tryTransfer(task);

// tryTransfer with timeout
boolean handed2 = q.tryTransfer(task, 100, TimeUnit.MILLISECONDS);
```

- ✅ Best throughput of all `BlockingQueue` implementations (uses sophisticated non-blocking algorithms)
- ✅ `transfer()` allows synchronous hand-off semantics when needed
- ⚠️ Unbounded — memory management is your responsibility

---

#### Comparison Summary

| Implementation | Bounded? | Ordering | Throughput | Memory | Best For |
|---|---|---|---|---|---|
| `LinkedBlockingQueue` | Optional | FIFO | ⭐⭐⭐⭐ | Medium | General-purpose, thread pools |
| `ArrayBlockingQueue` | Always | FIFO | ⭐⭐⭐ | Low (pre-allocated) | Memory-sensitive, strict capacity |
| `SynchronousQueue` | No buffer | N/A | ⭐⭐⭐⭐⭐ (direct hand-off) | Minimal | CachedThreadPool, direct dispatch |
| `PriorityBlockingQueue` | No | Priority | ⭐⭐⭐ | Grows unbounded | Prioritized task scheduling |
| `DelayQueue` | No | Time-delayed | ⭐⭐ | Grows unbounded | Scheduled task execution |
| `LinkedTransferQueue` | No | FIFO | ⭐⭐⭐⭐⭐ | Grows unbounded | Highest throughput pipelines |

---

#### Real Android Use Cases

| Use Case | Recommended Queue |
|---|---|
| `ThreadPoolExecutor` work queue (CPU tasks) | `LinkedBlockingQueue<>(N_cpu * 2)` |
| `ThreadPoolExecutor` work queue (I/O tasks) | `LinkedBlockingQueue<>(128)` |
| CachedThreadPool-style executor | `SynchronousQueue` |
| Camera frame processing pipeline | `ArrayBlockingQueue<>(3)` (bounded, low memory) |
| Priority download queue (user-visible vs background) | `PriorityBlockingQueue` |
| Scheduled retry queue (exponential backoff) | `DelayQueue` |
| High-throughput event pipeline | `LinkedTransferQueue` |

---

#### Common Pitfalls

```java
// ❌ PITFALL 1: Unbounded queue — silent OOM risk
BlockingQueue<Task> dangerous = new LinkedBlockingQueue<>(); // no capacity!

// ✅ FIX: Always specify a capacity
BlockingQueue<Task> safe = new LinkedBlockingQueue<>(100);

// ❌ PITFALL 2: Ignoring InterruptedException
try {
    queue.take();
} catch (InterruptedException e) {
    // swallowed! thread cannot be cancelled properly
}

// ✅ FIX: Re-interrupt or propagate
try {
    queue.take();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore interrupt status
    return; // or throw
}

// ❌ PITFALL 3: Using size() for control flow (TOCTOU race)
if (queue.size() < 10) {
    queue.put(item); // another thread may have filled it between the check and put
}

// ✅ FIX: Use offer() with a result check, or rely on put()'s blocking
if (!queue.offer(item)) {
    handleRejection(item); // queue was full
}

// ❌ PITFALL 4: Blocking the Main Thread
queue.put(item);   // on the Main Thread! Will ANR if queue is full
queue.take();      // on the Main Thread! Will ANR if queue is empty

// ✅ FIX: Always do BlockingQueue operations on background threads
executor.execute(() -> {
    try { queue.put(item); } catch (InterruptedException e) { /*...*/ }
});
```

---

### 10.8 `Exchanger<V>`

Two threads swap objects at a synchronization point:

```java
Exchanger<DataBuffer> exchanger = new Exchanger<>();

// Producer
executor.execute(() -> {
    DataBuffer buffer = fillBuffer();
    DataBuffer emptyBuffer = exchanger.exchange(buffer); // swap
    // now use emptyBuffer for next fill
});

// Consumer
executor.execute(() -> {
    DataBuffer emptyBuffer = new DataBuffer();
    DataBuffer fullBuffer = exchanger.exchange(emptyBuffer); // swap
    processBuffer(fullBuffer);
});
```

---

## 11. Locks and Synchronization Mechanisms

### 11.1 Overview

| Lock Type | Package | Reentrant | Interruptible | Timed | Condition Support | Fairness Option |
|---|---|---|---|---|---|---|
| `synchronized` | Java keyword | ✅ | ❌ | ❌ | ❌ (uses `wait`/`notify`) | ❌ |
| `ReentrantLock` | `java.util.concurrent.locks` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ReentrantReadWriteLock` | `java.util.concurrent.locks` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `StampedLock` | `java.util.concurrent.locks` | ❌ | ✅ | ✅ | ❌ | ❌ |

---

### 11.2 `synchronized` (Intrinsic Lock)

The simplest, JVM-managed lock. Best for simple mutual exclusion when you don't need advanced features.

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}
```

**Limitations**:
- Cannot try to acquire with a timeout
- Cannot be interrupted while waiting for the lock
- No way to query lock state
- Always uses a single condition (requires `wait()`/`notify()` for conditional waits)

---

### 11.3 `ReentrantLock`

An explicit lock with more control than `synchronized`. **Always unlock in a `finally` block.**

```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock(); // MUST be in finally
    }
}

// Try to acquire without blocking
if (lock.tryLock()) {
    try { count++; } finally { lock.unlock(); }
} else {
    // Lock not available, do something else
}

// Try to acquire with timeout
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try { count++; } finally { lock.unlock(); }
}

// Acquire in a way that can be interrupted (useful for cancellable tasks)
lock.lockInterruptibly();

// Fair lock: threads acquire in the order they requested the lock
ReentrantLock fairLock = new ReentrantLock(true /* fair */);
```

#### `Condition` — Fine-Grained Wait/Signal

`Condition` replaces `Object.wait()`/`notify()` when using `ReentrantLock`:

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notEmpty = lock.newCondition();
private final Condition notFull  = lock.newCondition();
private final Queue<Item> buffer = new ArrayDeque<>();
private final int capacity;

public void produce(Item item) throws InterruptedException {
    lock.lock();
    try {
        while (buffer.size() == capacity) {
            notFull.await(); // releases lock and waits
        }
        buffer.add(item);
        notEmpty.signal(); // wake one consumer
    } finally {
        lock.unlock();
    }
}

public Item consume() throws InterruptedException {
    lock.lock();
    try {
        while (buffer.isEmpty()) {
            notEmpty.await();
        }
        Item item = buffer.poll();
        notFull.signal(); // wake one producer
        return item;
    } finally {
        lock.unlock();
    }
}
```

---

### 11.4 `ReentrantReadWriteLock`

Allows **multiple concurrent readers** or **one exclusive writer**. Ideal when reads are frequent and writes are rare.

```java
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Lock readLock  = rwLock.readLock();
private final Lock writeLock = rwLock.writeLock();
private Map<String, User> cache = new HashMap<>();

public User getUser(String id) {
    readLock.lock();
    try {
        return cache.get(id); // many threads can read concurrently
    } finally {
        readLock.unlock();
    }
}

public void updateUser(String id, User user) {
    writeLock.lock();
    try {
        cache.put(id, user); // exclusive write access
    } finally {
        writeLock.unlock();
    }
}
```

**When to use**: Read-heavy, write-rare scenarios (caches, configuration, lookup tables).

**Caveats**:
- Write lock acquisition is blocked until ALL readers release.
- Can lead to **writer starvation** if reads are constant (fairness mode helps).
- Lock downgrading (write → read) is supported; lock upgrading (read → write) is NOT.

---

### 11.5 `StampedLock` (Java 8 / Android API 26+)

`StampedLock` was introduced in Java 8. On Android, it is available natively from **API 26 (Android 8.0)**. With Android Gradle Plugin's core library desugaring enabled, it can be used from **API 24 (Android 7.0)**.

`StampedLock` offers three modes and is designed for **high-throughput, read-heavy workloads**:

1. **Write lock**: Exclusive, like `writeLock` in `ReadWriteLock`.
2. **Read lock**: Shared, like `readLock` in `ReadWriteLock`.
3. **Optimistic read**: **No actual lock acquired** — just checks if a write occurred. Fastest, but must validate.

```java
private final StampedLock sl = new StampedLock();
private double x, y; // point coordinates

// Write
public void movePoint(double dx, double dy) {
    long stamp = sl.writeLock();
    try {
        x += dx;
        y += dy;
    } finally {
        sl.unlockWrite(stamp);
    }
}

// Optimistic read (no lock acquired — fastest path)
public double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead(); // returns a stamp, no blocking
    double currentX = x, currentY = y;
    if (!sl.validate(stamp)) {
        // A write occurred while we were reading — fall back to real read lock
        stamp = sl.readLock();
        try {
            currentX = x;
            currentY = y;
        } finally {
            sl.unlockRead(stamp);
        }
    }
    return Math.sqrt(currentX * currentX + currentY * currentY);
}
```

**Important**: `StampedLock` is **NOT reentrant**. Calling `writeLock()` from a thread that already holds the write lock will deadlock.

---

### 11.6 Decision Guide

```
Simple mutual exclusion, no advanced needs?
    └─► synchronized

Need tryLock, timeout, interruptibility, or multiple conditions?
    └─► ReentrantLock

Many more reads than writes?
    └─► ReentrantReadWriteLock

Maximum performance, read-heavy, low contention, API 26+ (or API 24+ with desugaring)?
    └─► StampedLock (with optimistic reads)
```

---

### 11.7 Deadlock

A deadlock occurs when two or more threads are **permanently blocked waiting for each other** to release locks.

#### Four Necessary Conditions (Coffman Conditions)
1. **Mutual exclusion**: Resources cannot be shared.
2. **Hold and wait**: A thread holds a lock and waits for another.
3. **No preemption**: Locks cannot be forcibly taken.
4. **Circular wait**: Thread A waits for Thread B's lock while Thread B waits for Thread A's lock.

#### Classic Example

```java
// Thread 1                      // Thread 2
synchronized (lockA) {           synchronized (lockB) {
    synchronized (lockB) {           synchronized (lockA) {
        // work                          // work — DEADLOCK!
    }                                }
}                                }
```

#### Prevention Strategies

1. **Lock ordering**: Always acquire locks in the same global order (e.g., alphabetical by lock name).
2. **Try-lock with timeout**: Use `ReentrantLock.tryLock(timeout)` and back off on failure.
3. **Lock-free design**: Use `Atomic` classes or `ConcurrentHashMap` where possible.
4. **Single lock**: Minimize lock scope to avoid needing multiple locks simultaneously.

---

## 12. Executors and `ThreadPoolExecutor`

### 12.1 The Executor Framework

The `java.util.concurrent` Executor framework separates **task submission** from **task execution**, allowing you to decouple your application logic from thread management.

```
Runnable/Callable  →  ExecutorService  →  Thread Pool  →  OS Threads
   (your task)          (manages                (executes
                          lifecycle)              tasks)
```

#### The Hierarchy

```
Executor
    └── ExecutorService
            ├── AbstractExecutorService
            │       └── ThreadPoolExecutor
            │               └── ScheduledThreadPoolExecutor
            └── ForkJoinPool
```

---

### 12.2 `ExecutorService` Lifecycle

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit tasks
executor.execute(runnable);                      // fire-and-forget
Future<Result> f = executor.submit(callable);    // returns Future
Future<?> f2 = executor.submit(runnable);        // submit Runnable as Future
List<Future<R>> futures = executor.invokeAll(callableList);    // submit all, wait all
R result = executor.invokeAny(callableList);                   // first completed wins

// Shutdown
executor.shutdown();        // no new tasks; existing tasks complete
executor.shutdownNow();     // attempts to interrupt running tasks; returns pending tasks
boolean done = executor.awaitTermination(5, TimeUnit.SECONDS); // wait for completion
```

---

### 12.3 `Executors` Factory Methods and Their Pitfalls

```java
// Fixed-size pool — N threads, UNLIMITED LinkedBlockingQueue
// ⚠️ Risk: queue can grow unbounded under load → OOM
ExecutorService fixed = Executors.newFixedThreadPool(4);

// Single-thread executor — serial execution, UNLIMITED queue
// ⚠️ Same risk as fixedThreadPool(1)
ExecutorService single = Executors.newSingleThreadExecutor();

// Cached pool — grows to UNLIMITED threads, 60s idle expiry
// ⚠️ Risk: can create thousands of threads under load → OOM / thrashing
ExecutorService cached = Executors.newCachedThreadPool();

// Scheduled pool — for delayed / periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.schedule(task, 1, TimeUnit.SECONDS);
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);
scheduled.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);
```

> ⚠️ **In production Android code, prefer constructing `ThreadPoolExecutor` directly** with explicit, bounded parameters rather than using the `Executors` factory methods, which can create unbounded queues or unbounded thread counts.

---

### 12.4 `ThreadPoolExecutor` — The Core Class

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,           // min threads kept alive
    maximumPoolSize,        // max threads allowed
    keepAliveTime,          // how long idle non-core threads live
    TimeUnit.SECONDS,
    workQueue,              // where tasks wait when all core threads are busy
    threadFactory,          // creates threads (for naming, priority, daemon)
    rejectedExecutionHandler // what to do when queue is full AND max threads reached
);
```

#### Task Submission Flow

```
New task submitted via execute() or submit()
       │
       ▼
Pool thread count < corePoolSize?
  YES → Start a NEW core thread to run this task immediately
        (even if other core threads are currently idle)
  NO  ↓
       ▼
Work queue not full? (queue.offer(task) succeeds)
  YES → Enqueue task — it waits until a thread becomes free
  NO  ↓
       ▼
Pool thread count < maximumPoolSize?
  YES → Start a NEW non-core (overflow) thread to run this task
  NO  ↓
       ▼
RejectedExecutionHandler.rejectedExecution() invoked
```

> **Key nuance**: `ThreadPoolExecutor` will create a new **core** thread even if existing core threads are idle. It only starts queueing once `corePoolSize` is reached. Non-core threads are only created when the queue is also full.

#### Work Queue Types

| Queue | Behavior |
|---|---|
| `SynchronousQueue` | No storage — task must be picked up by a thread immediately. Forces creation of new threads up to `maximumPoolSize`. |
| `LinkedBlockingQueue(n)` | Bounded FIFO queue of capacity n. Core threads fill up, queue fills up, then non-core threads start. |
| `ArrayBlockingQueue(n)` | Bounded, array-backed FIFO. Similar to `LinkedBlockingQueue` but uses a single lock. |
| `PriorityBlockingQueue` | Unbounded priority queue. Tasks execute in priority order. |

#### `ThreadFactory`

```java
ThreadFactory factory = new ThreadFactory() {
    private final AtomicInteger count = new AtomicInteger(0);

    @Override
    public Thread newThread(@NonNull Runnable r) {
        Thread t = new Thread(r, "MyPool-worker-" + count.incrementAndGet());
        t.setPriority(Thread.NORM_PRIORITY);
        t.setDaemon(false); // app won't exit while worker threads are running
        return t;
    }
};
```

#### `RejectedExecutionHandler` Policies

| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | **The calling thread runs the task itself.** Provides backpressure — slows down the producer naturally. |
| `DiscardPolicy` | Silently drops the task |
| `DiscardOldestPolicy` | Drops the oldest waiting task, then retries submission |

---

## 13. Efficient ThreadPool for CPU-Intensive Tasks

### 13.1 Characteristics of CPU-Bound Work

CPU-bound tasks keep the CPU busy throughout their execution:
- Image decoding / encoding (BitmapFactory, libjpeg)
- Video frame processing
- JSON / XML parsing of large documents
- Sorting and searching large datasets
- Cryptographic operations (AES, RSA, hashing)
- Mathematical computations (physics simulations, ML inference)

**Key insight**: A CPU-bound thread is **always runnable** (never waiting). Having more threads than CPU cores just adds context-switching overhead without increasing throughput.

---

### 13.2 Optimal Core Count

```java
int cpuCount = Runtime.getRuntime().availableProcessors();

// For pure CPU work: threads = cores (no idle time, no benefit from extra threads)
int corePoolSize    = cpuCount;
int maximumPoolSize = cpuCount;

// Some use cpuCount + 1 to absorb occasional page faults / GC pauses
// int maximumPoolSize = cpuCount + 1;
```

---

### 13.3 Building the CPU ThreadPool

```java
public static ThreadPoolExecutor createCpuBoundPool() {
    int cpuCount = Runtime.getRuntime().availableProcessors();

    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        cpuCount,                                   // corePoolSize
        cpuCount,                                   // maximumPoolSize (same as core for CPU-bound)
        0L,                                         // keepAliveTime: 0 = core threads never die
        TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<>(cpuCount * 2),    // small bounded queue
        new PriorityThreadFactory(                  // named, background priority
            "cpu-pool",
            Process.THREAD_PRIORITY_BACKGROUND),
        new ThreadPoolExecutor.CallerRunsPolicy()   // backpressure when full
    );

    // Allow core threads to time out too (optional, saves memory when idle)
    // executor.allowCoreThreadTimeOut(true);

    return executor;
}

// ThreadFactory with Android process priority support
public static class PriorityThreadFactory implements ThreadFactory {
    private final String namePrefix;
    private final int androidPriority;
    private final AtomicInteger count = new AtomicInteger(1);

    public PriorityThreadFactory(String namePrefix, int androidPriority) {
        this.namePrefix = namePrefix;
        this.androidPriority = androidPriority;
    }

    @Override
    public Thread newThread(@NonNull Runnable r) {
        return new Thread(() -> {
            // Set Android process-level thread priority (different from Java thread priority)
            Process.setThreadPriority(androidPriority);
            r.run();
        }, namePrefix + "-" + count.getAndIncrement());
    }
}
```

---

### 13.4 Android Thread Priority

Android has its own **Linux nice-value** based thread priority system exposed via `android.os.Process`. This is **separate from and independent of** Java's `Thread.setPriority()` (which maps to Java priority levels 1–10).

| Priority Constant | Linux Nice Value | Used By |
|---|---|---|
| `THREAD_PRIORITY_URGENT_DISPLAY` | -8 | SurfaceFlinger binder thread, input reader — system-level only |
| `THREAD_PRIORITY_DISPLAY` | -4 | **RenderThread** (hardware-accelerated drawing thread) |
| `THREAD_PRIORITY_URGENT_AUDIO` | -16 | Audio playback threads |
| `THREAD_PRIORITY_AUDIO` | -16 | Audio processing |
| `THREAD_PRIORITY_DEFAULT` | 0 | **Main (UI) Thread**, normal threads |
| `THREAD_PRIORITY_FOREGROUND` | -2 | Foreground app worker threads |
| `THREAD_PRIORITY_BACKGROUND` | 10 | **Recommended for background workers** |
| `THREAD_PRIORITY_LOWEST` | 19 | Minimum priority (idle tasks) |

> ⚠️ **Common misconception**: The **Main Thread** runs at `THREAD_PRIORITY_DEFAULT` (nice 0), the same as any regular Java thread. It is NOT elevated. Frame rendering is prioritized through the **sync barrier** mechanism in the `MessageQueue` (see Section 9.4), not by giving the Main Thread a higher OS priority. The **RenderThread** (which actually executes GPU drawing commands) runs at `THREAD_PRIORITY_DISPLAY` (-4).

Setting `THREAD_PRIORITY_BACKGROUND` (nice 10) on your worker threads ensures the Linux scheduler **yields CPU time** to the Main Thread and RenderThread whenever they have work to do, preventing your background tasks from causing jank.

---

### 13.5 Using `ForkJoinPool` for Divide-and-Conquer (API 21+)

For **recursive, divide-and-conquer** tasks (e.g., parallel merge sort, parallel image processing), `ForkJoinPool` uses a **work-stealing** algorithm where idle threads steal tasks from busy threads' queues:

```java
// Use the common pool (shared across the JVM process)
ForkJoinPool pool = ForkJoinPool.commonPool();

// Custom ForkJoinPool
ForkJoinPool customPool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors(),
    ForkJoinPool.defaultForkJoinWorkerThreadFactory,
    null,    // uncaught exception handler
    false    // asyncMode: false = LIFO (better for recursive tasks)
);

// Example: parallel merge sort using RecursiveAction
class SortTask extends RecursiveAction {
    private final int[] array;
    private final int from, to;
    private static final int THRESHOLD = 1000;

    @Override
    protected void compute() {
        if (to - from < THRESHOLD) {
            Arrays.sort(array, from, to);
        } else {
            int mid = (from + to) / 2;
            SortTask left  = new SortTask(array, from, mid);
            SortTask right = new SortTask(array, mid, to);
            invokeAll(left, right);
            merge(array, from, mid, to);
        }
    }
}

pool.invoke(new SortTask(array, 0, array.length));
```

---

### 13.6 Complete CPU Pool Example with Monitoring

```java
public class CpuTaskManager {
    private static final String TAG = "CpuTaskManager";
    private final ThreadPoolExecutor executor;

    public CpuTaskManager() {
        int cpuCount = Runtime.getRuntime().availableProcessors();
        int queueCapacity = cpuCount * 2;

        executor = new ThreadPoolExecutor(
            cpuCount,
            cpuCount,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>(queueCapacity),
            new PriorityThreadFactory("cpu", Process.THREAD_PRIORITY_BACKGROUND),
            (r, exec) -> {
                // Custom rejection: log + CallerRunsPolicy
                Log.w(TAG, "CPU pool saturated! Queue: " + exec.getQueue().size());
                if (!exec.isShutdown()) {
                    r.run(); // caller runs task (backpressure)
                }
            }
        );
    }

    public <T> Future<T> submit(Callable<T> task) {
        return executor.submit(task);
    }

    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    public void logStats() {
        Log.d(TAG, String.format(
            "Pool[active=%d, pool=%d, queue=%d, completed=%d]",
            executor.getActiveCount(),
            executor.getPoolSize(),
            executor.getQueue().size(),
            executor.getCompletedTaskCount()
        ));
    }
}
```

---

## 14. Efficient ThreadPool for IO-Intensive Tasks

### 14.1 Characteristics of I/O-Bound Work

I/O-bound tasks spend most of their time **blocked waiting** for external operations:
- Network requests (REST APIs, WebSockets)
- File reads/writes (large files, log files)
- Database queries (Room, SQLite)
- Content Provider queries
- IPC (Inter-Process Communication)

**Key insight**: When an I/O thread is blocked waiting for data, the CPU is idle. We can profitably run **many more threads than CPU cores** to keep the CPU busy while some threads wait.

---

### 14.2 Optimal Thread Count Formula

```
Optimal Threads = N_cpu × (1 + W/C)

Where:
  N_cpu = number of CPU cores
  W     = average wait time per task (blocked on I/O)
  C     = average compute time per task (CPU active)
```

For typical mobile network I/O where `W/C ≈ 10–50`:

```java
int cpuCount = Runtime.getRuntime().availableProcessors();

// Conservative estimate: 4 cores × (1 + 10) = 44 threads for I/O
// On mobile, cap at a reasonable limit to avoid memory pressure
int ioThreadCount = Math.min(cpuCount * 4, 32); // e.g., 16–32
```

Each idle thread consumes ~512 KB–1 MB of stack memory. On Android with limited RAM, cap your I/O thread count to **16–32** unless you have profiling data to justify more.

---

### 14.3 Building the I/O ThreadPool

```java
public static ThreadPoolExecutor createIoBoundPool() {
    int cpuCount = Runtime.getRuntime().availableProcessors();
    int coreThreads = cpuCount * 2;       // always-alive threads for sustained I/O
    int maxThreads  = cpuCount * 4;       // burst capacity, capped for mobile
    long keepAlive  = 60L;                // idle non-core threads die after 60s

    // Larger queue to absorb bursts of I/O tasks
    BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<>(128);

    return new ThreadPoolExecutor(
        coreThreads,
        maxThreads,
        keepAlive,
        TimeUnit.SECONDS,
        workQueue,
        new PriorityThreadFactory("io-pool", Process.THREAD_PRIORITY_BACKGROUND),
        new ThreadPoolExecutor.CallerRunsPolicy()   // backpressure
    );
}
```

---

### 14.4 Key Differences from CPU Pool

| Parameter | CPU-Bound Pool | I/O-Bound Pool |
|---|---|---|
| `corePoolSize` | `N_cpu` | `N_cpu × 2` |
| `maximumPoolSize` | `N_cpu` (or `N_cpu + 1`) | `N_cpu × 4` (capped at 32) |
| `keepAliveTime` | 0 (core threads never die) | 30–60 seconds (let idle threads expire) |
| `workQueue` capacity | Small (2×N_cpu) | Medium-large (64–256) |
| Thread priority | `THREAD_PRIORITY_BACKGROUND` | `THREAD_PRIORITY_BACKGROUND` |
| `RejectedExecutionHandler` | `CallerRunsPolicy` | `CallerRunsPolicy` |

---

### 14.5 Complete I/O Pool Example

```java
public class IoTaskManager {
    private static final String TAG = "IoTaskManager";
    private final ThreadPoolExecutor executor;

    public IoTaskManager() {
        int cpuCount = Runtime.getRuntime().availableProcessors();
        int corePoolSize = cpuCount * 2;
        int maxPoolSize  = Math.min(cpuCount * 4, 32);

        executor = new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L, TimeUnit.SECONDS,                   // idle threads expire after 60s
            new LinkedBlockingQueue<>(128),           // buffer up to 128 pending tasks
            new PriorityThreadFactory("io", Process.THREAD_PRIORITY_BACKGROUND),
            (r, exec) -> {
                Log.w(TAG, "I/O pool saturated! Applying backpressure.");
                if (!exec.isShutdown()) {
                    r.run(); // CallerRunsPolicy: submit thread executes the task
                }
            }
        );

        // Allow core threads to also expire when idle (reduces memory usage)
        executor.allowCoreThreadTimeOut(true);
    }

    public <T> Future<T> submit(Callable<T> task) {
        return executor.submit(task);
    }

    // Convenience: submit I/O task and deliver result on Main Thread
    public <T> void submitWithCallback(Callable<T> task, Consumer<T> onSuccess,
                                       Consumer<Throwable> onError) {
        Handler mainHandler = new Handler(Looper.getMainLooper());
        executor.execute(() -> {
            try {
                T result = task.call();
                mainHandler.post(() -> onSuccess.accept(result));
            } catch (Exception e) {
                mainHandler.post(() -> onError.accept(e));
            }
        });
    }

    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                List<Runnable> pending = executor.shutdownNow();
                Log.w(TAG, "Abandoned " + pending.size() + " pending I/O tasks");
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

---

### 14.6 Example: Network I/O with the I/O Pool

```java
public class UserRepository {
    private final IoTaskManager ioTaskManager;
    private final ApiService apiService;

    public void fetchUser(String userId, Consumer<User> onSuccess, Consumer<Throwable> onError) {
        ioTaskManager.submitWithCallback(
            () -> apiService.getUser(userId),   // Callable: blocking network call
            onSuccess,                           // called on Main Thread
            onError                              // called on Main Thread
        );
    }
}

// Usage in an Activity:
userRepository.fetchUser("user123",
    user -> nameTextView.setText(user.getName()),       // Main Thread: update UI
    error -> showErrorToast(error.getMessage())         // Main Thread: show error
);
```

---

### 14.7 Memory Considerations on Android

Each thread allocates a stack:
- Default Java stack: ~512 KB–1 MB
- 32 threads × 512 KB = **16 MB** just for stacks

On devices with 2 GB RAM where your app might have 256–512 MB heap limit, 16 MB for thread stacks is significant. Profile and tune your thread counts accordingly.

```java
// Log memory impact
long threadCount = Thread.activeCount();
long estimatedStackMb = threadCount * 512 / 1024;
Log.d(TAG, "Active threads: " + threadCount + " (~" + estimatedStackMb + " MB stacks)");
```

---

## Summary: Quick Reference

| Component | Purpose | Android API |
|---|---|---|
| `Thread` | Basic concurrency unit | Java |
| `Runnable` | Task with no return value | Java |
| `Callable<V>` | Task with return value + checked exception | Java |
| `Future<V>` | Handle to async result; cancel/get | Java |
| `Handler` | Post messages/runnables to a Looper | Android |
| `Looper` | Message loop for a thread | Android |
| `HandlerThread` | Thread + Looper, ready to use | Android |
| `volatile` | Visibility guarantee (no atomicity) | Java |
| `synchronized` | Mutual exclusion + visibility | Java |
| `AtomicInteger` | Lock-free integer ops | Java |
| `AtomicBoolean` | Lock-free boolean flag | Java |
| `AtomicReference<V>` | Lock-free object reference swap | Java |
| `ReentrantLock` | Explicit lock with tryLock/timeout | Java |
| `ReentrantReadWriteLock` | Multi-reader / single-writer | Java |
| `StampedLock` | Optimistic read, high-throughput | Java 8 / Android API 26+ |
| `CountDownLatch` | Wait for N events | Java |
| `CyclicBarrier` | Rendezvous point for N threads | Java |
| `Semaphore` | Resource pool access control | Java |
| `BlockingQueue` | Thread-safe producer-consumer queue | Java |
| `ThreadPoolExecutor` | Configurable thread pool | Java |
| `ForkJoinPool` | Work-stealing pool for divide-and-conquer | Java (API 21+) |
| `Choreographer` | VSYNC frame scheduling | Android |
| `MessageQueue` | Ordered message store for a Looper | Android |

---

*This document covers the foundation of multithreading in Android with Java. For modern Android development, consider also exploring Kotlin Coroutines and Flow, which build on these primitives to provide structured concurrency with cancellation and lifecycle-awareness built in.*

