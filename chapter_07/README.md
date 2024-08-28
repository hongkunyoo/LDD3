1. Measuring time lapses:
Every time a timer interrupt occurs, the value of an internal kernel counter is incremented. The counter is initialized to 0 at system boot. The counter is a 64-bit variable (even on 32-bit architecture) and is called jiffies_64. However, driver writers normally access the "jiffies" variable, which is an unsigned long. (Using jiffies is usually preferred because it is faster, and access to the 64-bit jiffies_64 value are not necessarily atomic on all architectures.)
• Sometimes you need to exchange time representations with user space programs that tend to represents time values with struct timeval and struct timespe. Kernel provides 4 helper functions to convert time values expressed as jiffies to and from those structures: (in <linux/time.h>)
	unsigned long timespec_to_jiffies(struct timespec *value);
	void jiffies_to_timespec(unsigned long jiffies, struct timespec *value);
	unsigned long timeval_to_jiffies(struct timeval *value);
	void jiffies_to_timeval(unsigned long jiffies, struct timeval *value);
• Since access to jiffies_64 is not atomic for 32-bit processors, and you may need to use locks (like seqlock) to fetch the jiffies_64. Kernel provides a specific helper function to do the thing:
	u64 get_jiffies_64(void);

• Processor-specific registers:
Most modern processors include a counter register which is steadily incremented once at each clock cycle. On some platforms supporting SMP, for example, the kernel depends on such a counter to be synchronized across processors. 
The most renowed counter register is TSC (timestamp counter), introduced in x86 processors. After including <asm/msr.h>, you can use these macros:
	rdtsc(low32, high32);
	rdtscl(low32);
	rdtscll(var64);
* Kernel also provides architecture-independent function that you can use instead of rdstc: (defined in <asm/timex.h>, included by <linux/timex.h>)
	cycles_t get_cycles(void);

• Knowing the current time:
Note that: i) jiffies is always sufficient for measuring time intervals; ii) if you need very precise measurements for short time lapses, processor-specific registers come to rescue, although they bring in serious portability issues.
* But there's a kernel function that turns a wall-time into number of seconds since epoch (1970-01-01):
	unsigned long mktime(unsigned int year, unsigned int mon, unsigned int day, unsigned int hour, unsigned int min, unsigned int sec);
* void do_gettimeofday(struct timeval *tv);
Fill in the timeval structure with seconds and milliseconds since the Epoch, with the best resolution the hardware can offer.
* struct timespec current_kernel_time(void);
The current time (wall time) is also available (though with jiffy granularity) from the xtime variable - a struct timespec value. (xtime stores the number of seconds since EPOCH.)


2. Delaying execution:
i)  Long delays:
• Busy waiting: 
	while (time_before(jiffies, j1))
		cpu_relax();
The loop is guaranteed to work because jiffies is declared as volatile by the kernel headers and therefore is fetched from memory when being accessed. Busy waiting severely degrades system performance. If you didn't configure the kernel for preemptive operation, the busy loop completely locks the processor for the duration of the delay. The scheduler never preempts a process that is running in kernel space. Still worse, if interrupts happen to be disabled when you enter the loop, jiffies won't be updated, and the while loop remains true forever. (And for this situation, running a preemptive kernel won't help either.)
• Yielding the processor:
	while (time_before(jiffies, j1))
		schedule();
This solution isn't optimal. The current process does nothing but release the CPU, but it remains in the run queue. If it is the only runnable process, the idle task never runs, so the machine load (average number of running processes) is at least 1. Another problem is that: once a process releases the processor using schedule(), there's no guarantee that it will again get the processor back anytime soon.
• Timeouts:
The best way to implement a delay, is usually to ask the kernel to do it for you. If the driver uses a wait queue to wait for some other event, but you also want to ensure that it runs within a certain period of time, use:
	long wait_event_timeout(wait_queue_head_t q, condition, long timeout);
	long wait_event_interruptible_timeout(wait_queue_head_t q, condition, long timeout);
The timeout value is the number of jiffies to wait, not the absolute time value. If the timeout expires, the functions return 0; if the process is awakened by another event, it returns the remaining delay expressed in jiffies. (The return value is never negative, even if the delay is greater than expected because of system load.) Since the process is not in the run queue, you see no difference in behavior whether the code is run in a preemptive kernel or not.
* long schedule_timeout(signed long timeout);
Put the task to sleep until at least the specified time has elapsed. The return value is 0 unless the function returns before the given timeout has elapsed (in response to a signal). Note that schedule_timeout() requires the caller to set the current process state first:
	set_current_state(TASK_INTERRUPTIBLE);  /* so that the scheduler won't run the current process again until the timeout places it back in TASK_RUNNING state */
	schedule_timeout(delay);
If you forget to change the state of the current process, a call to schedule_timeout() behaves like a call to schedule().

ii) Short delays:
	#include <linux/delay.h>
	void ndelay(unsigned long nsecs);
	void udelay(unsigned long usecs);
	void mdelay(unsigned long msecs);
Note that: to avoid integer overflows, udelay() and ndelay() impose an upper bound in the value passed to them. If the module fails to load and displays an unresolved symbol, __bad_udelay, it means you call udelay with an too large argument. It's important to remember that these three delay functions are busy waiting.

• Another way of achieving millisecond (and longer) delays that doesn't involve busy waiting:
	void msleep(unsigned int millisecs);
	unsigned long msleep_interruptible(unsigned int millisecs);
	void ssleep(unsigned int seconds);
The first two puts the calling process to sleep for the given number of milliseconds. The return value of msleep_interruptible() is the number of milliseconds remaining (since early wake up.) ssleep() puts the process into an uninterruptible sleep for the given number of seconds.
• In general, if you can tolerate longer delays than requested, you should use schedule_timeout(), msleep(), or ssleep().


3. Kernel timers:
Whenever you need to schedule an action to happen later, without blocking the current process until that time arrives, kernel timers are the tool for you.
* A kernel timer is a data structure that instructs the kernel to execute a user-defined function with a user-defined argument at a user-defined time. Details are in <linux/timer.h> and kernel/timer.c. When a timer runs, the process that scheduled it could be asleep, executing on a different processor, or quite possibly has exited. This behavior resembles hardware interrupt; in fact, kernel timers are run as the result of a "software interrupt". So kernel timer functions must be atomic, and there're some additional issues. When you are outside of process context (i.e. in interrupt context), you must observe the following rules:
• No access to user space is allowed, because there's no process context.
• The current pointer is not meaningful and cannot be used in interrupt context.
• No sleeping or scheduling may be performed. Atomic code may not call schedule() or any form of wait_event(), nor may it call any other function that could sleep, like calling kmalloc(..., GFP_KERNEL), using semaphores, etc.
• Kernel code can tell if it is running in interrupt context by calling: in_interrupt(), which returns nonzero if the processor is currently running in interrupt context, either hardware interrupt or software interrupt.
in_atomic() returns nonzero whenever scheduling is not allowed, this includes hardware and software interrupt contexts as well as any time a spinlock is held. In the latter case, current may be valid, but access to user-space is forbidden, since it can cause scheduling to happen.

* One other important feature of kernel timers is that a task can reregister itself to run again at a later time. Although rescheduling the same task over and over might appear to be pointless, but it is sometimes useful. For example, it can be used to implement the polling of devices.
* It's also worth knowing that in an SMP system, the timer function is executed by the same CPU that registered it, to achieve better cache locality whenever possible. 
* Kernel timer is also a potential source of race conditions, even on uniprocessor systems. So any data structures accessed by the timer function should be protected from concurrent access, either by being atomic types or using spinlocks. (Note that don't use semaphores, because semaphore can sleep, but kernel timer functions is a kind of software interrupt handler, so should be in atomic context.)

• The kernel timer API:
	struct timer_list {
		/* ... */
		unsigned long expires;  /* in jiffies */
		void (*function)(unsigned long);
		unsigned long data;
	};
	void init_timer(struct timer_list *timer);
	struct timer_list TIMER_INITIALIZER(_function, _expires, _data);
	void add_timer(struct timer_list *timer);  /* activates the timer to run on the current CPU */
	int del_timer(struct timer_list *timer);
	int mod_timer(struct time_list *timer, unsigned long expires);  /* Changes the expiration time of an already scheduled (active) timer structure; if the timer is inactive
                                                                     if will be activated. */
	int del_timer_sync(struct time_list *timer);  /* it guarantees that when it returns, the timer function is not running on any CPU. It is used to avoid race conditions on
                                                   SMP machines, and the same as del_timer() on UP machines. Note that this function can sleep if it is called from a 
                                                   nonatomic context, but busy waits in other situations. If the timer function reregisters itself, the caller must first 
                                                   ensure that this reregistration will not happen; this is usually accomplished by setting a "shutting down" flag, which is 
                                                   checked by the timer function. */
	int timer_pending(const struct timer_list *timer); /* Return a boolean whether the timer is already registered (scheduled) to run */
• Implementation of kernel timers:
i)   Kernel timer is based on a per-CPU data structure. struct timer_list includes a pointer to that data structure in its base field. If base is NULL, then the timer is not scheduled to run; otherwise, the pointer tells which data structure (and which CPU) runs it.
ii)  Whenever kernel code registers a timer (add_timer or mod_timer), the operation is eventually performed by internal_add_timer, which adds the new timer to a doubly-linked list of timers within a "cascading table" associated to the current CPU.
iii) Cascading table works like this: if the timer expires in the next 0 ~ 255 jiffies, it's added to one of the 256 lists devoted to short-range timers using the LSB of expires field. If it expires farther in the future, but before 16384 jiffies, it's added to one of 64 jiffies based on 9 ~ 14 bit of the jiffies field. For timers expiring even farther, the same trick is used for bits 15 ~ 20, 21 ~ 26, 27 ~ 31. Timers with expires in the past are scheduled to run at the next timer tick.
iv)  If jiffies is currently a multiple of 256, the function also rehashes one of the next level lists of timers into the 256 short-term lists, possibly cascading one or more of the other levels as well, according to the bit representation of jiffies.
• Also note that a kernel timer is far from perfect, it suffers from jitter and other artifacts induced by hardware interrupts. You may also need to resort to a real-time kernel extention.


4. Tasklets:
A tasklet, just like a kernel timer, is executed (in atomic mode) in the context of a "software interrupt", a kernel mechanism that executes asynchronous tasks with hardware interrupts enabled. Tasklets resemble kernel timers in some ways. They always run on the same CPU that schedules them, and they receive an unsigned long argument. Unlike kernel timers, however, you cannot execute the function at a specific time, the function executes at a later time chosen by the kernel.
	#include <linux/interruot.h>
	struct tasklet_struct {
		/* ... */
		void (*func)(unsigned long);
		unsigned long data;
	};
	void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long);
	DECLARE_TASKLET(name, func, data);
	DECLARE_TASKLET_DISABLED(name, func, data);
• A tasklet can be disabled and re-enabled later, it won't be executed until it is enabled as many times as it has been disabled.
• Like kernel timers, a tasklet can re-register itself.
• A tasklet can be scheduled to execute at normal priority or high priority.
• Tasklets may be run immediately if the system is not under heavy load, but never later than the next timer tick.
• The same tasklet never runs simultaneously on more than one processor, but a tasklet can be concurrent with other tasklets.
• A tasklet always runs on the same CPU that schedules it.

The kernel provides a set of ksoftirq kernel threads, one per CPU, to run "software interrupt" handlers.
	void tasklet_disable(struct tasklet_struct *t); /* The tasklet may still be scheduled with tasklet_schedule(), but its execution is deferred until it is enabled again.
                                                       After calling tasklet_disable(), you can be sure that the tasklet isn't running anywhere in the system. */
	void tasklet_disable_nosync(struct tasklet_struct *t); /* same as above, but don't wait for any current-running function to exit. */
	void tasklet_enable(struct tasklet_struct *t); /* must be balanced with time of disable */
	void tasklet_schedule(struct tasklet_struct *t); /* Schedule a tasklet for execution. If it is scheduled again before it has a chance to run, it runs only once. However,
                                                        if it is scheduled while it runs, it runs again after it completes. This allows a tasklet to reschedule itself. */
	void tasklet_hi_schedule(struct tasklet_struct *t); /* same as above, but schedule the tasklet with higher priority */
	void tasklet_kill(struct tasklet_struct *t); /* This ensures that the tasklet is not scheduled to run again, always called when a device is being closed or the module 
                                                  removed. If the tasklet is already scheduled to run, the function waits until it completes. If the tasklet reschedules
                                                  itself, you must prevent it from rescheduling itself before calling tasklet_kill(), as with del_timer_sync(). */
• Tasklets are simply linked in a linked list, because tasklets have none of the sorting requirements of kernel timers.


5. Workqueues:
Workqueues, similar to tasklets, allow kernel code to request a function to be called at some time later. But:
• Tasklets run in software interrupt context with the result that all tasklets code must be atomic. However, workqueue functions run in the context of a special kernel process, so workqueue functions can sleep.
• Tasklets always run on the same processor from which they are scheduled. So as workqueues.
• Kernel code can request the execution of workqueue functions be delayed for an explicit interval.
(Tasklets execute quickly, for a short period of time, and in atomic mode; while workqueue functions may have higher latency but need not to be atomic.)

A workqueue (a set of work threads) must be created before use:
	struct workqueue_struct *create_workqueue(const char *name);
	struct workqueue_struct *create_singlethread_workqueue(const char *name);
Each workqueue has one or more processes (kernel threads), which run functions submitted to the queue. create_workqueue() will create one kernel thread for each processor; if a single worker thread suffice, using create_singlethread_workqueue.
	DECLARE_WORK(name, void (*function)(void *), void *data); /* define a work */
	INIT_WORK(struct work_struct *work, void (*function)(void *), void *data);  /* initializing the work_struct structure */
	PREPARE_WORK(struct work_struct *work, void (*function)(void *), void *data);  /* does the same job, but it doesn't initialize the pointers used to link the work_struct 
                                                                                     structure into the workqueue */
	int queue_work(struct workqueue_struct *queue, struct work_struct *work); /* adds the work to the given queue */
	int queue_delayed_work(struct workqueue_struct *queue, struct work_struct *work, unsigned long delay); /* same as above, but the actual work isn't performed until at 
                                                                                                             least delay jiffies have passed. */
The return value of the above two functions is 0 if the work is successfully added to the queue. A nonzero value means that the work_struct is already waiting in the queue, and not added a second time.
• At some time in the future, the work function will be called with the given data value. The work function may sleep because it is in process context (kernel threads), but it cannot access user space because it is inside a kernel thread.
	int cancel_delayed_work(struct work_struct *work); /* The return value is nonzero if the entry was canceled before it began execution. If returns 0, then the entry may
                                                         have already been running on a different processor, and might still be running after a call to cancel_delayed_work()
                                                       */
To be absolutely sure that the work function isn't running anywhere in the system, you must follow that call with a call:
	void flush_workqueue(struct workqueue_struct *queue); /* after it returns, no work function submitted prior to the call is running anywhere in the system. */

• The shared queue:
Device driver, in many cases, doesn't need its own workqueue. The kernel provides the default events workqueue.
	schedule_work(struct work_struct *work);  /* equivalent to queue_work() */
	schdeule_dalayed_work(struct work_struct *work, unsigned long delay);  /* equivalent to queue_delayed_work() */
	flush_scheduled_work(void);  /* equivalent to flush_workqueue() */
If you want to cancel a work entry submitted to the shared queue, you can still use cancel_delayed_work().
