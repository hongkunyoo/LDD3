1. Split of mechanism and policy:
- Mechanism: WHAT capabilities are to be provided.
- Policy: HOW those capabilities can be used.

The driver should deal with making the hardware available, leaving all the issues about how to use the hardware to the application. 

2. Splitting the kernel:
    1. Process management
    2. memory management
    3. filesystems
    4. device control
    5. networking.

3. Three types of devices:
    1. Character devices:
    The only relevant difference between a char device and a regular file is that: you can always move back and forth in the regular file, whereas most char devices can only be accessed sequentially.
    2. Block devices:
    3. Network interfaces:
    A network driver knows nothing about individual connections, it only handles packets. The unix way to provide access to interfaces is still by assigning a unique name to them (like eth0), but the name doesn't have a corresponding entry in the filesystem. Instead of read and write, the kernel calls functions related to packet transmission.

4. Security issues:
Any security check in the system is enforced by kernel code.

Documentation/Changes --> Require tools/versions needed to build the kernel.
COPYING --> GPL licence file.
