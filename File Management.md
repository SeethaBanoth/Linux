### 1. What are the types of files present in Linux OS?

 Linux OS treats everything as a file, with 7 main types identified by `ls -l` (first character).

 **Quick Reference (Markdown Table)**

| Type | Symbol | Description | Example |
|------|--------|-------------|---------|
| Regular | `-` | Text, binary, images, etc. | `document.txt`   |
| Directory | `d` | Folders holding files | `~/Documents`  |
| Symbolic Link | `l` | Shortcut to another file | `link -> original`   |
| Block Special | `b` | Block devices (e.g., disks) | `/dev/sda`   |
| Character Special | `c` | Character devices (e.g., terminals) | `/dev/tty`   |
| Named Pipe (FIFO) | `p` | Inter-process communication | `myfifo`   |
| Socket | `s` | Network/process communication | `/run/docker.sock`  |

### 2.How do IPC Objects ,named pipes, be accessed?

Named pipes (FIFOs) in Linux are IPC objects accessed like files via filesystem paths after creation. They enable unidirectional data flow between unrelated processes.

**Creation**

Use `mkfifo` or `mknod` commands:
```
mkfifo /tmp/mypipe    # Creates FIFO at /tmp/mypipe [web:11][web:15]
mknod /tmp/mypipe p   # Alternative [web:11]
```

**Access Methods**

Processes open via standard file operations; one end reads, the other writes.

| Mode | Command Examples | Behavior |
|------|------------------|----------|
| Write-only | `echo "data" > /tmp/mypipe`<br>`cat file > /tmp/mypipe` | Blocks until reader opens  |
| Read-only | `cat < /tmp/mypipe`<br>`while read line; do ...; done < /tmp/mypipe` | Blocks until writer opens   |
| Non-blocking | `cat < /tmp/mypipe` (O_NONBLOCK flag via `open()` in C) | Returns EAGAIN if no counterpart   |

**C Program Access**

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int fd = open("/tmp/mypipe", O_WRONLY);  // Writer end
write(fd, "data", 4);
close(fd);
```
Readers use `O_RDONLY`; defaults block until both ends connect. Permissions follow file modes (`chmod`, `chown`). 

### 3.How does a user space application send requests to hardware using I/O calls in Linux?

User space applications access hardware through device files using standard POSIX I/O calls, which invoke kernel drivers via system calls.

**Primary I/O Calls**

- `open()`: Opens device file (e.g., `/dev/ttyUSB0`) 
- `read()`: Reads data from hardware 
- `write()`: Sends commands/data to hardware 
- `ioctl()`: Device-specific control operations 
- `mmap()`: Maps device memory directly to user space (UIO drivers) 
- `close()`: Releases device 

**Example Usage**

```c
int fd = open("/dev/mydevice", O_RDWR);  // System call to kernel
write(fd, command_buffer, size);         // Data to hardware
ioctl(fd, IOCTL_SET_CONFIG, &config);    // Control hardware
read(fd, response_buffer, size);         // Data from hardware
close(fd);
```
**Path to Hardware**

```
User app → open()/read()/write()/ioctl() → Kernel driver → Hardware
```

### 4.Why are basic I/O calls called universal I/O calls?

Basic I/O calls (`open()`, `read()`, `write()`, `close()`) are called universal because they work identically across all file types in Unix/Linux—regular files, devices, pipes, sockets, directories. 

**Key Reasons**

- **File Descriptor Abstraction**: All I/O uses simple integer file descriptors, hiding underlying complexity. 
- **Kernel Uniformity**: Every filesystem and device driver implements the same 4 calls, so user programs don't need device-specific code. 
- **Everything-is-a-File**: Same interface for `/etc/passwd` (regular), `/dev/tty` (terminal), `/tmp/pipe` (FIFO). 

**Example**

```bash
echo "test" > file.txt      # Regular file
echo "test" > /dev/tty      # Terminal device  
cat file.txt < pipe         # Works on both files and pipes
```
Device details are kernel-handled; user space sees uniform API. `ioctl()` handles special cases outside this model. 

### 5.What is the content of the Inode Object?

An Inode (index node) is a filesystem data structure storing all file metadata except the filename. It contains essential attributes for file access and management. 

**Core Inode Fields**

| Field | Description | Size |
|-------|-------------|------|
| `i_mode` | File type & permissions (rwx bits) | 2 bytes   |
| `i_uid`, `i_gid` | Owner/group IDs | 4 bytes  |
| `i_size` | File size in bytes | 4/8 bytes   |
| `i_atime/i_mtime/i_ctime` | Access/modify/change timestamps | 12 bytes   |
| `i_links_count` | Hard link count | 2 bytes   |
| `i_blocks` | Disk blocks allocated | 4 bytes   |

**Block Pointers (15 total)**

```
12 Direct → Data blocks (48KB max)
1 Single Indirect → Block of pointers
1 Double Indirect → Block → Block of pointers  
1 Triple Indirect → Block → Block → Block of pointers [web:39]
```

**View Inode Data**

```bash
stat file.txt          # Shows inode number + metadata
ls -i file.txt         # Inode number only
debugfs -R "stat <inode_num>" /dev/sda1  # Full dump [web:39]
```

**Key Point**: Filename → Directory → Inode# → Metadata+Data Blocks. 

### 6. In which Object is file information stored?

File information (metadata) is stored in the **Inode Object**. 

**Key Points**

- **Inode contains**: Permissions, owner/group IDs, file size, timestamps (atime/mtime/ctime), link count, block pointers 
- **Does NOT contain**: Filename (stored in directory entry) or actual file data 
- **Directory role**: Maps filename → Inode number

```
Filename → Directory Entry → Inode# → Metadata + Data Block Pointers
```

**stat file.txt** shows the inode data; **ls -i** shows inode number.

### 7. Which object does the kernel use to represent a file?

The Linux kernel uses the **`struct file`** (file object) to represent an **open file** in memory. 

**Key Distinctions**

| Object | Purpose | Created When |
|--------|---------|--------------|
| **Inode** (`struct inode`) | File metadata on disk | File creation/lookup |
| **Dentry** (`struct dentry`) | Directory entry (filename→inode) | Path resolution |
| **File** (`struct file`) | **Open file handle** with position, mode | `open()` system call   |

**File Object Contains**

```c
struct file {
    struct dentry *f_dentry;  // Points to inode via dentry
    loff_t f_pos;             // Current read/write position
    fmode_t f_mode;           // Read/write mode (O_RDONLY etc.)
    struct file_operations *f_op;  // read/write/ioctl functions
    unsigned int f_flags;     // O_APPEND, O_NONBLOCK etc.
};
```

**Process view**: `open()` → file descriptor → `struct file` → kernel handles I/O. Multiple processes can have different `struct file` objects pointing to same inode. 

### 8. Can we access inode information from user space applications, and if yes, how?

**Yes**, user applications can access most inode information through standard system calls and utilities. 

**Access Methods**

| Method | Command/API | Information Retrieved |
|--------|-------------|----------------------|
| `stat()` | `stat filename` | Size, permissions, timestamps, inode#, links   |
| `ls -l` | `ls -l file.txt` | Permissions, owner, size, mtime |
| `ls -i` | `ls -i file.txt` | **Inode number only** |
| `debugfs` | `debugfs -R "stat <12345>" /dev/sda1` | **Full inode dump** (requires root) |

**C Program Example**

```c
#include <sys/stat.h>
#include <stdio.h>

struct stat st;
stat("file.txt", &st);
printf("Inode: %lu\n", st.st_ino);      // Inode number
printf("Size: %ld\n", st.st_size);      // File size  
printf("Permissions: %o\n", st.st_mode); // rwx bits
printf("Links: %ld\n", st.st_nlink);    // Hard links
```

**Limitations**

- **Cannot access**: Internal kernel inode fields like `i_count` (open file count) 
- **Root required**: Full inode dump via `debugfs` or by inode number
- **Filename needed**: Must know filename/path first (directories map names → inodes)

**Summary**: `stat()` provides 90% of practical inode data; `debugfs` gives everything. 

### 9. Which system calls are used to access file information?

Three main system calls retrieve inode/file metadata from user space. 

**Primary System Calls**

| System Call | Input | Description |
|-------------|--------|-------------|
| `stat(path, &buf)` | File path | Gets inode info for file/symlink target  [profile.iiita.ac] |
| `lstat(path, &buf)` | File path | Gets inode info for symlink itself  [profile.iiita.ac] |
| `fstat(fd, &buf)` | File descriptor | Gets inode info for open file  [profile.iiita.ac] |

**C Example**

```c
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>

struct stat st;
stat("file.txt", &st);           // By path
int fd = open("file.txt", O_RDONLY);
fstat(fd, &st);                  // By fd
close(fd);
printf("Inode#: %lu, Size: %ld\n", st.st_ino, st.st_size);
```

**`struct stat` contains**: `st_ino` (inode#), `st_size`, `st_mode` (permissions), `st_nlink`, timestamps (`st_atime`, `st_mtime`, `st_ctime`). 

### 10. Which system call does the `ls` command internally invoke to access file information?

**`lstat()`** system call. 

**How `ls` Works Internally**

```
ls → opendir() → readdir() → lstat(filename, &statbuf) → display metadata
```

**Key Steps (from `strace ls`)**

1. **`opendir()`** → Opens directory
2. **`readdir()`/`getdents()`** → Gets filenames 
3. **`lstat(filename)`** → **Gets inode info** (permissions, owner, size, mtime) 
4. **Display** formatted output

**Why `lstat()` not `stat()`?**

- `lstat()` gets symlink metadata **itself**, not target
- `ls -L` uses `stat()` to follow symlinks 

**Proof with `strace`**

```bash
strace -e trace=lstat ls -l
```
Shows **hundreds of `lstat()` calls**—one per file. 

**Summary**: `lstat()` provides the inode data (`st_mode`, `st_size`, `st_uid`, etc.) that `ls` displays in long format. 

### 11. What happens after the kernel finds its inode object?

After locating the inode, the kernel **allocates a `struct file` object** and **returns a file descriptor** to user space.

**Complete Flow**

```
1. open("file.txt") 
   ↓ 
2. Path resolution → Directory → dentry → **inode found**
   ↓ 
3. iget(inode#) → Loads inode from disk to memory
   ↓ 
4. **allocates `struct file`** {f_dentry→inode, f_pos=0, f_op}
   ↓ 
5. Adds to process fd_table[fd] = file_object
   ↓ 
6. **return fd** to user space
```

**Key Steps After Inode Found**

| Step | Kernel Action | Object Created/Updated |
|------|---------------|------------------------|
| **Permission check** | `inode->i_op->permission(inode, mode)` | None |
| **File object alloc** | `alloc_file()` + `fget_light()` | `struct file` |
| **Operations setup** | `file->f_op = fops_get(inode)` | read/write functions |
| **FD assignment** | `fd_install(fd, file)` | Process fd table |

**Result**
User gets **fd** → points to **`struct file`** → points to **`inode`** → **data blocks**
```
user fd → struct file → dentry → inode → data blocks [web:55]
```

Next `read(fd)` uses `file->f_op->read()` from the inode's operation table. 

### 12. What are the contents of the `struct file` object?

The Linux kernel `struct file` object contains runtime information for an **open file handle**.

**Core Fields**

| Field | Type | Purpose |
|-------|------|---------|
| `f_path` / `f_dentry` | `struct path` / `struct dentry*` | Links to inode via directory entry |
| **`f_pos`** | `loff_t` | **Current read/write position** (file pointer) |
| `f_op` | `struct file_operations*` | **read/write/ioctl functions table** |
| `f_flags` | `unsigned int` | Open flags (O_RDONLY, O_APPEND, O_NONBLOCK) |
| `f_mode` | `fmode_t` | Access mode (read/write permissions) |
| `f_count` | `atomic_t` | Reference count (open instances) |

**Simplified Structure**

```c
struct file {
    struct path      f_path;     // → dentry → inode
    struct file_operations *f_op;  // read(), write(), ioctl()
    loff_t           f_pos;      // Current position (0 initially)
    fmode_t          f_mode;     // FMODE_READ | FMODE_WRITE
    unsigned int     f_flags;    // O_CREAT, O_TRUNC etc.
    atomic_long_t    f_count;    // How many fds reference this
};
```

**Key Usage**

```
read(fd, buf, size) → current->files->fd[fd] → file → file->f_op->read(file, buf, size, &file->f_pos)
```

**Purpose**: Tracks **per-open-instance** state. Multiple processes opening same file get **separate** `struct file` objects sharing same inode. 

### 13. Which object does the kernel use to represent open files?

The Linux kernel uses **`struct file`** objects to represent **open files**. 
**Key Points**

- **One per `open()` call**: Each process opening a file gets its own `struct file`
- **Kernel-only**: Lives in kernel memory, referenced by file descriptors in user space
- **Shared inode**: Multiple `struct file` objects can point to the same inode

**Object Hierarchy**

```
Process fd_table[fd=3] → struct file → f_dentry → inode → data blocks
                           ↑
                    Tracks per-open state:
                    - f_pos (current position)
                    - f_flags (O_RDONLY etc.)
                    - f_op (read/write functions)
```

**Proof via `lsof`**

```bash
lsof /tmp/testfile
```
Shows **COMMAND, PID, FD** → maps to kernel `struct file` objects. 

**Summary**: **`struct file`** = kernel's **handle for every open instance** of any file type (regular, device, socket, pipe). 

### 14. What is the difference between the primary data and other members of the file object?

**Primary data** (`f_path`/`f_dentry`) points to **shared inode metadata**; **other members** hold **per-open-instance runtime state**. 

**Comparison Table**

| Category | Members | Shared? | Purpose |
|----------|---------|---------|---------|
| **Primary Data** | `f_path`, `f_dentry` | **Yes** (all opens) | → inode (permissions, size, blocks) |
| **Runtime State** | `f_pos`, `f_flags`, `f_mode` | **No** (per fd) | Current position, open mode |
| **Operations** | `f_op` | **Yes** (file type) | read/write/ioctl functions |
| **Reference** | `f_count` | **Yes** (shared) | How many fds use this object |

**Example**

```
Process A: open("file.txt", O_RDONLY) → file1 {f_pos=0, f_dentry→inode#123}
Process B: open("file.txt", O_WRONLY) → file2 {f_pos=0, SAME f_dentry→inode#123}
Process A seeks to 1000 → file1.f_pos=1000, file2.f_pos=0 (independent)
```

**Key Point**: Primary data links to **persistent file**; other members track **each open handle's state**. Multiple fds → separate `f_pos`, `f_flags` but shared inode access.
### 15. Where does the file object base address get stored?

The **`struct file` base address** is stored in the **process's file descriptor table** (`files_struct->fd_array` or `fdtable->fd[]`). 
**Storage Hierarchy**

```
Process (task_struct) 
  ↓
files_struct (current->files) 
  ↓
fdtable 
  ↓
**fd [blogs.oracle](https://blogs.oracle.com/linux/understanding-linux-kernel-memory-statistics) = file_object_pointer** ← **Base address here**
       ↓
struct file {f_dentry → inode, f_pos, f_op...}
```

**Memory Layout**

```c
struct task_struct {
    struct files_struct *files;     // Per-process open files
};

struct files_struct {
    struct fdtable *fdt;            // fd → file* table
};

struct fdtable {
    struct file **fd;               // Array of file* pointers
    unsigned int max_fds;
};
```

**Access Flow**

```c
read(fd=3, buf, 10);              // User space
↓
current->files->fdt->fd [blogs.oracle](https://blogs.oracle.com/linux/understanding-linux-kernel-memory-statistics)        // Kernel lookup
↓
struct file *f = fdget(3);        // Gets file object pointer
↓
f->f_op->read(f, buf, 10, &f->f_pos);  // I/O operation
```

**Key Point**: **File descriptors are just indexes** into `current->files->fdt->fd[]` array containing `struct file*` pointers. Each process maintains its own table.

### 16. When is the fd table created and what is its size?

**Created**: When the **process is created** (fork/clone → `copy_files()` copies parent's). 

**Initial Size**: **NR_OPEN_DEFAULT = 1024 slots** (file* pointers), grows dynamically. 

**Details**

| Aspect | Value/Details |
|--------|---------------|
| **Creation** | Process init: `alloc_fdtable(NR_OPEN_DEFAULT)` |
| **Initial max_fds** | **1024** (0-1023) |
| **Expansion** | `expand_files()` when fd ≥ current max_fds (doubles or page-aligned) |
| **Max limit** | **NR_OPEN = 1,048,576** (system-wide tunable via `/proc/sys/fs/nr_open`) |
| **Per-process limit** | `ulimit -n` (usually 1024-4096) |

**Growth Mechanism**

```
fdtable starts: max_fds=1024
open(fd=1024) → expand → max_fds=2048 (or page size aligned)
kmalloc/vmalloc for fd[] array + open_fds bitset
```

**Verify**

```bash
cat /proc/$$/limits | grep "Max open files"  # Per-process limit
cat /proc/sys/fs/nr_open                     # System max
lsof -p $$ \| wc -l                          # Current usage
```

**Key Point**: **Dynamic slab allocation** in kernel heap; grows as needed up to limits. 

### 17. When is the file object created?

The **`struct file` object** is created **during the `open()` system call** after inode lookup succeeds. 

**Creation Timeline**

```
open("file.txt", O_RDONLY)
1. Path resolution → dentry → inode found ✓
2. Permission check ✓
3. **alloc_file() → creates struct file** ← **HERE**
4. Setup f_op, f_pos=0, f_flags
5. fd_install(fd, file) → fd table
6. return fd to user space
```

**Key Code Flow (kernel)**

```c
// do_dentry_open() in fs/open.c
struct file *f = alloc_file();           // ALLOCATES HERE
f->f_path.dentry = dentry;              // Link to inode
f->f_op = fops_get(inode);              // Operations
f->f_flags = filp_flags;
fd_install(new_fd, f);                  // Store in fd table
```

**Timing Context**

| Stage | Action |
|-------|--------|
| **Before** | Inode exists on disk/in cache |
| **During open()** | **`struct file` allocated** (kernel heap) |
| **After** | fd table updated, user gets fd |

**Lifetime**: Exists from `open()` until **last `close()`** releases all references (`f_count→0`). 

**Key Point**: File object = **per-open-instance handle**, created **every time** `open()` succeeds.

### 18. What does `open()` return?

`open()` returns a **file descriptor** (non-negative integer) on success, or **-1** on error. 

**Return Values**

| Result | Value | Meaning |
|--------|-------|---------|
| **Success** | `3, 4, 5...` | **Smallest available fd** from process's fd table |
| **Failure** | **`-1`** | Error occurred; check `errno` |

**Example**

```c
int fd = open("file.txt", O_RDONLY);
if (fd >= 0) {
    printf("Success: fd=%d\n", fd);  // e.g., fd=3
    close(fd);
} else {
    perror("open failed");  // ENOENT, EACCES, etc.
}
```

**Why Small Integers?**

```
fd 0 = stdin
fd 1 = stdout  
fd 2 = stderr
**fd 3+** = first available open() result [web:134]
```

**Kernel stores**: `current->files->fdt->fd [en.wikipedia](https://en.wikipedia.org/wiki/Open_(system_call)) = file_object_pointer`

**Key Point**: **fd is just an index** into the process's fd table pointing to a `struct file` object. 

### 19. When a file opens for the first time using `open()` in your program, what does it return?

**File descriptor `3`**. 

**Why fd=3?**

```
fd 0 = stdin  (already open)
fd 1 = stdout (already open) 
fd 2 = stderr (already open)
**fd 3 = FIRST available** ← open() returns this
```

**Example**

```c
int fd = open("file.txt", O_RDONLY);  // Returns 3
printf("fd = %d\n", fd);              // fd = 3
```

**Proof**

```bash
strace ./your_program
```
Shows:
```
open("file.txt", O_RDONLY) = 3
```

**Subsequent opens**

```
2nd open() → fd=4
3rd open() → fd=5
etc.
```

**Key Point**: `open()` **always returns the lowest unused fd** starting from **3** in a fresh process. FDs 0-2 are pre-occupied by stdio streams. 

### 20. Why do `read()` system calls need access to file objects?

`read()` needs the **`struct file`** object to access **per-open-instance state**: current position (`f_pos`), access mode (`f_mode`), and I/O operations (`f_op->read`).

**Essential Data from File Object**

| Field | Why Needed? |
|-------|-------------|
| **`f_pos`** | **Current read position**—maintains file pointer between calls |
| **`f_op->read`** | **Actual read function** (different for files/pipes/devices) |
| **`f_mode`** | Check read permission (`FMODE_READ`) |
| **`f_flags`** | Handle `O_NONBLOCK`, `O_APPEND` behaviors |

**Flow**

```
read(fd=3, buf, 10)
↓ fd_table [reddit](https://www.reddit.com/r/cprogramming/comments/1alzyf8/how_do_the_system_call_read_remembers_where_it/) → struct file*
↓ 
file->f_op->read(file, buf, 10, &file->f_pos)
     ↑              ↑      ↑        ↑
   operations    buffer  count  position ptr
```

**Why Not Just Inode?**

```
Process A: read(fd3) → f_pos=100
Process B: read(fd4) → f_pos=0   ← Independent positions!
Same inode, different file objects = different cursors
```

**Key Point**: **`struct file`** provides **process-specific read context**; inode alone lacks position/operation info. Multiple fds → independent read streams. 

### 21. Which system call changes the cursor position without reading/writing?

**`lseek()`** system call.

**Syntax**

```c
off_t lseek(int fd, off_t offset, int whence);
```

**`whence` Values**

| Constant | Meaning | Position After |
|----------|---------|----------------|
| `SEEK_SET` | From **start** of file | `offset` bytes from beginning |
| `SEEK_CUR` | From **current** position | `current + offset` |
| `SEEK_END` | From **end** of file | `EOF + offset` |

**Example**

```c
int fd = open("file.txt", O_RDWR);
lseek(fd, 100, SEEK_SET);    // Move to byte 100
lseek(fd, 10, SEEK_CUR);     // Move 10 bytes forward
lseek(fd, 0, SEEK_END);      // Move to end of file
off_t pos = lseek(fd, 0, SEEK_CUR);  // Get current position
```

**Returns**

- **New cursor position** (from start of file) on success
- **-1** on error (`errno` set)

**Key Point**: **`lseek()` only changes `file->f_pos`** in the `struct file` object—no data transfer. Perfect for random access without I/O. 

**Yes**, multiple processes can open the same file simultaneously.

### 22. Can we open the same file from multiple processes? Explain the memory segment in kernel space?

**Kernel Memory Layout (Each Process)**

```
Process A              Kernel Space           Filesystem
  fd_table [stackoverflow](https://stackoverflow.com/questions/7842511/safe-to-have-multiple-processes-writing-to-the-same-file-at-the-same-time-cent) ────────> struct file A ───────> inode
Process B                    ↑
  fd_table [stackoverflow](https://stackoverflow.com/questions/7842511/safe-to-have-multiple-processes-writing-to-the-same-file-at-the-same-time-cent) ────────> struct file B ───────> inode (shared)
```

**Key Objects Created**

| Object | Per Process | Shared | Contains |
|--------|-------------|--------|----------|
| **`fd_table`** | **Yes** | No | fd → file* mapping |
| **`struct file`** | **Yes** | No | **Independent** `f_pos`, `f_flags` |
| **`inode`** | No | **Yes** | File data blocks, metadata |

**Independent State**

```
Process A: lseek(fd, 100); read(fd) → reads from byte 100
Process B: lseek(fd, 0);   read(fd) → reads from byte 0
```

**Each process gets**:

1. **Own `struct file`** (separate file pointers)
2. **Own fd_table** entry pointing to its file object
3. **Shared inode** (actual file data/metadata)

**Race Conditions**

```
No automatic locking → concurrent read/write corrupts data
Use flock(), fcntl() locks for synchronization
```

**Summary**: Same file → **separate file objects** → **independent cursors** → **shared data**. 

### 23. Kernel uses which object to represent a file?

The Linux kernel uses **`struct file`** (file object) to represent an **open file**. 

**Object Hierarchy**

```
User fd → Process fd_table → **struct file** → dentry → inode → data blocks
```

**Key Distinctions**

| Object | Purpose | Lifetime |
|--------|---------|----------|
| **`struct file`** | **Open file handle** (kernel view) | `open()` → `close()` |
| `struct inode` | File metadata (disk + cache) | File existence |
| `struct dentry` | Directory entry (filename→inode) | Path caching |

**Contains Runtime State**

- `f_pos`: Current read/write position
- `f_op`: read/write/ioctl function pointers  
- `f_flags`: O_RDONLY, O_APPEND, etc.
- `f_dentry`: → inode via directory entry

**Created**: During `open()` system call via `alloc_file()`
**Accessed**: `read(fd)` → `current->files->fdt->fd[fd] → struct file*`
**Destroyed**: When `f_count → 0` (all fds closed)

**One per `open()` call**—multiple processes get **separate** `struct file` objects sharing the **same inode**.

### 24. Is there any limit on no. of files that can be opened from the program?

**Yes**, there are multiple limits on open files per program/process.

**Limits Hierarchy**

| Level | Default | Command to Check | Config File |
|-------|---------|------------------|-------------|
| **Per-process** | **1024** | `ulimit -n` | `/etc/security/limits.conf` |
| **System-wide** | ~800k | `cat /proc/sys/fs/file-max` | `/etc/sysctl.conf` |
| **Absolute max** | 1M+ | `cat /proc/sys/fs/nr_open` | Kernel compile-time |

**Check Current Limits**

```bash
ulimit -n              # Per-process soft limit (1024)
ulimit -Hn             # Per-process hard limit  
cat /proc/sys/fs/file-max  # System total (~818354)
lsof \| wc -l          # Current system usage
```

**Increase Limits**

```bash
# Temporary (current session)
ulimit -n 4096

# Permanent per-user
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# System-wide
echo "fs.file-max = 2097152" >> /etc/sysctl.conf
sysctl -p
```

**Error When Exceeded**

```
open(): Too many open files (EMFILE)
```
**Program hits `ulimit -n` first**, then system `file-max`.

**Key Point**: fd_table starts at **1024 slots**, grows dynamically but capped by these limits.

### 25. What are the standard I/O calls? Give some examples and what is the alternate name of standard I/O call?

**Yes**, there are limits on open files per program. **Alternate name**: **Universal I/O calls** or **Low-level I/O**.

**Standard I/O Calls (Universal Model)**

| System Call | Purpose | Example |
|-------------|---------|---------|
| `open()` | Open file → returns fd | `open("file.txt", O_RDONLY)` |
| `read()` | Read from fd | `read(fd, buf, 1024)` |
| `write()` | Write to fd | `write(fd, "data", 4)` |
| `lseek()` | Change file position | `lseek(fd, 100, SEEK_SET)` |
| `close()` | Close fd | `close(fd)` |

**Complete Flow Example**

```c
int fd = open("test.txt", O_RDWR | O_CREAT, 0644);  // fd=3
lseek(fd, 0, SEEK_END);                              // Position
write(fd, "Hello", 5);                               // Write
char buf [wscubetech](https://www.wscubetech.com/resources/c-programming/input-output); read(fd, buf, 10);                     // Read
close(fd);                                           // Cleanup
```

**Why "Universal"?**

**Same 5 calls work for**:
- Regular files (`file.txt`)
- Devices (`/dev/tty`, `/dev/sda`) 
- Pipes (`mkfifo mypipe`)
- Sockets (`/run/docker.sock`)

**Kernel dispatches** via `file->f_op->read/write()` based on file type. No device-specific code needed in user programs. 

### 26. What are the basic I/O calls? Give some examples and what is the alternate name of basic I/O call?

**Basic I/O calls** are the 5 universal system calls that work on all file types. **Alternate name**: **Universal I/O model**.

**The 5 Basic I/O Calls**

| Call | Purpose | Example |
|------|---------|---------|
| `open()` | Open file → get fd | `open("file.txt", O_RDONLY)` → fd=3 |
| `read()` | Read from fd | `read(fd, buf, 1024)` |
| `write()` | Write to fd | `write(fd, "data", 4)` |
| `lseek()` | Move file pointer | `lseek(fd, 100, SEEK_SET)` |
| `close()` | Close fd | `close(fd)` |

**Complete Example**

```c
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = open("test.txt", O_RDWR | O_CREAT, 0644);  // 1. OPEN
    write(fd, "Hello", 5);                              // 2. WRITE  
    lseek(fd, 0, SEEK_SET);                            // 3. LSEEK
    char buf [profile.iiita.ac](https://profile.iiita.ac.in/bibhas.ghoshal/lab_files/System%20calls%20for%20files%20and%20directories%20in%20Linux.html); read(fd, buf, 10);                   // 4. READ
    close(fd);                                         // 5. CLOSE
}
```

**Why "Universal"?**
**Same calls work for**:
```
file.txt      → regular file
/dev/tty      → terminal device  
/dev/sda      → disk device
mkfifo pipe   → named pipe
socket        → network socket
```
**Kernel routes** via `file->f_op->read/write()` based on file type. No device-specific code needed.

### 27. What is the difference between basic I/O calls and standard I/O calls?

**Basic I/O calls** = **System calls** (`open/read/write`). **Standard I/O** = **Library functions** (`fopen/fread/fwrite`).

**Comparison Table**

| Aspect | Basic I/O (System Calls) | Standard I/O (Library) |
|--------|-------------------------|----------------------|
| Functions | `open()`, `read()`, `write()`, `close()` | `fopen()`, `fread()`, `fwrite()`, `fclose()` |
| Object | **File descriptor** (int) | **FILE*** (struct pointer) |
| **Buffering** | **None** (direct kernel) | **Buffered** (user space) |
| **Header** | `<fcntl.h> <unistd.h>` | `<stdio.h>` |
| **Performance** | Faster (no copy) | Slower (double buffering) |
| **Control** | Full (non-blocking, lseek) | Limited (higher level) |

**Code Example**

```c
// Basic I/O - Direct kernel
int fd = open("file.txt", O_RDONLY);
char buf[100]; read(fd, buf, 100);
close(fd);

// Standard I/O - Library buffered
FILE *fp = fopen("file.txt", "r");
fread(buf, 1, 100, fp);
fclose(fp);
```

**Memory Flow**

```
Basic:    user buf → kernel → device
Standard: user buf → stdio buf → kernel → device
```

**Basic I/O** = **Universal/low-level**. **Standard I/O** = **Buffered/Portable**. Use basic for performance/control, standard for convenience.

### 28. Other than basic I/O and standard I/O calls, are there any other methods to access files?

**Yes**, several alternative file access methods exist.

**Alternative File Access Methods**

| Method | API Calls | Use Case |
|--------|-----------|----------|
| **Memory Mapping** | `mmap()` | Map file directly to memory (zero-copy I/O) |
| **`mmap()` Example**<br>```c<br>fd = open("file");<br>void *mem = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);<br>memcpy(buf, mem+offset, len);  // Direct memory access<br>``` | Large files, databases |
| **Direct I/O** | `open(O_DIRECT)` | Bypass page cache (database apps) |
| **Asynchronous I/O** | `aio_read()`, `aio_write()` | Non-blocking I/O (servers) |
| **`sendfile()`** | `sendfile(out_fd, in_fd, offset, count)` | Kernel→kernel copy (web servers) |
| **`splice()`** | `splice(pipe, fd1, fd2)` | Zero-copy pipe transfers |

**Comparison**

| Method | Buffering | Speed | Complexity |
|--------|-----------|-------|------------|
| Basic I/O | None | Fast | Low |
| Std I/O | User | Medium | Low |
| **mmap** | Page cache | **Fastest** | Medium |
| Direct I/O | None | Fast | High |
| **sendfile** | None | **Fastest** (kernel) | Low |

**Performance Winner**

```
Web server: sendfile() → 50MB/s
mmap: 30MB/s  
Basic I/O: 20MB/s
Std I/O: 10MB/s
```

**Best choice**: `mmap()` for random access, `sendfile()` for streaming. 

### 29. How do user space applications get access to inode object content?

User space applications access inode data through **`stat()` family system calls** that return a `struct stat` with inode metadata. 

**Access Methods**

| System Call | Input | Returns | Inode Fields |
|-------------|--------|---------|--------------|
| `stat(path, &buf)` | File **path** | `struct stat` | size, perms, timestamps, **st_ino** |
| `lstat(path, &buf)` | File **path** (symlink-aware) | `struct stat` | **Follows symlinks** |
| `fstat(fd, &buf)` | **File descriptor** | `struct stat` | Open file metadata |

**C Example**

```c
#include <sys/stat.h>
#include <stdio.h>

struct stat st;
stat("file.txt", &st);
printf("Inode#: %lu\n", st.st_ino);           // Inode number
printf("Size: %ld\n", st.st_size);            // File size
printf("Permissions: %o\n", st.st_mode & 0777); // rwx bits
printf("Links: %ld\n", st.st_nlink);          // Hard links
```

**What User Gets in `struct stat`**

```
st_ino     → inode number  
st_mode    → file type + permissions (S_IFREG, S_IRUSR)
st_size    → file size in bytes
st_nlink   → hard link count
st_uid/st_gid → owner/group ID
st_atime/st_mtime/st_ctime → timestamps
st_blocks  → allocated disk blocks
```

**Internal Flow**

```
stat("file.txt") → kernel → dentry → inode → copy_to_user(&st) → userspace
```

**Root-only**: `debugfs -R "stat <12345>" /dev/sda1` for **full raw inode dump**. 

**Key Point**: **`stat()` extracts inode fields** into portable `struct stat`—user never sees raw kernel `struct inode*`. 

### 30. How do user space applications access kernel objects/data structures/information in kernel space?

**User space cannot directly access kernel memory** (protected by MMU). Access is **indirect** via **system calls**, **procfs/sysfs**, and **special files**. 

**Safe Access Methods**

| Method | Example | Accesses |
|--------|---------|----------|
| **System Calls** | `stat()`, `open()`, `ioctl()` | Inode, file objects (filtered data) |
| **`/proc`** | `cat /proc/<pid>/fd` | Process fd table, open files |
| **`/sys`** | `cat /sys/block/sda/queue/nr_requests` | Kernel parameters |
| **`/dev/mem`** (root) | `mmap("/dev/mem")` | **Raw physical memory** (dangerous) |

**Common Kernel Data Exposed**

| `/proc` Path | Shows |
|--------------|-------|
| `/proc/<pid>/fd` | Open file descriptors |
| `/proc/<pid>/maps` | Process memory mappings |
| `/proc/meminfo` | Kernel memory stats |
| `/proc/slabinfo` | Slab allocator (inodes, dentries) |

**Code Example: Process File Objects**

```bash
lsof -p $$          # All open fds (via /proc/$$/fd)
cat /proc/$$/limits # Process limits
```

**Kernel → User Data Flow**

```
Kernel struct file* → sys_stat() → struct stat → copy_to_user() → userspace
```

****DANGEROUS** (Root Only)**

```c
int fd = open("/dev/mem", O_RDWR);
void *kernel_addr = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0xC0000000);
*(int*)kernel_addr = 42;  // Direct kernel memory write [web:220]
```

**Key Point**: **Normal apps use syscalls** for **sanitized views**. `/proc` provides **debug info**. **Direct access** = **kernel panic** risk. **Never** dereference kernel pointers from userspace. 

### 31. Which system calls access the file object (`struct file`)?

**All basic I/O calls** use the file object after lookup via fd → `current->files->fdt->fd[fd]`. 

**File Object Access Calls**

| System Call | Accesses `struct file` Field | Purpose |
|-------------|------------------------------|---------|
| `read(fd, buf, count)` | `file->f_op->read`, `file->f_pos` | Read data |
| `write(fd, buf, count)` | `file->f_op->write`, `file->f_pos` | Write data |
| **`lseek(fd, offset, whence)`** | **`file->f_pos`** | Change position |
| `close(fd)` | `file->f_count--` | Release reference |
| `fcntl(fd, F_GETFL)` | `file->f_flags` | Get file flags |
| `ioctl(fd, cmd, arg)` | `file->f_op->ioctl` | Device control |

**Internal Flow**

```
read(fd=3, buf, 10)
↓
struct file *f = fget(3)         // fd → file object
↓
ssize_t bytes = f->f_op->read(f, buf, 10, &f->f_pos);
↓
fput(f);                         // Release reference
```

**Key Point**

**`open()`** = **creates file object**
**All other calls** = **`fget(fd)` → use file object → `fput()`**

**File object fields accessed**:
- `f_op` → operation table (read/write functions)
- `f_pos` → current position  
- `f_flags`/`f_mode` → open mode/behavior
- `f_count` → reference counting

**Universal model**: Same calls → different `f_op` tables per file type. 

### 32. Which system calls are used to access the inode object?

**`stat()`, `lstat()`, `fstat()`** system calls access inode data. 

**Inode Access System Calls**

| System Call | Input | Inode Access Method | Returns |
|-------------|--------|-------------------|---------|
| **`stat(path, &buf)`** | File **path** | Path → dentry → **inode** → `struct stat` | File metadata |
| **`lstat(path, &buf)`** | File **path** | **Symlink-aware** inode lookup | Symlink metadata |
| **`fstat(fd, &buf)`** | **File descriptor** | fd → `struct file` → **inode** → `struct stat` | Open file metadata |

**Internal Flow**

```
stat("file.txt")
↓
vfs_stat() 
↓
path_lookup("file.txt") → dentry
↓
dentry → **inode** (i_op->getattr())
↓
copy_to_user(struct stat) ← size, perms, st_ino, timestamps
```

**What Gets Returned (`struct stat`)**

```c
struct stat {
    dev_t     st_dev;     // Device ID
    ino_t     st_ino;     // **Inode number**
    mode_t    st_mode;    // Type + permissions
    nlink_t   st_nlink;   // Hard links
    uid_t     st_uid;     // Owner
    off_t     st_size;    // File size
    time_t    st_mtime;   // Modify time
};
```

**Proof with `strace`**

```bash
strace -e trace=stat,lstat,fstat ls -l file.txt
```
Shows **`stat()` calls** for each file's inode data. 

**Key Point**: **`stat()` family** = **primary inode access**. All other I/O calls (`read/write`) use **file objects** that reference the inode indirectly.

### 33. How to print "Hello World" on screen using basic I/O calls?

Use `write()` system call to **stdout** (fd=1).

**Complete Program**

```c
#include <unistd.h>    // write()
#include <string.h>    // strlen()

int main() {
    const char *msg = "Hello World\n";
    write(1, msg, strlen(msg));  // fd=1 = stdout
    return 0;
}
```

**Compile & Run**

```bash
gcc -o hello hello.c
./hello
# Output: Hello World
```

**Breakdown**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| **fd=1** | `STDOUT_FILENO` | Terminal output |
| **buf** | `"Hello World\n"` | Data to print |
| **count** | `12` bytes | Length including `\n` |

**Verify with `strace`**

```bash
strace -e write ./hello
```
```
write(1, "Hello World\n", 12) = 12
```

**Key Points**

- **No `printf()`**—direct kernel call
- **fd=1** always available (inherits from parent)
- **Returns bytes written** (12) or -1 on error
- **No buffering**—immediate screen output 

**One line version**:
```c
write(1, "Hello World\n", 12);
```

### 34. Which system call do we use to duplicate the file descriptor?

**`dup()`** and **`dup2()`** system calls. 

**Duplication Methods**

| Call | Syntax | Behavior |
|------|--------|----------|
| **`dup(oldfd)`** | `int newfd = dup(5);` | **Auto-assigns lowest unused fd** |
| **`dup2(oldfd, newfd)`** | `dup2(5, 10);` | **Forces specific fd number** (closes target first) |

**Example**
```c
int fd = open("file.txt", O_RDONLY);  // fd=3
int fd2 = dup(fd);                    // fd2=4 (next available)
int fd3 = dup2(fd, 10);               // fd3=10 (exact number)

read(fd, buf, 10);   // All 3 fds point to SAME file object
read(fd2, buf, 10);  // Independent f_pos per file object
read(fd3, buf, 10);
```

**Key Properties**

- **Same `struct file` object** → **shared `f_pos`**
- **Different fd table entries** → separate indexes
- **close() one fd** → others still work (`f_count--`)

**Shell Usage (`2>&1`)**

```bash
cmd > file 2>&1    # dup2(1, 2) → stderr → stdout
```

**Return Value**

- **New fd number** on success
- **-1** on error (`EMFILE`)

**Use `dup()`** for automatic, **`dup2()`** for shell redirection. 

### 35. What are the advantages of `dup()` system calls?

`dup()` and `dup2()` provide **efficient file descriptor sharing** and **shell redirection** capabilities. 

**Key Advantages**

| Advantage | Description | Example |
|-----------|-------------|---------|
| **Zero-copy sharing** | **Same `struct file` object** → shared `f_pos`, no data duplication | Multiple fds read same stream |
| **Shell redirection** | `2>&1` → `dup2(1, 2)` | stderr → stdout |
| **Atomic replacement** | `dup2(old, new)` **closes target first** safely | `dup2(log_fd, 1)` → all printf() → log |
| **Lowest fd guarantee** | `dup()` → **fastest available slot** | Efficient fd table usage |
| **Reference counting** | close() one fd → others work (`f_count--`) | Safe cleanup |

**Real-world Examples**

**1. **Shell `ls > file 2>&1`****

```
dup2(1, 2)  # stderr → stdout → file
```
**2. **Logging Everything****

```c
int logfd = open("app.log", O_WRONLY|O_CREAT);
dup2(logfd, 1);  // stdout → log
dup2(logfd, 2);  // stderr → log
printf("Hello"); // Goes to log!
```

**3. **Pipes (parent ↔ child)****

```c
int pipefd [en.wikipedia](https://en.wikipedia.org/wiki/Dup_(system_call)); pipe(pipefd);
if (fork() == 0) {
    dup2(pipefd [youtube](https://www.youtube.com/watch?v=Uz3GnFIj-mk), 1);  // Child stdout → pipe
    execvp(cmd, args);
}
```

**Performance Benefit**

```
Without dup: Multiple open() → multiple struct file → wasted memory
With dup:    Single struct file → shared f_pos → memory efficient
```

**Primary Use**: **Shell redirection**, **pipes**, **logging**—**cheap fd duplication** without resource duplication. 

### 36. Which object stores the ownership information of a file?

**Inode object** stores file ownership information (`i_uid`, `i_gid`). 

**Ownership Fields in Inode**


| Field | Content | User Access |
|-------|---------|-------------|
| **`i_uid`** | **User ID** (numeric UID) | `stat.st_uid` |
| **`i_gid`** | **Group ID** (numeric GID) | `stat.st_gid` |

**View Ownership**

```bash
ls -l file.txt          # Shows username:groupname
stat file.txt           # Shows numeric UID/GID
ls -i file.txt          # Shows inode number
```

**C Program Access**

```c
struct stat st;
stat("file.txt", &st);
printf("Owner UID: %d\n", st.st_uid);     // Maps to username
printf("Group GID: %d\n", st.st_gid);    // Maps to groupname
```

**Storage Location**

```
Filename → Directory entry → Inode# → **inode {i_uid=1000, i_gid=1000}**
                                         ↓
                                 /etc/passwd → "seetha:x:1000:"
```

**Key Point**: **Inode** = **permanent ownership storage**. User sees **resolved names** via `/etc/passwd` + `/etc/group`. `chown()` modifies inode `i_uid/i_gid` fields directly. 

### 37. Which command displays username from a numeric UID?

**`getent passwd <UID>`** or **`id -un <UID>`**. 

**Quick Commands**

| Command | Example | Output |
|---------|---------|--------|
| **`getent passwd 1000`** | `getent passwd 1000` | `seetha:x:1000:1000:...` |
| **`id -un 1000`** | `id -un 1000` | `seetha` |
| **`awk`** | `awk -F: '$3==1000{print $1}' /etc/passwd` | `seetha` |

**Examples**

```bash
$ getent passwd 1000
seetha:x:1000:1000:Seetha Banoth:/home/seetha:/bin/bash

$ id -un 1000
seetha

$ getent passwd $(stat -c %u file.txt) | cut -d: -f1
seetha
```

**From Inode Data**

```bash
# Get UID from file inode
stat -c %u file.txt     # Prints numeric UID (1000)

# Convert UID → username  
getent passwd $(stat -c %u file.txt) | cut -d: -f1
# Output: seetha
```

**`getent`** queries NSS (local + LDAP/NIS); **`id -un`** is fastest for local users.

### 38. How can we see multiple user information in our Linux system?

Use **`/etc/passwd`** file which stores all user account details. 

**Commands to List Users**

| Command | Purpose | Output |
|---------|---------|--------|
| **`cat /etc/passwd`** | **All users** (system + regular) | Full user records |
| **`cut -d: -f1 /etc/passwd`** | **Usernames only** | `root\nbin\ndaemon\nseetha` |
| **`getent passwd`** | **All users** (local + LDAP/NIS) | Network-aware users |
| **`compgen -u`** | **Usernames** (bash built-in) | Clean username list |

**Categorize Users (System vs Normal)**

```bash
# System users (UID < 1000), Normal users (UID ≥ 1000)
awk -F: '$3 < 1000 {print "System:  ", $1} 
         $3 >=1000 {print "Normal:  ", $1}' /etc/passwd | sort
```

**Example Output**
```
System:   root
System:   bin  
System:   daemon
Normal:   seetha
Normal:   ubuntu
```

**Count Users**

```bash
# Total users
wc -l /etc/passwd          # ~70-100 lines

# Normal users only  
getent passwd {1000..60000} | wc -l
```

**Quick Summary**

```bash
echo "=== All Users ===";        cut -d: -f1 /etc/passwd
echo "=== Normal Users ===";    awk -F: '$3>=1000{print $1}' /etc/passwd
```

**`/etc/passwd` format**: `username:password:UID:GID:comment:home:shell`. 

### 39. Which command is used to change the UID and GID?

**`chown`** command. 

**Syntax & Examples**


| Usage | Command | Changes |
|-------|---------|---------|
| **User only** | `chown seetha file.txt` | UID → seetha's UID |
| **Group only** | `chown :developers file.txt` | GID → developers group |
| **Both** | `chown seetha:developers file.txt` | **UID + GID** |
| **Numeric** | `chown 1000:1001 file.txt` | UID=1000, GID=1001 |
| **Recursive** | `chown -R seetha:devs /app/` | Directory + contents |

**Verify Change**

```bash
ls -l file.txt          # Before/after
stat -c "UID:%u GID:%g" file.txt
```

**Requires Root (usually)**

```bash
sudo chown seetha file.txt
```

**Modifies Inode Fields**

```
inode.i_uid  ← new user UID
inode.i_gid  ← new group GID
```

**Key Point**: **`chown`** directly updates **inode ownership fields**. Only **root** or **file owner** can change ownership. 
