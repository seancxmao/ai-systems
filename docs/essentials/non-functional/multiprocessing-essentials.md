# Multiprocessing Essentials

## 1. What Problem Does It Solve?

`multiprocessing` is Python's standard library for running multiple processes concurrently. It allows a program to utilize multiple CPU cores and isolate work into independent processes, each with its own Python interpreter and memory space.

Without `multiprocessing`, a Python application is generally limited to a single process. CPU-intensive work cannot fully utilize modern multi-core machines, and a crash or memory leak in one component can affect the entire application.

For LLM serving, the primary motivation is not parallelizing inference itself, but **building a serving runtime composed of cooperating processes**. A production serving system often separates the HTTP server, scheduler, engine, tokenizer, and other components into different processes for scalability, isolation, and resource management.

## 2. If You Come from Java

For Java developers, `multiprocessing` is conceptually similar to running multiple JVM processes that communicate through operating system primitives.

| Python / multiprocessing | Java Ecosystem                     |
| ------------------------ | ---------------------------------- |
| Process                  | JVM Process                        |
| Process                  | `java.lang.Process`                |
| Queue                    | `BlockingQueue` (conceptually)     |
| Pipe                     | Pipe / Socket                      |
| Shared Memory            | Shared Memory / Memory-Mapped File |
| Pool                     | `ExecutorService` (conceptually)   |

The biggest difference is memory.

Threads within a JVM share the same heap. Processes do not. Every process has its own address space, and data must be communicated explicitly.

This distinction is especially important in LLM serving, where model weights, KV cache, and GPU resources are process-local unless special sharing mechanisms are used.

**Mental Model**

Think of a process as an independent Python runtime. Multiple processes cooperate by exchanging messages rather than directly sharing objects.

## 3. Core Concepts

### Process

A process is an independent execution environment with its own:

* Python interpreter
* memory space
* event loop (if any)
* resources

```text
Main Process
      │
      ├── Worker Process A
      ├── Worker Process B
      └── Worker Process C
```

Unlike threads, processes cannot directly access each other's Python objects.

### Process Lifecycle

Processes are typically:

* created
* started
* executed
* joined
* terminated

```python
p.start()
p.join()
```

`start()` launches the child process.

`join()` waits for it to finish.

### Queue

`multiprocessing.Queue` enables processes to exchange Python objects safely.

```text
Producer Process
        │
        ▼
multiprocessing.Queue
        │
        ▼
Consumer Process
```

Queues are the most common communication mechanism in multiprocessing applications.

### Pipe

A Pipe provides a direct communication channel between two processes.

```python
parent_conn, child_conn = multiprocessing.Pipe()
```

Compared with a Queue, a Pipe is simpler but typically connects only two endpoints.

### Shared Memory

Processes normally have isolated memory.

Shared memory allows selected data to be accessed by multiple processes without copying.

It is useful for large datasets, though most application code relies on message passing instead.

### Pool

A Process Pool manages a collection of worker processes.

```python
with multiprocessing.Pool() as pool:
    ...
```

Pools simplify parallel execution of many independent tasks.

They are common in data processing but less common in modern LLM serving frameworks, which typically manage long-lived worker processes explicitly.

## 4. Minimal Example

```python
from multiprocessing import Process

def worker():
    print("Hello from child process")

if __name__ == "__main__":
    p = Process(target=worker)

    p.start()
    p.join()
```

This example demonstrates the basic workflow:

* create a process
* start it
* wait for it to complete

## 5. Common Patterns

### Create a Process

```python
Process(target=worker)
```

Launch an independent execution environment.

### Start a Process

```python
process.start()
```

Begin execution.

### Wait for Completion

```python
process.join()
```

Synchronize with another process.

### Process Communication

```python
multiprocessing.Queue()
```

Exchange messages safely.

### Process Pool

```python
multiprocessing.Pool(...)
```

Execute many independent tasks using multiple worker processes.

### Shared Memory

```python
multiprocessing.shared_memory
```

Share selected data without serialization.

## 6. Why It Matters for LLM Serving

```text
                Client
                   │
                   ▼
             Uvicorn / FastAPI
                   │
                   ▼
          Serving Controller
              (Process)
                   │
      ┌────────────┴────────────┐
      ▼                         ▼
 Engine Process          Metrics Process
      │
      ▼
 GPU / Model
```

Modern LLM serving systems are built from multiple cooperating processes rather than one monolithic application.

Typical uses include:

* separating the HTTP layer from the inference engine
* isolating scheduler and worker processes
* improving fault isolation
* utilizing multiple CPU cores
* coordinating long-running services

Understanding multiprocessing makes it much easier to read the architecture of frameworks such as vLLM, SGLang, and Ray.

## 7. Common Misconceptions

### Multiprocessing = Multithreading ❌

Threads share memory.

Processes do not.

### Processes communicate automatically ❌

Data must be exchanged explicitly through queues, pipes, sockets, or shared memory.

### More processes always improve performance ❌

Additional processes increase memory usage and communication overhead.

For LLM serving, more processes may duplicate model weights or compete for GPU resources.

### Multiprocessing bypasses every Python limitation ❌

While separate processes avoid the Global Interpreter Lock (GIL), they introduce new costs such as serialization, inter-process communication (IPC), and process management.

## 8. Key Takeaways

* A process is an independent Python runtime with its own memory.
* Processes cooperate through message passing rather than shared objects.
* `Queue` is the most common IPC mechanism.
* Multiprocessing is about architecture and isolation as much as parallelism.
* Modern LLM serving frameworks are fundamentally multi-process systems.
* Understanding process boundaries is essential before studying serving schedulers and engine runtimes.

## 9. Further Reading

### Official

* Python `multiprocessing` Documentation
* Python `multiprocessing.shared_memory` Documentation

### Deep Dive

* Python Inter-Process Communication (IPC)
* Python Process Pools

### Source Code

* Python `multiprocessing`
* vLLM
* SGLang
* Ray

### Recommended Books

* *High Performance Python* (chapters on multiprocessing)

For AI infrastructure engineers, mastering the concepts of process isolation, IPC, and process lifecycle is far more important than memorizing every API. These concepts form the foundation of modern serving runtimes.
