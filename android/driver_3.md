# 5.2 usb_class_driver

## 1.介绍

usb function driver 通过给上层空间提供字符设备接口

#define USB_MAJOR           180

## 2.注册到系统

```c
int usb_major_init(void)
{
    int error;

    error = register_chrdev(USB_MAJOR, "usb", &usb_fops);
    if (error)
        printk(KERN_ERR "Unable to get major %d for usb devices\n",
               USB_MAJOR);

    return error;
}

```



## 3. 使用

```c
int usb_register_dev(struct usb_interface *intf,
             struct usb_class_driver *class_driver)
{
    int retval;
    int minor_base = class_driver->minor_base;
    int minor;
    char name[20];
    char *temp;

#ifdef CONFIG_USB_DYNAMIC_MINORS
    /*
     * We don't care what the device tries to start at, we want to start
     * at zero to pack the devices into the smallest available space with
     * no holes in the minor range.
     */
    minor_base = 0;
#endif

    if (class_driver->fops == NULL)
        return -EINVAL;
    if (intf->minor >= 0)
        return -EADDRINUSE;

    mutex_lock(&init_usb_class_mutex);
    retval = init_usb_class();
    mutex_unlock(&init_usb_class_mutex);

    if (retval)
        return retval;

    dev_dbg(&intf->dev, "looking for a minor, starting at %d\n", minor_base);

    down_write(&minor_rwsem);
    for (minor = minor_base; minor < MAX_USB_MINORS; ++minor) {
        if (usb_minors[minor])
            continue;

        usb_minors[minor] = class_driver->fops;//保存class_driver fops
        intf->minor = minor;
        break;
    }
    up_write(&minor_rwsem);
    if (intf->minor < 0)
        return -EXFULL;

    /* create a usb class device for this usb interface */
    snprintf(name, sizeof(name), class_driver->name, minor - minor_base);
    temp = strrchr(name, '/');
    if (temp && (temp[1] != '\0'))
        ++temp;
    else
        temp = name;
    intf->usb_dev = device_create(usb_class->class, &intf->dev,
                      MKDEV(USB_MAJOR, minor), class_driver,
                      "%s", temp);
    if (IS_ERR(intf->usb_dev)) {
        down_write(&minor_rwsem);
        usb_minors[minor] = NULL;
        intf->minor = -1;
        up_write(&minor_rwsem);
        retval = PTR_ERR(intf->usb_dev);
    }
    return retval;
}

```

在进行open的时候，会将fops 替换成class_drvier->fops

```c
static int usb_open(struct inode *inode, struct file *file)
{
    int err = -ENODEV;
    const struct file_operations *new_fops;

    down_read(&minor_rwsem);
    new_fops = fops_get(usb_minors[iminor(inode)]);

    if (!new_fops)
        goto done;

    replace_fops(file, new_fops);
    /* Curiouser and curiouser... NULL ->open() as "no device" ? */
    if (file->f_op->open)
        err = file->f_op->open(inode, file);
 done:
    up_read(&minor_rwsem);
    return err;
}

```

## 4.例子

usblp.c

```
static const struct file_operations usblp_fops = {
    .owner =    THIS_MODULE,
    .read =     usblp_read,
    .write =    usblp_write,
    .poll =     usblp_poll,
    .unlocked_ioctl =   usblp_ioctl,
    .compat_ioctl =     usblp_ioctl,
    .open =     usblp_open,
    .release =  usblp_release,
    .llseek =   noop_llseek,
};

static char *usblp_devnode(struct device *dev, umode_t *mode)
{
    return kasprintf(GFP_KERNEL, "usb/%s", dev_name(dev));
}

static struct usb_class_driver usblp_class = {
    .name =     "lp%d",
    .devnode =  usblp_devnode,
    .fops =     &usblp_fops,
    .minor_base =   USBLP_MINOR_BASE,
};

retval = usb_register_dev(intf, &usblp_class);

```



 

