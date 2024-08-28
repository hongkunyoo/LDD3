• Note that every time the kernel presents you with a sector number, it is working in a world of 512-byte sectors. If you're using a different hardware sector size, you have to scale the kernel's sector numbers accordingly.

1. Registration:
• Block driver registration:
	#include <linux/fs.h>
	int register_blkdev(unsigned int major, const char *name); // @name will be displayed in /proc/devices. If @major is passed as 0, then the kernel allocates a new major number and returns it to the caller.
	int unregister_blkdev(unsigned int major, const char *name); // Here the @major and @name must match those passed to register_blkdev(), or the function returns -EINVAL and not unregister anything.
Note that in future kernels, register_blkdev() may be removed altogether; the only tasks performed by this call are: i) allocating a dynamic major number if requested; ii) creating an entry in /proc/devices.

• Block device operations:
See struct block_device_operations.
	int (*open)(struct block_device *, fmode_t); // a block device may respond to an open/release call by spinning up the device, locking the door (for removable media), etc. Actions like partitioning a disk, building a filesystem on a partition, running a file system checker, mounting a partition could lead to the open() method called.
	int (*release)(struct gendisk *, fmode_t); 
	int (*ioctl)(struct block_device *, fmode_t, unsigned cmd, unsigned long arg); // Method that implements the ioctl syscall. The block layer first intercepts a large number of standard requests. So most block driver ioctl methods are fairly short.
	int (*media_changed)(struct gendisk *); // Called by the kernel to check whether the user has changed the media in the drive, returning a nonzero if so. This method is only applicable to drives that support removable media. (This may be deprecated in the future, recommended to use check_events() method.)
	int (*revalidate_disk)(struct gendisk *); // Called in response to a media change. It gives the driver a chance to perform whatever work is required to make the new media ready for use. After the revalidate_disk call, the kernel attempts to reread the partition table and start over with the device. Its return value is ignored by the kernel.
* Note that there's no function in struct block_device_operations that performs reading or writing data. In the block I/O subsystem, these operations are handled by the request() function.

• The gendisk structure:
#include <linux/genhd.h>
struct gendisk is the kernel's representation of an individual disk device. In fact the kernel also uses gendisk structures to represent partitions, but driver authors need not be aware of that. There're several fields in struct gendisk that must be initialized by a block driver:
	int major;
	int first_minor;
	int minors;  // A driver must use at least one minor number. If your drive is to be partitionable, you should allocate one minor number for each possible partition as well.
	char disk_name[32]; // Field that should be set to the name of the disk device. It shows up in /proc/partitions and sysfs.
	struct block_device_operations *fops;
	struct request_queue *queue; // Used by the kernel to manage I/O requests for this device.
	int flags; // Describing the state of the drive. If your device has removable media, you should set GENHD_FL_REMOVABLE. CD-ROM drives can set GENHD_FL_CD. If you do not want partition information to show up in /proc/partitions, set GENHD_FL_SUPRESS_PARTITION_INFO. 
	void *private_data; // Block drivers may use this field to store a pointer to their internal data.
* struct gendisk is a dynamically allocated structure that requires special kernel manipulation to be initialized; drivers cannot allocate the structure on their own.
	struct gendisk *alloc_disk(int minors); // The @minors argument should be the number of minor numbers this disk uses.
	void del_gendisk(struct gendisk *gd);
A gendisk is a reference-counted structure (use its containing kobject). Use get_disk() and put_disk() to manipulate the reference count, but drivers should never need to do that.
Allocating a gendisk structure does not make the disk available to the system. To do that, you must initialize the structure and call add_disk():
	void adD_disk(struct gendisk *gd);
Note that: as soon as you call add_disk(), the disk is "live" and its methods can be called at any time (even before add_disk() returns, the kernel will read the first few blocks in an attempt to find a partition table). So you should not call add_disk() until your driver is completely initialized and ready to respond to the requests on that disk. (Also see chapter_03: cdev_add())


2. Request processing:
The core of every block driver is its request() function. This function is where the real work gets done - or at least started.
	void request(struct request_queue *queue);
* This function is called whenever the kernel believes it is time for your driver to process some reads, writes, or other operations on the device. The request() function does not need to actually complete all of the requests on the queue before it returns. Indeed, it probably doesn't complete any of them for most real devices. It must, however, make a start on those requests and ensure that they all, eventually, processed by the driver.
* Every device has a request queue. And a request function is associated with a request queue when that queue is created:
	struct request_queue *blk_init_queue(void (*request_fn_proc)(struct request_queue *q), spinlock_t *lock);
Note that a spinlock must be provided in the queue creation process. No matter when the request() method is called by the kernel, that lock is held. So the request() method is running in an atomic context, it must follow the rules for atomic context discussed in chapter 05. The queue spinlock also prevents the kernel from queueing any other requests for your device while the request() function holds the lock (is running). In some conditions, you may want to consider dropping that lock while the request() function runs. If you do so, you must be sure not to access the request queue, or any other data structure protected by that lock. You must also re-acquire the lock before the request() function returns.
* In the invocation of request() function, any sort of operation that explicitly accesses user space is errornous.
* bool blk_end_request(struct request *rq, int error, int nr_bytes); // Helper function for drivers to complete the request
@rq: the request being processed.
@error: 0 for success; < 0 for error.
@nr_bytes: number of bytes to complete.
@return value: true - still buffers pending for this request; false - we are done with this request. (NOTE that this return value is special!)
See block/blk-core.c for more helper functions about _request.
* Some fields in struct request:
char *buffer: kernel virtual address of the current segment (which the data should be transferred), if available.
rq_data_dir(struct request *rq): The macro extracts the direction of the transfer. A zero denotes a read from the device; nonzero denotes a write to the device.

• Request queues:
Defined in <linux/blkdev.h>, a request queue keeps track of the outstanding block I/O requests. But they also play a crucial role in the creation of those requests. The request queue stores parameters that describe what kind of requests the device is able to service: the maximum size, how many separate segments may go into a request, the hardware sector size, alignment requirements, etc. If your request queue is properly configured, it should never present you with a request that your device cannot handle.
Request queues also implement a plug-in interface that allows the use of multiple I/O schedulers.
* Queue creation and deletion:
	struct request_queue *blk_init_queue(request_fn_proc *request, spinlock_t *lock); // (allocates memory) creates and initializes a request queue. In initialization, you can also set the field @void *queuedata to any value you like.
	void blk_cleanup_queue(struct request_queue *q); // To return a request queue to the system (at module unload time, generally)
	struct request *blk_peek_request(struct request_queue *q); // Return the next request to be processed. It leaves the request on the queue, but marks it as being active; this prevents the I/O scheduler from attempting to merge other requests with this one once you start to execute it.
	void blk_dequeue_request(struct request *rq); // To actually remove a request from a queue.
	void elv_requeue_request(struct request_queue *q, struct request *rq); // To put a dequeued request back on the queue
	void blk_stop_queue(struct request_queue *q); 
	void blk_start_queue(struct request_queue *q); // If your device has reached a state where it can handle no more outstanding commands, you can call blk_stop_queue() to tell the block layer. After this call, the request() function will not be called until you call blk_start_queue(). The queue lock must be held when calling either of these functions.
	void blk_queue_bounce_limit(struct request_queue *q, u64 dma_addr); // Function that tells the kernel the highest physical address to which your device can perform DMA. If a request comes from the memory above the limit, a bounce buffer will be used for the operation, which is an expensive way to perform block I/O and should be avoided whenever possible. The default value of dma_addr is BLK_BOUNCE_HIGH, which uses bounce buffers for high memory pages.
	void blk_queue_max_hw_segments(struct request_queue *q, unsigned short max); // maximum number of segments the device itself can handle.
	void blk_queue_max_segment_size(struct request_queue *q, unsigned int max); // how large any individual segments of a request can be in bytes.
	void blk_queue_segment_boundary(struct request_queue *q, unsigned long mask); // Some devices cannot handle requests that cross a particular size memory boundary. If your device is one of those, use this function to tell the kernel about this boundary. For example, if the boundary is 4MB, mask is set as 0x3fffff. The default mask is 0xffffffff.
	void blk_queue_dma_alignment(struct request_queue *q, int mask); // Function that tells the kernel about the memory alignment constraints the device imposes on DMA transfers. All requests are created with the given alignment, and the length of the request also matches the alignment. The default mask is 0x1ff, which causes all requests to be aligned on 512-byte boundaries.

• The anatomy of a request:
Each request structure represents one block I/O request. The request is represented as a set of segments, each of which corresponds to one in-memory buffer. The kernel may join multiple requests that involve adjacent sectors on the disk, but it never combines read and write operations within a single request structure.
The struct request is implemented as a linked list of bio structures combined with some housekeeping information to enable the driver to keep track of its position as it works through the request.
struct bio, see LKD chapter_14 and source code for more details. By the time a block I/O request is turned into a bio structure, it has been broken down into individual pages of physical memory. All drivers need to step through the array of structures (bi_vcnt in total), and transfer data within each page (but only @len bytes starting at @offset)
* 	int segno;
	struct bio_vec *bvec;
	bio_for_each_segment(bvec, bio, segno) {  // macro used to step through bi_io_vec array.
		...
	}
	void *__bio_kmap_atomic(struct bio *bio, int i); // directly map the buffer found in a given bio_vec, i is its index in the array. An atomic kmap is created.
	void __bio_kunmap_atomic(void *kaddr);
	struct page *bio_page(struct bio *bio); // Returns a pointer to the page structure representing the page to be transferred next.
	int bio_offset(struct bio *bio); // Returns the offset within the page for the data to be transferred.
	void *bio_data(struct bio *bio); // Returns a kernel virtual address pointing to the data to be transferred. See the source code for more details. Note that this k-v-addr is available only if the page is not located in high memory, or already mapped to the kernel virtual address space. (since it uses page_address() to obtain the kernel virtual address.)
	char *bio_kmap_irq(struct bio *bio, unsigned long *flags); // Returns a kernel virtual address for the current (@bi_idx in struct bio) buffer to be transferred, regardless of whether it resides in high or low memory. An atomic kmap is used, so your device cannot sleep while this mapping is active (in atomic context).
	void bio_kunmap_irq(char *buffer, unsigned long *flags);

• Some fields in struct request:
@struct list_head queuelist: Embedded linked list structure. Note that all the requests of the request queue are in a linked list.
@struct bio *bio: The linked list of bio structures for this request.
@char *buffer: Kernel virtual address of the current segment (which the data should be transferred), if available.
@unsigned short nr_phys_segments: The number of distinct segments occupied by this request in physical memory after adjacent pages have been merged.

• Barrier requests:
The block reorders requests before your driver sees them to improve I/O performance. Your driver can also reorder requests to the drive and letting the hardware figure out the optimal ordering. But if some applications require guarantees that certain operations will complete before other are started, in this case, you have to use a barrier request. If a request is marked with the REQ_HARDBARRERcflag, it must be written to the drive (the data must actually reside and be persistent on the physical media) before any following request is initiated.

• Nonretryable requests:
Block drivers often attempts to retry requests that fail the first time. However, the kernel sometimes marks requests as not being retryable. Such requests should simply fail as quickly as possible if they cannot be executed on the first try. If your driver is considering retrying a failed request, it should first make a call to:
	int blk_noretry_request(struct request *rq);
If the macro returns a nonzero value, your driver should simply abort the request with an error code instead of retrying it.

• Block requests and DMA:
A block driver can certainly step through the bio structures, creating a DMA mapping for each one and pass the result to the device. There's an easy way if your device can do scatter/gather I/O:
	int blk_rq_map_sg(struct request_queue *q, struct request *rq, struct scatterlist *list);
This function fills in the given @list with the full set of segments from the given request. Segments that are adjacent in memory are coalesced prior to insertion into the scatterlist. The return value is the number of entries in the list. The function also passes back the @list argument that could be passed to dma_map_sg(). (Your driver must allocate the storage for the scatterlist before calling blk_rq_map_sg(). Note that the list must be able to hold at least as many entries as the request has physical segments - @nr_phys_segments in the struct request holds the value.) Note that if you do not want blk_rq_map_sg() to coalesce adjacent segments, you can change the default behavior with a call:
	clear_bit(QUEUE_FLAG_CLUSTER, &queue->queue_flags);


• Doing without a request queue:
Many block-oriented devices, such as flash memory arrays, ramdisks that have truly random-access performance and do not benefit from advanced request queueing logic. For these kind of devices, it would be better to accept requests directly from the block layer and not bother with the request queue at all.
* The block layer supports a "no queue" mode of operation. Your driver must provide a "make request" function, rather than a request() function.
	typedef void (make_request_fn)(struct request_queue *q, struct bio *bio);
Note that a request_queue is still present, even though it will never actually hold any requests. @bio in the make_request_fn represents one or more buffers to be transferred. The make_request_fn can do two things: it can either perform the transfer directly, or it can redirect the request to another device.
i)  Performing the transfer directly:
It is just a matter of working through the bio. Since there's no request structure to work with, your function should signal completion directly to the creator of the bio structure with the call:
	void bio_endio(struct bio *bio, int error);
The make_request_fn should return 0, regardless of whether the I/O is successful or not; errors are indicated by providing a nonzero value for the @error parameter in bio_endio.
ii) Redirecting the request to another device:
Some block drivers, such as those implementing volume manages and software RAID arrays, need to redirect the request to another device that handles the actual I/O. But note that if the make_request_fn returns a nonzero value, the bio is submitted again.
* Either way, you must tell the block subsystem that your driver is using a custom make_request_fn function. To do so, you must first allocate a request queue with:
	struct request_queue *blk_alloc_queue(gfp_t flags); // This differs from blk_init_queue() in that it doesn't actually set up the queue to hold requests.
Once you have a queue, pass it and your make_request_fn to blk_queue_make_request:
	void blk_queue_make_request(struct request_queue *q, make_request_fn *func);
* Actually all queue have a make_request_fn function; the default verison, generic_make_request(), handles the incorporation of the bio into a request structure. By providing a make_request_fn of its own, a driver just overrides the default one.

• Command pre-preparation:
The block layer provides a mechanism for drivers to examine and preprocess requests before they are returned from blk_peek_request(). This mechanism allows drivers to set up the actual drive commands ahead of time, decide whether the request can be handled at all, or perform other sorts of housekeeping.
	typedef int (prep_rq_fn)(struct request_queue *q, struct request *rq);
The struct request includes a field called @unsigned char __cmd[BLK_MAX_CDB], which is an array of BLK_MAX_CDB bytes. This array may be used by the preparation function to store the actual hardware command (or any other useful information). This function should return one of the following values:
	BLKPREP_OK: The request can be handed to your driver's request function.
	BLKPREP_KILL: This request cannot be completed, it is failed with an error code.
	BLKPREP_DEFER: This request cannot be completed at this time. It stays at the front of the queue but is not handed to the request function.
* The preparation function is called by blk_peek_request() immediately before the request is returned to your driver. If this function returns BLKPREP_DEFER, the return value from blk_peek_request() is NULL. This can be useful if, for example, your device has reached the maximum number of requests it can have outstanding. (See source code of block/blk-core.c for more details.)
