1. Debugging support in the kernel:
See LDD3 p73 for more details. Some of the important configurations are listed below: 
• CONFIG_DEBUG_SLAB: each byte of allocated memory is set to 0xa5 before being handled to the caller, and then set to 0x6b when it is freed. (These poison values are useful when debugging)
• CONFIG_DEBUG_SPINLOCK_SLEEP: This option enables a check for attempts to sleep while holding a spinlock. In fact, it complains if you call a function that could potentially sleep, even if the call in question would not sleep..
• CONFIG_DEBUG_STACK_USAGE, CONFIG_DEBUG_STACKOVERFLOW: A sure sign of a stack overflow is an oops listing without any sort of reasonable back trace.
• CONFIG_IKCONFIG, CONFIG_IKCONFIG_PROC: These options cause the full kernel configuration state to be built into the kernel and to be made available via /proc. (In "general setup" menu)
• CONFIG_DEBUG_DRIVER: Turn on debugging information in the driver core.
• CONFIG_PROFILING: (found under "Profiling support") 

2. Debugging by printing:
• printk() doesn't flush until a trailing newline is provided. 
• A printk() with no specified priority defaults to DEFAULT_MESSAGE_LOGLEVEL specified in kernel/printk.c. (In 2.6 kernel, it is KERN_WARNING). /proc/kmsg.
• The variable console_loglevel is initialized to DEFAULT_CONSOLE_LOGLEVEL and can be modified through the sys_syslog syscall.
• It's also possible to read and modify the console loglevel using the text file /proc/sys/kernel/printk. This file contains 4 integer values: "current_loglevel	default_loglevel(for msgs that lack an explicit loglevel)	minimum_allowed_loglevel	boot-time_default_loglevel". Writing a single value to this file changes the current loglevel to that value. (Thus you can cause all kernel msgs to appear at the console simply by: "echo 8 > /proc/sys/kernel/printk".)
• printk() writes messages into a circular buffer that is __LOG_BUF_LEN bytes long: a value from 4KB to 1MB chosen when configuring the kernel. The function then wakes any process that is waiting for messages, that is, any process that is sleeping in the syslog syscall or that is reading /proc/kmsg.
• printk() can be invoked from anywhere, even from an interrupt handler, with no limit on how much data can be printed. The only disadvantage is the possibility of losing some data.

• Rate limiting:
	int printk_ratelimit(void);
If this function returns a nonzero value, go ahead and print your message; otherwise skip it. Typical usage:
	if (printk_ratelimit())
		printk(KERN_NOTICE "The printer is still on fire\n");
printk_ratelimit() works by tracking how many messages are sent to the console. When the level of output exceeds a threashold, printk_ratelimit() starts returning 0 and causing messages to be dropped.
* The behavior of printk_ratelimit() can be customized by modifying i) /proc/sys/kernel/ptintk_ratelimit (the number of seconds to wait before re-enabling messages) and ii) /proc/sys/kernel/printk_ratelimit_burst (the number of messages accepted before rate-limiting)

• Printing device numbers:
When printing a message from a driver, you will want to print the device number associated with the hardware of interest. The kernel provides a couple of utility macros (defined in <linux/kdev_t.h>) for this purpose:
	int print_dev_t(char *buffer, dev_t dev);
	char *format_dev_t(char *buffer, dev_t dev);
Both macros encode the device number into the given buffer. The only difference is that print_dev_t() returns the number of characters printed; while format_dev_t() returns buffer.

3. Debugging by quering:
procfs, sysfs, ioctl driver method.
• Using procfs:
Each file under /proc is tied to a kernel function that generates the file's contents "on the fly" when the file is read.
Kernel code is in <linux/proc_fs.h> and fs/proc/generic.c. See "procfs_guide.pdf" and "lkmpg.pdf" (both are in google drive) for more details." I also recommend to read the kernel source code. 
Some things need noticing:
i)   procfs is not recommended to use comparing with sysfs, but it is still widely used.
ii)  If you just a need a read-only proc file, just use:
	struct proc_dir_entry *create_proc_read_entry(const char *name, mode_t mode, struct proc_dir_entry *parent, read_proc_t *read_proc, void *data);
So no need to implement write_proc_t function.
iii) Providing file_operations is lower than providing read_proc_t and write_proc_t functions. And after 3.1 kernel, read_proc_t and write_proc_t are completely gone. Reading through procfs kernel code you'll find that, if file_operations are provided, then there's no need to provide read_proc_t/write_proc_t, even if you provide them, they won't be used.
iv)  procfs is buggy when the file is larger than PAGE_SIZE. (http://comments.gmane.org/gmane.linux.kernel/874924) And read_proc_t function has a tricky interface about how to use it correctly; it has 3 modes. I recommend reading the source code in fs/proc/generic.c, and this article: http://www.sudu.cn/info/html/edu/20070102/292890.html; in order to figure out the right usage of procfs.
v)   So procfs is still easy interface to use for files smaller than PAGE_SIZE. For large files, we use seq_file interface instead of read_proc_t and write_proc_t functions. (http://www.youback.net/kernel/proc%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E4%B9%8Bseq_file%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.html; http://taz.newffr.com/TAZ/Coding/c/kernel/seq_file_howto.txt)

• Using seq_file (in profs) dealing with large file:
See the two links above, and lkmpg.pdf for more details. Also check the seq.c example code. The most important thing that needs to be clarified is that: the calling sequence and the relationship of the four functions (start, show, next, stop), which is explained here: http://www.youback.net/kernel/proc%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E4%B9%8Bseq_file%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.html

• Using ioctl method:
ioctl is a syscall that acts on a file descriptor. It receives a number that identifies a command to be performed and (optionally) another argument, usually a pointer. 
As an alternative to using the /proc filesystem, you can implement a few ioctl commands tailored for debugging. These commands can copy relevant data structures from the driver to user space where you can examine them.
ioctl runs faster than reading /proc.

4. Debugging by watching:
• strace() syscall: strace -p [pid] could attach to a running process and trace.
• Debugging kernel oops:
A kernel oops is just a segfault in kernel space (you could understand in this way). See LKD to understand in what situation the system couldn't recover from a oops.
A link: http://www.linuxforu.com/2011/01/understanding-a-kernel-oops/ teaches how to analyse and debug/deal with kernel oops. Like how to use gdb to add-symbol-file, to check a module's sections in /sys/module/MOD_NAME/sections/*. How to use scripts/decodecode, how to use gdb to find the fault instruction, and in a C-format. Check that link.

5. System hangs:
Sysrq: also see LKD. Magic Sysrq may be disabled at runtime with a command:
	echo 0 > /proc/sys/kernel/sysrq;
The file /proc/sysrq-trigger is a write-only entry point, where you can trigger a specific sysrq action by writing the associated command character. You can then collect output data from the kernel logs. This entry point to sysrq is always working, even if sysrq is disabled on the console.

6. Debugging tools:
• gdb:
	$ gdb /usr/src/linux-2.6.39/vmlinux /proc/kcore;
The 1st argument is the name of the uncompressed ELF kernel executable, not the zImage or bzImage.
The 2nd argument is the name of the core file. Like any file in /proc, /proc/kcore is generated when read. kcore is used to represent the kernel executable in the format of a core file. It is huge because it represents the whole kernel address space, which corresponds to all physical memory. gdb optimizes access to the core file by caching data that has already been read, so if you try to look at the jiffies once again, you'll get the same answer as before. The solution to get rid of the cache is to issue the command "core-file /proc/kcore" when you want to flush the gdb cache.

	$ gdb add-symbol-files XXX.ko XXXXXXXX \
				-s .bss XXXXXXX \
				-s .data XXXXXXXX

But we still cannot perform typical debugging tasks like setting breakpoints or modifying data. To perform those operations, we need to use a tool like kdb or kgdb.

7. Other tools:
• The kdb debugger:
kdb works only on IA-32(x86) systems. Note that everything the kernel does stops when kdb is running. See p102.
• The kgdb patches:
There are two seperate patches. They work by seperating i) the system running the test kernel from ii) the system running the debugger. These two are typically connected via a serial cable. In addition to the serial port, kgdb can also communicate over a local-area network. The documentation under Documentation/i386/kgdb describes how to set things up.
kgdb patch could also be found on http://kgdb.sf.net/
• The User-Mode linux port:
User-Mode Linux (UML) doesn't run on a new type of hardware; instead, UML allows the Linux kernel to run as a separate, user-mode process on a linux system. This allows gdb to debug in user-space, because it is just another process. The only shortcoming may be for the driver writer because UML couldn't access the host system's hardware.
See http://user-mode-linux.sf.net for more information.
• The Linux Trace Toolkit (LTT)
See http://www.opersys.com/LTT
• Dynamic Probes (DProbes):
DProbes is a debugging tool released by IBM for linux on the IA-32 architecture. See http://oss.software.ibm.com
