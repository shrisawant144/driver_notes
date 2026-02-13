# IO Port Access

## Overview

Hardware devices communicate with the CPU through I/O ports (port-mapped I/O) or memory-mapped I/O (MMIO). This chapter covers accessing hardware registers and I/O operations.

## I/O Port vs Memory-Mapped I/O

### Port-Mapped I/O (x86)
- Separate address space for I/O
- Special CPU instructions (IN/OUT)
- Limited address space (64KB on x86)

### Memory-Mapped I/O (Most architectures)
- I/O registers mapped to memory addresses
- Standard memory access instructions
- Larger address space

## I/O Port Access (x86)

### Requesting I/O Ports

```c
#include <linux/ioport.h>

// Request port region
struct resource *request_region(unsigned long start,
                               unsigned long len,
                               const char *name);

// Release port region
void release_region(unsigned long start, unsigned long len);

// Check if available
int check_region(unsigned long start, unsigned long len);
```

### Example

```c
#define PORT_BASE 0x300
#define PORT_COUNT 8

static int __init port_init(void)
{
    if (!request_region(PORT_BASE, PORT_COUNT, "mydevice")) {
        printk(KERN_ERR "Failed to request I/O ports\n");
        return -EBUSY;
    }
    
    printk(KERN_INFO "I/O ports 0x%x-0x%x allocated\n",
           PORT_BASE, PORT_BASE + PORT_COUNT - 1);
    
    return 0;
}

static void __exit port_exit(void)
{
    release_region(PORT_BASE, PORT_COUNT);
    printk(KERN_INFO "I/O ports released\n");
}
```

### Reading/Writing Ports

```c
#include <asm/io.h>

// 8-bit
u8 inb(unsigned long port);
void outb(u8 value, unsigned long port);

// 16-bit
u16 inw(unsigned long port);
void outw(u16 value, unsigned long port);

// 32-bit
u32 inl(unsigned long port);
void outl(u32 value, unsigned long port);

// String operations
void insb(unsigned long port, void *addr, unsigned long count);
void outsb(unsigned long port, const void *addr, unsigned long count);
void insw(unsigned long port, void *addr, unsigned long count);
void outsw(unsigned long port, const void *addr, unsigned long count);
```

### Example

```c
#define DATA_PORT 0x300
#define STATUS_PORT 0x301
#define CONTROL_PORT 0x302

static void write_data(u8 data)
{
    // Wait until ready
    while (!(inb(STATUS_PORT) & 0x01))
        cpu_relax();
    
    // Write data
    outb(data, DATA_PORT);
}

static u8 read_data(void)
{
    // Wait for data available
    while (!(inb(STATUS_PORT) & 0x02))
        cpu_relax();
    
    // Read data
    return inb(DATA_PORT);
}

static void set_control(u8 flags)
{
    outb(flags, CONTROL_PORT);
}
```

## Memory-Mapped I/O

### Requesting Memory Region

```c
#include <linux/ioport.h>

// Request memory region
struct resource *request_mem_region(unsigned long start,
                                   unsigned long len,
                                   const char *name);

// Release memory region
void release_mem_region(unsigned long start, unsigned long len);
```

### Mapping Memory

```c
#include <asm/io.h>

// Map I/O memory
void __iomem *ioremap(phys_addr_t phys_addr, unsigned long size);

// Unmap I/O memory
void iounmap(void __iomem *addr);

// Variants
void __iomem *ioremap_nocache(phys_addr_t phys_addr, unsigned long size);
void __iomem *ioremap_wc(phys_addr_t phys_addr, unsigned long size);  // Write-combining
void __iomem *ioremap_wt(phys_addr_t phys_addr, unsigned long size);  // Write-through
```

### Example

```c
#define MMIO_BASE 0x10000000
#define MMIO_SIZE 0x1000

static void __iomem *base_addr;

static int __init mmio_init(void)
{
    // Request memory region
    if (!request_mem_region(MMIO_BASE, MMIO_SIZE, "mydevice")) {
        printk(KERN_ERR "Failed to request memory region\n");
        return -EBUSY;
    }
    
    // Map memory
    base_addr = ioremap(MMIO_BASE, MMIO_SIZE);
    if (!base_addr) {
        release_mem_region(MMIO_BASE, MMIO_SIZE);
        return -ENOMEM;
    }
    
    printk(KERN_INFO "MMIO mapped at virtual address %p\n", base_addr);
    return 0;
}

static void __exit mmio_exit(void)
{
    if (base_addr) {
        iounmap(base_addr);
        release_mem_region(MMIO_BASE, MMIO_SIZE);
    }
}
```

### Reading/Writing MMIO

```c
#include <linux/io.h>

// 8-bit
u8 ioread8(const volatile void __iomem *addr);
void iowrite8(u8 value, volatile void __iomem *addr);

// 16-bit
u16 ioread16(const volatile void __iomem *addr);
void iowrite16(u16 value, volatile void __iomem *addr);

// 32-bit
u32 ioread32(const volatile void __iomem *addr);
void iowrite32(u32 value, volatile void __iomem *addr);

// 64-bit
u64 ioread64(const volatile void __iomem *addr);
void iowrite64(u64 value, volatile void __iomem *addr);

// Repeated access
void ioread8_rep(const volatile void __iomem *addr, void *buffer, unsigned long count);
void iowrite8_rep(volatile void __iomem *addr, const void *buffer, unsigned long count);
```

### Example

```c
#define REG_CONTROL  0x00
#define REG_STATUS   0x04
#define REG_DATA     0x08
#define REG_INTERRUPT 0x0C

struct my_device {
    void __iomem *base;
};

static void device_init_hw(struct my_device *dev)
{
    // Reset device
    iowrite32(0x01, dev->base + REG_CONTROL);
    
    // Wait for reset complete
    while (ioread32(dev->base + REG_STATUS) & 0x01)
        cpu_relax();
    
    // Enable interrupts
    iowrite32(0xFF, dev->base + REG_INTERRUPT);
}

static void device_write_data(struct my_device *dev, u32 data)
{
    // Check if ready
    if (!(ioread32(dev->base + REG_STATUS) & 0x02)) {
        printk(KERN_WARNING "Device not ready\n");
        return;
    }
    
    // Write data
    iowrite32(data, dev->base + REG_DATA);
}

static u32 device_read_data(struct my_device *dev)
{
    return ioread32(dev->base + REG_DATA);
}
```

## Memory Barriers

Ensure proper ordering of I/O operations.

```c
#include <linux/io.h>

// Memory barriers
mb();   // Full memory barrier
rmb();  // Read memory barrier
wmb();  // Write memory barrier

// I/O barriers
mmiowb();  // MMIO write barrier

// Example
iowrite32(value1, reg1);
wmb();  // Ensure write completes
iowrite32(value2, reg2);
```

## Platform Device Resources

### Getting Resources

```c
static int my_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;
    
    // Get memory resource
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res) {
        dev_err(&pdev->dev, "No memory resource\n");
        return -ENODEV;
    }
    
    // Request and map
    if (!request_mem_region(res->start, resource_size(res), pdev->name)) {
        dev_err(&pdev->dev, "Memory region busy\n");
        return -EBUSY;
    }
    
    base = ioremap(res->start, resource_size(res));
    if (!base) {
        release_mem_region(res->start, resource_size(res));
        return -ENOMEM;
    }
    
    // Store for later use
    platform_set_drvdata(pdev, base);
    
    return 0;
}
```

### Managed Resources

```c
static int my_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;
    
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    
    // Managed ioremap - automatically freed on detach
    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);
    
    // No need to call iounmap or release_mem_region
    
    return 0;
}
```

## Register Access Macros

### Defining Registers

```c
#define REG_CONTROL     0x00
#define REG_STATUS      0x04
#define REG_DATA        0x08
#define REG_INTERRUPT   0x0C

// Bit definitions
#define CONTROL_ENABLE  BIT(0)
#define CONTROL_RESET   BIT(1)
#define STATUS_READY    BIT(0)
#define STATUS_ERROR    BIT(1)

// Helper macros
#define REG_READ(dev, reg)          ioread32((dev)->base + (reg))
#define REG_WRITE(dev, reg, val)    iowrite32((val), (dev)->base + (reg))

#define REG_SET_BIT(dev, reg, bit)  \
    REG_WRITE(dev, reg, REG_READ(dev, reg) | (bit))

#define REG_CLEAR_BIT(dev, reg, bit) \
    REG_WRITE(dev, reg, REG_READ(dev, reg) & ~(bit))
```

### Usage

```c
struct my_device {
    void __iomem *base;
};

static void device_enable(struct my_device *dev)
{
    REG_SET_BIT(dev, REG_CONTROL, CONTROL_ENABLE);
}

static void device_reset(struct my_device *dev)
{
    REG_SET_BIT(dev, REG_CONTROL, CONTROL_RESET);
    
    // Wait for reset complete
    while (REG_READ(dev, REG_STATUS) & STATUS_READY)
        cpu_relax();
}

static bool device_is_ready(struct my_device *dev)
{
    return REG_READ(dev, REG_STATUS) & STATUS_READY;
}
```

## Complete Example

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/io.h>
#include <linux/ioport.h>

#define DEVICE_NAME "mmio_device"

// Register offsets
#define REG_ID          0x00
#define REG_CONTROL     0x04
#define REG_STATUS      0x08
#define REG_DATA        0x0C

// Control bits
#define CTRL_ENABLE     BIT(0)
#define CTRL_RESET      BIT(1)
#define CTRL_IRQ_EN     BIT(2)

// Status bits
#define STATUS_READY    BIT(0)
#define STATUS_BUSY     BIT(1)
#define STATUS_ERROR    BIT(2)

struct mmio_device {
    void __iomem *base;
    struct resource *mem_res;
};

static u32 mmio_read_reg(struct mmio_device *dev, unsigned int offset)
{
    return ioread32(dev->base + offset);
}

static void mmio_write_reg(struct mmio_device *dev, unsigned int offset, u32 value)
{
    iowrite32(value, dev->base + offset);
}

static int mmio_device_init(struct mmio_device *dev)
{
    u32 id;
    
    // Read device ID
    id = mmio_read_reg(dev, REG_ID);
    printk(KERN_INFO "Device ID: 0x%08x\n", id);
    
    // Reset device
    mmio_write_reg(dev, REG_CONTROL, CTRL_RESET);
    msleep(10);
    
    // Wait for ready
    while (!(mmio_read_reg(dev, REG_STATUS) & STATUS_READY)) {
        cpu_relax();
    }
    
    // Enable device
    mmio_write_reg(dev, REG_CONTROL, CTRL_ENABLE | CTRL_IRQ_EN);
    
    return 0;
}

static int mmio_probe(struct platform_device *pdev)
{
    struct mmio_device *dev;
    struct resource *res;
    int ret;
    
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;
    
    // Get memory resource
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res) {
        dev_err(&pdev->dev, "No memory resource\n");
        return -ENODEV;
    }
    
    // Request memory region
    dev->mem_res = devm_request_mem_region(&pdev->dev,
                                          res->start,
                                          resource_size(res),
                                          pdev->name);
    if (!dev->mem_res) {
        dev_err(&pdev->dev, "Memory region busy\n");
        return -EBUSY;
    }
    
    // Map memory
    dev->base = devm_ioremap(&pdev->dev, res->start, resource_size(res));
    if (!dev->base) {
        dev_err(&pdev->dev, "Failed to map memory\n");
        return -ENOMEM;
    }
    
    // Initialize device
    ret = mmio_device_init(dev);
    if (ret) {
        dev_err(&pdev->dev, "Device initialization failed\n");
        return ret;
    }
    
    platform_set_drvdata(pdev, dev);
    
    dev_info(&pdev->dev, "Device initialized at 0x%lx\n",
            (unsigned long)res->start);
    
    return 0;
}

static int mmio_remove(struct platform_device *pdev)
{
    struct mmio_device *dev = platform_get_drvdata(pdev);
    
    // Disable device
    mmio_write_reg(dev, REG_CONTROL, 0);
    
    dev_info(&pdev->dev, "Device removed\n");
    return 0;
}

static struct platform_driver mmio_driver = {
    .probe = mmio_probe,
    .remove = mmio_remove,
    .driver = {
        .name = DEVICE_NAME,
    },
};

module_platform_driver(mmio_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("MMIO device driver example");
```

## Checking I/O Resources

```bash
# View I/O port allocations
cat /proc/ioports

# View memory allocations
cat /proc/iomem

# Example output:
# 0000-0cf7 : PCI Bus 0000:00
# 0300-0307 : mydevice
```

## Best Practices

1. **Always request resources** before accessing
2. **Use ioremap** for memory-mapped I/O
3. **Use ioread/iowrite** functions, not direct pointer access
4. **Add memory barriers** when needed
5. **Release resources** in reverse order
6. **Use managed resources (devm_*)** when possible
7. **Check resource availability** before use
8. **Document register layout** clearly
9. **Use macros** for register access
10. **Test on target hardware**

## Common Errors

- Accessing I/O without requesting resources
- Using wrong access size (8/16/32-bit)
- Missing memory barriers
- Not unmapping ioremap'd memory
- Direct pointer dereference instead of ioread/iowrite
- Incorrect register offsets

## Next Steps

Proceed to [Linux Driver Model](12-linux-driver-model.md) to learn about the kernel's device driver model.
