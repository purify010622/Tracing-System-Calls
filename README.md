# xv6-riscv: Tracing System Calls

This repository is a modified version of **xv6**, a re-implementation of Dennis Ritchie's and Ken Thompson's Unix Version 6 (v6). xv6 loosely follows the structure and style of v6, but is implemented for a modern RISC-V multiprocessor using ANSI C.

The main addition in this repository is a custom **`trace` system call**, which allows user-space programs to monitor and log system calls made during their execution. This is an educational modification typically done as part of the MIT 6.1810 / 6.S081 Operating System Engineering course.

## 🚀 Features

### The `trace` System Call
The `trace` system call allows you to monitor and log every system call made by a specific process and any of its children. It uses a **bitmask** to determine which system calls to intercept and log. 

**Usage Example:**
```bash
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 0
```
*(In xv6, `SYS_read` is system call `5`. `1 << 5` is `32`, so a mask of `32` traces `read` operations.)*

## 🛠️ How `trace` Works Under the Hood

The execution flow of a system call in xv6 acts like a conveyor belt across the privilege boundary:

1. **User Space**: The user program calls `trace(mask)`.
2. **Assembly Stub (`usys.S`)**: A generated assembly stub puts the `SYS_trace` number (`22`) into register `a7` and executes the `ecall` instruction.
3. **Privilege Boundary**: `ecall` transitions the CPU from User Mode to Supervisor Mode (Kernel space).
4. **Kernel Syscall Dispatcher (`syscall.c`)**: The `syscall()` multiplexer function kicks in. It maps the provided syscall number to the correct kernel function via a dispatch table `syscalls[]`.
5. **Modification (`sysproc.c`)**: The `sys_trace()` function simply reads the argument using `argint()` and stores the bitmask in the current process's `struct proc`.
6. **Logging (`syscall.c`)**: On *subsequent* system calls, the `syscall()` handler checks the logged bitmask (`p->trace_mask`). If the bit for the current syscall number is set to 1, the kernel logs the system call execution, its name, and its return value.
7. **Inheritance (`proc.c`)**: The trace mask is passed from a parent to its child processes during a `kfork()`. As a result, tracing an application like `grep` logs the operations triggered internally by `grep`. 

## 🏗️ Implementation Details

The `trace` implementation modifies the following parts of the xv6 kernel and user-space:
- **`kernel/syscall.h`**: Added `#define SYS_trace 22`.
- **`kernel/proc.h`**: Augmented `struct proc` with an `int trace_mask;` state variable for per-process filtering.
- **`kernel/sysproc.c`**: Implemented `uint64 sys_trace(void)` to pull arguments from user-space hardware registers.
- **`kernel/syscall.c`**: Added `trace` system call translation array, configured the dispatcher function table, and injected the bitmask `printf(...)` logic to intercept active active system calls.
- **`kernel/proc.c`**: Carried over the `trace_mask` during the `kfork()` sequence.
- **`user/user.h` & `user/usys.pl`**: Declared the cross-boundary `trace` interface and generated the required RISC-V assembly stubs for the ecall trap.
- **`user/trace.c`**: Built the user-space invocation program that executes the target command inside an overridden state.
- **`Makefile`**: Registered `$U/_trace` out of the source tree.

## ⚙️ Building and Running xv6

To build and run xv6, you will need a RISC-V "newlib" tool chain (e.g., from [riscv-gnu-toolchain](https://github.com/riscv/riscv-gnu-toolchain)) and `qemu` compiled for `riscv64-softmmu`. 

Once installed and added to your shell search path, start the xv6 instance:
```bash
make qemu
```
To clean the project:
```bash
make clean
```
To exit QEMU, press `Ctrl-a` followed by `x`.

## 🙏 Acknowledgments

xv6 is inspired by John Lions's *Commentary on UNIX 6th Edition (Peer to Peer Communications; ISBN: 1-57398-013-7; 1st edition (June 14, 2000))*. See the [MIT 6.1810 course page](https://pdos.csail.mit.edu/6.1810/) for pointers to on-line resources for v6.

xv6 contributors include Russ Cox, Cliff Frey, Xiao Yu, Nickolai Zeldovich, and Austin Clements.
For a fuller list, see the original commit history of `xv6-riscv`. Errors and suggestions concerning the core OS are typically directed to Frans Kaashoek and Robert Morris (kaashoek,rtm@mit.edu).
