# Kernel Memory Management

## Overview

Kernel memory management differs significantly from user space. This chapter covers memory allocation, DMA, and memory mapping in kernel drivers.

## Memory Zones

Linux divides physical memory into zones:

- **ZONE_DMA**: Memory for DMA (< 16MB on x86)
- **ZONE_DMA32**: DMA memory < 4GB (x86_64)
- **ZONE_NORMAL**: Normal memory
- **ZONE_HIGHMEM**: High memory (> 896MB on 32-bit)

## kmalloc

Allocates physically contiguous memory.

```c
#include <linux/slab.h>

void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *ptr);
```

### GFP Flags

```c
GFP_KERNEL      // Normal allocation, can sleep
GFP_ATOMIC      // Cannot sleep, for interrupt context
GFP_USER        // User space allocation
GFP_HIGHUSER    // High memory for user space
GFP_DMA         // DMA-capable memory
GFP_DMA32       // DMA memory < 4GB
```

### Common Combinations

```c
GFP_KERNEL      // Process context, can sleep
GFP_ATOMIC      // Interrupt context, cannot sleep
GFP_KERNEL | __GFP_ZERO  // Zero-initialized memory
```

### Example

```c
// Allocate memory
char *buffer = kmalloc(1024, GFP_KERNEL);
if (!buffer) {
    printk(KERN_ERR "Failed to allocate memory\n");
    return -ENOMEM;
}

// Use buffer
strcpy(buffer, "Hello");

// Free memory
kfree(buffer);
```

## kzalloc

Allocates zero-initialized memory.

```c
void *kzalloc(size_t size, gfp_t flags);

// Equivalent to:
void *ptr = kmalloc(size, flags);
if (ptr)
    memset(ptr, 0, size);
```

## vmalloc

Allocates virtually contiguous memory (may be physically fragmented).

```c
#include <linux/vmalloc.h>

void *vmalloc(unsigned long size);
void vfree(const void *addr);

// Example
char *buffer = vmalloc(1024 * 1024);  // 1MB
if (!buffer)
    return -ENOMEM;

// Use buffer
memset(buffer, 0, 1024 * 1024);

vfree(buffer);
```

### kmalloc vs vmalloc

| Feature | kmalloc | vmalloc |
|---------|---------|---------|
| Physical | Contiguous | Fragmented |
| Size | Limited | Large |
| Speed | Fast | Slower |
| Use case | Small allocations | Large buffers |

## Memory Pools

Pre-allocated memory for critical allocations.

```c
#include <linux/mempool.h>

mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn,
                         mempool_free_t *free_fn, void *pool_data);

// Using kmalloc
mempool_t *pool = mempool_create_kmalloc_pool(32, 1024);

// Allocate from pool
void *ptr = mempool_alloc(pool, GFP_KERNEL);

// Free to pool
mempool_free(ptr, pool);

// Destroy pool
mempool_destroy(pool);
```

## Slab Allocator

Efficient allocation for frequently used objects.

```c
#include <linux/slab.h>

struct kmem_cache *cache;

// Create cache
cache = kmem_cache_create("my_cache",
                         sizeof(struct my_object),
                         0,
                         SLAB_HWCACHE_ALIGN,
                         NULL);

// Allocate object
struct my_object *obj = kmem_cache_alloc(cache, GFP_KERNEL);

// Free object
kmem_cache_free(cache, obj);

// Destroy cache
kmem_cache_destroy(cache);
```

### Example

```c
struct my_data {
    int id;
    char name[32];
    void *ptr;
};

static struct kmem_cache *my_cache;

static int __init cache_init(void)
{
    my_cache = kmem_cache_create("my_data_cache",
                                sizeof(struct my_data),
                                0,
                                SLAB_HWCACHE_ALIGN,
                                NULL);
    if (!my_cache)
        return -ENOMEM;
    
    // Allocate objects
    struct my_data *obj1 = kmem_cache_alloc(my_cache, GFP_KERNEL);
    struct my_data *obj2 = kmem_cache_alloc(my_cache, GFP_KERNEL);
    
    // Use objects
    obj1->id = 1;
    strcpy(obj1->name, "Object 1");
    
    // Free objects
    kmem_cache_free(my_cache, obj1);
    kmem_cache_free(my_cache, obj2);
    
    return 0;
}

static void __exit cache_exit(void)
{
    kmem_cache_destroy(my_cache);
}
```

## Per-CPU Variables

Variables with separate instance per CPU.

```c
#include <linux/percpu.h>

// Static per-CPU variable
DEFINE_PER_CPU(int, my_counter);

// Access
int value = get_cpu_var(my_counter);
value++;
put_cpu_var(my_counter);

// Or with preemption disabled
int *ptr = per_cpu_ptr(&my_counter, cpu);

// Dynamic allocation
int __percpu *counter = alloc_percpu(int);

// Access
int *ptr = per_cpu_ptr(counter, smp_processor_id());
*ptr = 42;

// Free
free_percpu(counter);
```

## Page Allocation

Low-level page allocation.

```c
#include <linux/gfp.h>

// Allocate pages
struct page *page = alloc_pages(GFP_KERNEL, order);  // 2^order pages
void *addr = page_address(page);

// Allocate single page
unsigned long addr = __get_free_page(GFP_KERNEL);
unsigned long addr = get_zeroed_page(GFP_KERNEL);

// Free pages
free_pages(addr, order);
__free_page(page);
```

## DMA Memory

### Coherent DMA

```c
#include <linux/dma-mapping.h>

dma_addr_t dma_handle;
void *cpu_addr;

// Allocate coherent DMA memory
cpu_addr = dma_alloc_coherent(&pdev->dev, size, &dma_handle, GFP_KERNEL);
if (!cpu_addr)
    return -ENOMEM;

// Use memory
// cpu_addr: CPU address
// dma_handle: Device address

// Free memory
dma_free_coherent(&pdev->dev, size, cpu_addr, dma_handle);
```

### Streaming DMA

```c
// Map buffer for DMA
dma_addr_t dma_addr = dma_map_single(&pdev->dev, buffer, size, DMA_TO_DEVICE);

if (dma_mapping_error(&pdev->dev, dma_addr)) {
    printk(KERN_ERR "DMA mapping failed\n");
    return -ENOMEM;
}

// Perform DMA transfer
// ...

// Unmap buffer
dma_unmap_single(&pdev->dev, dma_addr, size, DMA_TO_DEVICE);
```

### DMA Direction

```c
DMA_TO_DEVICE       // Data to device
DMA_FROM_DEVICE     // Data from device
DMA_BIDIRECTIONAL   // Both directions
DMA_NONE            // No data transfer
```

### Example

```c
struct my_device {
    void *coherent_buf;
    dma_addr_t dma_handle;
    size_t buf_size;
};

static int setup_dma(struct platform_device *pdev, struct my_device *dev)
{
    dev->buf_size = 4096;
    
    // Set DMA mask
    if (dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32))) {
        dev_err(&pdev->dev, "Failed to set DMA mask\n");
        return -EINVAL;
    }
    
    // Allocate DMA buffer
    dev->coherent_buf = dma_alloc_coherent(&pdev->dev,
                                          dev->buf_size,
                                          &dev->dma_handle,
                                          GFP_KERNEL);
    if (!dev->coherent_buf)
        return -ENOMEM;
    
    dev_info(&pdev->dev, "DMA buffer at 0x%llx\n",
            (unsigned long long)dev->dma_handle);
    
    return 0;
}

static void cleanup_dma(struct platform_device *pdev, struct my_device *dev)
{
    if (dev->coherent_buf) {
        dma_free_coherent(&pdev->dev,
                         dev->buf_size,
                         dev->coherent_buf,
                         dev->dma_handle);
    }
}
```

## Memory Mapping (mmap)

Map kernel memory to user space.

```c
static int device_mmap(struct file *filp, struct vm_area_struct *vma)
{
    unsigned long size = vma->vm_end - vma->vm_start;
    unsigned long pfn;
    
    // Check size
    if (size > BUFFER_SIZE)
        return -EINVAL;
    
    // Get page frame number
    pfn = virt_to_phys(kernel_buffer) >> PAGE_SHIFT;
    
    // Map pages
    if (remap_pfn_range(vma,
                       vma->vm_start,
                       pfn,
                       size,
                       vma->vm_page_prot))
        return -EAGAIN;
    
    return 0;
}

// For DMA memory
static int device_mmap_dma(struct file *filp, struct vm_area_struct *vma)
{
    struct my_device *dev = filp->private_data;
    unsigned long size = vma->vm_end - vma->vm_start;
    
    return dma_mmap_coherent(&dev->pdev->dev,
                            vma,
                            dev->coherent_buf,
                            dev->dma_handle,
                            size);
}
```

### User Space Usage

```c
#include <sys/mman.h>

int fd = open("/dev/mydevice", O_RDWR);

void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);
if (addr == MAP_FAILED) {
    perror("mmap");
    return 1;
}

// Access mapped memory
char *buf = (char *)addr;
strcpy(buf, "Hello from user space");

munmap(addr, 4096);
close(fd);
```

## Memory Barriers

Ensure memory operation ordering.

```c
#include <linux/barrier.h>

// Full memory barrier
mb();

// Read memory barrier
rmb();

// Write memory barrier
wmb();

// SMP barriers
smp_mb();
smp_rmb();
smp_wmb();

// Example
writel(value, reg);
wmb();  // Ensure write completes before next operation
```

## Managed Resources (devm_*)

Automatic cleanup on driver detach.

```c
// Managed kmalloc
void *ptr = devm_kmalloc(&pdev->dev, size, GFP_KERNEL);
void *ptr = devm_kzalloc(&pdev->dev, size, GFP_KERNEL);

// Managed DMA
void *cpu_addr = dmam_alloc_coherent(&pdev->dev, size, &dma_handle, GFP_KERNEL);

// No need to free - automatically freed on driver detach
```

## Memory Debugging

### Kernel Configuration

```
CONFIG_DEBUG_SLAB          # Slab debugging
CONFIG_DEBUG_PAGEALLOC     # Page allocation debugging
CONFIG_KASAN               # Address sanitizer
CONFIG_SLUB_DEBUG          # SLUB debugging
```

### Detecting Leaks

```bash
# Check slab info
cat /proc/slabinfo

# Memory info
cat /proc/meminfo

# Check for leaks
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

## Best Practices

1. **Always check allocation return value**
2. **Free memory in reverse order of allocation**
3. **Use appropriate allocation function** for context
4. **Prefer managed resources (devm_*)** when possible
5. **Use GFP_ATOMIC** in interrupt context
6. **Avoid large kmalloc** allocations (use vmalloc)
7. **Set DMA mask** before DMA operations
8. **Use memory barriers** for hardware access
9. **Test with memory debugging** enabled

## Common Errors

- Memory leaks (not freeing allocated memory)
- Use after free
- Double free
- Buffer overflow
- Wrong GFP flags for context
- Not checking allocation failure

## Complete Example

```c
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/dma-mapping.h>

struct my_device {
    char *kmalloc_buf;
    char *vmalloc_buf;
    void *dma_buf;
    dma_addr_t dma_handle;
    struct kmem_cache *cache;
};

static struct my_device *dev;

static int __init mem_init(void)
{
    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;
    
    // kmalloc
    dev->kmalloc_buf = kmalloc(4096, GFP_KERNEL);
    if (!dev->kmalloc_buf)
        goto err_kmalloc;
    
    // vmalloc
    dev->vmalloc_buf = vmalloc(1024 * 1024);
    if (!dev->vmalloc_buf)
        goto err_vmalloc;
    
    // Slab cache
    dev->cache = kmem_cache_create("my_cache", 128, 0, 0, NULL);
    if (!dev->cache)
        goto err_cache;
    
    printk(KERN_INFO "Memory allocated successfully\n");
    return 0;

err_cache:
    vfree(dev->vmalloc_buf);
err_vmalloc:
    kfree(dev->kmalloc_buf);
err_kmalloc:
    kfree(dev);
    return -ENOMEM;
}

static void __exit mem_exit(void)
{
    if (dev) {
        if (dev->cache)
            kmem_cache_destroy(dev->cache);
        if (dev->vmalloc_buf)
            vfree(dev->vmalloc_buf);
        if (dev->kmalloc_buf)
            kfree(dev->kmalloc_buf);
        kfree(dev);
    }
    
    printk(KERN_INFO "Memory freed\n");
}

module_init(mem_init);
module_exit(mem_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Memory management example");
```

## Next Steps

Proceed to [IO Port Access](11-io-port-access.md) to learn about accessing hardware I/O ports and memory-mapped I/O.
