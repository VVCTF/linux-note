# 1 设备模型概述

设备模型是 Linux 内核中用于统一描述设备、总线、驱动以及类设备关系的一套基础框架。它解决的核心问题不是“如何收发数据”，而是“如何组织设备对象、建立层级、导出到 sysfs、触发热插拔事件、管理生命周期”。

核心作用：

- 给内核对象提供统一的层级组织方式
- 让对象可以映射到 sysfs，形成可观测的目录树
- 提供引用计数和释放回调，避免对象提前释放
- 为驱动匹配、设备枚举、用户态感知变化提供基础设施

## 1.1 设备模型中的核心对象

### 1.1.1 kobject 是什么

kobject 可以理解为“内核对象的最小管理单元”，它本身不代表具体业务设备，但负责承载对象的通用管理能力，例如：

- 父子层级关系（parent）
- 所属集合关系（kset）
- 对象类型行为（ktype）
- 引用计数与释放生命周期（kref + release）
- 在 sysfs 中对应目录节点

常见理解方式：

- 具体子系统对象（如 device、driver）通常“内嵌”一个 kobject
- 子系统通过封装 kobject，获得统一注册、引用和目录导出能力

### 1.1.2 kset 是什么

kset 是一组 kobject 的集合管理器。它的职责是把一批相关对象组织在一起，并提供集合级别的管理和事件策略。

核心作用：

- 维护成员 kobject 的链表
- 作为集合目录导出到 sysfs
- 通过 uevent_ops 统一影响集合中对象的热插拔事件行为

实践中通常会将“语义上同类”的 kobject 放进同一个 kset。很多场景下这些对象也会共享同一个 kobj_type，但二者并不是语法强制绑定关系，而是常见设计约定。

## 1.2 kobject 结构体成员解析

### 1.2.1 关键结构体

示例（不同内核版本成员可能略有差异，以下为常见主干字段）：

```c
struct kobject {
    const char              *name;
    struct list_head        entry;
    struct kobject          *parent;
    struct kset             *kset;
    const struct kobj_type  *ktype;
    struct kernfs_node      *sd;
    struct kref             kref;

    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

### 1.2.2 成员职责

参数说明：

- name：对象名。通常对应 sysfs 目录名的一部分。
- entry：链表节点。用于把当前 kobject 挂到其所属 kset 的 list 上。
- parent：父对象指针。用于建立对象树。
- kset：所属集合。决定该对象归属哪个集合管理域。
- ktype：对象类型描述，定义该类对象的默认行为（如 release、sysfs_ops、default_groups）。
- sd：指向 sysfs/kernfs 节点。
- kref：引用计数核心字段。
- state_* 与 uevent_suppress：记录对象状态和事件发送控制。

行为说明：

- kobject 的“层级关系”主要由 parent 决定
- kobject 的“集合关系”主要由 kset 决定
- kobject 的“行为模板”主要由 ktype 决定

## 1.3 kset 结构体成员解析

### 1.3.1 关键结构体

```c
struct kset {
    struct list_head list;
    spinlock_t list_lock;
    struct kobject kobj;
    const struct kset_uevent_ops *uevent_ops;
};
```

### 1.3.2 成员职责

参数说明：

- list：kset 成员链表头。链表中元素由每个成员 kobject.entry 串接。
- list_lock：保护 list 并发访问的自旋锁。
- kobj：kset 自身也是一个 kobject，因此它也能出现在设备模型层级与 sysfs 中。
- uevent_ops：集合级事件策略，例如过滤是否上报事件、补充环境变量、命名策略。

关键点：

- kset 不是“替代 kobject”，而是“管理一组 kobject”
- kset 自己带有 kobj，意味着“集合本身也是对象”

## 1.4 kobject、parent、kset、ktype 的连接关系

### 1.4.1 parent 如何连接 kobject

parent 用于形成树状层级：

- 子对象保存 parent 指针
- 通过 parent 链可向上追溯到更高层对象
- 该层级会体现在 sysfs 目录层次中

这条关系回答的是“我是谁的子对象”。

### 1.4.2 kset 如何把 kobject 组织起来

当一个 kobject 指定了 kset 后，会通过自身的 entry 节点接入该 kset.list。

连接方式可概括为：

1. kset.list 是双向链表头
2. 每个成员 kobject 提供 entry（list_head）
3. entry 挂入 list 后，kset 就拥有该成员的集合视图

这条关系回答的是“我属于哪个集合”。

### 1.4.3 kobj_type 在集合中的意义

kobj_type 用于定义对象行为，不直接承担“链表连接”动作。实际连接动作由 kset.list 与 kobject.entry 完成。

但在工程实践中，常见做法是：

- 将语义一致的一组对象放进同一个 kset
- 这些对象往往共享同一个 kobj_type，便于统一 release/sysfs 行为

因此可以把二者关系理解为：

- kset：负责“组织”
- kobj_type：负责“行为约束”

## 1.5 小结

本章建立以下三个基础认知：

- kobject 是设备模型的基本管理对象，重点是层级、引用计数和行为挂接
- kset 是对象集合管理器，通过 list 与 entry 串接成员
- parent、kset、ktype 分别解决层级关系、集合关系、行为定义三个不同问题

# 2 kobject 与 kset 的注册

本章从 API 和结构体出发，说明如何把 kobject/kset 注册到设备模型，并形成可见的 sysfs 节点。

## 2.1 注册相关核心结构体

### 2.1.1 `struct kobj_type`

`kobj_type` 用于描述某类 kobject 的默认行为，尤其是释放逻辑。

常见定义示例：

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);
    const struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
    const struct attribute_group **default_groups;
    const struct kobj_ns_type_operations *(*child_ns_type)(
        const struct kobject *kobj);
    const void *(*namespace)(const struct kobject *kobj);
    void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

关键点：

- `release` 是必须重点关注的回调
- 当引用计数降到 0 时，内核会走到 `release`
- 如果对象是 `kzalloc` 出来的，通常在 `release` 中 `kfree`

### 2.1.2 `struct kset`

`kset` 的结构体在上一章已介绍。本章关注其注册相关角色：

- `kset_create_and_add`：创建并注册一个集合
- `kset_unregister`：注销集合（内部会对其 `kobj` 执行 put 路径）

## 2.2 kobject 注册 API

### 2.2.1 `kobject_create_and_add`

```c
struct kobject *kobject_create_and_add(const char *name, struct kobject *parent);
```

参数说明：

- `name`：对象名，对应 sysfs 目录名
- `parent`：父 kobject；`NULL` 表示挂到默认层级

返回值说明：

- 成功：返回非 `NULL` 的 `kobject *`
- 失败：返回 `NULL`

使用特点：

- 简单快捷，适合快速创建测试对象
- 内部已经完成创建与 add 流程

### 2.2.2 `kobject_init_and_add`

```c
int kobject_init_and_add(struct kobject *kobj,
                         const struct kobj_type *ktype,
                         struct kobject *parent,
                         const char *fmt, ...);
```

参数说明：

- `kobj`：调用者分配的对象（常见 `kzalloc`）
- `ktype`：对象类型，至少要保证 `release` 合法
- `parent`：父对象
- `fmt`：对象名格式字符串

返回值说明：

- `0`：成功
- `< 0`：失败，返回负错误码

使用特点：

- 更灵活，常用于“业务结构体内嵌 kobject”或需要自定义生命周期控制
- 失败后要走错误回滚路径，避免泄漏

### 2.2.3 `kobject_put`

```c
void kobject_put(struct kobject *kobj);
```

行为说明：

- 对象引用计数减 1
- 计数归零时触发释放链路，最终调用 `ktype->release`

注意事项：

- `kobject_put` 不等于“立刻 free”，而是“减少引用，可能延迟释放”

## 2.3 kset 注册 API

### 2.3.1 `kset_create_and_add`

```c
struct kset *kset_create_and_add(const char *name,
                                 const struct kset_uevent_ops *uevent_ops,
                                 struct kobject *parent_kobj);
```

参数说明：

- `name`：kset 名称
- `uevent_ops`：可选的事件操作集
- `parent_kobj`：父对象

返回值说明：

- 成功：返回非 `NULL` 的 `kset *`
- 失败：返回 `NULL`

### 2.3.2 `kset_unregister`

```c
void kset_unregister(struct kset *k);
```

行为说明：

- 注销集合对象
- 一般在确保成员对象已正确 put 后再注销集合

## 2.4 kobject 注册示例

### 2.4.1 示例代码

示例（基于你给出的代码，补充了失败路径与资源回收）：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/init.h>
#include <linux/slab.h>

static void my_kobj_release(struct kobject *kobj)
{
    kfree(kobj);
}

static struct kobj_type mytype = {
    .release = my_kobj_release,
};

static struct kobject *my_kobj1;
static struct kobject *my_kobj2;
static struct kobject *my_kobj3;

static int __init mykobject_init(void)
{
    int ret;

    my_kobj1 = kobject_create_and_add("my_kobject1", NULL);
    if (!my_kobj1)
        return -ENOMEM;

    my_kobj2 = kobject_create_and_add("my_kobject2", NULL);
    if (!my_kobj2) {
        kobject_put(my_kobj1);
        return -ENOMEM;
    }

    my_kobj3 = kzalloc(sizeof(*my_kobj3), GFP_KERNEL);
    if (!my_kobj3) {
        kobject_put(my_kobj2);
        kobject_put(my_kobj1);
        return -ENOMEM;
    }

    ret = kobject_init_and_add(my_kobj3, &mytype, NULL, "%s", "my_kobject3");
    if (ret) {
        kobject_put(my_kobj3);
        kobject_put(my_kobj2);
        kobject_put(my_kobj1);
        return ret;
    }

    return 0;
}

static void __exit mykobject_exit(void)
{
    kobject_put(my_kobj3);
    kobject_put(my_kobj2);
    kobject_put(my_kobj1);
}

module_init(mykobject_init);
module_exit(mykobject_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_VERSION("1.0");
```

### 2.4.2 调用时序

推荐时序：

1. 先创建/初始化对象
2. 再 add 到模型
3. 出错时按逆序 put
4. 退出时按逆序 put

核心原则：

- 谁先成功创建，谁后释放
- 初始化路径和失败路径必须对齐

## 2.5 kset 注册示例

### 2.5.1 示例代码

示例（基于你给出的代码，补充 `release` 与失败回滚）：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/init.h>
#include <linux/slab.h>

static void my_kobj_release(struct kobject *kobj)
{
    kfree(kobj);
}

static struct kobj_type mytype = {
    .release = my_kobj_release,
};

static struct kobject *my_kobj1;
static struct kobject *my_kobj2;
static struct kobject *my_kobj3;
static struct kset *mykset;

static int __init mykset_demo_init(void)
{
    int ret;

    mykset = kset_create_and_add("my_kset", NULL, NULL);
    if (!mykset)
        return -ENOMEM;

    my_kobj1 = kzalloc(sizeof(*my_kobj1), GFP_KERNEL);
    if (!my_kobj1) {
        ret = -ENOMEM;
        goto err_kset;
    }
    my_kobj1->kset = mykset;
    ret = kobject_init_and_add(my_kobj1, &mytype, NULL, "%s", "my_kobject1");
    if (ret)
        goto err_put1;

    my_kobj2 = kzalloc(sizeof(*my_kobj2), GFP_KERNEL);
    if (!my_kobj2) {
        ret = -ENOMEM;
        goto err_put1;
    }
    my_kobj2->kset = mykset;
    ret = kobject_init_and_add(my_kobj2, &mytype, NULL, "%s", "my_kobject2");
    if (ret)
        goto err_put2;

    my_kobj3 = kzalloc(sizeof(*my_kobj3), GFP_KERNEL);
    if (!my_kobj3) {
        ret = -ENOMEM;
        goto err_put2;
    }
    my_kobj3->kset = mykset;
    ret = kobject_init_and_add(my_kobj3, &mytype, NULL, "%s", "my_kobject3");
    if (ret)
        goto err_put3;

    return 0;

err_put3:
    kobject_put(my_kobj3);
err_put2:
    kobject_put(my_kobj2);
err_put1:
    kobject_put(my_kobj1);
err_kset:
    kset_unregister(mykset);
    return ret;
}

static void __exit mykset_demo_exit(void)
{
    kobject_put(my_kobj3);
    kobject_put(my_kobj2);
    kobject_put(my_kobj1);
    kset_unregister(mykset);
}

module_init(mykset_demo_init);
module_exit(mykset_demo_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_VERSION("1.0");
```

### 2.5.2 关于 `kset->list` 与 `kobject->entry`

当 `my_kobjX->kset = mykset` 且 `kobject_init_and_add` 成功后：

- 该 kobject 会成为该 kset 的成员
- 其 `entry` 节点会链接到 `mykset->list`
- 从集合角度可遍历成员，从成员角度可知道所属集合

这也是“集合关系”在代码中的直接体现。

## 2.6 常见错误与注意事项

### 2.6.1 release 未定义或不匹配

常见问题：

- 使用 `kobject_init_and_add` 但没有正确的 `release`
- `release` 与分配方式不匹配（例如不是 `kzalloc` 却 `kfree`）

后果：

- 内存泄漏或非法释放

### 2.6.2 出错路径未按逆序回滚

常见问题：

- 初始化到一半失败，没有把前面成功对象 put
- kset 已创建但失败路径忘记 `kset_unregister`

推荐回滚顺序：

1. 先 put 最后一个成功 add 的 kobject
2. 再 put 更早的 kobject
3. 最后注销 kset

### 2.6.3 对象名冲突

若同一层级下对象名重复，`kobject_add` 类路径会失败并返回错误码。排查时可先检查：

- parent 是否一致
- 名称是否重复
- 旧对象是否已完整释放

## 2.7 小结

本章建立了注册层面的关键认识：

- `kobject_create_and_add` 适合快速创建，`kobject_init_and_add` 适合可控生命周期
- `kset_create_and_add` 创建集合，成员通过 `kobject->kset` 加入集合语义
- 退出与失败路径都必须遵守“逆序释放”

# 3 为什么引入设备模型

Linux 内核需要同时管理大量不同来源、不同类型的硬件与软件对象：USB 设备、I2C 设备、platform 设备、各类驱动、总线，以及面向用户功能划分的类别。若每个子系统各自维护对象关系、生命周期和用户态接口，就会造成管理方式分散、驱动匹配路径不统一、用户态难以观察设备状态。

设备模型的目的，是用统一的对象关系描述“设备连接到哪条总线、由哪个驱动驱动、属于哪个功能类别”，并将这些关系导出到 sysfs，供用户态查询和管理。

## 3.1 设备模型的四个核心概念

设备模型主要围绕设备、驱动、总线和类四类对象建立关系。

### 3.1.1 设备：`struct device`

设备（device）表示内核识别和管理的一个设备实例。它可以对应真实硬件，例如 I2C 控制器上的传感器；也可以对应逻辑设备，例如虚拟设备。

常见主干成员如下。不同内核版本的字段名称和排列可能略有差异：

```c
struct device {
    struct kobject kobj;
    struct device *parent;
    struct bus_type *bus;
    struct device_driver *driver;
    struct class *class;
    dev_t devt;
    void (*release)(struct device *dev);
    void *driver_data;
};
```

成员说明：

- `kobj`：内嵌的 kobject，使 device 可接入设备模型和 sysfs。
- `parent`：父设备，用于表示物理或逻辑从属关系。
- `bus`：设备所在总线。
- `driver`：当前已匹配并绑定的驱动。
- `class`：设备所属类别，用于按功能而非按物理拓扑组织设备。
- `devt`：设备号，字符设备和块设备通常会使用它。
- `release`：device 最终释放时的回调。

### 3.1.2 驱动：`struct device_driver`

驱动（driver）表示一类设备的驱动程序。一个驱动通常可与多个同类型设备实例绑定。

```c
struct device_driver {
    const char *name;
    struct bus_type *bus;
    struct module *owner;
    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);
    struct driver_private *p;
};
```

成员说明：

- `name`：驱动名，也是匹配规则可能使用的名称。
- `bus`：驱动注册到的总线。
- `probe`：总线匹配成功后，用于初始化具体设备实例。
- `remove`：设备移除或驱动解绑时的清理回调。
- `p`：驱动模型内部私有数据；其中维护驱动对应的 kobject 等管理信息。

注意事项：

- `struct device_driver` 的公开定义没有直接名为 `kobj` 的成员。
- 驱动对象通过内部 `struct driver_private` 接入 kobject 和 sysfs，因此它仍属于设备模型的一部分。

### 3.1.3 总线：`struct bus_type`

总线（bus）描述设备与驱动相遇、匹配和绑定的场所。这里的“总线”既可以是物理总线，如 USB、PCI、I2C，也可以是逻辑总线，如 platform 总线。

```c
struct bus_type {
    const char *name;
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);
    struct subsys_private *p;
};
```

成员说明：

- `name`：总线名称。
- `match`：判断一个 device 与一个 device_driver 是否匹配。
- `probe`：匹配成功后调用驱动的 probe 路径。
- `remove`：解绑或移除时调用驱动的 remove 路径。
- `p`：总线内部私有数据，其中管理总线、设备目录和驱动目录对应的 kobject/kset。

总线解决的核心问题是：设备注册和驱动注册的先后顺序可以不同，但二者最终都能在同一条总线上重新匹配。

### 3.1.4 类：`struct class`

类（class）按设备提供的功能来组织设备，不强调它们位于哪条物理总线。例如输入设备可统一归入 input 类，TTY 设备可统一归入 tty 类。

```c
struct class {
    const char *name;
    struct module *owner;
    int (*dev_uevent)(const struct device *dev, struct kobj_uevent_env *env);
    char *(*devnode)(const struct device *dev, umode_t *mode);
    struct subsys_private *p;
};
```

成员说明：

- `name`：类名称，对应 `/sys/class` 下的目录名。
- `dev_uevent`：为该类设备的 uevent 补充环境变量。
- `devnode`：为设备节点路径和权限提供策略。
- `p`：类内部私有数据，保存类相关的 kobject/kset 等信息。

设备、驱动、总线、类的关系可以概括为：

```text
device --注册到--> bus --match--> device_driver
   |
   +--按功能归类--> class
```

## 3.2 高级对象如何接入设备模型

### 3.2.1 kobject 是设备模型的基石

kobject 为内核对象提供统一的名称、父子关系、引用计数、释放回调和 sysfs 对应节点。可以把设备、总线、驱动和类看作建立在 kobject 机制之上的高级对象：它们不一定都在公开结构体中直接嵌入 `struct kobject`，但最终都会通过内嵌成员或内部私有对象使用 kobject。

一个已成功添加到 sysfs 的 kobject，通常对应 `/sys` 下的一个目录。目录中的普通属性文件则由该 kobject 对应的 attribute、attribute_group 等机制创建。

### 3.2.2 `cdev` 与 `platform_device` 的嵌入关系

字符设备核心对象 `cdev` 直接内嵌 kobject：

```c
struct cdev {
    struct kobject kobj;
    struct module *owner;
    const struct file_operations *ops;
    struct list_head list;
    dev_t dev;
    unsigned int count;
};
```

因此，`cdev` 可以利用其 `cdev.kobj` 参与对象生命周期管理。需要区分的是，字符设备在 sysfs 中是否形成用户可见节点，还取决于设备号注册和 device/class 等周边流程，不能简单理解为只要 `cdev_add()` 就必然拥有独立目录。

platform 设备的关系是“platform_device 内嵌 device，device 再内嵌 kobject”：

```c
struct platform_device {
    const char *name;
    int id;
    bool id_auto;
    struct device dev;
    u64 platform_dma_mask;
    struct pdev_archdata archdata;
};
```

层级关系为：

```text
platform_device
    └── device dev
            └── kobject kobj
```

这类“包含关系”使高级对象不必重复实现对象命名、引用计数和 sysfs 注册逻辑，而是复用 kobject 提供的通用能力。

## 3.3 sysfs 文件系统

### 3.3.1 sysfs 是什么

sysfs 是内核导出的伪文件系统，通常挂载在 `/sys`。它不对应磁盘上的普通文件系统，而是将内核中的对象、属性与关系实时呈现为目录、属性文件和符号链接。

常见挂载命令：

```sh
mount -t sysfs sysfs /sys
```

sysfs 的价值：

- 用户态可查看设备、总线、驱动和类别的组织关系。
- 内核可通过属性文件向用户态暴露状态与可配置参数。
- udev 等用户态程序可结合 uevent 和 sysfs 创建设备节点、加载策略。

### 3.3.2 kobject 与 sysfs 的关联

kobject 中的 `sd` 成员指向对应的 kernfs 节点：

```c
struct kobject {
    /* 其他成员省略 */
    struct kernfs_node *sd;
};
```

当 kobject 被添加到设备模型时，内核会根据其名称、parent 和所属 kset 确定 sysfs 目录的父节点和目录名，然后为其创建 kernfs 目录节点。创建成功后，`kobj->sd` 保存该节点，表示 kobject 与 sysfs 目录已经关联。

## 3.4 从 `kobject_create_and_add` 到 sysfs 目录

### 3.4.1 API 调用入口

前一章使用过以下 API：

```c
struct kobject *kobject_create_and_add(const char *name,
                                       struct kobject *parent);
```

它把“分配 kobject”和“注册到设备模型”组合在一起。为了理解 kobject 如何进入 `/sys`，可关注其典型调用链。

### 3.4.2 典型调用链

不同内核版本的辅助函数名称可能略有调整，但核心路径如下：

```text
kobject_create_and_add()
    ├── kobject_create()
    │   └── kobject_init()
    └── kobject_add()
        └── kobject_add_varg()
            └── kobject_add_internal()
                └── create_dir()
                    └── sysfs_create_dir_ns()
                        └── kernfs_create_dir_ns()
```

各层职责：

- `kobject_create()`：分配 kobject 并完成基础初始化。
- `kobject_init()`：设置 ktype、引用计数和初始化状态。
- `kobject_add()`：进入对象添加路径。
- `kobject_add_internal()`：解析 parent/kset 关系、设置对象名、将对象接入集合，并创建 sysfs 节点。
- `create_dir()`：为 kobject 创建 sysfs 目录，同时创建默认属性组。
- `sysfs_create_dir_ns()`：根据 kobject 和命名空间信息创建 sysfs 目录。
- `kernfs_create_dir_ns()`：在 kernfs 树中真正创建目录节点。

关键结论：

- `kobject_create_and_add()` 不是仅创建一块内存；它最终会请求 kernfs 创建目录。
- 成功返回后，该 kobject 具备 sysfs 目录映射；失败时应按前章所述调用 `kobject_put()` 回收。

### 3.4.3 `sysfs_root_kn` 的创建

sysfs 的目录树需要根 kernfs 节点。该根节点由 sysfs 初始化代码创建，典型源码位置为 `fs/sysfs/mount.c`。

初始化主干可概括为：

```c
static struct kernfs_node *sysfs_root_kn;

static int __init sysfs_init(void)
{
    sysfs_root_kn = kernfs_create_root(NULL,
                                       KERNFS_ROOT_CREATE_DEACTIVATED,
                                       NULL);
    if (IS_ERR(sysfs_root_kn))
        return PTR_ERR(sysfs_root_kn);

    return 0;
}
fs_initcall(sysfs_init);
```

说明：

- `sysfs_init()` 在内核文件系统初始化阶段执行。
- `kernfs_create_root()` 创建 sysfs 使用的 kernfs 根节点，并将其保存到 `sysfs_root_kn`。
- 用户态将 sysfs 挂载到 `/sys` 后，看到的目录树以该 kernfs 根节点为基础。
- 后续 `sysfs_create_dir_ns()` 创建的各级目录，都会连接到这棵根树中。

## 3.5 `/sys` 目录与设备模型层次

### 3.5.1 `/sys/bus`：按总线组织

`/sys/bus` 用于展示总线视角下的设备和驱动。以 platform 总线为例：

```text
/sys/bus/platform/
    ├── devices/
    └── drivers/
```

其中：

- `devices`：该总线上的设备视图。
- `drivers`：注册到该总线的驱动视图。

总线的 `match` 回调决定设备和驱动如何配对，匹配成功后形成绑定关系。

### 3.5.2 `/sys/class`：按功能类别组织

`/sys/class` 按设备功能分类，便于用户态从功能角度查找设备：

```text
/sys/class/
    ├── tty/
    ├── block/
    └── net/
```

例如，一个串口设备的物理连接路径可能很深，但用户态通常可通过 `/sys/class/tty` 快速定位到对应的 TTY 设备。

### 3.5.3 `/sys/devices`：按设备层级组织

`/sys/devices` 是设备树状层级的主要呈现位置。它反映 device 的 `parent` 关系，通常用于描述设备的物理拓扑或核心逻辑拓扑。

示意：

```text
/sys/devices/
    └── platform/
        └── soc/
            └── serial@xxxx/
```

目录的级联关系来自 kobject/device 的 parent 链，而不是来自 class 的分类关系。

### 3.5.4 目录、级联与软链接

同一个设备需要被不同视角访问：

- 从物理层级看，应位于 `/sys/devices`。
- 从所属总线看，应能在 `/sys/bus/<bus>/devices` 找到。
- 从功能分类看，应能在 `/sys/class/<class>` 找到。

为避免为同一个设备创建多份彼此独立的对象，sysfs 通常保留 `/sys/devices` 中的真实设备目录，再在 `/sys/bus/<bus>/devices` 和 `/sys/class/<class>` 中建立指向该目录的符号链接。

可概括为：

```text
/sys/devices/.../deviceX          实际设备目录
/sys/bus/<bus>/devices/deviceX    指向实际设备目录的软链接
/sys/class/<class>/deviceX        指向实际设备目录的软链接
```

这正是设备模型同时表达“层级关系、总线关系、类别关系”的方式：

- `/sys/devices`：体现设备所属的父设备层级。
- `/sys/bus`：体现设备和驱动所属的总线及匹配关系。
- `/sys/class`：体现设备向用户空间提供的功能类别。

## 3.6 小结

引入设备模型后，内核使用统一的对象和关系管理设备生态：

- `device` 表示设备实例，`device_driver` 表示驱动，`bus_type` 负责匹配，`class` 提供功能视图。
- kobject 是统一管理基础；高级对象通过直接内嵌 kobject，或经内部私有对象使用 kobject，接入设备模型。
- kobject 注册路径最终到达 `sysfs_create_dir_ns()` 和 `kernfs_create_dir_ns()`，从而在 sysfs 中创建目录。
- sysfs 根节点 `sysfs_root_kn` 由 `fs/sysfs/mount.c` 中的 `sysfs_init()` 创建。
- `/sys/devices` 呈现层级关系，`/sys/bus` 呈现总线关系，`/sys/class` 呈现功能类别；同一设备的多视角主要通过软链接关联。

# 4 引用计数器 kref

内核驱动中的对象经常会被多个执行路径同时使用。例如，应用程序已经通过 `open()` 打开设备文件时，设备可能正在被移除；驱动的退出路径也可能正在执行。如果某个路径直接释放设备私有数据，而另一个路径仍在 `read()`、`write()` 或 `release()` 中访问该数据，就会出现悬空指针、内核访问越界，甚至导致应用程序异常或内核崩溃。

`kref` 是 Linux 内核提供的轻量级引用计数器，用于解决“对象何时才能真正释放”的问题。

## 4.1 kref 的基本原理

### 4.1.1 什么是引用计数

引用计数记录“当前有多少有效使用者仍持有该对象”。只要引用计数大于 0，对象就不能释放；只有最后一个使用者归还引用后，内核才调用释放回调回收对象。

可将对象生命周期概括为：

```text
创建对象
    └── 初始引用计数为 1
            ├── 新使用者取得对象：计数加 1
            ├── 使用者结束访问：计数减 1
            └── 计数减至 0：执行 release，真正释放对象
```

`kref` 的核心原则：

- 每一次成功取得对象引用，最终必须有一次对应的 `kref_put()`。
- 不能在仍有人使用对象时直接 `kfree()`。
- 最后一次 `kref_put()` 才能触发实际释放。

### 4.1.2 `struct kref`

`kref` 的定义位于 `include/linux/kref.h`：

```c
struct kref {
    refcount_t refcount;
};
```

成员说明：

- `refcount`：底层引用计数，使用 `refcount_t` 类型。

`refcount_t` 相比直接使用 `atomic_t` 更适合引用计数场景。内核可利用它检查部分非法操作，例如引用计数溢出、对已经为 0 的对象继续加引用等。

### 4.1.3 kref 与 kobject 的关系

`kobject` 内部也使用 `kref` 管理自身生命周期：

```c
struct kobject {
    /* 其他成员省略 */
    struct kref kref;
};
```

但 `kref` 可以独立使用，不要求对象必须是 kobject。例如，字符设备私有结构体、DMA 缓冲区管理结构体、异步请求上下文，都可以内嵌一个 `struct kref`。

## 4.2 已打开设备与驱动卸载

### 4.2.1 为什么不能立即释放

考虑一个字符设备驱动：

1. 应用程序调用 `open("/dev/demo", ...)`。
2. 驱动的 `open()` 将设备私有对象保存到 `file->private_data`。
3. 应用程序尚未 `close()`，此时仍可能继续调用 `read()`、`write()` 或 `ioctl()`。
4. 若设备移除路径或驱动退出路径此时直接 `kfree()` 私有对象，`file->private_data` 就变成悬空指针。
5. 应用程序下一次系统调用进入驱动后访问悬空指针，可能触发内核错误；应用程序也会因系统调用失败或设备失效而异常。

因此，正确目标不是“卸载时立刻释放所有内存”，而是：

- 阻止新的用户继续取得对象。
- 已打开文件仍可安全走完已有的使用和关闭路径。
- 所有引用归还后，再释放私有对象。

### 4.2.2 kref 与模块引用计数的分工

这里必须区分两种不同的引用计数：

- `kref`：保护设备私有对象、上下文或缓冲区等数据内存。
- 模块引用计数：保护驱动模块的代码和静态数据不被卸载。

对于文件操作，通常应在 `struct file_operations` 中设置 `.owner = THIS_MODULE`：

```c
static const struct file_operations demo_fops = {
    .owner = THIS_MODULE,
    .open = demo_open,
    .release = demo_release,
    .read = demo_read,
};
```

当设备文件被打开时，VFS 会为该模块持有引用；文件关闭后再归还。因此，正常情况下模块仍被打开文件使用时，`rmmod` 不应让模块立即卸载。

结论：

- 仅使用 `kref`，不能保证已打开文件不会调用已经卸载的驱动代码。
- 仅依赖模块引用计数，也不能管理动态分配的设备私有对象。
- 两者配合，才能分别保证“代码还在”和“数据还在”。

### 4.2.3 kobject 父子关系中的引用计数

`kobject` 的 parent 关系不仅决定 sysfs 目录层级，也直接影响引用计数。当子对象以某个 kobject 作为 parent 成功注册时，子对象会持有父对象的引用，以保证子对象存活期间其父对象不会提前释放。

设 `my_kobj2` 的 parent 为 `my_kobj1`，而 `my_kobj3` 没有 parent，则引用关系如下：

```text
my_kobj1：创建完成后持有初始引用
    └── my_kobj2：注册时持有 my_kobj1 的 parent 引用

my_kobj3：独立的根对象，仅持有自身初始引用
```

因此，`my_kobj2` 注册成功后，`my_kobj1` 的引用计数会增加。释放时必须先释放 `my_kobj2`，由 kobject 核心归还其持有的 parent 引用；之后才能归还 `my_kobj1` 自身的初始引用。

## 4.3 kref 相关 API

### 4.3.1 `kref_init`

```c
void kref_init(struct kref *kref);
```

参数说明：

- `kref`：待初始化的引用计数器。

行为说明：

- 将引用计数设置为 1。
- 通常在对象创建完成后调用一次，表示创建者持有第一个引用。

示例：

```c
struct demo_dev *demo;

demo = kzalloc(sizeof(*demo), GFP_KERNEL);
if (!demo)
    return -ENOMEM;

kref_init(&demo->ref);
```

### 4.3.2 `kref_set`

```c
void kref_set(struct kref *kref, int num);
```

参数说明：

- `kref`：待设置的引用计数器。
- `num`：新的引用计数值。

行为说明：

- 直接将引用计数设置为指定值。
- 常规对象创建路径应优先使用 `kref_init()`，因为其语义明确且初始值固定为 1。

注意事项：

- `kref_set()` 不应用于正在被其他执行路径并发访问的对象。
- 不要把它当作普通的“加一”或“减一”接口；已发布对象的引用增减应使用 `kref_get()` 和 `kref_put()`。

### 4.3.3 `kref_get`

```c
void kref_get(struct kref *kref);
```

参数说明：

- `kref`：待增加引用的计数器。

行为说明：

- 引用计数加 1。
- 调用者因此获得对象的一个有效使用权。

使用时机：

- `open()` 成功后，文件将长期保存对象指针时。
- 异步工作、定时器或任务队列需要在稍后访问对象时。
- 将对象指针交给另一个可能延迟执行的上下文前。

### 4.3.4 `kref_put`

```c
int kref_put(struct kref *kref, void (*release)(struct kref *kref));
```

参数说明：

- `kref`：待减少引用的计数器。
- `release`：当引用计数减为 0 时调用的释放回调。

返回值说明：

- 非 0：本次调用使引用计数变为 0，且已执行 `release`。
- `0`：仍有其他引用，对象尚未释放。

注意事项：

- `release` 的参数是 `struct kref *`，不能直接写成设备结构体指针。
- release 中通常使用 `container_of()` 从 `kref` 找回外层对象，再执行 `kfree()` 或其他最终清理。

### 4.3.5 `kref_get_unless_zero`

```c
int kref_get_unless_zero(struct kref *kref);
```

行为说明：

- 仅当引用计数不为 0 时增加引用。
- 成功返回非 0；对象已经进入释放状态或计数为 0 时返回 0。

适用场景：

- 从全局链表、哈希表或 RCU 查找到对象指针后，需要尝试安全持有该对象。
- 不能确认对象是否已经进入最终释放阶段时。

## 4.4 kobject 引用计数观察示例

### 4.4.1 示例代码

下面的模块创建三个 kobject：

- `my_kobj1`：根对象。
- `my_kobj2`：以 `my_kobj1` 为 parent 的子对象。
- `my_kobj3`：通过 `kobject_init_and_add()` 创建的独立对象。

`kref_read()` 用于读取当前引用计数。该接口只适合教学、调试或已经具备同步保护的场景；生产驱动不应依据读取到的数值自行决定对象是否释放。

```c
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/kref.h>
#include <linux/module.h>
#include <linux/slab.h>

static struct kobject *my_kobj1;
static struct kobject *my_kobj2;
static struct kobject *my_kobj3;

static void my_kobj_release(struct kobject *kobj)
{
    kfree(kobj);
}

static const struct kobj_type mytype = {
    .release = my_kobj_release,
};

static void show_kref(const char *name, const struct kobject *kobj)
{
    pr_info("%s kref is %u\n", name, kref_read(&kobj->kref));
}

static int __init mykobject_init(void)
{
    int ret;

    my_kobj1 = kobject_create_and_add("my_kobject1", NULL);
    if (!my_kobj1)
        return -ENOMEM;

    show_kref("my_kobj1", my_kobj1);

    my_kobj2 = kobject_create_and_add("my_kobject2", my_kobj1);
    if (!my_kobj2) {
        ret = -ENOMEM;
        goto put_kobj1;
    }

    show_kref("my_kobj1", my_kobj1);
    show_kref("my_kobj2", my_kobj2);

    my_kobj3 = kzalloc(sizeof(*my_kobj3), GFP_KERNEL);
    if (!my_kobj3) {
        ret = -ENOMEM;
        goto put_kobj2;
    }

    ret = kobject_init_and_add(my_kobj3, &mytype, NULL, "%s", "my_kobject3");
    if (ret)
        goto put_kobj3;

    show_kref("my_kobj1", my_kobj1);
    show_kref("my_kobj2", my_kobj2);
    show_kref("my_kobj3", my_kobj3);
    return 0;

put_kobj3:
    kobject_put(my_kobj3);
put_kobj2:
    kobject_put(my_kobj2);
put_kobj1:
    kobject_put(my_kobj1);
    return ret;
}

static void __exit mykobject_exit(void)
{
    show_kref("my_kobj1 before put", my_kobj1);
    show_kref("my_kobj2 before put", my_kobj2);
    show_kref("my_kobj3 before put", my_kobj3);

    kobject_put(my_kobj3);
    my_kobj3 = NULL;

    kobject_put(my_kobj2);
    my_kobj2 = NULL;

    kobject_put(my_kobj1);
    my_kobj1 = NULL;
}

module_init(mykobject_init);
module_exit(mykobject_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_DESCRIPTION("kref and kobject demo module");
MODULE_VERSION("1.0");
```

### 4.4.2 引用计数变化分析

该示例中引用计数的典型变化如下：

1. `my_kobj1` 创建后持有自身初始引用。
2. `my_kobj2` 以 `my_kobj1` 为 parent 注册后，`my_kobj1` 额外获得一份 parent 引用；`my_kobj2` 持有自身初始引用。
3. `my_kobj3` 没有 parent，创建不会改变 `my_kobj1` 和 `my_kobj2` 的引用计数。
4. 退出时先执行 `kobject_put(my_kobj3)`，释放独立对象。
5. 再执行 `kobject_put(my_kobj2)`，子对象释放时归还 `my_kobj1` 的 parent 引用。
6. 最后执行 `kobject_put(my_kobj1)`，归还根对象自身初始引用并触发最终释放。

### 4.4.3 调试与释放注意事项

引用计数的读取和对象释放必须遵守以下边界：

- 尚未成功创建的对象指针为 `NULL`，不能读取其 kref。
- `kobject_put()` 可能立即触发 release；调用后不得再解引用该对象指针，也不能再读取其 kref。
- 将对象指针赋值为 `NULL` 只能防止后续误用，不能使已经释放的对象重新有效。
- `struct kref` 和 `refcount_t` 的内部字段不属于驱动可依赖的接口，应使用 `kref_read()` 等公开 API 进行调试观察。
- 存在 parent 关系时，退出和错误回滚必须遵循“子对象先于父对象”的顺序。

## 4.5 小结

`kref` 通过“持有引用才可使用、最后归还引用才释放”的规则管理对象生命周期：

- `kref_init()` 建立初始引用，`kref_get()` 取得新引用，`kref_put()` 归还引用并在归零时触发 release。
- 当应用程序已打开设备文件时，驱动不能立即释放 `file->private_data` 指向的私有对象；应由文件引用延后释放。
- kref 管理对象数据，模块引用计数管理驱动代码；二者职责不同，但在文件操作场景需要共同保证安全。
- 对象查找、open 和设备移除之间必须有明确并发同步，单纯增加引用计数不能替代锁或 RCU。

# 5 在 sysfs 中创建属性文件并实现读写

sysfs 将内核对象导出为目录后，还可以在该目录下创建属性文件，用于向用户空间展示状态或接收简单配置。属性文件通常对应一个确定的内核状态，例如设备开关、阈值、版本号或统计值。

本章以 `mykobj` 为例，在 `/sys/my_kobject1/` 下创建 `value1` 和 `value2` 两个属性文件，并通过 `cat` 和 `echo` 完成整数值的读取与修改。

## 5.1 sysfs 属性文件的工作逻辑

### 5.1.1 属性文件与 kobject 的关系

sysfs 属性必须依附于一个已经注册的 kobject。kobject 在 sysfs 中对应目录，属性在该目录中对应普通文本文件。

本章案例中的对象层次如下：

```text
struct mykobj
    ├── struct kobject kobj
    ├── int value1
    └── int value2

/sys/my_kobject1/
    ├── value1
    └── value2
```

`value1` 和 `value2` 文件不直接保存磁盘数据。读取文件时，sysfs 调用内核的 show 回调动态生成文本；写入文件时，sysfs 调用 store 回调解析用户输入并修改 `struct mykobj` 中的成员。

### 5.1.2 用户态与内核回调的调用关系

读取 `value1` 时，调用路径如下：

```text
cat /sys/my_kobject1/value1
    └── VFS read
        └── sysfs 属性读取路径
            └── my_kobj_sysfs_ops.show
                └── my_kobj_show
                    └── 读取 mykobj.value1 并格式化为文本
```

写入 `value2` 时，调用路径如下：

```text
echo 100 > /sys/my_kobject1/value2
    └── VFS write
        └── sysfs 属性写入路径
            └── my_kobj_sysfs_ops.store
                └── my_kobj_store
                    └── 解析 "100\n" 并更新 mykobj.value2
```

`echo` 默认会附加换行符，因此 store 回调接收到的字符串通常为 `"100\n"`。内核字符串转换接口可以正确处理这一形式。

## 5.2 相关结构体与 API

### 5.2.1 `struct attribute`

`struct attribute` 用于描述单个 sysfs 属性的名称和访问权限。

```c
struct attribute {
    const char *name;
    umode_t mode;
    struct module *owner;
};
```

成员说明：

- `name`：属性文件名，例如 `"value1"`。
- `mode`：文件权限，例如 `0644` 表示所有用户可读、仅特权用户可写。
- `owner`：属性所属模块，通常由内核框架或辅助宏处理。

`struct attribute` 只描述文件，不包含读写实现。实际的读写回调由 kobject 对应 `kobj_type` 中的 `sysfs_ops` 提供。

权限设计应遵循最小权限原则。教学案例使用 `0644`，避免将内核配置接口无条件暴露为所有用户可写的 `0666`。

### 5.2.2 `struct attribute_group`

多个相关属性可使用 `struct attribute_group` 作为整体管理。

```c
struct attribute_group {
    const char *name;
    umode_t (*is_visible)(struct kobject *kobj,
                          struct attribute *attr, int index);
    struct attribute **attrs;
};
```

成员说明：

- `name`：属性组目录名。为 `NULL` 时，属性直接创建在 kobject 对应目录下。
- `attrs`：属性指针数组，必须以 `NULL` 结尾。
- `is_visible`：可选的可见性回调，可根据运行状态隐藏属性或调整权限。

本章中 `my_kobj_attrs_group.name` 为 `NULL`，因此 `value1` 和 `value2` 直接出现在 `/sys/my_kobject1/` 下。

### 5.2.3 `struct sysfs_ops`

`sysfs_ops` 定义一类 kobject 属性的通用读写分派接口。

```c
struct sysfs_ops {
    ssize_t (*show)(struct kobject *kobj,
                    struct attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj,
                     struct attribute *attr,
                     const char *buf, size_t count);
};
```

参数说明：

- `kobj`：属性所属的 kobject。
- `attr`：当前被访问的属性，可通过 `attr->name` 区分 `value1` 和 `value2`。
- `buf`：sysfs 提供的缓冲区。show 向该缓冲区写入输出文本，store 从该缓冲区读取用户输入。
- `count`：本次写入的字节数。

返回值说明：

- show 成功时返回写入 `buf` 的字节数，失败时返回负错误码。
- store 成功时通常返回 `count`，失败时返回负错误码。

### 5.2.4 `struct kobj_type` 的 `default_groups`

`kobj_type` 的 `default_groups` 指向以 `NULL` 结尾的属性组数组：

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);
    const struct sysfs_ops *sysfs_ops;
    const struct attribute_group **default_groups;
};
```

当 `kobject_init_and_add()` 成功注册对象时，内核会创建 kobject 对应目录，并自动创建 `default_groups` 中的属性组。该方式将对象注册和固定属性创建绑定在同一生命周期中，通常比逐个调用 `sysfs_create_file()` 更容易维护。

### 5.2.5 `sysfs_create_file` 与 `sysfs_create_group`

属性也可以在对象注册完成后动态创建。

```c
int sysfs_create_file(struct kobject *kobj,
                      const struct attribute *attr);

int sysfs_create_group(struct kobject *kobj,
                       const struct attribute_group *grp);
```

适用场景：

- `sysfs_create_file()`：单个属性需要独立创建或删除。
- `sysfs_create_group()`：一组属性需要在运行时整体创建或删除。
- `default_groups`：属性在对象注册时就必须存在，且整个生命周期保持固定。

对应的删除接口为 `sysfs_remove_file()` 与 `sysfs_remove_group()`。使用 `default_groups` 创建的属性会在 kobject 删除时由 kobject/sysfs 核心统一清理，驱动不应重复删除。

## 5.3 属性读写实现案例

### 5.3.1 私有结构体与属性定义

`struct mykobj` 内嵌 `struct kobject`，并保存两个可由 sysfs 读写的整数成员。show/store 回调通过 `container_of()` 从 kobject 找回外层 `mykobj` 对象。

```c
struct mykobj {
    struct kobject kobj;
    struct mutex lock;
    int value1;
    int value2;
};

static struct attribute value1_attr = {
    .name = "value1",
    .mode = 0644,
};

static struct attribute value2_attr = {
    .name = "value2",
    .mode = 0644,
};
```

### 5.3.2 完整模块代码

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/kobject.h>
#include <linux/module.h>
#include <linux/mutex.h>
#include <linux/slab.h>
#include <linux/sysfs.h>

struct mykobj {
    struct kobject kobj;
    struct mutex lock;
    int value1;
    int value2;
};

static struct mykobj *my_kobj1;

static void my_kobj_release(struct kobject *kobj)
{
    struct mykobj *my_kobj;

    my_kobj = container_of(kobj, struct mykobj, kobj);
    kfree(my_kobj);
}

static struct attribute value1_attr = {
    .name = "value1",
    .mode = 0644,
};

static struct attribute value2_attr = {
    .name = "value2",
    .mode = 0644,
};

static struct attribute *my_kobj_attrs[] = {
    &value1_attr,
    &value2_attr,
    NULL,
};

static const struct attribute_group my_kobj_attrs_group = {
    .attrs = my_kobj_attrs,
};

static const struct attribute_group *my_kobj_groups[] = {
    &my_kobj_attrs_group,
    NULL,
};

static ssize_t my_kobj_show(struct kobject *kobj,
                            struct attribute *attr, char *buf)
{
    struct mykobj *my_kobj;
    int value;

    my_kobj = container_of(kobj, struct mykobj, kobj);

    mutex_lock(&my_kobj->lock);
    if (!strcmp(attr->name, "value1"))
        value = my_kobj->value1;
    else if (!strcmp(attr->name, "value2"))
        value = my_kobj->value2;
    else {
        mutex_unlock(&my_kobj->lock);
        return -EIO;
    }
    mutex_unlock(&my_kobj->lock);

    return sysfs_emit(buf, "%d\n", value);
}

static ssize_t my_kobj_store(struct kobject *kobj,
                             struct attribute *attr,
                             const char *buf, size_t count)
{
    struct mykobj *my_kobj;
    int value;
    int ret;

    ret = kstrtoint(buf, 0, &value);
    if (ret)
        return ret;

    my_kobj = container_of(kobj, struct mykobj, kobj);

    mutex_lock(&my_kobj->lock);
    if (!strcmp(attr->name, "value1"))
        my_kobj->value1 = value;
    else if (!strcmp(attr->name, "value2"))
        my_kobj->value2 = value;
    else {
        mutex_unlock(&my_kobj->lock);
        return -EIO;
    }
    mutex_unlock(&my_kobj->lock);

    return count;
}

static const struct sysfs_ops my_kobj_sysfs_ops = {
    .show = my_kobj_show,
    .store = my_kobj_store,
};

static const struct kobj_type mytype = {
    .release = my_kobj_release,
    .sysfs_ops = &my_kobj_sysfs_ops,
    .default_groups = my_kobj_groups,
};

static int __init mykobject_init(void)
{
    int ret;

    my_kobj1 = kzalloc(sizeof(*my_kobj1), GFP_KERNEL);
    if (!my_kobj1)
        return -ENOMEM;

    mutex_init(&my_kobj1->lock);
    my_kobj1->value1 = 1;
    my_kobj1->value2 = 1;

    ret = kobject_init_and_add(&my_kobj1->kobj, &mytype, NULL,
                               "%s", "my_kobject1");
    if (ret) {
        kobject_put(&my_kobj1->kobj);
        my_kobj1 = NULL;
        return ret;
    }

    return 0;
}

static void __exit mykobject_exit(void)
{
    kobject_put(&my_kobj1->kobj);
    my_kobj1 = NULL;
}

module_init(mykobject_init);
module_exit(mykobject_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_DESCRIPTION("sysfs attribute demo module");
MODULE_VERSION("1.0");
```

### 5.3.3 创建与访问结果

模块加载成功后，可观察到以下目录和属性文件：

```text
/sys/my_kobject1/
    ├── value1
    └── value2
```

示例操作：

```sh
cat /sys/my_kobject1/value1
echo 100 > /sys/my_kobject1/value1
cat /sys/my_kobject1/value1
```

第一次读取输出初始值 `1`，写入后再次读取输出 `100`。

## 5.4 错误处理与注意事项

### 5.4.1 输入和输出边界

属性文件属于文本接口，应使用与 sysfs 语义匹配的转换和输出函数：

- show 使用 `sysfs_emit()`，避免无长度限制的 `sprintf()`。
- store 使用 `kstrtoint()` 解析整数，避免未检查返回值的 `sscanf()`。
- 参数解析失败时返回负错误码，不能返回 `count` 伪装为写入成功。
- show 输出通常以换行符结尾，便于 shell 工具显示和脚本处理。

### 5.4.2 并发访问

sysfs 回调可以被多个进程并发执行。案例使用 `mutex` 保护 `value1` 和 `value2` 的读写，确保同一时刻只有一个执行路径修改或读取共享状态。

实际硬件驱动中，属性回调还可能与中断、工作队列或设备移除路径并发。锁的选择必须与执行上下文匹配；中断上下文不能直接获取可能睡眠的 mutex。

### 5.4.3 生命周期与清理

本章的属性组由 `default_groups` 创建，因此不需要在模块退出函数中手动调用 `sysfs_remove_group()`。`kobject_put()` 触发对象注销和最终 release 时，sysfs 核心会删除关联属性。

若属性使用 `sysfs_create_file()` 或 `sysfs_create_group()` 动态创建，则应在对象释放前调用对应的 remove 接口，并遵守“先删除用户可见接口，再释放回调使用的私有数据”的顺序。

## 5.5 基于 kobj_attribute 的属性封装

前面的案例在同一个 `my_kobj_show()` 和 `my_kobj_store()` 中通过 `attr->name` 区分 `value1`、`value2`。该方式适合属性数量较少且处理逻辑相近的场景。

当不同属性需要各自的格式化、范围校验或硬件操作时，可将属性描述扩展为 `struct kobj_attribute`。每个属性对象独立保存其 show/store 回调，通用的 `sysfs_ops` 仅负责完成一次分派。

### 5.5.1 `struct kobj_attribute`

`kobj_attribute` 是面向 kobject 的属性包装结构，在基础 `struct attribute` 之外增加专属读写回调。

```c
struct kobj_attribute {
    struct attribute attr;
    ssize_t (*show)(struct kobject *kobj,
                    struct kobj_attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj,
                     struct kobj_attribute *attr,
                     const char *buf, size_t count);
};
```

成员说明：

- `attr`：基础属性描述，提供文件名和访问权限。
- `show`：当前属性的专属读取回调。
- `store`：当前属性的专属写入回调。

与通用 `struct attribute` 相比，`kobj_attribute` 将“属性元数据”和“属性行为”放入同一个对象。`value1` 和 `value2` 可以分别实现不同的读写逻辑，无需在单个大回调中维护名称判断分支。

### 5.5.2 `__ATTR` 宏

`__ATTR` 宏用于静态初始化 `struct kobj_attribute`：

```c
#define __ATTR(_name, _mode, _show, _store) { \
    .attr = { .name = __stringify(_name), \
              .mode = VERIFY_OCTAL_PERMISSIONS(_mode) }, \
    .show = _show, \
    .store = _store, \
}
```

宏参数说明：

- `_name`：属性名称，不带引号，例如 `value1`。
- `_mode`：属性权限，例如 `0644` 或 `0664`。
- `_show`：读取属性时调用的函数；只写属性可设为 `NULL`。
- `_store`：写入属性时调用的函数；只读属性可设为 `NULL`。

以下定义创建两个可读写属性：

```c
static struct kobj_attribute value1_attr =
    __ATTR(value1, 0664, my_value1_show, my_value1_store);

static struct kobj_attribute value2_attr =
    __ATTR(value2, 0664, my_value2_show, my_value2_store);
```

`0664` 表示所有用户可读，属主和属组可写。实际驱动应结合设备管理权限选择模式；只允许系统管理员修改的配置通常使用 `0644`。

### 5.5.3 两层回调分派关系

基于 `kobj_attribute` 的属性访问包含两层分派：

```text
cat /sys/my_kobject1/value1
    └── sysfs_ops.show(kobj, attr, buf)
        └── container_of(attr, struct kobj_attribute, attr)
            └── kobj_attr->show(kobj, kobj_attr, buf)
                └── my_value1_show()

echo 100 > /sys/my_kobject1/value2
    └── sysfs_ops.store(kobj, attr, buf, count)
        └── container_of(attr, struct kobj_attribute, attr)
            └── kobj_attr->store(kobj, kobj_attr, buf, count)
                └── my_value2_store()
```

第一层 `sysfs_ops` 满足 kobject 类型对通用 sysfs 接口的要求；第二层由 `kobj_attribute` 按当前属性对象转发到实际回调。这样既保持 kobject/sysfs 的统一接口，又允许每个属性拥有独立行为。

### 5.5.4 属性定义与通用分派函数

以下代码保留 `mykobj`、属性组和 `default_groups` 的组织方式，并将每个 value 属性的实现拆分为独立回调。

```c
static ssize_t my_value1_show(struct kobject *kobj,
                              struct kobj_attribute *attr, char *buf)
{
    struct mykobj *my_kobj;
    int value;

    my_kobj = container_of(kobj, struct mykobj, kobj);

    mutex_lock(&my_kobj->lock);
    value = my_kobj->value1;
    mutex_unlock(&my_kobj->lock);

    return sysfs_emit(buf, "my value1 shows:%d\n", value);
}

static ssize_t my_value1_store(struct kobject *kobj,
                               struct kobj_attribute *attr,
                               const char *buf, size_t count)
{
    struct mykobj *my_kobj;
    int value;
    int ret;

    ret = kstrtoint(buf, 0, &value);
    if (ret)
        return ret;

    my_kobj = container_of(kobj, struct mykobj, kobj);

    mutex_lock(&my_kobj->lock);
    my_kobj->value1 = value;
    mutex_unlock(&my_kobj->lock);

    return count;
}

static ssize_t my_value2_show(struct kobject *kobj,
                              struct kobj_attribute *attr, char *buf)
{
    struct mykobj *my_kobj;
    int value;

    my_kobj = container_of(kobj, struct mykobj, kobj);

    mutex_lock(&my_kobj->lock);
    value = my_kobj->value2;
    mutex_unlock(&my_kobj->lock);

    return sysfs_emit(buf, "my value2 shows:%d\n", value);
}

static ssize_t my_value2_store(struct kobject *kobj,
                               struct kobj_attribute *attr,
                               const char *buf, size_t count)
{
    struct mykobj *my_kobj;
    int value;
    int ret;

    ret = kstrtoint(buf, 0, &value);
    if (ret)
        return ret;

    my_kobj = container_of(kobj, struct mykobj, kobj);

    mutex_lock(&my_kobj->lock);
    my_kobj->value2 = value;
    mutex_unlock(&my_kobj->lock);

    return count;
}

static struct kobj_attribute value1_attr =
    __ATTR(value1, 0664, my_value1_show, my_value1_store);
static struct kobj_attribute value2_attr =
    __ATTR(value2, 0664, my_value2_show, my_value2_store);

static struct attribute *my_kobj_attrs[] = {
    &value1_attr.attr,
    &value2_attr.attr,
    NULL,
};

static ssize_t my_kobj_show(struct kobject *kobj,
                            struct attribute *attr, char *buf)
{
    struct kobj_attribute *kobj_attr;

    kobj_attr = container_of(attr, struct kobj_attribute, attr);
    if (!kobj_attr->show)
        return -EIO;

    return kobj_attr->show(kobj, kobj_attr, buf);
}

static ssize_t my_kobj_store(struct kobject *kobj,
                             struct attribute *attr,
                             const char *buf, size_t count)
{
    struct kobj_attribute *kobj_attr;

    kobj_attr = container_of(attr, struct kobj_attribute, attr);
    if (!kobj_attr->store)
        return -EIO;

    return kobj_attr->store(kobj, kobj_attr, buf, count);
}

static const struct sysfs_ops my_kobj_sysfs_ops = {
    .show = my_kobj_show,
    .store = my_kobj_store,
};
```

### 5.5.5 适用场景与注意事项

`kobj_attribute` 模式适用于属性之间存在明显独立行为的场景：

- 不同属性具有不同的数据格式或输出内容。
- 不同属性需要不同的输入范围校验。
- 某些属性只读，另一些属性可读写。
- 属性数量增加后，单一 show/store 中的名称分支开始难以维护。

回调参数中的 `attr` 也可用于复用同一 show/store 函数，但属性行为已经独立时，优先提供专属回调更容易保持职责清晰。

`__ATTR` 定义的权限必须与回调能力对应：缺少 show 回调时不应提供读权限，缺少 store 回调时不应提供写权限。通用分派函数也应检查回调是否为 `NULL`，避免错误的属性定义导致空函数指针调用。

## 5.6 小结

sysfs 属性文件建立了用户空间与内核对象之间的轻量文本接口：

- `struct attribute` 描述文件名和权限，`attribute_group` 组织多个属性。
- `sysfs_ops.show` 和 `sysfs_ops.store` 分别处理用户态的读取与写入。
- `kobj_type.default_groups` 使属性组在 kobject 注册时自动创建，并随对象生命周期自动删除。
- show/store 通过 `container_of()` 找回外层私有对象，再访问具体的设备状态。
- 输入解析、输出格式、并发保护和对象释放顺序共同决定属性接口是否安全可靠。

# 6 总线如何注册

总线是 Linux 设备模型中连接设备与驱动的核心对象。设备和驱动分别注册到同一条总线后，总线负责维护二者的集合、执行匹配规则，并在匹配成功后进入驱动初始化路径。

总线不一定对应真实硬件连线。PCI、USB、I2C 是物理总线；platform 总线、虚拟总线等也可以作为设备与驱动的逻辑匹配域。

## 6.1 `struct bus_type`

### 6.1.1 核心结构体

`struct bus_type` 描述一类总线的名称、匹配策略、探测与移除策略，以及总线级 sysfs 属性。

```c
struct bus_type {
    const char *name;
    const char *dev_name;
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*uevent)(const struct device *dev,
                  struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    void (*sync_state)(struct device *dev);
    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);
    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);
    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;
    struct subsys_private *p;
};
```

不同内核版本的字段可能存在增减，但 `name`、`match`、`probe`、`remove` 和内部私有数据 `p` 是理解总线机制的关键。

### 6.1.2 主要成员

成员说明：

- `name`：总线名。注册成功后通常对应 `/sys/bus/<name>/` 目录。
- `match`：设备与候选驱动的匹配函数。返回非零表示匹配，返回 0 表示不匹配。
- `uevent`：为设备热插拔事件补充总线相关环境变量。
- `probe`：设备与驱动匹配后调用的探测路径。
- `remove`：设备解绑或驱动注销时调用的移除路径。
- `shutdown`：系统关机或重启时的回调。
- `bus_groups`：总线目录下默认创建的属性组。
- `dev_groups`：该总线上设备目录的默认属性组。
- `drv_groups`：该总线上驱动目录的默认属性组。
- `p`：指向 `struct subsys_private` 的内部私有数据，由 `bus_register()` 分配和管理，驱动不应自行读写。

### 6.1.3 `match` 的返回语义

`match` 的返回值不是 `strcmp()` 的原始返回值。`strcmp()` 在字符串相等时返回 0，而总线匹配要求“匹配成功时返回非零”，因此按名称匹配时应写为：

```c
static int mybus_match(struct device *dev, struct device_driver *drv)
{
    return !strcmp(dev_name(dev), drv->name);
}
```

等价的条件写法如下：

```c
if (!strcmp(dev_name(dev), drv->name))
    return 1;

return 0;
```

若直接返回 `strcmp(dev_name(dev), drv->name)`，名称相等时会返回 0，设备模型反而会把这对设备和驱动判断为不匹配。

## 6.2 总线注册与注销 API

### 6.2.1 `bus_register`

```c
int bus_register(struct bus_type *bus);
```

参数说明：

- `bus`：调用者定义的总线描述对象。

返回值说明：

- `0`：总线注册成功。
- `< 0`：注册失败，返回负错误码。

注册成功后，内核为总线建立内部对象关系，并在 sysfs 中创建总线目录及其 devices、drivers 子目录。

### 6.2.2 `bus_unregister`

```c
void bus_unregister(struct bus_type *bus);
```

`bus_unregister()` 撤销 `bus_register()` 建立的总线对象和 sysfs 节点。调用前必须保证该总线上的设备、驱动、接口和动态创建属性已经按其依赖关系完成注销。

基本原则：

1. 先注销或解绑总线上的设备与驱动。
2. 再删除动态创建的总线属性文件。
3. 最后执行 `bus_unregister()`。

### 6.2.3 `bus_create_file` 与 `bus_remove_file`

总线注册成功后，可在 `/sys/bus/<bus-name>/` 目录下创建总线级属性文件。

```c
int bus_create_file(struct bus_type *bus,
                    const struct bus_attribute *attr);

void bus_remove_file(struct bus_type *bus,
                     const struct bus_attribute *attr);
```

`bus_create_file()` 适合运行时单独增加总线属性；固定属性也可通过 `bus_type.bus_groups` 在注册时自动创建。

## 6.3 总线属性结构体

### 6.3.1 `struct bus_attribute`

`bus_attribute` 将基础 attribute 与总线专属的 show/store 回调组合起来。

```c
struct bus_attribute {
    struct attribute attr;
    ssize_t (*show)(const struct bus_type *bus, char *buf);
    ssize_t (*store)(const struct bus_type *bus,
                     const char *buf, size_t count);
};
```

成员说明：

- `attr`：属性名与权限，例如 `my_bus_attr` 和 `0644`。
- `show`：用户态读取总线属性时调用。
- `store`：用户态写入总线属性时调用；只读属性可设为 `NULL`。

不同内核版本中 `show/store` 的 bus 参数是否带 `const` 可能不同，应以目标内核头文件中的实际原型为准。

### 6.3.2 总线属性访问路径

注册名为 `mybus` 的总线并创建 `my_bus_attr` 后，属性文件路径为：

```text
/sys/bus/mybus/my_bus_attr
```

读取路径如下：

```text
cat /sys/bus/mybus/my_bus_attr
    └── VFS read
        └── sysfs bus 属性读取路径
            └── bus_attribute.show
                └── my_bus_show
```

## 6.4 总线注册与属性文件案例

### 6.4.1 案例逻辑

本案例定义名为 `mybus` 的逻辑总线：

1. `mybus_match()` 比较设备名与驱动名，名称相同则匹配成功。
2. `bus_register()` 注册总线并创建 `/sys/bus/mybus/`。
3. `bus_create_file()` 在总线目录下创建只读属性 `my_bus_attr`。
4. 用户态读取属性文件时，`my_bus_show()` 返回总线描述文本。
5. 模块退出时先删除属性文件，再注销总线。

本案例只演示总线框架、匹配函数和总线属性。设备与驱动的实际注册、probe 资源申请和 remove 清理将在设备与驱动章节中分别展开。

### 6.4.2 完整模块代码

```c
#include <linux/device.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/string.h>

static int mybus_match(struct device *dev, struct device_driver *drv)
{
    return !strcmp(dev_name(dev), drv->name);
}

static struct bus_type my_bus_type = {
    .name = "mybus",
    .match = mybus_match,
};

static ssize_t my_bus_show(const struct bus_type *bus, char *buf)
{
    return sysfs_emit(buf, "This is my bus attribute\n");
}

static const struct bus_attribute my_bus_attr = {
    .attr = {
        .name = "my_bus_attr",
        .mode = 0444,
    },
    .show = my_bus_show,
};

static int __init mybus_init(void)
{
    int ret;

    ret = bus_register(&my_bus_type);
    if (ret)
        return ret;

    ret = bus_create_file(&my_bus_type, &my_bus_attr);
    if (ret) {
        bus_unregister(&my_bus_type);
        return ret;
    }

    return 0;
}

static void __exit mybus_exit(void)
{
    bus_remove_file(&my_bus_type, &my_bus_attr);
    bus_unregister(&my_bus_type);
}

module_init(mybus_init);
module_exit(mybus_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_DESCRIPTION("custom bus and sysfs attribute demo");
MODULE_VERSION("1.0");
```

### 6.4.3 关于 `probe` 回调

`bus_type.probe` 是总线级 probe 包装回调。部分总线可在这里完成总线特有处理，然后再调用 `dev->driver->probe(dev)`；若总线不提供该回调，设备模型会使用默认 probe 路径调用已绑定驱动的 probe。

一个总线级包装函数可写为：

```c
static int mybus_probe(struct device *dev)
{
    struct device_driver *drv = dev->driver;

    if (!drv || !drv->probe)
        return -ENODEV;

    return drv->probe(dev);
}
```

若将该函数赋值给 `.probe`，必须直接返回 `drv->probe(dev)` 的返回值。probe 失败时，设备模型需要获知错误码并撤销本次绑定；不能无条件返回 0。

对于仅演示总线注册和属性文件的模块，省略 `.probe` 更简洁。只有后续真正注册 `device` 和 `device_driver` 时，才需要根据总线设计决定是否提供总线级 probe 包装。

## 6.5 `bus_register` 的实现过程

### 6.5.1 `subsys_private` 私有数据

`struct bus_type` 中的 `p` 指向设备模型内部的 `struct subsys_private`。该结构体用于保存总线注册后生成的运行时状态，典型成员如下：

```c
struct subsys_private {
    struct kset subsys;
    struct kset *devices_kset;
    struct list_head interfaces;
    struct mutex mutex;
    struct kset *drivers_kset;
    struct klist klist_devices;
    struct klist klist_drivers;
    struct blocking_notifier_head bus_notifier;
    struct bus_type *bus;
};
```

不同内核版本中成员名称或排列会变化，但其职责保持稳定：

- `subsys`：总线子系统自身的 kset，对应总线的根对象。
- `devices_kset`：总线设备集合，对应 `devices` sysfs 子目录。
- `drivers_kset`：总线驱动集合，对应 `drivers` sysfs 子目录。
- `klist_devices`：内核内部设备链表，用于匹配和遍历。
- `klist_drivers`：内核内部驱动链表，用于匹配和遍历。
- `interfaces`：挂接到该总线的 bus interface 链表。
- `bus_notifier`：设备添加、删除、绑定和解绑等事件的通知链。
- `bus`：回指拥有该私有数据的 `struct bus_type`。

`bus_type.p` 属于设备模型私有实现。总线实现者负责定义 `struct bus_type` 的公开回调和属性组，不应在驱动中直接操作 `p` 内部成员。

### 6.5.2 注册主链路

`bus_register()` 的典型实现路径可概括为：

```text
bus_register(bus)
    ├── 分配 struct subsys_private
    ├── bus->p = priv
    ├── 设置 priv->bus、通知链、内部链表和互斥锁
    ├── kset_register(&priv->subsys)
    │   └── 在 /sys/bus/ 下创建 <bus-name> 目录
    ├── 创建总线默认属性和 uevent 属性
    ├── kset_create_and_add("devices", ..., &priv->subsys.kobj)
    │   └── 创建 /sys/bus/<bus-name>/devices/
    ├── kset_create_and_add("drivers", ..., &priv->subsys.kobj)
    │   └── 创建 /sys/bus/<bus-name>/drivers/
    ├── 创建 bus_groups 中的默认属性组
    └── 注册完成
```

注册完成后，总线与 kobject/kset 的主要连接关系如下：

```text
struct bus_type my_bus_type
    └── p
        └── struct subsys_private
            ├── subsys.kobj
            │   └── /sys/bus/mybus/
            ├── devices_kset->kobj
            │   └── /sys/bus/mybus/devices/
            └── drivers_kset->kobj
                └── /sys/bus/mybus/drivers/
```

这里的 parent 关系由 kobject/kset 建立：`devices_kset->kobj` 和 `drivers_kset->kobj` 都以 `subsys.kobj` 为父对象。因此 sysfs 中形成总线目录包含 devices、drivers 两个子目录的层级结构。

### 6.5.3 注册失败回滚

`bus_register()` 内部创建多个资源。任一步骤失败时，内核会按创建顺序的反向路径注销已创建的属性组、devices/drivers kset 和 subsys kset，最后释放 `subsys_private`，并将 `bus->p` 恢复为 `NULL`。

自定义总线模块也应遵守相同原则：

1. `bus_register()` 成功后才允许调用 `bus_create_file()`。
2. `bus_create_file()` 失败时，调用 `bus_unregister()` 回滚总线注册。
3. 退出路径先调用 `bus_remove_file()`，再调用 `bus_unregister()`。

## 6.6 platform 总线的注册与匹配

platform 总线是 Linux 内核中的逻辑总线，主要用于描述不能由 PCI、USB、I2C 等可枚举总线自动发现的片上设备和板级设备，例如 SoC 内部的串口、GPIO、定时器、看门狗和显示控制器。

platform 设备和 platform 驱动都注册到 `platform_bus_type`。总线通过 `platform_match()` 判断设备与驱动是否匹配，匹配成功后进入 platform 驱动的 probe 路径。

### 6.6.1 `platform_bus_init` 初始化路径

platform 总线的初始化位于 `drivers/base/platform.c`，典型主干如下：

```c
int __init platform_bus_init(void)
{
    int error;

    early_platform_cleanup();

    error = device_register(&platform_bus);
    if (error) {
        put_device(&platform_bus);
        return error;
    }

    error = bus_register(&platform_bus_type);
    if (error)
        device_unregister(&platform_bus);

    return error;
}
```

该函数通常通过 initcall 机制在内核启动阶段执行。其注册顺序为：

1. 调用 `early_platform_cleanup()` 清理早期 platform 设备相关状态。
2. 调用 `device_register(&platform_bus)` 注册名为 `platform` 的根设备。
3. 调用 `bus_register(&platform_bus_type)` 注册 platform 总线。
4. 若总线注册失败，调用 `device_unregister(&platform_bus)` 回滚先前注册的根设备。

这里的 `platform_bus` 是一个 `struct device`，为 platform 设备提供顶层 parent；`platform_bus_type` 是一个 `struct bus_type`，为 platform 设备与 platform 驱动提供匹配和绑定域。二者职责不同：

- `platform_bus`：形成 `/sys/devices/platform/` 的设备层级根节点。
- `platform_bus_type`：形成 `/sys/bus/platform/`，并管理该总线的设备、驱动与匹配关系。

### 6.6.2 `platform_bus_type`

`platform_bus_type` 是内核预定义的总线对象，其典型关键成员如下：

```c
struct bus_type platform_bus_type = {
    .name = "platform",
    .dev_groups = platform_dev_group,
    .match = platform_match,
    .uevent = platform_uevent,
    .probe = platform_probe,
    .remove = platform_remove,
    .shutdown = platform_shutdown,
};
```

成员说明：

- `name`：总线名称。注册后对应 `/sys/bus/platform/`。
- `dev_groups`：platform 设备目录的默认属性组。
- `match`：指向 `platform_match()`，负责选择适合当前 platform 设备的驱动。
- `uevent`：为 platform 设备事件补充 `MODALIAS` 等环境变量。
- `probe`：把通用 `struct device` 转换为 `struct platform_device`，再调用 platform 驱动的 probe。
- `remove`：设备解绑时调用 platform 驱动的 remove。
- `shutdown`：系统关机或重启时调用 platform 驱动的 shutdown。

### 6.6.3 `platform_match` 的职责

`platform_match()` 的输入是一个设备模型通用的 `struct device` 和 `struct device_driver`。函数会使用 `to_platform_device()`、`to_platform_driver()` 等辅助宏恢复 platform 专属对象，再按若干规则判断是否匹配。

典型逻辑可概括为：

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    if (of_driver_match_device(dev, drv))
        return 1;

    if (acpi_driver_match_device(dev, drv))
        return 1;

    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    return strcmp(pdev->name, drv->name) == 0;
}
```

不同内核版本和配置条件会使某些分支被编译为条件代码，但教学上可将 platform 匹配优先级理解为以下顺序。

### 6.6.4 platform 匹配优先级

#### 6.6.4.1 Device Tree 的 `compatible`

当设备来自 Device Tree，并且驱动提供 `of_match_table` 时，内核优先按照设备节点的 `compatible` 属性匹配驱动的 `struct of_device_id` 表。

```c
static const struct of_device_id demo_of_match[] = {
    { .compatible = "acme,demo-device" },
    { }
};
MODULE_DEVICE_TABLE(of, demo_of_match);

static struct platform_driver demo_driver = {
    .driver = {
        .name = "demo-device",
        .of_match_table = demo_of_match,
    },
};
```

Device Tree 匹配可以区分同名但硬件兼容性不同的设备，是嵌入式平台驱动最常见的匹配方式。

#### 6.6.4.2 ACPI 设备标识

在使用 ACPI 描述硬件的平台上，若驱动提供 `acpi_match_table`，内核会按 ACPI 设备标识匹配。

```c
static const struct acpi_device_id demo_acpi_match[] = {
    { "ACME0001", 0 },
    { }
};
MODULE_DEVICE_TABLE(acpi, demo_acpi_match);
```

该规则主要用于 x86 等依赖 ACPI 枚举硬件的平台。

#### 6.6.4.3 `platform_device_id` 表

若驱动定义 `platform_driver.id_table`，`platform_match()` 会在该表中查找与 `platform_device.name` 相同的条目。

```c
static const struct platform_device_id demo_id_table[] = {
    { "demo-device", 0 },
    { }
};
MODULE_DEVICE_TABLE(platform, demo_id_table);

static struct platform_driver demo_driver = {
    .id_table = demo_id_table,
    .driver = {
        .name = "demo-device-driver",
    },
};
```

匹配成功后，`platform_get_device_id()` 可用于在 probe 中取得匹配条目及其 `driver_data`。

#### 6.6.4.4 名称匹配

当设备不使用 Device Tree、ACPI 和 ID 表匹配时，platform 总线最后比较：

```c
strcmp(pdev->name, drv->name) == 0
```

即 `platform_device.name` 与 `platform_driver.driver.name` 完全一致时匹配成功。

名称匹配适合最小示例或早期板级代码，但缺少硬件兼容性信息。新的设备树驱动通常应优先提供 `of_match_table`，避免仅依赖名称。

### 6.6.5 platform 设备、总线与 sysfs 关系

platform 总线注册完成后，设备模型中同时存在设备层级视图和总线视图：

```text
/sys/devices/platform/
    └── <platform-device>/

/sys/bus/platform/
    ├── devices/
    │   └── <platform-device> -> ../../../devices/platform/<platform-device>
    └── drivers/
        └── <platform-driver>/
            └── <platform-device> -> ../../../../devices/platform/<platform-device>
```

其中：

- `/sys/devices/platform/` 体现 `platform_bus` 作为 parent 的设备层级。
- `/sys/bus/platform/devices/` 提供 platform 总线中设备的索引视图。
- `/sys/bus/platform/drivers/` 提供 platform 驱动及其已绑定设备的索引视图。

### 6.6.6 注册失败与退出顺序

`platform_bus_init()` 的错误路径体现了设备模型资源回滚原则：总线注册依赖先前已注册的 `platform_bus` 根设备，因此总线注册失败时必须注销根设备。

运行中的 platform 设备和 platform 驱动由各自注册、解绑和 remove 路径管理。系统核心 platform 总线本身通常不会作为可卸载模块由驱动代码主动注销；自定义逻辑总线则应遵循“先注销设备和驱动，再注销总线”的顺序。

## 6.7 小结

总线将设备、驱动、匹配规则和总线级 sysfs 视图组织为统一整体：

- `bus_type.match` 决定设备与驱动是否匹配，名称匹配时必须在字符串相等时返回非零。
- `bus_register()` 创建 `subsys_private`、总线 kset，以及 devices/drivers 子 kset，从而构成 `/sys/bus/<name>/` 目录结构。
- `bus_attribute`、`bus_create_file()` 和 `bus_remove_file()` 用于管理总线目录下的动态属性文件。
- `bus_type.p` 保存设备模型私有状态，连接公开的总线对象与内部 kobject/kset、设备链表和驱动链表。
- 总线的设备、驱动和动态属性应在注销总线前完成清理，且回收顺序必须与创建顺序相反。

# 7 设备注册到自定义总线

设备注册是将 `struct device` 接入设备模型、sysfs 和所属总线的过程。设备成功注册后，内核会为其建立对象生命周期、sysfs 目录、总线索引、属性文件和驱动匹配关系。

本章使用第六章定义的 `my_bus_type`。因此模块加载顺序必须满足以下依赖关系：先注册 `my_bus_type`，再注册 `my_device_in_bus`；设备注销完成后，才允许注销总线。

## 7.1 注册到 `my_bus_type` 的设备案例

### 7.1.1 `struct device` 的关键成员

设备模型使用 `struct device` 表示一个设备实例。案例中直接定义静态设备对象：

```c
struct device my_device_in_bus = {
    .init_name = "my_device_in_bus",
    .bus = &my_bus_type,
    .release = my_device_in_bus_release,
    .devt = MKDEV(255, 0),
};
```

成员说明：

- `init_name`：设备的初始名称。`device_initialize()` 会将其转化为 kobject 名称；生产驱动也可在初始化后使用 `dev_set_name()` 设置名称。
- `bus`：设备所属的总线。这里指向已注册的 `my_bus_type`。
- `release`：设备最后一个引用归还时的释放回调。该回调是 `struct device` 生命周期的必要组成部分。
- `devt`：设备号，由主设备号和次设备号组成。仅设置 `devt` 不会自动生成可访问的字符设备节点，字符设备还需要 cdev、class 或 devtmpfs/udev 等配套机制。

### 7.1.2 静态设备对象的 release 回调

案例中的 `my_device_in_bus` 是静态全局对象，不由 `kzalloc()` 分配。因此 release 回调不应对 `dev` 调用 `kfree()`，只需完成与静态对象匹配的最终清理或记录：

```c
static void my_device_in_bus_release(struct device *dev)
{
    pr_info("release my_device_in_bus\n");
}
```

若 `struct device` 由动态内存分配，release 回调通常通过 `container_of()` 找回外层私有对象，再释放其动态内存。无论静态还是动态对象，注册前都必须提供有效 release 回调，否则设备模型无法安全完成对象生命周期管理。

### 7.1.3 完整模块代码

以下模块假定 `my_bus_type` 由另一个模块导出并已成功注册：

```c
#include <linux/device.h>
#include <linux/init.h>
#include <linux/kdev_t.h>
#include <linux/kernel.h>
#include <linux/module.h>

extern struct bus_type my_bus_type;

static void my_device_in_bus_release(struct device *dev)
{
    pr_info("release my_device_in_bus\n");
}

static struct device my_device_in_bus = {
    .init_name = "my_device_in_bus",
    .bus = &my_bus_type,
    .release = my_device_in_bus_release,
    .devt = MKDEV(255, 0),
};

static int __init mydeviceinbus_init(void)
{
    return device_register(&my_device_in_bus);
}

static void __exit mydeviceinbus_exit(void)
{
    device_unregister(&my_device_in_bus);
}

module_init(mydeviceinbus_init);
module_exit(mydeviceinbus_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_DESCRIPTION("device registration on a custom bus");
MODULE_VERSION("1.0");
```

若 `my_bus_type` 位于单独模块中，该模块还需要导出符号，例如：

```c
EXPORT_SYMBOL_GPL(my_bus_type);
```

模块依赖关系应由构建系统或模块元数据保证。设备模块不能在总线模块尚未注册时使用 `my_bus_type`。

## 7.2 `device_register` 的两阶段流程

### 7.2.1 函数原型与调用关系

```c
int device_register(struct device *dev);
```

`device_register()` 将设备注册过程分为初始化和添加两个阶段，典型实现如下：

```c
int device_register(struct device *dev)
{
    device_initialize(dev);
    return device_add(dev);
}
```

调用关系：

```text
device_register()
    ├── device_initialize()
    │   └── 初始化 struct device、kobject、锁和资源管理成员
    └── device_add()
        └── 将设备加入 sysfs、总线、class 和驱动匹配路径
```

两个阶段的职责必须区分：

- `device_initialize()` 不会把设备公布到 sysfs，也不会触发驱动匹配。
- `device_add()` 才会使设备对设备模型其他部分可见，并可能立即触发总线匹配和 probe。

### 7.2.2 `device_unregister`

```c
void device_unregister(struct device *dev);
```

`device_unregister()` 用于撤销已成功注册的设备。其典型效果包括从 sysfs 和总线中移除设备、解除与驱动的绑定，并归还设备自身引用。最终引用归零时调用 `dev->release`。

对于静态设备对象，`device_unregister()` 返回后 release 回调已经具备执行条件，因此不应再将该对象当作已注册设备访问。若后续需要再次注册同一个静态 `struct device`，应重新完整初始化相关成员，而不是直接重复调用 `device_add()`。

## 7.3 `device_initialize`：数据结构初始化

### 7.3.1 kobject 与 `devices_kset`

`device_initialize()` 的第一项关键工作，是建立 device 与通用设备集合的关系：

```c
dev->kobj.kset = devices_kset;
kobject_init(&dev->kobj, &device_ktype);
```

说明：

- `devices_kset` 是设备模型维护的全局设备集合。
- `dev->kobj.kset = devices_kset` 指定该 device 的 kobject 所属集合。
- `kobject_init()` 初始化引用计数、对象类型和基础状态；此时尚未创建 sysfs 目录，也尚未把对象接入 kset 链表。
- `device_ktype` 定义 device 对象的 release、sysfs 操作和默认属性等通用行为。

在后续 `device_add()` 成功后，设备会在 `/sys/devices/` 层级中拥有实际目录。目录位置由 `dev->parent`、总线父设备及设备模型的 parent 解析规则共同确定；不能简单理解为在初始化时立即于 `/sys/device/` 创建目录。

### 7.3.2 DMA、锁和资源管理成员

`device_initialize()` 还会初始化设备运行期间需要的通用数据结构。不同内核版本的源码细节有所差异，典型工作包括：

```c
INIT_LIST_HEAD(&dev->dma_pools);

mutex_init(&dev->mutex);
lockdep_set_novalidate_class(&dev->mutex);

spin_lock_init(&dev->devres_lock);
INIT_LIST_HEAD(&dev->devres_head);

device_pm_init(dev);
set_dev_node(dev, -1);
```

成员与函数说明：

- `dma_pools`：初始化设备相关 DMA 内存池链表。
- `dev->mutex`：保护设备对象的部分状态转换。`lockdep_set_novalidate_class()` 用于标记该锁的 lockdep 类别处理方式，避免设备模型内部锁类验证产生不必要的告警；它不直接提高设备功能或内存安全性。
- `devres_lock` 和 `devres_head`：保护和维护 devres 资源链表。`devm_kzalloc()`、`devm_request_irq()` 等 managed resource 接口申请的资源会记录在该链表中。
- `device_pm_init()`：初始化设备电源管理相关状态。
- `set_dev_node(dev, -1)`：初始化 NUMA 节点信息为未指定状态，不是为设备分配主次设备号。

### 7.3.3 MSI 与 device link 成员

根据内核配置和版本，`device_initialize()` 还会初始化 MSI 与 device link 相关成员：

```c
#ifdef CONFIG_GENERIC_MSI_IRQ
INIT_LIST_HEAD(&dev->msi_list);
#endif

INIT_LIST_HEAD(&dev->links.consumers);
INIT_LIST_HEAD(&dev->links.suppliers);
INIT_LIST_HEAD(&dev->links.needs_suppliers);
INIT_LIST_HEAD(&dev->links.defer_hook);
dev->links.status = DL_DEV_NO_DRIVER;
```

说明：

- `msi_list`：维护设备相关的 MSI/MSI-X 中断描述项。
- `links.consumers` 与 `links.suppliers`：维护 device link 的消费者和供应者关系。
- `links.needs_suppliers` 与 `links.defer_hook`：辅助处理依赖尚未满足时的延迟 probe 等状态。
- `DL_DEV_NO_DRIVER`：表示设备初始化后尚未绑定驱动；后续匹配和 probe 成功后状态会改变。

这些成员服务于设备依赖、电源管理、延迟探测和资源生命周期，不是普通的设备与驱动直接通信链表。

## 7.4 `device_add`：将设备接入设备模型

### 7.4.1 sysfs 目录与属性文件

`device_add()` 在设备初始化完成后执行实际注册。其典型工作包括：

1. 检查 device 名称、parent、release 等必要条件。
2. 解析 parent kobject，建立设备的 sysfs 父子层级。
3. 调用 kobject add 路径创建 `/sys/devices/` 下的实际设备目录。
4. 创建设备默认属性、`uevent` 属性及设备所属 bus/class/type 的属性组。
5. 生成设备热插拔事件，使用户空间能够感知设备出现。

设备目录示意：

```text
/sys/devices/
    └── my_device_in_bus/
```

实际目录可能位于更深的 parent 层级之下。对于本案例，设备未显式设置 `parent`，最终层级仍应以目标内核对总线 parent 的处理规则和实际 sysfs 输出为准。

### 7.4.2 总线与 class 软链接

当 `dev->bus = &my_bus_type` 时，`device_add()` 会把设备接入该总线的设备集合，并在总线目录中创建指向实际设备目录的软链接：

```text
/sys/bus/mybus/devices/
    └── my_device_in_bus -> ../../../devices/my_device_in_bus
```

若同时设置 `dev->class`，设备模型还会在 `/sys/class/<class-name>/` 创建指向该设备实际目录的软链接。总线视图和 class 视图不会复制新的 device 对象，而是提供通向 `/sys/devices/` 实际目录的不同索引路径。

### 7.4.3 驱动匹配与设备链表

`device_add()` 将设备加入总线的内部设备链表，并尝试与已经注册到 `my_bus_type` 的驱动匹配。匹配过程沿用第六章的总线机制：

```text
device_add()
    └── bus_add_device()
        ├── 创建 /sys/bus/mybus/devices/ 中的设备软链接
        ├── 加入 my_bus_type.p->klist_devices
        └── device_attach()
            └── 依次尝试 my_bus_type.match(dev, drv)
                └── 匹配成功后进入 probe 路径
```

若此时没有匹配驱动，设备仍保持已注册状态；后续驱动注册到同一总线时，设备模型会再次发起匹配。

### 7.4.4 错误回滚顺序

`device_add()` 创建目录、属性、软链接并加入内部链表。任何步骤失败时，内核会按反向顺序删除已经创建的资源，例如：

1. 停止匹配和热插拔相关通知。
2. 从总线和 class 的索引中移除软链接。
3. 删除设备属性文件和 sysfs 目录。
4. 归还在 add 路径中持有的 parent 或其他对象引用。

调用者通过 `device_register()` 注册设备时，若 `device_add()` 返回错误，应进一步调用 `put_device(dev)`，使 `device_initialize()` 建立的初始引用走到 release 回调。对动态分配设备对象，这一步通常决定何时释放内存。

## 7.5 platform_device 的注册

`platform_device` 是 platform 总线上的设备描述对象。它在通用 `struct device` 的基础上增加设备名称、ID、资源数组等 platform 专属信息，用于表示 SoC 内部控制器和板级逻辑设备。

前面 `device_initialize()` 与 `device_add()` 建立了所有设备共享的注册模型。platform 子系统并不替换这套模型，而是在进入 `device_add()` 前补充默认 parent、总线归属、实例 ID 和资源树登记等 platform 专属工作。

### 7.5.1 `struct platform_device`

#### 7.5.1.1 核心结构体

```c
struct platform_device {
    const char *name;
    int id;
    bool id_auto;
    struct device dev;
    u64 platform_dma_mask;
    struct pdev_archdata archdata;
    u32 num_resources;
    struct resource *resource;
};
```

成员说明：

- `name`：platform 设备名称，参与设备命名和 platform 名称匹配。
- `id`：设备实例 ID，用于区分同名设备。
- `id_auto`：标记该 ID 是否由内核自动分配。
- `dev`：嵌入的通用设备对象，承载 kobject、sysfs、总线和驱动绑定能力。
- `num_resources`：资源数组元素数量。
- `resource`：设备资源数组，常用于描述寄存器地址范围和 I/O 端口范围。

### 7.5.2 注册入口

#### 7.5.2.1 `platform_device_register`

```c
int platform_device_register(struct platform_device *pdev);
```

`platform_device_register()` 是一次性注册 platform 设备的常用接口。其逻辑可以概括为：

```text
platform_device_register(pdev)
    ├── platform_device_initialize(pdev)
    │   └── device_initialize(&pdev->dev)
    └── platform_device_add(pdev)
        └── 配置 parent、bus、名称、ID、资源后调用 device_add()
```

与通用 `device_register()` 类似，platform 注册也分为“初始化”与“添加”两个阶段。若调用者已经独立执行过 `platform_device_initialize()`，则应直接调用 `platform_device_add()`，不能再次调用 `platform_device_register()`。

#### 7.5.2.2 `platform_device_add`

```c
int platform_device_add(struct platform_device *pdev);
```

该函数负责在通用 `device_add()` 之前完成 platform 专属配置，包括设置默认 parent、指定 platform 总线、生成设备名、声明资源和处理自动 ID。

### 7.5.3 `platform_bus` 与默认父设备

#### 7.5.3.1 `platform_bus` 是什么

`platform_bus` 是由内核设备核心定义的 `struct device`。第六章中的 `platform_bus_init()` 会在注册 `platform_bus_type` 前调用：

```c
device_register(&platform_bus);
```

该调用注册名为 `platform` 的根设备，使 sysfs 中出现：

```text
/sys/devices/platform/
```

该目录是未显式设置 parent 的 platform 设备的默认父目录。

#### 7.5.3.2 设置 parent 和 bus

`platform_device_add()` 的开头包含以下关键设置：

```c
struct device *dev = &pdev->dev;

if (!dev->parent)
    dev->parent = &platform_bus;

dev->bus = &platform_bus_type;
```

逻辑说明：

- 若调用者已经设置 `pdev->dev.parent`，platform 设备保留该父设备，形成更具体的物理或逻辑层级。
- 若 parent 为空，内核将其设置为 `platform_bus`，因此设备目录默认创建在 `/sys/devices/platform/` 下。
- `dev->bus = &platform_bus_type` 使该设备加入 platform 总线的设备集合，并参与 `platform_match()` 驱动匹配。

父设备关系和总线关系彼此独立：

- `parent` 决定 `/sys/devices/` 中的层级位置。
- `bus` 决定 `/sys/bus/platform/` 中的索引、设备驱动匹配和绑定关系。

### 7.5.4 ID 与设备名称

#### 7.5.4.1 ID 分支

`platform_device_add()` 根据 `pdev->id` 生成 `dev_name(dev)`。典型逻辑如下：

```c
switch (pdev->id) {
default:
    dev_set_name(dev, "%s.%d", pdev->name, pdev->id);
    break;
case PLATFORM_DEVID_NONE:
    dev_set_name(dev, "%s", pdev->name);
    break;
case PLATFORM_DEVID_AUTO:
    ret = ida_alloc(&platform_devid_ida, GFP_KERNEL);
    if (ret < 0)
        return ret;

    pdev->id = ret;
    pdev->id_auto = true;
    dev_set_name(dev, "%s.%d.auto", pdev->name, pdev->id);
    break;
}
```

不同 ID 的命名结果如下：

| `pdev->id`            | 命名形式               | 说明                              |
| --------------------- | ------------------ | ------------------------------- |
| 普通非负 ID               | `<name>.<id>`      | 用于区分多个同名设备实例                    |
| `PLATFORM_DEVID_NONE` | `<name>`           | 不追加实例 ID                        |
| `PLATFORM_DEVID_AUTO` | `<name>.<id>.auto` | 内核通过 IDA 自动分配 ID，并添加 `.auto` 后缀 |

`.auto` 后缀用于避免自动分配的 ID 与调用者显式指定的同号 ID 在设备命名空间中冲突。

### 7.5.5 platform 资源的声明与资源树

#### 7.5.5.1 `struct resource`

platform 设备可通过 `pdev->resource` 描述硬件资源。例如寄存器地址范围、I/O 端口范围、IRQ 等信息可在设备创建阶段提供，并在驱动 probe 中通过 `platform_get_resource()` 等接口获取。

本节聚焦 `platform_device_add()` 对内存资源和 I/O 端口资源的资源树登记逻辑。

#### 7.5.5.2 遍历资源数组

注册过程中，内核遍历 `pdev->resource`：

```c
for (i = 0; i < pdev->num_resources; i++) {
    struct resource *p;
    struct resource *r = &pdev->resource[i];

    if (!r->name)
        r->name = dev_name(dev);

    p = r->parent;
    if (!p) {
        if (resource_type(r) == IORESOURCE_MEM)
            p = &iomem_resource;
        else if (resource_type(r) == IORESOURCE_IO)
            p = &ioport_resource;
    }

    if (p) {
        ret = insert_resource(p, r);
        if (ret)
            goto failed;
    }
}
```

处理规则：

- 未指定 `r->name` 时，使用 platform 设备的最终名称作为资源名。
- 资源已有 `parent` 时，优先插入该指定的父资源。
- 内存资源没有 parent 时，以全局 `iomem_resource` 作为默认根资源。
- I/O 端口资源没有 parent 时，以全局 `ioport_resource` 作为默认根资源。
- 其他资源类型（例如 IRQ）不通过这段逻辑插入 `iomem_resource` 或 `ioport_resource`，但仍可作为 platform 设备资源供驱动查询。

#### 7.5.5.3 资源树的作用

Linux 用资源树记录地址和端口范围的占用关系，避免多个设备声明彼此重叠且互斥的资源区域。每个资源节点都可拥有父资源，从而形成从全局根资源到具体设备资源的层级。

```text
iomem_resource
    └── <platform-device>
        └── 寄存器地址范围

ioport_resource
    └── <platform-device>
        └── I/O 端口范围
```

`insert_resource()` 会检查当前资源与父资源下已有子资源的范围关系。资源冲突或层级不合法时，函数返回相应负错误码；调用者应保留原始错误码并进入失败回滚，不能假定所有冲突都固定为 `-EBUSY`。

### 7.5.6 调用 `device_add` 与错误回滚

#### 7.5.6.1 接入通用设备模型

完成 parent、bus、名称和资源设置后，`platform_device_add()` 调用：

```c
ret = device_add(&pdev->dev);
if (ret)
    goto failed;
```

这一步复用第七章的通用 device 添加路径：

- 在 `/sys/devices/platform/` 或调用者指定的 parent 下创建实际设备目录。
- 在 `/sys/bus/platform/devices/` 创建指向实际目录的软链接。
- 将设备加入 `platform_bus_type.p->klist_devices`。
- 调用 `device_attach()`，按 `platform_match()` 尝试绑定 platform 驱动。
- 匹配并 probe 成功后，在驱动目录中建立设备软链接。

#### 7.5.6.2 失败路径

若 ID 自动分配、资源登记或 `device_add()` 失败，`platform_device_add()` 需要撤销已经完成的 platform 专属工作：

```c
failed:
    if (pdev->id_auto) {
        ida_free(&platform_devid_ida, pdev->id);
        pdev->id = PLATFORM_DEVID_AUTO;
        pdev->id_auto = false;
    }

    while (i--) {
        struct resource *r = &pdev->resource[i];

        if (r->parent)
            release_resource(r);
    }

    return ret;
```

回滚顺序体现以下原则：

1. 先归还自动分配的 ID。
2. 再按资源插入的反向顺序将资源从资源树移除。
3. 保留原始错误码返回给调用者。

`platform_device_register()` 的调用者在注册失败后还需要通过 `put_device(&pdev->dev)` 完成初始化阶段取得的引用回收；使用 `platform_device_register_simple()` 等辅助接口时，接口会按其约定处理失败路径。

### 7.5.7 platform 设备的 sysfs 与匹配关系

一个未显式设置 parent 的 platform 设备，例如名称为 `demo`、ID 为 `0`，注册成功后通常具有以下视图：

```text
/sys/devices/platform/demo.0/

/sys/bus/platform/devices/demo.0
    └── -> ../../../devices/platform/demo.0
```

当存在匹配成功的 platform 驱动后，还会出现：

```text
/sys/bus/platform/drivers/<driver-name>/demo.0
    └── -> ../../../../devices/platform/demo.0
```

设备层级、总线索引和驱动绑定索引指向同一个 `struct device`，只是分别表达 parent 关系、bus 归属和驱动绑定关系。

## 7.6 小结

本章从通用 `struct device` 到 `platform_device` 梳理了设备注册的完整路径：

- `device_register()` 由 `device_initialize()` 与 `device_add()` 两个阶段组成。前者初始化 kobject、锁、devres、电源管理、MSI 和 device link 等成员，后者创建 sysfs 目录、属性、bus/class 软链接并触发驱动匹配。
- 设置 `dev->bus = &my_bus_type` 后，通用 device 会进入自定义总线的匹配域；设备注销和错误回滚必须撤销 sysfs、总线和引用关系。
- `platform_device_register()` 复用通用 device 注册模型，先初始化 `pdev->dev`，再调用 `platform_device_add()` 补充 platform 专属配置。
- 未设置 parent 的 platform 设备自动以 `platform_bus` 为父对象，目录通常位于 `/sys/devices/platform/`。
- 每个 platform 设备都会设置 `dev->bus = &platform_bus_type`，从而接入 platform 驱动匹配域。
- `pdev->id` 决定设备名称后缀；自动 ID 使用 IDA 分配并以 `.auto` 避免名称冲突。
- 内存和 I/O 端口资源会插入 `iomem_resource`、`ioport_resource` 等资源树，失败时按逆序释放。
- `device_add()` 最终创建 sysfs 视图、加入总线设备链表并触发 `platform_match()` 和 probe。

# 8 驱动注册到自定义总线

设备注册完成后，驱动通过 `struct device_driver` 注册到同一条总线。设备模型会将驱动加入总线的驱动集合、创建驱动 sysfs 目录，并在自动探测开启时遍历总线上的未绑定设备，执行 match、probe 和绑定流程。

本章继续使用第六章定义的 `my_bus_type` 与第七章注册的 `my_device_in_bus`。案例中设备名和驱动名都为 `my_device_in_bus`，因此 `my_bus_type.match()` 按名称比较时会返回匹配成功。

## 8.1 注册到 `my_bus_type` 的驱动案例

### 8.1.1 `struct device_driver` 的关键成员

```c
struct device_driver my_driver_in_bus = {
    .name = "my_device_in_bus",
    .bus = &my_bus_type,
    .probe = mydriver_probe,
    .remove = mydriver_remove,
};
```

成员说明：

- `name`：驱动名。它是 sysfs 驱动目录名，也可被 `my_bus_type.match()` 用作名称匹配条件。
- `bus`：驱动所属总线，必须指向已经注册的 `my_bus_type`。
- `probe`：设备与驱动匹配成功后的初始化回调。
- `remove`：设备解绑或驱动注销时的清理回调。

### 8.1.2 完整模块代码

```c
#include <linux/device.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>

extern struct bus_type my_bus_type;

static int mydriver_probe(struct device *dev)
{
    pr_info("probe my_driver_in_bus for %s\n", dev_name(dev));
    return 0;
}

static void mydriver_remove(struct device *dev)
{
    pr_info("remove my_driver_in_bus for %s\n", dev_name(dev));
}

static struct device_driver my_driver_in_bus = {
    .name = "my_device_in_bus",
    .bus = &my_bus_type,
    .probe = mydriver_probe,
    .remove = mydriver_remove,
};

static int __init mydriverinbus_init(void)
{
    return driver_register(&my_driver_in_bus);
}

static void __exit mydriverinbus_exit(void)
{
    driver_unregister(&my_driver_in_bus);
}

module_init(mydriverinbus_init);
module_exit(mydriverinbus_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_DESCRIPTION("driver registration on a custom bus");
MODULE_VERSION("1.0");
```

现代内核中 `struct device_driver.remove` 通常为 `void` 返回类型。较旧内核版本可能使用返回 `int` 的 remove 回调，编写模块时应以目标内核头文件原型为准。

模块加载依赖关系如下：

```text
mybus.ko
    ├── 注册 my_bus_type
    ├── device.ko 注册 my_device_in_bus
    └── driver.ko 注册 my_driver_in_bus
```

设备模块与驱动模块的先后顺序不影响最终匹配，但两者都依赖总线模块已先成功注册。

## 8.2 `driver_register` 的注册过程

### 8.2.1 函数原型与调用链

```c
int driver_register(struct device_driver *drv);
```

`driver_register()` 的典型主链路如下：

```text
driver_register(drv)
    ├── driver_find(drv->name, drv->bus)
    ├── driver_initialize(drv)
    │   └── 初始化 driver_private、kobject 和设备链表
    └── driver_add(drv)
        ├── 将驱动加入总线驱动集合
        ├── 创建 sysfs 驱动目录和默认属性
        └── driver_attach(drv)
            └── 遍历总线设备并尝试自动匹配
```

注册前，`driver_register()` 会检查驱动是否指定总线以及同一总线下是否存在同名驱动。若名称冲突或总线无效，注册失败。

### 8.2.2 `driver_private` 与 `drv->p`

`struct device_driver` 的公开定义中包含内部私有指针：

```c
struct device_driver {
    const char *name;
    struct bus_type *bus;
    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);
    struct driver_private *p;
};
```

驱动注册时，内核为驱动分配 `struct driver_private`，并建立双向关联：

```c
priv->driver = drv;
drv->p = priv;
```

这两条赋值分别解决两个方向的访问需求：

- `drv->p`：从公开 `device_driver` 找到设备模型维护的驱动私有状态。
- `priv->driver`：从私有状态回到所属的公开 `device_driver`。

`driver_private` 的典型关键成员如下：

```c
struct driver_private {
    struct kobject kobj;
    struct klist klist_devices;
    struct klist_node knode_bus;
    struct module_kobject *mkobj;
    struct device_driver *driver;
};
```

成员说明：

- `kobj`：驱动在设备模型和 sysfs 中对应的 kobject。
- `klist_devices`：该驱动已绑定设备的内部链表。
- `knode_bus`：将当前驱动接入所属总线 `klist_drivers` 的链表节点。
- `driver`：回指公开的 `struct device_driver`。

`drv->p` 是设备核心维护的内部状态。驱动实现通常不应直接修改其中成员，而应使用设备模型提供的注册、解绑和遍历 API。

### 8.2.3 驱动 sysfs 目录

`driver_initialize()` 会初始化私有 kobject，随后 `driver_add()` 通过以下路径把它加入 sysfs：

```c
kobject_init_and_add(&priv->kobj, &driver_ktype, NULL, "%s", drv->name);
```

驱动 kobject 以总线的 drivers kset 为父级语义对象，注册成功后通常形成：

```text
/sys/bus/mybus/drivers/my_device_in_bus/
```

该目录可包含驱动默认属性、`bind`/`unbind` 等总线框架相关属性，以及后续成功绑定设备的符号链接。

## 8.3 驱动后注册时的自动匹配

### 8.3.1 `driver_attach` 遍历设备

当总线开启自动探测时，`driver_add()` 会执行：

```c
if (sp->drivers_autoprobe) {
    error = driver_attach(drv);
    if (error)
        goto out_del_list;
}
```

这里的 `sp` 是 `drv->bus->p` 对应的 `subsys_private`。`drivers_autoprobe` 控制驱动注册时是否自动遍历该总线上的设备；正常总线通常保持该选项开启。

调用关系如下：

```text
driver_attach(drv)
    └── bus_for_each_dev(drv->bus, NULL, drv, __driver_attach)
        └── 对总线中的每个 device 执行 __driver_attach()
            └── driver_match_device(drv, dev)
                └── drv->bus->match(dev, drv)
            └── device_driver_attach(drv, dev)
                └── driver_probe_device(drv, dev)
```

### 8.3.2 `driver_match_device`

`driver_match_device()` 统一调用总线的匹配回调：

```c
static inline int driver_match_device(const struct device_driver *drv,
                                      struct device *dev)
{
    return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}
```

逻辑含义：

- 总线实现了 `match` 时，调用 `match(dev, drv)` 决定是否匹配。
- 总线没有实现 `match` 时，默认返回 `1`，即允许进入后续 probe 路径。

对于 `my_bus_type`，`match` 使用 `!strcmp(dev_name(dev), drv->name)`。设备名和驱动名相等时返回 1，因此 `my_device_in_bus` 可与 `my_driver_in_bus` 匹配。

### 8.3.3 `driver_probe_device` 与 `really_probe`

匹配成功后，`device_driver_attach()` 会进入 probe 路径。不同内核版本的中间辅助函数名称可能略有调整，典型流程如下：

```text
device_driver_attach(drv, dev)
    └── driver_probe_device(drv, dev)
        └── __driver_probe_device(drv, dev)
            └── really_probe(dev, drv)
                └── call_driver_probe(dev, drv)
```

`really_probe()` 负责处理 probe 前后的设备状态、deferred probe、probe 计数和绑定成功后的收尾工作。真正调用 probe 回调的关键分支可以概括为：

```c
if (dev->bus->probe)
    ret = dev->bus->probe(dev);
else if (drv->probe)
    ret = drv->probe(dev);
```

这说明总线级 `probe` 优先于驱动级 `probe`：

- 当 `dev->bus->probe` 存在时，由总线 probe 负责处理；总线实现通常会在其内部转换对象并调用相应驱动的 probe。
- 当总线没有提供 probe 时，设备核心直接调用 `drv->probe(dev)`。

自定义 `my_bus_type` 若未定义 `.probe`，本案例最终会直接调用 `mydriver_probe()`。若总线定义了包装 probe，则包装函数必须返回实际驱动 probe 的返回值，不能无条件返回成功。

### 8.3.4 绑定成功后的状态

probe 返回 0 后，`really_probe()` 会完成绑定收尾，并通过 `driver_bound()` 等路径建立设备与驱动的关联：

- `dev->driver` 指向已成功绑定的 `my_driver_in_bus`。
- 设备被加入 `drv->p->klist_devices`。
- sysfs 在 `/sys/bus/mybus/drivers/my_device_in_bus/` 下创建指向设备实际目录的软链接。
- 设备目录中建立指向驱动目录的 `driver` 软链接。

若 probe 返回负错误码，设备不会完成本次绑定。`-EPROBE_DEFER` 表示依赖尚未就绪，设备模型会在后续适当时机重新尝试 probe。

## 8.4 设备先注册时的匹配路径

设备与驱动注册顺序不影响最终匹配，是因为设备模型在两侧注册路径上都安排了匹配入口。

当设备先于驱动注册时，第七章的 `device_add()` 会调用 `bus_probe_device()`。典型调用关系如下：

```text
device_add(dev)
    └── bus_probe_device(dev)
        └── device_initial_probe(dev)
            └── device_attach(dev)
                └── bus_for_each_drv(dev->bus, NULL, dev,
                                     __device_attach)
                    └── 对总线中的每个 driver 执行 __device_attach()
                        └── driver_match_device(drv, dev)
                        └── device_driver_attach(drv, dev)
                            └── driver_probe_device(drv, dev)
```

设备先注册时的绑定处理可分为两种状态：

- `dev->driver` 已存在：设备模型检查设备是否已绑定；若尚未绑定，则尝试 `device_bind_driver()`，将设备加入该驱动的设备链表。
- `dev->driver` 为空：通过 `bus_for_each_drv()` 遍历该总线的驱动链表，对每个候选驱动执行 match 和 probe。

probe 成功后，`really_probe()` 调用 `driver_bound()` 完成最终绑定。因此无论设备先出现还是驱动先出现，另一方注册时都会遍历总线中已存在的对象，最终执行同一套 match、probe 和绑定逻辑。

## 8.5 驱动注销与 `remove`

### 8.5.1 `driver_unregister`

```c
void driver_unregister(struct device_driver *drv);
```

`driver_unregister()` 撤销驱动注册。设备模型会从总线驱动链表和 sysfs 驱动目录中移除该驱动，并对该驱动已绑定的设备执行解绑路径。

### 8.5.2 `remove` 的执行

驱动注销或设备移除时，设备与驱动的绑定被解除。若总线提供 `remove`，设备核心优先调用总线级 remove；否则调用驱动的 remove：

```c
if (dev->bus->remove)
    dev->bus->remove(dev);
else if (drv->remove)
    drv->remove(dev);
```

本案例的 `my_bus_type` 未实现总线级 `.remove`，因此解绑时会调用 `mydriver_remove()`。remove 应释放 probe 中申请的资源、停止异步执行路径，并保证不再访问即将解绑的设备状态。

## 8.6 小结

驱动注册完成了总线驱动集合、sysfs 驱动目录和设备自动匹配三个层面的接入：

- `driver_register()` 初始化 `driver_private`，通过 `priv->driver = drv` 与 `drv->p = priv` 建立公开驱动对象和私有状态的双向关联。
- `driver_add()` 将驱动加入总线驱动链表，创建 `/sys/bus/<bus>/drivers/<driver>/` 目录，并在自动探测开启时调用 `driver_attach()` 遍历设备。
- `driver_match_device()` 在总线实现 match 时调用 `match(dev, drv)`；没有 match 时默认允许匹配。
- match 成功后经 `device_driver_attach()`、`driver_probe_device()`、`really_probe()` 进入 probe。总线级 probe 优先于驱动级 probe。
- 驱动先注册时遍历设备，设备先注册时遍历驱动，因此注册顺序不会阻止最终匹配。
- probe 成功后，设备加入驱动的绑定设备链表并创建 sysfs 双向软链接；驱动注销或设备移除时执行 remove 和解绑清理。
