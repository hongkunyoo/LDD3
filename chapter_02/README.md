- insmod and rmmod should be called using sudo or as root. Note that the printk() output of _init and _exit function could only be seen from text console, in a terminal emulator running under the window system maybe you cannot see the output. But it should be in /var/log/messages or /var/log/syslog, or through dmesg.
- Linux kernel code, including driver code, must be reentrant - it must be capable of running in more than one context at the same time.
- Functions with a double underscore (__) in kernel API should be used with caution.
- Kernel code cannot do floating point arithmetic.

1. Compiling and Loading modules:
This could also refer to the notes of LKD chapter_17. For here, we just focus on compiling modules outside the kernel source tree.
obj-m := hello.o
If your kernel source tree is located in, say, ~/kernel-2.6/ directory, and assuming you're currently in the module's directory which contains its own makefile and outside kernel source tree, do:
	```$ make -C ~/kernel-2.6 M=`pwd` modules; ($ make modules; is to build the modules.)```
This commands starts by changing its directory to the one proviced with the -C option, it then finds the kernel's top-level Makefile. The 'M=' option causes that Makefile to move back to the module source directory before trying to build the modules target. This target, in turn, refers to the list of modules found in obj-m variable.
* Another Makefile that does the same but just need to do $ make:

```Makefile
#==========================================================
ifneq ($(KERNELRELEASE),)
	obj-m := hello.o
else
	KERNEL_DIR ?= /lib/modules/$(shell uname -r)/build
	PWD := $(shell pwd)

	default:
		$(MAKE) -C $(KERNEL_DIR) M=$(PWD) modules

endif
#==========================================================
```
Call chain: make -> else -> default -> Makefile in kernel source tree, where defines KERNELRELEASE -> back to module dir -> Makefile in module dir -> obj-m := hello.o.

- Loading and unloading modules:
insmod could also assign values to parameters in the module before linking it to the current kernel. Thus a module can be configured at load time. (It relies on a syscall defined in kernel/module.c; the function sys_init_module allocates kernel memory to hold a module (this memory is allocated with vmalloc), it then copies the module text into that memory region, resolves kernel references in the module via the kernel symbol table, and calls the module's initialization function to get everything going.)
* lsmod gives a list of modules currently loaded in the kernel. It reads the /proc/modules virtual file. Information of currently loaded modules can also be found in the /sys/module/* directories.

- Version dependency:
Note that your module's code has to be recompiled for each version of the kernel that it is linked to. Modules are strongly tied to the data structures and function prototypes defined in a particular kernel version. <linux/version.h>, which is included by <linux/module.h>, defines the following macros:
UTS_RELEASE: defined in <linux/vermagic.h>, contains the string describing the version of this kernel tree, for example: "2.6.10".
LINUX_VERSION_CODE: This macro is a binary (integer) representation of the kernel version. For example: for 2.6.10 -> 0x02060a -> 132618 (LINUX_VERSION_CODE).
KERNEL_VERSION(major, minor, release): This macro converts from 2.6.10 -> 132618 (KERNEL_VERSION(2, 6, 10)).

2. The kernel symbol table:
Note that although modprobe could load all the depended modules automatically, it only looks in the standard installed module directories. So you'll still need insmod when loading your own modules from the current directory.
EXPORT_SYMBOL(name);
EXPORT_SYMBOL_GPL(name);
Symbols must be exported in the global part of the module's file, outside of any function. Because the above macros expand to the declaration of a special-purpose variable that is expected to be accessible globally. This variable is stored in a special part of the module executible (an ELF section) that is used by the kernel at load time to find the variables exported by the module.

- MODULE_LICENSE("GPL"): "GPL" could also be "GPL v2", "GPL and additional rights", "Dual BSD/GPL", "Dual MPL/GPL", and "Proprietary".

- Initialization and shutdown:
__init, __initdata, __exit, __exitdata:
For example: static int __init initialization_func(void); static void __exit cleanup_func(void);
Markers for functions (__init and __exit) and data(__initdata and __exitdata) that are only used at module initialization or cleanup time. Items marked for initialization may be discarded once initialization completes; the exit items may be discarded if module unloading has not been configured into the kernel.
Note that if your module doesn't define a cleanup function, the kernel doesn't allow it to be unloaded.

- Error handling during initialization:
One thing should always bear in mind when registering facilities with the kernel is that the registration could fail. So you may need to check each allocation/registration's return value before going on. 
Error recovery is sometimes best handled with the goto statement.

3. Module-loading races:
i)  The first is that you should always remember that some other part of the kernel can make use of any facility you register immediately after that registration has completed. It is entirely possible that the kernel will make calls into your module while your initialization function is still running. Do not register any facility until all of your internal initialization needed to support that facility has been completed.
ii) You must consider what happens if your initialization function decides to fail, but some part of the kernel is already making use of a facility your module has registered. If this situation is possible for your module, you should seriously consider not failing the initialization at all. After all, the module has succeeded in exporting something useful. If initialization must fail, it must carefully step around any possible operations going on elsewhere in the kernel until those operations have completed.

4. Module parameters:
Note that all module parameters should be given a default value; insmod changes the value only if explicitly told by the user.
If perm is set to 0, there's no sysfs entry at all. perm could be combinations of S_IRUGO, S_IWUSR, etc. Note that if a parameter is changed by sysfs, the value of that parameter as seen by your module changes, but your module is not notified in any other way. You should probably not make module parameters writable, unless you are prepared to detect the change and react accordingly.
