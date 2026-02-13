# Linux Driver Model

## 🎯 Layman's Explanation

**What is the Linux Driver Model?**
It's the **organizational system** for how Linux manages all devices and drivers. Think of it as the filing system for hardware.

**Real-World Analogy:**
Imagine a large company:
- **Devices** = Employees
- **Drivers** = Job descriptions/roles
- **Buses** = Departments (USB dept, PCI dept, etc.)
- **Classes** = Job categories (input devices, storage devices, etc.)
- **sysfs** = Company directory (org chart you can browse)

**The Hierarchy:**
```
Bus (USB)
  ├── Device (Mouse)
  │     └── Driver (USB HID Driver)
  └── Device (Keyboard)
        └── Driver (USB HID Driver)
```

**Why This Model?**
Before: Chaos - every driver did things differently
After: Organized - standard way to:
- Register devices
- Match drivers to devices
- Handle power management
- Support hot-plugging
- Expose info to user space

**Key Concepts:**

**1. Bus**
A communication channel. Examples:
- USB bus
- PCI bus
- I2C bus
- Platform bus (for built-in devices)

**Think of it as:** Different types of roads (highway, city street, dirt road)

**2. Device**
Actual hardware (or virtual hardware)

**3. Driver**
Software that knows how to control a device

**4. Class**
Groups devices by what they do, not how they connect:
- Input class (keyboards, mice, touchscreens)
- Block class (hard drives, USB drives)
- Network class (WiFi, Ethernet)

**The Matching Process:**
```
New USB mouse plugged in
    ↓
USB bus detects it
    ↓
Kernel creates device structure
    ↓
Kernel looks for matching driver
    ↓
Found! USB HID driver
    ↓
Driver's probe() function called
    ↓
Mouse works!
```

**Analogy:**
- New employee joins (device)
- HR checks their skills (device ID)
- Finds matching job role (driver)
- Employee starts working (probe() called)

**sysfs - The File System View:**
```
/sys/
  ├── bus/
  │   ├── usb/
  │   │   ├── devices/
  │   │   └── drivers/
  │   └── pci/
  ├── class/
  │   ├── input/
  │   ├── net/
  │   └── block/
  └── devices/
```

You can browse this! Try: `ls /sys/class/net/` to see network devices.

**Analogy:**
- `/sys/bus/` = View by department
- `/sys/class/` = View by job type
- `/sys/devices/` = View by hierarchy

**kobject - The Building Block:**
Every device, driver, bus is a `kobject` (kernel object). It provides:
- Reference counting (knows when to free memory)
- Hierarchy (parent-child relationships)
- sysfs representation

**Think of it as:** The employee ID card system - tracks everyone, their relationships, and their info.

**udev - The User Space Helper:**
When a device appears, kernel tells `udev`:
```
Kernel: "Hey udev, new USB device appeared!"
udev: "Got it! Let me create /dev/sda for it"
udev: "And I'll load the right driver"
udev: "And notify desktop environment"
```

**Analogy:**
- Kernel = Factory floor (detects new machine)
- udev = Office admin (handles paperwork, notifications)

**Power Management:**
The driver model handles:
- **Suspend** - Device goes to sleep
- **Resume** - Device wakes up
- **Runtime PM** - Device sleeps when idle

**Analogy:**
- Suspend = Employee goes on vacation
- Resume = Employee returns
- Runtime PM = Employee takes breaks when not busy

**Hot-Plugging:**
Devices can be added/removed while system runs:
```
USB drive plugged in → Detected → Driver loaded → Works
USB drive removed → Driver notified → Cleanup → Safe
```

**The Big Picture:**
```
User Space
    ↕ (sysfs, udev)
Linux Driver Model
    ↕
Drivers
    ↕
Hardware
```

Everything is organized, standardized, and visible!

**Why Should You Care?**
When writing drivers, you:
- Register with a bus
- Implement probe/remove functions
- Follow the model's rules
- Get power management for free
- Get sysfs entries automatically
- Work with hot-plugging automatically

**It's like:** Following company HR procedures - seems like paperwork, but gives you benefits, structure, and integration with the system.

## Overview

The Linux Driver Model provides a unified framework for device drivers, power management, and hot-plugging. It organizes devices, drivers, and buses in a hierarchical structure represented in sysfs.

## Core Components

### Devices
Physical or virtual hardware

### Drivers
Software that controls devices

### Buses
Communication channels connecting devices

### Classes
Grouping of devices by functionality

## Device Structure

```c
#include <linux/device.h>

struct device {
    struct device *parent;
    struct device_private *p;
    struct kobject kobj;
    const char *init_name;
    const struct device_type *type;
    struct bus_type *bus;
    struct device_driver *driver;
    void *platform_data;
    void *driver_data;
    struct dev_pm_info power;
    // ...
};
```

### Device Registration

```c
int device_register(struct device *dev);
void device_unregister(struct device *dev);

// Initialize device
void device_initialize(struct device *dev);
int device_add(struct device *dev);
void device_del(struct device *dev);
```

### Example

```c
static struct device my_device = {
    .init_name = "mydevice",
    .release = my_device_release,
};

static void my_device_release(struct device *dev)
{
    printk(KERN_INFO "Device released\n");
}

static int __init my_init(void)
{
    int ret;
    
    ret = device_register(&my_device);
    if (ret) {
        printk(KERN_ERR "Failed to register device\n");
        return ret;
    }
    
    return 0;
}

static void __exit my_exit(void)
{
    device_unregister(&my_device);
}
```

## Driver Structure

```c
struct device_driver {
    const char *name;
    struct bus_type *bus;
    struct module *owner;
    const struct of_device_id *of_match_table;
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
};
```

### Driver Registration

```c
int driver_register(struct device_driver *drv);
void driver_unregister(struct device_driver *drv);
```

## Bus Type

```c
struct bus_type {
    const char *name;
    const char *dev_name;
    struct device *dev_root;
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    // ...
};
```

### Bus Registration

```c
int bus_register(struct bus_type *bus);
void bus_unregister(struct bus_type *bus);
```

### Example: Custom Bus

```c
#include <linux/device.h>
#include <linux/module.h>

// Custom bus match function
static int my_bus_match(struct device *dev, struct device_driver *drv)
{
    // Match by name
    return !strcmp(dev_name(dev), drv->name);
}

static struct bus_type my_bus_type = {
    .name = "my_bus",
    .match = my_bus_match,
};

static int __init my_bus_init(void)
{
    int ret;
    
    ret = bus_register(&my_bus_type);
    if (ret) {
        printk(KERN_ERR "Failed to register bus\n");
        return ret;
    }
    
    printk(KERN_INFO "Bus registered\n");
    return 0;
}

static void __exit my_bus_exit(void)
{
    bus_unregister(&my_bus_type);
    printk(KERN_INFO "Bus unregistered\n");
}

module_init(my_bus_init);
module_exit(my_bus_exit);

MODULE_LICENSE("GPL");
```

## Device Class

Groups devices by functionality.

```c
struct class {
    const char *name;
    struct module *owner;
    const struct attribute_group **dev_groups;
    int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
};
```

### Class Operations

```c
// Create class
struct class *class_create(struct module *owner, const char *name);

// Destroy class
void class_destroy(struct class *cls);

// Create device in class
struct device *device_create(struct class *cls,
                            struct device *parent,
                            dev_t devt,
                            void *drvdata,
                            const char *fmt, ...);

// Destroy device
void device_destroy(struct class *cls, dev_t devt);
```

### Example

```c
static struct class *my_class;
static struct device *my_device;
static dev_t devt;

static int __init class_init(void)
{
    // Create class
    my_class = class_create(THIS_MODULE, "myclass");
    if (IS_ERR(my_class))
        return PTR_ERR(my_class);
    
    // Allocate device number
    alloc_chrdev_region(&devt, 0, 1, "mydevice");
    
    // Create device
    my_device = device_create(my_class, NULL, devt, NULL, "mydevice");
    if (IS_ERR(my_device)) {
        class_destroy(my_class);
        unregister_chrdev_region(devt, 1);
        return PTR_ERR(my_device);
    }
    
    return 0;
}

static void __exit class_exit(void)
{
    device_destroy(my_class, devt);
    class_destroy(my_class);
    unregister_chrdev_region(devt, 1);
}
```

## Kobject

Base object for device model.

```c
#include <linux/kobject.h>

struct kobject {
    const char *name;
    struct list_head entry;
    struct kobject *parent;
    struct kset *kset;
    struct kobj_type *ktype;
    struct kernfs_node *sd;
    struct kref kref;
    // ...
};
```

### Kobject Operations

```c
void kobject_init(struct kobject *kobj, struct kobj_type *ktype);
int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);
int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype,
                        struct kobject *parent, const char *fmt, ...);

void kobject_del(struct kobject *kobj);
void kobject_put(struct kobject *kobj);
struct kobject *kobject_get(struct kobject *kobj);
```

## Sysfs Attributes

### Device Attributes

```c
#include <linux/device.h>

// Define attribute
static ssize_t my_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    return sprintf(buf, "Hello from sysfs\n");
}

static ssize_t my_store(struct device *dev, struct device_attribute *attr,
                       const char *buf, size_t count)
{
    printk(KERN_INFO "Received: %s\n", buf);
    return count;
}

static DEVICE_ATTR(my_attr, 0644, my_show, my_store);

// Or use helper macros
static DEVICE_ATTR_RO(my_attr);  // Read-only
static DEVICE_ATTR_WO(my_attr);  // Write-only
static DEVICE_ATTR_RW(my_attr);  // Read-write
```

### Creating Attributes

```c
// Single attribute
int device_create_file(struct device *dev, const struct device_attribute *attr);
void device_remove_file(struct device *dev, const struct device_attribute *attr);

// Example
device_create_file(my_device, &dev_attr_my_attr);
device_remove_file(my_device, &dev_attr_my_attr);
```

### Attribute Groups

```c
static struct attribute *my_attrs[] = {
    &dev_attr_attr1.attr,
    &dev_attr_attr2.attr,
    &dev_attr_attr3.attr,
    NULL,
};

static struct attribute_group my_attr_group = {
    .name = "my_group",
    .attrs = my_attrs,
};

// Create group
sysfs_create_group(&dev->kobj, &my_attr_group);

// Remove group
sysfs_remove_group(&dev->kobj, &my_attr_group);
```

### Example

```c
struct my_device {
    struct device dev;
    int value;
    char name[32];
};

static ssize_t value_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    struct my_device *mydev = container_of(dev, struct my_device, dev);
    return sprintf(buf, "%d\n", mydev->value);
}

static ssize_t value_store(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t count)
{
    struct my_device *mydev = container_of(dev, struct my_device, dev);
    sscanf(buf, "%d", &mydev->value);
    return count;
}

static ssize_t name_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    struct my_device *mydev = container_of(dev, struct my_device, dev);
    return sprintf(buf, "%s\n", mydev->name);
}

static DEVICE_ATTR_RW(value);
static DEVICE_ATTR_RO(name);

static struct attribute *my_device_attrs[] = {
    &dev_attr_value.attr,
    &dev_attr_name.attr,
    NULL,
};

ATTRIBUTE_GROUPS(my_device);
```

## Driver Data

Store private data with device.

```c
// Set driver data
void dev_set_drvdata(struct device *dev, void *data);

// Get driver data
void *dev_get_drvdata(const struct device *dev);

// Example
struct my_private_data {
    int value;
    void *buffer;
};

static int my_probe(struct device *dev)
{
    struct my_private_data *priv;
    
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;
    
    priv->value = 42;
    dev_set_drvdata(dev, priv);
    
    return 0;
}

static int my_remove(struct device *dev)
{
    struct my_private_data *priv = dev_get_drvdata(dev);
    
    printk(KERN_INFO "Value: %d\n", priv->value);
    return 0;
}
```

## Power Management

### PM Operations

```c
struct dev_pm_ops {
    int (*suspend)(struct device *dev);
    int (*resume)(struct device *dev);
    int (*freeze)(struct device *dev);
    int (*thaw)(struct device *dev);
    int (*poweroff)(struct device *dev);
    int (*restore)(struct device *dev);
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```

### Example

```c
#ifdef CONFIG_PM
static int my_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    
    // Save device state
    // Disable clocks, power down
    
    dev_info(dev, "Device suspended\n");
    return 0;
}

static int my_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    
    // Restore device state
    // Enable clocks, power up
    
    dev_info(dev, "Device resumed\n");
    return 0;
}

static const struct dev_pm_ops my_pm_ops = {
    .suspend = my_suspend,
    .resume = my_resume,
};

#define MY_PM_OPS (&my_pm_ops)
#else
#define MY_PM_OPS NULL
#endif

static struct platform_driver my_driver = {
    .driver = {
        .name = "mydevice",
        .pm = MY_PM_OPS,
    },
};
```

## Runtime PM

Dynamic power management.

```c
#include <linux/pm_runtime.h>

// Enable runtime PM
pm_runtime_enable(dev);

// Disable runtime PM
pm_runtime_disable(dev);

// Get device (increment usage count)
pm_runtime_get_sync(dev);

// Put device (decrement usage count)
pm_runtime_put(dev);
pm_runtime_put_sync(dev);

// Mark as active/suspended
pm_runtime_set_active(dev);
pm_runtime_set_suspended(dev);
```

## Complete Example

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/device.h>

struct my_device {
    struct device dev;
    int value;
};

static void my_device_release(struct device *dev)
{
    struct my_device *mydev = container_of(dev, struct my_device, dev);
    kfree(mydev);
}

static ssize_t value_show(struct device *dev, struct device_attribute *attr, char *buf)
{
    struct my_device *mydev = container_of(dev, struct my_device, dev);
    return sprintf(buf, "%d\n", mydev->value);
}

static ssize_t value_store(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t count)
{
    struct my_device *mydev = container_of(dev, struct my_device, dev);
    sscanf(buf, "%d", &mydev->value);
    return count;
}

static DEVICE_ATTR_RW(value);

static struct attribute *my_device_attrs[] = {
    &dev_attr_value.attr,
    NULL,
};

ATTRIBUTE_GROUPS(my_device);

static struct class my_class = {
    .name = "my_class",
    .owner = THIS_MODULE,
    .dev_groups = my_device_groups,
};

static int __init driver_model_init(void)
{
    struct my_device *mydev;
    int ret;
    
    // Register class
    ret = class_register(&my_class);
    if (ret) {
        printk(KERN_ERR "Failed to register class\n");
        return ret;
    }
    
    // Create device
    mydev = kzalloc(sizeof(*mydev), GFP_KERNEL);
    if (!mydev) {
        class_unregister(&my_class);
        return -ENOMEM;
    }
    
    mydev->value = 42;
    mydev->dev.class = &my_class;
    mydev->dev.release = my_device_release;
    dev_set_name(&mydev->dev, "mydevice0");
    
    ret = device_register(&mydev->dev);
    if (ret) {
        kfree(mydev);
        class_unregister(&my_class);
        return ret;
    }
    
    printk(KERN_INFO "Driver model initialized\n");
    return 0;
}

static void __exit driver_model_exit(void)
{
    class_unregister(&my_class);
    printk(KERN_INFO "Driver model exited\n");
}

module_init(driver_model_init);
module_exit(driver_model_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Linux driver model example");
```

## Sysfs Hierarchy

```
/sys/
├── bus/           # Buses
│   └── platform/
│       ├── devices/
│       └── drivers/
├── class/         # Device classes
│   └── myclass/
│       └── mydevice/
├── devices/       # All devices
│   └── platform/
└── module/        # Loaded modules
```

## Best Practices

1. **Use device model** for proper integration
2. **Implement power management** for mobile devices
3. **Provide sysfs attributes** for configuration
4. **Use managed resources** (devm_*)
5. **Follow naming conventions**
6. **Document sysfs interface**
7. **Handle hotplug** properly
8. **Test with different configurations**

## Next Steps

Proceed to [Driver Debugging Techniques](13-driver-debugging-techniques.md) to learn about debugging kernel drivers.
