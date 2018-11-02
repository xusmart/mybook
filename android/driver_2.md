# 5.1 usbfs driver

## 1.介绍

usbfs driver主要是利用devio.c 提供的io接口，实现和usbcore交互

可以实现usb configure interface claim urb 等操作

#define USB_MAJOR           180
#define USB_DEVICE_MAJOR        189

usbfs 使用的是**USB_DEVICE_MAJOR**这个要做好区分，实现在***devio.c***

USB_MAJOR 这个是 usb class driver 使用的



## 2.注册

```c
int __init usb_devio_init(void)
{
    int retval;

    retval = register_chrdev_region(USB_DEVICE_DEV, USB_DEVICE_MAX,
                    "usb_device");
    if (retval) {
        printk(KERN_ERR "Unable to register minors for usb_device\n");
        goto out; 
    }    
    cdev_init(&usb_device_cdev, &usbdev_file_operations);
    retval = cdev_add(&usb_device_cdev, USB_DEVICE_DEV, USB_DEVICE_MAX);
    if (retval) {
        printk(KERN_ERR "Unable to get usb_device major %d\n",
               USB_DEVICE_MAJOR);
        goto error_cdev;
    }    
    usb_register_notify(&usbdev_nb);
out:
    return retval;

error_cdev:
    unregister_chrdev_region(USB_DEVICE_DEV, USB_DEVICE_MAX);
    goto out; 
}

```
### 2.1 usb_device 注册字符设备

在hub.c usb_new_device 这里会创建设备文件,通过uevent上报时间，由uventd 在/dev/下面创建设备文件

```

    dev_dbg(&udev->dev, "udev %d, busnum %d, minor = %d\n",
            udev->devnum, udev->bus->busnum,
            (((udev->bus->busnum-1) * 128) + (udev->devnum-1)));
    /* export the usbdev device-node for libusb */
    udev->dev.devt = MKDEV(USB_DEVICE_MAJOR,
            (((udev->bus->busnum-1) * 128) + (udev->devnum-1)));

    /* Tell the world! */
    announce_device(udev);

    if (udev->serial)
        add_device_randomness(udev->serial, strlen(udev->serial));
    if (udev->product)
        add_device_randomness(udev->product, strlen(udev->product));
    if (udev->manufacturer)
        add_device_randomness(udev->manufacturer,
                      strlen(udev->manufacturer));

    device_enable_async_suspend(&udev->dev);

    /* check whether the hub or firmware marks this port as non-removable */
    if (udev->parent)
        set_usb_port_removable(udev);

    /* Register the device.  The device driver is responsible
     * for configuring the device and invoking the add-device
     * notifier chain (used by usbfs and possibly others).
     */
    err = device_add(&udev->dev);

```



## 3.使用

```c
#include <ctype.h>
#include <dirent.h>
#include <errno.h>
#include <fcntl.h>
#include <linux/usb/ch9.h>
#include <linux/usbdevice_fs.h>
#include <linux/version.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

#define USB_DEVICE "/dev/bus/usb/001/003"
int main()
{
    int usb_fd;
    unsigned char devdesc[4096];
    struct usb_device_descriptor* device;
    struct usb_config_descriptor* config;
    struct usb_interface_descriptor *interface;
    struct usb_endpoint_descriptor *ep0;

    unsigned char* bufptr = devdesc;
    unsigned char* bufend;
    size_t desclength;

    usb_fd = open(USB_DEVICE, O_RDONLY | O_CLOEXEC);
    if (!usb_fd) {
        printf("open %s failed!\n", USB_DEVICE);
    }   

    desclength = read(usb_fd, devdesc, sizeof(devdesc));
    if (desclength < USB_DT_DEVICE_SIZE + USB_DT_CONFIG_SIZE) {
        printf("desclength too small\n");
        goto out;
    }   
    device = (struct usb_device_descriptor*)bufptr;
    printf("vid %02x pid %02x\n", device->idVendor, device->idProduct);


out:    
    close(usb_fd);
    return 0;
}

```



