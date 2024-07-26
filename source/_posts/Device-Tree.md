---
title: 设备树Device Tree内容简述
date: 2021-10-25 19:35:57
tags: dts
categories: 
 - kernel
---

# Device Tree 梳理与理解
设备树，可以理解为描述板级设备的树形结构。它的引入解决了传统linux板级代码冗余，驱动与硬件绑定严重的问题，使linux驱动变得更加优雅，Linux内核发展过程中的一个重要的里程碑。
**本质**：提供一种语言来解耦硬件配置信息
# 1. 设备树的引入背景
## 1.1 无设备树存在的问题
在没有设备树之前，Linux内核在处理不同硬件平台的板级配置信息时存在一系列问题，这些问题主要体现在以下几个方面：
**板级代码冗余**：
在Linux内核v2.6版本之前，ARM架构描述不同硬件信息的文件存放在arch/arm/文件夹下。每新出一款芯片及其对应的开发板，就要多一个板级信息文件，导致板级信息文件数量成指数级增长。这些文件为.c或.h，它们被编译进Linux内核中，造成了内核代码的冗余。且不同开发板的硬件配置不同，此类代码的可复用性低。

**驱动与硬件绑定严重**：
在没有设备树之前，硬件的具体信息（如GPIO、寄存器地址、中断号等）会直接写在驱动中。硬件资源发生变化时，驱动程序需要重新编写和编译，增加了开发和维护的难度。

## 1.2 引入设备树
设备树起源于OpenFirmware，是一种用于描述硬件平台硬件资源信息的数据结构和语言。引入设备树的优势：
**驱动与硬件分离**：
设备树通过描述硬件平台的硬件资源信息（如CPU数量、内存基地址、外设连接等），使得驱动代码不再需要直接包含这些硬件信息。当硬件资源发生变化时，只需修改设备树文件而无需修改驱动代码，提高了代码的可维护性和可移植性。

**减少代码冗余**：
通过将板级配置信息独立存放到设备树文件中（如.dts和.dtsi文件），Linux内核不再需要包含大量的板级配置代码。这有助于减少内核代码的冗余和复杂性，使内核更加简洁和高效。

**提高内核启动效率**：
引入设备树后，bootloader可以直接将dtb文件传递给内核。内核在启动过程中解析设备树文件来获取硬件信息，简化了启动流程并提高了启动效率。
**支持更多硬件设备**：
设备树提供了一种标准化的方式来描述硬件资源信息，使得Linux内核能够更容易地支持更多种类的硬件设备。

# 2. 设备树使用流程
1. 用户根据硬件配置和系统运行参数，并把这些信息组织成设备树源代码(DTS，Device Tree source)文件，共同的部分一般被提炼为一个.dtsi 文件，这个文件相当于C语言的头文件。
2. 通过DTC(Device Tree Compiler)，将DTS编译成适合机器处理的DTB(Device Tree binary)文件。
3. 在系统启动的时候，Bootloader将保存在flash中的DTB拷贝到内存，并把DTB的起始地址传递给kernel。

# 3. 设备树结构
设备树根据字面意思也可以理解：一个由设备组成的树形结构，大概是下面这个样子
![](image.png)
而实际的设备树代码和这个结构很像，设备树源码示例如下：

```C
/ {
	model = "LS1012A Freedom Board";
	compatible = "fsl,ls1012a-frdm", "fsl,ls1012a";
	chosen {
	bootargs = "console=ttyS0,115200 
	loglevel=8";
	};
	cpus {
		#address-cells = <1>;
		#size-cells = <0>;

		cpu0: cpu@0 {
			compatible = "arm,cortex-a53";
			device_type = "cpu";
			reg = <0>;
			enable-method = "psci";
			next-level-cache = <&L2>;
			clocks = <&ccu CLK_CPUX>;
			clock-names = "cpu";
			#cooling-cells = <2>;
		};

		L2: l2-cache {
			compatible = "cache";
			cache-level = <2>;
			cache-unified;
		};
	};

	soc {
		compatible = "simple-bus";
		#address-cells = <1>;
		#size-cells = <1>;
		ranges;

		bus@1000000 {
			compatible = "allwinner,sun50i-a64-de2";
			reg = <0x1000000 0x400000>;
			allwinner,sram = <&de2_sram 1>;
			#address-cells = <1>;
			#size-cells = <1>;
			ranges = <0 0x1000000 0x400000>;

			display_clocks: clock@0 {
				compatible = "allwinner,sun50i-a64-de2-clk";
				reg = <0x0 0x10000>;
				clocks = <&ccu CLK_BUS_DE>, <&ccu CLK_DE>;
				clock-names = "bus", "mod";
				resets = <&ccu RST_BUS_DE>;
				#clock-cells = <1>;
				#reset-cells = <1>;
			};
		};

		syscon: syscon@1c00000 {
			compatible = "allwinner,sun50i-a64-system-control";
			reg = <0x01c00000 0x1000>;
			#address-cells = <1>;
			#size-cells = <1>;
			ranges;
			sram_c: sram@18000 {
				compatible = "mmio-sram";
				reg = <0x00018000 0x28000>;
				#address-cells = <1>;
				#size-cells = <1>;
				ranges = <0 0x00018000 0x28000>;

				de2_sram: sram-section@0 {
					compatible = "allwinner,sun50i-a64-sram-c";
					reg = <0x0000 0x28000>;
				};
			};
		};

		mmc0: mmc@1c0f000 {
			compatible = "allwinner,sun50i-a64-mmc";
			reg = <0x01c0f000 0x1000>;
			clocks = <&ccu CLK_BUS_MMC0>, <&ccu CLK_MMC0>;
			clock-names = "ahb", "mmc";
			resets = <&ccu RST_BUS_MMC0>;
			reset-names = "ahb";
			interrupts = <GIC_SPI 60 IRQ_TYPE_LEVEL_HIGH>;
			max-frequency = <150000000>;
			status = "disabled";
			#address-cells = <1>;
			#size-cells = <0>;
		};

		timer@1c20c00 {
			compatible = "allwinner,sun50i-a64-timer",
				     "allwinner,sun8i-a23-timer";
			reg = <0x01c20c00 0xa0>;
			interrupts = <GIC_SPI 18 IRQ_TYPE_LEVEL_HIGH>,
				     <GIC_SPI 19 IRQ_TYPE_LEVEL_HIGH>;
			clocks = <&osc24M>;
		};

		uart0: serial@1c28000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x01c28000 0x400>;
			interrupt-parent = <&gic>;
			interrupts = <GIC_SPI 0 IRQ_TYPE_LEVEL_HIGH>;
			reg-shift = <2>;
			reg-io-width = <4>;
			clocks = <&ccu CLK_BUS_UART0>;
			resets = <&ccu RST_BUS_UART0>;
			status = "disabled";
		};

		gic: interrupt-controller@1c81000 {
			compatible = "arm,gic-400";
			reg = <0x01c81000 0x1000>,
			      <0x01c82000 0x2000>,
			      <0x01c84000 0x2000>,
			      <0x01c86000 0x2000>;
			interrupts = <GIC_PPI 9 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_HIGH)>;
			interrupt-controller;
			#interrupt-cells = <3>;
		};
	};
};
```
根据以上设备树可以得到一些粗浅的信息：
* 设备树含1个root节点(node)"/"
* root节点下面含一系列子节点，本例中为cpus和soc
* 设备节点被组织成树状结构，除了根节点，每个node都只有一个parent
* 每个节点包含一系列属性，属性值可为空如：cache-unified
* 节点名字的格式是node-name@unit-address

以上信息只是一些对设备树比较基本的认识，下面将说明细节。

# 4. 设备树语法

## 4.1 dtsi头文件
和C语言类似，设备树通用支持头文件，设备树的头文件扩展名为.dtsi，可以通过include来包含dtsi和头文件
```C
#include <dt-bindings/interrupt-controller/irq.h>
#include "fsl-ls1012a.dtsi"
```
一般.dtsi 文件用于描述 SOC 的内部外设信息，比如 CPU 架构、主频、外设寄存器范围等。

## 4.2 设备节点
在设备树中，每个设备都是一个节点；节点中包含很多属性信息，通过**键值对**的方式描述具体信息。
有以下信息需要注意：
* dtsi和dts中均可以包含根节点，两个或多个根节点会合并成一个
* 设备节点可以用":"隔开，前面是节点标签(label)，后面是节点名字
* 节点以NodeName@address的形式表示，NodeName是节点的名字，address是设备的寄存器首地址，若节点没有地址或寄存器，不需要加地址如：
```C
	cpu0: cpu@0 { 
	};
	uart0: serial@1c28000 {
	};
```
* 属性值可以是字符串、32位无符号整形和字符串列表
  + 字符串：compatible = "cache";
  + 32位无符号整形：max-frequency = <150000000>;
  + 字符串列表：compatible = "allwinner，sun50i-a64-timer", "allwinner,sun8i-a23-timer";

## 4.3 标准属性值
设备节点由一系列属性组成，不同的设备需要的属性不同，一些linux下通用的属性可以称为标准属性，此外用户也可以**自定义属性**。
相关解释可以在/kernel/Documentation/devicetree/bindings文档中查看。


### 4.3.1 compatible
compatible, 兼容性，其属性值为字符串列表，compatible 属性用于将设备和驱动绑定起来。设备驱动中of_device_id.compatible中会有对应s项与之匹配。
```C
compatible = "snps,dw-apb-uart";
```
root结点"/"的compatible 属性 compatible = "fsl,ls1012a-frdm", "fsl,ls1012a";定义了系统的名称，它的组织形式为："manufacturer,model"。Linux内核通过root结点"/"的compatible 属性即可判断启动的是什么machine。
### 4.3.2 model
```C
model = "LS1012A Freedom Board";
```
用于描述设备的型号或制造商信息。字符串类型属性，格式通常为“manufacturer,model-number”。此属性为系统提供了一种识别设备型号的方式，有助于在系统日志、调试信息或用户界面中正确地标识设备。

### 4.3.3 status
```C
status = "disabled";
```
描述设备的状态信息，值如下：
**okay**：表示设备正常可操作的，当此属性值不配置也认为是okay的
**disabled**：设备不可操作，禁用状态
**reserved**：表示设备是可操作的，但不应被系统直接使用
**fail**：表示设备不可操作，且很可能是由于严重的错误导致的，通常需要维修或更换

一般只使用okay和disabled，后两种值很少使用。

### 4.3.4 reg、address-cells和size-cells
```C
	#address-cells = <1>;
	#size-cells = <1>;
	reg = <0x01c00000 0x1000>;
```
这三个属性共同组合，用于准确地描述设备的物理地址空间。
reg属性值由一串数字组成:
```C
reg = <address1 length1 address2 length2 address3 length3……>
```
这些数据分为地址数据（地址字段），长度数据（大小字段）。
#address-cells：指定子节点reg属性“地址字段”所占的长度（单元格cells的个数） 
#size-cells：指定子节点reg属性“大小字段”所占的长度
如上图中的reg = <0x01c00000 0x1000>：
0x01c00000是地址的起始值，表示这个地址空间的起始物理地址
0x1000是这个地址空间的大小，以字节为单位
这个地址空间覆盖了从0x01c00000到0x01c00FFF的4096字节（因为0x1000等于4096）的物理内存区域。
其他:
* 当#size-cells的值为0时，表示在reg属性中不包含地址空间大小的信息。通常用于那些只需要指定起始地址而不需要指定大小的设备或资源。
* 64位情况，拆成高32位和低32位。
```C
	#address-cells = <2>;
	#size-cells = <2>;
	reg = <0x0 0x12420000 0x0 0x10000>, <0x0 0x12430000 0x0 0x10000>;
```

### 4.3.5 ranges
```C
ranges = <0 0x1000000 0x400000>;
```
描述子设备地址空间如何映射到父设备地址空间
ranges属性值的格式 <local地址， parent地址， size>。
比如对于#address-cells和#size-cells都为1的情况，以<0x0  0x10 0x20>为例，表示将local的从0x0\~(0x0 + 0x20)的地址空间映射到parent的0x10~(0x10 + 0x20)
 
解释：
1. 映射表中的子地址、父地址分别采用子地址空间的#address-cells和父地址空间的#address-cells大小
2. 对于含有ranges属性的节点的子节点来说，其reg都是基于local地址的
3. ranges属性值为空的话，表示1:1映射
4. 对于没有ranges属性的节点，代表不是memory map区域
![](image-1.png)

### 4.3.6 interrupt
interrupt-controller，中断控制器属性，GIC通过此属性表明身份；
```C
	gic: interrupt-controller {
	};
```
#interrupt-cells：表明连接此中断控制器的设备interrupts属性的cell的大小。
interrupt-parent：指定所依附的中断控制器，当结点没有指定interrupt-parent 时，则从父级结点继承。
```C
interrupt-parent = <&gic>;
```
interrupts: 指定中断类型、中断号和触发方法等，具体这个属性含有多少个cell，由它依附的中断控制器结点的#interrupt-cells属性决定。
```C
	interrupts = <GIC_SPI 18 IRQ_TYPE_LEVEL_HIGH>;
```
与中断类似，像clock、GPIO、pinmux和dma等属性都可以通过引用的方式进行属性描述。

# 5.设备树核心功能总结
**平台标识**：根节点compatible值标识特定的machine
**运行时配置**：如chosen节点声明console参数
**设备信息集合**：通过设备树配置具体的信息

# 6.设备树解析API（v5.19.7）
在Linux的kernel代码中，存在很多设备树相关API,这些API在/kernel/drivers/of目录下，通常以of_前缀，理解这些API对熟悉设备树有很大的作用。
## 6.1 of_machine_is_compatible
```C
int of_machine_is_compatible(const char *compat)；
```
遍历设备树的根节点，并检查其compatible属性是否包含给定的compat字符串。
compat：表示要在根节点的compatible属性中查找的兼容性字符串。
返回值：若根节点的compatible属性中包含给定的compat字符串，则函数返回一个正整数，表示匹配成功。否则表示匹配失败。
## 6.2 of_device_is_compatible
```C
int of_device_is_compatible(const struct device_node *device, const char *compat)；
```
检查一个特定的设备节点（device_node）是否与给定的兼容性字符串（compat）相匹配。
*device：代表设备树中的一个节点。
*compat：表示要检查的兼容性标识符。
返回值：如果设备节点的compatible属性中包含给定的compat字符串，则函数通常返回一个非零值，表示匹配成功。如果未找到匹配项，则返回零，表示匹配失败。

## 6.3 of_find_compatible_node
```C
struct device_node *of_find_compatible_node(struct device_node *from, const char *type, const char *compatible);
```
查找与给定类型和兼容性字符串相匹配的第一个设备节点。
*from：表示搜索的起始点。若此参数为NULL，搜索从设备树的根节点开始。
*type：表示要搜索的设备节点的类型。
*compatible：表示要搜索的兼容性字符串。
返回值：返回一个指针，指向找到的device_node结构体，该结构体代表了与给定类型和兼容性字符串相匹配的第一个设备节点。如果没有找到匹配的节点，返回NULL。

## 6.4 属性值相关
```C
int of_property_read_u8_array(const struct device_node *np,const char *propname, u8 *out_values, size_t sz);
int of_property_read_u16_array(const struct device_node *np,const char *propname, u16 *out_values, size_t sz);
int of_property_read_u32_array(const struct device_node *np,const char *propname, u32 *out_values, size_t sz);
int of_property_read_u64(const struct device_node *np, const char*propname, u64 *out_value);
```
以of_property_read_u32_array为例进行说明：
用于从设备树中的一个节点读取一个无符号32位整数数组的属性值。
*np：表示要从中读取属性的设备树节点。
*propname：表示要读取的属性的名称。
*out_values：用于存储从设备树节点中读取的属性值。
sz：这个参数指定了out_values数组的大小。
```C
static inline int of_property_read_u8(const struct device_node *np, const char *propname, u8 *out_value)；
static inline int of_property_read_u16(const struct device_node *np, const char *propname, u16 *out_value)；
static inline int of_property_read_u32(const struct device_node *np, const char *propname, u32 *out_value)；
static inline int of_property_read_s32(const struct device_node *np, const char *propname, s32 *out_value)；
```
以of_property_read_s32为例说明，功能为从设备树中的一个节点读取一个有符号32位整数（s32）的属性值。
*np：表示要从中读取属性的设备树节点。
*propname：表示要读取的属性的名称。
*out_values：用于存储从设备树节点中读取的属性值。
返回值：如果成功读取了属性并且值被成功复制到out_value中，则返回零。如果发生错误，则返回非零错误码。
```C
int of_property_read_string(struct device_node *np, const char*propname, const char **out_string);
```
从设备树中的一个节点读取一个字符串类型的属性值。
*np：表示要从中读取属性的设备树节点。
*propname：表示要读取的属性的名称。
**out_string：存储从设备树节点中读取的字符串值的地址。

```C
int of_property_read_string_index(struct device_node *np, const char*propname, int index, const char **output);
```
用于从节点读取一个**字符串数组**类型属性中的特定索引位置的字符串值。在设备树中，某些属性可能包含多个字符串值，这些值被组织成一个数组，此函数允许指定要读取的数组中的索引位置。
```C
static inline bool of_property_read_bool(const struct device_node *np, const char *propname)
```
从节点读取布尔值。

## 6.5 内存映射相关
```C
void __iomem *of_iomap(struct device_node *device, int index);
```
根据设备树节点中定义的资源信息，将指定的I/O内存资源映射到内核的虚拟地址空间中，以便内核代码可以直接访问这些物理内存区域。
*device：表示包含I/O内存资源定义的设备树节点。
index：指定了要映射的I/O内存资源的索引。设备树节点中可能定义了多个I/O内存资源，这个索引用于选择其中的一个。索引从0开始。
返回值：函数返回一个指向映射后的I/O内存区域的指针。这个指针的类型是void __iomem *，这是一个特殊的指针类型，用于表示内存映射的I/O区域。这个指针不能直接像访问普通内存那样访问它，而应该使用专门的I/O访问函数（如ioread32()、iowrite32()等）来访问。
```C
int of_address_to_resource(struct device_node *dev, int index, struct resource *r)；
```
用于从节点中提取I/O资源信息，并将其填充到一个struct resource结构体中的函数。它允许驱动程序获取关于设备I/O资源（如内存映射区域、I/O端口）的详细信息。
*dev：表示包含I/O资源定义的设备树节点。
index：指定了要提取的I/O资源的索引。
*r：函数将提取的I/O资源信息填充到这个结构体中。

## 6.6 irq_of_parse_and_map
```C
int irq_of_parse_and_map(struct device_node *node, int index);
```
解析并映射中断号（IRQ number）的函数
*node：指向device_node结构体的指针，表示包含中断信息定义的设备树节点。
index：这是一个整数，指定了要解析的中断信息的索引。
返回值int：如果成功，它将返回有效的中断号；如果失败，则返回特定的错误码，如-EINVAL（无效参数）、-ENOENT（未找到条目）等。
## 6.7 of_find_device_by_node
```C
struct platform_device *of_find_device_by_node(struct device_node *np);
```
根据节点查找并返回对应的platform_device结构体。
*np：表示要在设备树中查找的节点。这个节点应该代表了一个平台设备。
返回值：指向platform_device结构体的指针，该结构体与提供的设备树节点（np）相对应。如果找不到对应的platform_device，则返回NULL。

# 7. 总结
设备树的内容，看似简单，实际内容多且繁杂，本文只是简单对设备树的内容进行说明，关于kernel如何对设备树解析等其他复杂内容还没分析。