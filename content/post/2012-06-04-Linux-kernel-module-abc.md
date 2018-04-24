---
title: "Linux Kernel Module ABC"
date: 2012-06-04
categories:
  - Linux
tags:
  - Kernel Module
  - Syscall
toc: true
draft: false
---

# Install a kernel module in Linux

1. If new feature needed: kernel module daemon kmod will exec the `modprobe`. `modprobe` is passed in the form of a name (softdog or ppp) or generic string identifier (char-major-10-30). If `modprobe` is handed a string identifier, it will look for the string in the file `/etc/modprobe.conf`.
2. Then the modprobe will look through the file `/lib/modules/versions/modules.dep` to find out whether this new module depends on other modules.
3. Then the modeprobe will use the insmode to load the dependency modules into the kernel. The difference using the two commands to load the module is that `modprobe` takes the module name, while `insmode` should take the whole path of the modules and its dependencies. For example to initiate the msdos module:

    ```bash
    # Use the insmode
    insmod /lib/modules/2.6.11/kernel/fs/fat/fat.ko
    insmod /lib/modules/2.6.11/kernel/fs/msdos/msdos.ko
    
    # Use the modprobe
    modprobe msdos
    ```
4. Command to list all the modules that currently runs: `lsmod`.

# System call interception in loadable kernel module

Modern CPUs can run in two modes: kernel mode and user mode. When a CPU runs in kernel mode, an extended set of instructions is allowed, as is free access to anywhere in memory and device registers. Interrupt drivers and operating system services run in kernel mode. In contrast, when the CPU runs in user mode, only a restricted set of instructions is allowed, and the CPU has a restricted view of the memory and devices. Library functions and user programs run in user mode. Kernel and user mode together form the basis for security and reliability in modern operating systems.

System calls are "gates" into the kernel implemented with software interrupts. Program spend most of their times in the user mode. Software interrupts are interrupts produced by a program and processed in kernel mode by the operating system. The operating system maintains a "system call table" that has pointers to the functions that implement the system calls inside the kernel. From the program's point of view, this list of system calls provides a well-defined interface to the operating system services.

Loadable modules are piece of code that can be loaded and unloaded into the kernel without the kernel reboot. For example, the USB driver. The alternative to loadable modules is a monolithic kernel where new functionality is added directly into the kernel code. Monolithic kernels have the disadvantage of needing to be rebuilt and reinstalled every time new functionality is added.

# Example: Intercepting system salls via a loadable module

Assume that we want to intercept the exit system call and print a message on the console when any process invokes it. In order to do this, we have to write our own fake exit system call, then make the kernel call our fake exit function instead of the original exit call. At the end of our fake exit call, we can invoke the original exit call. In order to do this, we must manipulate the system call table array (`sys_call_table`). Take a look at `/usr/src/linux/arch/i386/kernel/entry.S` (assuming you are on an i386 architecture). This file contains a list of all the system calls implemented within the kernel and their position within the `sys_call_table` array.

```c
/* Listing 1. example 2.c */

#include <linux/kernel.h>
#include <linux/module.h>
#include <sys/syscall.h>

extern void *sys_call_table[];

asmlinkage int (*original_sys_exit)(int);

asmlinkage int our_fake_exit_function(int error_code)
{
    /* print message on console every time we are called */
    printk("HEY! sys_exit called with error_code=%d\n",error_code);

    /* call the original sys_exit */
    return original_sys_exit(error_code);
}

/* this function is called when the module is loaded (initialization) */
int init_module()
{
    /* store reference to the original sys_exit */
    original_sys_exit=sys_call_table[__NR_exit];

    /* manipulate sys_call_table to call our fake exit function
     * instead of sys_exit */
    sys_call_table[__NR_exit]=our_fake_exit_function;
}


/* this function is called when the module is unloaded */
void cleanup_module()
{
    /* make __NR_exit point to the original sys_exit when module unloaded */
    sys_call_table[__NR_exit]=original_sys_exit;
}
```

# Intercepting `sys_execve` to protect against "root-kits"

We can intercept the following system call to prevent the system from whimsically executing the Trojan horses program. We do this by comparing to the program hash databases. The database includes all the legal program that can be called by the system. While a mismatch of the hash occurred, the operation will be logged in the system.

The system call include `sys_execv`, `sys_delete_module`, `sys_create_module`, `sys_open`, `sys_unlink`, etc.. We know that it is impossible to assure the security of the program because we can intercept all the suspected system calls, and also the attacker might modify the kernel symbols in `/dev/kmem` or use raw device access to the hard disk, and bypass `open` to write to the hash database file.

Therefore, the flexibility and power of loadable kernel modules can be misused by malicious users who may have gained access to the system. For example, the `sys_execve` function call can be intercepted to invoke a trojan program instead of the one intended, and system calls such as `read` and `write` can be intercepted to perform keystroke logging.

# Conclusion

The study up until now reviles two methods to log the system calls. The first one is create a "faked module" to redirect the system call to our module the do something inside the module, and then call the originally intended system call at the end of our loadable kernel module. Another method is to create a program hash table. With the hash values of the program, when the system call doesn't lie in the program database, the operation will be logged.