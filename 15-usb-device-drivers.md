# USB Device Drivers

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
