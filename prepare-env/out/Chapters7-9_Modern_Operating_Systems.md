# Memory Management (Chapter 7)

Operating systems must manage the allocation and protection of physical memory among multiple processes. A central goal is to give each process the illusion of a large, private memory while using real RAM efficiently. **Virtual memory** techniques create this illusion by mapping *virtual (logical) addresses* used by a process into *physical addresses* in RAM. Address translation is typically handled by hardware (an MMU) using tables maintained by the OS. For example, if a process issues a reference to virtual address `V`, the OS/CPU translates it to a physical address `P` by locating the page table entry or segment descriptor for `V`, then combining it with the offset within that page or segment. This allows each process to behave as if it has a large contiguous memory space.

Memory management techniques include both **contiguous** and **noncontiguous** allocation. In contiguous allocation, each process occupies one continuous block of RAM (with a base and limit register). Simple schemes like fixed or variable partitions can be used, but fragmentation—wasted space that cannot be used for new processes—becomes an issue. **Internal fragmentation** occurs when fixed-sized blocks leave unused space inside an allocated region, while **external fragmentation** occurs when free memory is split into small pieces between allocated blocks. Compaction (shifting processes) can alleviate external fragmentation but is expensive.

To avoid fragmentation and allow more flexible allocation, modern systems use **paging** or **segmentation** (or both). *Paging* divides logical address space into fixed-size pages and physical memory into equal-size frames. A process’s page table maps each virtual page to a physical frame, meaning pages need not be contiguous in RAM. Paging thus **eliminates external fragmentation** (since any free frame can be used) at the cost of potential *internal* fragmentation. Paging combined with virtual memory allows programs to exceed physical RAM through on-demand loading of pages from disk. When a page fault occurs (a referenced page is not in memory), the OS finds a free frame (or evicts an existing page), reads the required page from disk into that frame, updates the page table, and resumes the process.

*Segmentation* divides memory into variable-sized segments corresponding to logical units (code, data, stack). Each address has a segment number and an offset. The OS keeps a segment table with base/limit for each segment. Segmentation allows protection and sharing at segment granularity but can lead to external fragmentation. Modern systems often combine segmentation with paging, by breaking segments into pages.

## Paging and Address Translation

In a paged system, the **page table** translates virtual addresses to physical ones. For a virtual address `V`, the CPU (with MMU) splits it into a page number `p` and offset `d`. The page table entry for page `p` gives a frame number `f`. The physical address is then:

```c
#define PAGE_SIZE 4096
int page_table[NUM_PAGES]; // maps page -> frame or invalid

int translate(int V) {
    int p = V / PAGE_SIZE;
    int d = V % PAGE_SIZE;
    int f = page_table[p];
    if (f < 0) {
        // page fault: page not in memory
        handle_page_fault(p);
        f = page_table[p];
    }
    return f * PAGE_SIZE + d;
}
```

When *demand paging* is used, pages are loaded only when referenced. A **page fault** traps to the OS, which:

1. Checks that the address is valid.
2. Locates a free frame or chooses a victim page to evict.
3. If the victim is dirty, writes it back to disk.
4. Reads the requested page into the free frame.
5. Updates the page table and marks the page present.
6. Restarts the faulting instruction.

This routine is transparent to the process.

## Page Replacement Algorithms

When physical memory is full, the OS must evict a page to make room. Common strategies:

- **FIFO (First-In-First-Out):** Remove the oldest loaded page.
- **LRU (Least Recently Used):** Evict the page unused for the longest time.
- **Optimal (Belady’s):** Replace the page not needed for the longest future time (requires future knowledge).
- **Clock (Second Chance):** Use a circular list with reference bits, giving pages a “second chance.”

Example LRU selection:

```c
int select_victim_LRU(int n_frames, int last_used[]) {
    int victim = 0;
    for(int i = 1; i < n_frames; i++){
        if(last_used[i] < last_used[victim]) {
            victim = i;
        }
    }
    return victim;
}
```

Advanced systems approximate LRU with aging counters or working-set information. The OS predicts which page is least likely to be used soon to optimize performance.

## Thrashing and Working Set

**Thrashing** occurs when processes spend more time swapping pages than executing instructions due to insufficient frames for combined working sets. The **working-set model** gives each process enough frames to hold its recent pages; systems may use admission control to avoid thrashing.

## Dynamic Memory Allocation

OS and application memory allocation use algorithms like **first-fit**, **best-fit**, and **buddy system**. Example first-fit allocator:

```c
struct Block {
    int size;
    struct Block *next;
} *free_list;

void *allocate(int size) {
    Block **prev = &free_list;
    for(Block *b = free_list; b != NULL; b = b->next) {
        if (b->size >= size) {
            *prev = b->next;
            return (void*)(b+1);
        }
        prev = &b->next;
    }
    return NULL; // out of memory
}
```

# I/O Systems (Chapter 8)

Operating systems provide a layered **I/O system** managing devices (disks, networks, printers). It includes user-level interfaces, device-independent software (buffering, spooling), and device drivers.

### Communication Modes

- **Polling:** CPU repeatedly checks device status—simple but potentially wasteful.
- **Interrupt-driven:** Device raises interrupt on completion; CPU handles asynchronously.
- **DMA (Direct Memory Access):** Device transfers data directly to memory; CPU sets up transfer and continues work.

Example ring buffer for interrupt-driven I/O:

```c
#define BUF_SIZE 256
char buffer[BUF_SIZE];
int head = 0, tail = 0;

void device_receive_char(char c) {
    buffer[tail] = c;
    tail = (tail + 1) % BUF_SIZE;
}

char read_char() {
    while (head == tail) {
        // wait or yield
    }
    char c = buffer[head];
    head = (head + 1) % BUF_SIZE;
    return c;
}
```

### Buffering and Spooling

- **Single buffering:** One buffer—may cause waits.
- **Double buffering:** Two buffers—overlaps I/O and processing.
- **Circular buffers:** For continuous streams.
- **Spooling:** Jobs queued on disk (e.g., printer spool).

### I/O Scheduling

For block devices, scheduling algorithms improve throughput:

- **FCFS:** Serve requests in arrival order.
- **SSTF:** Shortest seek time first.
- **SCAN/Elevator:** Sweep disk arm back and forth.
- **C-SCAN:** Circular SCAN with jump back.

# File Systems (Chapter 9)

File systems organize data on storage devices, abstracting raw blocks into files and directories.

## Files and Directories

- **Files:** Sequence of bytes with metadata (size, timestamps, permissions).
- **Directories:** Special files mapping names to file identifiers (inodes).

## File System Structure

Layers:

1. **Logical FS:** User syscalls (`open`, `read`, `write`).
2. **Organization Module:** Block allocation, free-space management.
3. **Device Driver:** Physical I/O.

Key on-disk structures:

- **Superblock:** Global metadata.
- **Inodes:** Metadata and pointers to data blocks.
- **Free-space:** Bitmap or free list.
- **Directory blocks:** Name-to-inode mappings.

Example inode read:

```c
struct inode get_inode(int inum) {
    lseek(fs_fd, SUPERBLOCK_SIZE + (inum-1)*sizeof(struct inode), SEEK_SET);
    struct inode node;
    read(fs_fd, &node, sizeof(node));
    return node;
}
```

## Allocation Methods

- **Contiguous:** Start block + length—fast but fragmented.
- **Linked:** Blocks linked—no fragmentation, slow random access.
- **Indexed:** Index block lists data blocks—balanced.

## Free Space Management

- **Bitmap:** Compact but scan cost.
- **Free list:** Quick allocation, overhead per block.

## Directory Implementation

Linear list or indexed structures (B-trees) optimize lookups.

## Block Access Example

```c
int get_block_num(struct inode *node, int b) {
    if (b < DIRECT_COUNT) {
        return node->blocks[b];
    } else if (b < DIRECT_COUNT + BLOCKS_PER_INDIRECT) {
        int idx = b - DIRECT_COUNT;
        int iblock = node->indirect;
        read_block(iblock, buffer);
        return ((int*)buffer)[idx];
    } else {
        // handle double indirect...
    }
}
```

## Protection and Attributes

Permissions enforced via metadata (mode bits, ACLs), checked on access.

**Summary:** Chapters 7–9 cover virtual memory, page replacement, thrashing, I/O systems (polling, interrupts, DMA, buffering, scheduling), and file systems (structures, allocation, free space, directories). Each area balances efficiency, flexibility, and reliability.
