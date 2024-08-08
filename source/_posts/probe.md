---
title: platform驱动注册与probe过程
date: 2023-08-08 20:45:26
tags: platform_device
categories: 
 - kernel
---
# 1. 文章目的
1. 梳理platform driver注册过程
2. 试图理清platform driver 和platform device是怎么match上的
# 2. platform driver 注册过程分析
之前在[设备解析与platform设备创建过程分析](https://wenbin666.github.io/2023/08/07/platform-device/)中，已经分析了kernel是如何解析设备树并创建platform_device的，现在直接分析设备驱动注册过程
## 2.1 platform_driver_register
```C
#define platform_driver_register(drv) \
	__platform_driver_register(drv, THIS_MODULE)
int __platform_driver_register(struct platform_driver *drv, struct module *owner)
{
	drv->driver.owner = owner;
	drv->driver.bus = &platform_bus_type;
	return driver_register(&drv->driver);
}
```
设置driver的bus的类型为platform_bus_type,并调用driver_register接口。

## 2.2 driver_register
```C
int driver_register(struct device_driver *drv)
{
	int ret;
    ...
	ret = bus_add_driver(drv);
	if (ret)
		return ret;
	ret = driver_add_groups(drv, drv->groups);
	if (ret) {
		bus_remove_driver(drv);
		return ret;
	}
	kobject_uevent(&drv->p->kobj, KOBJ_ADD);
	deferred_probe_extend_timeout();
	return ret;
}
```
逐个分析关键函数，分析之前先分析两个数据结构，后面的梳理会用的到：
### 2.2.1 subsys_private
```C
struct subsys_private {
	struct kset subsys;
	struct kset *devices_kset;
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset;
	struct klist klist_devices;
	struct klist klist_drivers;
	struct blocking_notifier_head bus_notifier;
	unsigned int drivers_autoprobe:1;
	const struct bus_type *bus;
	struct device *dev_root;

	struct kset glue_dirs;
	const struct class *class;

	struct lock_class_key lock_key;
};
```
subsys_private用于在Linux内核中表示和管理一个子系统:
**kset** : 集合结构，subsys 成员是子系统自身的 kset 集合，用于组织该子系统下的所有对象。
**devices_kset**: 用于遍历和管理子系统下的所有设备。
**interfaces**: 链接该子系统或总线上的所有接口, 通常用于定义设备与驱动程序之间的通信方式。
**mutex** :互斥锁，用于保护这个结构的访问，防止在多线程或多处理器环境中发生数据竞争。
**drivers_kset**: 管理属于这个子系统的所有驱动程序对象。
**klist_devices**: klist 内核链表，比标准链表提供了更多的功能，如自动去重。用于管理设备。
**klist klist_drivers**: 用于管理驱动程序的列表，可能用于快速查找或遍历。
**bus_notifier**: 用于在总线或子系统上注册和发送通知。
**drivers_autoprobe**:一个位字段，只有0和1两种值，表示是否自动探测驱动程序。
**bus**: 定义总线类型。
**dev_root**: 指向这个子系统或总线的根设备的指针。
**glue_dirs** :kset，用于管理一些特殊的目录或对象。
**class**: 用于确定这个子系统或总线下的设备应该属于哪个类。
**lock_key**: 锁类键，用于调试和性能分析,帮助内核跟踪和识别特定的锁使用情况。
### 2.2.2 driver_private
```C
struct driver_private {
	struct kobject kobj;
	struct klist klist_devices;
	struct klist_node knode_bus;
	struct module_kobject *mkobj;
	struct device_driver *driver;
};
```
driver_private用于表示和管理设备驱动程序私有信息的结构。
**kobj**：内核中的基础对象，用于表示内核中的几乎所有对象，如设备、驱动程序等
**klist_devices**：管理支持的设备对象,通过这个链表，驱动可以遍历和管理它所绑定的所有设备。
**knode_bus**：通过此节点，驱动可以被链接到它所属的总线或子系统的设备驱动列表中。
**mkobj**：指向module_kobject 的指针，允许驱动程序与其所属的模块建立关联。
**driver**：指向当前 driver_private 结构体所代表的驱动程序的 device_driver 结构体的指针。

## 2.3 bus_add_driver
```C
int bus_add_driver(struct device_driver *drv)
{
	struct subsys_private *sp = bus_to_subsys(drv->bus);
	struct driver_private *priv;
	int error = 0;

	if (!sp)
		return -EINVAL;

	/*
	 * Reference in sp is now incremented and will be dropped when
	 * the driver is removed from the bus
	 */
	pr_debug("bus: '%s': add driver %s\n", sp->bus->name, drv->name);

	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv) {
		error = -ENOMEM;
		goto out_put_bus;
	}
	klist_init(&priv->klist_devices, NULL, NULL);
	priv->driver = drv;
	drv->p = priv;
	priv->kobj.kset = sp->drivers_kset;
	error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
				     "%s", drv->name);
	if (error)
		goto out_unregister;

	klist_add_tail(&priv->knode_bus, &sp->klist_drivers);
	if (sp->drivers_autoprobe) {
		error = driver_attach(drv);
		if (error)
			goto out_del_list;
	}
	error = module_add_driver(drv->owner, drv);
	if (error) {
		printk(KERN_ERR "%s: failed to create module links for %s\n",
			__func__, drv->name);
		goto out_detach;
	}

	error = driver_create_file(drv, &driver_attr_uevent);
	if (error) {
		printk(KERN_ERR "%s: uevent attr (%s) failed\n",
			__func__, drv->name);
	}
	error = driver_add_groups(drv, sp->bus->drv_groups);
	if (error) {
		/* How the hell do we get out of this pickle? Give up */
		printk(KERN_ERR "%s: driver_add_groups(%s) failed\n",
			__func__, drv->name);
	}

	if (!drv->suppress_bind_attrs) {
		error = add_bind_files(drv);
		if (error) {
			/* Ditto */
			printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
				__func__, drv->name);
		}
	}

	return 0;

out_detach:
	driver_detach(drv);
out_del_list:
	klist_del(&priv->knode_bus);
out_unregister:
	kobject_put(&priv->kobj);
	/* drv->p is freed in driver_release()  */
	drv->p = NULL;
out_put_bus:
	subsys_put(sp);
	return error;
}
```
函数主要功能如下：
1. 通过kzalloc为driver_private分配内存。
2. 调用klist_init来初始化klist_devices链表（一个driver对应多个设备，在driver_private里用链表来挂载所有的devic）。
3. 通过kobject_init_and_add初始化和添加一个kobject对象到内核对象层次结构中。
4. 调用klist_add_tail把driver挂到bus的链表上，knode_bus就是挂载点，klist_drivers就是bus下挂的多个driver。
5. 判断driver所在的bus是否支持autoprobe，若支持，可以调用driver_attach来把driver绑定到device。在bus_init的时候会默认设置为1，所有此处调用。
6. module_add_driver，用于将一个设备驱动程序 (drv) 添加到其对应的模块 (mod) 的 sysfs 表示中，用于查看和管理。
7. 下面是file和groups相关内容。

重点分析driver_attach内容
## 2.4 driver_attach
```C
// driver_attach - try to bind driver to devices.
int driver_attach(const struct device_driver *drv)
{
	/* The (void *) will be put back to const * in __driver_attach() */
	return bus_for_each_dev(drv->bus, NULL, (void *)drv, __driver_attach);
}
```
遍历bus上所有设备，然后调用__driver_attach函数。
## 2.5 __driver_attach
```C
static int __driver_attach(struct device *dev, void *data)
{
    ...
    ret = driver_match_device(drv, dev);
    driver_probe_device(drv, dev);
    ...
}
```
## 2.6 driver_match_device
```c
static inline int driver_match_device(const struct device_driver *drv,
				      struct device *dev)
{
	return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}
```
通过之前的分析已经知道drv->driver.bus = &platform_bus_type，也就是说driver所在的bus为platform_bus。
```C
struct bus_type platform_bus_type = {
	.name		= "platform",
	.dev_groups	= platform_dev_groups,
	.match		= platform_match,
	.uevent		= platform_uevent,
	.probe		= platform_probe,
	.remove		= platform_remove,
	.shutdown	= platform_shutdown,
	.dma_configure	= platform_dma_configure,
	.dma_cleanup	= platform_dma_cleanup,
	.pm		= &platform_dev_pm_ops,
};
```
查看platform_match实现细节,正是**此成员函数决定了platform_device和platform_driver之前是如何匹配的！**
## 2.7 platform_match
```C
static int platform_match(struct device *dev, const struct device_driver *drv)
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
```
通过代码可以看出匹配platform_device和platform_driver有四种可能：
1. 基于设备树的匹配
2. 是基于ACPI风格的匹配
3. 基于ID table的匹配(即platform_device设备是否出现在platform_driver的ID表内)
4. 根据设备名字和驱动名字进行匹配。

目前最新版本内核都是通过设备树进行匹配
以常用驱动i2c为例：
设备树中:
```C
i2c1: i2c@30a20000 {
	compatible = "fsl,imx8mm-i2c", "fsl,imx21-i2c";
	#address-cells = <1>;
	#size-cells = <0>;
	reg = <0x30a20000 0x10000>;
	interrupts = <GIC_SPI 35 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clk IMX8MM_CLK_I2C1_ROOT>;
	status = "disabled";
};
```
驱动中：
```C
static const struct of_device_id i2c_imx_dt_ids[] = {
	{ .compatible = "fsl,imx7s-i2c", .data = &imx6_i2c_hwdata, },
	{ .compatible = "fsl,imx8mm-i2c", .data = &imx6_i2c_hwdata, },
	{ .compatible = "fsl,imx8mn-i2c", .data = &imx6_i2c_hwdata, },
	{ /* sentinel */ }
};
```
根据属性值即可完成匹配！
## 2.8 driver_probe_device
```C
driver_probe_device
 -> __driver_probe_device(drv, dev);
  -> really_probe
   -> call_driver_probe
```
最终调用到call_driver_probe
```C
static int call_driver_probe(struct device *dev, const struct device_driver *drv)
{
	int ret = 0;

	if (dev->bus->probe)
		ret = dev->bus->probe(dev);
	else if (drv->probe)
		ret = drv->probe(dev);
    ...
}
```
若dev有probe函数，则调用device的probe，否则调用driver的probe函数

# 总结
1. 驱动调用platform_driver_register()函数注册platform_driver。
2. 当platform_driver被注册后，内核会遍历所有已注册的platform_device，使用匹配函数platform_match匹配设备与驱动。
3. probe函数成功执行，设备和驱动将被绑定在一起，设备的驱动指针将指向该驱动，驱动的设备列表中也会加入该设备。