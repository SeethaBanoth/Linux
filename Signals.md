**1. Can you explain the concept of signals in programming? How are they used in Unixlike operating systems?**
A. Signals are asynchronous notifications sent to processes in Unix-like systems to inform them of events like interrupts, errors, or IPC requests.

**Core Concept**: Limited IPC mechanism—kernel or processes send numbered signals (SIGINT=2, SIGTERM=15) that trigger predefined/default actions or custom handlers.

**How Used**:
- **Kernel-generated**: SIGSEGV (segfault), SIGINT (Ctrl+C), SIGALRM (timer)
- **Process-to-process**: `kill(pid, SIGUSR1)` for custom events
- **Handling**:
  ```c
  void handler(int sig) { printf("Caught %d\n", sig); }
  signal(SIGINT, handler);  // Install handler
  ```
- **Actions**: Terminate, ignore (`SIG_IGN`), core dump, pause, or custom code

**Delivery**: Unblock → kernel interrupts process → runs handler → resumes normal flow.

**Key**: Reliable (once-only) vs unreliable (early Unix); modern POSIX standardizes behavior.

**2. What are software interrupts and hardware interrupts and mention potential issues
when dealing with them?**
A. **Hardware Interrupts**: External devices (keyboard, disk, timer) signal CPU via dedicated pins. **Asynchronous**—occur anytime.

**Software Interrupts**: Program-generated via instructions (`INT n`, syscalls like `read()`). **Synchronous**—explicitly triggered.

| Aspect | Hardware Interrupt | Software Interrupt |
|--------|--------------------|-------------------|
| Source | Physical devices | CPU instructions |
| Timing | Unpredictable | Program-controlled |
| Priority | Device-dependent | Highest (traps) |

**Issues**:
- **Race conditions**: Handler interrupts critical code → data corruption
- **Priority inversion**: Low-priority ISR blocks high-priority task
- **Stack overflow**: Nested interrupts exhaust stack
- **Reentrancy**: Handler calls itself → infinite loop
- **Latency**: Long ISRs delay other interrupts

**3. What is synchronous signal and asynchronous signal and how the process can be used
for both?**
A. Synchronous signals happen right when your code triggers them (e.g., divide by zero error—immediate interrupt).

Asynchronous signals arrive anytime from outside (e.g., Ctrl+C or timer—unpredictable timing).

**Processes handle both**: Use `sigaction()` to register one handler function that checks `sig` number and responds accordingly, like cleanup for SIGINT or ignore for errors. [en.wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC))

**4. Who is responsible for generating signals?**
A. **Kernel and Processes**

The OS kernel generates most signals for events like errors (SIGSEGV), timers (SIGALRM), or hardware (SIGINT from Ctrl+C). [stackoverflow](https://stackoverflow.com/questions/34279464/are-signals-generated-by-os-kernels-or-processes)

Other processes generate them too, via `kill(pid, signal)` or shell `kill` command (needs permission). [stackoverflow](https://stackoverflow.com/questions/34279464/are-signals-generated-by-os-kernels-or-processes)

Users trigger some manually (Ctrl+C). Kernel always delivers them. [www2.lawrence](http://www2.lawrence.edu/fast/GREGGJ/CMSC480/signals/signals.html)

**5. What is signal handler?**
A. A **signal handler** is a custom function you write that runs automatically when your program receives a signal (like SIGINT from Ctrl+C).

It lets you respond—e.g., save data or cleanup—instead of letting the default action (like crashing) happen. [tutorialspoint](https://www.tutorialspoint.com/what-is-default-signal-handler)

**Simple example** (C):
```c
void my_handler(int sig) { printf("Caught %d!\n", sig); }
signal(SIGINT, my_handler);
```
OS calls it mid-execution. [sanfoundry](https://www.sanfoundry.com/c-tutorials-signal-handlers/)

**6. Which system call is used to send a signal to the process?**
A. **`kill()` system call**

The primary system call to send a signal to a process (or process group) is `kill(pid, signal)`.

- `pid`: Target process ID (or -1 for all processes you own).
- `signal`: Number like SIGTERM (15) or SIGKILL (9).

Example: `kill(1234, SIGINT)` sends Ctrl+C-like interrupt to PID 1234. [man7](https://man7.org/linux/man-pages/man7/signal.7.html)

**Others**: `raise()` sends to self; `pidfd_send_signal()` for pidfds (Linux-specific). [man7](https://man7.org/linux/man-pages/man7/signal.7.html)

**7. Write a program to send a signal to itself (same process)?**
A. 
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    printf("Caught signal %d!\n", sig);
}

int main() {
    signal(SIGINT, handler);  // Set handler for SIGINT
    
    printf("PID: %d. Sending SIGINT to self...\n", getpid());
    raise(SIGINT);  // Send SIGINT to itself (or kill(getpid(), SIGINT))
    
    printf("Done.\n");
    return 0;
}
```

**Output**: `Caught signal 2!` (SIGINT=2). `raise()` is simplest for self-signals. [stackoverflow](https://stackoverflow.com/questions/63095475/how-to-make-program-send-sigint-to-itself-in-c)

**8. Explain the default action associated with the SIGKILL signal?**
A. ​**SIGKILL Default Action**

SIGKILL (signal 9) **immediately terminates** the process without cleanup, warning, or chance to save state.

- **Uncatchable**: Can't be handled, blocked, or ignored by the program. [baeldung](https://www.baeldung.com/linux/sigint-and-other-termination-signals)
- No handler runs; kernel kills it instantly, losing buffered data. [stackoverflow](https://stackoverflow.com/questions/15766036/sigkill-signal-handling)
- Used as last resort via `kill -9 PID` when SIGTERM fails. [man7](https://man7.org/linux/man-pages/man7/signal.7.html)

**9. How does a process handle a signal while it is executing in kernel mode?**
A.**Signal Handling in Kernel Mode**

Processes **don't handle user signals** while executing in kernel mode (e.g., during syscalls).

- Kernel **queues/pends** the signal until the process returns to **user mode**. [stackoverflow](https://stackoverflow.com/questions/68509464/when-are-signals-handled)
- On kernel→user transition (syscall end, interrupt return), kernel checks pending unblocked signals and runs the handler. [man7](https://man7.org/linux/man-pages/man7/signal.7.html)
- Exceptions: Kernel-generated signals (like SIGKILL) act immediately; others wait. [dl.acm](https://dl.acm.org/doi/fullHtml/10.5555/348902.349094)

**10. Describe the behaviour of a process when it receives a SIGSEGV signal?**
A. SIGSEGV (signal 11) triggers on invalid memory access (e.g., dereferencing null pointer, segfault). [stackoverflow](https://stackoverflow.com/questions/29184713/process-behaviour-after-receiving-a-sigsegv-signal)

**Default behavior**: Kernel **terminates** the process immediately + dumps a **core file** (memory snapshot for debugging). [en.wikipedia](https://en.wikipedia.org/?title=SIGSEGV)

**If caught**: Handler runs, but returns to **exact same instruction** that faulted → infinite loop unless you fix the pointer/state. [stackoverflow](https://stackoverflow.com/questions/29184713/process-behaviour-after-receiving-a-sigsegv-signal)

**Example**: Null dereference → handler → dereference again → handler → repeat forever. [stackoverflow](https://stackoverflow.com/questions/29184713/process-behaviour-after-receiving-a-sigsegv-signal)

**11. What is the role of the sigwait() function in signal handling?**
A. **`sigwait()` Role**

`sigwait()` makes a thread **wait synchronously** for a specific blocked signal, atomically consuming it and returning its number.

- **Use**: Block signals first (`sigprocmask(SIG_BLOCK, &set, NULL)`), then `sigwait(&set, &sig)` pauses until one arrives—no async handler needed. [man7](https://man7.org/linux/man-pages/man3/sigwait.3.html)
- **Benefit**: Safer for multithreading; avoids race conditions vs. handlers. Great for servers awaiting shutdown signals. [man7](https://man7.org/linux/man-pages/man3/sigwait.3p.html)

**12. Explain the concept of signal correlation in a distributed environment?**
A.**Signal Correlation in Distributed Environments**

Not a standard OS signals concept. Refers to **observability** in microservices/cloud systems.

Links related telemetry signals (logs, metrics, traces) across services using shared IDs like `trace_id`. [linkedin](https://www.linkedin.com/posts/cleutonsampaio_o11y-telemetry-otel-activity-7420039771105746944-dBV6)

**Why?** Pinpoints root causes in distributed traces—e.g., latency in Service A traces to DB timeout via correlated logs/metrics. [redhat](https://www.redhat.com/en/blog/observability-signal-correlation-red-hat-openshift)

Enables faster debugging without sifting unrelated data dumps. [eceweb1.rutgers](https://eceweb1.rutgers.edu/~gajic/solmanual/slides/chapter9_CORR.pdf)

**13. Explain how a process handles a signal while it is in the ready state**
A.**Processes in Ready State Can't Handle Signals Directly**

In the **ready state**, a process waits in the scheduler queue for CPU time—it's **not executing**. [studocu](https://www.studocu.com/en-ca/messages/question/6218858/during-which-state-transition-will-a-process-signal-handler-be-called-question-2-options)

- Signals sent now are **queued/pending** by kernel; no handler runs yet. [stackoverflow](https://stackoverflow.com/questions/68509464/when-are-signals-handled)
- When scheduler moves it to **running state** (dispatched to CPU), kernel delivers pending unblocked signals → handler executes in user mode. [stackoverflow](https://stackoverflow.com/questions/68509464/when-are-signals-handled)
- Non-executing states (ready/waiting) just mark signals as pending. [courses.cms.caltech](https://courses.cms.caltech.edu/cs124/lectures-wi2017/CS124Lec15.pdf)

**14. What is the role of the sigqueue() function in signal handling?**
A. **`sigqueue()` Role**

`sigqueue(pid, sig, value)` sends a **real-time signal** (`SIGRTMIN+` to `SIGRTMAX`) **with data** (integer `value`) attached, unlike plain `kill()`.

- Receiver uses `sigaction()` with `SA_SIGINFO` flag; handler gets `siginfo_t` struct containing the custom `value`. [qnx](https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.lib_ref/topic/s/sigqueue.html)
- **Queues multiple** same signals (FIFO); standard signals overwrite pending ones. [man7](https://man7.org/linux/man-pages/man3/sigqueue.3.html)
- Perfect for IPC: sender passes status codes or IDs to other processes/threads. [pubs.opengroup](https://pubs.opengroup.org/onlinepubs/9799919799/functions/sigqueue.html)

**15. Describe the interaction between signals and IPC mechanisms in Unix-like systems?**
A. Signals act as a **basic IPC mechanism** alongside pipes, semaphores, and message queues in Unix-like systems. [github](https://github.com/ANSANJAY/unix-signals-ipc-guide)

**Key Interactions**:
- **Signals notify** over other IPC: Parent sends SIGCHLD to self when child exits via pipe read; receiver wakes via signal. [opensource](https://opensource.com/article/19/4/interprocess-communication-linux-networking)
- **Synchronization**: `pause()` blocks until signal arrives, coordinating with shared memory or semaphores. [tldp](https://tldp.org/LDP/tlk/ipc/ipc.html)
- **Limitations**: Signals carry no data (except `sigqueue()`), unreliable delivery—use pipes/sockets for complex IPC. [github](https://github.com/ANSANJAY/unix-signals-ipc-guide)
- **Hybrid**: Daemon uses signals for shutdown + Unix sockets for config; SIGTERM triggers socket cleanup. [linkedin](https://www.linkedin.com/pulse/signal-based-inter-process-communication-ipc-in-depth-anamika-sanjay)

Signals provide lightweight, async "pings" to complement synchronous IPC primitives. [tutorialspoint](https://www.tutorialspoint.com/inter_process_communication/inter_process_communication_signals.htm)

**16. Explain how a process can determine the priority of a received signal?**
A. In Unix-like systems, a process does **not** directly “check” signal priority; the kernel decides the order.  

### 1. Standard vs real-time signals

- For **normal (standard) signals** like SIGINT, SIGTERM, SIGSEGV, POSIX says the **delivery order is unspecified** when multiple are pending. The process just sees handlers run one after another, but cannot rely on a strict priority order. [cs.auckland.ac](https://www.cs.auckland.ac.nz/references/unix/digital/APS33DTE/DOCU_006.HTM)
- For **real-time signals** (numbers from `SIGRTMIN` to `SIGRTMAX`), POSIX defines a **priority ordering**: lower-numbered real‑time signals are delivered first when multiple are pending (i.e., `SIGRTMIN` has higher priority than `SIGRTMIN+1`, and so on). [man7](https://man7.org/linux/man-pages/man7/signal.7.html)

### 2. How a process “determines” priority in practice

A user-space process cannot ask “what is the priority of this signal?” as a numeric property. Instead, it can infer behavior:

- **Signal number**:  
  - For real-time signals, the signal number itself indicates relative priority: a smaller real-time signal number means higher delivery priority. [cs.auckland.ac](https://www.cs.auckland.ac.nz/references/unix/digital/APS33DTE/DOCU_006.HTM)
- **Observed delivery order**:  
  - If the process sends several different real-time signals to itself, it can see that handlers for lower-numbered real-time signals run first, showing their higher priority.  
- **Pending sets**:  
  - Using functions like `sigpending()` and `sigismember()`, a process can see **which** signals are pending, but not an explicit priority value; it must know from POSIX rules that real-time signals are ordered and standard ones are not. [man7](https://man7.org/linux/man-pages/man7/signal.7.html)

### 3. Mixed standard and real-time signals (Linux behavior)

- POSIX leaves the order between **standard vs real-time** unspecified.  
- Linux (like many implementations) gives **standard signals priority** over real-time signals if both kinds are pending: standard ones are delivered first, then real-time ones (internally ordered by number). [stackoverflow](https://stackoverflow.com/questions/21424405/does-priority-exists-between-linux-signals)

So, a process “determines” priority indirectly:  
- By the **signal type and number** (standard vs real-time, and real-time index).  
- By observing **delivery order** enforced by the kernel, not by computing priority itself. [davmac](http://davmac.org/davpage/linux/rtsignals.html)

**17. What is the role of the sigaltstack() function in signal handling?**
A. `sigaltstack()` lets you set up an **alternate stack** for signal handlers.

**Why?** Normal stack might be corrupted (e.g., SIGSEGV segfault). Alternate stack ensures handler runs safely. [man7](https://www.man7.org/linux/man-pages/man3/sigaltstack.3p.html)

**How**:
1. Allocate memory (e.g., `SIGSTKSZ` bytes)
2. `sigaltstack(&ss, NULL)` to enable
3. `sigaction(sig, handler, SA_ONSTACK)` to use it for specific signals

Handler runs on alt stack, returns to normal one after. [manpages.ubuntu](https://manpages.ubuntu.com/manpages/bionic/man2/sigaltstack.2.html)

**18. Explain how a process can determine whether a signal was sent by the kernel or
another process?**
A. **Simple Answer**: Use `sigaction()` with `SA_SIGINFO` flag.

Handler gets `siginfo_t *info` → check `info->si_code`:

- `SI_USER` or `SI_QUEUE`: Sent by **process** (`info->si_pid` = sender PID).
- `SI_KERNEL`: Generated by **kernel** (no sender PID). [stackoverflow](https://stackoverflow.com/questions/2827877/tracing-unix-signal-origins)

**Example**:
```c
void handler(int sig, siginfo_t *info, void *uap) {
    if (info->si_code == SI_KERNEL)
        printf("Kernel generated!\n");
    else
        printf("Process %d sent it!\n", info->si_pid);
}
```

**19. Describe the interaction between signals and system calls in Unix-like systems?**
A. Signals can **interrupt system calls** in Unix-like systems.

**Key Interactions**:

1. **EINTR Return**: During a syscall (like `read()`, `wait()`), a signal arrives → syscall aborts, returns `-1` with `errno=EINTR`. [mohitmishra786.github](https://mohitmishra786.github.io/chessman/2025/03/31/Technical-Guide-to-System-Calls-Implementation-and-Signal-Handling-in-Modern-Operating-Systems.html)

2. **SA_RESTART Flag**: `sigaction(..., SA_RESTART)` → kernel **auto-restarts** syscall after handler finishes (no EINTR). [mohitmishra786.github](https://mohitmishra786.github.io/chessman/2025/03/31/Technical-Guide-to-System-Calls-Implementation-and-Signal-Handling-in-Modern-Operating-Systems.html)

3. **Kernel Mode Immunity**: Signals pend while in kernel → delivered when returning to user space. [mohitmishra786.github](https://mohitmishra786.github.io/chessman/2025/03/31/Technical-Guide-to-System-Calls-Implementation-and-Signal-Handling-in-Modern-Operating-Systems.html)

4. **Uninterruptible Calls**: Some syscalls (`[N]` suffix like `read()`) ignore signals, never return EINTR. [mohitmishra786.github](https://mohitmishra786.github.io/chessman/2025/03/31/Technical-Guide-to-System-Calls-Implementation-and-Signal-Handling-in-Modern-Operating-Systems.html)

**Example**:
```c
// Without SA_RESTART: read() → EINTR after SIGALRM
// With SA_RESTART: read() continues seamlessly
```

**Purpose**: Lets signals timeout/blocking operations safely.

**20. How does a process handle a signal while it is waiting for a semaphore?**
A. **Process waiting on semaphore = blocked state** (not running).

**Signal behavior**:
- Signal arrives → **queued/pending** by kernel
- Semaphore wait continues (uninterruptible by default in Unix)
- When **semaphore becomes available** → process wakes → kernel delivers pending signals → handler runs

**Key**: Signals pend during blocked waits; delivered after unblocking. [stackoverflow](https://stackoverflow.com/questions/68509464/when-are-signals-handled)

**21. Describe the difference between a signal handler and a signal mask?**
A. **Signal Handler vs Signal Mask**

- **Signal Handler**: Custom code that **runs** when signal arrives (like `void handler(int sig)`).

- **Signal Mask**: List of signals to **block/ignore temporarily** (`sigprocmask()`). Blocked signals wait in queue, don't run handler yet.

**Example**:
```c
// Handler: what to DO when SIGINT comes
signal(SIGINT, handler);     

// Mask: PREVENT SIGINT delivery now  
sigaddset(&mask, SIGINT);    
sigprocmask(SIG_BLOCK, &mask, NULL);
```

**Key**: Handler = action. Mask = filter (blocks delivery). [stackoverflow](https://stackoverflow.com/questions/48409070/difference-between-a-process-signal-mask-blocked-signal-set-and-a-blocked-sign)

**22. How does a process handle a signal while it is in the zombie state?**
A. **Zombie processes ignore all signals.**

A zombie (defunct) process is already terminated—it's just a process table entry waiting for parent to call `wait()`.

- **Cannot handle signals**: No memory, no execution, no handlers run
- Signals sent to zombie PID are **silently discarded**
- Only way to "handle": Parent reaps it via `wait()` or `SIGCHLD` handler

**Key**: Signals only work on **live** processes. [site24x7](https://www.site24x7.com/learn/linux/how-to-kill-zombie-process-in-linux.html)

**23. What are the advantages and disadvantages of using signals for interprocess
communication?**
A. ## Advantages of Signals for IPC

- **Lightweight & Fast**: Minimal overhead—no data copying, just kernel notification. [github](https://github.com/ANSANJAY/unix-signals-ipc-guide)
- **Built-in**: No setup (pipes/sockets need descriptors); use `kill()` instantly. [github](https://github.com/ANSANJAY/unix-signals-ipc-guide)
- **Async Notification**: Perfect for events like "shutdown now" or "data ready". [news.ycombinator](https://news.ycombinator.com/item?id=37820096)

## Disadvantages of Signals for IPC

- **No Data**: Only signal type (except `sigqueue()`); can't send messages. [github](https://github.com/ANSANJAY/unix-signals-ipc-guide)
- **Unreliable**: Same signals merge (lost); no delivery guarantee/order. [en.wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC))
- **Race Conditions**: Handlers must be async-safe—no `malloc()`, `printf()`. [en.wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC))
- **Global**: Hard in multithreaded apps; affects whole process. [news.ycombinator](https://news.ycombinator.com/item?id=37820096)

**Best for**: Simple alerts, not complex data exchange (use pipes/sockets instead). [github](https://github.com/ANSANJAY/unix-signals-ipc-guide)

**24. Explain how a process handles a signal while it is in a sleep state.**
A. **Process in Sleep = Interruptible Sleep state (S)**

- **Signal arrives** → kernel **wakes process immediately** from sleep
- Delivers signal → **runs handler** 
- Handler finishes → **sleep continues** (if time remains) OR returns early

**Example**: `sleep(60)` + Ctrl+C (SIGINT):
```
sleep starts → SIGINT wakes it → handler runs → sleep returns early
```

**Uninterruptible sleep (D)** ignores signals until I/O completes, but most `sleep()` calls use interruptible. [stackoverflow](https://stackoverflow.com/questions/12461061/sleeping-process-doesnt-respond-to-signals)

**25. How does a process handle a signal while it is in a critical section?**
A.**Critical Section = Non-Atomic Code Needing Protection**

Signals **can interrupt** anytime unless blocked.

**Handling Strategy**:

1. **Block Signals Before Entering**: `sigprocmask(SIG_BLOCK, &mask, &old)` around critical code → signals pend. [users.rust-lang](https://users.rust-lang.org/t/what-are-the-best-practices-for-unix-signal-handling/114206)

2. **Unblock After**: `sigprocmask(SIG_SETMASK, &old, NULL)` → kernel delivers pending signals safely.

**Why?** Handler won't corrupt shared data/locks (async-safe only).

**Example**:
```c
sigset_t mask;
sigemptyset(&mask); sigaddset(&mask, SIGINT);
sigprocmask(SIG_BLOCK, &mask, NULL);
// Critical: modify shared var
sigprocmask(SIG_UNBLOCK, &mask, NULL);  // Signals now delivered
```

**Risk**: Unblocked handlers must be **async-signal-safe** (no malloc/printf). [users.rust-lang](https://users.rust-lang.org/t/what-are-the-best-practices-for-unix-signal-handling/114206)

**26. What are some of the challenges associated with signal handling in multi-threaded
programs?**
A. ## Challenges in Multi-threaded Signal Handling

**Single Handler per Process**: Signals go to **one arbitrary thread** (often first-created), not specific ones. Unpredictable! [stackoverflow](https://stackoverflow.com/questions/11679568/signal-handling-with-multiple-threads-in-linux)

**Race Conditions**: Thread A blocks SIGINT, Thread B doesn't → SIGINT hits B unexpectedly [youtube](https://www.youtube.com/watch?v=pPwQlWBjL_w)

**Deadlocks**: Signal handler needs mutex Thread C holds → infinite wait [linuxjournal](https://www.linuxjournal.com/article/2121)

**Async-Safety**: Handlers can't use `malloc()`, `printf()` safely across threads [users.rust-lang](https://users.rust-lang.org/t/what-are-the-best-practices-for-unix-signal-handling/114206)

## Solutions

- **Dedicated Signal Thread**: Block signals everywhere except one thread using `sigwait()` [baeldung](https://www.baeldung.com/linux/signals-multi-threaded-app)
- `pthread_sigmask()` per-thread to control delivery
- Use thread-safe primitives only in handlers

**Example**: Main thread catches SIGINT, sets atomic flag → workers poll flag and exit cleanly.

**27. What is a race condition? Explain how it might occur in the context of signals?**
A.**Race Condition**: When program outcome depends on unpredictable timing between two events (like signal arrival vs. normal code).

**Signals Example**:
```c
int flag = 0;

// Thread 1
if (!flag) {  // Check
    open_file(); // Use  
}              // RACE!

// Meanwhile SIGINT handler: flag = 1;
```

**Problem**: Signal hits **between** check/use → file opens despite flag change.

**Fix**: Block signals around critical check-use sections with `sigprocmask()`. [users.rust-lang](https://users.rust-lang.org/t/what-are-the-best-practices-for-unix-signal-handling/114206)

**28. Discuss how a deadlock situation can be caused or resolved by signal handling?**
A. **Deadlocks Caused by Signal Handling**

**Cause**: Signal handler tries to acquire **lock** that main code already holds.

```
Thread holds mutex → SIGINT fires → handler needs same mutex → DEADLOCK!
```

**Example**:
```c
mutex_lock(&log_mutex);     // Thread holds lock
// ... critical section ...
// SIGINT handler runs:
handler() { 
    log_message();         // Tries mutex → hangs forever
}
```

**Resolution**:
- **Block signals** in critical sections (`sigprocmask(SIG_BLOCK)`)
- Use **async-safe functions** only in handlers (no locks, `printf`, `malloc`)
- Set **global flag** in handler → main code polls flag and cleans up

**29. How can you use signals to force a process to dump core?**
A. **Send SIGABRT or SIGQUIT**

```bash
kill -SIGABRT <pid>    # or kill -6 <pid>
kill -SIGQUIT <pid>    # or kill -3 <pid> 
```

**Why these?** Both have **"Core" default action** → kernel terminates process + dumps memory to core file. [baeldung](https://www.baeldung.com/linux/managing-core-dumps)

**Others that work**: SIGSEGV, SIGILL, SIGFPE, SIGBUS, SIGTRAP (if unhandled). [embeddedbits](https://embeddedbits.org/linux-core-dump-analysis-embeddedbits/)

**Check core**: `ulimit -c unlimited`; look in current dir or `/proc/sys/kernel/core_pattern` location.

**30. What are the implications of using longjmp() and setjmp() in signal handlers?**
A.**Problems with `longjmp()`/`setjmp()` in Signal Handlers**

**Dangerous & Undefined Behavior**:

1. **Stack Unwind Failure**: Signal handler stack frame never gets cleaned up—local vars, registers corrupted.

2. **Resource Leaks**: `malloc()`, file opens, locks held by interrupted code never released.

3. **Signal Mask Loss**: Signal mask from `setjmp()` may not match interrupted context → signals flood in.

4. **sigreturn() Skipped**: Kernel expects handler to call `sigreturn()` to restore state—`longjmp()` bypasses it.

**Example Disaster**:
```c
jmp_buf env;
void handler(int sig) {
    longjmp(env, 1);  // Skips cleanup, corrupts stack!
}
```

**Safe Alternative**: Set atomic flag in handler, check in main loop:
```c
volatile sig_atomic_t quit = 0;
void handler(int sig) { quit = 1; }
while (!quit) { /* work */ }
```

**31. Difference between termination and suspending of a signal?**
A.**Signal Termination vs Suspension (STOP)**

| Aspect | Termination Signals (SIGTERM, SIGKILL) | Suspension Signals (SIGSTOP, SIGTSTP) |
|--------|----------------------------------------|---------------------------------------|
| **Action** | **Ends process** completely  [suse](https://www.suse.com/c/observability-sigkill-vs-sigterm-a-developers-guide-to-process-termination/) | **Pauses process** execution |
| **State** | Process dies (no resume) | Process in `T` state (stopped) |
| **Resumable?** | **No** - gone forever | **Yes** - `SIGCONT` resumes |
| **Examples** | `kill PID`, SIGKILL(9) | Ctrl+Z, SIGSTOP(19) |
| **Handler?** | SIGTERM catchable, SIGKILL not | **Neither catchable/ignorable** |

**Use Cases**:
- **SIGTERM**: Graceful shutdown
- **SIGSTOP**: Debugger pause (`kill -STOP PID`)
- **SIGCONT**: `fg` command resumes suspended jobs

**Key**: Termination = death. Suspension = pause button.

**32. How CPU access the device register?**
A. CPU accesses device registers via **memory-mapped I/O** or **port-mapped I/O**.

## Memory-Mapped I/O (Most Common)
- Device registers appear as **special memory addresses**
- CPU uses normal `MOV`, `LOAD/STORE` instructions
- MMU translates virtual addresses → physical device register addresses
```
MOV R1, 0x10000000  ; Write to UART status register
```

## Port-Mapped I/O (x86 Legacy)
- Special `IN/OUT` instructions
- Dedicated port address space (0x3F8 for COM1)
```
OUT 0x3F8, AL     ; Write to COM1 data port
```

## How It Works
```
App → Virtual Addr → MMU → Physical Addr → Device Register
```

**Key**: Devices "pretend" to be RAM locations. CPU instructions work identically. [bob.cs.sonoma](https://bob.cs.sonoma.edu/IntroCompOrg-RPi/sec-cpureg.html)

**33. what is IRQ line?**
A. **IRQ Line** = Hardware wire connecting a device (keyboard, disk, network card) to CPU's interrupt controller.

**How it works**:
1. Device has event (key press, data arrived)
2. Device asserts **IRQ line** (electrically pulls it low/high)
3. **Interrupt Controller** (PIC/APIC) sees which line → tells CPU "interrupt #5!"
4. CPU pauses, runs **ISR** (Interrupt Service Routine) for that IRQ
5. ISR handles device, clears IRQ line

**Example**: Keyboard on IRQ 1 → keypress → CPU runs keyboard ISR → gets scancode.

**Modern**: IRQ lines **shared** via MSI/MSI-X (message interrupts) in PCIe. [ituonline](https://www.ituonline.com/tech-definitions/what-is-irq-interrupt-request/)

**34. How do you find out unique value for each IRQ line?**
A. **Check `/proc/interrupts`**

```bash
cat /proc/interrupts
```

**Output example**:
```
           CPU0       CPU1
   0:         45          0   IO-APIC-edge      timer
   1:        123         67   IO-APIC-edge      i8042
   8:         12          0   IO-APIC-edge      rtc0
  16:     245678     234567   IO-APIC-fasteoi   ahci[0000:00:1f.2]
```

**IRQ Numbers**: First column (0, 1, 8, 16...)
- **0**: System timer
- **1**: Keyboard  
- **16**: SATA disk controller

**Unique Values**: Fixed by hardware + BIOS + kernel driver assignment [en.wikipedia](https://en.wikipedia.org/wiki/Interrupt_request)

**35. when interrupt occurs?**
A. **Interrupts occur when**:

- **Hardware Events**: Keyboard press, disk data ready, network packet arrives, timer expires [ituonline](https://www.ituonline.com/tech-definitions/what-is-irq-interrupt-request/)
- **Software Events**: `syscall` (read, write), page fault, divide by zero 
- **Errors**: Invalid memory access (segfault), arithmetic overflow

**Timing**: **Asynchronous** to CPU execution—can happen **anytime** between instructions.

**Process**:
1. Device/condition triggers IRQ line
2. CPU finishes **current instruction**
3. Saves state (context switch) 
4. Jumps to **ISR** via Interrupt Vector Table
5. Handles event → returns

**Example**: You're typing → keystroke → IRQ1 → CPU runs keyboard ISR → character appears on screen.

**36. when are exceptions when they occur?**
A. **Exceptions** are **synchronous** CPU interrupts caused by the **currently executing instruction**.

**When they occur**:
- **Divide by zero** (`1/0`)
- **Invalid memory access** (null pointer dereference → SIGSEGV)
- **Page fault** (access unmapped memory)
- **Overflow/underflow** in arithmetic
- **Illegal instruction** (bad opcode)

**Key Difference from Interrupts**:
```
Interrupts: Async (keyboard press anytime)
Exceptions: Sync (bad instruction RIGHT NOW)
```

**Example**: `*(int*)0 = 5;` → CPU detects invalid address **during memory write** → raises exception → kernel sends SIGSEGV → process dies (unless handler catches it).

**37. How does kernel informs to the parent that it(child) process is terminated?**
A. **Kernel sends `SIGCHLD` signal** to parent when child terminates.

**Mechanism**:
1. Child process exits (`exit()`, returns from `main()`)
2. Kernel marks child as **zombie** (Z state)
3. Kernel delivers **SIGCHLD** to parent process
4. Parent's SIGCHLD handler runs OR default handler calls `waitpid()` 

**Parent must call `wait()` or `waitpid()`** to:
- Read child's exit status
- **Reap zombie** (clear process table entry)

**Example**:
```c
void sigchld_handler(int sig) {
    waitpid(-1, NULL, WNOHANG);  // Reap any child
}
signal(SIGCHLD, sigchld_handler);
```

**Without wait()**: Child becomes **zombie forever**, consuming PID slot until parent dies or reboots. 

**38. Which member of PCB contains the information about the signals?**
A. **`signal_struct` and `sighand_struct` in Linux `task_struct` (PCB)**

In Linux kernel, the Process Control Block (`task_struct`) contains:

```c
struct task_struct {
    ...
    struct signal_struct *signal;    // Pending signals, blocked mask
    struct sighand_struct *sighand;  // Signal handlers table
    sigset_t blocked;                // Per-thread blocked signals
    ...
};
```

**Key Members**:
- **`signal`**: Tracks pending signals, process signal actions
- **`sighand`**: Signal handler function table (one per process)
- **`blocked`**: Current signal mask (per-thread)

**Purpose**: Kernel uses these to deliver/check/block signals correctly. [baeldung](https://www.baeldung.com/linux/pcb)

**39. How can we replace SIGDEL with SIGING?**
A. **Use `signal()` or `sigaction()` to set `SIG_IGN` (ignore)**

```c
#include <signal.h>

// Method 1: signal()
signal(SIGDEL, SIG_IGN);  // Replace default with ignore

// Method 2: sigaction() (preferred)
struct sigaction sa;
sa.sa_handler = SIG_IGN;
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
sigaction(SIGDEL, &sa, NULL);
```

**Result**: Signal now **ignored** instead of default action (terminate/core/etc).

**Note**: SIGKILL/SIGSTOP cannot be ignored. Most others can.

**41. How do you modify signal behaviour table/signal disposition?**
A. ## Modifying Signal Disposition Table

Use **`sigaction()`** or **`signal()`** system calls to change signal behavior in the **Signal Disposition Table** (`sighand_struct->action[]`).

### Method 1: `sigaction()` (Recommended)
```c
#include <signal.h>

struct sigaction sa;
sa.sa_handler = handler_function;  // Or SIG_IGN, SIG_DFL
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;  // Add SA_RESTART, SA_SIGINFO as needed
sigaction(SIGINT, &sa, NULL);  // Modify SIGINT disposition
```

### Method 2: `signal()` (Simple)
```c
signal(SIGINT, handler_function);  // Or SIG_IGN, SIG_DFL
```

### Disposition Options
- **`handler_function`**: Custom handler
- **`SIG_DFL`**: Default action (terminate/core/ignore)
- **`SIG_IGN`**: Ignore signal

**Result**: Updates `task_struct->sighand->action[SIGINT].sa_handler` in PCB.

**Note**: SIGKILL/SIGSTOP cannot be modified. [linuxcampus](https://www.linuxcampus.net/documentation/man-html/htmlman7/signal.7.html)

**42. why crash in program occurs?**
A. **Program crashes occur due to**:

1. **SIGSEGV** (Segmentation Fault): Accessing invalid memory (null pointer, buffer overflow)
2. **SIGFPE** (Floating Point Exception): Divide by zero, arithmetic overflow
3. **SIGABRT**: `abort()` called or `assert()` fails
4. **SIGILL**: Illegal CPU instruction executed
5. **SIGBUS**: Invalid memory alignment/access
6. **Stack Overflow**: Recursive calls exhaust stack
7. **Unhandled SIGTERM/SIGKILL**: Process forcefully terminated

**Most Common**: **Null pointer dereference** (`*ptr` where `ptr=NULL`) → SIGSEGV → instant crash with core dump.

**43. How do you install signal handler in signal disposition table from user space?**
A. **Install Signal Handler from User Space**

Use **`sigaction()`** (preferred) or **`signal()`** system calls:

## Method 1: `sigaction()` - Full Control
```c
#include <signal.h>

void handler(int sig) {
    printf("Caught signal %d\n", sig);
}

struct sigaction sa = {0};
sa.sa_handler = handler;           // Set handler function
sigemptyset(&sa.sa_mask);          // Clear signal mask
sa.sa_flags = SA_RESTART;          // Restart syscalls on EINTR
sigaction(SIGINT, &sa, NULL);      // Install for SIGINT
```

## Method 2: `signal()` - Simple
```c
signal(SIGINT, handler);  // One-liner
```

**Effect**: Updates **Signal Disposition Table** (`sighand_struct->action[SIGINT]`) in PCB with your handler address.

**Result**: Next SIGINT → kernel calls **your `handler()`** instead of default termination.

**44. How do you catch signal?**
A. **Catch Signals with `sigaction()` or `signal()`**

```c
#include <signal.h>
#include <stdio.h>

// Signal handler function
void handler(int sig) {
    printf("Caught signal %d (Ctrl+C)\n", sig);
}

int main() {
    // Install handler for SIGINT (Ctrl+C)
    struct sigaction sa = {0};
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);
    
    printf("Press Ctrl+C...\n");
    while(1) {
        pause();  // Wait for signals
    }
    return 0;
}
```

**Steps**:
1. Define handler function: `void handler(int sig)`
2. Use `sigaction(SIGINT, &sa, NULL)` to register it
3. Kernel calls handler when signal arrives

**Compile & Run**: `gcc catch.c -o catch && ./catch` → Ctrl+C prints message instead of terminating.

**45. What does header file contains?**
A. **`<signal.h>` Header File Contains**:

## Data Types
- `sig_atomic_t` - Atomic integer type safe for signal handlers
- `sigset_t` - Signal set (bitmask of signals)
- `struct sigaction` - Signal action configuration
- `siginfo_t` - Signal information structure

## Macros (Signal Numbers)
```
SIGINT(2), SIGTERM(15), SIGKILL(9), SIGSEGV(11), SIGABRT(6)
SIG_DFL  (default handler)
SIG_IGN  (ignore signal) 
SIG_ERR  (signal() error)
```

## Key Functions
```
signal()     - Simple signal handler install
sigaction()  - Full-featured signal handling
raise()      - Send signal to self
kill()       - Send signal to process
sigprocmask()- Block/unblock signals
```

**Purpose**: Provides interface to install handlers, send signals, and manage signal masks for Unix signal handling. [tutorialspoint](https://www.tutorialspoint.com/c_standard_library/signal_h.htm)

**46. How do you clear the content of sa_mask?**
A. **Clear `sa_mask` with `sigemptyset()`**

```c
struct sigaction sa;
sigemptyset(&sa.sa_mask);  // Clears ALL signals from sa_mask
```

**What it does**:
- Initializes `sigset_t sa_mask` to **empty set**
- **No signals blocked** during handler execution
- Only blocked signals are **temporarily added** via `sigaddset(&sa.sa_mask, SIGINT)`

**Full example**:
```c
struct sigaction sa = {0};
sigemptyset(&sa.sa_mask);      // Clear mask (important!)
sa.sa_handler = handler;
sigaction(SIGINT, &sa, NULL);
```

**Result**: Handler runs with **normal process signal mask** (no extra blocking).

**47. Why signals are needed to be blocked?**
A. **Signals must be blocked to:**

1. **Protect Critical Sections**: Prevent signal handlers from interrupting atomic operations (shared data access, locks) → avoid **race conditions** and **corruption**.

2. **Guarantee Data Consistency**: Handler might see half-written data structures if it interrupts multi-instruction read/write.

3. **Avoid Reentrancy**: Same signal arriving during handler execution → **infinite recursion** or **deadlock**.

4. **Control Timing**: Defer signal delivery until safe point (after critical section).

**Example**:
```c
sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigprocmask(SIG_BLOCK, &mask, NULL);  // Block SIGINT
// Critical: modify shared counter atomically
sigprocmask(SIG_UNBLOCK, &mask, NULL); // Safe to deliver now
```

**Without blocking**: Handler corrupts data → crashes or wrong results.

**48. Signal no 2's corresponding bit is set to 1? what do you meant by it?**
A. **"Signal no 2's corresponding bit is set to 1" means**:

**SIGINT** (signal number 2) is **pending** in the signal set.

**Signal Sets = Bitmasks**:
```
sigset_t pending = 0;  // All bits 0 (no pending signals)

sigpending(&pending);  // Read pending signals

if (sigismember(&pending, SIGINT)) {  // Check bit 2
    printf("SIGINT pending!\n");      // Bit 1<<2 is 1
}
```

**Bit Position = Signal Number**:
- SIGINT(2): **Bit 2** set → `pending & (1<<2) != 0`
- SIGTERM(15): **Bit 15** set → `pending & (1<<15) != 0`

**Meaning**: Kernel queued SIGINT for delivery. When unblocked → handler runs.

**Check**: `sigismember(set, 2)` tests **SIGINT bit** (position 2).

**49. How can we see the blocking signal information in PC?**
A. **Check blocked signals via `/proc/[pid]/status`**:

```bash
grep SigBlk /proc/self/status    # Current process blocked signals
grep SigBlk /proc/1234/status    # PID 1234's blocked signals
```

**Output**: Hex bitmask where bit N=1 means signal N blocked.

**Programmatically**:
```c
sigset_t mask;
sigprocmask(SIG_BLOCK, NULL, &mask, 0);  // Read PCB blocked mask
```

**50. How do I get access to some inf0ormation present in kernel space?**
A. **From user space, access kernel info via**:

1. **System calls** (`getpid()`, `sigprocmask()`) → kernel returns specific PCB fields
2. **`/proc/[pid]/`** files → exposes signal masks, memory maps, stats from `task_struct`
3. **`ioctl()`** → device drivers return kernel data  
4. **`/sys/`** → kernel parameters, hardware info

**No direct access**—kernel validates/copies data via `copy_to_user()` for safety. [stackoverflow](https://stackoverflow.com/questions/9662193/how-to-accessif-possible-kernel-space-from-user-space)

**51. During the execution of signal handler, other than default signal. Can you block
additional signal?**
A. **Yes.** Handler's own signal auto-blocks. Block **additional signals** using `sigaction()` `sa_mask`:

```c
sigemptyset(&sa.sa_mask);
sigaddset(&sa.sa_mask, SIGTERM);  // Extra block during handler
sa.sa_handler = handler;
sigaction(SIGUSR1, &sa, NULL);
```

**During SIGUSR1 handler**: SIGUSR1 + SIGTERM blocked. Others deliver normally.

**On handler exit**: Original mask restored → pending signals fire immediately. [ld2016.scusa.lsu](http://ld2016.scusa.lsu.edu/gnu_c_libs/Blocking-for-Handler.html)

**52. Explain the scenario where signal is blocked?**
A. **Scenario: Critical Section Protection**

**Code modifying linked list** (multi-instruction atomic operation):

```c
sigprocmask(SIG_BLOCK, &mask, NULL);  // Block SIGINT,SIGTERM
list_insert(head, node);              // Vulnerable if interrupted
sigprocmask(SIG_UNBLOCK, &mask, NULL);
```

**If SIGINT arrives during `list_insert()`**:
- Handler runs → corrupts half-written list pointers
- **Result**: Segfault or memory corruption

**Blocking** → SIGINT **pends** until unblock → handler runs safely after operation completes. [github](https://github.com/Chaoses-Ib/Linux/blob/main/Kernel/Exceptions/Signals.md)

**53. From user space application can you access (Read/write) signal mask present in your
PCB?**
A. **Yes**, via `sigprocmask()` system call:

**Read**:
```c
sigset_t mask;
sigprocmask(SIG_BLOCK, NULL, &mask, 0);  // Copies PCB's blocked mask
```

**Write**:
```c
sigaddset(&mask, SIGINT);
sigprocmask(SIG_SETMASK, &mask, NULL, 0);  // Updates PCB
```

**Also**: `grep SigBlk /proc/self/status` shows the bitmask.

Kernel safely copies `task_struct->blocked` ↔ user space—no direct memory access.

**54. Where is the process information present?**
A. **Process information is stored in two places:**

**1. Kernel Space**: `task_struct` (PCB/process descriptor) 
- Contains PID, state, signal masks, registers, memory mappings
- Lives in kernel heap, linked by PID hash table

**2. User Space**: `/proc/[pid]/` filesystem
- Virtual files generated by kernel from `task_struct`
- `/proc/[pid]/status` → signals, memory  
- `/proc/[pid]/stat` → runtime stats
- `/proc/self/` → current process

**Tools like `ps/top`** read `/proc` to display process info. [stackoverflow](https://stackoverflow.com/questions/35638902/where-is-linux-process-descriptor-stored-and-what-can-access-it)
​
**55. From userspace how do I access PCB information present in kernel space?**
A. **Use `/proc/[pid]/` filesystem**—kernel exposes PCB data safely:

- `/proc/[pid]/status` → signal masks (SigBlk), memory, state [baeldung](https://www.baeldung.com/linux/pcb)
- `/proc/[pid]/stat` → runtime stats, CPU usage  
- `/proc/[pid]/maps` → memory mappings
- `/proc/self/` → current process

**Example**:
```bash
grep SigBlk /proc/self/status  # Blocked signals from PCB
cat /proc/1234/stat            # Process state
```

**System calls**: `sigprocmask(0,NULL,&mask)` reads your signal mask directly from `task_struct->blocked`.

**No direct access**—kernel copies validated data via `copy_to_user()`. Root sees all; users see own process.

**56. Can you create a proc virtual file system entry as user?**
A. **No.** Users cannot create `/proc` entries from user space.

**`/proc` filesystem** is **kernel-controlled**—entries created only by:
- Kernel code (`proc_create()` API)
- Loadable kernel modules 
- Built-in kernel subsystems

**User space** has **read access** to existing `/proc/[pid]/` files but **no write/create permissions** for new entries.

**Workaround**: Write a kernel module → `insmod` as root.

57. Without sending a signal can you invoke mysighand?
A. **No.** Signal handlers (`mysighand`) are **kernel-invoked only** when their signal arrives and is unblocked.

**Alternatives** (not true signal handler execution):
- `raise(SIGUSR1)` → sends real signal → invokes handler
- `sigqueue(getpid(), SIGUSR1, &siv)` → same
- Direct function call → bypasses kernel handler setup/restore

Kernel controls handler invocation—no user-space bypass exists.
