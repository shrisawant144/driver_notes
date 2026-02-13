# Embedded Linux

## Overview

Embedded Linux refers to the use of Linux operating system in embedded systems - dedicated computer systems designed for specific functions within larger systems. This chapter covers building and customizing Linux for embedded devices.

## Characteristics of Embedded Systems

- **Resource Constrained**: Limited CPU, memory, storage
- **Real-time Requirements**: Deterministic response times
- **Power Efficiency**: Battery-powered devices
- **Reliability**: Long-term operation without intervention
- **Cost Sensitive**: Optimized hardware costs
- **Application Specific**: Dedicated functionality

## Embedded Linux Architecture

```
+----------------------------------+
|     Application Layer            |
|  (User space applications)       |
+----------------------------------+
|     System Libraries             |
|  (glibc, uClibc, musl)          |
+----------------------------------+
|     Linux Kernel                 |
|  (Drivers, Subsystems)          |
+----------------------------------+
|     Bootloader                   |
|  (U-Boot, GRUB)                 |
+----------------------------------+
|     Hardware                     |
+----------------------------------+
```

## Build Systems

### Buildroot

Lightweight build system for embedded Linux.

#### Installation
```bash
git clone https://github.com/buildroot/buildroot.git
cd buildroot
```

#### Configuration
```bash
make menuconfig
```

Key configurations:
- Target architecture
- Toolchain options
- Kernel version
- Root filesystem type
- Package selection

#### Build
```bash
make -j$(nproc)
```

Output in `output/images/`:
- `rootfs.tar`: Root filesystem
- `zImage`: Kernel image
- `*.dtb`: Device tree blobs

### Yocto Project

Industrial-grade build system for custom Linux distributions.

#### Installation
```bash
git clone git://git.yoctoproject.org/poky
cd poky
```

#### Setup Environment
```bash
source oe-init-build-env
```

#### Configuration
Edit `conf/local.conf`:
```
MACHINE = "qemuarm"
```

#### Build
```bash
bitbake core-image-minimal
```

### OpenEmbedded

Metadata and build framework used by Yocto.

## Cross-Compilation Toolchain

### Components
- **Cross-compiler**: gcc for target architecture
- **Cross-linker**: ld for target
- **C library**: glibc, uClibc, musl
- **Binary utilities**: objdump, strip, etc.

### Installing Toolchain

#### Using Package Manager
```bash
# For ARM
sudo apt-get install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

# For ARM64
sudo apt-get install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
```

#### Using Crosstool-NG
```bash
git clone https://github.com/crosstool-ng/crosstool-ng
cd crosstool-ng
./bootstrap
./configure --prefix=/opt/crosstool-ng
make
sudo make install

# Configure and build toolchain
ct-ng menuconfig
ct-ng build
```

### Using Toolchain
```bash
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=/path/to/toolchain/bin:$PATH
```

## Bootloaders

### U-Boot (Universal Bootloader)

Most popular bootloader for embedded Linux.

#### Download and Build
```bash
git clone https://github.com/u-boot/u-boot.git
cd u-boot

# Configure for your board
make <board_name>_defconfig

# Build
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)
```

#### U-Boot Commands
```
# Environment variables
printenv
setenv
saveenv

# Memory operations
md    # Memory display
mm    # Memory modify
cp    # Copy memory

# Boot commands
bootm  # Boot from memory
bootz  # Boot zImage
bootelf # Boot ELF image

# Storage
fatload  # Load from FAT filesystem
ext4load # Load from ext4
mmc      # MMC/SD commands
```

#### Boot Process
```
1. Power-on/Reset
2. ROM bootloader (SoC internal)
3. SPL (Secondary Program Loader)
4. U-Boot proper
5. Load kernel and device tree
6. Boot Linux kernel
```

### Barebox

Alternative bootloader with shell-like interface.

## Device Tree

Hardware description format for embedded systems.

### Device Tree Structure
```dts
/dts-v1/;

/ {
    model = "My Embedded Board";
    compatible = "vendor,board";
    
    cpus {
        #address-cells = <1>;
        #size-cells = <0>;
        
        cpu@0 {
            device_type = "cpu";
            compatible = "arm,cortex-a9";
            reg = <0>;
        };
    };
    
    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x20000000>; /* 512MB */
    };
    
    uart0: serial@10000000 {
        compatible = "vendor,uart";
        reg = <0x10000000 0x1000>;
        interrupts = <0 1 4>;
    };
    
    i2c0: i2c@20000000 {
        compatible = "vendor,i2c";
        reg = <0x20000000 0x1000>;
        #address-cells = <1>;
        #size-cells = <0>;
        
        eeprom@50 {
            compatible = "atmel,24c02";
            reg = <0x50>;
        };
    };
};
```

### Compiling Device Tree
```bash
# Compile .dts to .dtb
dtc -I dts -O dtb -o board.dtb board.dts

# Decompile .dtb to .dts
dtc -I dtb -O dts -o board.dts board.dtb
```

### Device Tree Overlays
```dts
/dts-v1/;
/plugin/;

/ {
    fragment@0 {
        target = <&i2c0>;
        __overlay__ {
            sensor@48 {
                compatible = "ti,tmp102";
                reg = <0x48>;
            };
        };
    };
};
```

## Root Filesystem

### Minimal Root Filesystem Structure
```
/
├── bin/        # Essential binaries
├── sbin/       # System binaries
├── etc/        # Configuration files
├── dev/        # Device files
├── proc/       # Process information (virtual)
├── sys/        # System information (virtual)
├── tmp/        # Temporary files
├── usr/        # User programs
│   ├── bin/
│   ├── lib/
│   └── sbin/
├── var/        # Variable data
└── lib/        # Shared libraries
```

### Creating Minimal Rootfs

#### Using BusyBox
```bash
# Download and configure
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar xf busybox-1.36.0.tar.bz2
cd busybox-1.36.0

# Configure
make menuconfig
# Enable: Build static binary

# Build
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- install

# Create rootfs structure
mkdir -p rootfs
cd rootfs
mkdir -p bin sbin etc dev proc sys tmp usr/bin usr/sbin var lib

# Copy BusyBox
cp -a ../busybox-1.36.0/_install/* .

# Create device nodes
sudo mknod dev/console c 5 1
sudo mknod dev/null c 1 3

# Create init script
cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
exec /bin/sh
EOF
chmod +x init
```

### Init Systems

#### Simple init
```bash
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
exec /sbin/init
```

#### BusyBox init
Uses `/etc/inittab`:
```
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty 115200 ttyS0
::shutdown:/bin/umount -a -r
```

#### systemd
Full-featured init system (larger footprint).

## Kernel Configuration for Embedded

### Essential Options
```
CONFIG_EMBEDDED=y
CONFIG_EXPERT=y

# Reduce kernel size
CONFIG_MODULES=y
CONFIG_KALLSYMS=n
CONFIG_DEBUG_KERNEL=n
CONFIG_PRINTK=y

# Embedded features
CONFIG_EMBEDDED_RAMDISK=y
CONFIG_BLK_DEV_INITRD=y

# Device tree
CONFIG_OF=y
CONFIG_OF_FLATTREE=y
```

### Size Optimization
```bash
# Strip kernel modules
INSTALL_MOD_STRIP=1 make modules_install

# Compress with XZ
CONFIG_KERNEL_XZ=y

# Remove debug symbols
CONFIG_DEBUG_INFO=n
```

## Flash Memory

### Types
- **NOR Flash**: Random access, execute-in-place
- **NAND Flash**: Block access, higher density
- **eMMC**: Managed NAND with controller
- **SD/MMC**: Removable storage

### MTD (Memory Technology Devices)

Subsystem for flash memory.

#### MTD Partitions
```bash
# View MTD partitions
cat /proc/mtd

# Read from MTD
dd if=/dev/mtd0 of=backup.img

# Write to MTD
flash_erase /dev/mtd0 0 0
nandwrite -p /dev/mtd0 image.bin
```

#### MTD in Device Tree
```dts
flash@0 {
    compatible = "jedec,spi-nor";
    reg = <0>;
    
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;
        
        partition@0 {
            label = "bootloader";
            reg = <0x0 0x40000>;
            read-only;
        };
        
        partition@40000 {
            label = "kernel";
            reg = <0x40000 0x400000>;
        };
        
        partition@440000 {
            label = "rootfs";
            reg = <0x440000 0x0>;
        };
    };
};
```

## Real-Time Linux

### PREEMPT_RT Patch

Provides hard real-time capabilities.

#### Apply RT Patch
```bash
# Download kernel and RT patch
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.1/patch-6.1-rt.patch.xz

# Extract and patch
tar xf linux-6.1.tar.xz
cd linux-6.1
xzcat ../patch-6.1-rt.patch.xz | patch -p1

# Configure
make menuconfig
# Enable: Fully Preemptible Kernel (RT)

# Build
make -j$(nproc)
```

### RT Configuration
```
CONFIG_PREEMPT_RT=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_NO_HZ_FULL=y
```

## Power Management

### CPU Frequency Scaling
```bash
# View available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# Set governor
echo "powersave" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

### Suspend/Resume
```bash
# Suspend to RAM
echo mem > /sys/power/state

# Suspend to disk (hibernate)
echo disk > /sys/power/state
```

### Device Tree Power Domains
```dts
power: power-controller {
    compatible = "vendor,power";
    #power-domain-cells = <1>;
};

device@0 {
    power-domains = <&power 0>;
};
```

## Debugging Embedded Systems

### Serial Console
```bash
# Kernel command line
console=ttyS0,115200

# Connect via minicom
minicom -D /dev/ttyUSB0 -b 115200
```

### JTAG Debugging

Using OpenOCD:
```bash
# Start OpenOCD
openocd -f interface/jlink.cfg -f target/stm32f4x.cfg

# Connect GDB
arm-none-eabi-gdb vmlinux
(gdb) target remote localhost:3333
(gdb) load
(gdb) continue
```

### Early Printk
```
CONFIG_EARLY_PRINTK=y
# Kernel command line: earlyprintk
```

## Common Embedded Platforms

### Raspberry Pi
- ARM Cortex-A series
- Broadcom SoC
- GPIO, SPI, I2C, UART

### BeagleBone
- ARM Cortex-A8
- TI AM335x SoC
- PRU (Programmable Real-time Unit)

### i.MX Series
- NXP ARM processors
- Industrial applications
- Rich peripheral set

## Best Practices

1. **Minimize footprint**: Remove unnecessary features
2. **Optimize boot time**: Parallel initialization, reduce services
3. **Secure boot**: Verify bootloader and kernel integrity
4. **Watchdog**: Automatic recovery from hangs
5. **OTA updates**: Over-the-air firmware updates
6. **Logging**: Remote logging for debugging
7. **Testing**: Extensive testing on target hardware

## Next Steps

Proceed to [Linux Kernel Module Programming](03-kernel-module-programming.md) to learn how to develop loadable kernel modules.
