# Appendix-2: Linux

## 1. Historical context and evolution
Linux was created by Linus Torvalds in 1991 as a free UNIX-like kernel. Initially a hobby project, it quickly gained contributions worldwide. Early distributions (MCC, MCC Interim) led to formal distros like Slackware (1993) and Debian (1993). The GNU project’s tools combined with the Linux kernel formed the GNU/Linux OS. Over decades, distributions (Red Hat, SUSE, Ubuntu) standardized package management, while the kernel evolved through monolithic design, introduced loadable modules, and extended hardware support.

## 2. Operating system architecture
Linux follows a **monolithic** kernel design with loadable kernel modules. The kernel includes device drivers, file systems, networking stack, and core services in kernel space. User-space interacts via system calls through the **GNU C Library (glibc)**. The architecture is layered: hardware at bottom, kernel, user-space libraries, and user applications/shells.

## 3. Kernel and core system behavior
The Linux kernel manages interrupts, system calls, scheduling, and memory. Interrupts invoke handlers in kernel mode; system calls transition via `syscall` instruction. Core components include the **Process Scheduler**, **Virtual File System (VFS)**, **Network Stack**, **Memory Manager**, and **Device Drivers**. Kernel modules can be loaded/unloaded at runtime via `insmod`/`rmmod`.

## 4. Process management and scheduling
Linux uses **preemptive multitasking**. The current default scheduler is the **Completely Fair Scheduler (CFS)**, which models tasks as virtual runtimes, distributing CPU time fairly. Each task has scheduling policies (SCHED_OTHER, SCHED_FIFO, SCHED_RR). `nice` values adjust priority for CFS tasks. On SMP systems, load balancing distributes tasks across CPUs. Kernel threads (`kthreads`) handle background work, while user threads are managed via the NPTL (Native POSIX Thread Library).

## 5. Memory management and virtual memory
Linux VM uses demand paging with a **buddy allocator** for physical memory. Slab allocators (`kmalloc/slab`) manage kernel allocations. Each process has a page table managed by the hardware MMU. **OOM Killer** terminates processes when memory is exhausted. Transparent Huge Pages (THP) provide large pages for performance. `mmap` supports file mapping and anonymous memory, and swap space allows paging to disk.

## 6. File systems and storage architecture
Linux supports many file systems: ext4, XFS, Btrfs, F2FS, VFAT, NTFS via NTFS-3G. Each FS implements the VFS interface. Ext4 is default in many distros, providing journaling, extents, delayed allocation. Btrfs offers copy-on-write snapshots, compression, and RAID. The block layer manages request queueing and elevator schedulers (CFQ, deadline, noop). Udev and kernel hotplug manage device nodes in `/dev`.

## 7. User interface and user-level services
Linux user-space is diverse. Common shells (bash, zsh) provide CLI. Desktop environments (GNOME, KDE) run atop X11 or Wayland. System services are managed by **systemd** (PID1), which handles service units, socket activation, and journal logging. Traditional init scripts (SysV) are still supported. Package managers (APT, YUM, Pacman) handle software installs.

## 8. Security model and mechanisms
Linux uses UNIX permissions (user/group/other) and optional ACLs. Additional MAC frameworks include **SELinux**, **AppArmor**, and **smack**. `sudo` enables privilege escalation. Kernel capabilities break root privileges into granular rights. Namespaces and cgroups isolate resources for containers (Docker, LXC). Seccomp filters restrict syscalls. Security modules enforce policies at hook points.

## 9. Performance and optimization strategies
Tools: `top`, `htop`, `vmstat`, `iostat`, `perf`, `ftrace`, `systemtap`. Kernel tunables via `/proc/sys` and `sysctl`. CPU frequency scaling via `cpufreq`. I/O scheduling can be tuned per block device. Huge pages and NUMA policies (`numactl`) optimize memory. `tuned` or `tuned-adm` profiles for latency or throughput. TRIM support for SSDs.

## 10. Compatibility, portability, and virtualization
Linux runs on x86, ARM, PowerPC, RISC-V, and more. Cross-compilation and kernel config options support diverse architectures. Virtualization via **KVM**, **Xen**, **LXC**, and **Docker**. Kernel modules isolate hardware. Filesystems can be shared via NFS, SMB (Samba), or SSHFS. Interoperability via Wine for Windows applications.

## 11. Common real-world problems
- **Kernel panics**: due to bad modules or hardware.  
- **OOM issues**: configure swap or tune `oom_score_adj`.  
- **Filesystem corruption**: repair with `fsck`.  
- **Package conflicts**: resolve via package manager and dependencies.  
- **Networking issues**: diagnose with `ip`, `ss`, `ethtool`.  
- **Driver/modules**: use `modprobe` and check `dmesg`.  
- **SELinux/AppArmor denials**: audit via `ausearch` or `journalctl`.  
- **Resource hogs**: identify with `ps`, `top`, `systemd-analyze`.  

## 12. Practical examples
```bash
# List top CPU processes
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head

# Check memory and swap
free -h

# Display mounted filesystems
mount | column -t

# Network diagnostics
ip addr show
ss -tulpn

# Manage services (systemd)
systemctl status sshd
systemctl start nginx

# Create an ext4 filesystem on /dev/sdb1
mkfs.ext4 /dev/sdb1

# Memory-map a file in C
#include <sys/mman.h>
#include <fcntl.h>
int fd = open("file.dat", O_RDONLY);
char *p = mmap(NULL, length, PROT_READ, MAP_PRIVATE, fd, 0);

# Set SELinux to permissive
setenforce 0

# Start a KVM VM
virt-install --name vm1 --memory 2048 --disk size=10 --cdrom /iso/ubuntu.iso
```