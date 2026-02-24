## 1. Why use pthread_create instead of clone() for creating threads?**

pthread_create provides a portable, high-level POSIX interface that handles stack allocation, thread attributes, synchronization (like pthread_join), and cleanup automatically. clone() is a low-level Linux kernel syscall requiring manual flags (Cags (CLONE_VM | CLONE_FS | etc.), stack management, and futex handling for termination—error-prone and non-portable. 

**Key Reasons**
- **Portability**: pthread works across Unix systems; clone is Linux-only.
- **Abstraction**: Hides kernel details, manages TLS and resources. 
- **Safety**: Built-in error checking vs. raw syscall risks.

## 2. Why a stack grows?**

A stack grows to accommodate local variables, function parameters, and return addresses pushed onto it during nested function calls and recursion. 
### Growth Mechanism
It expands downward from high to low addresses in virtual memory. Each function call adds a stack frame; deeper calls or recursion increase usage until returns pop frames and shrink it. 
### Practical Limits
Exceeding the fixed stack size (e.g., 8MB default per thread) triggers a segmentation fault. Linux uses guard pages to detect overflows, with growth via page faults (demand-paged). 
