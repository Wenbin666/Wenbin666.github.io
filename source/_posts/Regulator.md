---
title: Linux Regulator子系统简介
date: 2025-12-24 21:50:48
tags: regulator
categories:
- kernel
---

---
## 1. 概述与架构

### 1.1 什么是 Regulator 子系统
Regulator（调节器/稳压器）子系统是 Linux 内核中用于管理电源供应的框架，它提供：
- 统一的电源管理接口
- 电压/电流调节控制
- 电源状态管理（启用/禁用）
- 功耗优化支持

### 1.2 系统架构
![架构图](arch.png)

### 1.3 核心概念
- Provider：调节器提供者（硬件驱动）
- Consumer：调节器消费者（使用电源的设备）
- Constraints：约束条件（电压/电流范围）
- Operations：操作集合（使能、设置电压等）

---

## 2. 核心数据结构
### 2.1 regulator_desc - 调节器描述符
```c
struct regulator_desc {
    const char *name;           // 调节器名称
    const char *supply_name;    // 上游电源名称
    int id;                     // 调节器ID
    unsigned n_voltages;        // 支持的电压档位数
    struct regulator_ops *ops;  // 操作函数集
    int irq;                    // 中断号
    enum regulator_type type;   // 类型（电压/电流）

    /* 电压范围 */
    unsigned int min_uV;
    unsigned int max_uV;
    unsigned int uV_step;

    /* 寄存器信息 */
    unsigned int vsel_reg;
    unsigned int vsel_mask;
    unsigned int vsel_shift;
    unsigned int enable_reg;
    unsigned int enable_mask;
    bool enable_is_inverted;

    /* 其他配置 */
    unsigned int ramp_delay;    // 压摆率（uV/us）
    unsigned int const *linear_ranges; // 线性范围表
};
```
### 2.2 regulator_ops - 操作函数集
```c
struct regulator_ops {
    /* 基本操作 */
    int (*enable)(struct regulator_dev *);
    int (*disable)(struct regulator_dev *);
    int (*is_enabled)(struct regulator_dev *);

    /* 电压控制 */
    int (*set_voltage)(struct regulator_dev *, int min_uV, int max_uV,
                       unsigned *selector);
    int (*get_voltage)(struct regulator_dev *);
    int (*list_voltage)(struct regulator_dev *, unsigned selector);

    /* 电流控制 */
    int (*set_current_limit)(struct regulator_dev *, int min_uA, int max_uA);
    int (*get_current_limit)(struct regulator_dev *);

    /* 模式控制 */
    int (*set_mode)(struct regulator_dev *, unsigned int mode);
    unsigned int (*get_mode)(struct regulator_dev *);
    int (*set_load)(struct regulator_dev *, int load_uA);

    /* 状态检查 */
    int (*get_status)(struct regulator_dev *);
    int (*get_error_flags)(struct regulator_dev *, unsigned int *flags);

    /* 特殊功能 */
    int (*set_voltage_time_sel)(struct regulator_dev *,
                                unsigned int old_selector,
                                unsigned int new_selector);
    int (*set_ramp_delay)(struct regulator_dev *, int ramp_delay);
};
```
### 2.3 regulator_config - 配置结构
```c
struct regulator_config {
    struct device *dev;          // 所属设备
    const struct regulator_init_data *init_data; // 初始化数据
    struct device_node *of_node; // 设备树节点
    void *driver_data;           // 驱动私有数据
    struct regmap *regmap;       // 寄存器映射
    bool ena_gpio_initialized;   // GPIO使能初始化标志
};
```

---

## 3. 驱动开发指南

### 3.1 基本驱动开发步骤

步骤1：定义调节器描述符
```c
static const struct regulator_desc my_regulator_desc = {
    .name = "my-regulator",
    .id = 0,
    .n_voltages = 64,
    .ops = &my_regulator_ops,
    .type = REGULATOR_VOLTAGE,
    .owner = THIS_MODULE,

    .min_uV = 600000,
    .max_uV = 1800000,
    .uV_step = 12500,

    .vsel_reg = MY_REG_VOUT,
    .vsel_mask = 0x3F,
    .vsel_shift = 0,

    .enable_reg = MY_REG_CTRL,
    .enable_mask = BIT(0),
    .enable_is_inverted = false,

    .ramp_delay = 12500, // 12.5 mV/µs
};
```

步骤2：实现操作函数
```c
static const struct regulator_ops my_regulator_ops = {
    .enable = my_regulator_enable,
    .disable = my_regulator_disable,
    .is_enabled = my_regulator_is_enabled,

    .set_voltage_sel = my_regulator_set_voltage_sel,
    .get_voltage_sel = my_regulator_get_voltage_sel,
    .list_voltage = my_regulator_list_voltage,

    .set_mode = my_regulator_set_mode,
    .get_mode = my_regulator_get_mode,

    .set_ramp_delay = my_regulator_set_ramp_delay,
    .set_voltage_time_sel = my_regulator_set_voltage_time_sel,
};

/* 设置电压选择器 */
static int my_regulator_set_voltage_sel(struct regulator_dev *rdev,
                                       unsigned selector)
{
    struct my_regulator_data *data = rdev_get_drvdata(rdev);

    /* 验证选择器范围 */
    if (selector >= rdev->desc->n_voltages)
        return -EINVAL;

    /* 写入寄存器 */
    return regmap_update_bits(data->regmap,
                             rdev->desc->vsel_reg,
                             rdev->desc->vsel_mask,
                             selector << rdev->desc->vsel_shift);
}

/* 获取电压列表 */
static int my_regulator_list_voltage(struct regulator_dev *rdev,
                                    unsigned selector)
{
    if (selector >= rdev->desc->n_voltages)
        return -EINVAL;

    return rdev->desc->min_uV + selector * rdev->desc->uV_step;
}
```

步骤3：注册调节器
```c
static int my_regulator_probe(struct platform_device *pdev)
{
    struct regulator_config config = {0};
    struct regulator_dev *rdev;
    struct my_regulator_data *data;

    /* 分配私有数据结构 */
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    /* 初始化硬件 */
    data->regmap = dev_get_regmap(pdev->dev.parent, NULL);
    if (!data->regmap)
        return -ENODEV;

    /* 准备配置 */
    config.dev = &pdev->dev;
    config.driver_data = data;
    config.regmap = data->regmap;
    config.of_node = pdev->dev.of_node;

    /* 注册调节器 */
    rdev = devm_regulator_register(&pdev->dev, &my_regulator_desc, &config);
    if (IS_ERR(rdev)) {
        dev_err(&pdev->dev, "Failed to register regulator\n");
        return PTR_ERR(rdev);
    }

    /* 保存私有数据 */
    platform_set_drvdata(pdev, data);

    return 0;
}
```

### 3.2 高级功能实现

#### 3.2.1 多范围电压调节
```c
/* 定义电压范围 */
static const struct linear_range my_voltage_ranges[] = {
    REGULATOR_LINEAR_RANGE( 600000,  0,  31, 12500),  // 0.6-0.9875V
    REGULATOR_LINEAR_RANGE(1000000, 32,  63, 25000),  // 1.0-1.7875V
    REGULATOR_LINEAR_RANGE(2000000, 64, 127, 50000),  // 2.0-5.175V
};

static const struct regulator_ops my_multi_range_ops = {
    /* 使用线性范围辅助函数 */
    .list_voltage = regulator_list_voltage_linear_range,
    .map_voltage = regulator_map_voltage_linear_range,
    .set_voltage_sel = regulator_set_voltage_sel_regmap,
    .get_voltage_sel = regulator_get_voltage_sel_regmap,
};

static const struct regulator_desc my_multi_range_desc = {
    .linear_ranges = my_voltage_ranges,
    .n_linear_ranges = ARRAY_SIZE(my_voltage_ranges),
    .n_voltages = 128,
    /* ... 其他字段 ... */
};
```

#### 3.2.2 压摆率控制
```c
/* 设置压摆率 */
static int my_regulator_set_ramp_delay(struct regulator_dev *rdev,
                                      int ramp_delay)
{
    struct my_regulator_data *data = rdev_get_drvdata(rdev);
    u8 reg_val;

    /* 计算寄存器值 */
    if (ramp_delay <= 15625)
        reg_val = 0x0;  // 15.625 µV/µs
    else if (ramp_delay <= 31250)
        reg_val = 0x1;  // 31.25 µV/µs
    else if (ramp_delay <= 62500)
        reg_val = 0x2;  // 62.5 µV/µs
    else
        reg_val = 0x3;  // 125 µV/µs

    return regmap_write(data->regmap, REG_SLEW_RATE, reg_val);
}

/* 计算电压切换时间 */
static int my_regulator_set_voltage_time_sel(struct regulator_dev *rdev,
                                            unsigned int old_selector,
                                            unsigned int new_selector)
{
    int old_uv, new_uv, diff_uv;

    old_uv = my_regulator_list_voltage(rdev, old_selector);
    new_uv = my_regulator_list_voltage(rdev, new_selector);
    diff_uv = abs(new_uv - old_uv);

    /* 根据压摆率计算时间 */
    return DIV_ROUND_UP(diff_uv, rdev->constraints->ramp_delay);
}
```

#### 3.2.3 工作模式控制
```c
/* 定义工作模式 */
#define MY_REGULATOR_MODE_NORMAL    0
#define MY_REGULATOR_MODE_FAST      1
#define MY_REGULATOR_MODE_IDLE      2
#define MY_REGULATOR_MODE_STANDBY   3

static int my_regulator_set_mode(struct regulator_dev *rdev,
                                unsigned int mode)
{
    struct my_regulator_data *data = rdev_get_drvdata(rdev);
    u8 reg_val;

    switch (mode) {
    case REGULATOR_MODE_FAST:
        reg_val = MODE_FAST;
        break;
    case REGULATOR_MODE_NORMAL:
        reg_val = MODE_NORMAL;
        break;
    case REGULATOR_MODE_IDLE:
        reg_val = MODE_IDLE;
        break;
    case REGULATOR_MODE_STANDBY:
        reg_val = MODE_STANDBY;
        break;
    default:
        return -EINVAL;
    }

    return regmap_update_bits(data->regmap, REG_MODE,
                             MODE_MASK, reg_val);
}

static unsigned int my_regulator_get_mode(struct regulator_dev *rdev)
{
    struct my_regulator_data *data = rdev_get_drvdata(rdev);
    unsigned int reg_val;
    int ret;

    ret = regmap_read(data->regmap, REG_MODE, &reg_val);
    if (ret < 0)
        return ret;

    reg_val &= MODE_MASK;

    switch (reg_val) {
    case MODE_FAST:    return REGULATOR_MODE_FAST;
    case MODE_NORMAL:  return REGULATOR_MODE_NORMAL;
    case MODE_IDLE:    return REGULATOR_MODE_IDLE;
    case MODE_STANDBY: return REGULATOR_MODE_STANDBY;
    default:           return -EINVAL;
    }
}
```

### 3.3 错误处理与保护
#### 3.3.1 过温保护
```c
static irqreturn_t my_regulator_thermal_irq(int irq, void *dev_id)
{
    struct my_regulator_data *data = dev_id;

    dev_emerg(data->dev, "Over temperature detected! Shutting down regulator\n");

    /* 紧急关闭调节器 */
    regulator_disable(data->rdev);

    /* 设置错误标志 */
    regulator_notifier_call_chain(data->rdev,
                                 REGULATOR_EVENT_OVER_TEMP,
                                 NULL);

    return IRQ_HANDLED;
}
```

#### 3.3.2 欠压/过压保护
```c
static int my_regulator_get_error_flags(struct regulator_dev *rdev,
                                       unsigned int *flags)
{
    struct my_regulator_data *data = rdev_get_drvdata(rdev);
    unsigned int status;
    int ret;

    ret = regmap_read(data->regmap, REG_STATUS, &status);
    if (ret < 0)
        return ret;

    *flags = 0;

    if (status & STATUS_UNDERVOLTAGE)
        *flags |= REGULATOR_ERROR_UNDER_VOLTAGE;

    if (status & STATUS_OVERVOLTAGE)
        *flags |= REGULATOR_ERROR_OVER_VOLTAGE;

    if (status & STATUS_OVERCURRENT)
        *flags |= REGULATOR_ERROR_OVER_CURRENT;

    if (status & STATUS_OVERTEMP)
        *flags |= REGULATOR_ERROR_OVER_TEMP;

    return 0;
}
```

---

## 4. 消费者API
### 4.1 基本使用模式
获取调节器引用
```c
/* 方式1：通过名称获取 */
struct regulator *regulator_get(struct device *dev, const char *id);

/* 方式2：通过设备树属性获取 */
struct regulator *devm_regulator_get(struct device *dev, const char *id);
struct regulator *devm_regulator_get_optional(struct device *dev, const char *id);

/* 方式3：获取排他访问权 */
struct regulator *regulator_get_exclusive(struct device *dev, const char *id);
```
完整使用示例
```c
static int my_device_probe(struct platform_device *pdev)
{
    struct regulator *vdd, *vdd_io;
    int ret;

    /* 获取核心电压 */
    vdd = devm_regulator_get(&pdev->dev, "vdd-core");
    if (IS_ERR(vdd)) {
        if (PTR_ERR(vdd) != -EPROBE_DEFER)
            dev_err(&pdev->dev, "Failed to get vdd-core regulator\n");
        return PTR_ERR(vdd);
    }

    /* 获取IO电压（可选） */
    vdd_io = devm_regulator_get_optional(&pdev->dev, "vdd-io");
    if (IS_ERR(vdd_io))
        vdd_io = NULL;  // 可选调节器可能不存在

    /* 设置电压范围 */
    ret = regulator_set_voltage(vdd, 1200000, 1200000);
    if (ret < 0) {
        dev_err(&pdev->dev, "Failed to set voltage: %d\n", ret);
        return ret;
    }

    /* 设置负载电流（可选，用于优化效率） */
    regulator_set_load(vdd, 100000);  // 100mA

    /* 启用调节器 */
    ret = regulator_enable(vdd);
    if (ret < 0) {
        dev_err(&pdev->dev, "Failed to enable regulator: %d\n", ret);
        return ret;
    }

    /* 保存引用供后续使用 */
    my_device->vdd_regulator = vdd;
    my_device->vdd_io_regulator = vdd_io;

    return 0;
}

static void my_device_shutdown(struct platform_device *pdev)
{
    struct my_device *dev = platform_get_drvdata(pdev);

    /* 禁用调节器 */
    if (dev->vdd_regulator)
        regulator_disable(dev->vdd_regulator);

    if (dev->vdd_io_regulator)
        regulator_disable(dev->vdd_io_regulator);
}
```

### 4.2 电压控制API

设置电压
```c
/* 设置具体电压值 */
int regulator_set_voltage(struct regulator *regulator,
                         int min_uV, int max_uV);

/* 设置电压容差 */
int regulator_set_voltage_tol(struct regulator *regulator,
                             int target_uV, int tol_uV);

/* 示例 */
regulator_set_voltage(vdd_core, 1200000, 1200000);  // 精确1.2V
regulator_set_voltage(vdd_core, 1100000, 1300000);  // 范围1.1-1.3V
```
获取电压信息
```c
/* 获取当前电压 */
int voltage = regulator_get_voltage(regulator);

/* 检查是否支持某电压 */
int supported = regulator_is_supported_voltage(regulator,
                                              min_uV, max_uV);

/* 示例 */
int current_voltage = regulator_get_voltage(vdd_core);
if (regulator_is_supported_voltage(vdd_core, 900000, 900000)) {
    // 支持0.9V
}
```
### 4.3 电流控制API
```c
/* 设置电流限制 */
int regulator_set_current_limit(struct regulator *regulator,
                               int min_uA, int max_uA);

/* 获取电流限制 */
int regulator_get_current_limit(struct regulator *regulator);

/* 设置负载（用于效率优化） */
int regulator_set_load(struct regulator *regulator, int load_uA);

/* 示例 */
regulator_set_current_limit(vdd_core, 0, 2000000);  // 最大2A
regulator_set_load(vdd_core, 500000);  // 预期负载500mA
```
### 4.4 模式控制API
```c
/* 设置工作模式 */
int regulator_set_mode(struct regulator *regulator, unsigned int mode);

/* 获取当前模式 */
unsigned int regulator_get_mode(struct regulator *regulator);

/* 可用模式常量 */
#define REGULATOR_MODE_FAST      0x1  // 快速响应模式
#define REGULATOR_MODE_NORMAL    0x2  // 普通模式
#define REGULATOR_MODE_IDLE      0x4  // 空闲模式
#define REGULATOR_MODE_STANDBY   0x8  // 待机模式

/* 示例 */
regulator_set_mode(vdd_core, REGULATOR_MODE_FAST);  // 高性能模式
regulator_set_mode(vdd_core, REGULATOR_MODE_IDLE);  // 低功耗模式
```
### 4.5 批量操作API
```c
/* 批量控制多个调节器 */
int regulator_bulk_get(struct device *dev, int num_consumers,
                       struct regulator_bulk_data *consumers);

int regulator_bulk_enable(int num_consumers,
                          struct regulator_bulk_data *consumers);

int regulator_bulk_disable(int num_consumers,
                           struct regulator_bulk_data *consumers);

void regulator_bulk_free(int num_consumers,
                         struct regulator_bulk_data *consumers);

/* 示例 */
struct regulator_bulk_data supplies[] = {
    { .supply = "vdd-core" },
    { .supply = "vdd-io" },
    { .supply = "vdd-ddr" },
};

ret = regulator_bulk_get(dev, ARRAY_SIZE(supplies), supplies);
if (ret)
    return ret;

ret = regulator_bulk_enable(ARRAY_SIZE(supplies), supplies);
if (ret) {
    regulator_bulk_free(ARRAY_SIZE(supplies), supplies);
    return ret;
}

/* 使用后清理 */
regulator_bulk_disable(ARRAY_SIZE(supplies), supplies);
regulator_bulk_free(ARRAY_SIZE(supplies), supplies);
```
### 4.6 通知机制

事件通知
```c
/* 注册通知回调 */
struct notifier_block regulator_nb = {
    .notifier_call = my_regulator_event_handler,
};

int regulator_register_notifier(struct regulator *regulator,
                               struct notifier_block *nb);

/* 事件类型 */
#define REGULATOR_EVENT_UNDER_VOLTAGE        0x01
#define REGULATOR_EVENT_OVER_VOLTAGE         0x02
#define REGULATOR_EVENT_OVER_CURRENT         0x04
#define REGULATOR_EVENT_OVER_TEMP            0x08
#define REGULATOR_EVENT_FORCE_DISABLE        0x10
#define REGULATOR_EVENT_VOLTAGE_CHANGE       0x20
#define REGULATOR_EVENT_DISABLE              0x40
#define REGULATOR_EVENT_PRE_DISABLE          0x80
#define REGULATOR_EVENT_ABORT_DISABLE        0x100

/* 示例 */
static int my_regulator_event_handler(struct notifier_block *nb,
                                     unsigned long event, void *data)
{
    switch (event) {
    case REGULATOR_EVENT_OVER_VOLTAGE:
        pr_emerg("Over voltage detected!\n");
        emergency_shutdown();
        break;
    case REGULATOR_EVENT_OVER_TEMP:
        pr_warn("Regulator over temperature\n");
        reduce_power_consumption();
        break;
    case REGULATOR_EVENT_VOLTAGE_CHANGE:
        pr_info("Regulator voltage changed\n");
        break;
    }
    return NOTIFY_OK;
}

/* 注册 */
regulator_register_notifier(vdd_core, &regulator_nb);
```

---

## 5. 设备树配置

### 5.1 基本设备树绑定

PMIC节点定义
```c
&i2c1 {
    status = "okay";

    pmic: ip6103@68 {
        compatible = "injoinic,ip6103";
        reg = <0x68>;
        interrupt-parent = <&gpio>;
        interrupts = <10 IRQ_TYPE_LEVEL_LOW>;

        regulators {
            /* 必需属性 */
            compatible = "injoinic,ip6103-regulator";

            /* 定义调节器 */
            ddc0_reg: ddc0 {
                regulator-name = "vdd-core";
                regulator-min-microvolt = <600000>;
                regulator-max-microvolt = <1375000>;
                regulator-boot-on;
                regulator-always-on;
                regulator-ramp-delay = <12500>; /* 12.5mV/µs */
                regulator-enable-ramp-delay = <100>;
                regulator-disable-ramp-delay = <2000>;
                regulator-state-mem {
                    regulator-on-in-suspend;
                    regulator-suspend-microvolt = <900000>;
                };
                regulator-state-disk {
                    regulator-off-in-suspend;
                };
            };

            ddc1_reg: ddc1 {
                regulator-name = "vdd-cpu";
                regulator-min-microvolt = <600000>;
                regulator-max-microvolt = <1375000>;
                regulator-ramp-delay = <12500>;
                regulator-coupled-with = <&ddc0_reg>;
                regulator-coupled-max-spread = <100000>;
            };
        };
    };
};
```
消费者设备绑定
```c
/* CPU使用PMIC的ddc0作为电源 */
cpu: cpu@0 {
    compatible = "arm,cortex-a53";
    reg = <0x0>;
    device_type = "cpu";

    /* 绑定到PMIC调节器 */
    cpu-supply = <&ddc0_reg>;

    /* 动态电压频率调节表 */
    operating-points-v2 = <&cpu_opp_table>;

    /* CPU频率表 */
    cpu_opp_table: opp-table {
        compatible = "operating-points-v2";

        opp-200000000 {
            opp-hz = /bits/ 64 <200000000>;
            opp-microvolt = <900000 900000 1375000>;
            opp-suspend-microvolt = <900000>;
            clock-latency-ns = <40000>;
        };

        opp-400000000 {
            opp-hz = /bits/ 64 <400000000>;
            opp-microvolt = <950000 950000 1375000>;
            clock-latency-ns = <40000>;
        };

        opp-800000000 {
            opp-hz = /bits/ 64 <800000000>;
            opp-microvolt = <1100000 1100000 1375000>;
            clock-latency-ns = <40000>;
        };

        opp-1000000000 {
            opp-hz = /bits/ 64 <1000000000>;
            opp-microvolt = <1200000 1200000 1375000>;
            clock-latency-ns = <40000>;
        };

        opp-1200000000 {
            opp-hz = /bits/ 64 <1200000000>;
            opp-microvolt = <1300000 1300000 1375000>;
            clock-latency-ns = <40000>;
        };
    };
};

/* GPU使用PMIC的ddc1 */
gpu: gpu@1c00000 {
    compatible = "arm,mali-450";
    reg = <0x1c00000 0x10000>;

    mali-supply = <&ddc1_reg>;

    gpu_opp_table: opp-table {
        compatible = "operating-points-v2";

        opp-100000000 {
            opp-hz = /bits/ 64 <100000000>;
            opp-microvolt = <900000>;
        };

        opp-400000000 {
            opp-hz = /bits/ 64 <400000000>;
            opp-microvolt = <1100000>;
        };
    };
};
```
### 5.2 设备树绑定属性详解
```c
通用属性
regulator-name = "vdd-core";                // 调节器名称
regulator-min-microvolt = <600000>;         // 最小电压 (0.6V)
regulator-max-microvolt = <1375000>;        // 最大电压 (1.375V)
regulator-microvolt-offset = <0>;           // 电压偏移
regulator-step-microvolt = <12500>;         // 电压步进 (12.5mV)

/* 时序控制 */
regulator-ramp-delay = <12500>;             // 压摆率 (12.5mV/µs)
regulator-enable-ramp-delay = <100>;        // 使能上升时间
regulator-disable-ramp-delay = <2000>;      // 禁用下降时间
regulator-settling-time-up-us = <100>;      // 电压上升稳定时间
regulator-settling-time-down-us = <1000>;   // 电压下降稳定时间

/* 电流限制 */
regulator-max-microamp = <2000000>;         // 最大电流 (2A)
regulator-min-microamp = <100000>;          // 最小电流 (100mA)
regulator-init-microamp = <500000>;         // 初始电流 (500mA)

/* 系统控制 */
regulator-boot-on;                          // 启动时自动使能
regulator-always-on;                        // 始终使能（不可禁用）
regulator-allow-bypass;                     // 允许旁路模式
regulator-allow-set-load;                   // 允许设置负载

睡眠状态配置
regulator-state-mem {
    regulator-on-in-suspend;                // 睡眠时保持使能
    regulator-off-in-suspend;               // 睡眠时禁用
    regulator-suspend-microvolt = <900000>; // 睡眠时电压 (0.9V)
    regulator-mode = <REGULATOR_MODE_IDLE>; // 睡眠时模式
};

regulator-state-disk {
    regulator-off-in-suspend;               // 磁盘休眠时状态
    regulator-suspend-microvolt = <800000>;
};

耦合调节器配置
regulator-coupled-with = <&ddc0_reg>;       // 耦合的调节器
regulator-coupled-max-spread = <100000>;    // 最大电压差 (100mV)
regulator-max-step-microvolt = <50000>;     // 单步最大变化 (50mV)
```
### 5.3 解析设备树的驱动代码

```c
static int ip6103_parse_dt(struct platform_device *pdev,
                          struct ip6103_data *data)
{
    struct device_node *np = pdev->dev.of_node;
    struct device_node *regulators_np;
    struct regulator_init_data *init_data;
    int i;

    /* 获取regulators子节点 */
    regulators_np = of_get_child_by_name(np, "regulators");
    if (!regulators_np) {
        dev_err(&pdev->dev, "No regulators node found\n");
        return -ENODEV;
    }

    /* 解析每个调节器 */
    for (i = 0; i < ARRAY_SIZE(ip6103_regulators); i++) {
        struct device_node *reg_np;
        const char *reg_name;

        reg_name = ip6103_regulators[i].desc.name;
        reg_np = of_get_child_by_name(regulators_np, reg_name);

        if (!reg_np) {
            dev_dbg(&pdev->dev, "No node for regulator %s\n", reg_name);
            continue;
        }

        /* 解析regulator初始化数据 */
        init_data = of_get_regulator_init_data(&pdev->dev, reg_np,
                                              &ip6103_regulators[i].desc);
        if (!init_data) {
            dev_err(&pdev->dev, "Failed to parse DT for %s\n", reg_name);
            of_node_put(reg_np);
            continue;
        }

        /* 保存解析结果 */
        data->init_data[i] = init_data;
        data->of_node[i] = reg_np;

        /* 解析特殊属性 */
        of_property_read_u32(reg_np, "regulator-ramp-delay",
                            &data->ramp_delay[i]);

        /* 解析睡眠状态 */
        if (of_get_property(reg_np, "regulator-on-in-suspend", NULL))
            data->suspend_flags[i] |= IP6103_SUSPEND_ENABLE;

        of_node_put(reg_np);
    }

    of_node_put(regulators_np);
    return 0;
}
```

---

## 6. 调试与监控

### 6.1 Sysfs调试接口

调节器状态查看
```bash
# 查看所有regulator
ls /sys/class/regulator/
# regulator.0 regulator.1 regulator.2 ...

# 查看特定regulator信息
cd /sys/class/regulator/regulator.0/

# 基本信息
cat name        # 调节器名称
cat type        # 类型 (voltage/current)
cat userspace   # 用户空间控制状态

# 电压信息
cat voltage     # 当前电压 (uV)
cat min_microvolts  # 最小电压
cat max_microvolts  # 最大电压

# 状态信息
cat state       # 使能状态 (enabled/disabled)
cat suspend_state  # 睡眠状态
cat suspend_mem_state  # 内存睡眠状态
cat suspend_disk_state # 磁盘睡眠状态

# 电流信息
cat current_now     # 当前电流
cat min_microamps   # 最小电流
cat max_microamps   # 最大电流

# 模式信息
cat mode        # 工作模式
cat supported_modes # 支持的模式

# 错误标志
cat error_flags # 错误标志位

动态控制调节器
# 设置电压 (需要root权限)
echo 1200000 > /sys/class/regulator/regulator.0/voltage

# 启用/禁用调节器
echo 1 > /sys/class/regulator/regulator.0/state
echo 0 > /sys/class/regulator/regulator.0/state

# 设置模式
echo "fast" > /sys/class/regulator/regulator.0/mode
# 或使用数值
echo 1 > /sys/class/regulator/regulator.0/mode

# 设置负载
echo 500000 > /sys/class/regulator/regulator.0/load
```
## 6.2 Debugfs调试接口

### 6.2.1 驱动中实现debugfs
```c
#include <linux/debugfs.h>

static int ip6103_debugfs_show(struct seq_file *s, void *v)
{
    struct ip6103_data *data = s->private;
    int i;

    seq_puts(s, "=== IP6103 Debug Information ===\n\n");

    for (i = 0; i < IP6103_PWR_NUM; i++) {
        struct regulator *reg = data->regulators[i];
        int voltage, current, enabled;

        if (!reg)
            continue;

        voltage = regulator_get_voltage(reg);
        current = regulator_get_current_limit(reg);
        enabled = regulator_is_enabled(reg);

        seq_printf(s, "Regulator %d (%s):\n", i, data->reg_names[i]);
        seq_printf(s, "  Status: %s\n", enabled ? "ENABLED" : "DISABLED");
        seq_printf(s, "  Voltage: %d.%03d V\n",
                   voltage / 1000000, (voltage % 1000000) / 1000);
        seq_printf(s, "  Current: %d mA\n", current / 1000);

        /* 读取硬件寄存器原始值 */
        if (data->regmap) {
            unsigned int reg_val;
            regmap_read(data->regmap, data->reg_base + i, &reg_val);
            seq_printf(s, "  Raw register: 0x%02x\n", reg_val);
        }

        seq_putc(s, '\n');
    }

    return 0;
}

static int ip6103_debugfs_open(struct inode *inode, struct file *file)
{
    return single_open(file, ip6103_debugfs_show, inode->i_private);
}

static const struct file_operations ip6103_debugfs_fops = {
    .owner = THIS_MODULE,
    .open = ip6103_debugfs_open,
    .read = seq_read,
    .llseek = seq_lseek,
    .release = single_release,
};

static void ip6103_init_debugfs(struct ip6103_data *data)
{
    struct dentry *root;

    root = debugfs_create_dir("ip6103", NULL);
    if (IS_ERR(root))
        return;

    debugfs_create_file("status", 0444, root, data, &ip6103_debugfs_fops);
    debugfs_create_x32("debug_flags", 0644, root, &data->debug_flags);

    data->debug_dir = root;
}

/* 在probe函数中调用 */
static int ip6103_probe(struct i2c_client *client)
{
    // ... 初始化代码 ...

    ip6103_init_debugfs(data);

    return 0;
}
```

### 6.2.2 用户空间使用debugfs
```bash
# 查看IP6103状态
cat /sys/kernel/debug/ip6103/status

# 设置调试标志
echo 1 > /sys/kernel/debug/ip6103/debug_flags

### 6.3 日志与跟踪

启用调试日志
# 启用regulator子系统调试
echo 1 > /sys/module/regulator/parameters/debug_core
echo 1 > /sys/module/regulator/parameters/debug_consumers

# 动态调试（需要内核配置CONFIG_DYNAMIC_DEBUG）
echo "file regulator/* +p" > /sys/kernel/debug/dynamic_debug/control

# 查看regulator相关内核日志
dmesg | grep regulator

Tracepoints跟踪
# 启用regulator跟踪点
echo 1 > /sys/kernel/debug/tracing/events/regulator/enable

# 查看跟踪结果
cat /sys/kernel/debug/tracing/trace | head -50

# 过滤特定调节器
echo 'name=="vdd-core"' > /sys/kernel/debug/tracing/events/regulator/filter

# 跟踪特定函数
echo 'regulator_set_voltage' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
```
### 6.4 性能监控

电源测量脚本
```bash
#!/bin/bash
# power_monitor.sh - 监控调节器性能

INTERVAL=1  # 采样间隔（秒）
DURATION=60 # 持续时间（秒）
OUTPUT_FILE="/tmp/power_monitor_$(date +%s).csv"

echo "Starting power monitor for $DURATION seconds..."
echo "Time,Regulator,Voltage(mV),Current(mA),Power(mW),Mode" > $OUTPUT_FILE

end_time=$((SECONDS + DURATION))

while [ $SECONDS -lt $end_time ]; do
    timestamp=$(date +%s.%N)

    for reg in /sys/class/regulator/regulator.*; do
        name=$(cat $reg/name 2>/dev/null || echo "unknown")
        voltage=$(cat $reg/voltage 2>/dev/null || echo "0")
        current=$(cat $reg/current_now 2>/dev/null || echo "0")
        mode=$(cat $reg/mode 2>/dev/null || echo "unknown")

        # 计算功率（mW）
        power=$(( (voltage * current) / 1000000 ))

        # 输出到CSV
        echo "$timestamp,$name,$((voltage/1000)),$((current/1000)),$power,$mode" >> $OUTPUT_FILE
    done

    sleep $INTERVAL
done

echo "Data saved to $OUTPUT_FILE"
echo "Generating plot..."

# 使用gnuplot生成图表（如果可用）
if command -v gnuplot &> /dev/null; then
    gnuplot << EOF
    set terminal png size 1024,768
    set output "/tmp/power_plot.png"
    set datafile separator ","
    set xlabel "Time (s)"
    set ylabel "Power (mW)"
    set title "Regulator Power Consumption"
    plot "$OUTPUT_FILE" using 0:5 with lines title "Power"
EOF
    echo "Plot saved to /tmp/power_plot.png"
fi
```
实时监控工具
```c
/* realtime_monitor.c - 实时监控调节器状态 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/perf_event.h>
#include <asm/unistd.h>

static long perf_event_open(struct perf_event_attr *hw_event, pid_t pid,
                           int cpu, int group_fd, unsigned long flags)
{
    return syscall(__NR_perf_event_open, hw_event, pid, cpu, group_fd, flags);
}

int main(void)
{
    struct perf_event_attr pe;
    long long count;
    int fd;

    memset(&pe, 0, sizeof(pe));
    pe.type = PERF_TYPE_HARDWARE;
    pe.size = sizeof(pe);
    pe.config = PERF_COUNT_HW_CPU_CYCLES;
    pe.disabled = 1;
    pe.exclude_kernel = 1;
    pe.exclude_hv = 1;

    fd = perf_event_open(&pe, 0, -1, -1, 0);
    if (fd == -1) {
        perror("perf_event_open");
        return 1;
    }

    ioctl(fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(fd, PERF_EVENT_IOC_ENABLE, 0);

    printf("Monitoring regulator events...\n");

    while (1) {
        sleep(1);
        read(fd, &count, sizeof(count));

        /* 这里可以添加调节器状态读取逻辑 */
        printf("CPU cycles: %lld\n", count);

        /* 读取sysfs中的调节器信息 */
        system("grep -h . /sys/class/regulator/regulator.*/voltage 2>/dev/null | head -5");
    }

    ioctl(fd, PERF_EVENT_IOC_DISABLE, 0);
    close(fd);

    return 0;
}
```

---

## 7. 性能优化

### 7.1 DVFS（动态电压频率调节）优化

CPU频率与电压对应表
```c
static struct dvfs_table {
    unsigned long freq_hz;
    unsigned int voltage_uv;
    unsigned int efficiency; /* 能效评分 */
} cpu_dvfs_table[] = {
    {  200000000,  900000, 95 },  /* 低频率，高能效 */
    {  400000000,  950000, 92 },
    {  600000000, 1000000, 88 },
    {  800000000, 1050000, 85 },
    { 1000000000, 1100000, 80 },
    { 1200000000, 1150000, 75 },
    { 1400000000, 1200000, 70 },
    { 1600000000, 1250000, 65 },  /* 高频率，低能效 */
};
```
智能DVFS算法
```c
static int smart_dvfs_controller(struct device *dev,
                                unsigned long target_freq)
{
    struct regulator *vdd_core = dev_get_drvdata(dev);
    int i, target_voltage = -1;

    /* 查找目标频率对应的电压 */
    for (i = 0; i < ARRAY_SIZE(cpu_dvfs_table); i++) {
        if (cpu_dvfs_table[i].freq_hz == target_freq) {
            target_voltage = cpu_dvfs_table[i].voltage_uv;
            break;
        }
    }

    if (target_voltage < 0)
        return -EINVAL;

    /* 获取当前电压 */
    int current_voltage = regulator_get_voltage(vdd_core);

    /* 智能电压调整策略 */
    if (target_voltage > current_voltage) {
        /* 升压：先升压后升频 */
        regulator_set_voltage(vdd_core, target_voltage, target_voltage);
        usleep(100); /* 等待电压稳定 */
        set_cpu_frequency(target_freq);
    } else {
        /* 降压：先降频后降压 */
        set_cpu_frequency(target_freq);
        usleep(100); /* 等待频率稳定 */
        regulator_set_voltage(vdd_core, target_voltage, target_voltage);
    }

    /* 记录能效数据 */
    record_efficiency_data(target_freq, target_voltage,
                          cpu_dvfs_table[i].efficiency);

    return 0;
}
```
### 7.2 功耗优化策略

负载预测与预调节
```c
/* 基于负载预测的电压调节 */
static int predictive_voltage_scaling(struct regulator *reg,
                                     struct workload_predictor *predictor)
{
    int predicted_load, target_voltage;
    unsigned long next_check;

    while (!kthread_should_stop()) {
        /* 预测未来负载 */
        predicted_load = predictor->predict_next_load();

        /* 根据负载选择电压 */
        if (predicted_load < 20)   /* 轻负载 */
            target_voltage = 900000;
        else if (predicted_load < 50) /* 中等负载 */
            target_voltage = 1100000;
        else                       /* 重负载 */
            target_voltage = 1300000;

        /* 应用电压 */
        regulator_set_voltage(reg, target_voltage, target_voltage);

        /* 动态调整预测间隔 */
        next_check = msecs_to_jiffies(
            predicted_load < 30 ? 100 : 50);

        schedule_timeout_interruptible(next_check);
    }

    return 0;
}
```
多调节器协同优化
```c
/* 耦合调节器优化 */

static int optimize_coupled_regulators(struct regulator **regs, int num_regs)
{
    int i, j;
    int voltages[10];
    int total_power, best_power = INT_MAX;
    int best_voltages[10];

    /* 获取当前电压 */
    for (i = 0; i < num_regs; i++)
        voltages[i] = regulator_get_voltage(regs[i]);

    /* 遍历电压组合寻找最优解 */
    for (i = 0; i < 8; i++) {  /* 假设每个调节器有8个可选电压 */
        total_power = 0;

        for (j = 0; j < num_regs; j++) {
            int test_voltage = get_voltage_option(j, i);
            int current = estimate_current(j, test_voltage);

            /* 计算功耗 */
            total_power += (test_voltage * current) / 1000000;

            /* 检查耦合约束 */
            if (j > 0) {
                int diff = abs(test_voltage - voltages[j-1]);
                if (diff > MAX_COUPLED_SPREAD)
                    break;
            }
        }

        /* 更新最优解 */
        if (total_power < best_power) {
            best_power = total_power;
            memcpy(best_voltages, voltages, sizeof(voltages));
        }
    }

    /* 应用最优电压 */
    for (i = 0; i < num_regs; i++)
        regulator_set_voltage(regs[i], best_voltages[i], best_voltages[i]);

    return 0;
}
```
### 7.3 温度管理

温度自适应电压调整
```c
static int thermal_adaptive_voltage(struct regulator *reg,
                                   struct thermal_zone_device *tz)
{
    int temp, voltage, max_voltage;

    /* 获取当前温度 */
    thermal_zone_get_temp(tz, &temp);

    /* 根据温度调整最大允许电压 */
    if (temp < 60000) {          /* < 60°C */
        max_voltage = 1300000;   /* 全电压 */
    } else if (temp < 80000) {   /* 60-80°C */
        max_voltage = 1200000;   /* 降额10% */
    } else if (temp < 95000) {   /* 80-95°C */
        max_voltage = 1100000;   /* 降额20% */
    } else {                     /* > 95°C */
        max_voltage = 1000000;   /* 降额30% */
    }

    /* 获取当前电压并调整 */
    voltage = regulator_get_voltage(reg);
    if (voltage > max_voltage) {
        regulator_set_voltage(reg, max_voltage, max_voltage);

        /* 通知系统电压已降额 */
        pr_warn("Thermal throttling: voltage reduced to %d mV\n",
                max_voltage / 1000);
    }

    return 0;
}
```

---

## 8. 常见问题

### 8.1 常见错误及解决方法

**问题1：调节器注册失败**
```bash
错误信息
[   10.123456] ip6103-regulator: Failed to register regulator ddc0: -22
```
可能原因及解决：

设备树配置错误
```bash
# 检查设备树
dtc -I fs /sys/firmware/devicetree/base | grep -A5 ip6103
```
约束条件冲突
```c
// 检查约束
if (min_uV < desc->min_uV || max_uV > desc->max_uV) {
    dev_err(dev, "Voltage out of range\n");
    return -EINVAL;
}
```
ops函数未实现
```c
// 确保所有必需的ops都已实现
static const struct regulator_ops ip6103_ops = {
    .enable = ip6103_enable,
    .disable = ip6103_disable,
    .is_enabled = ip6103_is_enabled,
    .set_voltage_sel = ip6103_set_voltage_sel,
    .get_voltage_sel = ip6103_get_voltage_sel,
    .list_voltage = ip6103_list_voltage,
    // 必须至少实现这些基本操作
};
```
**问题2：电压设置无效**
```bash
# 设置电压但实际不变
echo 1200000 > /sys/class/regulator/regulator.0/voltage
cat voltage  # 输出可能还是旧值
```
调试步骤：

检查硬件通信
```bash
# 使用i2c-tools验证
i2cdetect -y 1  # 扫描I2C总线
i2cget -y 1 0x68 0x21  # 读取电压寄存器
```

检查掩码和移位
```c
// 确保正确使用vout_mask和vout_shift
selector = (reg_val & ri->vout_mask) >> ri->vout_shift;
// 写入时：
reg_val &= ~ri->vout_mask;
reg_val |= (selector << ri->vout_shift) & ri->vout_mask;
```
添加调试日志
```c
dev_dbg(ri->dev, "Setting voltage: reg=0x%02x, mask=0x%02x, shift=%d\n",
        ri->vout_reg, ri->vout_mask, ri->vout_shift);
dev_dbg(ri->dev, "Before: 0x%02x, After: 0x%02x\n", old_val, reg_val);
```
**问题3：调节器无法使能/禁用**
```bash
# 尝试使能/禁用
echo 1 > /sys/class/regulator/regulator.0/enable
echo 0 > /sys/class/regulator/regulator.0/enable
```
可能原因：
```c
1. always-on约束
// 设备树中设置了always-on
regulator-always-on;

2. 其他消费者正在使用
# 检查使用者数量
cat /sys/class/regulator/regulator.0/use_count

3. 硬件故障或保护
// 检查状态寄存器
regmap_read(data->regmap, REG_STATUS, &status);
if (status & STATUS_FAULT) {
    dev_err(dev, "Hardware fault detected: 0x%02x\n", status);
}
```
### 8.2 性能调优建议

优化电压切换延迟
```c
/* 减少电压稳定等待时间 */
static int optimize_voltage_switch(struct regulator_dev *rdev)
{
    /* 根据电压差动态调整等待时间 */
    int voltage_diff = abs(new_voltage - current_voltage);
    int delay_us;

    if (voltage_diff < 100000)      /* < 0.1V变化 */
        delay_us = 10;
    else if (voltage_diff < 500000) /* 0.1-0.5V变化 */
        delay_us = 50;
    else                            /* > 0.5V变化 */
        delay_us = 100;

    /* 应用优化后的延迟 */
    return delay_us;
}

减少sysfs访问开销
/* 批量读取sysfs属性 */
static int read_regulator_attributes_batch(struct regulator *reg,
                                          struct reg_attributes *attrs)
{
    char path[256];
    FILE *fp;

    /* 一次性读取多个属性 */
    snprintf(path, sizeof(path), "/sys/class/regulator/%s/",
             reg->name);

    /* 实现批量读取逻辑 */
    return 0;
}
```
### 8.3 兼容性问题

处理不同版本的regulator API
```c
/* 版本兼容性处理 */
#include <linux/version.h>

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 14, 0)
/* 旧版本API */
static int ip6103_set_voltage(struct regulator_dev *rdev,
                             int min_uV, int max_uV,
                             unsigned *selector)
{
    /* 旧实现 */
}
#else
/* 新版本API */
static int ip6103_set_voltage_sel(struct regulator_dev *rdev,
                                 unsigned selector)
{
    /* 新实现 */
}
#endif

/* 条件编译ops结构 */
static struct regulator_ops ip6103_ops = {
#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 14, 0)
    .set_voltage = ip6103_set_voltage,
#else
    .set_voltage_sel = ip6103_set_voltage_sel,
#endif
    /* 其他ops... */
};
```
## 总结

Regulator 框架是 Linux 电源管理的基石，它将分散的电源控制抽象为统一的软件接口，让开发者从复杂的硬件差异中解脱出来。它不仅仅是电压调节器，更是硬件资源的管理者，为系统提供了稳定、可靠的供电保障。