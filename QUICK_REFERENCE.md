# 🎯 Quick Reference Card - Driver Development Analogies

## One-Line Analogies for Every Concept

| Concept | Analogy | Remember This |
|---------|---------|---------------|
| **Kernel** | Restaurant manager | Coordinates everything |
| **Kernel Module** | Phone app | Install/uninstall without reboot |
| **Device Driver** | Translator | Hardware ↔ Software |
| **Character Device** | Water hose | Stream of bytes |
| **Block Device** | Boxes on shelf | Chunks of data |
| **Pseudo Device** | Flight simulator | Fake but feels real |
| **Platform Device** | Built-in furniture | Always there, not pluggable |
| **Device Tree** | Instruction manual | "Hardware is at address X" |
| **Interrupt** | Phone ringing | "URGENT! Deal with me NOW!" |
| **Top Half** | Grab package at door | Fast, urgent |
| **Bottom Half** | Open package later | Slower, can take time |
| **Spinlock** | Keep trying door | Busy-wait, fast |
| **Mutex** | Wait to be called | Sleep-wait, slower |
| **Semaphore** | Multiple allowed | 3 people in bathroom |
| **Race Condition** | Shared bank account | Two people, same data, chaos |
| **Deadlock** | Car keys standoff | Both wait forever |
| **kmalloc** | Parking in one row | Physically continuous |
| **vmalloc** | Parking scattered | Virtually continuous |
| **GFP_KERNEL** | Regular shipping | Can wait |
| **GFP_ATOMIC** | Express overnight | Need it NOW |
| **DMA** | Direct delivery | Hardware → Memory (no CPU) |
| **ioremap** | GPS to address | Physical → Virtual |
| **Memory Barrier** | "Wait! Finish first!" | Enforce order |
| **Port I/O** | PO Box | Separate address space |
| **Memory-Mapped I/O** | Home mailbox | Part of memory |
| **sysfs** | Company directory | Browse device info |
| **udev** | Office admin | Creates /dev files |
| **printk** | Breadcrumbs | Trace where you've been |
| **dmesg** | Logbook | Read what happened |
| **Oops** | Non-fatal crash | System survives |
| **Panic** | Fatal crash | System dies |
| **Jiffies** | Heartbeat | Kernel's clock tick |
| **udelay** | Stand at door | Busy-wait |
| **msleep** | Sit on couch | Sleep-wait |
| **Timer** | Kitchen timer | Do something later |
| **Workqueue** | To-do inbox | Process when convenient |
| **USB** | Conference call | Multiple devices, one line |
| **VID/PID** | Company ID/Employee ID | Device identity |
| **URB** | Delivery package | USB data transfer |
| **GPIO** | Light switch | On/off control |
| **I2C** | Party line phone | Shared, multiple devices |
| **SPI** | Private phone line | Fast, dedicated |
| **Probe** | Employee onboarding | Driver meets device |
| **Remove** | Employee leaving | Device unplugged |

---

## Quick Decision Trees

### Which Lock?
```
Interrupt context? → Spinlock
Can sleep? → Mutex
Multiple allowed? → Semaphore
Simple counter? → Atomic
```

### Which Memory?
```
< 128KB? → kmalloc
> 128KB? → vmalloc
Interrupt? → GFP_ATOMIC
Normal? → GFP_KERNEL
```

### Which Delay?
```
< 1ms? → udelay
1-10ms? → mdelay or msleep
> 10ms? → msleep
Interrupt? → Only udelay/mdelay
```

### Which Protocol?
```
Simple on/off? → GPIO
Multiple sensors? → I2C
High-speed data? → SPI
Hot-pluggable? → USB
```

---

## Memory Tricks

### Kernel Space vs User Space
```
User Space = Restaurant customers (safe, protected)
Kernel Space = Kitchen staff (powerful, dangerous)
```

### Context Rules
```
Process Context:
  ✅ Can sleep
  ✅ Can use mutex
  ✅ Can use GFP_KERNEL
  
Interrupt Context:
  ❌ Cannot sleep
  ❌ Cannot use mutex
  ✅ Must use spinlock
  ✅ Must use GFP_ATOMIC
```

### Copy Rules
```
Kernel → User: copy_to_user()
User → Kernel: copy_from_user()
Never access user memory directly!
```

---

## Common Patterns

### Driver Init
```
module_init → Allocate → Register → Setup → Ready
```

### Driver Exit
```
module_exit → Stop → Unregister → Free → Done
```

### Read Operation
```
User read() → Kernel → Driver → Hardware → Data back
```

### Interrupt
```
Hardware → CPU → Top Half → Schedule → Bottom Half
```

---

## Golden Rules

1. **Always check allocation** - Can fail!
2. **Free what you allocate** - No leaks
3. **Can't sleep in interrupt** - Use GFP_ATOMIC
4. **Lock in same order** - Prevent deadlock
5. **Use copy_to/from_user** - Never direct access
6. **Check return values** - Handle errors
7. **Use devm_* functions** - Auto cleanup
8. **Keep locks short** - Don't hold long
9. **Test in VM first** - Kernel bugs crash system
10. **Read dmesg often** - Your best friend

---

## Essential Commands

```bash
# Module operations
insmod mydriver.ko      # Load
rmmod mydriver          # Unload
lsmod                   # List
modinfo mydriver.ko     # Info

# Debugging
dmesg | tail            # Recent messages
dmesg -w                # Watch live
dmesg -c                # Clear buffer

# Device info
cat /proc/devices       # Character/block devices
cat /proc/interrupts    # Interrupt stats
cat /proc/iomem         # Memory map
ls /sys/class/          # Device classes

# Hardware
lsusb                   # USB devices
lspci                   # PCI devices
lsmod                   # Loaded modules
```

---

## Common Mistakes

| Mistake | Why Bad | Fix |
|---------|---------|-----|
| `kmalloc` in interrupt with `GFP_KERNEL` | Can sleep | Use `GFP_ATOMIC` |
| Sleep while holding spinlock | Deadlock | Use mutex instead |
| Forget to free memory | Memory leak | Always `kfree()` |
| Direct user memory access | Security/crash | Use `copy_to/from_user()` |
| Not checking allocation | NULL pointer crash | Always check `if (!ptr)` |
| Wrong lock order | Deadlock | Always same order |
| Long critical section | Performance | Keep locks short |

---

## File Locations

```
/sys/                   # sysfs (device info)
/dev/                   # Device files
/proc/                  # Process/kernel info
/lib/modules/           # Kernel modules
/usr/src/linux/         # Kernel source
```

---

## Quick Syntax

```c
// Memory
ptr = kmalloc(size, GFP_KERNEL);
kfree(ptr);

// Copy
copy_to_user(to, from, size);
copy_from_user(to, from, size);

// Locks
spin_lock(&lock);
spin_unlock(&lock);
mutex_lock(&mutex);
mutex_unlock(&mutex);

// Interrupts
request_irq(irq, handler, flags, name, dev);
free_irq(irq, dev);

// Time
msleep(ms);
udelay(us);

// I/O
readl(addr);
writel(val, addr);
```

---

## Learning Path (12 Weeks)

```
Week 1-2:  Foundation (Kernel, Embedded, Modules)
Week 3-4:  Drivers (Device, Pseudo, Platform)
Week 5-6:  Core (Data, Interrupts, Sync)
Week 7-8:  Advanced (Memory, I/O, Model)
Week 9-10: Practical (Debug, Time)
Week 11-12: Hardware (USB, GPIO/SPI/I2C)
```

---

## When Stuck

1. Check `dmesg` - What does kernel say?
2. Check return values - Did function fail?
3. Check context - Interrupt or process?
4. Check locks - Holding too long?
5. Check memory - Freed? Leaked?
6. Read existing drivers - How do they do it?
7. Ask community - Stack Overflow, mailing lists

---

## Success Checklist

- ✅ Understand kernel space vs user space
- ✅ Can write basic module
- ✅ Know when to use which lock
- ✅ Understand interrupt handling
- ✅ Can allocate memory correctly
- ✅ Know how to debug with printk/dmesg
- ✅ Understand device model
- ✅ Can write platform driver
- ✅ Know GPIO/I2C/SPI basics
- ✅ Built at least one real driver

---

**Print this card and keep it handy while coding!** 📋

Happy Driver Development! 🚀
