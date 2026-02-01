# Complete Guide to Python Async Programming

> From Synchronous to Asynchronous: Deep Dive into Event Loop, Coroutines & High-Concurrency IO

---

## 📚 Table of Contents

1. [Theoretical Foundations](#1-theoretical-foundations)
2. [Code Implementation](#2-code-implementation)
3. [Event Loop Mechanism Diagrams](#3-event-loop-mechanism-diagrams)
4. [Performance Comparison Analysis](#4-performance-comparison-analysis)
5. [Multithreading vs Async: In-Depth Comparison](#5-multithreading-vs-async-in-depth-comparison)

---

## 1. Theoretical Foundations

### 1.1 From `yield` to `await`: The Evolution of Coroutines

`await` is essentially syntactic sugar for Python's coroutine mechanism, evolved from `yield` and `send`.

```python
# Generator - The prototype of coroutines
def simple_generator():
    print("Starting execution")
    x = yield 1          # Pause, return 1, wait for external send value
    print(f"Received: {x}")
    y = yield 2          # Pause, return 2
    print(f"Received: {y}")

# Usage
gen = simple_generator()
print(next(gen))         # Output: Starting execution → 1
print(gen.send(10))      # Output: Received: 10 → 2
```

**Key Understanding**:
- `yield` allows functions to **pause** and **preserve state**
- `send()` can **inject data** into the pause point
- This "pause-resume" mechanism is the core of coroutines

### 1.2 The Role of Event Loop

The Event Loop is the **scheduling center** of async programming:

```
┌─────────────────────────────────────────────────────────┐
│                     Event Loop                          │
│                                                         │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│   │ Task Queue  │  │ Pending IO  │  │  Callbacks  │    │
│   │(Ready Tasks)│  │(Waiting IO) │  │ (Callback   │    │
│   │             │  │             │  │  Functions) │    │
│   └─────────────┘  └─────────────┘  └─────────────┘    │
│         │                │                 │            │
│         ▼                ▼                 ▼            │
│   ┌───────────────────────────────────────────────┐    │
│   │           Select / Epoll System Call           │    │
│   │    (Monitor IO status of multiple file         │    │
│   │              descriptors)                      │    │
│   └───────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 1.3 What Happens at `await`?

When code executes `await client.get(url)`:

1. **Current coroutine pauses** → State saved to stack frame
2. **IO request registered** → Tell OS: "Notify me when this socket has data"
3. **Control returns to Event Loop** → Loop executes other ready tasks
4. **IO completes** → OS notifies Loop, Loop resumes the coroutine
5. **Coroutine continues** → Execution continues after `await`

---

## 2. Code Implementation

### 2.1 Synchronous Version (using requests)

```python
"""
Synchronous HTTP Request Benchmark Test
Using requests library for serial URL access
"""
import requests
import time


def timer(times: int = 20):
    """
    Timing decorator
    
    Args:
        times: Number of repetitions, default 20
    
    Returns:
        Decorated function that prints execution time and total time
    """
    def decorator(func):
        def wrapper(*args, **kwargs):
            total_start = time.time()
            result = None
            
            for i in range(times):
                start = time.time()
                result = func(*args, **kwargs)
                end = time.time()
                print(f"Request {i + 1:2d} | Time: {end - start:.4f}s")
            
            total_end = time.time()
            print(f"\n{'='*40}")
            print(f"Sync Mode | Total Requests: {times} | Total Time: {total_end - total_start:.2f}s")
            return result
        return wrapper
    return decorator


def run_sync(url: str, times: int = 20) -> None:
    """
    Execute multiple HTTP GET requests synchronously
    
    Characteristics: Serial execution, each request must wait for the previous one
    """
    @timer(times)
    def fetch(target_url: str) -> requests.Response:
        return requests.get(target_url)
    
    fetch(url)


if __name__ == '__main__':
    # httpbin.org/delay/1 forces 1 second delay, simulating high-latency scenario
    run_sync("https://httpbin.org/delay/1", times=20)
```

### 2.2 Asynchronous Version (using httpx)

```python
"""
Asynchronous HTTP Request Benchmark Test
Using httpx.AsyncClient for concurrent URL access
"""
import asyncio
import httpx
import time
from functools import wraps


def async_timer(func):
    """
    Async function timing decorator
    
    Precisely measures async function execution time
    """
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        result = await func(*args, **kwargs)
        end = time.time()
        print(f"Total time: {end - start:.2f}s")
        return result
    return wrapper


async def fetch_url(client: httpx.AsyncClient, url: str, task_id: int) -> None:
    """
    Single async request task
    
    Args:
        client: Reused AsyncClient instance (key optimization!)
        url: Target URL
        task_id: Task number for tracking
    """
    start = time.time()
    print(f"Task {task_id:2d} | Starting request...")
    
    # Key point: await pauses here, control returns to Event Loop
    response = await client.get(url)
    
    end = time.time()
    print(f"Task {task_id:2d} | Done | Time: {end - start:.4f}s | Status: {response.status_code}")


@async_timer
async def run_async(url: str, times: int = 20) -> None:
    """
    Execute multiple HTTP GET requests asynchronously
    
    Key optimizations:
    1. Reuse AsyncClient (connection pool, keep-alive)
    2. asyncio.gather() for concurrent task scheduling
    """
    # Use async with to ensure connections are properly closed
    async with httpx.AsyncClient() as client:
        # Create task list
        tasks = [
            fetch_url(client, url, i + 1) 
            for i in range(times)
        ]
        # gather() executes all tasks concurrently
        await asyncio.gather(*tasks)


if __name__ == '__main__':
    print("="*50)
    print("Async Mode | Concurrent Request Test")
    print("="*50)
    asyncio.run(run_async("https://httpbin.org/delay/1", times=20))
```

### 2.3 Complete Comparison Script

```python
"""
Sync vs Async Complete Comparison Script
Includes multi-scenario testing and performance analysis
"""
import asyncio
import httpx
import requests
import time
from dataclasses import dataclass
from typing import Callable


@dataclass
class BenchmarkResult:
    """Benchmark test result"""
    mode: str
    requests_count: int
    total_time: float
    avg_time: float


def benchmark_sync(url: str, count: int) -> BenchmarkResult:
    """Synchronous benchmark test"""
    print(f"\n{'='*50}")
    print(f"Sync Mode | {count} requests")
    print('='*50)
    
    start = time.time()
    for i in range(count):
        req_start = time.time()
        requests.get(url)
        req_end = time.time()
        print(f"  Request {i+1:2d} | {req_end - req_start:.3f}s")
    
    total = time.time() - start
    return BenchmarkResult("Sync", count, total, total / count)


async def benchmark_async(url: str, count: int) -> BenchmarkResult:
    """Asynchronous benchmark test"""
    print(f"\n{'='*50}")
    print(f"Async Mode | {count} concurrent requests")
    print('='*50)
    
    async def fetch(client, idx):
        req_start = time.time()
        await client.get(url)
        print(f"  Task {idx+1:2d} | {time.time() - req_start:.3f}s")
    
    start = time.time()
    async with httpx.AsyncClient() as client:
        await asyncio.gather(*[fetch(client, i) for i in range(count)])
    
    total = time.time() - start
    return BenchmarkResult("Async", count, total, total / count)


def run_benchmark():
    """Run complete benchmark test"""
    url = "https://httpbin.org/delay/1"
    results = []
    
    # Scenario A: Low load (5 requests)
    print("\n" + "🔬 Scenario A: Low Load Test (5 requests)".center(50))
    results.append(benchmark_sync(url, 5))
    results.append(asyncio.run(benchmark_async(url, 5)))
    
    # Scenario B: Simulated LLM scenario (20 requests)
    print("\n" + "🔬 Scenario B: High Load Test (20 requests)".center(50))
    results.append(benchmark_sync(url, 20))
    results.append(asyncio.run(benchmark_async(url, 20)))
    
    # Summary report
    print("\n" + "="*60)
    print("📊 Performance Comparison Summary")
    print("="*60)
    print(f"{'Mode':<8} | {'Count':<6} | {'Total':>10} | {'Average':>10} | Speedup")
    print("-"*60)
    
    for i in range(0, len(results), 2):
        sync_r = results[i]
        async_r = results[i+1]
        speedup = sync_r.total_time / async_r.total_time
        
        print(f"{sync_r.mode:<8} | {sync_r.requests_count:<6} | {sync_r.total_time:>8.2f}s | {sync_r.avg_time:>8.3f}s |")
        print(f"{async_r.mode:<8} | {async_r.requests_count:<6} | {async_r.total_time:>8.2f}s | {async_r.avg_time:>8.3f}s | {speedup:.1f}x ⚡")
        print("-"*60)


if __name__ == '__main__':
    run_benchmark()
```

---

## 3. Event Loop Mechanism Diagrams

### 3.1 Core Mechanism Diagram

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                      EVENT LOOP                              │
                    │                  (Single-Thread Scheduler)                   │
                    └─────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                          │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              TASK QUEUE                                          │   │
│   │                                                                                  │   │
│   │    ┌──────────┐    ┌──────────┐    ┌──────────┐                                 │   │
│   │    │  Task 1  │    │  Task 2  │    │  Task 3  │    ...                          │   │
│   │    │ (READY)  │    │ (READY)  │    │ (READY)  │                                 │   │
│   │    └──────────┘    └──────────┘    └──────────┘                                 │   │
│   │                                                                                  │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                                │
│                                         ▼                                                │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                     Execute Task 1 until await is encountered                    │   │
│   │                                                                                  │   │
│   │    async def fetch(url):                                                         │   │
│   │        print("Starting request")   ◀─── Execute immediately                      │   │
│   │        resp = await client.get()   ◀─── 🔴 Pause here!                          │   │
│   │        print("Request complete")   ◀─── Resume after IO completes               │   │
│   │                                                                                  │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                                │
│                     ┌───────────────────┴───────────────────┐                           │
│                     │                                       │                           │
│                     ▼                                       ▼                           │
│   ┌──────────────────────────────────┐    ┌──────────────────────────────────┐         │
│   │     Task 1 → PENDING state        │    │    Continue executing Task 2, 3...│         │
│   │                                   │    │                                  │         │
│   │   ┌─────────────────────────┐    │    │   Loop doesn't wait idly for     │         │
│   │   │  Save coroutine state   │    │    │   Task 1's IO, but immediately   │         │
│   │   │  to heap (local vars,   │    │    │   executes other ready tasks     │         │
│   │   │  execution position)    │    │    │                                  │         │
│   │   └─────────────────────────┘    │    └──────────────────────────────────┘         │
│   │                                   │                                                 │
│   │   ┌─────────────────────────┐    │                                                 │
│   │   │   Register IO with      │    │                                                 │
│   │   │   Selector              │    │                                                 │
│   │   │  (epoll/kqueue/select)  │    │                                                 │
│   │   └─────────────────────────┘    │                                                 │
│   └──────────────────────────────────┘                                                 │
│                         │                                                               │
└─────────────────────────┼───────────────────────────────────────────────────────────────┘
                          │
                          ▼
          ┌───────────────────────────────────────┐
          │         OPERATING SYSTEM              │
          │                                       │
          │   ┌─────────────────────────────┐    │
          │   │     Epoll / Kqueue / IOCP    │    │
          │   │                             │    │
          │   │  Monitor IO status of       │    │
          │   │  multiple sockets           │    │
          │   │  Notify Loop when data      │    │
          │   │  arrives                    │    │
          │   └─────────────────────────────┘    │
          └───────────────────────────────────────┘
                          │
                          │ IO completion notification
                          ▼
          ┌───────────────────────────────────────┐
          │   Task 1 rejoins READY queue          │
          │   Waiting for Loop to schedule        │
          │   and resume execution                │
          └───────────────────────────────────────┘
```

### 3.2 Concurrent Timeline of Three HTTP Requests

```
Timeline →  0s        0.01s      0.02s      0.03s          1.0s       1.01s      1.02s
            │          │          │          │              │          │          │
            ▼          ▼          ▼          ▼              ▼          ▼          ▼

Task 1:     ████████▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓████████
            │Execute│    │       Waiting IO (PENDING)       │    │Resume exec│
            │ code  │await                                  │IO done│ rest code│
            └───────┘    └──────────────────────────────────┘    └──────────┘

Task 2:              ████████▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓████████
                     │Execute│    │     Waiting IO (PENDING)     │    │Resume│
                     └───────┘    └──────────────────────────────┘    └──────┘

Task 3:                       ████████▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓████████
                              │Execute│    │   Waiting IO (PENDING)   │    │Resume│
                              └───────┘    └──────────────────────────┘    └──────┘

Event Loop:
            ┌─────────────────────────────────────────────────────────────────────┐
  Activity: │Run T1│Run T2│Run T3│  idle (waiting IO)  │Resume T1│T2│T3│  Done   │
            └─────────────────────────────────────────────────────────────────────┘

Legend:
  ████  = CPU executing code
  ▓▓▓   = await point (pause/resume)
  ░░░   = IO waiting (not using CPU)
  idle  = Event Loop calls epoll_wait(), CPU sleeps

Key Insights:
  • IO wait time of all three requests **completely overlaps**
  • Event Loop finishes sending all requests by 0.03s
  • Total time ≈ max(IO latency) + minimal scheduling overhead ≈ 1.02s
  • NOT sum(IO latency) = 3s
```

### 3.3 The Role of Epoll

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          HOW EPOLL WORKS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Python Code Layer                                                         │
│   ─────────────────                                                         │
│   await client.get(url1)  ─┐                                                │
│   await client.get(url2)  ─┼─► asyncio registers 3 sockets with epoll      │
│   await client.get(url3)  ─┘                                                │
│                                                                             │
│                             │                                               │
│                             ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     EPOLL INSTANCE                                   │  │
│   │                                                                      │  │
│   │   Interest List:                                                     │  │
│   │   ┌─────────────┬─────────────┬─────────────┐                       │  │
│   │   │ socket_fd=5 │ socket_fd=6 │ socket_fd=7 │                       │  │
│   │   │ EPOLLIN     │ EPOLLIN     │ EPOLLIN     │                       │  │
│   │   │(wait read)  │(wait read)  │(wait read)  │                       │  │
│   │   └─────────────┴─────────────┴─────────────┘                       │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                             │                                               │
│                             │ epoll_wait() - blocking wait                  │
│                             │ but only uses 1 thread!                       │
│                             ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         Kernel                                       │  │
│   │                                                                      │  │
│   │   NIC receives data → triggers interrupt → kernel wakes epoll_wait()│  │
│   │                                                                      │  │
│   │   Ready List:                                                        │  │
│   │   ┌─────────────┐                                                   │  │
│   │   │ socket_fd=6 │  ← "socket 6 has data ready to read!"             │  │
│   │   └─────────────┘                                                   │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                             │                                               │
│                             │ Returns list of ready fds                     │
│                             ▼                                               │
│   Event Loop receives notification:                                         │
│   "Oh! Task 2's IO is complete, add it back to ready queue!"               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Why is Epoll Efficient?

  Traditional select/poll:  Every call copies **all** fds from user space to kernel
                           Kernel must **traverse** all fds to check status
                           Time complexity: O(n)

  epoll:                   fds copied only once during registration (epoll_ctl)
                           Kernel uses callback mechanism, returns only **ready** fds
                           Time complexity: O(1) for ready events
```

---

## 4. Performance Comparison Analysis

### 4.1 Test Environment

| Item | Configuration |
|------|---------------|
| Test URL | `https://httpbin.org/delay/1` (forced 1 second delay) |
| Python Version | 3.10+ |
| Sync Library | `requests` |
| Async Library | `httpx` |

### 4.2 Test Results

#### Scenario A: Low Load (5 requests)

| Mode | Total Time | Avg Per Request | Speedup |
|------|------------|-----------------|---------|
| **Sync** | ~5.5s | ~1.1s | 1x |
| **Async** | ~1.2s | ~0.24s | **4.6x** |

#### Scenario B: Simulated LLM Scenario (20 requests)

| Mode | Total Time | Avg Per Request | Speedup |
|------|------------|-----------------|---------|
| **Sync** | ~22s | ~1.1s | 1x |
| **Async** | ~1.3s | ~0.065s | **17x** |

### 4.3 Expected vs Actual

| Scenario | Expected Sync Time | Expected Async Time | Matches Expectation? |
|----------|-------------------|---------------------|---------------------|
| 5 requests | 5s+ | ~1.x s | ✅ |
| 20 requests | 20s+ | ~1.x s | ✅ |

### 4.4 CPU Usage Observation (Scenario C)

**Experimental Observation**:
- Sync mode: CPU nearly 0% (most time spent waiting for IO)
- Async mode: CPU also nearly 0%

**Why is async 17x faster but CPU usage unchanged?**

> Because whether sync or async, the CPU's actual work time is very short (sending requests, parsing responses). **99% of time is spent waiting for network IO**.
> 
> Sync: Serial waiting, total wait time = N × IO latency
> Async: Parallel waiting, total wait time = max(IO latency)
> 
> CPU workload is the same, only the **waiting method differs**.

### 4.5 One-Sentence Summary

> **Why is async faster? Where does the speed come from?**
> 
> Async is faster because it **eliminates serial blocking of IO waits**. Sync code **blocks the entire thread** waiting for network responses on each request; async code **yields control** when encountering IO, letting the Event Loop execute other tasks. All requests' **IO wait times overlap**, with no extra thread/process creation and **context switch overhead**.
> 
> Keywords: **Blocking** → **Non-blocking**, **Serial waiting** → **Parallel waiting**, **Zero extra threads**

---

## 5. Multithreading vs Async: In-Depth Comparison

### 5.1 Can Multithreading Solve This Problem?

**Yes!** Multithreaded version:

```python
import concurrent.futures
import requests
import time

def fetch(url, task_id):
    start = time.time()
    requests.get(url)
    print(f"Task {task_id} | {time.time() - start:.2f}s")

def run_threaded(url, count):
    start = time.time()
    with concurrent.futures.ThreadPoolExecutor(max_workers=count) as executor:
        futures = [executor.submit(fetch, url, i) for i in range(count)]
        concurrent.futures.wait(futures)
    print(f"Total: {time.time() - start:.2f}s")

# 20 threads concurrent, also takes about 1.x seconds
run_threaded("https://httpbin.org/delay/1", 20)
```

### 5.2 Then Why Use Async?

| Dimension | Multithreading | Async (asyncio) |
|-----------|----------------|-----------------|
| **GIL Lock** | ⚠️ Limited by GIL, only one thread executes Python bytecode at a time | ✅ Single thread, no GIL contention |
| **Memory Overhead** | ⚠️ Each thread ~**8MB stack space** (Linux default) | ✅ Each coroutine ~**few KB** |
| **Creation Overhead** | ⚠️ Thread creation requires **system call** (clone/pthread_create) | ✅ Coroutine creation is just a **Python object** |
| **Context Switch** | ⚠️ Kernel-mode switch, **save/restore registers**, ~1-10μs | ✅ User-mode switch, only switch **stack pointer**, ~100ns |
| **Scalability** | ⚠️ 1000 threads ≈ 8GB memory | ✅ 100K coroutines ≈ few hundred MB |
| **Race Conditions** | ⚠️ Needs locks, semaphores, etc. | ✅ Single thread, no data races |
| **Debug Difficulty** | ⚠️ Concurrency bugs hard to reproduce | ✅ Deterministic execution, easy to debug |

### 5.3 Deep Understanding of GIL Lock

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GIL (Global Interpreter Lock)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Python's GIL ensures only one thread executes bytecode at a time         │
│                                                                             │
│   Multithreaded CPU-intensive tasks:                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Thread 1:  ████████░░░░░░░░████████░░░░░░░░████████                 │  │
│   │ Thread 2:  ░░░░░░░░████████░░░░░░░░████████░░░░░░░░████████         │  │
│   │                                                                      │  │
│   │ Note: Threads are **alternating execution**, not truly parallel!    │  │
│   │ Plus additional GIL acquire/release overhead                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   But for IO-intensive tasks, GIL is released during IO wait:              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Thread 1:  ██▓▓▓░░░░░░░░░░░░░░░░▓▓▓██                               │  │
│   │ Thread 2:  ░░▓▓▓██▓▓▓░░░░░░░░░░░▓▓▓░░██                             │  │
│   │ Thread 3:  ░░░░░░░░▓▓▓██▓▓▓░░░░░▓▓▓░░░░██                           │  │
│   │                                                                      │  │
│   │ ██ = Holding GIL executing code                                      │  │
│   │ ▓▓ = GIL acquire/release                                            │  │
│   │ ░░ = IO wait (GIL released)                                         │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   Conclusion: Multithreading works for IO-intensive tasks, but has overhead│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 When to Use What?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Decision Tree                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   What type is your task?                                                   │
│                │                                                            │
│                ▼                                                            │
│   ┌───────────────────────────┐                                             │
│   │  IO-intensive? CPU-intensive? │                                         │
│   └───────────────────────────┘                                             │
│          │              │                                                   │
│          ▼              ▼                                                   │
│   ┌──────────────┐ ┌──────────────┐                                        │
│   │ IO-intensive │ │CPU-intensive │                                        │
│   │(network/disk)│ │(compute/proc)│                                        │
│   └──────────────┘ └──────────────┘                                        │
│          │              │                                                   │
│          ▼              ▼                                                   │
│   ┌──────────────┐ ┌──────────────┐                                        │
│   │ Need high    │ │ Multiprocess │  ← Bypass GIL, true parallelism        │
│   │ concurrency? │ │(multiproc.)  │                                        │
│   │(1000+ conn)  │ │              │                                        │
│   └──────────────┘ └──────────────┘                                        │
│     │        │                                                              │
│     ▼        ▼                                                              │
│  ┌─────┐ ┌─────┐                                                           │
│  │ Yes │ │ No  │                                                           │
│  └─────┘ └─────┘                                                           │
│     │        │                                                              │
│     ▼        ▼                                                              │
│ ┌────────┐ ┌────────────────┐                                              │
│ │ Async  │ │ Multithreading │                                              │
│ │Coroutine│ │ also works     │                                              │
│ │ ✅     │ │(simple cases)  │                                              │
│ └────────┘ └────────────────┘                                              │
│                                                                             │
│   Summary:                                                                  │
│   • IO-intensive + high concurrency → asyncio (best choice)                │
│   • IO-intensive + low concurrency → threading/asyncio both work           │
│   • CPU-intensive → multiprocessing                                        │
│   • Mixed tasks → asyncio + ProcessPoolExecutor                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 Actual Numbers Comparison

Assuming 10,000 concurrent HTTP requests:

| Approach | Memory Usage | Creation Time | Switch Overhead |
|----------|--------------|---------------|-----------------|
| 10,000 threads | ~80 GB | Seconds | High (kernel-mode) |
| 10,000 coroutines | ~100 MB | Milliseconds | Very low (user-mode) |

---

## 📝 Key Concepts Quick Reference

| Concept | Explanation |
|---------|-------------|
| **Coroutine** | Function that can pause and resume, defined with `async def` |
| **Event Loop** | Scheduler for coroutines, decides when to run which coroutine |
| **await** | Pause current coroutine, wait for another coroutine to complete |
| **asyncio.gather()** | Run multiple coroutines concurrently |
| **Non-blocking IO** | IO operations don't block the entire thread |
| **epoll/kqueue** | OS-level IO multiplexing mechanisms |
| **GIL** | Python Global Interpreter Lock, limits multithreading parallelism |

---

## ✅ Learning Checklist

- [ ] Understand the relationship between `yield/send` and `await`
- [ ] Can draw the Event Loop execution flow
- [ ] Understand how control transfers at await
- [ ] Can explain why async IO-intensive tasks are faster
- [ ] Know when to choose multithreading vs async vs multiprocessing
- [ ] Understand GIL's impact on multithreading
- [ ] Can write correct async code (reuse Client, use gather)

---

*Notes completed: January 2025*
