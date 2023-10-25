# 驱动device与driver的匹配

platform总线上有设备链表和驱动链表，注册设备会遍历总线上的驱动链表找寻与之匹配的驱动，注册驱动会遍历总线上的设备链表找寻与之匹配的设备

```c
device_add ---> ... ---> bus_probe_device ---> device_initial_probe ---> __device_attach ---> bus_for_each_drv ---> __device_attach_driver ---> platform_match
platform_driver_register ---> ... ---> bus_add_driver ---> ... ---> platform_match
```

首先，有一个平台总线platform_bus_type，总线上分别挂着平台设备platform_device和平台驱动platform_driver。
```c
struct bus_type platform_bus_type = {
        .name                = "platform",
        .dev_groups        = platform_dev_groups,
        .match                = platform_match,    // platform_device和platform_driver的匹配函数
        .uevent                = platform_uevent,
        .dma_configure        = platform_dma_configure,
        .pm                = &platform_dev_pm_ops,
};
```
所谓平台设备platform_device和平台驱动platform_driver，就是一个又一个platform_device和platform_driver结构体； 依次分析

## platform_device和platform_driver结构体。

```c
platform_device结构体
struct platform_device {
        const char        *name;
        int                id;
        bool                id_auto;
        struct device        dev;
        u32                num_resources;
        struct resource        *resource;
 
        const struct platform_device_id        *id_entry;
        char *driver_override; /* Driver name to force a match */
 
        /* MFD cell pointer */
        struct mfd_cell *mfd_cell;
 
        /* arch specific additions */
        struct pdev_archdata        archdata;
};
```
platform_device主要有3个成员与匹配相关：
```c
platform_device->driver_override；
platform_device.dev->of_node（主要是of_node里面包含的节点的compatible属性）；
platform_device.name；
```
platform_driver结构体
```c
struct platform_driver {
        int (*probe)(struct platform_device *);
        int (*remove)(struct platform_device *);
        void (*shutdown)(struct platform_device *);
        int (*suspend)(struct platform_device *, pm_message_t state);
        int (*resume)(struct platform_device *);
        struct device_driver driver;
        const struct platform_device_id *id_table;
        bool prevent_deferred_probe;
};

struct platform_device_id {
        char name[PLATFORM_NAME_SIZE];
        kernel_ulong_t driver_data;
};
struct device_driver {
        const char                *name;
        const struct of_device_id        *of_match_table;
    ...
};
struct of_device_id {
        char        name[32];
        char        type[32];
        char        compatible[128];
        const void *data;
};
```
类似的，platform_driver也有3个成员与匹配相关：
```c
platform_driver.driver->name；
platform_driver.driver->of_match_table;
platform_driver->id_table;
```
## platform_match

platform_device和platform_driver的匹配，是通过平台总线platform_bus_type的match成员实现的。
match成员是一个函数指针，指向platform_match函数，这个函数就是专门用来匹配platform_device和platform_driver的。
重点分析一下platform_match函数。

综上所述，device与driver的结构体参数如下
```c
platform_device->driver_override；
platform_device.dev->of_node（主要是of_node里面包含的节点的compatible属性）；
platform_device.name；

platform_driver.driver->name；
platform_driver.driver->of_match_table;
platform_driver->id_table;
```

/**
 * platform_match - bind platform device to platform driver.
 * @dev: device.
 * @drv: driver.
 *
 * Platform device IDs are assumed to be encoded like this:
 * "<name><instance>", where <name> is a short description of the type of
 * device, like "pci" or "floppy", and <instance> is the enumerated
 * instance of the device, like '0' or '42'.  Driver IDs are simply
 * "<name>".  So, extract the <name> from the platform_device structure,
 * and compare it against the name of the driver. Return whether they match
 * or not.
 */
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    /* When driver_override is set, only bind to the matching driver */
    if (pdev->driver_override)
        return !strcmp(pdev->driver_override, drv->name);

    /* Attempt an OF style match first */
    if (of_driver_match_device(dev, drv))
        return 1;

    /* Then try ACPI style match */
    if (acpi_driver_match_device(dev, drv))
        return 1;

    /* Then try to match against the id table */
    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    /* fall-back to driver name match */
    return (strcmp(pdev->name, drv->name) == 0);
}
```c
总结（匹配优先级）
最后，匹配的优先级总结如下：
比较 platform_dev.driver_override 和 platform_driver.drv->name；
比较 platform_dev.dev.of_node的compatible属性 和 platform_driver.drv->of_match_table；
比较 platform_dev.name 和 platform_driver.id_table；
比较 platform_dev.name 和 platform_driver.drv->name；
有一个匹配成功，即表示匹配成功，会调用platform_driver->probe函数。
https://blog.csdn.net/qq_33141353/article/details/127481859
```

## 源码
注册 platform_driver 的过程
主要分为以下4步：
1. 将platform_driver放入platform_bus_type的driver链表中；
2. 对于platform_bus_type下的每一个device，调用__driver_attach；
3. 调用 platform_bus_type.match，判断dev和drv是否匹配成功；
如果匹配成功，最终会调用drv的probe函数；
```c
platform_driver_register
    __platform_driver_register
        drv->driver.probe = platform_drv_probe;
        driver_register
            bus_add_driver
                klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);    // 把 platform_driver 放入 platform_bus_type 的driver链表中
                driver_attach
                    bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);  // 对于plarform_bus_type下的每一个设备, 调用__driver_attach
                        __driver_attach
                            ret = driver_match_device(drv, dev);  // 判断dev和drv是否匹配成功
                                        return drv->bus->match ? drv->bus->match(dev, drv) : 1;  // 调用 platform_bus_type.match
                            __device_attach_driver
                                driver_probe_device(drv, dev);
                                            really_probe
                                                drv->probe  // platform_drv_probe
                                                    platform_drv_probe
                                                        struct platform_driver *drv = to_platform_driver(_dev->driver);
                                                        drv->probe
```
注册 platform_device 的过程
与注册 platform_driver 类似，也可以分为以下几步：
1. 将 platform_device 放入platform_bus_type的device链表中；
2. 对于platform_bus_type下的每一个driver，调用__device_attach_driver；
3. 调用 platform_bus_type.match，判断dev和drv是否匹配成功；
4. 如果匹配成功，最终会调用drv的probe函数；

```c
platform_device_register
    platform_device_add
        device_add
            bus_add_device
                klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devices); // 把 platform_device 放入 platform_bus_type的device链表中
            bus_probe_device(dev);
                device_initial_probe
                    __device_attach
                        ret = bus_for_each_drv(dev->bus, NULL, &data, __device_attach_driver); // // 对于plarform_bus_type下的每一个driver, 调用 __device_attach_driver
                                    __device_attach_driver
                                        ret = driver_match_device(drv, dev);
                                                    return drv->bus->match ? drv->bus->match(dev, drv) : 1;  // 调用platform_bus_type.match
                                        driver_probe_device
```