# 1 Platform 驱动模型

## 1.1 Platform 概述

### 1.1.1 什么是 Platform

Platform 是 Linux 内核中用于描述片上、不可自动枚举硬件的一套设备驱动模型。它通常用于 SoC 内部集成的外设，例如 GPIO、I2C 控制器、串口控制器、看门狗和 PWM 控制器等。

与 USB、PCI 等具有硬件枚举能力的总线不同，Platform 设备通常不能自行报告设备型号、寄存器地址和中断号。因此，内核需要通过设备树、ACPI 表或板级代码向 Platform 总线注册设备信息，再由对应的 Platform 驱动完成匹配和初始化。

Platform 模型的核心对象：

- `platform_device`：描述一个具体硬件设备及其资源，例如寄存器地址、中断号和 DMA 资源。
- `platform_driver`：描述设备的驱动程序，提供 `probe`、`remove` 等回调函数。
- `platform_bus_type`：Platform 设备与 Platform 驱动所挂载的总线对象，负责二者的匹配。

典型工作流程如下：

1. 内核注册 `platform_device`，提供设备名称和硬件资源。
2. 驱动模块注册 `platform_driver`，提供可支持的设备标识和回调函数。
3. Platform 总线调用匹配函数查找对应的设备与驱动。
4. 匹配成功后，内核调用驱动的 `probe` 函数完成资源申请和硬件初始化。

### 1.1.2 Platform 的驱动分离

Platform 模型体现了设备与驱动分离的思想：设备信息与驱动代码分别维护，二者通过总线匹配机制建立关联。

设备侧主要描述“硬件是什么、资源在哪里”，例如设备树中的 `compatible` 属性、寄存器地址和中断号；驱动侧主要描述“如何操作该硬件”，例如寄存器配置、字符设备注册和中断处理。

这种分离带来的主要好处：

- 同一份驱动可以支持多个使用相同 IP 核的硬件平台。
- 板级硬件变化时，通常只需修改设备树或设备资源描述。
- 驱动加载顺序不再要求严格固定，设备和驱动任意一方先注册均可等待匹配。

本节仅介绍其基本思想。设备树匹配、资源获取和 `probe` 函数将在后续章节展开。

## 1.2 Linux 设备驱动模型

### 1.2.1 `bus_type` 结构体

Linux 系统内核使用 `struct bus_type` 表示总线，该结构体定义在 `include/linux/device.h` 文件中。总线不仅表示一种硬件连接方式，也负责管理挂载在该总线上的设备和驱动，并决定设备与驱动如何匹配。

Linux 设备驱动模型的核心对象包括总线、设备和驱动：

- `struct bus_type`：描述一类总线，并定义设备和驱动的匹配规则。
- `struct device`：内核中设备对象的通用表示。
- `struct device_driver`：内核中驱动对象的通用表示。

当设备和驱动注册到同一总线时，内核驱动核心会调用该总线的匹配函数。匹配成功后，设备将与驱动绑定，并进入驱动的 `probe` 初始化流程。

在不同内核版本中，`struct bus_type` 的字段可能略有增减；与驱动开发最密切相关的字段如下：

```c
struct bus_type {
    const char *name;                  /* 总线名字 */
    const char *dev_name;
    struct device *dev_root;
    struct device_attribute *dev_attrs;

    const struct attribute_group **bus_groups;    /* 总线属性 */
    const struct attribute_group **dev_groups;     /* 设备属性 */
    const struct attribute_group **drv_groups;     /* 驱动属性 */

    // 设备与驱动匹配回调
    int (*match)(struct device *dev, struct device_driver *drv);
    // 热插拔环境变量回调
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);

    // 设备探测、移除、关机
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    // 设备上下线
    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);

    // 电源管理：休眠、唤醒
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
    const struct dev_pm_ops *pm;

    // IOMMU 操作集
    const struct iommu_ops *iommu_ops;

    // 子系统私有数据、锁调试key
    struct subsys_private *p;
    struct lock_class_key lock_key;
};
```

主要成员说明：

- `name`：总线名称，会出现在 sysfs 的 `/sys/bus/` 目录下。
- `dev_groups`：为该总线上的设备创建 sysfs 属性组。
- `match`：设备与驱动的匹配函数。返回非零值表示匹配成功。
- `uevent`：生成用户空间热插拔事件环境变量的回调函数。
- `pm`：电源管理操作集合，包含休眠、唤醒等相关回调。

当一个设备或驱动注册到总线后，驱动核心会遍历同一总线中的候选对象，并调用该总线的 `match` 函数。只有匹配成功的设备和驱动才会绑定，随后执行驱动的 `probe` 回调。

### 1.2.2 Platform 总线实例

Platform 总线是 `struct bus_type` 的一个具体实例，定义在内核源码的 `drivers/base/platform.c` 文件中。Platform 总线定义如下：

```c
struct bus_type platform_bus_type = {
    .name = "platform",
    .dev_groups = platform_dev_groups,
    .match = platform_match,
    .uevent = platform_uevent,
    .pm = &platform_dev_pm_ops,
};
```

各成员含义：

- `.name = "platform"`：指定该总线名称为 `platform`，对应 sysfs 中的 `/sys/bus/platform/`。
- `.dev_groups = platform_dev_groups`：为 Platform 设备提供默认 sysfs 属性组。
- `.match = platform_match`：指定 Platform 设备与 Platform 驱动的匹配函数。
- `.uevent = platform_uevent`：在生成 uevent 时补充 Platform 设备相关信息。
- `.pm = &platform_dev_pm_ops`：指定 Platform 设备使用的电源管理操作。

其中，`.match` 是设备和驱动能够自动绑定的关键，它指定的函数就是 `platform_match`。该函数将通用的 `struct device` 和 `struct device_driver` 转换为 Platform 类型对象，再按照 Platform 模型定义的规则进行匹配。

### 1.2.3 `platform_match` 函数

### 1.2.3.1 函数作用

`platform_match` 是 Platform 总线的设备驱动匹配函数。当 Platform 设备或 Platform 驱动注册到内核时，驱动核心会调用此函数判断两者是否对应。

函数原型如下：

```c
static int platform_match(struct device *dev, struct device_driver *drv);
```

参数说明：

- `dev`：待匹配的通用设备对象，实际类型通常为 `struct platform_device` 中的 `dev` 成员。
- `drv`：待匹配的通用驱动对象，实际类型通常为 `struct platform_driver` 中的 `driver` 成员。

返回值说明：

- 非零值：设备与驱动匹配成功，驱动核心随后尝试调用该驱动的 `probe` 函数。
- `0`：设备与驱动不匹配，驱动核心继续寻找其他候选驱动。

### 1.2.3.2 匹配顺序

现代 Linux 内核中的 `platform_match` 会按优先级尝试多种匹配方式。常见顺序如下：

1. 驱动覆盖匹配：若设备设置了 `driver_override`，仅与指定驱动名比较。
2. 设备树匹配：调用 `of_driver_match_device()`，比较设备树中的 `compatible` 与驱动的 `of_match_table`。
3. ACPI 匹配：在启用 ACPI 的平台上，调用 ACPI 相关匹配逻辑。
4. `id_table` 匹配：比较 `platform_device_id` 表中的设备名称。
5. 名称匹配：最后比较 `platform_device` 与 `platform_driver` 的名称。

简化后的逻辑可理解为：

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    if (dev->driver_override)
        return !strcmp(dev->driver_override, drv->name);

    if (of_driver_match_device(dev, drv))
        return 1;

    if (acpi_driver_match_device(dev, drv))
        return 1;

    if (platform_match_id(pdrv->id_table, pdev))
        return 1;

    return !strcmp(pdev->name, drv->name);
}
```

说明：不同内核版本的实际实现会有条件编译和细节差异，上述代码用于说明匹配优先级，不应直接复制为驱动代码。

### 1.2.3.3 匹配后的行为

当 `platform_match` 返回非零值后，驱动核心会建立设备与驱动的绑定关系，并通过 Platform 总线的 `probe` 流程调用 `platform_driver` 提供的回调函数。

典型的驱动定义形式如下：

```c
static int demo_probe(struct platform_device *pdev)
{
    return 0;
}

static struct platform_driver demo_driver = {
    .probe = demo_probe,
    .driver = {
        .name = "demo-platform",
    },
};
```

此示例中，若 Platform 设备名称同样为 `demo-platform`，且没有更高优先级的匹配规则，则名称匹配成功后会调用 `demo_probe()`。

### 1.2.3.4 注意事项

- 设备树平台优先使用 `compatible` 与 `of_match_table` 匹配，不应仅依赖 `.driver.name`。
- `.driver.name` 主要用于驱动标识和兜底名称匹配；它不是设备树 `compatible` 的替代品。
- `platform_match` 只负责判断能否绑定，不负责申请寄存器、中断等资源。
- 硬件资源的获取和初始化应放在驱动的 `probe` 函数中；释放操作应放在 `remove` 或 devres 管理的资源回收流程中。

至此，Platform 总线通过 `platform_match` 完成设备和驱动的绑定判断，后续驱动初始化则由匹配成功后的 `probe` 回调负责。

## 1.3 Platform 设备

### 1.3.1 `platform_device` 概述

Platform 设备使用 `struct platform_device` 描述硬件设备信息。对于不能自动枚举的片上外设，设备对象需要提供驱动初始化所需的硬件资源，例如寄存器物理地址、中断号、DMA 通道或总线地址。

`struct platform_device` 定义在 `include/linux/platform_device.h` 文件中，常用成员如下：

```c
struct platform_device {
    const char *name;
    int id;
    bool id_auto; // 如果 ls /sys/bus/platform/devices后缀有.auto，代表使用自动设置ID
    struct device dev;
    u32 num_resources;
    struct resource *resource;
    const struct platform_device_id *id_entry;
    char *driver_override;
    /* 其余成员省略 */
};
```

主要成员说明：

- `name`：设备名称，用于与 `platform_driver` 的名称或 `id_table` 进行匹配。
- `id`：设备编号。同一类设备有多个实例时可使用不同编号；设置为 `-1` 表示不附加编号。
- `dev`：通用设备对象，Platform 设备通过该成员接入 Linux 设备模型。
- `num_resources`：资源数组中有效资源的数量。
- `resource`：指向 `struct resource` 资源数组，用于描述设备使用的硬件资源。
- `id_entry`：匹配成功后指向对应的 `platform_device_id` 表项，通常由内核匹配流程设置。
- `driver_override`：强制指定匹配驱动名称的字符串，设置后会优先参与 `platform_match` 匹配。

设备信息只描述硬件“有什么资源”，不负责操作硬件寄存器。驱动在 `probe` 函数中通过 `platform_get_resource()`、`platform_get_irq()` 等接口获取这些资源，再进行映射和初始化。

### 1.3.2 `struct resource` 结构体

`struct resource` 用于描述一段硬件资源，其定义位于 `include/linux/ioport.h` 文件中。Platform 设备可以通过资源数组统一向驱动提供寄存器地址、中断号等信息。

常用结构形式如下：

```c
struct resource {
    resource_size_t start;
    resource_size_t end;
    const char *name;
    unsigned long flags;
    struct resource *parent;
    struct resource *sibling;
    struct resource *child;
};
```

主要成员说明：

- `start`：资源起始地址或起始编号。
- `end`：资源结束地址或结束编号，资源范围包含 `end`。
- `name`：资源名称，用于调试或按名称获取资源。
- `flags`：资源类型和属性标志。

对于寄存器资源，`start` 和 `end` 是物理地址。若寄存器区域长度为 `length`，则结束地址应写为：

```c
end = start + length - 1;
```

减去 `1` 的原因是 `start` 和 `end` 都包含在资源范围内。若误写为 `start + length`，资源范围会比实际寄存器区域多出 1 字节。

### 1.3.3 `flags` 资源类型

`flags` 用于标识资源类型，驱动据此确定资源的用途。Platform 设备中最常用的资源类型如下：

```c
#define IORESOURCE_BITS       0x000000ff  /* Bus-specific bits */

#define IORESOURCE_TYPE_BITS  0x00001f00  /* Resource type */
#define IORESOURCE_IO         0x00000100  /* PCI/ISA I/O ports */
#define IORESOURCE_MEM        0x00000200
#define IORESOURCE_REG        0x00000300  /* Register offsets */
#define IORESOURCE_IRQ        0x00000400
#define IORESOURCE_DMA        0x00000800
#define IORESOURCE_BUS        0x00001000
```

注意事项：

- `IORESOURCE_MEM` 描述的是物理地址，驱动不能直接解引用，通常需要使用 `devm_ioremap_resource()` 映射为内核虚拟地址。
- `IORESOURCE_IRQ` 的 `start` 和 `end` 通常填写同一个中断号。
- 资源数组的顺序就是资源索引；当同类资源有多个时，驱动通过 `index` 区分它们。

### 1.3.4 Platform 设备注册框架

下面给出一个 Platform 设备模块的完整框架。示例定义一段寄存器资源和一个中断资源，并通过 `.dev.release` 提供设备对象的释放回调。

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/platform_device.h>

/* 寄存器地址和长度定义 */
#define MYDEVICE_REGISTER_BASE  0xfdd60000
#define MYDEVICE_REGISTER_SIZE  0x5
#define MYDEVICE_IRQ            13

static struct resource my_device_resources[] = {
    [0] = {
        .start = MYDEVICE_REGISTER_BASE,
        .end = MYDEVICE_REGISTER_BASE + MYDEVICE_REGISTER_SIZE - 1,
        .flags = IORESOURCE_MEM,
    },
    [1] = {
        .start = MYDEVICE_IRQ,
        .end = MYDEVICE_IRQ,
        .flags = IORESOURCE_IRQ,
    },
};

static void mydevice_release(struct device *dev)
{
}

static struct platform_device platform_device_test = {
    .name = "mydevice",
    .id = -1,
    .resource = my_device_resources,
    .num_resources = ARRAY_SIZE(my_device_resources),
    .dev = {
        .release = mydevice_release,
    },
};

static int __init platform_device_test_init(void)
{
    return platform_device_register(&platform_device_test);
}

static void __exit platform_device_test_exit(void)
{
    platform_device_unregister(&platform_device_test);
}

module_init(platform_device_test_init);
module_exit(platform_device_test_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("example");
MODULE_DESCRIPTION("Platform device example");
```

`release` 是 `struct device` 的释放回调。对于通过静态对象定义的 `platform_device_test`，回调无需释放动态内存，因此函数体可以为空；但 `.release` 不能省略，否则设备模型在注销设备时会给出警告。

`platform_device_register()` 成功后，设备会加入 Platform 总线。若总线上存在能够匹配名称 `mydevice` 的 Platform 驱动，内核会调用该驱动的 `probe` 函数；驱动可在其中获取本节定义的内存和中断资源。

## 1.4 Platform 驱动

### 1.4.1 `platform_driver` 结构体

Platform 驱动使用 `struct platform_driver` 表示，该结构体定义在 `include/linux/platform_device.h` 文件中。它在通用驱动对象 `struct device_driver` 的基础上，增加了 Platform 设备的 `probe`、`remove` 等回调函数。

常用结构形式如下：

```c
struct platform_driver {
    int (*probe)(struct platform_device *pdev);
    void (*remove)(struct platform_device *pdev);
    void (*shutdown)(struct platform_device *pdev);
    int (*suspend)(struct platform_device *pdev, pm_message_t state);
    int (*resume)(struct platform_device *pdev);
    struct device_driver driver;
    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
};
```

主要成员说明：

- `probe`：设备与驱动匹配成功后调用。驱动在此申请资源、映射寄存器、注册中断并初始化硬件。
- `remove`：设备解绑或驱动卸载时调用，用于释放未由 devres 管理的资源。
- `shutdown`：系统关机或重启前调用，用于让设备进入安全状态。
- `driver`：通用驱动对象，包含驱动名称、设备树匹配表和电源管理信息。
- `id_table`：传统 Platform 设备 ID 匹配表，用于非设备树平台或兼容旧驱动。

不同内核版本中，`remove` 的函数原型可能是 `int (*remove)(struct platform_device *pdev)` 或 `void (*remove)(struct platform_device *pdev)`。编写驱动时应以正在使用的内核头文件为准。

### 1.4.2 `device_driver` 结构体

`struct device_driver` 是 Linux 设备模型中的通用驱动结构体，定义在 `include/linux/device.h` 文件中。`struct platform_driver` 的 `driver` 成员就是该结构体。

与 Platform 驱动最相关的成员如下：

```c
struct device_driver {
    const char *name;
    struct bus_type *bus;
    const struct of_device_id *of_match_table;
    const struct acpi_device_id *acpi_match_table;
    const struct dev_pm_ops *pm;
    /* 其余成员省略 */
};
```

主要成员说明：

- `name`：驱动名称，用于名称匹配，也是驱动在 sysfs 中显示的名称。
- `bus`：驱动所在总线。`platform_driver_register()` 会将其设置为 Platform 总线。
- `of_match_table`：设备树匹配表，Platform 总线的 `platform_match()` 会使用它匹配设备树节点。
- `acpi_match_table`：ACPI 平台使用的匹配表。
- `pm`：驱动的电源管理操作集合。

### 1.4.3 `struct of_device_id` 结构体

设备树匹配表由 `struct of_device_id` 数组组成。该结构体定义在 `include/linux/mod_devicetable.h` 文件中，常用定义如下：

```c
struct of_device_id {
    char name[32];
    char type[32];
    char compatible[128];
    const void *data;
};
```

主要成员说明：

- `name`：设备节点名称匹配字段，实际驱动中通常不使用。
- `type`：设备类型匹配字段，实际驱动中通常不使用。
- `compatible`：设备树 `compatible` 属性的匹配字符串，是最常用的成员。
- `data`：驱动私有数据指针。多个兼容设备共用驱动时，可通过它保存型号差异数据。

典型设备树匹配表示例如下：

```c
static const struct of_device_id mydevice_of_match[] = {
    { .compatible = "example,mydevice" },
    { }
};
MODULE_DEVICE_TABLE(of, mydevice_of_match);
```

最后的空表项表示匹配表结束。`MODULE_DEVICE_TABLE()` 用于将匹配信息导出到模块别名，便于用户空间根据设备树节点自动加载对应模块。

### 1.4.4 驱动注册与匹配流程

Platform 驱动注册应调用 `platform_driver_register()`，而不是 `platform_register()`。前者会进一步调用通用驱动注册接口，将 `platform_driver` 注册到 Platform 总线。

简化调用关系如下：

```c
platform_driver_register(pdrv)
    -> __platform_driver_register(pdrv, owner, mod_name)
        -> driver_register(&pdrv->driver)
            -> driver_attach(&pdrv->driver)
                -> platform_match(dev, drv)
                -> platform_drv_probe(pdrv, pdev)
                -> pdrv->probe(pdev)
```

实际匹配由 Platform 总线的 `platform_match()` 完成。对于设备树设备，`platform_match()` 会调用设备树匹配逻辑，并将设备节点的 `compatible` 属性与 `pdrv->driver.of_match_table` 中的表项比较；`of_match_table` 本身不是回调函数。

匹配成功后的行为：

1. 驱动核心将该设备与该驱动绑定，并设置 `dev->driver` 指向对应的 `struct device_driver`。
2. 内核调用 Platform 驱动的 `probe` 回调，并将匹配到的 `struct platform_device *pdev` 传入。
3. 驱动从 `pdev` 获取寄存器、中断等资源，完成硬件初始化。

设备信息不会整体复制或存放到 `platform_driver` 中。一个 `platform_driver` 可以绑定多个同类型的 `platform_device`；每次调用 `probe` 都会获得当前设备自己的 `pdev`。若驱动需要保存每个设备的私有状态，应在 `probe` 中分配私有数据，并使用 `platform_set_drvdata(pdev, data)` 保存，再通过 `platform_get_drvdata(pdev)` 获取。

### 1.4.5 Platform 驱动框架

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/mod_devicetable.h>

struct resource *myresources;

int mydriver_probe(struct platform_device *dev)
{
    printk("This is mydriver_probe\n");

    printk("IRQ is %lld\n", dev->resource[1].start);

    myresources = platform_get_resource(dev, IORESOURCE_IRQ, 0);

    printk("IRQ is %lld\n", dev->resource[1].start);

    return 0;
}

static void mydriver_remove(struct platform_device *pdev)
{
    pr_info("This is mydriver_remove\n");
}

static const struct platform_device_id mydriver_id_table[] = {
    .name = "mydevice" ,

};
MODULE_DEVICE_TABLE(platform, mydriver_id_table);

static struct platform_driver platform_driver_test = {
    .probe = mydriver_probe,
    .remove = mydriver_remove,
    .driver = {
        .name = "mydevice",
    },
    .id_table = mydriver_id_table,
};

module_platform_driver(platform_driver_test);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("example");
MODULE_DESCRIPTION("Platform driver example");
```

示例中的 `mydriver_id_table` 与 1.3.4 中 `platform_device_test.name = "mydevice"` 对应。Platform 总线在设备树和 ACPI 匹配均未命中时，会使用该表进行 ID 匹配。

图片中可以通过 `pdev->resource[1].start` 直接读取 IRQ，但该写法依赖资源数组中 IRQ 固定在下标 `1`，不利于复用。示例改用 `platform_get_resource(pdev, IORESOURCE_IRQ, 0)`，按资源类型取得第 0 个中断资源；`%pa` 与 `&irq_res->start` 用于正确输出 `resource_size_t` 类型的资源值。

`module_platform_driver()` 宏会自动生成模块入口和出口，分别调用 `platform_driver_register()` 与 `platform_driver_unregister()`。若需要在模块入口完成额外初始化，也可以手动编写 `module_init()` 和 `module_exit()` 函数。
