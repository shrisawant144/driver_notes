# USB Device Drivers

## 🎯 Layman's Explanation

**What is USB?**
USB = Universal Serial Bus - the most common way to connect devices to computers. Your mouse, keyboard, phone, printer - probably all USB!

**Why is USB Special?**
- **Hot-pluggable** - Plug in/out while system runs
- **Auto-detection** - System knows what you plugged in
- **Power delivery** - Can charge devices
- **Universal** - One port for many device types

**USB Hierarchy:**
```
Computer (Host)
    ↓
USB Host Controller (hardware chip)
    ↓
USB Hub (optional, splits one port into many)
    ↓
USB Device (your mouse, keyboard, etc.)
```

**Analogy:**
- Host = Main office
- Hub = Branch office
- Device = Employee

**USB Device Anatomy:**

**1. Device** - The physical thing (e.g., webcam)
```
Device
  ├── Configuration (power mode, etc.)
  │     └── Interface (camera function)
  │           └── Endpoints (data pipes)
  └── Configuration 2 (alternate mode)
```

**2. Interface** - A function of the device
- Example: Webcam might have:
  - Interface 0: Video
  - Interface 1: Audio (microphone)

**3. Endpoints** - Data pipes
- **IN endpoint** - Device → Computer (like receiving mail)
- **OUT endpoint** - Computer → Device (like sending mail)

**Analogy:**
- Device = Smartphone
- Interface = Apps (camera app, music app)
- Endpoints = Data channels (upload, download)

**USB Transfer Types:**

**1. Control** - Configuration and commands
```
Like: Sending instructions to device
Example: "Set resolution to 1080p"
```

**2. Bulk** - Large data, no timing guarantee
```
Like: Copying files to USB drive
Example: Transferring photos
```

**3. Interrupt** - Small, periodic data
```
Like: Mouse movements, keyboard presses
Example: "Mouse moved 5 pixels left"
```

**4. Isochronous** - Streaming, guaranteed timing
```
Like: Video/audio streaming
Example: Webcam video feed
```

**USB Driver Flow:**

**When You Plug In a USB Device:**
```
1. Device plugged in
    ↓
2. Host controller detects it
    ↓
3. USB core reads device descriptor
    ↓
4. "Who are you?" → Device: "I'm VID:0x1234, PID:0x5678"
    ↓
5. Kernel searches for matching driver
    ↓
6. Found! Driver's probe() called
    ↓
7. Driver initializes device
    ↓
8. Device ready to use!
```

**Analogy:**
- New employee arrives (device plugged)
- Security checks ID (read descriptor)
- HR finds their job role (match driver)
- Manager onboards them (probe() function)
- Employee starts working (device operational)

**VID and PID - Device Identity:**
- **VID** = Vendor ID (who made it)
- **PID** = Product ID (what it is)

**Example:**
- VID: 0x046d = Logitech
- PID: 0xc52b = Logitech Mouse Model XYZ

**Analogy:**
- VID = Company name
- PID = Employee ID

**URB - USB Request Block:**
The way to send/receive data:
```
Create URB → Fill with data → Submit → Wait → Complete → Process result
```

**Analogy:**
- URB = Delivery package
- Fill it with data (contents)
- Submit to USB core (mail it)
- Completion callback (delivery confirmation)

**Common USB Driver Tasks:**

**1. Probe Function:**
```c
Device connected → probe() called
  - Allocate resources
  - Setup endpoints
  - Register char device
  - Ready!
```

**2. Disconnect Function:**
```c
Device removed → disconnect() called
  - Stop transfers
  - Free resources
  - Cleanup
```

**3. Read/Write:**
```c
User reads → Driver submits URB → Device sends data → Callback → Copy to user
```

**USB Speeds:**
```
USB 1.0: Low Speed    = 1.5 Mbps   (keyboards, mice)
USB 1.1: Full Speed   = 12 Mbps    (audio devices)
USB 2.0: High Speed   = 480 Mbps   (webcams, drives)
USB 3.0: SuperSpeed   = 5 Gbps     (external SSDs)
USB 3.1: SuperSpeed+  = 10 Gbps    (high-end devices)
```

**Analogy:**
- Low Speed = Walking
- Full Speed = Biking
- High Speed = Car
- SuperSpeed = Airplane

**Power Management:**
USB devices can:
- **Suspend** - Sleep to save power
- **Resume** - Wake up
- **Remote wakeup** - Device wakes up host (e.g., keyboard press)

**Common USB Classes:**
Instead of writing custom drivers, use standard classes:

- **HID** (Human Interface Device) - Keyboards, mice
- **Mass Storage** - USB drives
- **CDC** (Communications) - Modems, serial ports
- **Audio** - Speakers, microphones
- **Video** - Webcams

**Analogy:**
- Classes = Job categories (everyone in category works similarly)
- HID class = All input devices work the same way

**Debugging USB:**
```bash
lsusb                    # List USB devices
lsusb -v                 # Verbose info
dmesg | grep -i usb      # Kernel messages
cat /sys/kernel/debug/usb/devices  # Detailed info
```

**Real-World Example - USB LED Driver:**
```
1. Plug in USB LED device
2. Driver probe() called
3. Driver finds OUT endpoint
4. User writes "1" to /dev/usbled0
5. Driver sends USB control message
6. LED turns on!
```

**Key Takeaways:**
- USB is hot-pluggable and auto-detecting
- Devices have VID/PID for identification
- Data flows through endpoints
- URBs are used for communication
- Standard classes simplify driver development

## Overview

USB (Universal Serial Bus) driver development for interfacing with USB devices in Linux kernel.

## USB Architecture

- Host Controller (EHCI, XHCI, OHCI, UHCI)
- USB Core
- USB Device Drivers
- USB Devices (endpoints, interfaces, configurations)

## USB Driver Structure

```c
#include <linux/usb.h>

static struct usb_device_id usb_table[] = {
    { USB_DEVICE(VENDOR_ID, PRODUCT_ID) },
    { }
};
MODULE_DEVICE_TABLE(usb, usb_table);

static int usb_probe(struct usb_interface *interface,
                     const struct usb_device_id *id)
{
    struct usb_device *udev = interface_to_usbdev(interface);
    pr_info("USB device connected: VID=0x%04x PID=0x%04x\n",
            le16_to_cpu(udev->descriptor.idVendor),
            le16_to_cpu(udev->descriptor.idProduct));
    return 0;
}

static void usb_disconnect(struct usb_interface *interface)
{
    pr_info("USB device disconnected\n");
}

static struct usb_driver usb_driver = {
    .name = "my_usb_driver",
    .id_table = usb_table,
    .probe = usb_probe,
    .disconnect = usb_disconnect,
};

module_usb_driver(usb_driver);
```

## USB Device Information

```c
struct usb_device *udev = interface_to_usbdev(interface);
struct usb_host_interface *iface_desc = interface->cur_altsetting;

// Device descriptor
pr_info("Vendor: 0x%04x\n", le16_to_cpu(udev->descriptor.idVendor));
pr_info("Product: 0x%04x\n", le16_to_cpu(udev->descriptor.idProduct));

// Interface information
pr_info("Interface: %d\n", iface_desc->desc.bInterfaceNumber);
pr_info("Endpoints: %d\n", iface_desc->desc.bNumEndpoints);

// Endpoint information
for (i = 0; i < iface_desc->desc.bNumEndpoints; i++) {
    struct usb_endpoint_descriptor *endpoint = 
        &iface_desc->endpoint[i].desc;
    pr_info("Endpoint %d: addr=0x%02x type=%d\n",
            i, endpoint->bEndpointAddress,
            usb_endpoint_type(endpoint));
}
```

## USB Transfers

### Bulk Transfer

```c
int usb_bulk_msg(struct usb_device *usb_dev,
                 unsigned int pipe,
                 void *data,
                 int len,
                 int *actual_length,
                 int timeout);

// Example: Read
unsigned char buffer[64];
int actual_len;
int ret = usb_bulk_msg(udev,
                       usb_rcvbulkpipe(udev, endpoint_addr),
                       buffer, sizeof(buffer),
                       &actual_len, 5000);

// Example: Write
ret = usb_bulk_msg(udev,
                   usb_sndbulkpipe(udev, endpoint_addr),
                   data, len, &actual_len, 5000);
```

### Control Transfer

```c
int usb_control_msg(struct usb_device *dev,
                    unsigned int pipe,
                    __u8 request,
                    __u8 requesttype,
                    __u16 value,
                    __u16 index,
                    void *data,
                    __u16 size,
                    int timeout);

// Example
ret = usb_control_msg(udev,
                      usb_rcvctrlpipe(udev, 0),
                      request,
                      USB_DIR_IN | USB_TYPE_VENDOR,
                      value, index,
                      buffer, size, 5000);
```

### Interrupt Transfer

```c
struct urb *urb;

void usb_int_callback(struct urb *urb)
{
    if (urb->status) {
        pr_err("URB error: %d\n", urb->status);
        return;
    }
    // Process data
    pr_info("Received %d bytes\n", urb->actual_length);
    
    // Resubmit URB
    usb_submit_urb(urb, GFP_ATOMIC);
}

// Setup interrupt URB
urb = usb_alloc_urb(0, GFP_KERNEL);
usb_fill_int_urb(urb, udev,
                 usb_rcvintpipe(udev, endpoint_addr),
                 buffer, size,
                 usb_int_callback, dev,
                 interval);
usb_submit_urb(urb, GFP_KERNEL);

// Cleanup
usb_kill_urb(urb);
usb_free_urb(urb);
```

## URB (USB Request Block)

```c
struct urb *urb = usb_alloc_urb(0, GFP_KERNEL);

// Fill URB for bulk transfer
usb_fill_bulk_urb(urb, udev,
                  usb_rcvbulkpipe(udev, endpoint_addr),
                  buffer, size,
                  completion_callback, context);

// Submit URB
int ret = usb_submit_urb(urb, GFP_KERNEL);

// Cancel URB
usb_kill_urb(urb);

// Free URB
usb_free_urb(urb);
```

## Example: USB LED Driver

```c
#include <linux/module.h>
#include <linux/usb.h>

#define VENDOR_ID  0x1234
#define PRODUCT_ID 0x5678

static struct usb_device_id led_table[] = {
    { USB_DEVICE(VENDOR_ID, PRODUCT_ID) },
    { }
};
MODULE_DEVICE_TABLE(usb, led_table);

struct usb_led {
    struct usb_device *udev;
    u8 bulk_out_endpointAddr;
};

static ssize_t led_write(struct file *file, const char __user *user_buffer,
                         size_t count, loff_t *ppos)
{
    struct usb_led *dev = file->private_data;
    char cmd;
    int retval;
    
    if (copy_from_user(&cmd, user_buffer, 1))
        return -EFAULT;
    
    retval = usb_bulk_msg(dev->udev,
                          usb_sndbulkpipe(dev->udev, dev->bulk_out_endpointAddr),
                          &cmd, 1, NULL, 5000);
    return (retval < 0) ? retval : count;
}

static int led_probe(struct usb_interface *interface,
                     const struct usb_device_id *id)
{
    struct usb_led *dev;
    struct usb_host_interface *iface_desc;
    
    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;
    
    dev->udev = usb_get_dev(interface_to_usbdev(interface));
    iface_desc = interface->cur_altsetting;
    
    // Find bulk-out endpoint
    for (int i = 0; i < iface_desc->desc.bNumEndpoints; i++) {
        struct usb_endpoint_descriptor *endpoint = 
            &iface_desc->endpoint[i].desc;
        if (usb_endpoint_is_bulk_out(endpoint)) {
            dev->bulk_out_endpointAddr = endpoint->bEndpointAddress;
            break;
        }
    }
    
    usb_set_intfdata(interface, dev);
    pr_info("USB LED device connected\n");
    return 0;
}

static void led_disconnect(struct usb_interface *interface)
{
    struct usb_led *dev = usb_get_intfdata(interface);
    usb_put_dev(dev->udev);
    kfree(dev);
    pr_info("USB LED device disconnected\n");
}

static struct usb_driver led_driver = {
    .name = "usb_led",
    .id_table = led_table,
    .probe = led_probe,
    .disconnect = led_disconnect,
};

module_usb_driver(led_driver);
MODULE_LICENSE("GPL");
```

## USB Debugging

```bash
# List USB devices
lsusb

# Detailed device info
lsusb -v -d vendor:product

# USB monitoring
usbmon
cat /sys/kernel/debug/usb/usbmon/0u

# dmesg for driver messages
dmesg | grep usb
```

## Best Practices

- Always check return values
- Use appropriate transfer types for device requirements
- Handle device disconnection gracefully
- Free all allocated resources in disconnect
- Use URBs for asynchronous operations
- Implement proper error handling
- Use `usb_get_dev()` and `usb_put_dev()` for reference counting

## Common USB Classes

- HID (Human Interface Device): 0x03
- Mass Storage: 0x08
- CDC (Communications): 0x02
- Audio: 0x01
- Video: 0x0E
- Vendor Specific: 0xFF
