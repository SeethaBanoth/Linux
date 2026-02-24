# 1. What are the types of files present in Linux OS?

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

# 2.How do IPC Objects ,named pipes, be accessed?

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

# 3.How does a user space application send requests to hardware using I/O calls in Linux?

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

# 4.Why are basic I/O calls called universal I/O calls?

Basic I/O calls (`open()`, `read()`, `write()`, `close()`) are called universal because they work identically across all file types in Unix/Linux—regular files, devices, pipes, sockets, directories. [hackmd](https://hackmd.io/@happy-kernel-learning/SJDeNr7OI)

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

# 5.What is the content of the Inode Object?

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

# 6. In which Object is file information stored?

File information (metadata) is stored in the **Inode Object**. 

**Key Points**

- **Inode contains**: Permissions, owner/group IDs, file size, timestamps (atime/mtime/ctime), link count, block pointers 
- **Does NOT contain**: Filename (stored in directory entry) or actual file data 
- **Directory role**: Maps filename → Inode number

```
Filename → Directory Entry → Inode# → Metadata + Data Block Pointers
```

**stat file.txt** shows the inode data; **ls -i** shows inode number.

# 7. Which object does the kernel use to represent a file?

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

# 8. Can we access inode information from user space applications, and if yes, how?

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

# 9. Which system calls are used to access file information?

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

# 10. Which system call does the `ls` command internally invoke to access file information?

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

# 11. What happens after the kernel finds its inode object?

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

# 12. What are the contents of the `struct file` object?

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

# 13. Which object does the kernel use to represent open files?

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

# 14. What is the difference between the primary data and other members of the file object?

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
# 15. Where does the file object base address get stored?

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

# 16. When is the fd table created and what is its size?

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

# 17. When is the file object created?

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

# 18. What does `open()` return?

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

# When a file opens for the first time using `open()` in your program, what does it return?

**File descriptor `3`**. 

## Why fd=3?

```
fd 0 = stdin  (already open)
fd 1 = stdout (already open) 
fd 2 = stderr (already open)
**fd 3 = FIRST available** ← open() returns this
```

## Example

```c
int fd = open("file.txt", O_RDONLY);  // Returns 3
printf("fd = %d\n", fd);              // fd = 3
```

## Proof

```bash
strace ./your_program
```
Shows:
```
open("file.txt", O_RDONLY) = 3
```

## Subsequent opens

```
2nd open() → fd=4
3rd open() → fd=5
etc.
```

**Key Point**: `open()` **always returns the lowest unused fd** starting from **3** in a fresh process. FDs 0-2 are pre-occupied by stdio streams. 
