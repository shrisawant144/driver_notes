# Platform Device Drivers

## Overview

Platform devices are pseudo-devices used to represent devices that are not discoverable by hardware (unlike PCI or USB). They are commonly used for SoC (System-on-Chip) integrated peripherals and devices described in device tree.

## Platform Device Model

### Architecture

```
Platform Bus
    ├── Platform Device (Hardware description)
    └── Platform Driver (Software driver)
```

When a platform device and driver match, the driver's probe function is called.

## Platform Device

### Structure

```c
struct platform_device {
    const char *name;
    int id;
    struct device dev;
    u32 num_resources;
    struct resource *resource;
};
```

### Creating Platform Device

```c
#include <linux/platform_device.h>

static struct resource my_resources[] = {
    {
        .start = 0x10000000,
        .end   = 0x10000FFF,
        .flags = IORESOURCE_MEM,
    },
    {
        .start = 42,
        .end   = 42,
        .flags = IORESOURCE_IRQ,
    },
};

static struct platform_device my_device = {
    .name = "my-platform-device",
    .id = -1,
    .num_resources = ARRAY_SIZE(my_resources),
    .resource = my_resources,
};

static int __init device_init(void)
{
    return platform_device_register(&my_device);
}

static void __exit device_exit(void)
{
    platform_device_unregister(&my_device);
}
```

## Platform Driver

### Structure

```c
struct platform_driver {
    int (*probe)(struct platform_device *);
    int (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int (*suspend)(struct platform_device *, pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;
};
```

### Implementing Platform Driver

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/mod_devicetable.h>

static int my_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;
    int irq;
    
    printk(KERN_INFO "Device probed: %s\n", pdev->name);
    
    // Get memory resource
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res) {
        dev_err(&pdev->dev, "No memory resource\n");
        return -ENODEV;
    }
    
    // Request memory region
    if (!request_mem_region(res->start, resource_size(res), pdev->name)) {
        dev_err(&pdev->dev, "Memory region busy\n");
        return -EBUSY;
    }
    
    // Map memory
    base = ioremap(res->start, resource_size(res));
    if (!base) {
        release_mem_region(res->start, resource_size(res));
        return -ENOMEM;
    }
    
    // Get IRQ
    irq = platform_get_irq(pdev, 0);
    if (irq < 0) {
        dev_err(&pdev->dev, "No IRQ resource\n");
        iounmap(base);
        release_mem_region(res->start, resource_size(res));
        return irq;
    }
    
    // Store in device data
    platform_set_drvdata(pdev, base);
    
    dev_info(&pdev->dev, "Initialized at 0x%lx, IRQ %d\n",
             (unsigned long)res->start, irq);
    
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    void __iomem *base = platform_get_drvdata(pdev);
    struct resource *res;
    
    printk(KERN_INFO "Device removed: %s\n", pdev->name);
    
    // Unmap memory
    if (base)
        iounmap(base);
    
    // Release memory region
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (res)
        release_mem_region(res->start, resource_size(res));
    
    return 0;
}

static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-platform-device",
        .owner = THIS_MODULE,
    },
};

module_platform_driver(my_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Platform device driver example");
```

## Device Tree Integration

### Device Tree Node

```dts
my_device: mydevice@10000000 {
    compatible = "vendor,my-device";
    reg = <0x10000000 0x1000>;
    interrupts = <0 42 4>;
    clock-frequency = <100000000>;
    status = "okay";
};
```

### Driver with Device Tree

```c
#include <linux/of.h>
#include <linux/of_device.h>

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device", },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static int my_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    u32 clock_freq;
    
    // Read device tree properties
    if (of_property_read_u32(np, "clock-frequency", &clock_freq)) {
        dev_err(&pdev->dev, "Missing clock-frequency property\n");
        return -EINVAL;
    }
    
    dev_info(&pdev->dev, "Clock frequency: %u Hz\n", clock_freq);
    
    // ... rest of probe
    
    return 0;
}

static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-platform-device",
        .owner = THIS_MODULE,
        .of_match_table = my_of_match,
    },
};
```

## Resource Management

### Resource Types

```c
IORESOURCE_MEM    // Memory region
IORESOURCE_IO     // I/O port
IORESOURCE_IRQ    // Interrupt
IORESOURCE_DMA    // DMA channel
IORESOURCE_BUS    // Bus
```

### Getting Resources

```c
// Get resource by index
struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);

// Get IRQ
int irq = platform_get_irq(pdev, 0);

// Get resource by name
res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "control");
```

### Managed Resources (devm_*)

```c
static int my_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;
    
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    
    // Managed ioremap - automatically freed on driver detach
    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);
    
    // Managed memory allocation
    struct my_data *data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    // No need to free in remove function
    
    return 0;
}
```

## Complete Platform Driver Example

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>
#include <linux/interrupt.h>
#include <linux/fs.h>
#include <linux/cdev.h>

#define DEVICE_NAME "myplatdev"

struct myplat_device {
    void __iomem *base;
    int irq;
    struct cdev cdev;
    dev_t devt;
    struct class *class;
    struct device *device;
};

static irqreturn_t myplat_irq_handler(int irq, void *dev_id)
{
    struct myplat_device *mydev = dev_id;
    
    printk(KERN_INFO "Interrupt received\n");
    
    // Handle interrupt
    
    return IRQ_HANDLED;
}

static int myplat_open(struct inode *inode, struct file *filp)
{
    struct myplat_device *mydev;
    mydev = container_of(inode->i_cdev, struct myplat_device, cdev);
    filp->private_data = mydev;
    return 0;
}

static int myplat_release(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t myplat_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    struct myplat_device *mydev = filp->private_data;
    u32 value;
    
    // Read from hardware register
    value = ioread32(mydev->base);
    
    if (copy_to_user(buf, &value, sizeof(value)))
        return -EFAULT;
    
    return sizeof(value);
}

static ssize_t myplat_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct myplat_device *mydev = filp->private_data;
    u32 value;
    
    if (count < sizeof(value))
        return -EINVAL;
    
    if (copy_from_user(&value, buf, sizeof(value)))
        return -EFAULT;
    
    // Write to hardware register
    iowrite32(value, mydev->base);
    
    return sizeof(value);
}

static struct file_operations myplat_fops = {
    .owner = THIS_MODULE,
    .open = myplat_open,
    .release = myplat_release,
    .read = myplat_read,
    .write = myplat_write,
};

static int myplat_probe(struct platform_device *pdev)
{
    struct myplat_device *mydev;
    struct resource *res;
    int ret;
    
    dev_info(&pdev->dev, "Probing device\n");
    
    // Allocate device structure
    mydev = devm_kzalloc(&pdev->dev, sizeof(*mydev), GFP_KERNEL);
    if (!mydev)
        return -ENOMEM;
    
    // Get and map memory resource
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    mydev->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(mydev->base))
        return PTR_ERR(mydev->base);
    
    // Get IRQ
    mydev->irq = platform_get_irq(pdev, 0);
    if (mydev->irq >= 0) {
        ret = devm_request_irq(&pdev->dev, mydev->irq, myplat_irq_handler,
                              0, dev_name(&pdev->dev), mydev);
        if (ret) {
            dev_err(&pdev->dev, "Failed to request IRQ\n");
            return ret;
        }
    }
    
    // Allocate character device number
    ret = alloc_chrdev_region(&mydev->devt, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        dev_err(&pdev->dev, "Failed to allocate device number\n");
        return ret;
    }
    
    // Initialize and add cdev
    cdev_init(&mydev->cdev, &myplat_fops);
    mydev->cdev.owner = THIS_MODULE;
    
    ret = cdev_add(&mydev->cdev, mydev->devt, 1);
    if (ret < 0) {
        unregister_chrdev_region(mydev->devt, 1);
        return ret;
    }
    
    // Create device class
    mydev->class = class_create(THIS_MODULE, DEVICE_NAME);
    if (IS_ERR(mydev->class)) {
        cdev_del(&mydev->cdev);
        unregister_chrdev_region(mydev->devt, 1);
        return PTR_ERR(mydev->class);
    }
    
    // Create device
    mydev->device = device_create(mydev->class, &pdev->dev, mydev->devt,
                                  NULL, DEVICE_NAME);
    if (IS_ERR(mydev->device)) {
        class_destroy(mydev->class);
        cdev_del(&mydev->cdev);
        unregister_chrdev_region(mydev->devt, 1);
        return PTR_ERR(mydev->device);
    }
    
    platform_set_drvdata(pdev, mydev);
    
    dev_info(&pdev->dev, "Device initialized successfully\n");
    return 0;
}

static int myplat_remove(struct platform_device *pdev)
{
    struct myplat_device *mydev = platform_get_drvdata(pdev);
    
    device_destroy(mydev->class, mydev->devt);
    class_destroy(mydev->class);
    cdev_del(&mydev->cdev);
    unregister_chrdev_region(mydev->devt, 1);
    
    dev_info(&pdev->dev, "Device removed\n");
    return 0;
}

static const struct of_device_id myplat_of_match[] = {
    { .compatible = "vendor,myplatdev", },
    { }
};
MODULE_DEVICE_TABLE(of, myplat_of_match);

static struct platform_driver myplat_driver = {
    .probe = myplat_probe,
    .remove = myplat_remove,
    .driver = {
        .name = DEVICE_NAME,
        .owner = THIS_MODULE,
        .of_match_table = myplat_of_match,
    },
};

module_platform_driver(myplat_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Complete platform device driver");
MODULE_VERSION("1.0");
```

## Power Management

```c
#ifdef CONFIG_PM
static int myplat_suspend(struct device *dev)
{
    struct myplat_device *mydev = dev_get_drvdata(dev);
    
    // Save device state
    // Disable clocks, power down
    
    dev_info(dev, "Device suspended\n");
    return 0;
}

static int myplat_resume(struct device *dev)
{
    struct myplat_device *mydev = dev_get_drvdata(dev);
    
    // Restore device state
    // Enable clocks, power up
    
    dev_info(dev, "Device resumed\n");
    return 0;
}

static const struct dev_pm_ops myplat_pm_ops = {
    .suspend = myplat_suspend,
    .resume = myplat_resume,
};

#define MYPLAT_PM_OPS (&myplat_pm_ops)
#else
#define MYPLAT_PM_OPS NULL
#endif

static struct platform_driver myplat_driver = {
    .probe = myplat_probe,
    .remove = myplat_remove,
    .driver = {
        .name = DEVICE_NAME,
        .owner = THIS_MODULE,
        .of_match_table = myplat_of_match,
        .pm = MYPLAT_PM_OPS,
    },
};
```

## Platform Data

### Passing Platform Data

```c
struct myplat_platform_data {
    int gpio_pin;
    int clock_rate;
    bool enable_feature;
};

static struct myplat_platform_data my_pdata = {
    .gpio_pin = 42,
    .clock_rate = 100000000,
    .enable_feature = true,
};

static struct platform_device my_device = {
    .name = "myplatdev",
    .id = -1,
    .dev = {
        .platform_data = &my_pdata,
    },
};

// In driver probe
static int myplat_probe(struct platform_device *pdev)
{
    struct myplat_platform_data *pdata = dev_get_platdata(&pdev->dev);
    
    if (pdata) {
        printk(KERN_INFO "GPIO pin: %d\n", pdata->gpio_pin);
        printk(KERN_INFO "Clock rate: %d\n", pdata->clock_rate);
    }
    
    return 0;
}
```

## Best Practices

1. **Use managed resources (devm_*)** for automatic cleanup
2. **Implement proper error handling** in probe
3. **Support device tree** for modern systems
4. **Add power management** support
5. **Use platform_set_drvdata** to store device context
6. **Check resource availability** before use
7. **Provide module device table** for auto-loading

## Next Steps

Proceed to [Kernel Data Structures](07-kernel-data-structures.md) to learn about kernel-provided data structures.
