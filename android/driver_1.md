# 5.1 usb host driver
## 1.文件目录结构
- 目录 drivers/usb/core

- 功能 usb-core usb核心功能实现  

  hcd.c 控制器

  hub.c hub功能实现

  message.c usb 枚举相关

  devio.c usb userspace 利用io接口 可以实现具体的usbdriver的功能

  usb.c usb device 创建相关

## 2.核心数据结构

### 2.1 usb_device

描述usb 设备，和设备打交道，和usb_device_driver 绑定

```c
struct usb_device {
	int		devnum; //设备号
	char		devpath[16];
	u32		route;
	enum usb_device_state	state;
	enum usb_device_speed	speed;

	struct usb_tt	*tt;
	int		ttport;

	unsigned int toggle[2];

	struct usb_device *parent;
	struct usb_bus *bus;
	struct usb_host_endpoint ep0;

	struct device dev;

	struct usb_device_descriptor descriptor;//设备描述符
	struct usb_host_bos *bos;
	struct usb_host_config *config;//设备配置

	struct usb_host_config *actconfig;
	struct usb_host_endpoint *ep_in[16];//设备端点
	struct usb_host_endpoint *ep_out[16];

	char **rawdescriptors;

	unsigned short bus_mA;
	u8 portnum;
	u8 level;

	unsigned can_submit:1;
	unsigned persist_enabled:1;
	unsigned have_langid:1;
	unsigned authorized:1;
	unsigned authenticated:1;
	unsigned wusb:1;
	unsigned lpm_capable:1;
	unsigned usb2_hw_lpm_capable:1;
	unsigned usb2_hw_lpm_besl_capable:1;
	unsigned usb2_hw_lpm_enabled:1;
	unsigned usb2_hw_lpm_allowed:1;
	unsigned usb3_lpm_enabled:1;
	unsigned usb3_lpm_u1_enabled:1;
	unsigned usb3_lpm_u2_enabled:1;
	int string_langid;

	/* static strings from the device */
	char *product;
	char *manufacturer;
	char *serial;

	struct list_head filelist;

	int maxchild;

	u32 quirks;
	atomic_t urbnum;

	unsigned long active_duration;

#ifdef CONFIG_PM
	unsigned long connect_time;

	unsigned do_remote_wakeup:1;
	unsigned reset_resume:1;
	unsigned port_is_suspended:1;
#endif
	struct wusb_dev *wusb_dev;
	int slot_id;
	enum usb_device_removable removable;
	struct usb2_lpm_parameters l1_params;
	struct usb3_lpm_parameters u1_params;
	struct usb3_lpm_parameters u2_params;
	unsigned lpm_disable_count;
};
```

```c
struct usb_device_driver {
	const char *name;

	int (*probe) (struct usb_device *udev);
	void (*disconnect) (struct usb_device *udev);

	int (*suspend) (struct usb_device *udev, pm_message_t message);
	int (*resume) (struct usb_device *udev, pm_message_t message);
	struct usbdrv_wrap drvwrap;
	unsigned int supports_autosuspend:1;
};
```



### 2.2 usb_interface 

usb的function device，和function driver 即usb_driver 绑定

```c
struct usb_interface {
	/* array of alternate settings for this interface,
	 * stored in no particular order */
	struct usb_host_interface *altsetting;

	struct usb_host_interface *cur_altsetting;	/* the currently
					 * active alternate setting */
	unsigned num_altsetting;	/* number of alternate settings */

	/* If there is an interface association descriptor then it will list
	 * the associated interfaces */
	struct usb_interface_assoc_descriptor *intf_assoc;

	int minor;			/* minor number this interface is
					 * bound to */
	enum usb_interface_condition condition;		/* state of binding */
	unsigned sysfs_files_created:1;	/* the sysfs attributes exist */
	unsigned ep_devs_created:1;	/* endpoint "devices" exist */
	unsigned unregistering:1;	/* unregistration is in progress */
	unsigned needs_remote_wakeup:1;	/* driver requires remote wakeup */
	unsigned needs_altsetting0:1;	/* switch to altsetting 0 is pending */
	unsigned needs_binding:1;	/* needs delayed unbind/rebind */
	unsigned resetting_device:1;	/* true: bandwidth alloc after reset */
	unsigned authorized:1;		/* used for interface authorization */

	struct device dev;		/* interface specific device info */
	struct device *usb_dev;
	atomic_t pm_usage_cnt;		/* usage counter for autosuspend */
	struct work_struct reset_ws;	/* for resets in atomic context */
};

struct usb_driver {
	const char *name;

	int (*probe) (struct usb_interface *intf,
		      const struct usb_device_id *id);

	void (*disconnect) (struct usb_interface *intf);

	int (*unlocked_ioctl) (struct usb_interface *intf, unsigned int code,
			void *buf);

	int (*suspend) (struct usb_interface *intf, pm_message_t message);
	int (*resume) (struct usb_interface *intf);
	int (*reset_resume)(struct usb_interface *intf);

	int (*pre_reset)(struct usb_interface *intf);
	int (*post_reset)(struct usb_interface *intf);

	const struct usb_device_id *id_table;

	struct usb_dynids dynids;
	struct usbdrv_wrap drvwrap;
	unsigned int no_dynamic_id:1;
	unsigned int supports_autosuspend:1;
	unsigned int disable_hub_initiated_lpm:1;
	unsigned int soft_unbind:1;
};
```

## 3.驱动注册

### 3.1 usb 总线注册



```c
driver.c
struct bus_type usb_bus_type = {
	.name =		"usb",
	.match =	usb_device_match,
	.uevent =	usb_uevent,
};
usb.c
static int __init usb_init(void)
{
	int retval;
	if (usb_disabled()) {
		pr_info("%s: USB support disabled\n", usbcore_name);
		return 0;
	}
	usb_init_pool_max();

	retval = usb_debugfs_init();
	if (retval)
		goto out;

	usb_acpi_register();
	retval = bus_register(&usb_bus_type);//注册usb总线
	if (retval)
		goto bus_register_failed;
	retval = bus_register_notifier(&usb_bus_type, &usb_bus_nb);
	if (retval)
		goto bus_notifier_failed;
	retval = usb_major_init();
	if (retval)
		goto major_init_failed;
	retval = usb_register(&usbfs_driver);
	if (retval)
		goto driver_register_failed;
	retval = usb_devio_init();
	if (retval)
		goto usb_devio_init_failed;
	retval = usb_hub_init();
	if (retval)
		goto hub_init_failed;
	retval = usb_register_device_driver(&usb_generic_driver, THIS_MODULE);//注册usb_device_driver
	if (!retval)
		goto out;

	usb_hub_cleanup();
hub_init_failed:
	usb_devio_cleanup();
usb_devio_init_failed:
	usb_deregister(&usbfs_driver);
driver_register_failed:
	usb_major_cleanup();
major_init_failed:
	bus_unregister_notifier(&usb_bus_type, &usb_bus_nb);
bus_notifier_failed:
	bus_unregister(&usb_bus_type);
bus_register_failed:
	usb_acpi_unregister();
	usb_debugfs_cleanup();
out:
	return retval;
}

usb_device设备是通过device_add 注册到usb 总线，然后执行usb_device_match


```

### 3.2 usb_device 注册

***usb.c*** **usb_alloc_dev**  初始化设备

```c
struct usb_device *usb_alloc_dev(struct usb_device *parent,
				 struct usb_bus *bus, unsigned port1)
{
	struct usb_device *dev;
	struct usb_hcd *usb_hcd = bus_to_hcd(bus);
	unsigned root_hub = 0;

	dev = kzalloc(sizeof(*dev), GFP_KERNEL);
	if (!dev)
		return NULL;

	if (!usb_get_hcd(usb_hcd)) {
		kfree(dev);
		return NULL;
	}
	/* Root hubs aren't true devices, so don't allocate HCD resources */
	if (usb_hcd->driver->alloc_dev && parent &&
		!usb_hcd->driver->alloc_dev(usb_hcd, dev)) {
		usb_put_hcd(bus_to_hcd(bus));
		kfree(dev);
		return NULL;
	}

	device_initialize(&dev->dev);
	dev->dev.bus = &usb_bus_type; //注册到对应的总线
	dev->dev.type = &usb_device_type;
	dev->dev.groups = usb_device_groups;
	dev->dev.dma_mask = bus->controller->dma_mask;
	set_dev_node(&dev->dev, dev_to_node(bus->controller));
	dev->state = USB_STATE_ATTACHED;
	dev->lpm_disable_count = 1;
	atomic_set(&dev->urbnum, 0);

	INIT_LIST_HEAD(&dev->ep0.urb_list);
	dev->ep0.desc.bLength = USB_DT_ENDPOINT_SIZE;
	dev->ep0.desc.bDescriptorType = USB_DT_ENDPOINT;
	/* ep0 maxpacket comes later, from device descriptor */
	usb_enable_endpoint(dev, &dev->ep0, false);
	dev->can_submit = 1;

	/* Save readable and stable topology id, distinguishing devices
	 * by location for diagnostics, tools, driver model, etc.  The
	 * string is a path along hub ports, from the root.  Each device's
	 * dev->devpath will be stable until USB is re-cabled, and hubs
	 * are often labeled with these port numbers.  The name isn't
	 * as stable:  bus->busnum changes easily from modprobe order,
	 * cardbus or pci hotplugging, and so on.
	 */
	if (unlikely(!parent)) {
		dev->devpath[0] = '0';
		dev->route = 0;

		dev->dev.parent = bus->controller;
		dev_set_name(&dev->dev, "usb%d", bus->busnum);
		root_hub = 1;
	} else {
		/* match any labeling on the hubs; it's one-based */
		if (parent->devpath[0] == '0') {
			snprintf(dev->devpath, sizeof dev->devpath,
				"%d", port1);
			/* Root ports are not counted in route string */
			dev->route = 0;
		} else {
			snprintf(dev->devpath, sizeof dev->devpath,
				"%s.%d", parent->devpath, port1);
			/* Route string assumes hubs have less than 16 ports */
			if (port1 < 15)
				dev->route = parent->route +
					(port1 << ((parent->level - 1)*4));
			else
				dev->route = parent->route +
					(15 << ((parent->level - 1)*4));
		}

		dev->dev.parent = &parent->dev;
		dev_set_name(&dev->dev, "%d-%s", bus->busnum, dev->devpath);

		/* hub driver sets up TT records */
	}

	dev->portnum = port1;
	dev->bus = bus;
	dev->parent = parent;
	INIT_LIST_HEAD(&dev->filelist);

#ifdef	CONFIG_PM
	pm_runtime_set_autosuspend_delay(&dev->dev,
			usb_autosuspend_delay * 1000);
	dev->connect_time = jiffies;
	dev->active_duration = -jiffies;
#endif
	if (root_hub)	/* Root hub always ok [and always wired] */
		dev->authorized = 1;
	else {
		dev->authorized = !!HCD_DEV_AUTHORIZED(usb_hcd);
		dev->wusb = usb_bus_is_wusb(bus) ? 1 : 0;
	}
	return dev;
}
```

hub.c usb_new_device 调用 device_add(&udev->dev);

### 3.3 usb_interface接口注册

message.c

usb_set_configuration

```c
		intf->dev.parent = &dev->dev;
		intf->dev.driver = NULL;
		intf->dev.bus = &usb_bus_type;
		intf->dev.type = &usb_if_device_type;
		intf->dev.groups = usb_interface_groups;
		intf->dev.dma_mask = dev->dev.dma_mask;


		ret = device_add(&intf->dev);


```



### 3.4 匹配

```c
static int usb_device_match(struct device *dev, struct device_driver *drv)
{
    /* devices and interfaces are handled separately */
    if (is_usb_device(dev)) {    /* interface drivers never match devices */
    if (!is_usb_device_driver(drv))
        return 0;

    /* TODO: Add real matching code */
    return 1;

} else if (is_usb_interface(dev)) {
    struct usb_interface *intf;
    struct usb_driver *usb_drv;
    const struct usb_device_id *id; 

    /* device drivers never match interfaces */
    if (is_usb_device_driver(drv))
        return 0;

    intf = to_usb_interface(dev);
    usb_drv = to_usb_driver(drv);

    id = usb_match_id(intf, usb_drv->id_table);
    if (id) 
        return 1;

    id = usb_match_dynamic_id(intf, usb_drv);
    if (id) 
        return 1;
}    

return 0;
}
```


### 3.5  执行设备驱动probe

```c
struct usb_device_driver usb_generic_driver = { 
    .name = "usb",
    .probe = generic_probe,
    .disconnect = generic_disconnect,
#ifdef  CONFIG_PM
    .suspend = generic_suspend,
    .resume = generic_resume,
#endif
    .supports_autosuspend = 1,
};
注册usb_device_driver
int usb_register_device_driver(struct usb_device_driver *new_udriver,
        struct module *owner)
{
    int retval = 0; 

    if (usb_disabled())
        return -ENODEV;

    new_udriver->drvwrap.for_devices = 1; 
    new_udriver->drvwrap.driver.name = new_udriver->name;
    new_udriver->drvwrap.driver.bus = &usb_bus_type;
    new_udriver->drvwrap.driver.probe = usb_probe_device;
    new_udriver->drvwrap.driver.remove = usb_unbind_device;
    new_udriver->drvwrap.driver.owner = owner;

    retval = driver_register(&new_udriver->drvwrap.driver);

    if (!retval)
        pr_info("%s: registered new device driver %s\n",
            usbcore_name, new_udriver->name);
    else 
        printk(KERN_ERR "%s: error %d registering device "
            "   driver %s\n",
            usbcore_name, retval, new_udriver->name);

    return retval;
}


static int usb_probe_device(struct device *dev)
{
    struct usb_device_driver *udriver = to_usb_device_driver(dev->driver);
    struct usb_device *udev = to_usb_device(dev);
    int error = 0;

    dev_dbg(dev, "%s\n", __func__);

    /* TODO: Add real matching code */

    /* The device should always appear to be in use
     * unless the driver supports autosuspend.
     */
    if (!udriver->supports_autosuspend)
        error = usb_autoresume_device(udev);

    if (!error)
        error = udriver->probe(udev);
    return error;
}

这个函数蛮关键，进行usb 设备的配置
static int generic_probe(struct usb_device *udev)
{
    int err, c;

    /* Choose and set the configuration.  This registers the interfaces
     * with the driver core and lets interface drivers bind to them.
     */
    if (udev->authorized == 0)
        dev_err(&udev->dev, "Device is not authorized for usage\n");
    else {
        c = usb_choose_configuration(udev);
        if (c >= 0) {
            err = usb_set_configuration(udev, c); 
            if (err && err != -ENODEV) {
                dev_err(&udev->dev, "can't set config #%d, error %d\n",
                    c, err);
                /* This need not be fatal.  The user can try to
                 * set other configurations. */
            }   
        }   
    }   
    /* USB device state == configured ... usable */
    usb_notify_add_device(udev);

    return 0;
}

```



### 3.6 执行function driver 驱动 probe

```c
注册usb_driver
int usb_register_driver(struct usb_driver *new_driver, struct module *owner,
            const char *mod_name)
{
    int retval = 0; 

    if (usb_disabled())
        return -ENODEV;

    new_driver->drvwrap.for_devices = 0; 
    new_driver->drvwrap.driver.name = new_driver->name;
    new_driver->drvwrap.driver.bus = &usb_bus_type;
    new_driver->drvwrap.driver.probe = usb_probe_interface;
    new_driver->drvwrap.driver.remove = usb_unbind_interface;
    new_driver->drvwrap.driver.owner = owner;
    new_driver->drvwrap.driver.mod_name = mod_name;
    spin_lock_init(&new_driver->dynids.lock);
    INIT_LIST_HEAD(&new_driver->dynids.list);

    retval = driver_register(&new_driver->drvwrap.driver);
    if (retval)
        goto out; 

    retval = usb_create_newid_files(new_driver);
    if (retval)
        goto out_newid;

    pr_info("%s: registered new interface driver %s\n",
            usbcore_name, new_driver->name);

out:
    return retval;

out_newid:
    driver_unregister(&new_driver->drvwrap.driver);

    printk(KERN_ERR "%s: error %d registering interface "
            "   driver %s\n",
            usbcore_name, retval, new_driver->name);
    goto out; 
}
执行匹配
static int usb_probe_interface(struct device *dev)
{
    ....
    error = driver->probe(intf, id);
    ....
}

```



