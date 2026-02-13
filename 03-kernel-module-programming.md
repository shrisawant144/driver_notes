# Linux Kernel Module Programming

## 🎯 Layman's Explanation

**What is a Kernel Module?**
Imagine your phone. You can install/uninstall apps without reinstalling the entire OS, right? Kernel modules are like **apps for the kernel** - you can add features (like support for a new device) without rebuilding the whole kernel.

**Real-World Analogy:**
Your car has a basic engine, but you can:
- Plug in a dashcam (module loaded)
- Remove it when not needed (module unloaded)
- Engine keeps running (kernel doesn't reboot)

**Why Modules Matter:**
Without modules, every driver would be built into the kernel = huge, slow, wasteful.
With modules = load only what you need, when you need it.

**The Module Lifecycle:**
```
Write Code (.c file)
    ↓
Compile (make) → Creates .ko file
    ↓
Load (insmod/modprobe) → Module enters kernel
    ↓
Module Running → Does its job
    ↓
Unload (rmmod) → Module removed from kernel
```

**Key Concept - Kernel Space vs User Space:**
```
┌─────────────────────────────────┐
│   User Space (Your Apps)        │ ← Safe, protected
│   - Can crash without harm      │
├─────────────────────────────────┤
│   Kernel Space (Modules)        │ ← Powerful, dangerous
│   - Full hardware access        │
│   - Bug = system crash          │
└─────────────────────────────────┘
```

**Think of it like:**
- User space = Customers in a restaurant (can't enter kitchen)
- Kernel space = Kitchen staff (full access, but mistakes affect everyone)

## Overview

Kernel modules are pieces of code that can be loaded and unloaded into the kernel on demand, extending kernel functionality without rebooting. This chapter covers the fundamentals of writing, compiling, and managing kernel modules.

## What is a Kernel Module?

- Dynamically loadable code that runs in kernel space
- Extends kernel functionality without recompilation
- Can be loaded/unloaded at runtime
- Has full access to kernel resources
- No memory protection (bugs can crash system)

## Advantages of Modules

- **Dynamic loading**: Add features without reboot
- **Reduced memory**: Load only needed modules
- **Easier development**: Test without kernel rebuild
- **Maintainability**: Modular code organization

## Hello World Module

### Basic Module Structure

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, World!\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, World!\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple Hello World module");
MODULE_VERSION("1.0");
```

### Makefile

```makefile
obj-m += hello.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### Building the Module

```bash
make
```

Output files:
- `hello.ko`: Kernel object (loadable module)
- `hello.mod.c`: Generated module metadata
- `hello.o`: Object file

### Loading and Unloading

```bash
# Load module
sudo insmod hello.ko

# View kernel messages
dmesg | tail

# List loaded modules
lsmod | grep hello

# Module information
modinfo hello.ko

# Unload module
sudo rmmod hello
```

## Module Macros

### Initialization and Cleanup

```c
module_init(init_function);  // Called when module loaded
module_exit(exit_function);  // Called when module unloaded
```

### Module Information

```c
MODULE_LICENSE("GPL");              // License type
MODULE_AUTHOR("Author Name");       // Author
MODULE_DESCRIPTION("Description");  // Description
MODULE_VERSION("1.0");              // Version
MODULE_ALIAS("alias_name");         // Alias name
```

### License Types
- `"GPL"`: GNU Public License
- `"GPL v2"`: GPL version 2
- `"Dual BSD/GPL"`: Dual licensed
- `"Proprietary"`: Closed source (taints kernel)

## Kernel Logging

### printk Function

```c
printk(KERN_LEVEL "message");
```

### Log Levels

```c
KERN_EMERG    // System is unusable
KERN_ALERT    // Action must be taken immediately
KERN_CRIT     // Critical conditions
KERN_ERR      // Error conditions
KERN_WARNING  // Warning conditions
KERN_NOTICE   // Normal but significant
KERN_INFO     // Informational
KERN_DEBUG    // Debug-level messages
```

### Example

```c
printk(KERN_INFO "Module loaded successfully\n");
printk(KERN_ERR "Error: Failed to allocate memory\n");
printk(KERN_DEBUG "Debug: Variable value = %d\n", value);
```

### Viewing Logs

```bash
dmesg                    # View all kernel messages
dmesg | tail -20         # Last 20 messages
dmesg -w                 # Watch mode (follow)
dmesg -l err             # Only errors
journalctl -k            # Kernel messages (systemd)
```

## Module Parameters

Allow passing arguments to modules at load time.

### Defining Parameters

```c
#include <linux/moduleparam.h>

static int count = 1;
static char *name = "default";

module_param(count, int, S_IRUGO);
MODULE_PARM_DESC(count, "Number of iterations");

module_param(name, charp, S_IRUGO);
MODULE_PARM_DESC(name, "Name to display");

static int __init param_init(void)
{
    int i;
    for (i = 0; i < count; i++) {
        printk(KERN_INFO "Hello %s (%d/%d)\n", name, i+1, count);
    }
    return 0;
}
```

### Parameter Types

```c
module_param(name, type, permissions);
```

Types:
- `int`, `uint`: Integer
- `long`, `ulong`: Long integer
- `short`, `ushort`: Short integer
- `bool`: Boolean
- `charp`: Character pointer (string)

### Permissions

```c
S_IRUGO  // Read by all
S_IWUSR  // Write by owner
S_IRUSR  // Read by owner
0        // No sysfs entry
```

### Loading with Parameters

```bash
sudo insmod module.ko count=5 name="Linux"
```

### Viewing Parameters

```bash
# Via sysfs
cat /sys/module/module_name/parameters/count

# Via modinfo
modinfo module.ko
```

## Module Dependencies

### Exporting Symbols

```c
// module1.c
int shared_function(int x)
{
    return x * 2;
}
EXPORT_SYMBOL(shared_function);

int shared_var = 100;
EXPORT_SYMBOL(shared_var);
```

### Using Exported Symbols

```c
// module2.c
extern int shared_function(int x);
extern int shared_var;

static int __init mod2_init(void)
{
    int result = shared_function(5);
    printk(KERN_INFO "Result: %d, Var: %d\n", result, shared_var);
    return 0;
}
```

### Export Types

```c
EXPORT_SYMBOL(symbol);         // Export to all modules
EXPORT_SYMBOL_GPL(symbol);     // Export only to GPL modules
```

## __init and __exit Macros

### Purpose
- Mark functions for special handling
- Memory optimization

### __init Macro

```c
static int __init my_init(void)
{
    // Initialization code
    return 0;
}
```

- Memory freed after initialization
- Used for one-time setup code

### __exit Macro

```c
static void __exit my_exit(void)
{
    // Cleanup code
}
```

- Omitted if module built into kernel
- Used for cleanup code

### __initdata and __exitdata

```c
static int __initdata init_value = 100;
static char __exitdata exit_msg[] = "Goodbye";
```

## Error Handling

### Return Values

```c
static int __init my_init(void)
{
    int ret;
    
    ret = some_function();
    if (ret < 0) {
        printk(KERN_ERR "Initialization failed: %d\n", ret);
        return ret;  // Return error code
    }
    
    return 0;  // Success
}
```

### Common Error Codes

```c
-ENOMEM   // Out of memory
-EINVAL   // Invalid argument
-EBUSY    // Device or resource busy
-EIO      // I/O error
-ENODEV   // No such device
-EFAULT   // Bad address
-EPERM    // Operation not permitted
```

### Cleanup on Error

```c
static int __init my_init(void)
{
    int ret;
    void *ptr1, *ptr2;
    
    ptr1 = kmalloc(100, GFP_KERNEL);
    if (!ptr1)
        return -ENOMEM;
    
    ptr2 = kmalloc(200, GFP_KERNEL);
    if (!ptr2) {
        ret = -ENOMEM;
        goto err_ptr2;
    }
    
    return 0;

err_ptr2:
    kfree(ptr1);
    return ret;
}
```

## Module Utilities

### insmod
```bash
sudo insmod module.ko [parameters]
```
- Loads module
- No dependency resolution

### rmmod
```bash
sudo rmmod module_name
```
- Unloads module
- Fails if module in use

### modprobe
```bash
sudo modprobe module_name [parameters]
```
- Loads module with dependencies
- Searches standard locations

```bash
sudo modprobe -r module_name
```
- Removes module and unused dependencies

### lsmod
```bash
lsmod
```
- Lists loaded modules
- Shows size and usage count

### modinfo
```bash
modinfo module.ko
```
- Shows module information
- Parameters, author, license, etc.

### depmod
```bash
sudo depmod -a
```
- Generates module dependency database
- Updates `/lib/modules/$(uname -r)/modules.dep`

## Module Locations

```
/lib/modules/$(uname -r)/
├── kernel/           # Kernel modules
│   ├── drivers/
│   ├── fs/
│   ├── net/
│   └── ...
├── modules.dep       # Dependency information
├── modules.alias     # Module aliases
└── modules.symbols   # Exported symbols
```

## Advanced Module Example

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/moduleparam.h>

static int debug = 0;
static int count = 1;
static char *message = "Hello";

module_param(debug, int, S_IRUGO | S_IWUSR);
MODULE_PARM_DESC(debug, "Enable debug messages");

module_param(count, int, S_IRUGO);
MODULE_PARM_DESC(count, "Number of times to print message");

module_param(message, charp, S_IRUGO);
MODULE_PARM_DESC(message, "Message to print");

static int __init advanced_init(void)
{
    int i;
    
    printk(KERN_INFO "Module loading...\n");
    
    if (debug)
        printk(KERN_DEBUG "Debug mode enabled\n");
    
    if (count <= 0) {
        printk(KERN_ERR "Invalid count: %d\n", count);
        return -EINVAL;
    }
    
    for (i = 0; i < count; i++) {
        printk(KERN_INFO "[%d/%d] %s\n", i+1, count, message);
    }
    
    printk(KERN_INFO "Module loaded successfully\n");
    return 0;
}

static void __exit advanced_exit(void)
{
    if (debug)
        printk(KERN_DEBUG "Cleaning up...\n");
    
    printk(KERN_INFO "Module unloaded\n");
}

module_init(advanced_init);
module_exit(advanced_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Advanced module example");
MODULE_VERSION("1.0");
```

## Multiple Source Files

### Makefile for Multiple Files

```makefile
obj-m += mymodule.o
mymodule-objs := main.o helper.o utils.o

KDIR := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

## Kernel Version Compatibility

### Checking Kernel Version

```c
#include <linux/version.h>

#if LINUX_VERSION_CODE >= KERNEL_VERSION(5,0,0)
    // Code for kernel 5.0+
#else
    // Code for older kernels
#endif
```

## Best Practices

1. **Always check return values**
2. **Clean up resources on error**
3. **Use appropriate log levels**
4. **Document module parameters**
5. **Handle module unload gracefully**
6. **Avoid blocking operations in init**
7. **Use GPL license when possible**
8. **Test thoroughly before deployment**
9. **Keep modules small and focused**
10. **Follow kernel coding style**

## Common Pitfalls

- **Memory leaks**: Always free allocated memory
- **Unbalanced locks**: Match every lock with unlock
- **Sleeping in atomic context**: No blocking in interrupt handlers
- **Race conditions**: Proper synchronization needed
- **Unregistered resources**: Clean up everything in exit function

## Debugging Modules

```bash
# Enable dynamic debug
echo "module mymodule +p" > /sys/kernel/debug/dynamic_debug/control

# Check module load errors
dmesg | grep -i error

# Verify module loaded
lsmod | grep mymodule

# Check module usage
cat /proc/modules
```

## Next Steps

Proceed to [Linux Device Driver Programming](04-device-driver-programming.md) to learn about writing device drivers.
