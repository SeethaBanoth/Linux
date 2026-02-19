**1. Explain the concept of process creation in operating systems.**

A. Process creation in OS involves a parent process using system calls (like `fork()` in Unix/Linux) to spawn a child process, forming a tree hierarchy. The OS allocates a unique PID, memory, and a Process Control Block (PCB) for the child, which starts in a "New" or "Ready" state. [tutorialspoint](https://www.tutorialspoint.com/what-is-process-creation-in-operating-systems)

**Key Steps**

- Assign unique PID and add to process table.
- Allocate memory for code, data, stack.
- Initialize PCB with state, registers, and priority.
- Child shares or inherits parent resources.
  
**Triggers**

- System init, user request, batch jobs, or parent system call. 

**2. Differentiate between the fork() and exec() system calls.**

A. 
**fork() vs exec()**

Fork() creates a duplicate child process with identical code, data, and execution context; returns 0 to child and parent's PID to parent.
Exec() overlays a new program on the current process, replacing its memory image while keeping the PID and environment. 

| Aspect          | fork()                          | exec()                          |
|-----------------|---------------------------------|---------------------------------|
| Purpose        | Duplicate process              | Replace process image          |
| Return         | Child: 0, Parent: child PID    | No return (success)            |
| Memory         | Copies address space           | Overlays new executable        |
| Usage          | Often paired with exec         | Loads new program post-fork    | 


**3. Write a C program to demonstrate the use of fork() system call.**

A. 
**fork() Demo Program**

This C program uses `fork()` to create a child process; both print their PIDs and identities, showing concurrent execution. 

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("Fork failed");
        return 1;
    } else if (pid == 0) {
        printf("Child process (PID: %d)\n", getpid());
    } else {
        printf("Parent process (PID: %d, Child PID: %d)\n", getpid(), pid);
    }

    return 0;
}
```

**Sample Output**

```
Parent process (PID: 1234, Child PID: 1235)
Child process (PID: 1235)
```
Order may vary due to scheduling. 

**4. What is the purpose of the wait() system call in process management?**


A. **Purpose**

The `wait()` system call makes the parent process suspend execution until a child process terminates or changes state (exit/signal). It prevents zombie processes by reaping child exit status and freeing resources. [en.wikipedia](https://en.wikipedia.org/wiki/Wait_(system_call))

**Key Benefits**

- Retrieves child's exit status/PID.
- Parent synchronization after `fork()`.
- Avoids resource leaks from zombies. 

**Usage**: Called after `fork()` in parent to `wait(NULL)` for any child. 

**5. Describe the role of the exec() family of functions in process management.**

The exec() family replaces the current process's memory image (code, data, stack) with a new executable program while preserving PID, open files, and environment. 

**Key Roles**

- Execute new programs after fork() (fork-exec pattern).
- Used by shells to run commands.
- Overlays process without creating new PID. 

**Variants**

- `execl/execv`: List/array args, full path.
- `execlp/execvp`: PATH search.
- Success: No return (new main() runs). 

**6. Write a C program to illustrate the use of the execvp() function.**

A. This program forks a child that uses `execvp("ls", args)` to list files with `-l`, while parent waits; demonstrates process replacement. 

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>
#include <stdlib.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("Fork failed");
        exit(1);
    } else if (pid == 0) {
        char *args[] = {"ls", "-l", NULL};
        execvp("ls", args);
        perror("execvp failed");  // Only if execvp fails
        exit(1);
    } else {
        wait(NULL);
        printf("Child completed.\n");
    }
    return 0;
}
```

**Sample Output**

```
total 8
-rwxr-xr-x 1 user user 16752 Feb 19 10:52 a.out
Child completed.
```
Child's "ls -l" output appears; parent prints after. 

**7. How does the vfork() system call differ from fork()?**

A. `vfork()` creates a child process that shares the parent's address space (no copy-on-write), suspending parent execution until child calls `exec()` or `exit()`. `fork()` creates a full copy via copy-on-write, allowing both to run concurrently.

| Aspect | fork() | vfork() |
|--------|--------|---------|
| Memory | Copy-on-write (separate eventually) | Shares address space |
| Parent | Runs concurrently | Suspended until child exec/exit |
| Use case | General process creation | Followed immediately by exec() |
| Safety | Can modify independently | Child must not modify parent memory | 

**8. Discuss the significance of the getpid() and getppid() system calls.**

`getpid()` returns the calling process's unique Process ID (PID), essential for logging, temp file naming, and process identification.

`getppid()` returns the parent process's PID, crucial for process hierarchy tracking, child-parent communication, and avoiding orphan issues (returns 1 for init if parent died). 

**Key Uses**

- Debug process relationships in `fork()` programs.
- Shells track command PIDs.
- Signal handling and cleanup coordination. 

**9. Explain the concept of process termination in UNIX-like operating systems.**

Process termination occurs when a process completes execution (`exit()`) or receives a fatal signal, transitioning to "Zombie" state until parent reaps it via `wait()`. The kernel then deallocates PCB, memory, and resources. [cemozdogan](http://cemozdogan.net/OperatingSystems/week4/node13.html)

**Termination Methods**

- **Voluntary**: `exit(status)` called by process.
- **External**: Signals like SIGTERM (graceful), SIGKILL (forced) via `kill(pid, sig)`.
- **Parent-induced**: Parent uses `kill()` on child. 

**Zombie Cleanup**: Parent `wait()` retrieves status; kernel removes zombie if parent ignores SIGCHLD. 

**10. Write a program in C to create a child process using fork() and print its PID.**

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        printf("Child process - PID: %d\n", getpid());
    } else {
        printf("Parent process - Child PID: %d\n", pid);
    }
    
    return 0;
}
```

**Output:**
```
Parent process - Child PID: 1234
Child process - PID: 1234
```

**11. Describe the process hierarchy in UNIX-like operating systems.**

UNIX process hierarchy forms a tree with PID 1 (init/systemd) as root. Each process spawns children via `fork()`, creating parent-child relationships tracked by PPID in the Process Control Block. 

**Structure**

- **Root**: init (PID 1) starts all processes post-boot.
- **Parent-child**: Shells fork login sessions; commands spawn subprocesses.
- **Visualization**: `pstree` shows tree; orphans reparented to PID 1.

**Key Commands**
```
pstree          # Full process tree
pstree $$       # Current shell tree
ps -ef          # PID/PPID table view
```

**12. What is the purpose of the exit() function in C programming?**

The `exit()` function terminates the calling process immediately, performing cleanup like flushing buffers, closing open files, and calling `atexit()` handlers before returning an exit status to the OS. 

**Key Actions**

- Flushes stdio buffers and closes streams.
- Executes registered cleanup functions.
- Passes status (0/EXIT_SUCCESS for success, non-zero for failure). 

**vs return in main()**: `exit()` works from any function and does full library cleanup; `return` only exits the function (though equivalent in `main()`). 

**13. Explain how the execve() system call works and provide a code example.**

`execve()` is the underlying system call that replaces the current process image with a new program specified by `path`. It takes three parameters: `path` (executable path), `argv` (argument array ending with NULL), and `envp` (environment array ending with NULL). On success, it doesn't return—the new program's `main()` starts immediately. 

**Key Features**

- Overlays code, data, stack, heap with new executable.
- Preserves PID, open file descriptors.
- All other exec variants (execvp, execl) are library wrappers calling execve.

**Code Example**
```c
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>

int main() {
    char *argv[] = {"ls", "-l", NULL};
    char *envp[] = {"PATH=/bin:/usr/bin", NULL};
    
    printf("Before execve\n");
    if (execve("/bin/ls", argv, envp) == -1) {
        perror("execve failed");
        exit(1);
    }
    // Never reached on success
    return 0;
}
```
**Output**: `Before execve` followed by `ls -l` directory listing. 

**14. Discuss the role of the fork() system call in implementing multitasking.**

`fork()` enables multitasking by creating identical child processes that run concurrently with the parent, allowing parallel execution of tasks on multi-core systems or time-sharing via scheduler. [stackoverflow](https://stackoverflow.com/questions/10160583/fork-multiple-processes-and-system-calls)

**Multitasking Implementation**

- **Process duplication**: Creates independent execution contexts sharing CPU time.
- **Shell commands**: Each `ls`, `gcc` runs as separate process via fork-exec.
- **Concurrency**: Multiple children handle I/O, computation simultaneously. [scaler](https://www.scaler.com/topics/fork-system-call/)

**Key Benefits**

- Enables pipelining (`cmd1 | cmd2`).
- Supports process pools for servers.
- Foundation of Unix process model.

**15. Write a C program to create multiple child processes using fork() and display their PIDs.**

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    int n = 3;  // Create 3 child processes
    
    for (int i = 0; i < n; i++) {
        pid_t pid = fork();
        
        if (pid < 0) {
            perror("fork failed");
            return 1;
        } else if (pid == 0) {
            // Child process
            printf("Child %d - PID: %d\n", i+1, getpid());
            return 0;  // Child exits
        } else {
            // Parent continues to fork more
            printf("Parent created child %d with PID: %d\n", i+1, pid);
        }
    }
    
    // Parent waits for all children
    for (int i = 0; i < n; i++) {
        wait(NULL);
    }
    
    printf("All children completed.\n");
    return 0;
}
```

**Sample Output:**
```
Parent created child 1 with PID: 1234
Child 1 - PID: 1234
Parent created child 2 with PID: 1235
Child 2 - PID: 1235
Parent created child 3 with PID: 1236
Child 3 - PID: 1236
All children completed.
```

**16. How does the exec() system call replace the current process image with a new one?**

`exec()` completely overlays the current process's memory segments (text/code, data, heap, stack) with a new executable file while preserving the PID, open file descriptors, and signal handlers. [edu.info.uaic](https://edu.info.uaic.ro/sisteme-de-operare/lectures/Linux/P11_exec_print-en.pdf)

**Replacement Process**

- Kernel validates executable and maps it into process address space.
- Replaces program code, static data, initializes new stack/heap.
- New `main()` starts with provided `argc/argv`; code after `exec()` never executes.
- Failure returns -1; success doesn't return (process transformed). [en.wikipedia](https://en.wikipedia.org/wiki/Exec_(system_call))

**What Persists**

```
✓ PID
✓ Open files/signals
✗ Original code/data/stack
✗ Global variables
```
