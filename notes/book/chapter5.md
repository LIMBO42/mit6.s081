#### 驱动和中断

A driver is the code in an operating system that manages a particular device: it configures the device hardware, tells the device to perform operations, handles the resulting interrupts, and interacts with processes that may be waiting for I/O from the device.Driver code can be tricky because a driver executes concurrently with the device that it manages.

Many device drivers execute code in two contexts: a top half that runs in a process’s kernel thread, and a bottom half that executes at interrupt time.

The top half is called via system calls such as read and write that want the device to perform I/O. This code may ask the hardware to start an operation (e.g., ask the disk to read a block); then the code waits for the operation to complete. Eventually the device completes the operation and raises an interrupt. The driver’s interrupt handler, acting as the bottom half, figures out what operation has completed, wakes up a waiting process if appropriate, and tells the hardware to start work on any waiting next operation.

