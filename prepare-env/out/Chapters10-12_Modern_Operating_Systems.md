# Chapter 10: Virtualization

Virtualization allows multiple operating systems or instances to run concurrently on the same physical hardware by abstracting machine resources. There are two main approaches:

- **Full Virtualization**: A hypervisor (Virtual Machine Monitor) completely simulates underlying hardware. Guest OSes run unmodified. Examples: VMware ESXi, KVM, Xen (HVM mode).
- **Paravirtualization**: The guest OS is modified to make hypercalls to the hypervisor rather than executing privileged CPU instructions. Example: Xen (PV mode).
- **Container-based Virtualization**: OS-level virtualization where containers share the host kernel but have isolated user spaces. Examples: Docker, LXC.

## Hypervisor Types

1. **Type 1 (Bare Metal)**: Runs directly on hardware. Low overhead, high performance.  
2. **Type 2 (Hosted)**: Runs on a host OS as an application. Easier to install, higher latency.

## Hardware Support

Modern CPUs include virtualization extensions:
- **Intel VT-x**: VMX modes with VMCS for guest state and VM exits.
- **AMD-V**: SVM modes with VMCB for control blocks.

The hypervisor uses these features to trap privileged operations and emulate or forward them.

## Memory and I/O Virtualization

- **Nested Page Tables (EPT/NPT)**: Hardware assists guest physical to host physical mapping, reducing VM exits on memory access.
- **I/O MMU (VT-d/AMD-Vi)**: Direct device assignment via DMA remapping.

## Example: Creating a Linux Container with `clone()`

```c
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static int child_fn(void *arg) {
    printf("In container: PID = %d
", getpid());
    execl("/bin/sh", "sh", NULL);
    return 0;
}

int main() {
    const int STACK_SIZE = 1024 * 1024;
    void *stack = malloc(STACK_SIZE);
    if (!stack) { perror("malloc"); exit(1); }

    pid_t pid = clone(child_fn, (char*)stack + STACK_SIZE,
                      CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD,
                      NULL);
    if (pid < 0) {
        perror("clone");
        exit(1);
    }
    printf("Launched container with PID %d
", pid);
    waitpid(pid, NULL, 0);
    free(stack);
    return 0;
}
```

This code uses the `clone()` syscall to create a new process in its own UTS (hostname), PID, and mount namespaces.

---

# Chapter 11: Security

Security in operating systems encompasses mechanisms to protect confidentiality, integrity, and availability.

## Authentication and Access Control

- **User IDs (UID/GID)** and permission bits (rwx) control file and resource access.
- **setuid/setgid** binaries run with the file owner’s privileges.  
- **Access Control Lists (ACLs)** extend Unix permissions for finer-grained control.

### Example: Changing File Ownership and Permissions

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    if (chown("/tmp/secret.txt", 1001, 1001) < 0) perror("chown");
    if (chmod("/tmp/secret.txt", S_IRUSR | S_IWUSR) < 0) perror("chmod");
    return 0;
}
```

## Process Isolation and Sandboxing

- **Namespaces** (as used by containers) isolate processes.
- **Capabilities** break root’s power into distinct privileges (network, mount, etc.).

## Encryption and Secure Communication

OSes often provide kernel modules or system calls for cryptographic operations. User-space libraries (e.g., OpenSSL) interact via system calls like `read()/write()` on `/dev/crypto`.

### Example: Simple AES Encryption with OpenSSL (User Space)

```c
#include <openssl/aes.h>
#include <string.h>
#include <stdio.h>

int main() {
    AES_KEY enc_key;
    unsigned char key[16] = "0123456789abcdef";
    unsigned char input[16] = "SecretMessage123";
    unsigned char output[16];

    AES_set_encrypt_key(key, 128, &enc_key);
    AES_encrypt(input, output, &enc_key);
    printf("Encrypted block: ");
    for(int i = 0; i < 16; i++) printf("%02x", output[i]);
    printf("\n");
    return 0;
}
```

## Security Policies and Enforcement

- **Mandatory Access Control (MAC)**: e.g. SELinux, AppArmor. Labels on subjects/objects enforce rules beyond DAC.  
- **Audit and Logging**: Kernel logs (`dmesg`), syslog, auditd.

---

# Chapter 12: Case Studies

## Linux Kernel

A monolithic, modular kernel with loadable kernel modules (LKMs).

### Example: Hello World Kernel Module

```c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, kernel!\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("User");
MODULE_DESCRIPTION("A simple Hello World module");
```

- **Process Scheduler**: Completely Fair Scheduler (CFS) using red-black tree, prioritizes fairness.
- **Memory Management**: Buddy allocator, slab allocators, mmaps, transparent huge pages.
- **I/O and File Systems**: VFS layer abstracts ext4, XFS, Btrfs.

## Windows NT

A hybrid kernel with microkernel influences. Key components:
- **Executive**: I/O manager, memory manager, security reference monitor.
- **Kernel**: thread scheduler, interrupt dispatch.
- **Device Drivers** in kernel mode.

### Example: Win32 File I/O

```c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE h = CreateFile("C:\temp\data.txt", GENERIC_WRITE,
                          0, NULL, CREATE_ALWAYS,
                          FILE_ATTRIBUTE_NORMAL, NULL);
    if (h == INVALID_HANDLE_VALUE) {
        printf("Error opening file\n");
        return 1;
    }
    const char *msg = "Hello, Windows!\n";
    DWORD written;
    WriteFile(h, msg, strlen(msg), &written, NULL);
    CloseHandle(h);
    return 0;
}
```

## Android OS

Based on Linux kernel, adds:
- **Binder IPC**: lightweight mechanism for inter-process communication.
- **Zygote process**: preloads classes and forks for each app to reduce startup time.

## Summary

Chapters 10–12 cover advanced topics—virtualization mechanisms and hardware support; security models, policies, and sample code; and case studies of Linux, Windows, and Android illustrating real-world OS architectures.  

