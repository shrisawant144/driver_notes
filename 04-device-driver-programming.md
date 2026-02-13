# Linux Device Driver Programming

## 🎯 Layman's Explanation

**What is a Device Driver?**
A driver is a **translator** between hardware (which speaks electrical signals) and software (which speaks code). 

**Real-World Analogy:**
You're in a foreign country and don't speak the language. You need a translator to communicate with locals. Similarly:
- **Hardware** = Foreign person (speaks voltage/signals)
- **Driver** = Translator
- **Your App** = You (speaks English/code)

**Why Do We Need Drivers?**
Without drivers, your app would need to know:
- How to send electrical signals to a printer
- The exact timing for USB communication
- Register addresses of a WiFi chip

Drivers hide this complexity. Your app just says "print this" and the driver handles the messy details.

**Types of Devices (Simple Explanation):**

1. **Character Devices** = Stream of data (like a water hose)
   - Examples: Keyboard (stream of keypresses), Serial port
   - You read/write one byte at a time

2. **Block Devices** = Chunks of data (like boxes on a shelf)
   - Examples: Hard disk, USB drive
   - You read/write in blocks (512 bytes, 4KB, etc.)

3. **Network Devices** = Special (packets, not files)
   - Examples: WiFi, Ethernet
   - No `/dev` file, uses network stack

**The Communication Flow:**
```
User App (e.g., cat /dev/mydevice)
    ↓
System Call (read())
    ↓
Kernel (finds the right driver)
    ↓
Driver (talks to hardware)
    ↓
Hardware (does the work)
    ↓
Data flows back up
```

**Major & Minor Numbers (Simple):**
Think of a company:
- **Major number** = Department (e.g., Sales dept = driver)
- **Minor number** = Employee in that dept (e.g., device instance)

Example: `/dev/sda1` and `/dev/sda2` share the same driver (major) but are different partitions (minor).

## Overview

Device drivers are specialized kernel modules that provide an interface between hardware devices and the kernel. This chapter covers the fundamentals of device driver development in Linux.

## Device Driver Types

### Character Devices
- Stream of bytes (sequential access)
- Examples: serial ports, keyboards, mice
- Access via `/dev/ttyS0`, `/dev/input/mouse0`

### Block Devices
- Random access in fixed-size blocks
- Examples: hard drives, USB drives, SD cards
- Access via `/dev/sda`, `/dev/mmcblk0`

### Network Devices
- Network interfaces
- Examples: Ethernet, WiFi
- Access via `eth0`, `wlan0` (no device files)

## Device Numbers

### Major and Minor Numbers

```
Major Number: Identifies the driver
Minor Number: Identifies the specific device
```

### Viewing Device Numbers

```bash
ls -l /dev/
crw-rw---- 1 root tty     4,  0 Feb 13 10:00 tty0
brw-rw---- 1 root disk    8,  0 Feb 13 10:00 sda
```

Format: `major, minor`

### Allocating Device Numbers

#### Static Allocation

```c
#include <linux/fs.h>

#define MAJOR_NUM 240
#define MINOR_NUM 0

int major = MAJOR_NUM;
dev_t dev = MKDEV(major, MINOR_NUM);

ret = register_chrdev_region(dev, 1, "mydevice");
if (ret < 0) {
    printk(KERN_ERR "Failed to register device number\n");
    return ret;
}
```

#### Dynamic Allocation

```c
dev_t dev;
int ret;

ret = alloc_chrdev_region(&dev, 0, 1, "mydevice");
if (ret < 0) {
    printk(KERN_ERR "Failed to allocate device number\n");
    return ret;
}

int major = MAJOR(dev);
int minor = MINOR(dev);
```

### Unregistering Device Numbers

```c
unregister_chrdev_region(dev, 1);
```

## File Operations

### file_operations Structure

```c
#include <linux/fs.h>

struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = device_open,
    .release = device_release,
    .read = device_read,
    .write = device_write,
    .llseek = device_llseek,
    .unlocked_ioctl = device_ioctl,
};
```

### Common Operations

```c
int (*open)(struct inode *, struct file *);
int (*release)(struct inode *, struct file *);
ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
loff_t (*llseek)(struct file *, loff_t, int);
long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
```

## Character Device Registration

### Using cdev Structure

```c
#include <linux/cdev.h>

struct cdev my_cdev;
dev_t dev;

// Allocate device number
alloc_chrdev_region(&dev, 0, 1, "mydevice");

// Initialize cdev
cdev_init(&my_cdev, &fops);
my_cdev.owner = THIS_MODULE;

// Add cdev to system
ret = cdev_add(&my_cdev, dev, 1);
if (ret < 0) {
    unregister_chrdev_region(dev, 1);
    return ret;
}
```

### Cleanup

```c
cdev_del(&my_cdev);
unregister_chrdev_region(dev, 1);
```

## Implementing File Operations

### Open Function

```c
static int device_open(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Device opened\n");
    
    // Get device-specific data
    struct my_device *dev = container_of(inode->i_cdev, struct my_device, cdev);
    filp->private_data = dev;
    
    return 0;
}
```

### Release Function

```c
static int device_release(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Device closed\n");
    return 0;
}
```

### Read Function

```c
static ssize_t device_read(struct file *filp, char __user *buf, 
                          size_t count, loff_t *f_pos)
{
    char kernel_buf[100] = "Hello from kernel";
    size_t len = strlen(kernel_buf);
    
    if (*f_pos >= len)
        return 0;  // EOF
    
    if (*f_pos + count > len)
        count = len - *f_pos;
    
    if (copy_to_user(buf, kernel_buf + *f_pos, count))
        return -EFAULT;
    
    *f_pos += count;
    return count;
}
```

### Write Function

```c
static ssize_t device_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    char kernel_buf[100];
    
    if (count > sizeof(kernel_buf) - 1)
        count = sizeof(kernel_buf) - 1;
    
    if (copy_from_user(kernel_buf, buf, count))
        return -EFAULT;
    
    kernel_buf[count] = '\0';
    printk(KERN_INFO "Received: %s\n", kernel_buf);
    
    return count;
}
```

## User Space Data Transfer

### copy_to_user

```c
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
```

Returns: Number of bytes that could not be copied (0 on success)

### copy_from_user

```c
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
```

### put_user and get_user

```c
// Copy single value to user space
put_user(value, user_ptr);

// Copy single value from user space
get_user(value, user_ptr);
```

## Device Classes

### Creating Device Class

```c
#include <linux/device.h>

struct class *my_class;
struct device *my_device;

// Create class
my_class = class_create(THIS_MODULE, "myclass");
if (IS_ERR(my_class)) {
    unregister_chrdev_region(dev, 1);
    return PTR_ERR(my_class);
}

// Create device
my_device = device_create(my_class, NULL, dev, NULL, "mydevice");
if (IS_ERR(my_device)) {
    class_destroy(my_class);
    unregister_chrdev_region(dev, 1);
    return PTR_ERR(my_device);
}
```

### Cleanup

```c
device_destroy(my_class, dev);
class_destroy(my_class);
```

### Benefits
- Automatic `/dev` node creation via udev
- Sysfs entries for device information
- Better device management

## Complete Character Driver Example

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "mychardev"
#define CLASS_NAME "myclass"

static dev_t dev_num;
static struct cdev my_cdev;
static struct class *my_class;
static struct device *my_device;

static char kernel_buffer[256];
static int buffer_size = 0;

static int device_open(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Device opened\n");
    return 0;
}

static int device_release(struct inode *inode, struct file *filp)
{
    printk(KERN_INFO "Device closed\n");
    return 0;
}

static ssize_t device_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    size_t to_read;
    
    if (*f_pos >= buffer_size)
        return 0;
    
    to_read = min(count, (size_t)(buffer_size - *f_pos));
    
    if (copy_to_user(buf, kernel_buffer + *f_pos, to_read))
        return -EFAULT;
    
    *f_pos += to_read;
    return to_read;
}

static ssize_t device_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    size_t to_write;
    
    to_write = min(count, sizeof(kernel_buffer) - 1);
    
    if (copy_from_user(kernel_buffer, buf, to_write))
        return -EFAULT;
    
    kernel_buffer[to_write] = '\0';
    buffer_size = to_write;
    
    printk(KERN_INFO "Received %zu bytes\n", to_write);
    return to_write;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = device_open,
    .release = device_release,
    .read = device_read,
    .write = device_write,
};

static int __init chardev_init(void)
{
    int ret;
    
    // Allocate device number
    ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        printk(KERN_ERR "Failed to allocate device number\n");
        return ret;
    }
    
    printk(KERN_INFO "Device number: Major=%d, Minor=%d\n",
           MAJOR(dev_num), MINOR(dev_num));
    
    // Initialize and add cdev
    cdev_init(&my_cdev, &fops);
    my_cdev.owner = THIS_MODULE;
    
    ret = cdev_add(&my_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        return ret;
    }
    
    // Create class
    my_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(my_class)) {
        cdev_del(&my_cdev);
        unregister_chrdev_region(dev_num, 1);
        return PTR_ERR(my_class);
    }
    
    // Create device
    my_device = device_create(my_class, NULL, dev_num, NULL, DEVICE_NAME);
    if (IS_ERR(my_device)) {
        class_destroy(my_class);
        cdev_del(&my_cdev);
        unregister_chrdev_region(dev_num, 1);
        return PTR_ERR(my_device);
    }
    
    printk(KERN_INFO "Character device registered successfully\n");
    return 0;
}

static void __exit chardev_exit(void)
{
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    
    printk(KERN_INFO "Character device unregistered\n");
}

module_init(chardev_init);
module_exit(chardev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Simple character device driver");
MODULE_VERSION("1.0");
```

## Testing the Driver

### User Space Test Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main()
{
    int fd;
    char write_buf[] = "Hello, Driver!";
    char read_buf[256];
    ssize_t ret;
    
    // Open device
    fd = open("/dev/mychardev", O_RDWR);
    if (fd < 0) {
        perror("Failed to open device");
        return 1;
    }
    
    // Write to device
    ret = write(fd, write_buf, strlen(write_buf));
    printf("Wrote %zd bytes\n", ret);
    
    // Reset file position
    lseek(fd, 0, SEEK_SET);
    
    // Read from device
    ret = read(fd, read_buf, sizeof(read_buf));
    if (ret > 0) {
        read_buf[ret] = '\0';
        printf("Read %zd bytes: %s\n", ret, read_buf);
    }
    
    // Close device
    close(fd);
    
    return 0;
}
```

### Compile and Run

```bash
gcc -o test test.c
sudo ./test
```

## IOCTL Operations

### Defining IOCTL Commands

```c
#include <linux/ioctl.h>

#define MAGIC_NUM 'M'

#define IOCTL_GET_VALUE _IOR(MAGIC_NUM, 0, int)
#define IOCTL_SET_VALUE _IOW(MAGIC_NUM, 1, int)
#define IOCTL_EXCHANGE  _IOWR(MAGIC_NUM, 2, int)
```

Macros:
- `_IO(type, nr)`: No data transfer
- `_IOR(type, nr, datatype)`: Read from driver
- `_IOW(type, nr, datatype)`: Write to driver
- `_IOWR(type, nr, datatype)`: Read and write

### Implementing IOCTL

```c
static long device_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int value;
    
    switch (cmd) {
    case IOCTL_GET_VALUE:
        value = 42;  // Example value
        if (copy_to_user((int __user *)arg, &value, sizeof(value)))
            return -EFAULT;
        break;
        
    case IOCTL_SET_VALUE:
        if (copy_from_user(&value, (int __user *)arg, sizeof(value)))
            return -EFAULT;
        printk(KERN_INFO "Received value: %d\n", value);
        break;
        
    case IOCTL_EXCHANGE:
        if (copy_from_user(&value, (int __user *)arg, sizeof(value)))
            return -EFAULT;
        value *= 2;  // Process value
        if (copy_to_user((int __user *)arg, &value, sizeof(value)))
            return -EFAULT;
        break;
        
    default:
        return -EINVAL;
    }
    
    return 0;
}
```

### User Space IOCTL

```c
#include <sys/ioctl.h>

int fd = open("/dev/mychardev", O_RDWR);
int value;

// Get value
ioctl(fd, IOCTL_GET_VALUE, &value);
printf("Value: %d\n", value);

// Set value
value = 100;
ioctl(fd, IOCTL_SET_VALUE, &value);

// Exchange value
value = 50;
ioctl(fd, IOCTL_EXCHANGE, &value);
printf("Exchanged value: %d\n", value);

close(fd);
```

## Sysfs Interface

### Creating Sysfs Attributes

```c
static ssize_t value_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    return sprintf(buf, "%d\n", my_value);
}

static ssize_t value_store(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t count)
{
    sscanf(buf, "%d", &my_value);
    return count;
}

static DEVICE_ATTR_RW(value);

static struct attribute *mydev_attrs[] = {
    &dev_attr_value.attr,
    NULL,
};

static struct attribute_group mydev_attr_group = {
    .attrs = mydev_attrs,
};

// In init function
sysfs_create_group(&my_device->kobj, &mydev_attr_group);

// In exit function
sysfs_remove_group(&my_device->kobj, &mydev_attr_group);
```

### Accessing Sysfs

```bash
# Read attribute
cat /sys/class/myclass/mydevice/value

# Write attribute
echo 42 > /sys/class/myclass/mydevice/value
```

## Procfs Interface

```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

static int my_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Device status: OK\n");
    seq_printf(m, "Value: %d\n", my_value);
    return 0;
}

static int my_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, my_proc_show, NULL);
}

static const struct proc_ops my_proc_ops = {
    .proc_open = my_proc_open,
    .proc_read = seq_read,
    .proc_lseek = seq_lseek,
    .proc_release = single_release,
};

// In init function
proc_create("mydevice", 0, NULL, &my_proc_ops);

// In exit function
remove_proc_entry("mydevice", NULL);
```

## Poll/Select Support

```c
#include <linux/poll.h>

static unsigned int device_poll(struct file *filp, poll_table *wait)
{
    unsigned int mask = 0;
    
    poll_wait(filp, &my_wait_queue, wait);
    
    if (data_available)
        mask |= POLLIN | POLLRDNORM;
    
    if (space_available)
        mask |= POLLOUT | POLLWRNORM;
    
    return mask;
}
```

## Best Practices

1. **Always validate user input**
2. **Use proper error handling**
3. **Clean up resources in reverse order**
4. **Protect shared data with locks**
5. **Use copy_to_user/copy_from_user for user space data**
6. **Check return values of all functions**
7. **Provide meaningful error messages**
8. **Document driver interface**

## Common Errors

- **Segmentation fault**: Invalid memory access
- **EFAULT**: Bad user space address
- **EINVAL**: Invalid argument
- **EBUSY**: Device already in use
- **ENOMEM**: Out of memory

## Next Steps

Proceed to [Pseudo Char Device Driver](05-pseudo-char-device-driver.md) for a practical implementation example.
