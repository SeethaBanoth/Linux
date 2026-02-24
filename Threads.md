### 1. Why use pthread_create instead of clone() for creating threads?**

pthread_create provides a portable, high-level POSIX interface that handles stack allocation, thread attributes, synchronization (like pthread_join), and cleanup automatically. clone() is a low-level Linux kernel syscall requiring manual flags (Cags (CLONE_VM | CLONE_FS | etc.), stack management, and futex handling for termination—error-prone and non-portable. 

**Key Reasons**
- **Portability**: pthread works across Unix systems; clone is Linux-only.
- **Abstraction**: Hides kernel details, manages TLS and resources. 
- **Safety**: Built-in error checking vs. raw syscall risks.

### 2. Why a stack grows?**

A stack grows to accommodate local variables, function parameters, and return addresses pushed onto it during nested function calls and recursion. 
#### Growth Mechanism
It expands downward from high to low addresses in virtual memory. Each function call adds a stack frame; deeper calls or recursion increase usage until returns pop frames and shrink it. 
#### Practical Limits
Exceeding the fixed stack size (e.g., 8MB default per thread) triggers a segmentation fault. Linux uses guard pages to detect overflows, with growth via page faults (demand-paged). 

### 3. What segments are shared by multiple threads within a process?

Threads within a process share the text (code), data (initialized globals), BSS (uninitialized globals), and heap segments. 

#### Shared Segments

- **Text**: Executable code and read-only data—identical for all threads. 
- **Data & BSS**: Global/static variables accessible by all threads (requires synchronization). 
- **Heap**: Dynamically allocated memory via malloc()—shared address space. 

#### Private Segments

Each thread has its own stack for locals, parameters, and return addresses.

### 4. Can you fetch the thread entry point return value in your main thread?

Yes, use `pthread_join()` to retrieve it. The main thread passes a pointer to a `void*` variable as the second argument, which receives the thread's return value (cast from `void*`). 

#### How It Works
```c
void* thread_func(void* arg) {
    int result = 42;
    return (void*)(intptr_t)result;  // Return value safely
}

int main() {
    pthread_t tid;
    void* retval;
    pthread_create(&tid, NULL, thread_func, NULL);
    pthread_join(tid, &retval);  // Fetches return value into retval
    printf("Result: %ld\n", (intptr_t)retval);  // Cast back to int
    return 0;
}
```
The thread function returns via `return` or `pthread_exit()`. `pthread_join()` waits for completion and copies the exit status. 

#### Key Notes

- Always check `pthread_join()` return code for errors.
- Use `intptr_t` for integer returns to avoid pointer issues.
- Dynamically allocate complex structures before returning.

### 5. What happens when main function is invoked?

The operating system loader calls `_start()` (runtime startup), which initializes the process (argc/argv setup, environment, constructors), then invokes `main()` with command-line arguments. 

#### Execution Sequence

1. **CRT0 Startup**: Sets up stack frame, calls global constructors (C++), initializes libc.
2. **main() Entry**: Receives `argc`, `argv[]`, `envp[]`; executes user code sequentially.
3. **main() Exit**: Returns status → `exit()` cleans up (destructors, atexit handlers, flush stdio).

#### Thread Context

Main thread stack is allocated. Any `pthread_create()` calls spawn new threads sharing process address space (text/data/heap). 

### 6. What happens when CPU stops executing?

The CPU stops executing a thread when an interrupt, timer expiration, or blocking syscall occurs, triggering a context switch. 

#### Context Switch Process

1. **Save State**: Current thread's registers, PC, stack pointer saved to Thread Control Block (TCB). 
2. **Scheduler**: Kernel selects next runnable thread from ready queue. 
3. **Load State**: New thread's context restored to CPU registers. 
4. **Resume**: CPU continues from new thread's program counter.

#### Thread vs Process Switch

- **Same process**: Only thread context (registers/stack pointer) switches—fast (~1μs). 
- **Different process**: Full context + TLB flush + memory mappings—slower (~5μs). 

Main thread stops when `pthread_create()` schedules new threads or timer preemption occurs.

### 6. During the context switch, which instruction will be used to copy the contents of CPU register to your context area of PCB?

No single instruction copies *all* CPU registers to PCB/TCB. Kernel assembly code uses **MOV** (x86_64) or **STR** (ARM) instructions in sequence to save registers to thread's kernel stack or TCB.

#### Save Sequence (x86_64 example)

```
push %rax     ; Save general registers
push %rbx
mov %rsp, thread->rsp  ; Save stack pointer to TCB
mov %cr3, thread->cr3  ; Save page tables
```
1. **PUSH/STORE**: Each register saved individually to kernel stack.
2. **MOV**: Stack pointer & control registers → TCB fields.
3. **Hardware assist**: x86 FAR RETURN or SYSRET for user-mode resume.

#### Linux Thread Context (TCB = task_struct)

Registers saved to `pt_regs` struct on kernel stack, pointer stored in `task_struct->thread.regs`. ARM Cortex-M uses hardware **PUSH** for exception entry. 

### 7. How do you create separate process?

Use `fork()` system call to create a child process with separate address space, then optionally `execve()` to load new program code.

#### Process Creation Steps

```
pid_t pid = fork();
if (pid == 0) {  // Child process
    execve("/bin/ls", args, envp);  // Replace image
} else if (pid > 0) {  // Parent
    waitpid(pid, &status, 0);  // Wait for child
}
```

#### Key Differences from Threads

| Aspect | Process (`fork()`) | Thread (`pthread_create()`) |
|--------|-------------------|---------------------------|
| Address Space | **Separate** (copy-on-write) | **Shared** (text/data/heap) |
| Stack | **Separate** | **Separate** |
| PID/TID | **New PID** | **New TID** (same PID) |
| Kernel Cost | High (full context + MMU) | Low (registers only) |

**fork()** clones entire process via kernel `do_fork()` → `copy_process()`, using COW for efficient memory sharing until write. 

### 8. How do server create separate thread?

Servers create separate threads per client connection using `pthread_create()` in a `while(accept())` loop on the listening socket. 

#### Server Thread Pattern (C/Linux)

```c
int sock = socket(AF_INET, SOCK_STREAM, 0);
bind(sock, &addr, sizeof(addr)); listen(sock, 5);

while(1) {
    struct sockaddr_in client;
    socklen_t len = sizeof(client);
    int client_fd = accept(sock, (struct sockaddr*)&client, &len);
    
    pthread_t tid;
    pthread_create(&tid, NULL, handle_client, &client_fd);  // New thread per client
    pthread_detach(tid);  // Avoid zombie threads
}
```

#### void* handle_client(void* arg)

```c
void* handle_client(void* arg) {
    int fd = *(int*)arg;
    // Read request, process, send response
    recv(fd, buf, sizeof(buf), 0);
    send(fd, response, strlen(response), 0);
    close(fd);
    return NULL;
}
```

#### Thread Pool Alternative

Pre-create N worker threads + task queue (better scalability vs 1000s of threads). Main acceptor thread submits `client_fd` to queue; workers dequeue and process. 

### 9. Advantage of Thread over process?

Threads offer faster creation/switching and share memory/resources within the same process address space.

#### Key Advantages

| Aspect | Thread | Process |
|--------|--------|---------|
| **Creation Cost** | Low (~1μs, kernel stack only)  | High (~30μs, full copy-on-write)   |
| **Context Switch** | Fast (registers + stack ptr) | Slow (full MMU + TLB flush)   |
| **Memory Sharing** | Direct access (text/data/heap)  | IPC required (pipes/shm)   |
| **Communication** | Zero-copy via shared vars   | Message passing overhead   |
| **Scalability** | 1000s efficient   | Limited by memory (~100s)   |

**Server Context**: Thread-per-client handles concurrent requests sharing global config/code—processes can't without IPC complexity.

### 10. Advantage of process over Thread?

Processes provide **isolation**, **stability**, and **simpler concurrency** compared to threads.

#### Key Advantages

| Aspect | Process | Thread |
|--------|---------|--------|
| **Fault Isolation** | Crash doesn't affect others | One thread crash kills entire process |
| **Memory Safety** | Separate address space—no corruption | Shared memory risks race conditions/data races |
| **Simpler Debugging** | Independent—no shared state issues | Synchronization bugs (deadlocks, races) |
| **Resource Limits** | Can limit CPU/memory per process | All threads share process quota |
| **Security** | No direct memory access between processes | Threads access all process memory |

**Server Context**: Process-per-client (Apache prefork) safer—one bad client can't crash entire server via buffer overflow or segfault.

### 11. How do you overcome this updation or synchronization issue when multiple threads access global variables?

Use **mutex locks** (`pthread_mutex_t`) to protect critical sections around global variable access.

### Primary Solutions

#### 1. Mutex Lock (Most Common)
```c
pthread_mutex_t global_lock = PTHREAD_MUTEX_INITIALIZER;
int global_counter = 0;

void* thread_func(void* arg) {
    pthread_mutex_lock(&global_lock);     // Acquire lock
    global_counter++;                     // Critical section
    pthread_mutex_unlock(&global_lock);   // Release lock
    return NULL;
}
```

#### 2. Atomic Operations (Lock-free, faster)
```c
#include <stdatomic.h>
atomic_int global_counter = 0;

void* thread_func(void* arg) {
    atomic_fetch_add(&global_counter, 1);  // Single instruction
    return NULL;
}
```

#### Synchronization Primitives Comparison

| Technique | Use Case | Performance |
|-----------|----------|-------------|
| **Mutex** | Complex operations (multiple lines) | Slower, but flexible |
| **Atomic** | Single counter/integer ops | Fastest (CPU instruction) |
| **RWLock** | Many readers, few writers | Best for read-heavy |
| **Condition Variable** | Thread signaling/waiting | Event-driven |

**Server Context**: Mutex protects shared client connection pool or request counters; atomics for simple stats. 

### 12. How much cpu time is given to user space thread and kernel space thread?

There's no fixed CPU time allocation. Both user-space and kernel threads receive **equal scheduler timeslices** (typically 4-100ms via CFS) based on priority and nice value. 

#### Linux Scheduling Reality

- **User threads**: Run in user-mode until syscall → kernel-mode → back to user-mode
- **Kernel threads** (kworker, ksoftirqd): Run entirely in kernel-mode, same scheduler priority
- **Timeslice**: Completely Fair Scheduler (CFS) allocates based on `vruntime` (virtual runtime), not space distinction

#### Key Insight
```
top  # us = user time, sy = kernel time (both spaces combined)
24.8 us,  2.1 sy,  0.0 ni  # Total CPU usage regardless of space
```

**All threads compete equally**. Kernel threads (like `ksoftirqd`) preempt user threads normally. User/kernel space affects *mode* (ring3/ring0), not *time quantum*. 

### 13. Explain POSIX and System-V difference?

POSIX offers portable, **thread-safe** IPC APIs; System V provides older, **non-thread-safe** mechanisms with different interfaces. 

#### Key Differences

| Aspect | System V IPC | POSIX IPC |
|--------|--------------|-----------|
| **APIs** | `msgget()`, `msgsnd()`, `shmat()` | `mq_open()`, `mq_send()`, `shm_open()` |
| **Naming** | Integer **keys** (`ftok()`) | **String names** (file-like) |
| **Thread Safety** | ❌ Not safe | ✅ Thread-safe + mutex support |
| **Cleanup** | `msgctl(IPC_RMID)` | `mq_unlink()`, auto-close |
| **Monitoring** | None | `select()`, `poll()`, `mq_notify()` |

#### Usage Context
```
# System V (legacy)
int msqid = msgget(key, 0666 | IPC_CREAT);
msgsnd(msqid, &msg, sizeof(msg), 0);

# POSIX (modern)
mqd_t mq = mq_open("/queue", O_CREAT | O_WRONLY, 0666, NULL);
mq_send(mq, data, len, 0);
```

**POSIX preferred** for new multithreaded servers due to portability and safety; System V common in legacy Unix codebases. 

### 14. What are the points to remember when mutex locks are used to protect the critical section?

Always **lock before** critical section, **unlock after**, use RAII/guards to prevent leaks, minimize lock hold time, and maintain consistent lock ordering.

#### Critical Rules

- **Always pair**: `lock()` → critical section → `unlock()`
- **Use lock guards**: `pthread_mutex_lock(&m);` → auto-unlock on scope exit
- **Minimize time**: Only protect actual shared data access—keep sections tiny
- **Lock ordering**: Always acquire locks in same sequence (A→B, never B→A) to avoid deadlocks

#### Best Practices
```
pthread_mutex_t counter_lock = PTHREAD_MUTEX_INITIALIZER;
int shared_counter = 0;

// ✅ GOOD: Minimal critical section
void increment() {
    pthread_mutex_lock(&counter_lock);
    shared_counter++;  // Only this line needs protection
    pthread_mutex_unlock(&counter_lock);
}

// ❌ BAD: Too much time locked
void bad_increment() {
    pthread_mutex_lock(&counter_lock);
    printf("Logging...\n");  // I/O while locked!
    shared_counter++;
    sleep(1);               // Blocking while locked!
    pthread_mutex_unlock(&counter_lock);
}
```

#### Common Pitfalls

| Issue | Prevention |
|-------|------------|
| **Forget unlock** | Use `pthread_cleanup_push()` or scoped guards |
| **Deadlock** | Fixed lock acquisition order |
| **Reentrancy** | Use `PTHREAD_MUTEX_RECURSIVE` if needed |
| **Convoy** | Short critical sections + try-lock patterns

### 15. By using mutex_lock what you are achieving?

**Mutual Exclusion**: Ensures only **one thread** executes the critical section at a time, preventing race conditions on shared globals. 

#### Core Achievements

- **Data Consistency**: `counter++` becomes atomic (read-modify-write as single operation)
- **Race Prevention**: No torn reads/writes or lost updates
- **Serialization**: Threads queue orderly—first-come, first-served access

```
Without mutex: 10 threads × counter++ = unpredictable (8-12)
With mutex:    10 threads × counter++ = exactly 10
```

#### Server Context

```
pthread_mutex_lock(&conn_pool_lock);
available_connections--;
serve_client(client_fd);
available_connections++;
pthread_mutex_unlock(&conn_pool_lock);
```
**Prevents**: Two threads grabbing same connection → double-free crash. 

**Bottom Line**: Mutex turns non-atomic shared variable operations into **atomic critical sections**.

### 16. Difference between mutex locks and semaphore?

**Mutex**: Ownership-based lock (one thread locks, same thread unlocks). **Semaphore**: Counter-based signaling (any thread can signal).

#### Core Differences

| Aspect | Mutex | Semaphore |
|--------|--------|-----------|
| **Type** | Object with owner | Integer counter (0-N) |
| **Operations** | `lock()`/`unlock()` | `wait()`/`post()` (P/V) |
| **Ownership** | **Only owner unlocks** | **Any thread signals** |
| **Use Case** | Critical sections | Resource pools (N connections) |
| **Binary Semaphore** | ≈ Mutex (but no ownership) | Counter = {0,1} |

#### Code Examples

```c
// Mutex: Protect shared counter
pthread_mutex_lock(&mtx);
counter++;
pthread_mutex_unlock(&mtx);  // MUST be same thread

// Semaphore: Limit 3 DB connections
sem_wait(&db_sem);  // Decrement (blocks if 0)
query_db();
sem_post(&db_sem);  // Increment (wakes waiter)
```

#### Server Context

- **Mutex**: Protect `shared_config` or `request_queue`
- **Semaphore**: Limit concurrent DB connections (pool size = 5)

**Key Rule**: Wrong thread unlocking mutex = deadlock. Semaphore has no ownership—pure signaling. 

### 17. Explain the variants of pthread_mutex_lock?

POSIX defines **4 mutex types** via `pthread_mutexattr_settype()`, each with different locking behaviors controlling reentrancy and error handling.

#### Mutex Variants

| Type | Lock Behavior | Unlock Behavior | Use Case |
|------|---------------|-----------------|----------|
| **`PTHREAD_MUTEX_NORMAL`** | Blocks if locked. **Deadlocks** on self-relock | Only owner unlocks | Simple mutual exclusion |
| **`PTHREAD_MUTEX_RECURSIVE`** | **Allows recursive locking** (lock count++) | Decrements count | Recursive functions |
| **`PTHREAD_MUTEX_ERRORCHECK`** | **Returns EDEADLK** on self-relock/wrong unlock | **EACCES** if wrong owner | Debug safety |
| **`PTHREAD_MUTEX_DEFAULT`** | Usually maps to `NORMAL` or `RECURSIVE` | Implementation-defined | Default portable |

#### Code Example

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);  // Variant selection
pthread_mutex_init(&mutex, &attr);

void recursive_func() {
    pthread_mutex_lock(&mutex);     // Safe to call recursively
    recursive_func();               // Increments lock count
    pthread_mutex_unlock(&mutex);   // Decrements count
}
```

#### Key Properties

```
int pthread_mutexattr_settype(attr, type);  // NORMAL=0, RECURSIVE=1, ERRORCHECK=2
```
- **NORMAL**: Fastest but fragile (undefined self-lock)
- **RECURSIVE**: Safe for recursive code (server common)
- **ERRORCHECK**: Production debugging
- **Linux default**: `PTHREAD_MUTEX_TIMED_NP` (adds timeout)

**Server tip**: Use `RECURSIVE` for signal handlers or callback chains that might reenter. 

### 18. How to create a thread?

Use `pthread_create()` with thread ID, attributes (usually NULL), start function pointer, and argument.

#### pthread_create() Signature

```c
int pthread_create(pthread_t *thread, 
                   const pthread_attr_t *attr, 
                   void *(*start_routine)(void*), 
                   void *arg);[web:171]
```

#### Complete Working Example

```c
#include <pthread.h>
#include <stdio.h>

void* worker(void* arg) {
    int id = *(int*)arg;
    printf("Thread %d running\n", id);
    return NULL;
}

int main() {
    pthread_t threads [stackoverflow](https://stackoverflow.com/questions/6990888/c-how-to-create-thread-using-pthread-create-function);
    int ids [stackoverflow](https://stackoverflow.com/questions/6990888/c-how-to-create-thread-using-pthread-create-function) = {1, 2, 3};
    
    // Create 3 threads
    for(int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }
    
    // Wait for completion
    for(int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }
    return 0;
}
```

#### Key Points

- **Compile**: `gcc -o prog prog.c -lpthread`
- **Return**: 0=success, stores thread ID in `*thread`
- **Thread func**: Must return `void*` or call `pthread_exit()`
- **Arg**: Single `void*` parameter (pass struct for multiple values)
- **Join**: Use `pthread_join()` to wait and get return value

**Server pattern**: Create thread per `accept()`ed client FD. 

### 19. Explain the compilation of a thread?**

Use `gcc -pthread` flag for **both compilation and linking** to enable POSIX threads support.

#### Compilation Commands

```bash
# Basic compilation
gcc -o myprog myprog.c -pthread

# With warnings and optimization
gcc -Wall -O2 -o myprog myprog.c -pthread

# Multiple files
gcc -o server main.c server.c client_handler.c -pthread
```

#### Why `-pthread` (not just `-lpthread`)?

| Flag | Effect |
|------|--------|
| **`-pthread`** | **✅ Correct**: Enables preprocessor macros (`_REENTRANT`), thread-safe libc, links libpthread |
| **`-lpthread`** | ❌ Incomplete: Only links library, misses compiler flags/macros |

#### Makefile Example

```makefile
CC = gcc
CFLAGS = -Wall -g -pthread
LDFLAGS = -pthread

server: main.o server.o
	$(CC) -o $@ $^ $(LDFLAGS)
```

#### What `-pthread` Does

1. **Preprocessor**: Defines `_REENTRANT` → thread-safe `errno`, `malloc`
2. **Compiler**: Enables thread-local storage (TLS) support
3. **Linker**: Links `libpthread.so` + thread startup code

**Without `-pthread`**: `undefined reference to pthread_create` OR silent failures (non-thread-safe libc).

**Verify**: `ldd myprog` shows `libpthread.so.0` linked. 

### 20. What are the arguments of pthread_create?

`pthread_create()` takes **4 arguments** to create and configure a new thread.

#### Arguments Breakdown

| Parameter | Type | Purpose |
|-----------|------|---------|
| **`thread`** | `pthread_t *` | **Output**: Stores new thread ID |
| **`attr`** | `const pthread_attr_t *` | Thread attributes (stack size, detach state). **NULL** = defaults |
| **`start_routine`** | `void* (*)(void*)` | **Function pointer**: Thread entry point |
| **`arg`** | `void*` | **Input**: Single argument passed to thread function |

#### Signature

```c
int pthread_create(pthread_t *restrict thread,
                   const pthread_attr_t *restrict attr,
                   void *(*start_routine)(void*),
                   void *restrict arg);
```

#### Usage Example

```c
pthread_t tid;                    // 1. Thread ID storage
pthread_create(&tid,             // Output: stores thread ID
               NULL,             // 2. Default attributes  
               worker_thread,    // 3. Thread function
               &client_fd);      // 4. Pass client file descriptor
```

#### Thread Function Must Match

```c
void* worker_thread(void* arg) {  // Matches start_routine signature
    int fd = *(int*)arg;          // Extract argument
    // Handle client...
    return NULL;
}
```

**Return**: `0` = success, `-1` = error (check `errno`). Thread runs **immediately** after creation. 

### 21. Explain the return value of thread?

`pthread_create()` returns **int error code**. Thread **function** returns `void*` exit status, retrieved via `pthread_join()`.

### Two Return Values Explained

#### 1. pthread_create() Return (Creation Status)
```c
int ret = pthread_create(&tid, NULL, worker, NULL);
if (ret != 0) {
    perror("pthread_create failed");  // EAGAIN, EINVAL, EPERM
    exit(1);
}
```
| Value | Meaning |
|-------|---------|
| **0** | Success |
| **-1** | Error (`errno` set) |

#### 2. Thread Function Return (Exit Status)

```c
void* worker(void* arg) {
    int result = 42;
    return (void*)(intptr_t)result;  // Thread exit value
}

int main() {
    void *retval;
    pthread_join(tid, &retval);     // Captures thread's return
    printf("Thread result: %ld\n", (intptr_t)retval);  // Cast back
}
```

#### Complete Flow

```
main ── pthread_create() ──→ 0 (success) ──→ thread starts
                           ↓
                      thread exits ──→ return void* ──→ pthread_join() captures
```

**Key**: `pthread_create()` tells if **creation worked**. `pthread_join()` gets **what thread computed**. Use `(intptr_t)` for integers. 

### 22. Explain the working of pthread_mutex_trylock()?

`pthread_mutex_trylock()` attempts to lock a mutex **without blocking**. Returns immediately: **success (0)** if acquired, **EBUSY (-1)** if already locked. 

#### Key Differences

| Function | Behavior if Locked | Return |
|----------|-------------------|---------|
| `pthread_mutex_lock()` | **Blocks** until available | 0 (after wait) |
| `pthread_mutex_trylock()` | **Returns immediately** | 0 (success) or EBUSY |

#### Usage Pattern

```c
if (pthread_mutex_trylock(&mutex) == 0) {
    // ✅ Got lock - do critical section
    shared_counter++;
    pthread_mutex_unlock(&mutex);
} else {
    // ❌ Lock busy - skip or retry later
    printf("Resource busy, skipping...\n");
}
```

#### Perfect For

- **Non-blocking resource grabs** (connection pools)
- **Lock-free fallbacks**: Try lock → if fail, use alternative path
- **Server hot paths**: Avoid blocking on contended locks

```
Server pattern:
if (pthread_mutex_trylock(&db_conn_lock) == 0) {
    use_db_connection();
    pthread_mutex_unlock(&db_conn_lock);
} else {
    log_to_file();  // Fallback
}
```

**Advantage**: Zero-wait contention handling vs `lock()` blocking. 

### 23. Explain the application of pthread_mutex_timedlock()?

`pthread_mutex_timedlock()` locks a mutex with a **timeout**. Blocks until available **OR** absolute time `abstime` expires—prevents indefinite blocking. 

### Signature & Usage

```c
int pthread_mutex_timedlock(pthread_mutex_t *mutex, 
                           const struct timespec *abstime);
```

### Key Applications

#### 1. **Server Request Timeouts**

```c
struct timespec timeout;
clock_gettime(CLOCK_REALTIME, &timeout);
timeout.tv_sec += 2;  // 2 second timeout

if (pthread_mutex_timedlock(&db_lock, &timeout) == 0) {
    // Got DB lock within 2s
    query_database();
    pthread_mutex_unlock(&db_lock);
} else {
    // ETIMEDOUT: Use cache or drop request
    serve_from_cache(client_fd);
}
```

#### 2. **Lock Contention Avoidance**

```
if (locked) {
    log_error("DB lock timeout - high contention");
    return 503;  // Service unavailable
}
```

#### Return Values

| Return | Meaning |
|--------|---------|
| **0** | Lock acquired |
| **ETIMEDOUT** | Timeout expired |
| **EINVAL** | Bad mutex/abstime |

#### Real-Time Variant

```c
pthread_mutex_clocklock(&mutex, CLOCK_MONOTONIC, &abstime);  // Immune to NTP
```

**Server Use Case**: Protect shared resources (DB pools, config) without hanging entire worker thread pool on deadlock/contention. 

### 24. What is mutual exclusion?

Mutual exclusion ensures **only one thread** accesses a shared resource (critical section) at a time, preventing race conditions and data corruption. 

#### Core Properties (Dijkstra's Requirements)

- **Mutual Exclusion**: At most 1 thread in critical section simultaneously
- **Progress**: No deadlock—someone eventually enters if free
- **Bounded Waiting**: No starvation (fairness)

#### Without vs With Mutex

```
No Mutex (Race):
T1: counter++  // Reads 5, writes 6
T2: counter++  // Reads 5, writes 6 (lost update!)
Result: 6 ❌

With Mutex:
T1: lock → counter++ → unlock  // 5→6
T2: wait → lock → counter++ → unlock  // 6→7
Result: 7 ✅
```

#### pthread_mutex_t Implementation

```c
pthread_mutex_lock(&mtx);     // Acquire (block if busy)
shared_resource_access();     // Critical section
pthread_mutex_unlock(&mtx);   // Release
```

**Server Reality**: Without mutex, 100 client threads writing logs → garbled output. Mutex serializes safely. 
