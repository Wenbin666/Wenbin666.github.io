---
title: 设备解析与platform设备创建过程分析
date: 2023-08-07 20:19:41
tags: platform_device
categories: 
 - kernel
---
# 1.文章目的
试图理清两件事情：
1. kernel是如何解析设备树的
2. 解析后的设备节点又是怎样被注册到platform bus上的

# 2.kernel解析设备树过程分析
```C
//kernel/init/main.c
start_kernel
 -> setup_arch(&cmdline) //uboot传过来的cmdline，用于配置系统arch
  -> setup_machine_fdt(__fdt_pointer); //__fdt_pointer为dtb的物理地址
  -> unflatten_device_tree();//根据设备描述结构体生成对应设备节点结构体
```
整体过程可以分成两部分：
**setup_machine_fdt**：设备树解析到device node过程
**unflatten_device_tree**：device node到platform_device过程
## 2.1 setup_machine_fdt
### 2.1.1 setup_machine_fdt源码
```C
static void __init setup_machine_fdt(phys_addr_t dt_phys)
{
    int size;                           
    void *dt_virt = fixmap_remap_fdt(dt_phys, &size, PAGE_KERNEL);
    if (!dt_virt || !early_init_dt_scan(dt_virt)) {
		...
        while (true)
            cpu_relax();
    }
	...
}
```
fixmap_remap_fdt函数的作用是为dtb所在的物理内存区域建立映射，使其能够被内核在虚拟地址空间中访问。
然后调用early_init_dt_scan进行进一步dtb分析
### 2.1.2 early_init_dt_scan
```C
bool __init early_init_dt_scan(void *params)
{
	bool status;
	status = early_init_dt_verify(params);
	if (!status)
		return false;
	early_init_dt_scan_nodes();
	return true;
}
```
通过early_init_dt_verify进行dtb验证，通过early_init_dt_scan_nodes进行设备节点扫描。

### 2.1.3 early_init_dt_verify
```C
early_init_dt_verify
 -> fdt_check_header  //检查设备树有效性
  -> early_init_dt_scan_root(); //扫描设备树根节点
```
* fdt_check_header 函数通过检查设备树头部的几个关键字段(如MAGIC、大小、版本等)。
* early_init_dt_scan_root 扫描设备树的根节点,并初始化{size,address}等cells info信息。

### 2.1.4 early_init_dt_scan_nodes
```C
void __init early_init_dt_scan_nodes(void)
{
	int rc;
	rc = early_init_dt_scan_chosen(boot_command_line);
	early_init_dt_scan_memory();
	early_init_dt_check_for_usable_mem_range();
}
```
总体可以分为三部分：scan_chosen节点、scan_memory和检查可用memory范围。
### 2.1.5 early_init_dt_scan_chosen
```C         
early_init_dt_scan_chosen
 -> node = fdt_path_offset(fdt, "/chosen"); //返回chosen节点
  -> p = of_get_flat_dt_prop(node, "bootargs", &l); 
   -> if (p != NULL && l > 0) 
    -> strscpy(cmdline, p, min(l, COMMAND_LINE_SIZE));
```
根据chosen节点，获取bootargs等参数

### 2.1.6 early_init_dt_scan_memory
```C
early_init_dt_scan_memory
 -> fdt_for_each_subnode(node, fdt, 0) 
  -> of_get_flat_dt_prop(node, "device_type", NULL);
   -> of_fdt_device_is_available(fdt, node))
    -> of_get_flat_dt_prop(node, "linux,usable-memory", &l);
	  -> reg = of_get_flat_dt_prop(node, "reg", &l);
       -> early_init_dt_add_memory_arch(base, size);
```
主要内容如下：
1. 遍历设备树中的所有节点，查找类型为"memory"的节点
2. 解析memory节点属性：特别是"reg"属性，"linux,usable-memory"属性等。
3. 初始化memblock子系统：early_init_dt_add_memory_arch将这些信息添加到memblock子系统中。

**early_init_dt_check_for_usable_mem_range**
1. 检查并确认设备树中定义的物理内存区域中，哪些是可被Linux内核安全且有效地使用的。
2. 此过程通常涉及到对设备树中memory节点的进一步分析和筛选，以确保内核在后续的内存管理和访问过程中不会遇到不可预见的问题。

### 2.1.7 of_flat_dt_get_machine_name
```C
const char * __init of_flat_dt_get_machine_name(void)
{
	const char *name;
	unsigned long dt_root = of_get_flat_dt_root();

	name = of_get_flat_dt_prop(dt_root, "model", NULL);
	if (!name)
		name = of_get_flat_dt_prop(dt_root, "compatible", NULL);
	return name;
}
```
根据设备树根节点model和compatilbe属性获取machine name信息。

## 2.2 unflatten_device_tree
逐步分析下，是怎么通过设备树生成设备节点的。
```C
void __init unflatten_device_tree(void)
{
	void *fdt = initial_boot_params;
    ...
	__unflatten_device_tree(fdt, NULL, &of_root, early_init_dt_alloc_memory_arch, false);

	/* Get pointer to "/chosen" and "/aliases" nodes for use everywhere */
	of_alias_scan(early_init_dt_alloc_memory_arch);

	unittest_unflatten_overlay_base();
}
```

### 2.2.1 __unflatten_device_tree
```C
__unflatten_device_tree
 -> size = unflatten_dt_nodes(blob, NULL, dad, NULL);
  -> mem = dt_alloc(size + 4, __alignof__(struct device_node));
   -> ret = unflatten_dt_nodes(blob, mem, dad, mynodes);
```
函数原型：
```C
void *__unflatten_device_tree(const void *blob, struct device_node *dad, 
    struct device_node **mynodes, void *(*dt_alloc)(u64 size, u64 align), bool detached)；
```
传入参数： 
**blob**：指向设备树在内存中的起始地址
**dad**: root节点
**mynodes**：指向device_node结构的根节点的指针的指针，用于**存储解析后设备树根节点的地址**
**dt_alloc**：用于为设备树中的device_node和属性结构分配内存的函数
**detached**：指示解析操作是否应该在“detached”模式下进行
函数说明：
__unflatten_device_tree函数的主要作用包括以下几个步骤：

1. 解析设备树头信息：首先检查传入的FDT数据的头部信息，以确保数据的有效性。
2. 计算并分配内存：遍历FDT数据，计算所需的device_node结构和属性结构的总大小，并调用dt_alloc函数分配足够的内存空间。
3. 解析设备节点：通过递归或迭代的方式，从FDT数据中解析出所有的设备节点（device_node）和属性（property），并将它们填充到之前分配的内存空间中。
4. 在这个过程中，会构建出设备节点的树状结构，每个节点通过parent、child和sibling等指针与其他节点想关联。

# 3. device_node到platform_device分析
## 3.1 函数入口分析
**说明**：如果从C语言的开始函数start_kernel进行追溯，是找不到platform device这一部分转换的源头的。
事实上，这个转换过程的函数是of_platform_default_populate_init,它被调用的方式是这样一个声明：
```C
arch_initcall_sync(of_platform_default_populate_init);
```
其中of_platform_default_populate_init会调用of_platform_default_populate
## 3.1 of_platform_default_populate_init
```C
static int __init of_platform_default_populate_init(void)
{  
    ...
    of_platform_default_populate(NULL, NULL, NULL);
    ...
}
```
而of_platform_default_populate的函数实现如下：
## 3.2 of_platform_default_populate
```C
const struct of_device_id of_default_bus_match_table[] = {
	{ .compatible = "simple-bus", },
	{ .compatible = "simple-mfd", },
	{ .compatible = "isa", },
#ifdef CONFIG_ARM_AMBA
	{ .compatible = "arm,amba-bus", },
#endif /* CONFIG_ARM_AMBA */
	{} /* Empty terminated list */
};

int of_platform_default_populate(struct device_node *root,
				 const struct of_dev_auxdata *lookup,
				 struct device *parent)
{
	return of_platform_populate(root, of_default_bus_match_table, lookup,
				    parent);
}
```
官方对of_platform_populate的解释翻译如下：
与of_platform_bus_probe()类似，该函数遍历设备树并从节点创建设备。它的不同之处在于它遵循现代惯例，要求所有设备节点具有'compatible'属性，并且它适合创建根节点的子设备。
## 3.3 of_platform_populate
```C
int of_platform_populate(struct device_node *root,
			const struct of_device_id *matches,
			const struct of_dev_auxdata *lookup,
			struct device *parent)
{
	struct device_node *child;
	int rc = 0;

	root = root ? of_node_get(root) : of_find_node_by_path("/");
	if (!root)
		return -EINVAL;

	pr_debug("%s()\n", __func__);
	pr_debug(" starting at: %pOF\n", root);

	device_links_supplier_sync_state_pause();
	for_each_child_of_node(root, child) {  //遍历每个节点，并调用platform接口
		rc = of_platform_bus_create(child, matches, lookup, parent, true);
		if (rc) {
		    of_node_put(child);
        	break;
		}
	}
	device_links_supplier_sync_state_resume();

	of_node_set_flag(root, OF_POPULATED_BUS);

	of_node_put(root);
	return rc;
}
```

## 3.4 of_platform_bus_create
```C
of_platform_bus_create
 -> of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
  -> of_match_node(matches, bus);
```
底层进一步调用of_platform_device_create_pdata，根据名称猜测为分配实际数据。
## 3.5 of_platform_device_create_pdata
```C
static struct platform_device *of_platform_device_create_pdata(
					struct device_node *np,
					const char *bus_id,
					void *platform_data,
					struct device *parent)
{
	struct platform_device *dev;  //终于看到platform_device设备了

	pr_debug("create platform device: %pOF\n", np);

	if (!of_device_is_available(np) ||
	    of_node_test_and_set_flag(np, OF_POPULATED))
		return NULL;

	dev = of_device_alloc(np, bus_id, parent);
	if (!dev)
		goto err_clear_flag;

	dev->dev.coherent_dma_mask = DMA_BIT_MASK(32);
	if (!dev->dev.dma_mask)
		dev->dev.dma_mask = &dev->dev.coherent_dma_mask;
	dev->dev.bus = &platform_bus_type;
	dev->dev.platform_data = platform_data;
	of_msi_configure(&dev->dev, dev->dev.of_node);

	if (of_device_add(dev) != 0) {
		platform_device_put(dev);
		goto err_clear_flag;
	}
	return dev;

err_clear_flag:
	of_node_clear_flag(np, OF_POPULATED);
	return NULL;
}
```
先通过of_device_alloc分配设备，然后调用of_device_add添加设备。
## 3.6 of_device_alloc
```C
struct platform_device *of_device_alloc(struct device_node *np,
				  const char *bus_id,
				  struct device *parent)
{
	struct platform_device *dev; 
	int rc, i, num_reg = 0;
	struct resource *res;

	dev = platform_device_alloc("", PLATFORM_DEVID_NONE); //通过platform接口分配设备
	if (!dev)
		return NULL;

	/* count the io resources */
	num_reg = of_address_count(np);

	/* Populate the resource table */
	if (num_reg) {
		res = kcalloc(num_reg, sizeof(*res), GFP_KERNEL);
		if (!res) {
			platform_device_put(dev);
			return NULL;
		}

		dev->num_resources = num_reg;
		dev->resource = res;
		for (i = 0; i < num_reg; i++, res++) {
			rc = of_address_to_resource(np, i, res);
			WARN_ON(rc);
		}
	}

	/* setup generic device info */
	device_set_node(&dev->dev, of_fwnode_handle(of_node_get(np)));
	dev->dev.parent = parent ? : &platform_bus;

	if (bus_id)
		dev_set_name(&dev->dev, "%s", bus_id);
	else
		of_device_make_bus_id(&dev->dev);

	return dev;
}
```
函数功能如下：
1. platform_device_alloc创建一个具体的platform_devices设备
2. 分配resource和parent，配置name等
## 3.7 of_device_add
```C
int of_device_add(struct platform_device *ofdev){
    ...
    return device_add(&ofdev->dev);
}
```
将当前platform_device中的struct device成员注册到系统device中，并为其在用户空间创建相应的访问节点。
## 3.8 总结
函数整体调用关系如下：
```C
of_platform_default_populate_init();
   -> of_platform_default_populate();
    -> of_platform_populate();
     -> of_platform_bus_create();
      -> of_platform_device_create_pdata();
       -> of_device_alloc();
        -> of_device_add()         
```
经过一系列的操作，最终终于将device node变为platform_device

# 4.总结
通过分析可以得出以下内容：
1. kernel会将uboot传递过来的dtb进行解析，解析后得到是设备节点树
2. kernel启动过程中会通过一系列接口将设备节点树转换为具体的platform_device设备。
3. 当platform_driver去register的时候会去platform bus上寻找现有的platform_device，如果match，调用相应的probe函数。

借用网上一张总结比较好的图，找不到出处了。
![](image.png)