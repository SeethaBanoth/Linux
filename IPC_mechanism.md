**Question:** What is meant by an IPC Mechanism?

**Answer:** An IPC (Inter-Process Communication) mechanism enables processes to exchange data and synchronize actions in an operating system, using methods like pipes, shared memory, message queues, signals, and sockets.

**Question:** Why do we use IPC Mechanism?

**Answer:** IPC mechanisms are used to enable processes to share data, synchronize actions, and coordinate tasks efficiently; support modularity by allowing independent components to communicate; facilitate resource sharing without duplication; enhance concurrency and parallelism on multi-core systems; and improve scalability for distributed applications. 

**Question:** What are the types of IPC Mechanisms?

**Answer:** Common IPC mechanisms include: Shared Memory (fastest, requires synchronization), Message Passing (send/receive via OS, safer), Pipes (unidirectional for related processes), Message Queues (FIFO storage), Semaphores (synchronization), Signals (notifications), and Sockets (network communication). 

**Question:** What is meant by “unicast” and “multicast” IPC?

**Answer:** In IPC, **unicast** refers to one-to-one communication where a process sends data directly to a single specific process. **Multicast** refers to one-to-many communication where a process sends data to a group of processes simultaneously, often using message queues or sockets for efficient group delivery. 

**Question:** What is meant by PIPES?

**Answer:** Pipes are a unidirectional IPC mechanism providing a fixed-size buffer for data transfer between related processes (typically parent-child via fork()), with one end for writing and the other for reading; they block on full writes or empty reads. 

**Question:** What is meant by Blocking Calls?

**Answer:** In IPC, blocking calls (synchronous) make a process suspend execution and wait until the operation completes, such as a pipe write blocking when the buffer is full or a read blocking when empty, ensuring data integrity but potentially reducing responsiveness. 

**Question:** What is meant by Blocking Calls?

**Answer:** In IPC, blocking calls (synchronous) make a process suspend execution and wait until the operation completes, such as a pipe write blocking when the buffer is full or a read blocking when empty, ensuring data integrity but potentially reducing responsiveness. 

**Question:** What are the different types of I/O Calls?

**Answer:** In IPC contexts, I/O calls are classified as **blocking (synchronous)**, where the process waits until the operation completes (e.g., read blocks on empty pipe), and **non-blocking (asynchronous)**, where the call returns immediately with an error or partial result if data isn't ready (e.g., using O_NONBLOCK flag). 

**Question:** What are the I/O calls we use in IPC Mechanisms?

**Answer:** Common I/O calls in IPC include `pipe()` (create pipe), `read()`/`write()` (data transfer on pipes/FIFOs), `open()`/`close()` (file descriptors), `msgget()`/`msgsnd()`/`msgrcv()` (message queues), `shmat()`/`shmdt()` (attach/detach shared memory), and `socket()`/`send()`/`recv()` (network sockets). 

**Question:** What are the Blocking Calls used in IPC?

**Answer:** Blocking calls in IPC include `read()` (waits for data availability), `write()` (waits for buffer space), `msgsnd()`/`msgrcv()` (wait for message queue operations), and default pipe/socket I/O without non-blocking flags. 

**Question:** What is meant by Named Pipes?

**Answer:** Named pipes (FIFOs) are persistent, filesystem-visible IPC pipes created with `mkfifo()` that allow unidirectional communication between unrelated processes (not just parent-child), unlike anonymous pipes; they appear as special files and support standard I/O operations with permissions.

**Question:** Where is the FIFO Object created?

**Answer:** The FIFO object (named pipe) is created in the filesystem namespace as a special file using `mkfifo()` or `mknod()`, appearing as a regular file entry (e.g., `/tmp/myfifo`) that processes can open with standard I/O calls; the data buffer itself resides in kernel memory. 

**Question:** What is the call used to create a FIFO Object?

**Answer:** The `mkfifo()` system call is used to create a FIFO object in the filesystem, with prototype `int mkfifo(const char *pathname, mode_t mode);` returning 0 on success. 

**Question:** What are the Blocking Calls used in Named Pipes?

**Answer:** Blocking calls in named pipes (FIFOs) include `open()` (blocks until both reader and writer are present, unless O_NONBLOCK), `read()` (blocks when no data), and `write()` (blocks when pipe is full). 

**Question:** Why does the read system call act as a blocking call?

**Answer:** The `read()` system call blocks when no data is available in the pipe/FIFO buffer (empty condition) to ensure the calling process waits for the writer to provide data, preventing busy-waiting and guaranteeing data integrity through kernel-enforced synchronization. 

**Question:** Difference between the Named Pipes and Pipes?

**Answer:** 
| Aspect | Regular Pipes | Named Pipes (FIFOs) |
|---|---|---|
| Creation | `pipe()` system call | `mkfifo()` or `mknod()` |
| Visibility | No filesystem entry; anonymous FDs | Filesystem special file (e.g., `/tmp/pipe`) |
| Process Relation | Related processes only (parent-child via fork) | Unrelated processes |
| Lifetime | Destroyed when all FDs closed | Persists until explicitly removed (`unlink()`) |
| Access | Inherited via fork | Opened by pathname with permissions |
| Reusability | Single use per creation | Reusable across multiple process pairs |

**Question:** What is return value of read system call?

**Answer:** The `read()` system call returns: number of bytes read (>0) on success, 0 on EOF/pipe closed, or -1 on error (with errno set). 

**Question:** What is meant by message queue?

**Answer:** A message queue is a kernel-managed IPC mechanism that stores structured messages (with type and priority) in a FIFO queue identified by a key/ID, allowing unrelated processes to send (`msgsnd()`) and receive (`msgrcv()`) messages asynchronously with blocking/non-blocking options. 

**Question:** Why we use message queues?

**Answer:** Message queues are used in IPC for asynchronous communication between unrelated processes, decoupling senders/receivers, load balancing across consumers, reliable message persistence (no data loss on crashes), scalability under high loads, and selective message retrieval by type/priority. 

**Question:** What is difference between Named Pipe and Message Queue?

**Answer:** | Aspect | Named Pipes (FIFOs) | Message Queues |
|--------|---------------------|----------------|
| Communication | Unidirectional byte stream  | Bidirectional structured messages with types/priorities  |
| Creation | `mkfifo()` creates filesystem entry   | `msgget()` creates kernel queue ID   |
| Data Order | Strict FIFO   | Selective retrieval by message type  |
| Process Relation | Any processes via pathname  | Unrelated processes via queue ID  |
| Persistence | Until `unlink()`   | Until `msgctl(IPC_RMID)`  |
| Size Limit | Pipe buffer (~4KB/page)  | Per-message limit (~8KB), queue configurable  |
| API Style | File I/O: `open/read/write`   | Message-specific: `msgsnd/msgrcv`   |

**Question:** What is the system call used to create the message queue?

**Answer:** The `msgget()` system call is used to create or access a message queue, with prototype `int msgget(key_t key, int msgflg);` returning a queue ID on success or -1 on error. 

**Question:** Where was the message queue created?

**Answer:** Message queues are created in the kernel's IPC namespace (not in the filesystem), managed as internal kernel objects identified by a unique queue ID (msgid) returned by `msgget()`, viewable via `ipcs -q` command. 

**Question:** What is meant by Shared Memory?

**Answer:** Shared memory is an IPC mechanism where multiple processes map the same physical memory region into their virtual address spaces using `shmget()` and `shmat()`, allowing direct, high-speed read/write access without kernel data copying; requires semaphores for synchronization to prevent race conditions. 

**Question:** Why we use Shared Memory?

**Answer:** Shared memory is used for its superior speed (fastest IPC method via direct memory access without kernel copying), low overhead for large data transfers (no repeated data copies), flexibility with complex data structures (arrays, lists), and efficiency in high-performance computing, real-time systems, and frequent producer-consumer scenarios. 

**Question:** Difference between Shared Memory and Message Queues?

**Answer:** | Aspect | Shared Memory | Message Queues |
|--------|---------------|----------------|
| Data Access | Direct memory read/write (fastest) | Kernel-mediated send/receive syscalls   |
| Synchronization | Manual (semaphores/mutexes required) | Built-in by kernel   |
| Data Structure | Raw bytes/arbitrary structures | Structured messages with types  |
| Performance | Highest throughput, low latency   | Slower due to copying/context switches  [stackoverflow](https://stackoverflow.com/questions/2287965/difference-between-message-queue-and-shared-memory) |
| Creation | `shmget()` → `shmat()` | `msgget()`  [tldp](https://tldp.org/LDP/lpg/node34.html) |
| Coupling | Tightly coupled processes | Decoupled sender/receiver  [youtube](https://www.youtube.com/watch?v=hae4nJEDskg)|
| Use Case | Large data, high-performance | Reliable, ordered small messages  [geeksforgeeks](https://www.geeksforgeeks.org/operating-systems/difference-between-shared-memory-model-and-message-passing-model-in-ipc/)

**Question:** What is use of stat command?

**Answer:** The `stat` command displays detailed file metadata including permissions, owner/group, size, timestamps (access/modify/change), inode number, block count, and file type; `stat -f` shows filesystem stats like block size and usage. 

**Question:** What is the use of semctl command?

**Answer:** The `semctl()` system call performs control operations on System V semaphores, such as getting/setting semaphore values (`GETVAL`/`SETVAL`), retrieving semaphore info (`IPC_STAT`), removing semaphore sets (`IPC_RMID`), or querying wait counts (`GETNCNT`/`GETZCNT`). 

**Question:** How do we destroy the shared memory object?

**Answer:** Use `shmctl(shmid, IPC_RMID, 0)` to mark the shared memory segment for destruction; it is fully removed only after all processes detach via `shmdt()` (when `shm_nattch` reaches 0). 

**Question:** What is meant by Semaphores?

**Answer:** Semaphores are kernel-managed integer counters used for process synchronization in IPC, controlling access to shared resources; processes use `P()` (wait/decrement, blocks if zero) and `V()` (signal/increment) atomic operations to coordinate critical sections and prevent race conditions. 

**Question:** Why we use semaphores?

**Answer:** Semaphores are used to achieve mutual exclusion (only one process in critical section), process synchronization (coordinate execution order), resource counting (limit access to finite resources), prevent race conditions, and solve producer-consumer/reader-writer problems without busy waiting. 

**Question:** What is meant by Synchronization?

**Answer:** Synchronization coordinates concurrent processes sharing resources, ensuring mutual exclusion (one-at-a-time critical section access), preventing race conditions/data corruption, and maintaining execution order via mechanisms like semaphores, preventing inconsistent states from simultaneous shared data modification. 

**Question:** What is meant by Asynchronization?

**Answer:** Asynchronization (asynchronous communication) allows processes to continue execution without waiting for IPC operations to complete, using non-blocking calls (e.g., `O_NONBLOCK` flag on pipes, `IPC_NOWAIT` on message queues) or callbacks/events, improving responsiveness but requiring polling or signal handling for completion notification. 

**Question:** Why we use mutex locks?

**Answer:** Mutex locks ensure mutual exclusion by allowing only one thread/process to access a critical section/shared resource at a time, preventing race conditions and data corruption in concurrent environments; they provide simple, efficient synchronization with ownership semantics (only lock owner can unlock). 

**Question:** What is the difference between mutex locks and semaphores?

**Answer:** | Aspect | Mutex | Semaphore |
|--------|-------|-----------|
| Purpose | Mutual exclusion (lock for 1 thread)  [geeksforgeeks](https://www.geeksforgeeks.org/operating-systems/mutex-vs-semaphore/) | Counting/signaling (N resources)  [byjus](https://byjus.com/gate/difference-between-semaphore-and-mutex/) |
| Type | Object with ownership | Integer counter  [byjus](https://byjus.com/gate/difference-between-semaphore-and-mutex/) |
| Operations | lock()/unlock() by owner only | wait(P)/signal(V) by any process  [geeksforgeeks](https://www.geeksforgeeks.org/operating-systems/mutex-vs-semaphore/) |
| Access | Only lock owner unlocks | Any process can signal  [stackoverflow](https://stackoverflow.com/questions/62814/difference-between-binary-semaphore-and-mutex) |
| Use Case | Single resource protection | Multiple identical resources  [baeldung](https://www.baeldung.com/cs/semaphore-vs-mutex) |
| Priority Inversion | Yes (owner tracking) | Less likely  [geeksforgeeks](https://www.geeksforgeeks.org/operating-systems/mutex-vs-semaphore/) |

**Question:** What is meant by Race Condition?

**Answer:** A race condition occurs when multiple processes/threads access shared data concurrently, and the final outcome depends on unpredictable execution timing, leading to data corruption or inconsistent results (e.g., two processes incrementing a counter simultaneously results in lost updates). 

**Question:** What is meant by Deadlock?

**Answer:** Deadlock is a situation where two or more processes are unable to proceed because each holds a resource while waiting for another resource held by another process, creating a circular wait chain that prevents progress. 

**Question:** What is meant by Critical Section?

**Answer:** A critical section is a code segment where a process accesses/modifies shared resources (variables, memory, files), requiring mutual exclusion so only one process executes it at a time to prevent race conditions and ensure data consistency. 

**Question:** What is the difference between system v and POSIX?

**Answer:** | Aspect | System V IPC | POSIX IPC |
|--------|--------------|-----------|
| Age/Origin | Older (1983), legacy standard  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) | Newer, portable standard  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) |
| Identification | Numeric keys/IDs (msgget/shmget)  [stackoverflow](https://stackoverflow.com/questions/4582968/system-v-ipc-vs-posix-ipc) | ASCII names/file descriptors  [stackoverflow](https://stackoverflow.com/questions/4582968/system-v-ipc-vs-posix-ipc) |
| Thread Safety | Not thread-safe  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) | Thread-safe (mutexes, cond vars)  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) |
| Shared Memory | shmget/shmat/shmdt/shmctl  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) | shm_open/mmap/shm_unlink  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) |
| Deletion | Manual (ipcrm/shmctl IPC_RMID)  [ranler.github](http://ranler.github.io/2013/07/01/System-V-and-POSIX-IPC/) | Reference-counted unlink  [ranler.github](http://ranler.github.io/2013/07/01/System-V-and-POSIX-IPC/) |
| Monitoring | Limited (ipcs command)  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) | select/poll/epoll + mq_notify  [scribd](https://www.scribd.com/document/849862197/2-6-POSIX-vs-System-V) |
| API Style | Complex, separate syscalls  [ranler.github](http://ranler.github.io/2013/07/01/System-V-and-POSIX-IPC/) | File-like (open/read/write)  [ranler.github](http://ranler.github.io/2013/07/01/System-V-and-POSIX-IPC/) |

**Question:** Describe the steps involved in creating and using a named pipe (FIFO) for IPC

**Answer:** 1. Create FIFO: `mkfifo("pathname", mode)` (e.g., 0666 permissions). 

2. Writer process: Open with `open("pathname", O_WRONLY)`; write data using `write(fd, buf, size)`; close with `close(fd)`. 
3. Reader process: Open with `open("pathname", O_RDONLY)`; read data using `read(fd, buf, size)`; close with `close(fd)`. 
4. Cleanup: `unlink("pathname")` to remove FIFO file.

**Question:** Describe the steps involved in creating and using a pipe for IPC

**Answer:** 1. Create pipe: `pipe(pipefd)` returns two FDs - `pipefd[0]` (read end), `pipefd [support](https://support.tools/linux-pipes-ipc-mastery/)` (write end). 

2. Fork child: `pid_t pid = fork()` creates parent-child processes. 
3. Child process: Close write end `close(pipefd [support](https://support.tools/linux-pipes-ipc-mastery/))`; read data `read(pipefd[0], buf, size)`; close read end. 

4. Parent process: Close read end `close(pipefd[0])`; write data `write(pipefd [support](https://support.tools/linux-pipes-ipc-mastery/), buf, size)`; close write end; `wait()` for child. 

5. Pipe auto-destroys when both FDs closed (no explicit cleanup needed).
6. 
