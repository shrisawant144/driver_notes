# Pseudo Char Device Driver

## Overview

A pseudo character device driver is a software-only driver that doesn't interact with actual hardware. It's useful for learning driver development concepts and creating virtual devices for testing and inter-process communication.

## Use Cases

- Learning and testing driver concepts
- Virtual devices for IPC
- Device simulation
- Data buffering and processing
- Debugging and development tools

## Simple Pseudo Device

### Basic Implementation

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "pseudodev"
#define CLASS_NAME "pseudo"
#define BUFFER_SIZE 1024

static dev_t dev_num;
static struct cdev pseudo_cdev;
static struct class *pseudo_class;
static struct device *pseudo_device;

static char *device_buffer;
static size_t buffer_ptr = 0;

static int pseudo_open(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Pseudo device opened\n");
    return 0;
}

static int pseudo_release(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Pseudo device closed\n");
    return 0;
}

static ssize_t pseudo_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    size_t to_read;
    
    if (*f_pos >= buffer_ptr)
        return 0;
    
    to_read = min(count, buffer_ptr - (size_t)*f_pos);
    
    if (copy_to_user(buf, device_buffer + *f_pos, to_read))
        return -EFAULT;
    
    *f_pos += to_read;
    printk(KERN_INFO "Read %zu bytes\n", to_read);
    
    return to_read;
}

static ssize_t pseudo_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    size_t to_write;
    
    to_write = min(count, (size_t)(BUFFER_SIZE - buffer_ptr));
    
    if (to_write == 0)
        return -ENOSPC;
    
    if (copy_from_user(device_buffer + buffer_ptr, buf, to_write))
        return -EFAULT;
    
    buffer_ptr += to_write;
    printk(KERN_INFO "Wrote %zu bytes\n", to_write);
    
    return to_write;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = pseudo_open,
    .release = pseudo_release,
    .read = pseudo_read,
    .write = pseudo_write,
};

static int __init pseudo_init(void)
{
    int ret;
    
    // Allocate buffer
    device_buffer = kmalloc(BUFFER_SIZE, GFP_KERNEL);
    if (!device_buffer)
        return -ENOMEM;
    
    memset(device_buffer, 0, BUFFER_SIZE);
    
    // Allocate device number
    ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        kfree(device_buffer);
        return ret;
    }
    
    // Initialize cdev
    cdev_init(&pseudo_cdev, &fops);
    pseudo_cdev.owner = THIS_MODULE;
    
    ret = cdev_add(&pseudo_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        kfree(device_buffer);
        return ret;
    }
    
    // Create class
    pseudo_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(pseudo_class)) {
        cdev_del(&pseudo_cdev);
        unregister_chrdev_region(dev_num, 1);
        kfree(device_buffer);
        return PTR_ERR(pseudo_class);
    }
    
    // Create device
    pseudo_device = device_create(pseudo_class, NULL, dev_num, NULL, DEVICE_NAME);
    if (IS_ERR(pseudo_device)) {
        class_destroy(pseudo_class);
        cdev_del(&pseudo_cdev);
        unregister_chrdev_region(dev_num, 1);
        kfree(device_buffer);
        return PTR_ERR(pseudo_device);
    }
    
    printk(KERN_INFO "Pseudo device initialized\n");
    return 0;
}

static void __exit pseudo_exit(void)
{
    device_destroy(pseudo_class, dev_num);
    class_destroy(pseudo_class);
    cdev_del(&pseudo_cdev);
    unregister_chrdev_region(dev_num, 1);
    kfree(device_buffer);
    
    printk(KERN_INFO "Pseudo device removed\n");
}

module_init(pseudo_init);
module_exit(pseudo_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Simple pseudo character device driver");
MODULE_VERSION("1.0");
```

## Multiple Device Support

### Device Structure

```c
#define NUM_DEVICES 4

struct pseudo_device {
    struct cdev cdev;
    char *buffer;
    size_t size;
    size_t buffer_ptr;
    struct mutex lock;
};

static struct pseudo_device *devices;
static dev_t dev_num;
static struct class *pseudo_class;
```

### Implementation

```c
static int pseudo_open(struct inode *inode, struct file *filp)
{
    struct pseudo_device *dev;
    
    dev = container_of(inode->i_cdev, struct pseudo_device, cdev);
    filp->private_data = dev;
    
    printk(KERN_INFO "Device opened\n");
    return 0;
}

static ssize_t pseudo_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    struct pseudo_device *dev = filp->private_data;
    size_t to_read;
    int ret;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    if (*f_pos >= dev->buffer_ptr) {
        mutex_unlock(&dev->lock);
        return 0;
    }
    
    to_read = min(count, dev->buffer_ptr - (size_t)*f_pos);
    
    if (copy_to_user(buf, dev->buffer + *f_pos, to_read)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    
    *f_pos += to_read;
    mutex_unlock(&dev->lock);
    
    return to_read;
}

static ssize_t pseudo_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct pseudo_device *dev = filp->private_data;
    size_t to_write;
    
    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;
    
    to_write = min(count, dev->size - dev->buffer_ptr);
    
    if (to_write == 0) {
        mutex_unlock(&dev->lock);
        return -ENOSPC;
    }
    
    if (copy_from_user(dev->buffer + dev->buffer_ptr, buf, to_write)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    
    dev->buffer_ptr += to_write;
    mutex_unlock(&dev->lock);
    
    return to_write;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = pseudo_open,
    .release = pseudo_release,
    .read = pseudo_read,
    .write = pseudo_write,
};

static int __init pseudo_init(void)
{
    int i, ret;
    
    // Allocate device structures
    devices = kzalloc(NUM_DEVICES * sizeof(struct pseudo_device), GFP_KERNEL);
    if (!devices)
        return -ENOMEM;
    
    // Allocate device numbers
    ret = alloc_chrdev_region(&dev_num, 0, NUM_DEVICES, "pseudodev");
    if (ret < 0) {
        kfree(devices);
        return ret;
    }
    
    // Create class
    pseudo_class = class_create(THIS_MODULE, "pseudo");
    if (IS_ERR(pseudo_class)) {
        unregister_chrdev_region(dev_num, NUM_DEVICES);
        kfree(devices);
        return PTR_ERR(pseudo_class);
    }
    
    // Initialize each device
    for (i = 0; i < NUM_DEVICES; i++) {
        devices[i].size = 1024;
        devices[i].buffer = kmalloc(devices[i].size, GFP_KERNEL);
        if (!devices[i].buffer) {
            ret = -ENOMEM;
            goto fail;
        }
        
        mutex_init(&devices[i].lock);
        
        cdev_init(&devices[i].cdev, &fops);
        devices[i].cdev.owner = THIS_MODULE;
        
        ret = cdev_add(&devices[i].cdev, MKDEV(MAJOR(dev_num), i), 1);
        if (ret < 0)
            goto fail;
        
        device_create(pseudo_class, NULL, MKDEV(MAJOR(dev_num), i),
                     NULL, "pseudodev%d", i);
    }
    
    printk(KERN_INFO "Created %d pseudo devices\n", NUM_DEVICES);
    return 0;

fail:
    while (i--) {
        device_destroy(pseudo_class, MKDEV(MAJOR(dev_num), i));
        cdev_del(&devices[i].cdev);
        kfree(devices[i].buffer);
    }
    class_destroy(pseudo_class);
    unregister_chrdev_region(dev_num, NUM_DEVICES);
    kfree(devices);
    return ret;
}

static void __exit pseudo_exit(void)
{
    int i;
    
    for (i = 0; i < NUM_DEVICES; i++) {
        device_destroy(pseudo_class, MKDEV(MAJOR(dev_num), i));
        cdev_del(&devices[i].cdev);
        kfree(devices[i].buffer);
    }
    
    class_destroy(pseudo_class);
    unregister_chrdev_region(dev_num, NUM_DEVICES);
    kfree(devices);
    
    printk(KERN_INFO "Pseudo devices removed\n");
}

module_init(pseudo_init);
module_exit(pseudo_exit);

MODULE_LICENSE("GPL");
```

## FIFO Device (Circular Buffer)

```c
#define BUFFER_SIZE 1024

struct fifo_device {
    char buffer[BUFFER_SIZE];
    size_t read_ptr;
    size_t write_ptr;
    size_t count;
    struct mutex lock;
    wait_queue_head_t read_queue;
    wait_queue_head_t write_queue;
};

static struct fifo_device fifo_dev;

static ssize_t fifo_read(struct file *filp, char __user *buf,
                        size_t count, loff_t *f_pos)
{
    size_t to_read, chunk;
    
    if (mutex_lock_interruptible(&fifo_dev.lock))
        return -ERESTARTSYS;
    
    // Wait for data
    while (fifo_dev.count == 0) {
        mutex_unlock(&fifo_dev.lock);
        
        if (filp->f_flags & O_NONBLOCK)
            return -EAGAIN;
        
        if (wait_event_interruptible(fifo_dev.read_queue, fifo_dev.count > 0))
            return -ERESTARTSYS;
        
        if (mutex_lock_interruptible(&fifo_dev.lock))
            return -ERESTARTSYS;
    }
    
    to_read = min(count, fifo_dev.count);
    
    // Handle wrap-around
    chunk = min(to_read, (size_t)(BUFFER_SIZE - fifo_dev.read_ptr));
    
    if (copy_to_user(buf, fifo_dev.buffer + fifo_dev.read_ptr, chunk)) {
        mutex_unlock(&fifo_dev.lock);
        return -EFAULT;
    }
    
    if (chunk < to_read) {
        if (copy_to_user(buf + chunk, fifo_dev.buffer, to_read - chunk)) {
            mutex_unlock(&fifo_dev.lock);
            return -EFAULT;
        }
    }
    
    fifo_dev.read_ptr = (fifo_dev.read_ptr + to_read) % BUFFER_SIZE;
    fifo_dev.count -= to_read;
    
    mutex_unlock(&fifo_dev.lock);
    
    wake_up_interruptible(&fifo_dev.write_queue);
    
    return to_read;
}

static ssize_t fifo_write(struct file *filp, const char __user *buf,
                         size_t count, loff_t *f_pos)
{
    size_t to_write, chunk;
    
    if (mutex_lock_interruptible(&fifo_dev.lock))
        return -ERESTARTSYS;
    
    // Wait for space
    while (fifo_dev.count == BUFFER_SIZE) {
        mutex_unlock(&fifo_dev.lock);
        
        if (filp->f_flags & O_NONBLOCK)
            return -EAGAIN;
        
        if (wait_event_interruptible(fifo_dev.write_queue,
                                     fifo_dev.count < BUFFER_SIZE))
            return -ERESTARTSYS;
        
        if (mutex_lock_interruptible(&fifo_dev.lock))
            return -ERESTARTSYS;
    }
    
    to_write = min(count, (size_t)(BUFFER_SIZE - fifo_dev.count));
    
    // Handle wrap-around
    chunk = min(to_write, (size_t)(BUFFER_SIZE - fifo_dev.write_ptr));
    
    if (copy_from_user(fifo_dev.buffer + fifo_dev.write_ptr, buf, chunk)) {
        mutex_unlock(&fifo_dev.lock);
        return -EFAULT;
    }
    
    if (chunk < to_write) {
        if (copy_from_user(fifo_dev.buffer, buf + chunk, to_write - chunk)) {
            mutex_unlock(&fifo_dev.lock);
            return -EFAULT;
        }
    }
    
    fifo_dev.write_ptr = (fifo_dev.write_ptr + to_write) % BUFFER_SIZE;
    fifo_dev.count += to_write;
    
    mutex_unlock(&fifo_dev.lock);
    
    wake_up_interruptible(&fifo_dev.read_queue);
    
    return to_write;
}

static int __init fifo_init(void)
{
    mutex_init(&fifo_dev.lock);
    init_waitqueue_head(&fifo_dev.read_queue);
    init_waitqueue_head(&fifo_dev.write_queue);
    
    fifo_dev.read_ptr = 0;
    fifo_dev.write_ptr = 0;
    fifo_dev.count = 0;
    
    // ... rest of initialization
    
    return 0;
}
```

## IOCTL Support

```c
#include <linux/ioctl.h>

#define PSEUDO_IOC_MAGIC 'P'

#define PSEUDO_IOCRESET    _IO(PSEUDO_IOC_MAGIC, 0)
#define PSEUDO_IOCGSIZE    _IOR(PSEUDO_IOC_MAGIC, 1, int)
#define PSEUDO_IOCSSIZE    _IOW(PSEUDO_IOC_MAGIC, 2, int)
#define PSEUDO_IOCGCOUNT   _IOR(PSEUDO_IOC_MAGIC, 3, int)

static long pseudo_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct pseudo_device *dev = filp->private_data;
    int ret = 0;
    int value;
    
    if (_IOC_TYPE(cmd) != PSEUDO_IOC_MAGIC)
        return -ENOTTY;
    
    switch (cmd) {
    case PSEUDO_IOCRESET:
        if (mutex_lock_interruptible(&dev->lock))
            return -ERESTARTSYS;
        dev->buffer_ptr = 0;
        memset(dev->buffer, 0, dev->size);
        mutex_unlock(&dev->lock);
        printk(KERN_INFO "Buffer reset\n");
        break;
        
    case PSEUDO_IOCGSIZE:
        value = dev->size;
        if (copy_to_user((int __user *)arg, &value, sizeof(value)))
            return -EFAULT;
        break;
        
    case PSEUDO_IOCSSIZE:
        if (copy_from_user(&value, (int __user *)arg, sizeof(value)))
            return -EFAULT;
        
        if (value < 0 || value > 1024 * 1024)
            return -EINVAL;
        
        // Reallocate buffer
        if (mutex_lock_interruptible(&dev->lock))
            return -ERESTARTSYS;
        
        kfree(dev->buffer);
        dev->buffer = kmalloc(value, GFP_KERNEL);
        if (!dev->buffer) {
            mutex_unlock(&dev->lock);
            return -ENOMEM;
        }
        
        dev->size = value;
        dev->buffer_ptr = 0;
        mutex_unlock(&dev->lock);
        break;
        
    case PSEUDO_IOCGCOUNT:
        value = dev->buffer_ptr;
        if (copy_to_user((int __user *)arg, &value, sizeof(value)))
            return -EFAULT;
        break;
        
    default:
        return -ENOTTY;
    }
    
    return ret;
}
```

## Sysfs Attributes

```c
static ssize_t buffer_size_show(struct device *dev,
                               struct device_attribute *attr, char *buf)
{
    struct pseudo_device *pdev = dev_get_drvdata(dev);
    return sprintf(buf, "%zu\n", pdev->size);
}

static ssize_t buffer_count_show(struct device *dev,
                                 struct device_attribute *attr, char *buf)
{
    struct pseudo_device *pdev = dev_get_drvdata(dev);
    return sprintf(buf, "%zu\n", pdev->buffer_ptr);
}

static ssize_t buffer_reset_store(struct device *dev,
                                  struct device_attribute *attr,
                                  const char *buf, size_t count)
{
    struct pseudo_device *pdev = dev_get_drvdata(dev);
    
    if (mutex_lock_interruptible(&pdev->lock))
        return -ERESTARTSYS;
    
    pdev->buffer_ptr = 0;
    memset(pdev->buffer, 0, pdev->size);
    
    mutex_unlock(&pdev->lock);
    
    return count;
}

static DEVICE_ATTR_RO(buffer_size);
static DEVICE_ATTR_RO(buffer_count);
static DEVICE_ATTR_WO(buffer_reset);

static struct attribute *pseudo_attrs[] = {
    &dev_attr_buffer_size.attr,
    &dev_attr_buffer_count.attr,
    &dev_attr_buffer_reset.attr,
    NULL,
};

static struct attribute_group pseudo_attr_group = {
    .attrs = pseudo_attrs,
};

// In device creation
device_create(pseudo_class, NULL, dev_num, &devices[i], "pseudodev%d", i);
dev_set_drvdata(pseudo_device, &devices[i]);
sysfs_create_group(&pseudo_device->kobj, &pseudo_attr_group);
```

## Testing

### Test Script

```bash
#!/bin/bash

DEVICE="/dev/pseudodev0"

# Write to device
echo "Hello, Pseudo Device!" > $DEVICE

# Read from device
cat $DEVICE

# Check sysfs
cat /sys/class/pseudo/pseudodev0/buffer_size
cat /sys/class/pseudo/pseudodev0/buffer_count

# Reset buffer
echo 1 > /sys/class/pseudo/pseudodev0/buffer_reset

# Verify reset
cat /sys/class/pseudo/pseudodev0/buffer_count
```

### C Test Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/ioctl.h>

#define PSEUDO_IOC_MAGIC 'P'
#define PSEUDO_IOCRESET    _IO(PSEUDO_IOC_MAGIC, 0)
#define PSEUDO_IOCGSIZE    _IOR(PSEUDO_IOC_MAGIC, 1, int)
#define PSEUDO_IOCGCOUNT   _IOR(PSEUDO_IOC_MAGIC, 3, int)

int main()
{
    int fd;
    char write_buf[] = "Test data for pseudo device";
    char read_buf[256];
    int size, count;
    
    fd = open("/dev/pseudodev0", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }
    
    // Get buffer size
    ioctl(fd, PSEUDO_IOCGSIZE, &size);
    printf("Buffer size: %d\n", size);
    
    // Write data
    write(fd, write_buf, strlen(write_buf));
    
    // Get data count
    ioctl(fd, PSEUDO_IOCGCOUNT, &count);
    printf("Data count: %d\n", count);
    
    // Read data
    lseek(fd, 0, SEEK_SET);
    int n = read(fd, read_buf, sizeof(read_buf));
    read_buf[n] = '\0';
    printf("Read: %s\n", read_buf);
    
    // Reset buffer
    ioctl(fd, PSEUDO_IOCRESET);
    
    close(fd);
    return 0;
}
```

## Best Practices

1. **Use proper locking** for concurrent access
2. **Implement blocking I/O** with wait queues
3. **Support non-blocking mode** (O_NONBLOCK)
4. **Provide sysfs interface** for configuration
5. **Handle errors gracefully**
6. **Clean up resources** properly
7. **Test with multiple processes**

## Next Steps

Proceed to [Platform Device Drivers](06-platform-device-drivers.md) to learn about platform-specific device drivers.
