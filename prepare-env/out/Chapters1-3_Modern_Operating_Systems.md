# Chapter 1: Introduction to Operating Systems

An **operating system (OS)** is system software that manages computer hardware and software resources and provides common services for computer programs. It hides the complexity of hardware and presents users with a friendly “extended machine” abstraction. In essence, the OS is a **resource manager**: it allocates CPU time, memory space, and I/O devices among competing tasks.

In practical terms, the OS provides common services (file handling, networking interfaces, etc.) so that applications need not manage hardware directly. For hardware functions like input/output and memory access, the OS acts as an intermediary between programs and devices. By doing so, it presents a simpler interface to programmers (e.g. system calls to read a file or send data to the display) rather than requiring them to manipulate hardware registers directly. In modern systems, this also enables **virtualization**: the OS can make one physical CPU appear as many (time-sharing among processes) and even present the illusion of a large continuous memory by swapping data to disk when RAM is full. These techniques make it seem to applications as if there are nearly limitless CPUs and memory, even when physical resources are limited.

Early computers had no operating system: each program had to include its own hardware drivers and was run to completion before the next began. As hardware and workload complexity grew, primitive “monitor” programs evolved. These resident monitors could read jobs (from punched cards or tape), initiate execution, and handle basic I/O – functions that today’s OS kernels still perform. Over time, more sophisticated multitasking and multiuser OSs emerged (e.g. UNIX, Windows, Linux) along with special-purpose systems (embedded, real-time, mobile) to meet diverse needs. Today, mobile operating systems like Android and iOS dominate phones, while Linux and Windows run servers and desktops.

The internal **structure** of an OS can vary. In a **monolithic kernel** design, the entire OS (device drivers, file system, process scheduler, etc.) is one large program running in privileged mode. This offers fast communication between components but can be complex to maintain. Alternatively, **microkernels** strip out non-essential services: only basic functions (like low-level scheduling and communication) run in kernel mode, while other services run in user space. This modular approach can improve security and reliability (a failure in one service won’t crash the entire OS) at the cost of more overhead due to inter-component messaging. Many modern OSs use **hybrid** or **modular** kernels that mix these approaches. Another model is a **layered** architecture, where the OS is broken into layers (e.g. hardware at layer 0 up to user interface at top). Each layer provides services to the one above it, which simplifies design and debugging.

**Key services and goals** of an OS include: managing processes and threads, scheduling the CPU, controlling memory allocation, handling I/O and file storage, and providing networking and security. The OS abstracts hardware so that applications can be portable across devices. It also enforces isolation and protection so that one misbehaving program cannot corrupt others or the kernel. High-level aims of scheduling and resource management are to keep the CPU and devices busy while minimizing response time and preventing any single program from monopolizing resources.

# Chapter 2: Processes and Threads

A **process** is a running instance of a program. In computing terms, “a process is the instance of a computer program that is being executed.” While a program is static code on disk, a process is that code loaded into memory with an associated execution context. This context includes the program counter (next instruction), CPU registers, memory space, open file handles, and other resources. Multiple processes may run concurrently on a multitasking OS; the kernel switches among them so that each appears to run simultaneously on one or more CPUs.

Processes cycle through distinct **states** as they execute. After creation, a process enters the *ready* state (loaded into RAM and waiting for CPU time). When the scheduler picks a ready process, it moves to the *running* state and the CPU executes its instructions. If the process must wait (for I/O, user input, etc.), it goes into the *blocked* (waiting) state, relinquishing the CPU until the event occurs. Once the wait is over, it returns to ready. Eventually when the program finishes or is terminated, the process enters the *terminated* state, and the OS reclaims its resources. The OS keeps track of a process’s state and context in a **process control block (PCB)** or similar data structure.

In Unix-like systems, processes can create new processes. The standard system call is `fork()`, which duplicates the current process. After `fork()`, two processes exist (parent and child) running the same code. Both return from `fork()`, but with different return values: the child sees 0, the parent gets the child’s PID. Each then continues execution independently. Commonly, the child will immediately replace its memory image by calling `exec` to run a new program.

```c
pid_t pid = fork();
if (pid < 0) {
    // fork failed 
} else if (pid == 0) {
    // Child process
    execl("/bin/program", "program", (char*)NULL);
    // (never returns unless exec fails)
} else {
    // Parent process: wait for child to finish
    int status;
    wait(&status);
}
```

Processes can communicate and synchronize in many ways. The OS provides **interprocess communication (IPC)** primitives. For example, a simple unidirectional pipe can connect two related processes. The C library call `pipe(int fd[2])` creates a pipe: `fd[1]` becomes the write end, and `fd[0]` the read end. Data written by one process to `fd[1]` can be read by the other from `fd[0]`. Many other IPC mechanisms exist (named pipes, message queues, shared memory, sockets, signals), but the core idea is the OS coordinates data exchange and mutual exclusion between processes.

Modern OS also support **threads**, which are lightweight units of execution within a process. A thread is “the smallest sequence of programmed instructions that can be managed independently by a scheduler.” Unlike separate processes, threads in the same process share the same code, data, and address space; they differ in having individual program counters, stacks, and registers. Because threads share memory, creating and switching threads has less overhead than creating whole processes. Multithreading allows a program to perform concurrent tasks (for example, a web browser handling multiple connections). The OS kernel schedules threads in much the same way as processes: any runnable thread can be chosen to execute on a CPU, with context switches saving and restoring the thread’s state.

```c
#include <pthread.h>
void *thread_func(void *arg) {
    // Thread work goes here
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    // ... main thread can do other work here ...
    pthread_join(tid, NULL);  // wait for thread to finish
    return 0;
}
```

# Chapter 3: Memory Management and Address Spaces

Modern computers have a **memory hierarchy**. At the top are very fast CPU registers and caches; all current CPU requests first go to the L1 cache (often split into an instruction cache and a data cache). Larger L2 caches back up L1 but are somewhat slower. Beyond cache is the main memory (RAM), which is much larger (often gigabytes) but also slower. Finally, secondary storage (magnetic disks or flash SSDs) provides vast, non-volatile storage at the lowest speed. The OS manages each level: it loads programs from disk into RAM on demand, and the hardware/OS work together to keep frequently used data in cache.

An OS must manage this memory to ensure each process has its own **address space** and cannot corrupt others. A multiprogramming OS maintains memory protection so that a process can only access addresses assigned to it. Early systems used *segmentation*, where memory addresses were split into segments with base/limit registers. Access outside a segment caused a trap. More commonly today, *paging* is used: the virtual address space of each process is divided into fixed-size pages (e.g. 4KB), and physical memory is divided into frames of the same size. The OS maintains a **page table** mapping each virtual page to a physical frame.

```c
#define PAGE_SIZE 4096
typedef struct { bool present; int frame; } PTE;
PTE page_table[MAX_PAGES];

unsigned int translate(unsigned int vaddr) {
    unsigned int vpn = vaddr / PAGE_SIZE;
    unsigned int offset = vaddr % PAGE_SIZE;
    PTE entry = page_table[vpn];
    if (!entry.present) {
        // Trigger a page fault and load the page
        handle_page_fault(vpn);
    }
    return entry.frame * PAGE_SIZE + offset;
}
```

To speed up address translation, hardware typically includes a **translation lookaside buffer (TLB)** – a small associative cache of recent page-table entries. On every memory access, the CPU first checks the TLB. If the virtual page is in the TLB (a *TLB hit*), the physical address is obtained instantly from the cached entry. If not (a *TLB miss*), the CPU must walk the page table in memory to get the frame number, which is slower. After the translation, the new mapping is loaded into the TLB.

Beyond basic paging, operating systems may use multi-level page tables, inverted tables, or other schemes to handle very large address spaces efficiently. But the core idea is that each process has its own virtual address space, and the OS plus hardware (MMU) enforce memory protection. In this way, an address in one process’s space cannot accidentally or maliciously affect another. Virtual memory also allows the system to run processes that collectively need more memory than physically available: less-used pages can be temporarily stored on disk and brought back in on demand, giving programs the **illusion** of a large contiguous memory.

**Example Code – Pipes (IPC)**

```c
int fd[2];
if (pipe(fd) == 0) {
    // fd[0] is read end, fd[1] is write end
    write(fd[1], "data", 4);
    char buf[5] = {0};
    read(fd[0], buf, 4);
    printf("Received: %s
", buf);
}
```

**Example Code – Page Table Translation (Conceptual)**

```c
#define PAGE_SIZE 4096
unsigned int translate(unsigned int vaddr) {
    unsigned int vpn = vaddr / PAGE_SIZE;
    unsigned int offset = vaddr % PAGE_SIZE;
    PTE entry = page_table[vpn];
    if (!entry.present) {
        handle_page_fault(vpn);  // load page into memory
        entry = page_table[vpn];
    }
    return entry.frame * PAGE_SIZE + offset;
}
```
