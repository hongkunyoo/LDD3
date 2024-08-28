1. Kobject, kobj_type, and kset:
Kobject subsystem handles the generation of events that notify user space about the comings and goings of hardware on the system.
i) Kobject basics (also see LKD/chapter_17):
It's rare for kernel code to create a standalone kobject, instead, kobjects are always found embedded in other structures.

• kobject initialization:
* The first is to set the entire kobject to 0, usually with a call to memset(). Failure to zero out a kobject often leads to very strange crashes further down the line, this step shouldn't be omitted.
* The next step is to set up some of the internal fields with a call to kobject_init(): (also see LKD/chapter_17)
	void kobject_init(struct kobject *kobj, struct kobj_type *ktype);
	int kobject_set_name(struct kobject *kobj, const char *fmt, ...); // set the name of the kobject, which is used in sysfs entries. Note that this call may fail, so you should check its return value.

• Reference count manipulation:
	struct kobject *kobject_get(struct kobject *kobj); // Increase kobj's ref_cnt by 1.
	void kobject_put(struct kobject *kobj);  // Decrease kobj's ref_cnt by 1, and if it becomes 0, the release() function pointed at by the kobj_type associated with the kobject is invoked.
Note that in many cases, the reference count in the kobject itself may not be sufficient to prevent race conditions. The existence of a kobject may require the continued existence of the module that created that kobject. It would not do to unload that module while the kobject is still being passed around. See source code of cdev_get().

• Release functions and kobj_type:
Every kobject must have a release() method, which is embedded in kobj_type.
	struct kobj_type *get_ktype(struct kobject *kobj);  // finds the kobj_type pointer for a given kobject.

• kobject hierarchies, kset:
* kobject structure has a parent pointer, which is used to position the object in the sysfs hierarchy.
* kset is used as containment, it can be thought of as the top-level container class for kobjects. It's worth noting that ksets are always represented in sysfs once a kset has been set up and added to the system, there will be a sysfs directory for it. Kobjects do not necessarily show up in sysfs, but every kobject that is a member of a kset is represented there.
	int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...); // see LKD/chapter_17 for more details.
	void kobject_del(struct kobject *kobj); // Remove a kobject's sysfs representation
* A kset keeps its children in a standard kernel linked list. In almost all cases, the contained kobjects also have pointers (parent pointer) to the kset (or strictly, kset's embedded kobject)
        ----------------------
        | kset   ----------- | 
|------>|        | kobject | |----------     ----> : kset child list
|       ----------------------         |           
|         ^  ^        ^   ^            |         ^
|        // //       /   /             |        // : kobject->kset
|                                      |
|   -----------		 -----------       |        ^
<---| kobject | <--- | kobject | <------       /   : kobject->parent
    -----------      -----------

* Operations on kset:
	void kset_init(struct kset *kset);  
	int kset_register(struct kset *kset); // initialize and add a kset
	void kset_unregister(struct kset *kset);
	struct kset *kset_get(struct kset *kset); // manage the reference counts of ksets.
	void kset_put(struct kset *kset);
Note that a kset's ref_cnt is actually its embedded kobject's reference count.


2. Low-level sysfs operations:
For every directory found in sysfs, there is a kobject lurking somewhere within the kernel (kobject is embedded in kset). Every kobject of interest also exports one or more attributes, which appear in that kobject's sysfs directory as files containing kernel-generated information.
• kobject_add() see LKD/chapter_17's notes. Sysfs entries for kobjects are always directories, so a call to kobject_add() results in the creation of a directory in sysfs. Usually that directory contains one or more attributes.
i)   Default attributes:
When created, every kobject is given a set of default attributes. These attributes are specified by way of the kobj_type structure:
	struct kobj_type {
		void (*release)(struct kobject *);
		struct sysfs_ops *sysfs_ops;
		struct attribute **default_attrs;
	};
The @default_attrs field lists the attributes to be created for _EVERY_ kobject of this type, and sysfs_ops provides the methods to implement those attributes. Note that the last entry in @default_attrs must be NULL.
	struct sysfs_ops {
		ssize_t (*show)(struct kobject *kobj, struct attribute *attr, char *buffer);
		ssize_t (*store)(struct kobject *kobj, struct attribute *attr, const char *buffer, size_t size);
	};
The same show/store method is used for all attributes associated with a given kobject. (See fs/sysfs/file.c: fill_read_buffer(), flush_write_buffer(). See also drivers/base/core.c: device_initialize(), device_ktype, etc.) The show() method should encode the value into buffer, which length is limited to PAGE_SIZE. The store() method should decode the data stored in buffer and respond to the kernel, note that you should validate it carefully because it comes from user space.
• Nondefault attributes:
If you wish to add a new attribute to a kobject's sysfs directory:
	int sysfs_create_file(struct kobject *kobj, struct attribute *attr);
Note that the same show() and store() functions are called to implement operations on the new attribute.

ii)  Binary attributes:
Here's need to pass large chunks of binary data between user space and kernel space, like uploading firmware to devices. 
	struct bin_attribute {
		struct attribute attr;
		size_t size;
		void *private;
		ssize_t (*read)(struct kobject *kobj, struct bin_attribute *bin_attr, char *buffer, loff_t pos, size_t size);
		ssize_t (*write)(struct kobject *kobj, struct bin_attribute *bin_attr, char *buffer, loff_t pos, size_t size);
		int (*mmap)(struct kobject *kobj, struct bin_attribute *bin_attr, struct vm_area_struct *vma);
	};
The @size is the maximum size of the binary attribute (or 0 if there is no maximum). Note there's no way for sysfs to signal the last of a set of write operations, so code implementing a binary attribute must be able to determine the end of the data some other way.
	int sysfs_create_bin_file(struct kobject *kobj, struct bin_attribute *attr);
	int sysfs_remove_bin_file(struct kobject *kobj, struct bin_attribute *attr);

iii) Symbolic links:
	int sysfs_create_link(struct kobject *kobj, struct kobject *target, const char *name); // Creates a link (called @name) pointing to target's sysfs entry as an attribute of kobj.
	void sysfs_remove_link(struct kobject *kobj, const char *name);
Note that the link persists even if target is removed from the system. So you should probably have a way of knowing about changes to those kobjects or some sort of assurance that the target kobjects will not disappear.

iv)  Hotplug event generation (uevent):
A hotplug event is a notification to user space from the kernel that something has changed in the system's configuration. They are generated whenever(after) a kobject is created or destroyed. Before the event is handed to user space, code associated with the kobject (or more specifically the kset to which it belongs) has the opportunity to add information for user space or to disable event generation entirely.
	struct kset_uevent_ops {
		int (*filter)(struct kset *kset, struct kobject *kobj);
		const char *(*name)(struct kset *kset, struct kobject *kobj);
		int (*uevent)(struct kset *kset, struct kobject *kobj, struct kobj_uevent_env *env);
	};
	struct kobj_uevent_env {
		char *envp[UEVENT_NUM_ENVP];
		int envp_idx;
		char buf[UEVENT_BUFFER_SIZE];
		int buflen;
	};
• A pointer to this structure is found in the @kset_uevent_ops field of the kset structure. If a given kobject is not contained within a kset, the kernel searches up through the hierarchy (use parent pointer) until it finds a kobject that does have a kset, that kset's kset_uevent_ops is then used.
• The @filter function is called whenever kernel is considering generating an event for a given kobject. If filter() returns 0, the event is not created. This method thus gives kset code an opportunity to decide which events should be passed on to user space. (See the source code of kobject_uevent_env().)
• The @name function should return a simple string suitable for passing to user space.
• The final @uevent function gives an opportunity to add useful environment variables prior to the invocation of that (user-space) script. The @envp array is used to store additional environment variable definitions (in the NAME=value format). It has @envp_idx entries in total. The variables themselves should be encoded into @buf, which is @buflen bytes long. Be sure to use NULL as the last entry in the envp. The return value should normally be 0, any nonzero return aborts the generation of the hotplug event.


3. Buses, devices, and drivers:
i)   Buses:
A bus is a channel between the processor and one or more devices. Buses can plug into each other. See <linux/device.h>: struct bus_type { ...
A new bus must be registered with the system via a call bus_register(); which could fail so the return value must always be checked. If succeed, the new bus subsystem has been added to the system and it is visible in sysfs under /sys/bus/.
	int bus_register(struct bus_type *bus);
	void bus_unregister(struct bus_type *bus);
• Bus methods:
	int (*match)(struct device *device, struct device_driver *driver); // This method is called multiple times whenever a new device or driver is added for this bus. It should return a nonzero value if the given device can be handled by the given driver. The function must be handled at bus level because that is where the proper logic exists.
	int (*uevent)(struct device *device, struct kobj_uevent_env *env); // This allows the bus to add variables to the environment prior to the generation of a hotplug event in user space.
• Iterating over devices and drivers:
You may find yourself having to perform some operation on all devices or drivers that have been registered with your bus. To operate on every device known to the bus:
	int bus_for_each_dev(struct bus_type *bus, struct device *start, void *data, int (*fn)(struct device *, void *)); // If @start is NULL the iteration begins with the first device on bus; otherwise iteration starts with the first device after start. If @fn returns a nonzero value, iteration stops and that value is returned from bus_for_each_dev.
	int bus_for_each_drv(struct bus_type *bus, struct device_driver *start, void *data, int (*fn)(struct device_driver *, void *));
• Bus attributes:
	struct bus_attribute {
		struct attribute attr;
		ssize_t (*show)(struct bus_type *bus, char *buf);
		ssize_t (*store)(struct bus_type *bus, const char *buf, size_t count);
	};
	BUS_ATTR(name, mode, show, store); // This macro declares a structure, generating its name by prepending the string "bus_attr_" to the given @name. See drivers/base/bus.c, bus_sysfs_ops.
	int bus_create_file(struct bus_type *bus, struct bus_attribute *attr);
	void bus_remove_file(struct bus_type *bus, struct bus_attribute *attr);

ii)  Devices:
	struct device {
		...
		void (*release)(struct device *dev); // This method is called when the last reference to the device is removed. All device structures registered with the core must have a release method, or the kernel prints out complains. See drivers/base/core.c: devide_ktype for more details.
	}; // As a general rule, (dev->kobj).parent == &(dev->parent->kobj).
• Device registration:
	int device_register(struct device *dev);
	void device_unregister(struct device *dev);
• Device attributes:
	struct device_attribute {
		struct attribute attr;
		ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
		ssiez_t (*store)(struct device *dev, struct device_attribute *attr, const char *buf, size_t count);
	};
	DEVICE_ATTR(name, mode, show, store); // This creates a structure with the name of prepending "dev_attr_" to the given @name.
	int device_create_file(struct device *dev, struct device_attribute *attr);
	void device_remove_file(struct device *dev, struct device_attribute *attr);
• The @dev_attrs field in struct bus_type points to a list of default attributes created for every device added to that bus.
• struct device is usually embedded within a higher-level representation of the device.

iii) Device drivers:
Device drivers can export information and configuration variables that are independent of any specific device.
	struct device_driver {
		const char *name;
		struct bus_type *bus;
		struct module *owner;
		const char *mod_name;
		bool suppress_bind_attrs;
		int (*probe)(struct device *dev);
		int (*remove)(struct device *dev);
		void (*shutdown)(struct device *dev);
		int (*suspend)(struct device *dev, pm_message_t state);
		int (*resume)(struct device *dev, );
		const struct attribute_group **groups;
		const struct dev_pm_ops *pm;
		struct driver_private *p;
	};
	int driver_register(struct device_driver *drv);
	void driver_unregister(struct device_driver *drv);
	struct driver_attribute {
		struct attribute attr;
		ssize_t (*show)(struct device_driver *drv, char *buf);
		ssize_t (*store)(struct device_driver *drv, const char *buf, size_t count);
	};
	DRIVER_ATTR(name, mode, show, store);
	int driver_create_file(struct device_driver *drv, struct driver_attribute *attr);
	void driver_remove_file(struct device_driver *drv, struct driver_attribute *attr);
• The struct bus_type contains a field @drv_attrs that points to a set of default attributes, which are created for all drivers associated with that bus.
• struct device_driver is usually embedded within a higher-level, bus-specific structure.

iv)  Classes:
A class is a higher-level view of a device that abstracts out low-level implementation details.
	struct class {
		const char *name;
		struct module *owner;
		struct class_attribute *class_attrs;
		struct device_attribute *dev_attrs;
		struct kobject *dev_kobj;
		int (*dev_uevent)(struct device *device, struct kobj_uevent_env *env);
		char *(*devnode)(struct device *dev, mode_t *mode);
		void (*class_release)(struct class *class);
		void (*dev_release)(struct device *dev);
		int (*suspend)(struct device *dev, pm_message_t state);
		int (*resume)(struct device *dev);
		const struct dev_pm_ops* pm;
		struct class_private *p;
	};
Each class has a unique name, which appears under /sys/class/. When the class is registered, all of the attributes listed in the NULL-terminated array pointed to by @class_attrs is created. There's also a set of default attributes for every device added to the class, @dev_attrs points to them. @dev_release is called whenever a device is removed from the class; @class_release is called when the class itself is released.
	int class_register(struct class *cls);
	void class_unregister(struct class *cls);
	struct class_attribute {
		struct attribute attr;
		ssize_t (*show)(struct class *cls, char *buf);
		ssize_t (*store)(struct class *cls, const char *buf, size_t count);
	};
	CLASS_ATTR(name, mode, show, store);
	int class_create_file(struct class *cls, const struct class_attribute *attr);
	void class_remove_file(struct class *cls, const struct class_attribute *attr);
• Class interfaces:
	struct class_interface {
		struct list_head node;
		struct class *class;
		int (*add_dev)(struct device *, struct class_interface *);
		void (*remove_dev)(struct device *, struct class_interface *);
	};
	int class_interface_register(struct class_interface *intf);
	void class_interface_unregister(struct class_interface *intf);
* Class interface is a sort of trigger mechanism that can be used to get notification when devices enter or leave the class. Whenever a device is added to the class specified in clasS_interface, the @add_dev is called, it could perform any additional setup required for that device. When the device is removed from the class, the @remove_dev method is called to perform any required cleanup.
* Multiple interfaces can be registered for a class.

• Put it all together:
See the picture of LDD3 P392, device-creation process.


4. Hotplug:
The kernel views hotplugging as an interaction between the hardware, the kernel, and the kernel driver. Users view hotplugging as the interaction between the kernel and user space.
• udev: (also see the documentation of udev)
For udev to work properly, all drivers need to ensure that any major and minor numbers assigned to a device controlled by the driver are exported to user space through sysfs. i) For any driver that uses a subsystem to assign it a major and minor number, this is already done by the subsystem, so driver needs to do nothing about it. (like tty, misc, usb, input, scsi, block, i2c, network, etc.)  ii) If the driver handles getting a major/minor number on its own, (through a call to cdev_init() or the older register_chrdev()), the driver needs to be modified in order for udev to work properly.
* udev looks for a file called 'dev' in the /sys/class/ tree in order to determine what major and minor number is assigned to a specific device. A device driver merely need to create that file for every device it controls. (Could use print_dev_t() to properly format the major and minor number for the specific device.)


5. Dealing with firmware:
• The kernel firmware interface:
The proper solution is to obtain the firmware from user space when you need it.
	#include <linux/firmware.h>
	struct firmware {
		size_t size;
		const u8 *data;
		struct page **pages;
	};
	int request_firmware(const struct firmware **fw, const char *name, struct device *device); // A call to reques_firmware() requests user space to locate and provide a firmware image to the kernel. If a firmware is successfully loaded, it returns 0 and the @fw argument is set. fw contains the actual firmware, which can now be downloaded to the device.
	void release_firmware(const struct firmware *fw); // After sent firmware to device, you should release the in-kernel structure.
	int request_firmware_nowait(struct module *module, const char *name, struct device *device, void *context, void (*cont)(const struct firmware *fw, void *context));
	// Since request_firmware() asks user space to help, it is guaranteed to sleep before returning. If your driver is not in a position allowed to sleep when it must ask for firmware, you must use the _nowait alternative. (@module is always set to THIS_MODULE, @context is a private data pointer not used by the firmware subsystem.) If all goes well request_firmware_nowait() begins a firmware load process and returns 0. At some future time @cont will be called with the result of the load.

• How it works:
The firmware subsystem works with sysfs and hotplug mechanism. When a call is made to request_firmware(), a new directory is created under /sys/class/firmware/ using your device's name. The directory contains 3 attributes:
* loading: It should be set to 1 by the user-space process that is loading the firmware. When the load completes, it should be set to 0. Writing -1 to it aborts the firmware loading process.
* data: It is a binary attribute that receives the firmware data itself. After setting loading, the user-space process should write the firmware to this attribute.
* device: It is a symbolic link to the associated entry under /sys/devices.

Once the sysfs entries have been created, the kernel generates a hotplug event for the device. The environment passed to the uevent handler includes a variable FIRMWARE, which is set to the name provided to request_firmware(). The handler should locate the firmware file, copy it to kernel using the atrributes provided. if the file couldn't be found, the handler should set the loading file to -1.
If a firmware request isn't serviced within 10 seconds, the kernel gives up and returns a failure status to the driver. That time-out value can be changed via the sysfs entry /sys/class/firmware/timeout.
