# Synchronization & Waiting Queue

## Overview

Kernel synchronization mechanisms prevent race conditions when multiple execution contexts access shared resources. This chapter covers locks, semaphores, and waiting queues.

## Race Conditions

Occur when multiple threads access shared data concurrently without proper synchronization.

```c
// Race condition example
static int counter = 0;

// Thread 1 and Thread 2 both execute:
counter++;  // Read, increment, write - not atomic!
```

## Atomic Operations

### Atomic Integers

```c
#include <linux/atomic.h>

atomic_t counter = ATOMIC_INIT(0);

// Operations
atomic_inc(&counter);           // counter++
atomic_dec(&counter);           // counter--
atomic_add(5, &counter);        // counter += 5
atomic_sub(3, &counter);        // counter -= 3

// Read
int value = atomic_read(&counter);

// Set
atomic_set(&counter, 10);

// Test and modify
if (atomic_dec_and_test(&counter))
    printk("Counter reached zero\n");

// Compare and swap
int old = 5;
atomic_cmpxchg(&counter, old, 10);  // If counter==5, set to 10
```

### Atomic Bitops

```c
unsigned long flags = 0;

set_bit(0, &flags);           // Set bit 0
clear_bit(0, &flags);         // Clear bit 0
change_bit(0, &flags);        // Toggle bit 0

if (test_bit(0, &flags))
    printk("Bit 0 is set\n");

// Test and modify
if (test_and_set_bit(0, &flags))
    printk("Bit was already set\n");
```

## Spinlocks

Busy-wait locks for short critical sections.

### Basic Spinlock

```c
#include <linux/spinlock.h>

static DEFINE_SPINLOCK(my_lock);

// Lock
spin_lock(&my_lock);
// Critical section
spin_unlock(&my_lock);
```

### Spinlock with IRQ

```c
unsigned long flags;

// Disable interrupts and acquire lock
spin_lock_irqsave(&my_lock, flags);
// Critical section
spin_unlock_irqrestore(&my_lock, flags);

// Alternative (if already in interrupt context)
spin_lock_irq(&my_lock);
// Critical section
spin_unlock_irq(&my_lock);
```

### Spinlock Variants

```c
// Bottom half disable
spin_lock_bh(&my_lock);
spin_unlock_bh(&my_lock);

// Try lock (non-blocking)
if (spin_trylock(&my_lock)) {
    // Got lock
    spin_unlock(&my_lock);
} else {
    // Lock busy
}
```

### Read-Write Spinlock

```c
static DEFINE_RWLOCK(my_rwlock);

// Multiple readers
read_lock(&my_rwlock);
// Read data
read_unlock(&my_rwlock);

// Single writer
write_lock(&my_rwlock);
// Modify data
write_unlock(&my_rwlock);

// With IRQ disable
unsigned long flags;
read_lock_irqsave(&my_rwlock, flags);
read_unlock_irqrestore(&my_rwlock, flags);
```

### Example

```c
struct my_device {
    spinlock_t lock;
    int counter;
    char buffer[256];
};

static struct my_device dev;

static void increment_counter(void)
{
    unsigned long flags;
    
    spin_lock_irqsave(&dev.lock, flags);
    dev.counter++;
    spin_unlock_irqrestore(&dev.lock, flags);
}
```

## Mutexes

Sleep locks for longer critical sections (process context only).

```c
#include <linux/mutex.h>

static DEFINE_MUTEX(my_mutex);

// Lock
mutex_lock(&my_mutex);
// Critical section (can sleep)
mutex_unlock(&my_mutex);

// Interruptible lock
if (mutex_lock_interruptible(&my_mutex))
    return -ERESTARTSYS;
// Critical section
mutex_unlock(&my_mutex);

// Try lock
if (mutex_trylock(&my_mutex)) {
    // Got lock
    mutex_unlock(&my_mutex);
}

// Check if locked
if (mutex_is_locked(&my_mutex))
    printk("Mutex is locked\n");
```

### Example

```c
struct my_device {
    struct mutex lock;
    char buffer[1024];
    size_t size;
};

static ssize_t device_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    struct my_device *dev = filp->private_data;
    size_t to_read;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    to_read = min(count, dev->size);
    
    if (copy_to_user(buf, dev->buffer, to_read)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    
    mutex_unlock(&dev->lock);
    return to_read;
}
```

## Semaphores

Counting locks allowing multiple holders.

```c
#include <linux/semaphore.h>

static DEFINE_SEMAPHORE(my_sem);

// Down (acquire)
down(&my_sem);
// Critical section
up(&my_sem);

// Interruptible
if (down_interruptible(&my_sem))
    return -ERESTARTSYS;
up(&my_sem);

// Try down
if (down_trylock(&my_sem) == 0) {
    // Got semaphore
    up(&my_sem);
}

// Initialize with count
struct semaphore sem;
sema_init(&sem, 3);  // Allow 3 concurrent holders
```

## Completion

Synchronization for waiting on events.

```c
#include <linux/completion.h>

static DECLARE_COMPLETION(my_completion);

// Waiting thread
wait_for_completion(&my_completion);

// Signaling thread
complete(&my_completion);

// Wake all waiters
complete_all(&my_completion);

// Timeout
unsigned long timeout = msecs_to_jiffies(5000);
if (wait_for_completion_timeout(&my_completion, timeout) == 0)
    printk("Timeout\n");

// Interruptible
if (wait_for_completion_interruptible(&my_completion))
    return -ERESTARTSYS;

// Reinitialize
reinit_completion(&my_completion);
```

### Example

```c
struct my_device {
    struct completion data_ready;
    char buffer[256];
};

// Producer
static void produce_data(struct my_device *dev)
{
    // Generate data
    strcpy(dev->buffer, "Data ready");
    
    // Signal completion
    complete(&dev->data_ready);
}

// Consumer
static void consume_data(struct my_device *dev)
{
    // Wait for data
    wait_for_completion(&dev->data_ready);
    
    // Process data
    printk("Received: %s\n", dev->buffer);
}
```

## Wait Queues

Sleep until condition becomes true.

### Declaration

```c
#include <linux/wait.h>

static DECLARE_WAIT_QUEUE_HEAD(my_wait_queue);

// Or dynamic
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
```

### Waiting

```c
// Wait until condition is true
wait_event(my_wait_queue, condition);

// Interruptible
if (wait_event_interruptible(my_wait_queue, condition))
    return -ERESTARTSYS;

// With timeout
unsigned long timeout = msecs_to_jiffies(5000);
int ret = wait_event_timeout(my_wait_queue, condition, timeout);
if (ret == 0)
    printk("Timeout\n");

// Interruptible with timeout
ret = wait_event_interruptible_timeout(my_wait_queue, condition, timeout);
```

### Waking

```c
// Wake one waiter
wake_up(&my_wait_queue);

// Wake all waiters
wake_up_all(&my_wait_queue);

// Wake interruptible waiters
wake_up_interruptible(&my_wait_queue);
wake_up_interruptible_all(&my_wait_queue);
```

### Example: Blocking Read

```c
struct my_device {
    wait_queue_head_t read_queue;
    char buffer[256];
    int data_available;
    spinlock_t lock;
};

static ssize_t device_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    struct my_device *dev = filp->private_data;
    unsigned long flags;
    
    // Wait for data
    if (wait_event_interruptible(dev->read_queue, dev->data_available))
        return -ERESTARTSYS;
    
    spin_lock_irqsave(&dev->lock, flags);
    
    if (copy_to_user(buf, dev->buffer, count)) {
        spin_unlock_irqrestore(&dev->lock, flags);
        return -EFAULT;
    }
    
    dev->data_available = 0;
    spin_unlock_irqrestore(&dev->lock, flags);
    
    return count;
}

// Interrupt handler
static irqreturn_t device_irq(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    unsigned long flags;
    
    spin_lock_irqsave(&dev->lock, flags);
    
    // Read data from hardware
    // ...
    
    dev->data_available = 1;
    spin_unlock_irqrestore(&dev->lock, flags);
    
    // Wake up readers
    wake_up_interruptible(&dev->read_queue);
    
    return IRQ_HANDLED;
}
```

## RCU (Read-Copy-Update)

Lock-free synchronization for read-mostly data.

```c
#include <linux/rcupdate.h>

struct my_data {
    int value;
    struct rcu_head rcu;
};

static struct my_data *global_ptr;

// Reader
rcu_read_lock();
struct my_data *p = rcu_dereference(global_ptr);
if (p)
    printk("Value: %d\n", p->value);
rcu_read_unlock();

// Writer
struct my_data *new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
new_data->value = 42;

struct my_data *old_data = global_ptr;
rcu_assign_pointer(global_ptr, new_data);

// Free old data after grace period
synchronize_rcu();
kfree(old_data);
```

## Choosing Synchronization

| Mechanism | Context | Can Sleep | Use Case |
|-----------|---------|-----------|----------|
| Atomic ops | Any | No | Simple counters |
| Spinlock | Any | No | Short critical sections |
| Mutex | Process | Yes | Long critical sections |
| Semaphore | Process | Yes | Resource counting |
| Completion | Process | Yes | Event notification |
| Wait queue | Process | Yes | Condition waiting |
| RCU | Any | No | Read-mostly data |

## Complete Example

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/wait.h>
#include <linux/sched.h>
#include <linux/uaccess.h>

#define BUFFER_SIZE 256

struct sync_device {
    struct cdev cdev;
    dev_t devt;
    char buffer[BUFFER_SIZE];
    size_t size;
    struct mutex lock;
    wait_queue_head_t read_queue;
    wait_queue_head_t write_queue;
};

static struct sync_device dev;

static int device_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &dev;
    return 0;
}

static ssize_t device_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    struct sync_device *dev = filp->private_data;
    size_t to_read;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    // Wait for data
    while (dev->size == 0) {
        mutex_unlock(&dev->lock);
        
        if (filp->f_flags & O_NONBLOCK)
            return -EAGAIN;
        
        if (wait_event_interruptible(dev->read_queue, dev->size > 0))
            return -ERESTARTSYS;
        
        if (mutex_lock_interruptible(&dev->lock))
            return -ERESTARTSYS;
    }
    
    to_read = min(count, dev->size);
    
    if (copy_to_user(buf, dev->buffer, to_read)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    
    // Shift remaining data
    memmove(dev->buffer, dev->buffer + to_read, dev->size - to_read);
    dev->size -= to_read;
    
    mutex_unlock(&dev->lock);
    
    // Wake up writers
    wake_up_interruptible(&dev->write_queue);
    
    return to_read;
}

static ssize_t device_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct sync_device *dev = filp->private_data;
    size_t to_write;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    // Wait for space
    while (dev->size >= BUFFER_SIZE) {
        mutex_unlock(&dev->lock);
        
        if (filp->f_flags & O_NONBLOCK)
            return -EAGAIN;
        
        if (wait_event_interruptible(dev->write_queue,
                                     dev->size < BUFFER_SIZE))
            return -ERESTARTSYS;
        
        if (mutex_lock_interruptible(&dev->lock))
            return -ERESTARTSYS;
    }
    
    to_write = min(count, (size_t)(BUFFER_SIZE - dev->size));
    
    if (copy_from_user(dev->buffer + dev->size, buf, to_write)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    
    dev->size += to_write;
    
    mutex_unlock(&dev->lock);
    
    // Wake up readers
    wake_up_interruptible(&dev->read_queue);
    
    return to_write;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = device_open,
    .read = device_read,
    .write = device_write,
};

static int __init sync_init(void)
{
    int ret;
    
    mutex_init(&dev.lock);
    init_waitqueue_head(&dev.read_queue);
    init_waitqueue_head(&dev.write_queue);
    
    ret = alloc_chrdev_region(&dev.devt, 0, 1, "syncdev");
    if (ret < 0)
        return ret;
    
    cdev_init(&dev.cdev, &fops);
    ret = cdev_add(&dev.cdev, dev.devt, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev.devt, 1);
        return ret;
    }
    
    printk(KERN_INFO "Sync device initialized\n");
    return 0;
}

static void __exit sync_exit(void)
{
    cdev_del(&dev.cdev);
    unregister_chrdev_region(dev.devt, 1);
    printk(KERN_INFO "Sync device removed\n");
}

module_init(sync_init);
module_exit(sync_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Synchronization example");
```

## Best Practices

1. **Use appropriate mechanism** for the context
2. **Keep critical sections short**
3. **Avoid nested locks** (deadlock risk)
4. **Always unlock** in error paths
5. **Use interruptible waits** in process context
6. **Document locking order** for multiple locks
7. **Test with lockdep** enabled

## Deadlock Prevention

- Acquire locks in consistent order
- Use try-lock variants
- Avoid holding locks across blocking operations
- Use lock validator (CONFIG_PROVE_LOCKING)

## Next Steps

Proceed to [Kernel Memory Management](10-kernel-memory-management.md) to learn about memory allocation in the kernel.
