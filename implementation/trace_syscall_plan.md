# 🔍 Implementing the `trace` System Call in xv6-RISC-V

## Complete Step-by-Step Guide

> **Goal**: Add a new `trace` system call that monitors and logs every system call
> made by a specific process, controlled by a bitmask.
>
> **Expected result**: Running `trace 32 grep hello README` inside xv6 prints lines like:
> ```
> 3: syscall read -> 1023
> 3: syscall read -> 966
> 3: syscall read -> 0
> ```

---

## 📐 Architecture Overview — How a System Call Flows in xv6

Before we touch any code, let's understand the **full journey** of a system call.
Think of it as a conveyor belt with 4 stages:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        USER SPACE (unprivileged)                        │
│                                                                         │
│  1. User program calls  trace(mask)                                     │
│  2. user.h declares:    int trace(int);                                 │
│  3. usys.pl generates:  assembly stub that does:                        │
│         li a7, SYS_trace    ← put syscall number in register a7        │
│         ecall               ← TRAP into kernel (privilege boundary!)    │
│         ret                 ← return to caller with result in a0       │
│                                                                         │
├──────────────── ecall crosses the privilege boundary ────────────────────┤
│                                                                         │
│                       KERNEL SPACE (privileged)                         │
│                                                                         │
│  4. syscall() in syscall.c reads a7 → looks up dispatch table          │
│  5. dispatch table maps SYS_trace → sys_trace function pointer         │
│  6. sys_trace() in sysproc.c:                                          │
│         - uses argint() to fetch the mask argument                     │
│         - stores mask in struct proc (the process's own data)          │
│  7. syscall() checks mask after EVERY syscall and prints if matched    │
│                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 🔑 Key Concept: What is `ecall`?

`ecall` is a RISC-V instruction that means **"environment call"**. It is the
**privilege boundary** — the single point where user code asks the kernel to do
something. When `ecall` runs:

1. The CPU **switches from User mode to Supervisor mode** (like going through
   airport security — you can't bring your own bags)
2. Control jumps to the kernel's **trap handler** (`uservec` in trampoline.S)
3. The kernel saves all user registers into the process's `trapframe`
4. The kernel reads register `a7` to find out WHICH syscall the user wants
5. The kernel calls the appropriate `sys_xxx()` function
6. The result goes into register `a0` and the CPU returns to user mode

**Why this matters for security**: User code can NEVER directly call kernel
functions. It can only ask politely via `ecall`. The kernel decides whether to
honor the request.

---

## 📋 Files We Will Modify (in Order)

| #  | File                | Role                              | What we add                          |
|----|---------------------|-----------------------------------|--------------------------------------|
| 1  | `kernel/syscall.h`  | Syscall number definitions        | `#define SYS_trace 22`               |
| 2  | `kernel/proc.h`     | Process structure definition      | `int trace_mask` field               |
| 3  | `kernel/sysproc.c`  | Kernel syscall implementations    | `sys_trace()` function               |
| 4  | `kernel/syscall.c`  | Dispatch table + syscall handler  | Register + tracing print logic       |
| 5  | `kernel/proc.c`     | Process management (fork, etc.)   | Copy mask in `kfork()`               |
| 6  | `user/user.h`       | User-space function declarations  | `int trace(int)` prototype           |
| 7  | `user/usys.pl`      | Assembly stub generator           | `entry("trace")` line                |
| 8  | `user/trace.c`      | User-space test program           | The `trace` command itself           |
| 9  | `Makefile`          | Build system                      | `$U/_trace` in UPROGS                |

> [!NOTE]
> Your original list had 8 files. We actually need **9 changes** because
> `kernel/proc.c` also needs a one-line change in the `kfork()` function to
> copy the trace mask from parent to child process. This is critical for the
> trace to work across `fork()` + `exec()` calls.

---

## Step 1 of 9: `kernel/syscall.h` — Define the Syscall Number

### 📖 What this file does
This is the **phone directory** of the kernel. Every system call gets a unique
number. When user code wants to call `read()`, it doesn't call a function
directly — it puts the number `5` in register `a7` and executes `ecall`.
The kernel then looks up number `5` to find the `sys_read` function.

### 📍 Where to make the change
Open the file and look at the **very last line**. Currently it ends with:

```c
#define SYS_close  21
```

Add your new definition right after it.

### ✏️ Exact Code to Add

```c
// --- EXISTING CODE (do NOT modify) ---
#define SYS_close  21

// --- ADD THIS LINE ---
#define SYS_trace  22  // Our new trace syscall; number 22 is the next available
```

### 🤔 Why This is Necessary
Without a number, the kernel has no way to identify your syscall in the dispatch
table. The number `22` is simply the next available integer after `21` (SYS_close).

**Analogy**: If syscalls were a Python dictionary, this is like adding a new key:
```python
syscalls = {1: "fork", 2: "exit", ..., 21: "close", 22: "trace"}  # ← new entry
```

### 🔗 How it Connects
- **Previous**: Nothing — this is the first step
- **Next**: Step 2 (`proc.h`) needs a place to store the mask per-process

### ✅ Verification
After this change alone, **nothing visible happens yet**. But you can verify:
```bash
cd ~/xv6-riscv
make kernel/kernel
```
This should compile without errors. If you see errors, check for typos.

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Using number 21 | Already taken by `SYS_close` | Use 22 |
| Forgetting the `#define` keyword | Won't create a macro | Copy the exact format above |
| Adding a semicolon at the end | `#define` doesn't use semicolons | No `;` needed |
| Naming it `SYS_TRACE` (uppercase) | Convention is `SYS_` + lowercase name | Use `SYS_trace` |

> [!IMPORTANT]
> ### 🛡️ Security Insight: Syscall Numbering as an ABI Contract
> The syscall number is part of the **Application Binary Interface (ABI)**. This
> is a contract between user programs and the kernel. Once you assign number 22
> to `trace`, every compiled program will use that number. If you later change
> it, old programs break silently (they'd call the wrong syscall!).
>
> **Real-world parallel**: Linux maintains a stable syscall table that NEVER
> changes numbers. `read` has been number 0 on x86-64 since 2003. This is why
> Linux has 400+ syscalls — they keep adding, never renumbering.

---

## Step 2 of 9: `kernel/proc.h` — Add the Trace Mask to `struct proc`

### 📖 What this file does
`proc.h` defines the **blueprint** for every process in the system. `struct proc`
is like a Python class — it describes all the data that belongs to a single
process: its PID, its state (running/sleeping/zombie), its memory, its open
files, etc.

### 🔑 Key Concept: Why `struct proc` and not a global variable?

Imagine you have 10 processes running. If you stored the trace mask in a global
variable, ALL processes would share it — tracing process A would also trace
process B, C, D... That's like putting one volume knob for all apartments in
a building!

Each process needs its **own** mask. That's why we put it inside `struct proc`.
Every process gets its own copy of this struct, so each has its own trace mask.

```
┌─────────── struct proc ───────────┐
│  pid = 3                          │
│  state = RUNNING                  │
│  name = "grep"                    │
│  trace_mask = 32  ← NEW FIELD    │  ← Only THIS process traces SYS_read
│  ofile[16] = { ... }             │
│  ...                              │
└───────────────────────────────────┘

┌─────────── struct proc ───────────┐
│  pid = 4                          │
│  state = RUNNABLE                 │
│  name = "sh"                      │
│  trace_mask = 0   ← NEW FIELD    │  ← This process does NOT trace anything
│  ofile[16] = { ... }             │
│  ...                              │
└───────────────────────────────────┘
```

### 📍 Where to make the change
Open `kernel/proc.h` and find `struct proc` (starts at **line 85**). Look for
the last field before the closing `};`:

```c
  char name[16];               // Process name (debugging)
};
```

Add your new field **right before** the closing brace.

### ✏️ Exact Code to Add

```c
  // --- EXISTING CODE (do NOT modify) ---
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  // --- ADD THIS LINE ---
  int trace_mask;              // Bitmask: if bit N is set, syscall N is traced
};
```

### 🔑 Key Concept: Why `->` instead of `.` when accessing struct proc?

In C, you access struct fields with `.` (dot) when you have the struct directly,
and `->` (arrow) when you have a **pointer** to the struct.

In xv6, `myproc()` returns a **pointer** (`struct proc *`), not the struct
itself. Why? Because the actual `struct proc` lives in a global array
(`proc[NPROC]` in proc.c), and we just have an address pointing to it.

```c
// Python analogy:
process = get_current_process()  # process is a reference (pointer)
mask = process.trace_mask        # Python uses . for everything

// C equivalent:
struct proc *p = myproc();       // p is a pointer
int mask = p->trace_mask;        // C uses -> for pointers to structs
// p->trace_mask is shorthand for (*p).trace_mask
```

### 🤔 Why This is Necessary
The `sys_trace()` function (Step 3) needs somewhere to **store** the mask.
The `syscall()` function (Step 4) needs somewhere to **read** the mask.
Both access the same `struct proc` for the current process.

### 🔗 How it Connects
- **Previous**: Step 1 gave us the syscall number `SYS_trace = 22`
- **Next**: Step 3 will write to `p->trace_mask`, Step 4 will read from it

### ✅ Verification
```bash
cd ~/xv6-riscv
make kernel/kernel
```
Should compile without errors. The new field is just declared — nothing uses
it yet.

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Putting it outside `struct proc { }` | Would be a global variable | Place inside the braces |
| Using `char` instead of `int` | A `char` is only 8 bits; mask could be larger | Use `int` (32 bits) |
| Naming it `mask` (too generic) | Could conflict with other variables | Use `trace_mask` |
| Forgetting the semicolon after the field | C requires `;` after struct fields | Add `;` |

> [!IMPORTANT]
> ### 🛡️ Security Insight: Per-Process State Isolation
> Storing the mask in `struct proc` is an example of **process isolation**. Each
> process's tracing configuration is completely independent. A malicious process
> cannot see or modify another process's trace mask because each `struct proc`
> is only accessible by the kernel (it lives in kernel memory, not user memory).
>
> **Real-world parallel**: Linux's `seccomp` (Secure Computing) filter works
> similarly — each process has its own seccomp policy stored in its
> `task_struct` (Linux's equivalent of `struct proc`). Process A's seccomp
> policy cannot affect process B.

---

## Step 3 of 9: `kernel/sysproc.c` — Implement `sys_trace()`

### 📖 What this file does
`sysproc.c` contains the **kernel-side implementations** of process-related
system calls. When user code calls `trace(32)`, the kernel eventually runs the
`sys_trace()` function defined here. This is where the actual work happens.

### 🔑 Key Concept: What is `argint()` and why do we need it?

When the user calls `trace(32)`, the number `32` is placed in register `a0`
**before** the `ecall`. But functions on the kernel side can't just read
arguments from their own function parameters — the `ecall` instruction doesn't
"pass" arguments in the C function-call sense.

Instead, `argint()` reaches into the process's saved registers (the `trapframe`)
and extracts the argument:

```
User space:              trace(32)
                           ↓
                    a0 = 32  (in register)
                           ↓
                        ecall  ← traps into kernel, saves all registers
                           ↓
Kernel space:        sys_trace(void)  ← note: NO parameters!
                           ↓
                    argint(0, &mask)  ← reads a0 from trapframe → mask = 32
```

**Why can't we just write `sys_trace(int mask)`?** Because the `ecall`
instruction doesn't call C functions — it triggers a hardware trap. The kernel
trap handler always calls `sys_xxx()` with **no arguments**. We must manually
extract them from the saved registers.

`argint(0, &mask)` means: "Get the 0th argument (which is in register `a0`)
and store it in the variable `mask`."

### 🔑 Key Concept: What does `myproc()` return?

`myproc()` returns a **pointer** to the `struct proc` of the currently running
process. Think of it as Python's `self` — it tells you "which process is
asking."

```c
struct proc *p = myproc();
// p now points to the struct proc of the process that called trace()
// p->pid is its process ID
// p->trace_mask is where we'll store the mask
```

Looking at the actual code in `proc.c` (line 82-90):
```c
struct proc*
myproc(void)
{
  push_off();             // disable interrupts (so we don't move CPUs mid-read)
  struct cpu *c = mycpu(); // get the current CPU
  struct proc *p = c->proc; // get the process running on this CPU
  pop_off();              // re-enable interrupts
  return p;               // return pointer to current process
}
```

### 📍 Where to make the change
Open `kernel/sysproc.c`. Add the new function at the **very end** of the file,
after the existing `sys_uptime()` function (after line 109).

### ✏️ Exact Code to Add

```c
// --- EXISTING CODE at the end of the file (do NOT modify) ---
// return how many clock tick interrupts have occurred
// since start.
uint64
sys_uptime(void)
{
  uint xticks;

  acquire(&tickslock);
  xticks = ticks;
  release(&tickslock);
  return xticks;
}

// --- ADD EVERYTHING BELOW THIS LINE ---

// sys_trace: Set the trace mask for the current process.
// The mask determines which system calls to log.
// Called by user code: trace(mask)
uint64
sys_trace(void)
{
  int mask;

  // Extract the first argument (register a0) from the trapframe.
  // argint(0, &mask) means: "get argument #0 and store it in 'mask'"
  // If the user called trace(32), then mask will be 32.
  argint(0, &mask);

  // Store the mask in the current process's struct proc.
  // myproc() returns a pointer to the current process.
  // We use -> because myproc() returns a pointer (struct proc *).
  myproc()->trace_mask = mask;

  // Return 0 to indicate success.
  // This value will be placed in register a0 and returned to user space.
  return 0;
}
```

### 🤔 Why This is Necessary
This is the actual "brain" of the `trace` syscall. Without it, the kernel has no
code to run when the user calls `trace()`. The function's job is simple:
1. Read the mask from the user's arguments
2. Store it in the process's data structure

The **printing** of trace output happens elsewhere (Step 4) — this function
just sets up the configuration.

### 🔗 How it Connects
- **Previous**: Step 2 gave us the `trace_mask` field to write to
- **Next**: Step 4 will register this function in the dispatch table and add
  print logic that reads `trace_mask`

### ✅ Verification
```bash
cd ~/xv6-riscv
make kernel/kernel
```
**Expected**: This will produce an "unused function" warning or error because
`sys_trace` isn't registered in the dispatch table yet. That's okay — we'll
fix it in Step 4- but it should still compile.

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Writing `sys_trace(int mask)` | Kernel syscall functions take `void` | Use `(void)` |
| Forgetting `argint()` | `mask` would be uninitialized garbage | Always use `argint` |
| Using `argint(1, &mask)` | Argument 0 = `a0`, argument 1 = `a1` | First arg is `0` |
| Writing `return mask` | Convention is to return 0 for success | Return `0` |
| Forgetting `#include "proc.h"` | Can't access `struct proc` | Already included at top of file ✓ |

> [!IMPORTANT]
> ### 🛡️ Security Insight: Input Validation at the Kernel Boundary
> Notice that we accept **any** integer as the mask without validation. In a
> production kernel, you might want to check:
> - Is the mask a valid set of syscall bits?
> - Does the calling process have permission to trace?
>
> **Real-world parallel**: Linux's `ptrace()` system call (which `strace` uses)
> requires the caller to have `CAP_SYS_PTRACE` capability or be the parent of
> the traced process. Our `trace` is simpler — any process can trace itself,
> which is actually a safer design (self-tracing only).

---

## Step 4 of 9: `kernel/syscall.c` — Register in Dispatch Table + Add Print Logic

### 📖 What this file does
`syscall.c` is the **central switchboard** of the kernel. It contains:
1. **The dispatch table** — an array of function pointers that maps syscall
   numbers to their handler functions
2. **The `syscall()` function** — the main entry point called by the trap
   handler every time a user program makes a system call

### 🔑 Key Concept: The Syscall Dispatch Table (like a Python dictionary)

In Python, you might write:
```python
handlers = {
    1: fork_handler,
    2: exit_handler,
    # ...
    22: trace_handler,  # ← we're adding this
}

def syscall(num):
    result = handlers[num]()  # call the function
```

In C, xv6 does exactly the same thing but with an **array of function pointers**:
```c
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,     // syscalls[1] = pointer to sys_fork
[SYS_exit]    sys_exit,     // syscalls[2] = pointer to sys_exit
// ...
[SYS_trace]   sys_trace,    // syscalls[22] = pointer to sys_trace  ← NEW
};
```

The `[SYS_fork]` syntax is a **C designated initializer** — it means "put this
value at index `SYS_fork` (which is 1)". It's like Python's `{1: func}`.

### 🔑 Key Concept: How bitmask tracing works

A **bitmask** is a number where individual bits act as on/off switches.

```
mask = 32  →  binary: 0 0 1 0 0 0 0 0
                       │ │ │ │ │ │ │ │
                bit:   7 6 5 4 3 2 1 0

Bit 5 is ON → trace syscall number 5 (SYS_read)
```

To check if syscall number `num` should be traced:
```c
if (p->trace_mask & (1 << num))
```

Let's trace through this with `mask = 32` and `num = 5` (SYS_read):
```
1 << 5  = 32  = binary 00100000
mask    = 32  = binary 00100000
                       ────────
32 & 32 = 32  = binary 00100000  ← NOT zero, so the condition is TRUE → trace!
```

And with `num = 7` (SYS_exec):
```
1 << 7  = 128 = binary 10000000
mask    = 32  = binary 00100000
                       ────────
128 & 32 = 0  = binary 00000000  ← IS zero, so the condition is FALSE → don't trace
```

**Python equivalent**: `if mask & (1 << num) != 0:`

You can set **multiple** bits: `mask = 2147483647` (all bits set) would trace
every syscall.

### 📍 Where to make the change (THREE changes in this file)

This file needs **three** modifications:

#### Change 4A: Add the `extern` declaration (around line 103)
#### Change 4B: Add to the dispatch table (around line 128)
#### Change 4C: Add a syscall names array AND modify the `syscall()` function (around line 131)

### ✏️ Exact Code — Change 4A: Extern Declaration

Find the block of `extern` declarations (lines 83-103). Add one line at the end:

```c
// --- EXISTING CODE (do NOT modify) ---
extern uint64 sys_link(void);
extern uint64 sys_mkdir(void);
extern uint64 sys_close(void);

// --- ADD THIS LINE ---
extern uint64 sys_trace(void);   // Tell the compiler: sys_trace exists in another file
```

**Why**: The compiler needs to know `sys_trace` exists before we use it in the
dispatch table. `extern` means "this function is defined in a different `.c`
file" (it's in `sysproc.c`).

### ✏️ Exact Code — Change 4B: Dispatch Table Entry

Find the dispatch table (lines 107-129). Add one line before the closing `};`:

```c
// --- EXISTING CODE (do NOT modify) ---
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,

// --- ADD THIS LINE ---
[SYS_trace]   sys_trace,    // Map SYS_trace (22) to the sys_trace function
};
```

### ✏️ Exact Code — Change 4C: Syscall Names Array + Modified `syscall()` Function

This is the biggest change. We need:
1. A **names array** to convert syscall numbers to readable names (like "read")
2. Modified **`syscall()` function** with tracing logic

**Replace** the entire `syscall()` function (lines 131-147) with:

```c
// Human-readable names for syscalls, indexed by syscall number.
// Used by the trace feature to print which syscall was called.
// This is like a Python list: names[5] = "read"
static char *syscall_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_pause]   "pause",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace",
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  // Read the syscall number from register a7.
  // The user-space stub put it there before ecall.
  num = p->trapframe->a7;

  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();

    // === TRACE LOGIC (NEW) ===
    // After the syscall has executed, check if this process wants
    // to trace this particular syscall.
    //
    // p->trace_mask is the bitmask set by the trace() syscall.
    // (1 << num) creates a number with only bit 'num' set.
    // The & (bitwise AND) checks if that specific bit is on in the mask.
    //
    // Example: mask=32 (binary 100000), num=5 (SYS_read)
    //   (1 << 5) = 32 = binary 100000
    //   32 & 32  = 32 → non-zero → TRUE → print!
    if(p->trace_mask & (1 << num)) {
      // Print: "<pid>: syscall <name> -> <return_value>"
      printf("%d: syscall %s -> %d\n",
             p->pid,                      // process ID
             syscall_names[num],          // human-readable name
             (int)p->trapframe->a0);      // return value of the syscall
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

### 🤔 Why This is Necessary
- **4A (extern)**: Without it, the compiler doesn't know `sys_trace` exists
- **4B (dispatch table)**: Without it, calling `trace()` gets "unknown syscall"
- **4C (print logic)**: Without it, the mask is set but nothing is ever printed

### 🔗 How it Connects
- **Previous**: Steps 1-3 gave us the number, the storage, and the handler
- **Next**: Step 5 will copy the mask during `fork()` so child processes inherit it

### ✅ Verification
```bash
cd ~/xv6-riscv
make kernel/kernel
```
Should compile cleanly now. The kernel side is complete!

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Forgetting the `extern` declaration | Linker error: "undefined reference" | Add the `extern` line |
| Missing comma in dispatch table | Syntax error | Each entry ends with `,` |
| Printing BEFORE calling `syscalls[num]()` | Return value isn't available yet | Print AFTER the call |
| Using `p->trace_mask & num` | Wrong! Should be `(1 << num)` | Use bitshift |
| Forgetting `"trace"` in names array | Would print `(null)` for trace calls | Add it |

> [!IMPORTANT]
> ### 🛡️ Security Insight: Audit Logging and the Syscall Interception Point
> We're adding trace logic **inside** the `syscall()` function — the single
> chokepoint through which ALL system calls pass. This is exactly how real
> audit frameworks work:
>
> - **Linux Audit Framework** (`auditd`): Hooks into the syscall entry/exit
>   path to log all system calls matching a rule set
> - **seccomp-BPF**: Intercepts syscalls at this same point to BLOCK them
>   (not just log them)
> - **Windows ETW** (Event Tracing for Windows): Similar concept for Win32 API
>   calls
>
> Our tracing is **read-only observation** (logging). A real security tool
> could also **modify** behavior here — blocking dangerous syscalls, faking
> return values, etc.

---

## Step 5 of 9: `kernel/proc.c` — Copy Mask in `kfork()`

### 📖 What this file does
`proc.c` manages the lifecycle of processes: creating them (`allocproc`),
forking them (`kfork`), exiting them (`kexit`), and scheduling them
(`scheduler`).

### 🤔 Why This Step is Needed
When the user runs `trace 32 grep hello README`, the `trace` program:
1. Calls `trace(32)` — sets `trace_mask = 32` on itself
2. Calls `fork()` — creates a child process
3. In the child, calls `exec("grep", ...)` — replaces itself with grep

The child process needs to **inherit** the parent's `trace_mask`. If we don't
copy it during `fork()`, the child starts with `trace_mask = 0` (no tracing),
and we'd never see any output!

```
┌─────────── trace process ──────────┐
│  pid = 2                           │
│  trace_mask = 32                   │
│                                    │
│  1. trace(32)  → sets mask         │
│  2. fork()     → creates child     │──────┐
│  3. wait()     → waits for child   │      │
└────────────────────────────────────┘      │
                                            ↓
                              ┌─────────── child process ────────────┐
                              │  pid = 3                             │
                              │  trace_mask = 32  ← COPIED from     │
                              │                     parent!          │
                              │  exec("grep", ...) → runs grep      │
                              │  (grep calls read(), write(), etc.)  │
                              │  → kernel prints: 3: syscall read…  │
                              └──────────────────────────────────────┘
```

### 📍 Where to make the change
Open `kernel/proc.c` and find the `kfork()` function (starts at **line 259**).
Look for the line that copies the process name (line 291):

```c
  safestrcpy(np->name, p->name, sizeof(p->name));
```

Add the mask copy **right after this line**.

### ✏️ Exact Code to Add

```c
  // --- EXISTING CODE (do NOT modify) ---
  safestrcpy(np->name, p->name, sizeof(p->name));

  // --- ADD THIS LINE ---
  np->trace_mask = p->trace_mask;  // Inherit trace mask from parent to child
```

### 🔗 How it Connects
- **Previous**: Step 4 completed the kernel-side syscall machinery
- **Next**: Steps 6-9 will build the user-space side so users can actually call `trace()`

### ✅ Verification
```bash
cd ~/xv6-riscv
make kernel/kernel
```
Should compile cleanly. The complete kernel side is now done!

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Forgetting this step entirely | Child process would have `mask = 0` → no output | Add the copy line |
| Putting it after `release(&np->lock)` | The child might already be running! | Put it before the first `release` |
| Copying in `allocproc()` instead | New processes shouldn't inherit from random old data | Copy in `kfork()` from parent |

> [!IMPORTANT]
> ### 🛡️ Security Insight: State Inheritance During fork()
> `fork()` copies almost everything from parent to child: memory, open files,
> environment. Our trace mask follows the same pattern. In real operating
> systems, this inheritance model has security implications:
>
> - **Linux**: `prctl(PR_SET_SECCOMP)` filters are inherited across `fork()`
>   but survive `exec()` — this is how sandboxing works (Chrome's renderer
>   processes inherit the sandbox from the browser process)
> - **Capability inheritance**: Which privileges a child gets from its parent
>   is one of the most security-critical design decisions in an OS
>
> Our trace mask inherits across both `fork()` AND `exec()` — the `exec()`
> call replaces the program code but keeps the `struct proc` (and thus the
> mask). This is by design: we want to trace what the exec'd program does.

---

## Step 6 of 9: `user/user.h` — Declare User-Space Prototype

### 📖 What this file does
`user.h` is the **public API** of xv6 for user-space programs. Any function
listed here can be called from any user program. It's like Python's `import`
— if you want to use `trace()`, it must be declared here.

### 📍 Where to make the change
Find the system calls section. Look for `int uptime(void);` (line 26).
Add your declaration right after it.

### ✏️ Exact Code to Add

```c
// --- EXISTING CODE (do NOT modify) ---
int pause(int);
int uptime(void);

// --- ADD THIS LINE ---
int trace(int);              // Declare trace(mask) for user programs
```

### 🤔 Why This is Necessary
Without this declaration, any user program that calls `trace(32)` would get a
compiler error: "implicit declaration of function 'trace'". The prototype tells
the compiler:
- `trace` exists
- It takes one `int` argument
- It returns an `int`

### 🔗 How it Connects
- **Previous**: The kernel side is complete
- **Next**: Step 7 generates the assembly stub that actually executes `ecall`

### ✅ Verification
No separate compilation step needed — this will be tested in Step 9.

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Writing `void trace(int)` | Should return `int` (0 for success) | Use `int trace(int)` |
| Putting it in the "ulib.c" section | It's a syscall, not a library function | Put it with other syscalls |
| Forgetting the semicolon | Syntax error | End with `;` |

> [!IMPORTANT]
> ### 🛡️ Security Insight: User-Space API Surface
> Every function in `user.h` represents an **attack surface** — a way for user
> code to interact with the kernel. Keeping this list small and well-audited is
> a fundamental security practice.
>
> **Real-world parallel**: Android restricts which syscalls apps can use via
> seccomp-BPF. The allowed set is carefully curated to minimize attack surface.
> Adding a new syscall is always a security decision.

---

## Step 7 of 9: `user/usys.pl` — Add Assembly Stub Entry

### 📖 What this file does
`usys.pl` is a **Perl script** that auto-generates assembly code (`usys.S`).
For each system call, it generates a tiny function (a "stub") that:
1. Loads the syscall number into register `a7`
2. Executes the `ecall` instruction to trap into the kernel
3. Returns the result (which is in register `a0`)

This is what the generated assembly for `trace` will look like:
```asm
.global trace           # Make this function visible to the linker
trace:                  # Label: the address of the trace function
  li a7, SYS_trace      # Load Immediate: put 22 into register a7
  ecall                 # Trap to kernel! (the privilege boundary)
  ret                   # Return to caller; result is in a0
```

### 🔑 Key Concept: Why do we need assembly stubs?

In Python, you can call `os.read()` directly. But in a real OS, user code
can't call kernel functions — they live in different memory spaces with
different privileges. The assembly stub is the "bridge":

```
Your C code:    trace(32)
      ↓
C compiler converts to:  call trace   (jump to the trace label)
      ↓
Assembly stub:  trace:
                  li a7, 22    ← "I want syscall #22"
                  ecall        ← "Dear kernel, please handle this"
                  ret          ← kernel has put the result in a0
      ↓
Back in your C code: the return value of trace() is now in a0
```

### 📍 Where to make the change
Open `user/usys.pl`. Find the last entry (line 44):

```perl
entry("uptime");
```

Add your new entry right after it.

### ✏️ Exact Code to Add

```perl
# --- EXISTING CODE (do NOT modify) ---
entry("pause");
entry("uptime");

# --- ADD THIS LINE ---
entry("trace");
```

### 🤔 Why This is Necessary
Without this, there's no assembly function to bridge the gap between user-space
C code and the kernel. The linker would fail with "undefined reference to
`trace`" when building the user-space program.

### 🔗 How it Connects
- **Previous**: Step 6 declared the function prototype
- **Next**: Step 8 writes the user program that calls `trace()`

### ✅ Verification
```bash
cd ~/xv6-riscv
make user/usys.S
```
Then check the generated file:
```bash
grep -A 3 "trace" user/usys.S
```
You should see:
```
.global trace
trace:
 li a7, SYS_trace
 ecall
 ret
```

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Writing `entry("SYS_trace")` | The script adds `SYS_` prefix automatically | Just use `"trace"` |
| Forgetting the semicolon after `entry(...)` | Perl syntax error | End with `;` |
| Adding to the wrong file (`.S` instead of `.pl`) | `.S` is auto-generated and will be overwritten | Edit the `.pl` file |

> [!IMPORTANT]
> ### 🛡️ Security Insight: The `ecall` Instruction as the Security Boundary
> The `ecall` instruction is the **only** legal way for user code to request
> kernel services. The CPU hardware enforces this:
>
> - If user code tries to access kernel memory → **page fault** (crash)
> - If user code tries to execute privileged instructions → **illegal
>   instruction fault** (crash)
> - The ONLY way in is through `ecall`, which transfers control to a
>   **kernel-controlled** handler
>
> **Real-world parallel**: This is identical to `syscall` on x86-64,
> `svc` on ARM, and `int 0x80` on older x86 Linux. Every architecture
> has exactly one "door" from user space to kernel space.

---

## Step 8 of 9: `user/trace.c` — Write the User-Space Program

### 📖 What this file does
This is the **user-facing command** — the program you actually type in the xv6
shell. It parses command-line arguments, calls `trace()` to set the mask, then
`fork()` + `exec()` to run the target program.

### 📍 Where to create the file
Create a **new file**: `user/trace.c`

### ✏️ Exact Code to Write

```c
//
// trace.c - User-space program for the trace system call.
//
// Usage:  trace <mask> <command> [args...]
// Example: trace 32 grep hello README
//
// The mask is a bitmask. If bit N is set, syscall number N is traced.
// Example: mask=32 = binary 100000 → bit 5 is set → SYS_read (5) is traced.
//

#include "kernel/param.h"    // MAXARG - maximum number of exec arguments
#include "kernel/types.h"    // uint, uint64 type definitions
#include "kernel/stat.h"     // struct stat (needed by user.h)
#include "user/user.h"       // system call declarations (trace, fork, exec, etc.)

int
main(int argc, char *argv[])
{
  // --- Argument validation ---
  // We need at least 3 arguments:
  //   argv[0] = "trace"    (program name, always present)
  //   argv[1] = "32"       (the mask)
  //   argv[2] = "grep"     (the command to run)
  //   argv[3+] = optional additional args for the command
  if(argc < 3){
    // fprintf(2, ...) prints to stderr (file descriptor 2)
    fprintf(2, "Usage: trace mask command [args...]\n");
    exit(1);  // Exit with error code 1
  }

  // --- Set the trace mask ---
  // atoi() converts a string to an integer: "32" → 32
  // trace() is OUR new system call — it sets trace_mask in struct proc
  int mask = atoi(argv[1]);
  if(trace(mask) < 0){
    fprintf(2, "trace: trace syscall failed\n");
    exit(1);
  }

  // --- Execute the target command ---
  // exec() replaces the current process with a new program.
  // argv + 2 skips past "trace" and "32" to get ["grep", "hello", "README"]
  //
  // We don't need fork() here because exec() replaces the current process.
  // The trace mask survives exec() because it's stored in struct proc,
  // which is NOT replaced by exec (only the program code and memory are).
  exec(argv[2], argv + 2);

  // If exec() returns, it means it FAILED (e.g., command not found).
  // On success, exec() never returns — the new program starts running.
  fprintf(2, "trace: exec %s failed\n", argv[2]);
  exit(1);
}
```

### 🤔 Why This Design (exec without fork)?

You might wonder: "Don't we usually fork before exec?" In many programs,
yes. But the `trace` command itself doesn't need to survive — its only
job is to:
1. Set the mask
2. Replace itself with the target program

Since `exec()` preserves the `struct proc` (including `trace_mask`),
the target program inherits the mask. No fork needed in the `trace`
program itself.

However, the traced program (e.g., `grep`) might internally call
`fork()` — and that's why Step 5 copies the mask in `kfork()`.

### 🔗 How it Connects
- **Previous**: Steps 1-7 built the full kernel-to-user pipeline
- **Next**: Step 9 registers this program in the Makefile so it gets built

### ✅ Verification
Can't test yet — need to add to Makefile first (Step 9).

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Including `<stdio.h>` | xv6 is NOT Linux — no standard library | Use `user/user.h` |
| Using `printf(...)` for errors | Should use `fprintf(2, ...)` for stderr | Use fd 2 for errors |
| Forgetting `exit(1)` after failed exec | Process would fall through | Always call `exit()` |
| Writing `exec(argv[2], argv)` | Would pass "trace" and "32" to grep | Use `argv + 2` |
| Using `return 0` instead of `exit(0)` | xv6 user programs must call `exit()` | Use `exit()` |

> [!IMPORTANT]
> ### 🛡️ Security Insight: exec() and State Persistence
> When `exec()` replaces a process's program, some state is preserved (PID,
> open files, trace mask) and some is wiped (memory, code, stack). This is a
> deliberate security design:
>
> - **Preserved**: File descriptors → allows shell I/O redirection
> - **Preserved**: PID, trace mask → allows tracing across exec
> - **Wiped**: User memory → prevents information leakage to the new program
>
> **Real-world parallel**: Linux's `execve()` preserves signal dispositions,
> file descriptors (unless marked close-on-exec), and certain `prctl` flags.
> The exact set of preserved vs. wiped state is a careful security decision
> documented in `man execve`.

---

## Step 9 of 9: `Makefile` — Register `trace` in UPROGS

### 📖 What this file does
The Makefile is the **build recipe** for xv6. The `UPROGS` variable lists every
user program that should be compiled and included in the filesystem image
(`fs.img`). If your program isn't listed here, it simply won't exist inside xv6.

### 📍 Where to make the change
Find the `UPROGS` list (starts at **line 128**). Look for the last entry before
the empty line:

```makefile
	$U/_dorphan\
```

This line does NOT end with `\` → it's the last entry. We need to:
1. Add `\` to this line (to continue the list)
2. Add our new entry on the next line

### ✏️ Exact Code to Modify

```makefile
# --- EXISTING CODE: change this line ---
	$U/_dorphan\
# (was previously the last line without a backslash)

# --- TO THIS (add backslash + new line) ---
	$U/_dorphan\
	$U/_trace\
```

The complete UPROGS section should look like:

```makefile
UPROGS=\
	$U/_cat\
	$U/_hello\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_logstress\
	$U/_forphan\
	$U/_dorphan\
	$U/_trace\
```

> [!CAUTION]
> The `\` at the end of each line is a **line continuation** in Makefiles. It
> means "this line continues on the next line." If you forget the `\` on the
> line BEFORE yours, the build system won't see your entry!
>
> Also: **use TABS, not spaces** for indentation in Makefiles. This is a
> notorious gotcha — Make requires literal tab characters.

### 🤔 Why This is Necessary
Without this, the build system doesn't know about `trace.c`. The program won't
be compiled, won't be included in `fs.img`, and won't exist inside xv6.

### 🔗 How it Connects
- **Previous**: Step 8 created the source file
- **Next**: This is the final step — now we can build and test!

### ✅ Verification
This is the **full end-to-end test**:

```bash
# Step 1: Clean and rebuild everything
cd ~/xv6-riscv
make clean
make qemu

# Step 2: Wait for xv6 to boot (you'll see the $ prompt)

# Step 3: Run the trace command
$ trace 32 grep hello README

# Step 4: Expected output (PID may differ):
# 3: syscall read -> 1023
# 3: syscall read -> 966
# 3: syscall read -> 0

# Step 5: Test with a different mask (trace write = SYS_write = 16, mask = 1<<16 = 65536)
$ trace 65536 grep hello README
# Should show write syscalls instead of read

# Step 6: Test tracing ALL syscalls (mask with all bits set)
$ trace 2147483647 grep hello README
# Should show every syscall grep makes

# Step 7: Exit QEMU
# Press Ctrl+A, then X
```

### ⚠️ Common Mistakes
| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Using spaces instead of tab | Makefile requires REAL tabs | Press Tab key |
| Forgetting `\` on the line above | Line continuation broken | Add `\` |
| Writing `$U/trace` instead of `$U/_trace` | xv6 convention: binaries have `_` prefix | Use `_trace` |
| Not running `make clean` first | Old object files may conflict | Always `make clean` first |

> [!IMPORTANT]
> ### 🛡️ Security Insight: Build System as Security Control
> The Makefile controls which programs are available in the OS. In production
> systems, the build configuration is a **security boundary**:
>
> - **Minimal installs**: Production servers should only include necessary
>   programs (reducing attack surface)
> - **Build reproducibility**: Security audits require knowing EXACTLY what
>   went into the build
> - **Supply chain security**: Each entry in UPROGS is a piece of code that
>   gets root-level trust inside xv6
>
> Tools like `trace` could be a security concern in production — you might
> want to remove it from release builds (like how Linux distros don't ship
> `strace` by default).

---

## 🗺️ Complete Data Flow Diagram

Here's the complete journey when a user types `trace 32 grep hello README`:

```
                          USER SPACE
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Shell (sh.c) parses "trace 32 grep hello README"        │
│       ↓                                                  │
│  fork() → child runs exec("trace", ["trace","32",        │
│                                      "grep","hello",     │
│                                      "README"])          │
│       ↓                                                  │
│  trace.c:main()                                          │
│    │                                                     │
│    ├─ trace(32)  ──────────── ecall ──────────────────────┼───┐
│    │   ← returns 0                                       │   │
│    │                                                     │   │
│    └─ exec("grep", ["grep","hello","README"])             │   │
│         ↓                                                │   │
│  grep.c:main()  (trace_mask=32 is still in struct proc)  │   │
│    │                                                     │   │
│    ├─ open("README")  ────── ecall ──────────────────────┼───┼─┐
│    │   ← returns fd=3       (mask & (1<<15)=0 → silent)  │   │ │
│    │                                                     │   │ │
│    ├─ read(3, buf, 1024)  ── ecall ──────────────────────┼───┼─┼─┐
│    │   ← returns 1023       (mask & (1<<5)=32 → PRINT!)  │   │ │ │
│    │   stdout: "3: syscall read -> 1023"                 │   │ │ │
│    │                                                     │   │ │ │
│    ├─ read(3, buf, 1024)  ── ecall ──────────────────────┼───┼─┼─┤
│    │   ← returns 966        (mask & (1<<5)=32 → PRINT!)  │   │ │ │
│    │                                                     │   │ │ │
│    └─ read(3, buf, 1024)  ── ecall ──────────────────────┼───┼─┼─┤
│        ← returns 0          (mask & (1<<5)=32 → PRINT!)  │   │ │ │
│                                                          │   │ │ │
└──────────────────────────────────────────────────────────┘   │ │ │
                                                               │ │ │
═══════════════════ PRIVILEGE BOUNDARY (ecall) ═════════════════╪═╪═╪═
                                                               │ │ │
                         KERNEL SPACE                          │ │ │
┌──────────────────────────────────────────────────────────┐   │ │ │
│                                                          │   │ │ │
│  syscall() in syscall.c                                  │←──┘ │ │
│    ├─ reads a7 → SYS_trace (22)                         │     │ │
│    ├─ calls syscalls[22] → sys_trace()                   │     │ │
│    │   └─ argint(0, &mask) → mask = 32                  │     │ │
│    │   └─ myproc()->trace_mask = 32                     │     │ │
│    └─ check (32 & (1<<22))? No → silent                 │     │ │
│                                                          │     │ │
│  syscall() for open                                      │←────┘ │
│    ├─ reads a7 → SYS_open (15)                          │       │
│    ├─ calls syscalls[15] → sys_open()                    │       │
│    └─ check (32 & (1<<15))? No → silent                 │       │
│                                                          │       │
│  syscall() for read                                      │←──────┘
│    ├─ reads a7 → SYS_read (5)                           │
│    ├─ calls syscalls[5] → sys_read()                     │
│    └─ check (32 & (1<<5))? 32≠0 → PRINT! ✓             │
│       printf("3: syscall read -> 1023\n")               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## ✅ Final Checklist

Use this checklist to track your progress:

- [ ] **Step 1**: `kernel/syscall.h` — Add `#define SYS_trace 22`
- [ ] **Step 2**: `kernel/proc.h` — Add `int trace_mask;` to `struct proc`
- [ ] **Step 3**: `kernel/sysproc.c` — Add `sys_trace()` function at the end
- [ ] **Step 4A**: `kernel/syscall.c` — Add `extern uint64 sys_trace(void);`
- [ ] **Step 4B**: `kernel/syscall.c` — Add `[SYS_trace] sys_trace,` to dispatch table
- [ ] **Step 4C**: `kernel/syscall.c` — Add `syscall_names[]` array + modify `syscall()` function
- [ ] **Step 5**: `kernel/proc.c` — Add `np->trace_mask = p->trace_mask;` in `kfork()`
- [ ] **Step 6**: `user/user.h` — Add `int trace(int);` declaration
- [ ] **Step 7**: `user/usys.pl` — Add `entry("trace");`
- [ ] **Step 8**: `user/trace.c` — Create the user-space trace program (NEW FILE)
- [ ] **Step 9**: `Makefile` — Add `$U/_trace\` to UPROGS

## 🏁 Final Verification

After completing all steps:

```bash
cd ~/xv6-riscv
make clean
make qemu
```

Inside xv6:
```bash
# Test 1: Trace only read syscalls (SYS_read=5, mask = 1<<5 = 32)
$ trace 32 grep hello README

# Expected output (PID may vary):
# 3: syscall read -> 1023
# 3: syscall read -> 966
# 3: syscall read -> 0

# Test 2: Trace all syscalls
$ trace 2147483647 grep hello README

# Should show fork, exec, open, read, write, close, etc.

# Test 3: Invalid usage (should print usage message)
$ trace

# Expected: Usage: trace mask command [args...]
```

To exit QEMU: press **Ctrl+A**, then **X**.

---

## 📚 Concept Summary Table

| Concept | What it means | Where it appears |
|---------|--------------|-----------------|
| `ecall` | RISC-V instruction to trap into kernel | `usys.pl` generated stubs |
| `argint()` | Extracts syscall arguments from saved registers | `sysproc.c` (`sys_trace`) |
| `myproc()` | Returns pointer to current process's `struct proc` | `sysproc.c`, `syscall.c` |
| `->` (arrow) | Accesses field through a pointer | `p->trace_mask` |
| Bitmask | Number where individual bits are flags | `(1 << num) & mask` |
| Dispatch table | Array mapping syscall numbers to functions | `syscall.c` (`syscalls[]`) |
| `struct proc` | Per-process data (like a Python object) | `proc.h` |
| Fork inheritance | Child copies parent's state | `proc.c` (`kfork()`) |
| `exec()` persistence | `struct proc` survives exec | `trace.c` design |
| `extern` | "This function exists in another file" | `syscall.c` declarations |

---

> **You've got this!** 💪 Each step builds on the previous one. If something
> breaks, check the most recent step first. The most common issue is a typo
> or a missing comma/semicolon.
