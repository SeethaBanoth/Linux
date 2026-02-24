**1. What are the types of files present in Linux OS?**
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

**2.How do IPC Objects ,named pipes, be accessed?**
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

**3.How does a user space application send requests to hardware using I/O calls in Linux?**

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

**4.Why are basic I/O calls called universal I/O calls?**

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
