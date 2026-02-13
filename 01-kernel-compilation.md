# Linux Kernel Compilation

## 🎯 Layman's Explanation

**What is the Kernel?**
Think of the kernel as the **manager of a restaurant**. It coordinates everything: takes orders (user requests), manages the kitchen (CPU), handles ingredients (memory), and serves food (data). Without the manager, chaos!

**Why Compile the Kernel?**
Imagine buying a pre-made pizza vs making your own. Pre-made = convenient but fixed toppings. Making your own = you choose exactly what you want. Compiling the kernel lets you:
- Add only features you need (smaller, faster)
- Remove unnecessary stuff (security)
- Add custom hardware support
- Experiment and learn

**The Process in Simple Terms:**
```
Download Recipe (Source Code)
    ↓
Choose Ingredients (Configuration)
    ↓
Mix & Bake (Compilation)
    ↓
Serve (Install & Boot)
```

## Overview

Kernel compilation is the process of building the Linux kernel from source code. This chapter covers downloading, configuring, and compiling the Linux kernel.

## Prerequisites

- Build tools: gcc, make, binutils
- Development libraries
- Sufficient disk space (15-20 GB)
- Root or sudo access

## Installing Build Dependencies

### Debian/Ubuntu
```bash
sudo apt-get update
sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev
```

### RHEL/CentOS/Fedora
```bash
sudo yum groupinstall "Development Tools"
sudo yum install ncurses-devel bison flex elfutils-libelf-devel openssl-devel
```

## Downloading Kernel Source

### Method 1: Using Git
```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

### Method 2: Download Tarball
```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.x.tar.xz
tar -xf linux-6.x.tar.xz
cd linux-6.x
```

## Kernel Configuration

### Using Existing Configuration
```bash
# Copy current system config
cp /boot/config-$(uname -r) .config
make oldconfig
```

### Configuration Methods

#### 1. menuconfig (Text-based UI)
```bash
make menuconfig
```
- Navigate with arrow keys
- Space to select/deselect
- '/' to search
- Save and exit

#### 2. xconfig (Qt-based GUI)
```bash
make xconfig
```

#### 3. gconfig (GTK-based GUI)
```bash
make gconfig
```

#### 4. Default Configuration
```bash
make defconfig        # Default config
make allmodconfig     # All modules
make allyesconfig     # Everything built-in
```

## Important Configuration Options

### General Setup
- `CONFIG_LOCALVERSION`: Custom kernel version string
- `CONFIG_DEFAULT_HOSTNAME`: Default hostname

### Processor Type and Features
- `CONFIG_SMP`: Symmetric multi-processing support
- `CONFIG_NR_CPUS`: Maximum number of CPUs

### Device Drivers
- Enable drivers for your hardware
- Built-in (Y) vs Module (M)

### File Systems
- Enable required filesystem support
- ext4, btrfs, xfs, etc.

## Compiling the Kernel

### Determine Number of CPU Cores
```bash
nproc
```

### Compile Kernel
```bash
# Clean previous builds (optional)
make clean
make mrproper  # Complete clean

# Compile (use -j for parallel jobs)
make -j$(nproc)
```

### Compile Modules
```bash
make modules -j$(nproc)
```

### Compilation Time
- Depends on hardware and configuration
- Typically 30 minutes to 2 hours

## Installing the Kernel

### Install Modules
```bash
sudo make modules_install
```
- Installs to `/lib/modules/<kernel-version>/`

### Install Kernel
```bash
sudo make install
```

This command:
1. Copies kernel to `/boot/vmlinuz-<version>`
2. Copies System.map to `/boot/`
3. Creates initramfs
4. Updates bootloader configuration

### Manual Installation

#### Copy Kernel Image
```bash
sudo cp arch/x86/boot/bzImage /boot/vmlinuz-<version>
sudo cp System.map /boot/System.map-<version>
```

#### Generate initramfs
```bash
# Debian/Ubuntu
sudo update-initramfs -c -k <version>

# RHEL/CentOS
sudo dracut /boot/initramfs-<version>.img <version>
```

#### Update Bootloader

**GRUB2:**
```bash
sudo update-grub           # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg  # RHEL/CentOS
```

## Booting the New Kernel

### Reboot System
```bash
sudo reboot
```

### Select Kernel at Boot
- GRUB menu appears during boot
- Select your new kernel version
- Press Enter

### Verify Kernel Version
```bash
uname -r
```

## Kernel Configuration Files

### Important Files
- `.config`: Current kernel configuration
- `System.map`: Kernel symbol table
- `vmlinux`: Uncompressed kernel (ELF)
- `arch/x86/boot/bzImage`: Compressed bootable kernel

## Cross-Compilation

### For ARM Architecture
```bash
# Install cross-compiler
sudo apt-get install gcc-arm-linux-gnueabihf

# Set architecture and cross-compiler
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

# Configure and compile
make defconfig
make -j$(nproc)
```

### For ARM64
```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make defconfig
make -j$(nproc)
```

## Kernel Build Artifacts

```
vmlinux              # Uncompressed kernel
arch/*/boot/bzImage  # Compressed bootable image
System.map           # Symbol table
.config              # Configuration file
modules              # Loadable kernel modules
```

## Troubleshooting

### Compilation Errors

**Missing dependencies:**
```bash
# Install missing packages based on error messages
sudo apt-get install <package-name>
```

**Out of memory:**
```bash
# Reduce parallel jobs
make -j2
```

### Boot Issues

**Kernel panic:**
- Boot into old kernel from GRUB menu
- Check kernel logs: `dmesg`
- Verify configuration

**Missing modules:**
```bash
# Reinstall modules
sudo make modules_install
```

## Cleaning Build Files

```bash
make clean          # Remove most generated files
make mrproper       # Remove all generated files including .config
make distclean      # mrproper + remove editor backup files
```

## Kernel Versioning

### Version Format
```
Major.Minor.Patch[-ExtraVersion]
Example: 6.1.25-custom
```

### Set Custom Version
```bash
# In .config or via menuconfig
CONFIG_LOCALVERSION="-custom"
```

## Best Practices

1. **Backup current kernel** before installing new one
2. **Keep old kernel** in bootloader for fallback
3. **Test in VM** before production systems
4. **Document changes** to configuration
5. **Use version control** for custom patches
6. **Enable only needed features** for smaller, faster kernel

## Useful Commands

```bash
# Show kernel build version
cat /proc/version

# Show loaded modules
lsmod

# Show kernel messages
dmesg

# Show kernel configuration
cat /proc/config.gz | gunzip

# Module information
modinfo <module_name>
```

## Next Steps

After successfully compiling and booting your custom kernel, proceed to [Embedded Linux](02-embedded-linux.md) to learn about kernel configuration for embedded systems.
