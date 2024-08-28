• The linux kernel supports two main types of USB drivers: i) drivers on a host system (USB host drivers); ii) drivers on a device (USB gadget drivers). This chapter we'll discuss about USB host drivers.

1. USB device basics:
• USB driver overview:
    ----------------------------------------------------------------------
    | VFS layer | Block layer | Net layer | Char layer | TTY layer | ... |
    ----------------------------------------------------------------------
    |                        USB Device Drivers                          |
    ----------------------------------------------------------------------
    |                             USB Core                               |
    ----------------------------------------------------------------------
    |                        USB Host Controllers                        |
    ----------------------------------------------------------------------
                                       ^
                                   Hardware

• USB device overview:
    -----------------------------------------------
    | Device                                      |
    | ------------------------------------------  |
    | | Config                                 |  |
    | |         ----------------------------   |  |
    | |         | Interface                |   |  |       --------------
    | |         |           ------------   |<--|--|-------| USB Driver |
    | |         |           | Endpoint |   |   |  |       --------------
    | |         |           ------------   |   |  |
    | |         |           | Endpoint |   |   |  |
    | |         |           ------------   |   |  |
    | |         |           | Endpoint |   |   |  |
    | |         |           ------------   |   |  |
    | |         ----------------------------   |  |
    | |                                        |  |
    | |         ----------------------------   |  |
    | |         | Interface                |   |  |       --------------
    | |         |           ------------   |<--|--|-------| USB Driver |
    | |         |           | Endpoint |   |   |  |       --------------
    | |         |           ------------   |   |  |
    | |         |           | Endpoint |   |   |  |
    | |         |           ------------   |   |  |
    | |         |           | Endpoint |   |   |  |
    | |         |           ------------   |   |  |
    | |         ----------------------------   |  |
    | |                                        |  |
    | ------------------------------------------  |
    |                                             |
    -----------------------------------------------

* Endpoints:
A USB endpoint can carry data in only one direction, eigher from the host PC to device (OUT endpoint) or from the device to host PC (IN endpoint). A USB endpoint can be one of the four types:
i)   CONTROL:
Every USB device has a control endpoint called "endpoint 0" that is used by the USB core to configure the device at insertion time. These transfers are guaranteed by the USB protocol to always have enough reserved bandwidth to make it through to the device.
ii)  INTERRUPT:
Interrupt endpoints transfer small amounts of data at a fixed rate every time the USB host asks the device for data. They are also commonly used to send data to USB devices to control the device, but are not generally used to transfer large amounts of data. These transfers are guaranteed by the USB protocol to always have enough reserved bandwidth to make it through.
iii) BULK:
Bulk endpoints transfer large amounts of data. They are common for devices that need to transfer any data that must get through with no data loss. These transfers are NOT guaranteed by the USB protocol to always make it through in a specific amount of time. If there is not enough room on the bus to send the whole BULK packet, it is split up across multiple transfers to or from the device. (These endpoints are common on printers, storage, and network devices.)
iv)  ISOCHRONOUS:
Isochronous endpoints also transfer large amounts of data, but the data is NOT always guaranteed to make it through. These endpoints are used in device that can handle loss of data, and rely more on keeping a constant stream of data flowing. Real-time collections, such as audio and video devices, almost always use these endpoints.
• Control and bulk endpoints are used for asynchronous data transfers, whenever the driver decides to use them. Interrupt and isochronous endpoints are periodic. This means that these endpoints are set up to transfer data at fixed times continuously, which causes their bandwidth to be reserved by the USB core.
• USB endpoints are described in the kernel with the structure @usb_host_endpoint, which contains the real endpoint information in another struture called @usb_endpoint_descriptor. The fields in @usb_endpoint_descriptor that drivers care about are:
	bEndpointAddress: 8-bit value, which could be masked using USB_DIR_OUT and USB_DIR_IN to determine if the data for this endpoint is directed to the device or the host.
	bmAttributes: the type of this endpoint, which could be masked using USB_ENDPOINT_XFERTYPE_MASK to determine if the endpoint is of type USB_ENDPOINT_XFER_ISOC, USB_ENDPOINT_XFER_BULK, or USB_ENDPOINT_XFER_INT.
	wMaxPacketSize: The maximum size in bytes that this endpoint can handle at once.
	bInterval: If this endpoint of type interrupt, this value is the interval setting for the endpoint - the time between interrupt requests for the endpoint.

* Interfaces:
USB endpoints are bundled up into interfaces. USB interfaces handle only one type of a USB logical connection, such as a mouse, a keyboard, etc. Because a USB interface represents basic functionality, each USB driver controls an interface. USB interfaces are described in the kernel with @struct usb_interface (main USB device structure that all USB drivers use to communicate with the USB core.), some of the important fields in it are:
	@struct usb_host_interface *altsetting: An array of interface structures containing all of the alternate settings that may be selected for this interface. Each struct usb_host_interface consists of a set of endpoint configurations as defined by the struct usb_host_endpoint.
	@unsigned num_altsetting: number of alternate settings pointed to by the altsetting pointer.
	@struct usb_host_interface *cur_altsetting: A pointer into the array altsetting, denoting the currently active setting for this interface.
	@int minor: If the USB driver bound to this interface uses the USB major number, this variable contains the minor number assigned by the USB core to the interface. This is valid only only after a successful call to usb_register_dev().

* Configurations:
USB interfaces themselves are bundled up into configurations. Linux describes USB configurations with @struct usb_host_config, describes the entire USB devices with @struct usb_device. 
• The USB driver commonly has to convert data from a given @struct usb_interface into @struct usb_device that the USB core needs for a wide range of function calls:
	struct usb_device *interface_to_usbdev(struct usb_interface *intf);

* Devices usually have one or more configurations; Configurations often have one or more interfaces; Interfaces usually have one or more settings, interfaces often have zero or more endpoints.


2. USB and sysfs:
• Both the physical USB device (as represented by @struct usb_device) and the individual USB interfaces (as represented by @struct usb_interface) are shown in sysfs as individual devices. (This is because both of these structures contain a @struct device structure.) See the example at LDD3 P333.
• To summarize, the USB sysfs device naming scheme is:
	root_hub-hub_port:config.interface
As the devices go further down in the USB tree, and as more and more USB hubs are used, the hub port number is added to the string following the previous hub port number in the chain. For a two-deep tree, the device name looks like:
	root_hub-hub_port-hub_port:config.interface
• Sysfs doesn't expose all of the different parts of a USB device, as it stops at the interface level. There's usbfs filesystem, which is mounted in the /proc/bus/usb/. Search the web for more information.


3. USB Urbs:
• The USB code in the kernel communicates with all USB devices using something called a urb (USB request block). This request block is described with @struct urb, which is defined in <linux/usb.h>.
• A urb is used to send or receive data to or from a specific USB endpoint on a specific USB device in an asynchronous manner. A USB device driver may allocate many urbs for a single endpoint, or may reuse a single urb for many different endpoints. 
• Every endpoint in a device can handle a queue of urbs, so that multiple urbs can be sent to the same endpoint before the queue is empty.

• The typical lifecycle of a urb is as follows: (also refer to the two diagrams above)
i)   Created by a USB device driver.
ii)  Assigned to a specific endpoint of a specific USB device.
iii) Submitted to the USB core, by the USB device driver.
iv)  Submitted to the specific USB host controller driver for the specified device by the USB core.
v)   Processed by the USB host controller driver that makes a USB transfer to the device.
vi)  When the urb is completed, the USB host controller driver notifies the USB device driver.
• Urbs can also be canceled any time by the driver that submitted the urb, or by the USB core if the device is removed from the system.
Detailed fields in the @struct urb see LDD3 P336 ~ P341 and the source code for more details.

• Creating and destroying urbs:
* Note that @struct urb structure must NEVER be created statically in a driver or within another structure, because that would break the reference counting scheme used by the USB core for urbs. It must be created with:
	struct urb *usb_alloc_urb(int iso_packets, gfp_t mem_flags);
	@iso_packets: the number of osochronous packets this urb should contain. If you do not want to create an isochronous urb, this variable should be set to 0.
	@mem_flags: the same type of flag that is passed to kmalloc().
	void usb_free_urb(struct urb *urb);
* Interrupt urbs:
A helper function to properly initialize a urb to be sent to an interrupt endpoint of a USB device:
	void usb_fill_int_urb(struct urb *urb, struct usb_device *dev, unsigned int pipe, void *trandfer_buffer, int buffer_length, usb_complete_t complete, void *context, int interval);
	@urb: A pointer to the urb to be initialized.
	@dev: The USB device to which this urb is to be sent.
	@pipe: The specific endpoint of the USB device to which this urb is to be sent. This value is created with the previously mentioned usb_sndintpipe() or usb_rcvintpipe() functions.
	@transfer_buffer: A pointer to the buffer from which outgoing data is taken or into which incoming data is received. Note that this cannot be a static buffer and must be created with a call to kmalloc.
	@buffer_length: The length of the buffer pointed to by the transfer_buffer pointer.
	@complete: Pointer to the completion handler that is called when this urb is completed.
	@context: Pointer to the blob that is added to the urb structure for later retrieval by the completion handler function.
	@interval: The interval at which that this urb should be scheduled.
* Bulk urbs:
	void usb_fill_bulk_urb(struct urb *urb, struct usb_device *dev, unsigned int pipe, void *transfer_buffer, int buffer_length, usb_complete_t complete, void *context);
There is no interval argument because bulk urbs have no interval value. And also note that the @pipe parameter must be initialized with a call to the usb_sndbulkpipe() or usb_rcvbulkpipe() function.
* Control urbs:
	void usb_fill_control_urb(struct urb *urb, struct usb_device *dev, unsigned int pipe, unsigned char *setup_packet, void *transfer_buffer, int buffer_length, usb_complete_t complete, void *context);
@setup_packet: must point to the setup packet data that is to be sent to the endpoint.
Also, @pipe must be initialized with a call to usb_sndctrlpipe() or usb_rcvctrlpipe().
* Isochronous urbs:
Isochronous urbs do not have an initialization function like the interrupt, control, and bulk urbs do. So they must be initialized by hand in the driver before they can be submitted to the USB core.

• Submitting urbs:
Once the urb has been properly created and initialized by the USB driver, it is ready to be submitted to the USB core to be sent out to the USB device:
	int usb_submit_urb(struct urb *urb, gfp_t mem_flags);
After a urb has been submitted to the USB core successfully, it should never try to access any fields of the urb structure until the complete function is called.
Because the function @usb_submit_urb() can be called at any time (including from within an interrupt context), the specification of @mem_flags variable must be correct. There are only three valid values that should be used, depending on when usb_submit_urb() is called:
i)   GFP_ATOMIC: This flag should be used if: the caller is within a urb completion handler, an interrupt, a bottom half, a tasklet, or a timer callback. Or if the caller is holding a spinlock or rwlock. Or if the current->state is not TASK_RUNNING.
ii)  GFP_NOIO: This flag should be used if the driver is in the block I/O path, it should also be used in the error handling path of all storage-type devices.
iii) GFP_KERNEL: This could be used for any other situations.

• Completing urbs: the completion callback handler:
If the call to usb_submit_urb() is successful, it returns 0; otherwise it returns a negative error code. Once it succeeds, the completion handler of the urb is called exactly once when the urb is completed. When this function is called, the USB core is finished with the urb, and control of it is now returned to the device driver.

• Canceling urbs:
To stop the urb that has been submitted to the USB core, the functions usb_kill_urb() or usb_unlink_urb() should be called:
	int usb_kill_urb(struct urb *urb);
	int usb_unlink_urb(struct urb *urb);
When the function usb_kill_urb() is called, the urb lifecycle is stopped. This function is usually used when the device is disconnected from the system in the disconnect callback.
For some drivers, the usb_unlink_urb() should be used to tell the USB core to stop a urb. This function doesn't wait for the urb to be fully stopped before returning to the caller. This is useful for stopping the urb while in an interrupt handler or when a spinlock is held. This function requires that the URB_ASYNC_UNLINK flag be set in the urb that is being asked to be stopped in order to work properly.


4. Writing a USB driver:
Writing a USB driver is similar to a pco driver: the driver registers its driver object with the USB subsystem and later uses vendor and device identifiers to tell if its hardware has been installed.
• The @struct usb_device_id provides a list of different types of USB devices that this driver supports.
As with PCI drivers, the @MODULE_DEVICE_TABLE macro is necessary to allow user-space tools to figure out what devices this driver can control. But for USB drivers, the string "usb" must be the first value in the macro. For example:
	MODULE_DEVICE_TABLE(usb, skel_table);

• Registering a USB driver:
The main structure that all USB driver must create is @struct usb_driver, which consists of a number of function callbacks and variables that describe the USB driver to the USB core code:
	@struct module *owner: often set to THIS_MODULE.
	@const char *name: It must be unique among all USB drivers in the kernel. It shows in sysfs under /sys/bus/usb/drivers/.
	@const struct usb_device_id *id_table: Pointer to struct usb_device_id that contains a list of all of the different kinds of USB devices this driver can accept. If this variable is not set, the probe() function in the USB driver is never called. And if you want your driver to be always called for every USB device in the system, create an entry that sets only the @driver_info field:
	static struct usb_device_id usb_ids[] = {
		{ .driver_info = 42 },
		{}
	};
	int (*probe)(struct usb_interface *intf, const struct usb_device_id *id): This function is called by the USB core when it thinks it has a @struct usb_interface that this driver can handle. A pointer to the @struct usb_device_id that the USB core used to make this decision is also passed to this function. If the USB driver claims the struct usb_interface that is passed to it, it should initialize the device properly and return 0. If the driver doesn't want to claim the device, or an error occurs, it should return a nagative error value.
	void (*disconnect)(struct usb_interface *intf): This function is called by the USB core when the @struct usb_interface has been removed from the system or when the driver is being unloaded from the USB core.
	int (*ioctl)(struct usb_interface *intf, unsigned int code, void *buf): It is called when a user-space program makes a ioctl() call on the usbfs filesystem.
	int (*suspend)(struct usb_interface *intf, u32 state);
	int (*resume)(struct usb_interface *intf);
* To register a @usb_driver with the USB core:
	int usb_register(struct usb_driver *driver);
	void usb_deregister(struct usb_driver *driver);

* probe and disconnect in detail:
Both the probe and disconnect function callbacks are called in the context of the USB hub kernel thread, so it is legal to sleep within them. However, it's recommended that the majority of the work be done when the device is opened by a user if possible, in order to keep the USB probing time to a minimum. This is because the USB core handles the addition and removal of USB devices in a single thread, so any slow device driver can cause the USB device detection time to slow down and become noticeable by the user.
• Because the USB driver needs to retrieve the local data structure that is associated with this @struct usb_interface later in the lifecycle of the device, the following function could be called:
	void usb_set_intfdata(struct usb_interface *intf, void *data); // set the private data pointer within the usb_interface.
	void *usb_get_intfdata(struct usb_interface *intf);
• struct usb_class_driver: A structure that describes a USB driver that wants to use the USB major number to communicate with user-space programs. 
	int usb_register_dev(struct usb_interface *intf, struct usb_class_driver *class_driver);
	void usb_deregister_dev(struct usb_interface *intf, struct usb_class_driver *class_driver);
Functions used to register and unregister a specific struct usb_interface * with a struct usb_class_driver *.


5. USB transfers without urbs:
Sometimes a USB driver doesn't want to go through all of the hassle of creating a @struct urb, initializing it, and then waiting for the urb completion function to run, just to send or receive some simple USB data.
i)   usb_bulk_msg():
	int usb_bulk_msg(struct usb_device *usb_dev, unsigned int pipe, void *data, int len, int *actual_length, int timeout);
	@usb_dev: The pointer to the USB device to send to or receive from the bulk message.
	@pipe: The specific endpoint of the USB device to which this bulk message is to be sent. This value is created with a call to either usb_sndbulkpipe() or usb_rcvbulkpipe().
	@data: A pointer to the data to send to the device if this is an OUT endpoint; if this is an IN endpoint, this is a pointer to where the data should be placed after being read from the device.
	@len: The length of the buffer pointed to by @data.
	@actual_length: A pointer to where the function places the actual number of bytes that have either been transferred to or received from the device.
	@timeout: The amount of time, in jiffies, that should be waited before timing out. If this value is 0, the function waits forever for the message to complete.
If the function is successful, the return value is 0; otherwise a negative error number is returned. If successful, the actual_length parameter contains the number of bytes that were transferred or received from this message.

ii)  usb_control_msg():
	int usb_control_msg(struct usb_device *dev, unsigned int pipe, __u8 request, __requesttype, __u16 value, __u16 index, void *data, __u16 size, int timeout);
	If successful, it returns the number of bytes that were transferred to or from the device; otherwise, a nagative error number is returned. The parameters request, requesttype, value, and index all directly map to the USB specification for how a USB control message is defined.
• Note that usb_bulk_msg() and usb_control_msg() cannot be called from within interrupt context or with a spinlock held. Also, they could not be canceled by any other function, so be careful when using it, make sure the driver's disconnect function knows enough to wait for the call to complete before allowing itself to be unloaded from memory.

iii) Other USB data functions:
A number of helper functions in the USB core can be used to retrieve standard information from all USB devices. These functions cannot be called from within interrupt context, or with spinlocks held.
	int usb_get_descriptor(struct usb_device *dev, unsigned char type, unsigned char index, void *buf, int size); // It could be used by a USB driver to retrieve from the @struct usb_device any of the device descriptors that are not already present in the existing struct usb_device and struct usb_interface, such as audio descriptors or other class specific information. See the source code, it uses usb_confif_msg() to do this.
	int usb_get_string(struct usb_device *dev, unsigned short langid, unsigned char index, void *buf, int size); // If successful, this function returns a string encoded in the UTF-16LE format in the buffer pointed to by the @buf. As this format is usually not very useful, there's another function:
	int usb_string(struct usb_device *dev, int index, char *buf, size_t size); // If successful return a string read from a USB device and is already converted into UTF-8 format. So usb_string() is recommended instead of usb_get_string().
