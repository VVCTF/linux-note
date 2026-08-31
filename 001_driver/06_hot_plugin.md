# 1 热插拔机制概述

热插拔（Hotplug）指设备在系统运行期间动态出现或消失时，内核与用户空间能够及时感知并完成相应处理的一整套机制。典型场景包括插入 U 盘后自动创建设备节点、加载或卸载内核模块时用户态守护进程同步更新设备信息、网卡状态变化时触发配置脚本等。

热插拔要解决的核心问题不是“设备如何被内核识别”，而是“内核识别到设备状态变化后，如何把这一事件通知给用户空间，并由用户空间完成节点创建、权限设置、脚本调用等后续动作”。这一整套通知与响应链路，就是热插拔机制。

## 1.1 什么是热插拔

热插拔机制建立在设备模型之上：设备、总线、驱动的注册与绑定发生在内核态，而设备节点的创建、外部脚本的执行通常需要在用户态完成，因为用户态更适合承载策略性、可配置的逻辑（例如按厂商 ID 加载特定驱动、按设备类型设置不同权限）。内核只负责机制，不负责策略；热插拔正是这一设计原则的具体体现。

热插拔机制的基本结构可以概括为：

```text
内核态：设备状态变化（添加/删除/修改）
    └── 生成事件（uevent）
        └── 发送到用户空间
            └── 用户态程序接收事件并执行相应动作
```

内核只负责在合适的时机“广播”一个事件，事件内容说明发生了什么变化、涉及哪个对象；具体如何响应完全交由用户空间决定。

## 1.2 热插拔事件从内核到用户空间的路径

`05_device_model.md` 介绍的设备模型中，`kobject` 是内核对象在 sysfs 中的最小管理单元。热插拔机制正是围绕 `kobject` 展开：当一个 `kobject` 被添加、移除或状态发生变化时，内核可以为其生成一个 uevent（用户空间事件），并通过某种通道把这个事件发送出去。

用户空间存在多种不同的事件接收方式，其中最常见的是 udev 和 mdev。二者接收事件的通道不同，但都以“内核产生 uevent，用户态程序据此执行热插拔动作”为共同模型。

## 1.3 udev：基于 netlink 的热插拔机制

### 1.3.1 netlink 事件通道

udev 基于 netlink 机制实现。netlink 是一种用于内核与用户空间之间双向通信的套接字机制，支持多播（multicast），因此内核可以把一个事件广播给所有对该事件感兴趣的用户态进程，而不需要事先知道有哪些进程在监听。

udev 在用户态创建一个 netlink 套接字，并加入内核 uevent 使用的多播组。内核侧每当某个 `kobject` 需要上报状态变化，就通过该多播组发送一条 uevent 消息。

### 1.3.2 udev 的处理流程

udev 收到 uevent 消息后，并不是简单地转发，而是执行一套规则匹配与处理逻辑：

```text
内核生成 uevent
    └── 通过 netlink 多播发送
        └── udev 进程接收该消息
            └── 按规则文件（/etc/udev/rules.d 等）匹配
                └── 创建/删除设备节点、设置权限、执行自定义脚本
```

核心作用：

- 监听内核通过 netlink 发送的 uevent，无需内核额外调用外部程序
- 支持规则化配置，能够按设备属性（如厂商 ID、设备类型）执行不同处理
- 是目前主流发行版（依赖 systemd 的系统通常使用其内建的 udev 实现）采用的热插拔方案

## 1.4 mdev：基于 uevent_helper 的热插拔机制

### 1.4.1 uevent_helper 机制

mdev 常见于 BusyBox 等面向嵌入式系统的精简环境，它不依赖 netlink 多播，而是基于内核提供的 `uevent_helper` 机制。`uevent_helper` 是内核维护的一个路径字符串，对应 `/proc/sys/kernel/hotplug` 节点。

当内核产生一个 uevent 时，如果 `uevent_helper` 被设置为某个可执行程序的路径，内核会调用该路径指向的用户态程序，并把事件相关信息通过环境变量传递给它。该用户态程序通常就是 mdev。

### 1.4.2 mdev 的处理流程

```text
内核生成 uevent
    └── uevent_helper 指向的路径被调用（如 /sbin/mdev）
        └── mdev 进程启动，读取环境变量中的事件信息
            └── 按 /etc/mdev.conf 规则创建/删除设备节点、执行脚本
```

核心作用：

- 不依赖 netlink 多播，实现更轻量，适合资源受限的嵌入式系统
- 内核每次产生 uevent 都需要 fork/exec 一次 mdev 进程，开销相对更高
- 配置文件通常为 `/etc/mdev.conf`，规则形式比 udev 简单

## 1.5 udev 与 mdev 的对比

| 对比项    | udev                        | mdev                    |
| ------ | --------------------------- | ----------------------- |
| 通信机制   | netlink 多播套接字               | `uevent_helper` 触发的进程调用 |
| 触发方式   | udev 进程常驻，监听事件              | 内核每次事件都调用一次外部程序         |
| 典型使用场景 | 桌面/服务器发行版                   | BusyBox 等嵌入式精简系统        |
| 配置文件   | `/etc/udev/rules.d/*.rules` | `/etc/mdev.conf`        |
| 开销特点   | 常驻进程，事件处理延迟低                | 每次事件都有进程创建开销            |

两者的共同前提相同：内核侧必须先把“对象状态发生了变化”这件事变成一个 uevent 并发送出去，用户态才有机会响应。第二章从内核侧出发，说明内核如何生成并发送这个事件。

## 1.6 小结

- 热插拔机制解决“内核检测到设备变化后如何通知用户空间”的问题，内核只提供机制，具体处理策略放在用户态。
- udev 基于 netlink 多播机制，监听内核发送的 uevent 并按规则执行处理，是目前主流的实现方式。
- mdev 基于 `uevent_helper` 机制，内核在产生 uevent 时直接调用该变量指向的用户态程序完成处理，实现更轻量，常见于嵌入式系统。
- 无论 udev 还是 mdev，都依赖内核主动生成并发送 uevent，这是热插拔链路的起点。

# 2 内核向用户空间发送 uevent

内核态生成 uevent 并将其发送到用户空间，是热插拔链路中真正的起点。内核提供了统一的 API 完成这一动作，使驱动或内核模块可以在需要的时机主动通知用户空间某个对象的状态变化。

## 2.1 `kobject_uevent`

### 2.1.1 函数原型

```c
int kobject_uevent(struct kobject *kobj, enum kobject_action action);
```

`kobject_uevent()` 以指定的 `action` 为一个已经注册的 `kobject` 生成并发送一条 uevent 消息。该消息既会通过 netlink 多播发送给 udev 一类的监听者，也会在配置了 `uevent_helper` 时触发对应的用户态程序（如 mdev）。

### 2.1.2 参数说明

- `kobj`：要上报事件的 `kobject`。该对象必须已经通过 `kobject_add()`、`kobject_init_and_add()` 等接口成功接入设备模型，否则无法确定其在 sysfs 中的路径，事件也就缺少定位信息。
- `action`：本次事件的动作类型，说明该对象发生了什么变化。

### 2.1.3 返回值说明

- `0`：事件发送成功。
- `< 0`：发送失败，返回负错误码。

### 2.1.4 `action` 参数详解

`action` 的类型是 `enum kobject_action`，定义于 `include/linux/kobject.h`，用于描述本次上报的具体动作：

```c
enum kobject_action {
    KOBJ_ADD,
    KOBJ_REMOVE,
    KOBJ_CHANGE,
    KOBJ_MOVE,
    KOBJ_ONLINE,
    KOBJ_OFFLINE,
    KOBJ_BIND,
    KOBJ_UNBIND,
};
```

各取值含义：

- `KOBJ_ADD`：对象被添加，对应设备出现、模块加载等场景。`kobject_add()`/`device_add()` 等接口在内部成功路径上通常会自动上报该事件。
- `KOBJ_REMOVE`：对象被移除，对应设备拔出、模块卸载等场景。
- `KOBJ_CHANGE`：对象状态发生变化，但对象本身既未新增也未移除。适合驱动主动通知“某个属性值变了”，例如本章示例中的用法。
- `KOBJ_MOVE`：对象在设备模型中的位置发生变化，例如设备被移动到不同的 parent 下，或者设备名被重新命名。
- `KOBJ_ONLINE` / `KOBJ_OFFLINE`：对象上线或下线，常见于支持在线/离线切换的设备。
- `KOBJ_BIND` / `KOBJ_UNBIND`：对象与驱动完成绑定或解除绑定。

用户空间接收到 uevent 后，会在事件内容中看到与 `action` 对应的动作字符串（如 `add`、`remove`、`change`），udev 规则和 mdev 配置都可以据此过滤和区分不同类型的事件。

## 2.2 案例：手动触发 `KOBJ_CHANGE` 事件

### 2.2.1 案例逻辑

以下模块创建一个归属于自定义 `kset` 的 `kobject`，并在模块加载时主动调用 `kobject_uevent()` 上报一次 `KOBJ_CHANGE` 事件，用于观察内核如何把该事件发送到用户空间。

### 2.2.2 完整模块代码

```c
#include <linux/kernel.h>
#include <linux/kobject.h>

struct kobject *mykoject01;
struct kset *mykset;
struct kobj_type mytype;

static int mykobj_init(void)
{
    int ret;

    mykset = kset_create_and_add("mykset", NULL, NULL);

    mykoject01 = kzalloc(sizeof(struct kobject), GFP_KERNEL);
    mykoject01->kset = mykset;
    ret = kobject_init_and_add(mykoject01, &mytype, NULL, "%s", "mykoject01");

    ret = kobject_uevent(mykoject01, KOBJ_CHANGE);

    return 0;
}

static void mykobj_exit(void)
{
    kobject_put(mykoject01);
    kset_unregister(mykset);
}

module_init(mykobj_init);
module_exit(mykobj_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fashi");
MODULE_VERSION("V1.0");
```

代码逻辑：

1. `kset_create_and_add()` 创建集合 `mykset`，作为 `mykoject01` 的所属集合。
2. `mykoject01->kset = mykset` 指定该 kobject 归属的集合，使其对应的 uevent 消息会带上 `mykset` 相关的路径信息。
3. `kobject_init_and_add()` 把 `mykoject01` 接入设备模型并创建对应的 sysfs 目录 `/mykset/mykoject01`。
4. `kobject_uevent(mykoject01, KOBJ_CHANGE)` 以 `KOBJ_CHANGE` 动作主动上报一次事件，用户空间会看到一条 `change` 消息。

案例仅用于演示 `kobject_uevent()` 的调用位置和效果，`mytype` 未设置 `.release` 回调。按照 `05_device_model.md` 第 2 章的注册规范，正式驱动中的 `kobj_type` 必须提供有效的 `release`，否则 `kobject_put()` 触发引用计数归零时无法完成正确的资源释放。

## 2.3 使用 `udevadm monitor` 观察热插拔事件

### 2.3.1 `udevadm monitor` 的作用

`udevadm monitor` 是用户空间提供的调试工具，用于实时打印 udev 接收到的事件。它可以同时显示两类消息：

- `KERNEL` 前缀：内核通过 netlink 直接发出的原始 uevent，尚未经过 udev 规则处理。
- `UDEV` 前缀：udev 完成规则匹配和处理后再次发出的事件，供其他监听 udev 事件的用户态程序使用。

由于内核发送 uevent 是一次性动作，若驱动先加载完毕才启动监听工具，之前发出的事件将无法被捕获。因此观察热插拔事件时，必须先启动监听，再触发内核发出事件。

### 2.3.2 观察步骤

1. 启动监听，并放入后台执行，使终端可以继续输入后续命令：

```sh
udevadm monitor &
```

2. 加载包含 `kobject_uevent()` 调用的驱动模块：

```sh
insmod uevent.ko
```

3. 观察终端输出的 `KERNEL` 与 `UDEV` 事件行。

### 2.3.3 输出结果分析

实际执行结果如下：

```text
[root@RK356X:/]# udevadm monitor &
[root@RK356X:/]# monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

[root@RK356X:/]#
[root@RK356X:/]# insmod uevent.ko
[  182.874045] uevent: loading out-of-tree module taints kernel.
KERNEL[182.289412] change   /mykset/mykoject01 (mykset)
[root@RK356X:/]# KERNEL[182.295003] add      /module/uevent (module)
UDEV  [182.299235] change   /mykset/mykoject01 (mykset)
UDEV  [182.304163] add      /module/uevent (module)
```

结果解读：

- `KERNEL[...] change /mykset/mykoject01 (mykset)`：对应案例代码中 `kobject_uevent(mykoject01, KOBJ_CHANGE)` 触发的原始内核事件。路径 `/mykset/mykoject01` 与 `mykoject01->kset = mykset` 时形成的 sysfs 层级一致，括号中的 `mykset` 表示该 kobject 所属的集合。
- `KERNEL[...] add /module/uevent (module)`：`insmod` 加载模块时，内核模块子系统自身也会为该模块对应的 `kobject` 发出一次 `add` 事件，用于通知用户空间有新模块被加载，这与驱动代码中的调用无关。
- `UDEV [...] change ...` 与 `UDEV [...] add ...`：udev 接收到上述两条内核事件后，完成规则处理并各自重新发出一条对应的 `UDEV` 事件，供其他监听 udev 输出的用户态组件使用。

`KERNEL` 事件与 `UDEV` 事件的时间戳非常接近但不完全相同，说明二者不是同一条消息：前者是内核发出的原始事件，后者是 udev 处理完成后的再次上报，这正是 1.3 节中 udev 处理流程在实际系统中的体现。

## 2.4 小结

- `kobject_uevent()` 是内核主动上报热插拔事件的核心 API，前提是目标 `kobject` 已经成功接入设备模型。
- `action` 参数使用 `enum kobject_action` 枚举，`KOBJ_ADD`、`KOBJ_REMOVE`、`KOBJ_CHANGE` 等取值分别对应不同的对象状态变化，用户空间据此区分事件类型。
- 观察内核发出的 uevent 应遵循“先监听、后触发”的顺序：先执行 `udevadm monitor &` 启动后台监听，再加载驱动模块，避免错过一次性发出的事件。
- `udevadm monitor` 的输出同时包含 `KERNEL` 原始事件和 `UDEV` 处理后事件，二者路径信息一致但分别代表热插拔链路中的不同阶段。

# 3 `kobject_uevent` 内部实现分析

第二章的案例中，`mykoject01` 在调用 `kobject_init_and_add()` 之前先被设置了 `mykoject01->kset = mykset`。这一步不是可有可无的装饰：如果去掉这一行，`kobject_uevent()` 在 `udevadm monitor` 中将完全看不到对应事件。原因需要深入 `kobject_uevent()` 的实现才能说清楚，本章从源码角度逐段分析。

## 3.1 `kobject_uevent` 与 `kobject_uevent_env`

`kobject_uevent()` 本身只是一层薄封装，真正的逻辑在 `kobject_uevent_env()` 中完成：

```c
int kobject_uevent(struct kobject *kobj, enum kobject_action action)
{
    return kobject_uevent_env(kobj, action, NULL);
}
```

`kobject_uevent_env()` 比 `kobject_uevent()` 多一个 `envp_ext` 参数，用于让调用者附加额外的环境变量。第二章调用的 `kobject_uevent(kobj, action)` 等价于 `kobject_uevent_env(kobj, action, NULL)`，即不附加任何额外环境变量。

`kobject_uevent_env()` 的函数原型：

```c
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
                       char *envp_ext[]);
```

参数说明：

- `kobj`：事件所针对的 `kobject`。
- `action`：事件动作类型，即 `enum kobject_action`。
- `envp_ext`：调用者附加的环境变量数组，以 `NULL` 结尾；不需要附加时传 `NULL`。

## 3.2 关键局部变量

```c
struct kobj_uevent_env *env;
const char *action_string = kobject_actions[action];
const char *devpath = NULL;
const char *subsystem;
struct kobject *top_kobj;
struct kset *kset;
const struct kset_uevent_ops *uevent_ops;
int i = 0;
int retval = 0;
```

变量职责：

- `env`：指向 `struct kobj_uevent_env` 的指针，用于保存本次事件携带的全部环境变量（`ACTION`、`DEVPATH`、`SUBSYSTEM` 等）。
- `action_string`：把枚举值 `action` 转换成对应的字符串，例如 `KOBJ_CHANGE` 对应 `"change"`，这正是 `udevadm monitor` 中看到的动作文本。
- `devpath`：该 `kobject` 在 sysfs 中的完整路径，稍后通过 `kobject_get_path()` 生成。
- `subsystem`：事件所属子系统的名称，用于填充 `SUBSYSTEM` 环境变量。
- `top_kobj`：向上查找 kset 过程中使用的临时指针。
- `kset`：`kobj` 最终归属的集合。
- `uevent_ops`：该 `kset` 的事件操作集，决定是否过滤事件、如何命名子系统、如何补充自定义环境变量。
- `i`、`retval`：分别用于遍历 `envp_ext` 数组和记录函数执行结果。

## 3.3 标记 "remove" 事件已发出

```c
if (action == KOBJ_REMOVE)
    kobj->state_remove_uevent_sent = 1;
```

无论后续流程是否成功发送事件，只要 `action` 是 `KOBJ_REMOVE`，就先把 `state_remove_uevent_sent` 置 1。这是为了避免设备模型在对象自动清理阶段重复补发一次 "remove" 事件——`05_device_model.md` 第 1.2.1 节介绍过 `kobject` 用若干 `state_*` 位记录事件发送状态，这里正是其中一处使用场景。

## 3.4 沿父链查找所属 kset

```c
top_kobj = kobj;
while (!top_kobj->kset && top_kobj->parent)
    top_kobj = top_kobj->parent;

if (!top_kobj->kset) {
    pr_debug("kobject: '%s' (%p): %s: attempted to send uevent "
             "without kset!\n", kobject_name(kobj), kobj, __func__);
    return -EINVAL;
}

kset = top_kobj->kset;
uevent_ops = kset->uevent_ops;
```

这段代码从当前 `kobj` 出发，沿 `parent` 指针向上查找，直到遇到某个对象自身设置了 `kset`，或者已经走到没有 parent 的根节点为止：

- 若某一层对象的 `kset` 非空，`top_kobj` 就停在这一层，`kset = top_kobj->kset` 即为找到的集合。
- 若一路向上都没有任何对象设置 `kset`，循环会在 `top_kobj->parent` 为空时终止，此时 `top_kobj->kset` 仍为空，函数直接返回 `-EINVAL`，不再继续执行后面任何发送逻辑。

`kobject` 本身既可以直接设置 `kset`，也可以不设置而依赖某个祖先对象的 `kset`。`kobject_uevent_env()` 允许这种“子对象没有直接 kset，但父对象有”的场景，因为它顺着 parent 链查找而非只检查 `kobj` 自身。

这里给出了第二章问题的直接答案：`kobject_uevent()` 发送事件的前提，是 `kobj` 或其某个祖先对象必须挂在一个 `kset` 下。`kset` 承载了 `uevent_ops`，而后续的子系统命名、事件过滤、事件内容补充都依赖这个 `uevent_ops`。如果第二章的案例去掉 `mykoject01->kset = mykset` 这一行，`mykoject01` 没有 parent、也没有自身的 `kset`，`top_kobj->kset` 恒为空，`kobject_uevent()` 会在这一步直接返回 `-EINVAL`，函数尚未走到发送环节就已经结束，`udevadm monitor` 自然什么都看不到。

## 3.5 事件抑制与过滤

```c
if (kobj->uevent_suppress) {
    pr_debug("kobject: '%s' (%p): %s: uevent_suppress "
             "caused the event to drop!\n",
             kobject_name(kobj), kobj, __func__);
    return 0;
}

if (uevent_ops && uevent_ops->filter)
    if (!uevent_ops->filter(kobj)) {
        pr_debug("kobject: '%s' (%p): %s: filter function "
                 "caused the event to drop!\n",
                 kobject_name(kobj), kobj, __func__);
        return 0;
    }
```

找到 `kset` 之后，函数还提供两层丢弃事件的机制：

- `kobj->uevent_suppress` 为真时直接丢弃事件，返回 `0`（不是错误，而是"本次不发送"）。`05_device_model.md` 第 1.2.1 节 `kobject` 结构体中的 `uevent_suppress:1` 位域正是为此设计。
- 若 `kset` 定义了 `uevent_ops->filter`，且该回调对当前 `kobj` 返回 `0`，同样丢弃事件。这使得某个 `kset` 可以按自定义规则（例如设备是否已完全初始化）统一决定其成员是否上报事件。

两种丢弃都返回 `0`，调用者看到的是"调用成功"，但事件实际没有被发送。

## 3.6 确定 subsystem 名称

```c
if (uevent_ops && uevent_ops->name)
    subsystem = uevent_ops->name(kobj);
else
    subsystem = kobject_name(&kset->kobj);

if (!subsystem) {
    pr_debug("kobject: '%s' (%p): %s: unset subsystem caused the "
             "event to drop!\n", kobject_name(kobj), kobj, __func__);
    return 0;
}
```

`subsystem` 决定了事件中 `SUBSYSTEM=` 环境变量的值：

- 若 `uevent_ops` 提供了 `name` 回调，优先调用它取得自定义子系统名。
- 否则退回使用 `kset` 自身 kobject 的名字（`kset->kobj` 的 `name`）。第二章案例中 `mykset` 没有自定义 `uevent_ops`，因此 `subsystem` 就是 `"mykset"`。
- 若两种方式都取不到名字，同样丢弃事件并返回 `0`。

## 3.7 分配环境变量缓冲区与设备路径

```c
env = kzalloc(sizeof(struct kobj_uevent_env), GFP_KERNEL);
if (!env)
    return -ENOMEM;

devpath = kobject_get_path(kobj, GFP_KERNEL);
if (!devpath) {
    retval = -ENOENT;
    goto exit;
}
```

`env` 用于收集本次事件的所有环境变量（键值对形式的字符串数组）。`kobject_get_path()` 根据 `kobj` 在设备模型中的层级关系，动态生成其对应的完整路径字符串，也就是 `udevadm monitor` 输出中看到的 `/mykset/mykoject01` 这一部分。

## 3.8 写入默认环境变量

```c
retval = add_uevent_var(env, "ACTION=%s", action_string);
if (retval)
    goto exit;
retval = add_uevent_var(env, "DEVPATH=%s", devpath);
if (retval)
    goto exit;
retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
if (retval)
    goto exit;

if (envp_ext) {
    for (i = 0; envp_ext[i]; i++) {
        retval = add_uevent_var(env, "%s", envp_ext[i]);
        if (retval)
            goto exit;
    }
}
```

`add_uevent_var()` 向 `env` 追加一条形如 `KEY=VALUE` 的字符串。每一次 `kobject_uevent_env()` 调用都会至少产生三个默认环境变量：

- `ACTION`：即 `action_string`，例如 `"change"`。
- `DEVPATH`：即上一步生成的 `devpath`。
- `SUBSYSTEM`：即上一步确定的 `subsystem`。

若调用者通过 `envp_ext` 传入了额外环境变量（`kobject_uevent()` 传的是 `NULL`，因此这段循环在第二章案例中不会执行），会被逐条追加到 `env` 后面。

## 3.9 kset 自定义 uevent 回调

```c
if (uevent_ops && uevent_ops->uevent) {
    retval = uevent_ops->uevent(kobj, env);
    if (retval) {
        pr_debug("kobject: '%s' (%p): %s: uevent() returned %d\n",
                 kobject_name(kobj), kobj, __func__, retval);
        goto exit;
    }
}
```

在默认变量写入完成后，若 `kset` 提供了 `uevent_ops->uevent` 回调，函数会调用它，让该 `kset` 有机会向 `env` 中补充子系统特有的环境变量。例如 block、net 等子系统会借助该回调添加 `DEVNAME`、`MODALIAS` 等自身关心的字段。第二章案例中 `kset_create_and_add("mykset", NULL, NULL)` 传入的 `uevent_ops` 为 `NULL`，因此这一步不会追加任何额外变量。

## 3.10 action 相关的特殊处理

```c
switch (action) {
case KOBJ_ADD:
    kobj->state_add_uevent_sent = 1;
    break;

case KOBJ_UNBIND:
    zap_modalias_env(env);
    break;

default:
    break;
}
```

不同 `action` 会触发不同的附加动作：

| `action`      | 处理内容                                                                               |
| ------------- | ---------------------------------------------------------------------------------- |
| `KOBJ_ADD`    | 将 `state_add_uevent_sent` 置 1，使设备模型能在对象被移除时确认必须补发一次 "remove" 事件，保证 add/remove 成对出现 |
| `KOBJ_UNBIND` | 调用 `zap_modalias_env()` 清除 `env` 中已经写入的 `MODALIAS` 变量，避免解绑阶段仍然携带模块别名信息误导用户空间       |
| 其他            | 不做特殊处理                                                                             |

第二章案例使用的是 `KOBJ_CHANGE`，落入 `default` 分支，不会触发上述任何一种附加处理。

## 3.11 写入序列号

```c
retval = add_uevent_var(env, "SEQNUM=%llu",
                        atomic64_inc_return(&uevent_seqnum));
if (retval)
    goto exit;
```

每次即将真正发送事件前，都会追加一个 `SEQNUM` 环境变量，其值来自全局原子计数器 `uevent_seqnum` 的自增结果。`SEQNUM` 让用户空间可以判断多个事件的先后顺序，即使它们的时间戳非常接近（如第二章截图中 `KERNEL` 事件与 `UDEV` 事件时间戳相差极小的情况）。

## 3.12 事件发送的两条路径

写入 `SEQNUM` 之后，`env` 中已经包含 `ACTION`、`DEVPATH`、`SUBSYSTEM`、`SEQNUM` 及全部附加变量，函数随即执行真正的发送动作。这一步同时对应第一章介绍的两种热插拔实现方式。

### 3.12.1 netlink 广播：udev 的事件来源

```c
retval = kobject_uevent_net_broadcast(kobj, env, action_string, devpath);
```

`kobject_uevent_net_broadcast()` 把 `env` 中的全部环境变量通过 netlink 多播套接字发送出去。这正是第一章 1.3 节所述 udev 事件通道的内核侧实现：udev 常驻进程持有对应的 netlink 套接字，只要内核走到这一步，`udevadm monitor` 就能打印出一行 `KERNEL[...]` 消息。

### 3.12.2 `uevent_helper`：mdev 的事件来源

```c
#ifdef CONFIG_UEVENT_HELPER
    if (uevent_helper[0] && !kobj_usermode_filter(kobj)) {
        struct subprocess_info *info;

        retval = add_uevent_var(env, "HOME=/");
        if (retval)
            goto exit;
        retval = add_uevent_var(env, "PATH=/sbin:/bin:/usr/sbin:/usr/bin");
        if (retval)
            goto exit;
        retval = init_uevent_argv(env, subsystem);
        if (retval)
            goto exit;

        retval = -ENOMEM;
        info = call_usermodehelper_setup(env->argv[0], env->argv,
                                         env->envp, GFP_KERNEL,
                                         NULL, cleanup_uevent_env, env);
        if (info) {
            retval = call_usermodehelper_exec(info, UMH_NO_WAIT);
            env = NULL; /* freed by cleanup_uevent_env */
        }
    }
#endif
```

这一段只有内核编译时启用 `CONFIG_UEVENT_HELPER` 才会生效，对应第一章 1.4 节所述 mdev 事件通道。逻辑要点：

- 只有 `uevent_helper[0]` 非空（即 `/proc/sys/kernel/hotplug` 被设置为某个可执行程序路径）且 `kobj_usermode_filter()` 未过滤该对象时，才会继续。
- 补充 `HOME`、`PATH` 两个基础环境变量，保证被拉起的用户态程序有最基本的运行环境。
- `init_uevent_argv()` 根据 `subsystem` 构造待执行程序的参数列表（`argv[0]` 即 `uevent_helper` 指向的路径）。
- `call_usermodehelper_setup()` 与 `call_usermodehelper_exec()` 以 `UMH_NO_WAIT` 方式创建并执行一个新的用户态进程，也就是内核直接 fork/exec 出 mdev（或 `uevent_helper` 指向的其他程序），并把 `env` 中的全部变量作为该进程的环境变量传入。
- 一旦 `call_usermodehelper_setup()` 成功接管 `env`，`env = NULL` 表示其后续释放交给 `cleanup_uevent_env()` 负责，不再由本函数末尾的 `kfree(env)` 处理，避免重复释放。

现代主流发行版通常不再设置 `uevent_helper`（`CONFIG_UEVENT_HELPER` 默认也已在较新内核中移除或默认关闭），完全依赖 netlink 广播配合常驻的 udev 进程；`uevent_helper` 路径主要保留给 mdev 一类需要在没有常驻进程的极简环境中工作的场景。

## 3.13 退出清理

```c
exit:
    kfree(devpath);
    kfree(env);
    return retval;
```

无论函数从哪个分支跳转到 `exit`，都会释放 `devpath` 和 `env` 占用的内存后返回 `retval`。3.12.2 节中提到，一旦 `env` 被 `call_usermodehelper_setup()` 接管，本处的 `kfree(env)` 因 `env` 已被置 `NULL` 而不会重复释放。

## 3.14 小结

- `kobject_uevent()` 只是 `kobject_uevent_env(kobj, action, NULL)` 的简化调用，真正的发送逻辑集中在 `kobject_uevent_env()` 中。
- 函数首先沿 `kobj` 的 parent 链向上查找所属 `kset`；若 `kobj` 自身和所有祖先都没有设置 `kset`，函数在此处直接返回 `-EINVAL`，后续所有发送逻辑都不会执行——这就是第二章案例中必须先设置 `mykoject01->kset = mykset` 的原因。
- 找到 `kset` 后，`uevent_suppress` 标志和 `uevent_ops->filter` 回调可以让该事件被静默丢弃；`uevent_ops->name` 或 `kset` 自身名称决定 `SUBSYSTEM` 环境变量。
- `ACTION`、`DEVPATH`、`SUBSYSTEM`、`SEQNUM` 是每次事件的默认环境变量，`envp_ext` 与 `uevent_ops->uevent` 回调可以在此基础上继续追加内容。
- `action` 为 `KOBJ_ADD` 或 `KOBJ_UNBIND` 时会触发标记状态位或清理 `MODALIAS` 等附加处理，其余动作不做特殊处理。
- 事件最终通过两条独立路径送达用户空间：`kobject_uevent_net_broadcast()` 对应 udev 依赖的 netlink 广播，`CONFIG_UEVENT_HELPER` 分支对应 mdev 依赖的 `uevent_helper` 用户态程序调用，二者分别呼应第一章介绍的两种热插拔实现机制。

# 4 `kset_uevent_ops`：定制事件过滤、内容与子系统名

第三章分析 `kobject_uevent_env()` 时已经看到，`kset` 携带的 `uevent_ops` 在事件发送路径上被使用了三次：过滤事件（3.5 节）、确定 `SUBSYSTEM`（3.6 节）、补充自定义环境变量（3.9 节）。第二章的案例把 `kset_create_and_add()` 的 `uevent_ops` 参数传成了 `NULL`，因此这三处都走的是默认逻辑。本章补充 `uevent_ops` 的具体实现，观察同一个 `kset` 下的不同 `kobject` 如何通过它获得差异化的事件行为。

## 4.1 `struct kset_uevent_ops`

### 4.1.1 结构体定义

```c
struct kset_uevent_ops {
    int (*filter)(struct kobject *kobj);
    const char *(*name)(struct kobject *kobj);
    int (*uevent)(struct kobject *kobj, struct kobj_uevent_env *env);
};
```

成员说明：

- `filter`：事件发送前的过滤回调，返回值决定该次事件是否真的发出。
- `name`：子系统命名回调，返回值会覆盖默认的 `SUBSYSTEM` 取值。
- `uevent`：事件内容补充回调，可以在默认环境变量之外追加自定义变量。

三个回调都是可选的：`kset_create_and_add()` 的第二个参数允许传 `NULL`，此时第三章描述的默认逻辑（不过滤、`SUBSYSTEM` 取 `kset` 自身名字、不追加额外变量）继续生效。

### 4.1.2 版本差异

较旧的内核版本中，`filter`、`name`、`uevent` 三个回调各自还带有一个 `struct kset *kset` 参数：

```c
struct kset_uevent_ops {
    int (*filter)(struct kset *kset, struct kobject *kobj);
    const char *(*name)(struct kset *kset, struct kobject *kobj);
    int (*uevent)(struct kset *kset, struct kobject *kobj,
                  struct kobj_uevent_env *env);
};
```

第三章分析的源码中，调用点写作 `uevent_ops->filter(kobj)`、`uevent_ops->name(kobj)`、`uevent_ops->uevent(kobj, env)`，属于去掉 `kset` 参数之后的写法——`kobj->kset` 本身已经能取到所属集合，回调里再重复传一次 `kset` 属于冗余参数，因此较新内核精简掉了这个参数。两种签名在语义上完全等价，编写驱动前应以目标内核头文件 `include/linux/kobject.h` 中 `struct kset_uevent_ops` 的实际定义为准；较新内核已经不再使用带 `kset` 参数的旧版本写法，本章后续案例统一采用新版本的单参数写法，与 4.1.1 节的结构体定义、第三章分析的调用点保持一致。

## 4.2 `filter`：按需过滤事件

### 4.2.1 调用位置与返回语义

第三章 3.5 节给出的调用点使用的是新版本单参数签名：

```c
if (uevent_ops && uevent_ops->filter)
    if (!uevent_ops->filter(kobj)) {
        /* 丢弃事件 */
        return 0;
    }
```

若目标内核仍使用 4.1.2 节所述的旧版本三参数签名，调用点相应写作 `uevent_ops->filter(kset, kobj)`，含义不变。`filter` 返回 `0` 表示丢弃事件，返回非 `0` 表示放行。这个返回语义与第六章总线 `match` 回调的"非零表示匹配"看起来相似，但作用相反：`match` 的非零表示"允许绑定"，`filter` 的非零表示"允许发送"——共同点只是二者都用非零表示"通过"。

### 4.2.2 示例

```c
static int myfilter(const struct kobject *kobj)
{
    if (strcmp(kobj->name, "my_kobject1") == 0)
        return 0;

    return 1;
}
```

逻辑说明：

- 当被检查的 `kobj` 名字等于 `"my_kobject1"` 时返回 `0`，该对象产生的事件会被丢弃。
- 其余名字的 `kobj` 返回 `1`，事件正常发送。

`kobj->name` 是 `struct kobject` 的公开字段，可以直接比较；也可以使用 `kobject_name(kobj)` 访问器，二者取的是同一个字段。

需要注意的是，这里不能像总线匹配那样直接写成 `return !strcmp(kobj->name, "my_kobject1");`。总线 `match` 要求"名字相等则返回非零"，而这里的 `filter` 要求"名字相等则返回零（丢弃）"，两者对同一个 `strcmp()` 结果的期望正好相反。若照搬 `!strcmp()` 的写法，会把"应当丢弃"的对象误判为"允许发送"，因此案例中使用显式的 `if/else` 而非简化写法。

## 4.3 `uevent`：追加自定义环境变量

### 4.3.1 调用位置

对应第三章 3.9 节的新版本单参数调用点：

```c
if (uevent_ops && uevent_ops->uevent) {
    retval = uevent_ops->uevent(kobj, env);
    ...
}
```

旧版本三参数签名下，调用点相应写作 `uevent_ops->uevent(kset, kobj, env)`。

该回调在默认的 `ACTION`、`DEVPATH`、`SUBSYSTEM` 三个环境变量写入之后被调用，可以借助 `add_uevent_var()` 继续向 `env` 追加变量。

### 4.3.2 示例

```c
static int myuevent(const struct kobject *kobj,
                    struct kobj_uevent_env *env)
{
    add_uevent_var(env, "MYDEVICE=%s", "TENGFEICHEN");
    return 0;
}
```

调用 `kobject_uevent()` 触发事件时，若该 `kobj` 未被 `filter` 拦截，最终发送的环境变量中会额外包含 `MYDEVICE=TENGFEICHEN`。`add_uevent_var()` 的用法与第三章 3.8 节写入默认变量时完全一致，都是拼接 `KEY=VALUE` 形式的字符串。

## 4.4 `name`：覆盖子系统名称

```c
static const char *myname(const struct kobject *kobj)
{
    return "my_kset";
}
```

按照第三章 3.6 节的逻辑，`uevent_ops->name` 存在时优先使用它的返回值作为 `subsystem`，即事件中 `SUBSYSTEM=` 的取值。这里无论实际 `kset` 叫什么名字，`SUBSYSTEM` 都会被固定为 `"my_kset"`。

需要区分的是，`name` 回调只影响 `SUBSYSTEM` 环境变量，不影响 `kobj` 在 sysfs 中的实际路径。`DEVPATH` 由 `kobject_get_path()`（第三章 3.7 节）根据 `kobj` 自身名称、`parent` 和 `kset` 的真实名字生成，不会因为 `name` 回调返回了别的字符串而改变。

## 4.5 组装 `kset_uevent_ops` 并绑定到 `kset`

```c
static const struct kset_uevent_ops my_uevent_ops = {
    .filter = myfilter,
    .uevent = myuevent,
    .name = myname,
};
```

`kset_create_and_add()` 的第二个参数即为该结构体指针，第二章案例中这里传的是 `NULL`：

```c
mykset = kset_create_and_add("my_kset", &my_uevent_ops, NULL);
```

一旦传入 `&my_uevent_ops`，该 `kset` 下所有成员 `kobject` 发送事件时，都会经过 `myfilter`、`myuevent`、`myname` 三个回调。

## 4.6 案例：同一 `kset` 下两个 `kobject` 的差异化事件

### 4.6.1 案例逻辑

在第二章案例基础上新增第二个 `kobject`（`my_kobject2`），两者共用同一个绑定了 `my_uevent_ops` 的 `mykset`：

1. `my_kobject1` 的名字与 `myfilter` 中的过滤条件相同，触发的 `KOBJ_CHANGE` 事件会被丢弃，`udevadm monitor` 看不到对应输出。
2. `my_kobject2` 的名字不满足过滤条件，触发的 `KOBJ_ADD` 事件正常发送，且携带 `SUBSYSTEM=my_kset` 与 `MYDEVICE=TENGFEICHEN`。

### 4.6.2 完整模块代码

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/init.h>
#include <linux/slab.h>
#include <linux/string.h>

static struct kobj_type mytype;

struct kobject *my_kobj1;
struct kobject *my_kobj2;

struct kset *mykset;

static int myfilter(const struct kobject *kobj)
{
    if (strcmp(kobj->name, "my_kobject1") == 0)
        return 0;

    return 1;
}

static int myuevent(const struct kobject *kobj,
                    struct kobj_uevent_env *env)
{
    add_uevent_var(env, "MYDEVICE=%s", "TENGFEICHEN");
    return 0;
}

static const char *myname(const struct kobject *kobj)
{
    return "my_kset";
}

static const struct kset_uevent_ops my_uevent_ops = {
    .filter = myfilter,
    .uevent = myuevent,
    .name = myname,
};

static int __init mykobject_init(void)
{
    int ret;

    mykset = kset_create_and_add("my_kset", &my_uevent_ops, NULL);

    my_kobj1 = kzalloc(sizeof(struct kobject), GFP_KERNEL);
    my_kobj1->kset = mykset;
    ret = kobject_init_and_add(my_kobj1, &mytype, NULL, "%s", "my_kobject1");

    my_kobj2 = kzalloc(sizeof(struct kobject), GFP_KERNEL);
    my_kobj2->kset = mykset;
    ret = kobject_init_and_add(my_kobj2, &mytype, NULL, "%s", "my_kobject2");

    ret = kobject_uevent(my_kobj1, KOBJ_CHANGE);
    ret = kobject_uevent(my_kobj2, KOBJ_ADD);

    return 0;
}

static void __exit mykobject_exit(void)
{
    kobject_put(my_kobj1);
    kobject_put(my_kobj2);
    kset_unregister(mykset);
}

module_init(mykobject_init);
module_exit(mykobject_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("TF");
MODULE_VERSION("1.0");
```

案例仍未给 `mytype` 设置 `.release`，与第二章案例存在相同的教学简化，正式驱动中必须补齐。

### 4.6.3 预期现象与观察方式

按照第二章 2.3 节的顺序，先执行 `udevadm monitor &`，再 `insmod` 该模块，可观察到：

- `my_kobject1` 的 `KOBJ_CHANGE` 事件被 `myfilter` 拦截，终端不会打印与 `my_kobject1` 相关的 `KERNEL`/`UDEV` 行。
- `my_kobject2` 的 `KOBJ_ADD` 事件正常发出，会出现一行形如 `KERNEL[...] add /my_kset/my_kobject2 (my_kset)` 的输出——括号中的子系统名由 `myname` 回调决定，这里恰好与 `kset` 注册时使用的名字 `"my_kset"` 相同；`name` 回调的返回值是否与 `kset` 真实名字一致完全取决于其实现，二者并无强制关联。
- 默认的 `udevadm monitor` 只打印动作类型和路径，不显示完整环境变量；若要确认 `MYDEVICE=TENGFEICHEN` 是否被正确携带，需要改用 `udevadm monitor -e`（打印完整环境变量）或 `udevadm monitor -p`（打印属性）等带环境变量输出的选项。

## 4.7 小结

- `kset_uevent_ops` 的 `filter`、`uevent`、`name` 三个回调分别对应第三章源码中的事件过滤、内容补充、子系统命名三个环节，`kset_create_and_add()` 传入 `NULL` 时这三处都使用默认逻辑。
- `filter` 返回 `0` 表示丢弃事件、非 `0` 表示放行，与总线 `match` 回调"非零表示匹配"的语义相似但用途不同，直接套用 `!strcmp()` 容易把返回值语义写反，应使用显式的 `if/else`。
- `uevent` 回调在默认环境变量写入后被调用，通过 `add_uevent_var()` 可以继续追加自定义变量，随事件一起发送到用户空间。
- `name` 回调只改变事件中的 `SUBSYSTEM` 取值，不影响 `kobject` 在 sysfs 中的真实路径。
- 同一个 `kset` 下的多个 `kobject` 共用同一套 `uevent_ops`，但 `filter` 可以按 `kobj` 自身信息（如名字）做差异化处理，使部分对象的事件被丢弃、部分对象正常发送。
- `kset_uevent_ops` 的回调签名在不同内核版本间存在是否携带 `kset` 参数的差异，编写驱动前应核对目标内核头文件中的实际定义。

# 5 使用 netlink 监听内核 uevent 广播

## 5.1 从 uevent 广播到用户空间监听

第三章 3.12.1 节分析过，`kobject_uevent_env()` 在写完所有环境变量之后，会调用 `kobject_uevent_net_broadcast()` 将事件通过 netlink 广播出去，udev 正是通过监听这条 netlink 通道获得 `ACTION`、`DEVPATH`、`SUBSYSTEM`、`SEQNUM` 以及第四章案例中追加的 `MYDEVICE=TENGFEICHEN` 等全部环境变量。`udevadm monitor` 默认只挑选其中的动作类型和路径打印，若要直接观察一次 uevent 广播中携带的完整环境变量，可以绕过 udev，在用户空间自行创建一个 netlink 套接字，直接订阅这条广播通道并打印收到的原始数据。本章即实现这样一个最小化的用户态监听程序。

## 5.2 netlink 机制简介

系统调用和 sysfs 文件都要求用户空间主动发起访问：系统调用由用户进程主动陷入内核并等待返回，sysfs 属性文件也需要用户进程主动发起读写，内核都无法主动向用户空间投递消息。netlink 则基于 socket 实现，内核可以在事件发生时主动构造消息并广播给所有已订阅的用户态进程，配合组播（multicast）机制还能同时通知多个监听者，这是 netlink 相比系统调用、sysfs 等机制的关键差异，也是内核选择用它来分发 uevent 的原因。

netlink 使用独立的地址族 `AF_NETLINK`，同一地址族下又通过协议号区分不同子系统的通信通道，例如路由表变更使用 `NETLINK_ROUTE`，而 `kobject_uevent()` 广播使用的是 `NETLINK_KOBJECT_UEVENT`。用户空间程序只要以相同的地址族和协议号创建 socket 并绑定到内核约定的组播组，就能收到该子系统发出的所有广播消息。

## 5.3 创建 netlink 套接字

```c
int socket(int domain, int type, int protocol);
```

参数说明：

- `domain`：协议族，监听 uevent 广播时固定为 `AF_NETLINK`。
- `type`：套接字类型，netlink 使用 `SOCK_RAW`（原始数据报）。
- `protocol`：具体的 netlink 子协议，监听内核 uevent 广播时固定为 `NETLINK_KOBJECT_UEVENT`。

返回值说明：

- `>= 0`：套接字描述符。
- `< 0`：创建失败。

对应的调用代码：

```c
int socket_fd;

socket_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT);
if (socket_fd < 0) {
    printf("socket error\n");
    return -1;
}
```

## 5.4 绑定 socket

### 5.4.1 struct sockaddr_nl

netlink 地址不是 IP 地址，而是使用专门的 `struct sockaddr_nl` 结构体描述：

```c
struct sockaddr_nl {
    sa_family_t nl_family; /* AF_NETLINK */
    unsigned short nl_pad;  /* 填充字段，恒为 0 */
    __u32 nl_pid;           /* 端点地址标识 */
    __u32 nl_groups;        /* 组播组掩码 */
};
```

字段说明：

- `nl_family`：固定填 `AF_NETLINK`，与创建 socket 时的 `domain` 保持一致。
- `nl_pad`：填充字段，无实际意义，固定填 `0`。
- `nl_pid`：该 netlink 端点的地址标识，通常称为端口 ID。用户态进程一般直接填 `0`，由内核自动分配一个与调用者唯一对应的标识；是否加入组播与该字段无关，加入哪个组播组由 `nl_groups` 决定。
- `nl_groups`：组播组掩码，每一位对应一个组播组编号。`kobject_uevent_net_broadcast()` 把所有 uevent 消息都发送到同一个组播组，对应掩码的第 0 位，因此用户态只需将 `nl_groups` 置为 `1`（即第 0 位置 1），即可订阅内核发出的全部 uevent 广播。

### 5.4.2 bind 函数

函数功能：将一个地址与一个套接字进行绑定（建立套接字与地址的关联关系）。

头文件：

```c
#include <sys/types.h>
#include <sys/socket.h>
```

函数原型：

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

参数说明：

- `sockfd`：待绑定的 socket 描述符。
- `addr`：指向地址结构体的指针。用户态程序按照实际使用的地址族构造对应结构体，再强制转换为 `struct sockaddr *` 传入；监听 uevent 广播时传入的是 `struct sockaddr_nl`。
- `addrlen`：`addr` 所指向结构体的字节长度。

返回值说明：

- `0`：绑定成功。
- `-1`：绑定失败。

对应的绑定代码：

```c
struct sockaddr_nl nl;
int ret;

bzero(&nl, sizeof(struct sockaddr_nl));
nl.nl_family = AF_NETLINK;
nl.nl_pid = 0;
nl.nl_groups = 1;

ret = bind(socket_fd, (struct sockaddr *)&nl, sizeof(struct sockaddr_nl));
if (ret < 0) {
    printf("bind error\n");
    return -1;
}
```

`nl` 在栈上声明为结构体变量而非指针，`bzero()`、赋值操作以及传入 `bind()` 时都通过取地址 `&nl` 完成，避免使用未分配内存的裸指针。

## 5.5 接收数据

netlink 套接字不需要调用 `listen()`，绑定完成后即可直接调用 `recv()` 读取内核广播的消息。

头文件：

```c
#include <sys/types.h>
#include <sys/socket.h>
```

函数原型：

```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

参数说明：

- `sockfd`：已绑定的套接字描述符。
- `buf`：接收数据的缓冲区。
- `len`：期望读取的最大字节数。
- `flags`：控制接收行为的标志位，一般填 `0`。

返回值说明：

- `> 0`：实际读取到的字节数。
- `<= 0`：连接关闭或读取出错。

### 5.5.1 消息内容布局

`kobject_uevent_net_broadcast()` 在组装广播消息时，先写入一行 `"动作@设备路径"` 形式的字符串（例如 `add@/my_kset/my_kobject2`），随后依次拼接第三章 3.8 节写入的默认环境变量（`ACTION`、`DEVPATH`、`SUBSYSTEM`、`SEQNUM` 等）以及第四章 `uevent` 回调通过 `add_uevent_var()` 追加的自定义变量（如 `MYDEVICE=TENGFEICHEN`）。这些字符串在同一块缓冲区中依次排列，彼此以 `\0`分隔，而不是以 `\n` 分隔，因此 `recv()` 读到的原始缓冲区若直接用 `printf("%s", buf)` 打印，只会显示第一个 `\0` 之前的内容。要把整段缓冲区中的所有字段都显示出来，需要先把其中的 `\0` 分隔符替换成 `\n`，再整体打印。

## 5.6 完整示例：用户态 uevent 监听程序

```c
#include <stdio.h>
#include <strings.h>
#include <sys/socket.h>
#include <linux/netlink.h>

int main(int argc, char *argv[])
{
    int socket_fd;
    int ret;
    int len;
    int i;
    struct sockaddr_nl nl;
    char buf[4096];

    socket_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT);
    if (socket_fd < 0) {
        printf("socket error\n");
        return -1;
    }

    bzero(&nl, sizeof(struct sockaddr_nl));
    nl.nl_family = AF_NETLINK;
    nl.nl_pid = 0;
    nl.nl_groups = 1;

    ret = bind(socket_fd, (struct sockaddr *)&nl, sizeof(struct sockaddr_nl));
    if (ret < 0) {
        printf("bind error\n");
        return -1;
    }

    while (1) {
        bzero(buf, sizeof(buf));
        len = recv(socket_fd, buf, sizeof(buf), 0);
        if (len <= 0)
            continue;

        for (i = 0; i < len; i++) {
            if (buf[i] == '\0')
                buf[i] = '\n';
        }

        printf("%s\n", buf);
    }

    return 0;
}
```

## 5.7 运行效果

在终端执行该监听程序后，再触发第二章、第四章中的任意模块（`insmod` 对应 `.ko` 文件），监听程序会打印出一整块以换行分隔的文本，第一行是 `动作@设备路径` 形式的头部，随后各行依次是 `ACTION=change`、`DEVPATH=...`、`SUBSYSTEM=...`、`SEQNUM=...`；若触发的是第四章 `my_kobject2` 的事件，还会额外看到 `SUBSYSTEM=my_kset` 与 `MYDEVICE=TENGFEICHEN` 这两行——这正是 `udevadm monitor` 默认不会显示、需要加 `-e` 或 `-p` 参数才能看到的完整环境变量内容，此处通过直接监听 netlink 广播即可原样获得。

## 5.8 小结

- netlink 基于 socket 实现，允许内核主动向用户空间广播消息，这是它区别于系统调用、sysfs 等机制的核心特点，`kobject_uevent()` 正是通过这条通道把 uevent 事件发给用户空间。
- 监听内核 uevent 广播需要以 `socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT)` 创建套接字，并使用 `struct sockaddr_nl` 描述本地地址，其中 `nl_groups` 置 `1` 表示订阅 uevent 广播所在的组播组。
- netlink 套接字无需 `listen()`，`bind()` 完成后直接调用 `recv()` 即可收取广播消息。
- 一次 uevent 广播的原始数据由 `动作@设备路径` 头部和若干条以 `\0` 分隔的 `KEY=VALUE` 环境变量拼接而成，将其中的 `\0` 替换为 `\n` 后整体打印，即可还原出内核在 `kobject_uevent_env()` 中写入的全部环境变量，包括驱动通过 `add_uevent_var()` 追加的自定义内容。

# 6 配置 `uevent_helper`

第三章 3.12.2 节分析过，`kobject_uevent_env()` 在 `CONFIG_UEVENT_HELPER` 分支中会检查全局变量 `uevent_helper[0]` 是否非空，一旦非空，就以该字符串为路径，通过 `call_usermodehelper_setup()`/`call_usermodehelper_exec()` fork/exec 出对应的用户态程序（通常是 mdev）。这个全局变量从哪里获得内容、以及如何在运行时修改它，是让 mdev 一类基于 `uevent_helper` 的热插拔方案真正生效的前提，本章说明具体的配置方式。

## 6.1 三种配置方式概览

`uevent_helper` 的取值可以通过三种途径确定，三者作用的对象完全相同，区别只在于生效时机和操作接口：

1. 编译内核时通过 `CONFIG_UEVENT_HELPER_PATH` 静态指定初始值，内核启动后不再修改。
2. 系统运行后，通过写 `/sys/kernel/uevent_helper` 动态修改。
3. 系统运行后，通过写 `/proc/sys/kernel/hotplug` 动态修改。

## 6.2 方法一：编译时静态配置

在 `make menuconfig` 图形化配置界面中，需要依次开启以下选项：

- `Device Drivers` → `Generic Driver Options` → `[*] Support for uevent helper`（对应 `CONFIG_UEVENT_HELPER`），以及紧随其后的 `(...) path to uevent helper`（对应 `CONFIG_UEVENT_HELPER_PATH`，填入 mdev 等程序的路径，如 `/sbin/mdev`）。
- `File systems` → `Pseudo filesystems` → `[*] /proc file system support`（对应 `CONFIG_PROC_FS`），这是 `/proc` 目录本身存在的前提。
- `File systems` → `Pseudo filesystems` → `[*] Sysctl support (/proc/sys)`（对应 `CONFIG_PROC_SYSCTL`），这是 `/proc/sys/kernel/hotplug` 一类可读写内核参数节点存在的前提，若不开启，即使 `CONFIG_PROC_FS` 打开，`/proc/sys` 下也不会出现对应文件。
- `[*] Networking support`（对应 `CONFIG_NET`）。这一项并非 `uevent_helper` 生效的必要条件，而是第三章 3.12.1 节所述 `kobject_uevent_net_broadcast()` 走 netlink 广播路径的前提；实际产品中若同时需要 udev（依赖 netlink）与 mdev（依赖 `uevent_helper`）两条通道都可用，通常会一并开启。

这种方式的特点是内核一旦编译完成，`uevent_helper` 在启动时就已经被初始化为 `CONFIG_UEVENT_HELPER_PATH` 指定的路径；只要系统运行期间不再通过下面两种运行时方式修改它，这个编译期指定的值会一直生效。

## 6.3 方法二：运行时写 `/sys/kernel/uevent_helper`

```sh
echo /sbin/mdev > /sys/kernel/uevent_helper
```

无论内核是否配置了 `CONFIG_UEVENT_HELPER_PATH`，只要 `CONFIG_UEVENT_HELPER` 被编译进内核，`/sys/kernel/uevent_helper` 这个属性文件就会存在，系统运行后随时可以通过该命令覆盖当前生效的路径。

## 6.4 方法三：运行时写 `/proc/sys/kernel/hotplug`

```sh
echo /sbin/mdev > /proc/sys/kernel/hotplug
```

同样不依赖 `CONFIG_UEVENT_HELPER_PATH` 是否配置，只要开启了 `CONFIG_PROC_FS` 与 `CONFIG_PROC_SYSCTL`，就可以通过写 `/proc/sys/kernel/hotplug` 达到与方法二相同的效果。

## 6.5 为什么两个节点操作的是同一份数据

`/sys/kernel/uevent_helper` 与 `/proc/sys/kernel/hotplug` 是两个路径完全不同的文件，之所以对它们的读写效果一致，原因在于二者在内核源码中分别是同一个全局变量 `uevent_helper` 的两套独立的访问接口，而不是两份互相同步的数据。

### 6.5.1 sysfs 侧：`kernel/ksysfs.c` 中的属性定义

```c
#ifdef CONFIG_UEVENT_HELPER
/* uevent helper program, used during early boot */
static ssize_t uevent_helper_show(struct kobject *kobj,
                   struct kobj_attribute *attr, char *buf)
{
    return sprintf(buf, "%s\n", uevent_helper);
}

static ssize_t uevent_helper_store(struct kobject *kobj,
                    struct kobj_attribute *attr,
                    const char *buf, size_t count)
{
    if (count + 1 > UEVENT_HELPER_PATH_LEN)
        return -ENOENT;

    memcpy(uevent_helper, buf, count);
    uevent_helper[count] = '\0';
    if (count && uevent_helper[count - 1] == '\n')
        uevent_helper[count - 1] = '\0';

    return count;
}
KERNEL_ATTR_RW(uevent_helper);
#endif
```

`kernel/ksysfs.c` 里维护的是 `kernel_kobj` 这个 `kobject` 下的一组属性文件，`/sys/kernel/` 目录正是它在 sysfs 中的映射。`KERNEL_ATTR_RW(uevent_helper)` 是基于 `05_device_model.md` 介绍过的 sysfs 属性机制封装的宏，效果是声明一个名为 `uevent_helper` 的 `struct kobj_attribute`，把 `uevent_helper_show`/`uevent_helper_store` 分别绑定为该属性的读、写回调，最终以 `/sys/kernel/uevent_helper` 的形式呈现给用户空间。

`uevent_helper_show()` 直接把全局字符数组 `uevent_helper` 的内容格式化输出；`uevent_helper_store()` 把用户写入的内容拷贝进这同一个数组，并去掉末尾可能带有的换行符。这里读写的 `uevent_helper`，正是第三章 3.12.2 节 `kobject_uevent_env()` 中判断 `uevent_helper[0]` 是否非空时使用的同一个全局变量。

### 6.5.2 proc 侧：`kernel/sysctl.c` 中的 `hotplug` 项

```c
#ifdef CONFIG_UEVENT_HELPER
    {
        .procname   = "hotplug",
        .data       = &uevent_helper,
        .maxlen     = UEVENT_HELPER_PATH_LEN,
        .mode       = 0644,
        .proc_handler   = proc_dostring,
    },
#endif
```

字段说明：

- `procname`：该节点在 `/proc/sys/kernel/` 目录下显示的文件名，这里为 `"hotplug"`，对应完整路径 `/proc/sys/kernel/hotplug`。
- `data`：指向该节点实际存储数据的内存地址，这里填的是 `&uevent_helper`，也就是 6.5.1 节展示的同一个全局字符数组的地址，而不是它的副本。
- `maxlen`：该节点内容允许占用的最大字节数，取值为 `UEVENT_HELPER_PATH_LEN`，与 `uevent_helper` 数组本身的长度一致，防止写入内容溢出数组边界。
- `mode`：该文件的访问权限位，`0644` 对应 `-rw-r--r--`：属主（默认是 root）可读可写，属组和其他用户只能读取，不能写入。
- `proc_handler`：实际处理该节点读写请求的回调函数，`proc_dostring` 是内核提供的通用回调，专门用于内容是普通字符串的 sysctl 节点，读时把 `data` 指向的字符串输出，写时把用户写入的内容拷贝进 `data` 指向的缓冲区（同样遵守 `maxlen` 限制）。

### 6.5.3 结论：同一个变量，两套访问入口

`uevent_helper` 本身是 `lib/kobject_uevent.c` 中声明的一个全局字符数组，`kernel/ksysfs.c` 通过 `kobj_attribute` 的 `show`/`store` 回调把它暴露为 `/sys/kernel/uevent_helper`，`kernel/sysctl.c` 通过 `ctl_table` 的 `data` 指针把它暴露为 `/proc/sys/kernel/hotplug`。两套接口各自独立注册、各自处理文件的读写请求，但操作的目标是同一块内存，因此无论通过哪一个路径写入新值，另一个路径读到的都是刚刚写入的结果；`kobject_uevent_env()` 在发送事件时也只读取这一份数据，不关心它最近一次是通过哪个接口被修改的。

## 6.6 proc 文件系统简介

`/proc` 是一种伪文件系统（pseudo filesystem），它以文件和目录的形式呈现内核内部状态，但这些“文件”并不对应磁盘上的真实数据块：每当用户进程读取或写入某个 `/proc` 节点，VFS 层会调用该节点注册时提供的回调函数（如上文的 `proc_dostring`）现场生成或解析内容，而不是像普通文件那样从磁盘块设备读取。`/proc/sys` 是 `/proc` 下专门用来暴露可调内核参数（sysctl）的子目录，`struct ctl_table` 数组中的每一项都会在这里生成一个对应的节点。

尽管节点内容是动态生成的，VFS 依然为这些节点维护标准的文件属性，包括 `mode` 权限位，并按照与普通文件完全相同的权限检查逻辑生效：`0644` 表示属主拥有读写权限、属组和其他用户只有读权限，因此非 root 用户可以执行 `cat /proc/sys/kernel/hotplug` 查看当前的 `uevent_helper` 路径，但执行写入操作会因权限不足被拒绝。

## 6.7 小结

- `uevent_helper` 是内核中的一个全局字符数组，`kobject_uevent_env()` 在 `CONFIG_UEVENT_HELPER` 分支中读取它来决定是否以及使用哪个路径的用户态程序处理本次 uevent。
- 编译期可以通过 menuconfig 中的 `CONFIG_UEVENT_HELPER`、`CONFIG_UEVENT_HELPER_PATH` 静态指定其初始值，还需要 `CONFIG_PROC_FS`、`CONFIG_PROC_SYSCTL` 支持 `/proc/sys` 节点存在；`CONFIG_NET` 则是 udev 依赖的 netlink 广播通道的前提，与 `uevent_helper` 本身无直接关系。
- 运行时可以通过写 `/sys/kernel/uevent_helper` 或 `/proc/sys/kernel/hotplug` 两种方式修改其值，二者分别由 `kernel/ksysfs.c` 的 `kobj_attribute` 回调和 `kernel/sysctl.c` 的 `ctl_table` 项实现，但 `data`/操作对象都指向同一个全局数组，因此效果完全等价。
- `/proc` 是伪文件系统，节点内容由内核回调函数动态生成，但依然遵守标准的 Unix 文件权限模型，`ctl_table.mode` 字段即用于设置这一权限。

# 7 编写绑定到 `uevent_helper` 的用户态处理程序

第六章说明了如何把一个可执行程序的路径写入 `uevent_helper`，使内核在产生 uevent 时通过 `call_usermodehelper_setup()`/`call_usermodehelper_exec()`（第三章 3.12.2 节）拉起该程序。本章编写这个程序本身，说明它如何拿到事件携带的环境变量，以及在实际调试时会遇到的一个常见问题：`printf` 打印的内容为什么在终端上看不到。

## 7.1 用 `getenv` 读取事件环境变量

第三章 3.12.2 节分析过，`call_usermodehelper_exec()` 拉起的新进程，其环境变量正是 `kobject_uevent_env()` 中通过 `ACTION`、`DEVPATH`、`SUBSYSTEM` 等默认变量以及 `uevent_ops->uevent()` 回调追加的自定义变量组成的 `env->envp`。因此在被 `uevent_helper` 拉起的用户态程序里，直接调用标准库的 `getenv()` 就可以读到这些变量：

```c
printf("SUBSYSTEM is %s\n", getenv("SUBSYSTEM"));
```

`getenv()` 本身能够正确取到 `SUBSYSTEM` 的值，问题出在下一步——`printf` 的输出并不会显示在操作者正在观察的终端上。

## 7.2 为什么 `printf` 看不到输出

`call_usermodehelper_exec()` 启动用户程序的方式，与在 shell 中直接运行一个命令完全不同：它是内核直接以子进程形式 fork/exec 出该程序，这个新进程并非从操作者当前使用的交互式终端会话派生而来，因此也就没有继承该终端作为自己的控制终端。它的标准输出文件描述符（文件描述符 `1`）默认连接到的是内核为这类用户态辅助进程准备的默认目标，而不是操作者此刻正在盯着看的那个终端窗口。

结果就是：程序内部的 `printf` 调用本身没有问题，字符串也确实被写到了文件描述符 `1`，只是这个描述符当前指向的位置，恰好不是操作者能看到输出的地方。要让这些输出出现在指定终端上，需要在 `printf` 之前，把标准输出重定向到该终端对应的设备节点。

## 7.3 重定向标准输出到终端设备

### 7.3.1 基本思路：`dup2`

```c
int fd = open("/dev/tty", O_WRONLY);
dup2(fd, STDOUT_FILENO);
close(fd);
```

`open()` 打开目标终端设备节点得到一个新的文件描述符，`dup2(fd, STDOUT_FILENO)` 让文件描述符 `1` 指向这个新打开的设备，此后所有写向标准输出的内容（包括 `printf`）都会实际写入该设备；`close(fd)` 关闭多余的原始描述符，此时描述符 `1` 上的引用已经足够维持该设备打开。这段代码必须放在第一次 `printf` 之前，才能让输出走新的目标。

### 7.3.2 `/dev/tty` 在这里不适用

`/dev/tty` 这个设备节点代表的是"调用进程的控制终端"，其具体指向哪一个终端，是内核根据调用进程所属会话（session）动态解析出来的。而 7.2 节已经说明，`uevent_helper` 拉起的进程并非由某个交互式会话派生，也就没有控制终端，因此对这样的进程而言，`open("/dev/tty", ...)` 通常无法解析到任何有效目标。

因此实际生效的写法，是把 `open()` 的第一个参数直接换成这块单板实际使用的物理终端设备节点。以本章使用的开发板为例，串口控制台对应的设备节点是 `/dev/ttyFIQ0`（与第二、四章终端提示符 `[root@RK356X:/]#` 所在的那个串口终端是同一个）。

## 7.4 案例：绑定环境变量打印程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    int fd;

    fd = open("/dev/ttyFIQ0", O_WRONLY);
    dup2(fd, STDOUT_FILENO);
    close(fd);

    printf("SUBSYSTEM is %s\n", getenv("SUBSYSTEM"));

    return 0;
}
```

将该程序编译为可执行文件后放到根文件系统中（例如放在根目录下，命名为 `/mdev`，与第六章示例保持一致，注意这里只是借用了 `mdev`这个名字，实际内容是本章自行编写的程序，并非 BusyBox 自带的 mdev），再按照第六章 6.3 节的方法把它设置为 `uevent_helper`：

```sh
echo /mdev > /sys/kernel/uevent_helper
```

## 7.5 触发效果

加载第四章 4.6.2 节的 `uevent_ops.ko` 模块：

```sh
insmod uevent_ops.ko
```

终端上会依次看到两行输出：

```text
SUBSYSTEM is my_kset
SUBSYSTEM is module
```

结果解读：

- `SUBSYSTEM is my_kset`：对应 `my_kobj2` 触发的 `KOBJ_ADD` 事件。`my_kobj1` 的事件已经在内核侧被 `myfilter` 拦截，从未到达 `uevent_helper` 分支，因此不会产生对应的一次程序调用；`my_kobj2` 的 `SUBSYSTEM` 被 `myname` 回调覆盖为 `my_kset`，与第四章 4.6.3 节 `udevadm monitor` 观察到的结果一致。
- `SUBSYSTEM is module`：与第二章 2.3.3 节 `udevadm monitor` 输出中 `KERNEL[...] add /module/uevent (module)` 对应的是同一个事件——`insmod` 加载模块时，内核模块子系统自身也会为该模块的 `kobject` 发出一次 `add` 事件，其 `SUBSYSTEM` 固定为 `module`。这次事件同样会经过 `uevent_helper` 分支，拉起同一个用户态程序，只是这次 `getenv("SUBSYSTEM")` 读到的是 `"module"`。

`filter`、`uevent`、`name` 三个回调在第三章的调用位置早于 `kobject_uevent_net_broadcast()` 与 `CONFIG_UEVENT_HELPER` 两条分支的分叉点，因此它们对事件的过滤和改写效果，netlink 广播（udev 一侧）与 `uevent_helper` 调用（本章一侧）会同时生效，这也是两次输出中的 `SUBSYSTEM` 取值与第四章 udev 观察结果完全一致的原因。

## 7.6 小结

- `uevent_helper` 拉起的用户态程序可以直接用 `getenv()` 读取事件携带的环境变量，这部分数据本身是完整且正确的。
- 该程序由内核以 `call_usermodehelper_exec()` 直接 fork/exec 启动，不继承任何交互式终端作为控制终端，因此默认的 `printf` 输出不会出现在操作者正在使用的终端上。
- 解决方式是在 `printf` 之前用 `open()` 打开目标终端设备节点，再用 `dup2()` 将其绑定到 `STDOUT_FILENO`；由于该进程没有控制终端，不能使用 `/dev/tty` 这个依赖控制终端解析的设备节点，必须直接指定具体的物理终端设备（如 `/dev/ttyFIQ0`）。
- 同一次 `insmod` 会触发多次独立的 uevent（自定义 `kobject` 的事件与内核模块子系统自身的事件），`uevent_helper` 拉起的程序会针对每一次事件被独立调用一次，`filter`/`name`/`uevent` 回调对这些事件的处理结果在 netlink 广播和 `uevent_helper` 调用两条路径上完全一致。

# 8 使用 udev 实现 U 盘与 TF 卡自动挂载

前几章分别从内核侧（`kobject_uevent()`、`kset_uevent_ops`）和用户侧（netlink 监听、`uevent_helper` 程序）分析了热插拔事件的产生与接收，本章把这些机制落到一个实际场景：在 Buildroot 构建的文件系统中启用 udev，让 U 盘或 TF 卡插入后自动挂载、拔出后自动卸载。

## 8.1 案例总体流程

在 Buildroot 文件系统中启用 udev 之后，根文件系统的 `/sbin` 目录下会常驻一个 udev 守护进程。该进程在系统启动时启动一次，此后一直运行，同时监听第五章介绍的 netlink uevent 广播；只要内核侧发生热插拔事件，守护进程就会收到通知，并按照配置好的规则决定要执行的动作：

```text
U 盘/TF 卡插入或拔出
    └── 内核生成 uevent（block 子系统，add/remove）
        └── udev 守护进程通过 netlink 收到事件
            └── 按 /etc/udev/rules.d/ 下的规则匹配
                └── 匹配成功则执行规则中 RUN+= 指定的脚本
                    └── 脚本调用 mount/umount 完成挂载或卸载
```

这与第一章 1.3 节介绍的 udev 处理流程完全一致，区别只是这里的"规则"不再是抽象描述，而是本章要动手编写的具体规则文件和脚本。

## 8.2 在 Buildroot 中启用 udev

### 8.2.1 menuconfig 配置路径

Buildroot 默认使用 BusyBox 内建的简化热插拔处理，要换成功能完整的 udev，需要在 `make menuconfig` 中修改 `System configuration` 下的 `/dev management` 选项：

```text
System configuration
    Root FS skeleton (default target skeleton)
    (RK356X) System hostname
    (Welcome to RK356X Buildroot) System banner
    Passwords encoding (md5)
    Init system (BusyBox)
    /dev management (Dynamic using devtmpfs + eudev)  --->
    (system/device_table.txt) Path to the permission tables
```

将 `/dev management` 设置为 `Dynamic using devtmpfs + eudev`。这里的 `eudev` 是 udev 从 systemd 中剥离出来的独立分支，供不依赖 systemd 的发行版（包括 Buildroot）单独使用，功能上与传统 udev 一致，仍然依赖第五章介绍的 netlink 机制接收内核事件。`devtmpfs` 与 `eudev` 是配合关系而非替代关系：内核启动阶段先由 `devtmpfs` 自动创建基本的设备节点，保证系统尽早可用；`eudev` 守护进程随后接管热插拔事件的后续处理，包括本章要配置的规则匹配与脚本执行。

### 8.2.2 确认守护进程已启动

重新编译并烧录系统后，可以通过以下命令确认 udev 守护进程是否已经运行：

```sh
ps -aux | grep "udev"
```

若配置生效，应当能看到一个常驻的 udev 守护进程（`eudev` 提供的 `udevd`）。若看不到该进程，说明 `/dev management` 的配置未生效或系统未重新烧录，需要重新检查 8.2.1 节的配置并重新编译。

## 8.3 编写 udev 规则文件

### 8.3.1 规则文件的位置

udev 规则文件统一存放在 `/etc/udev/rules.d/` 目录下，若该目录不存在需先手动创建：

```sh
mkdir -p /etc/udev/rules.d/
```

在该目录下创建规则文件 `001.rules`。文件名前缀的数字决定了多个规则文件按字典序被加载的先后顺序，当同一事件被多份规则文件处理时，数字较小的文件先生效。

### 8.3.2 规则内容与语法

`001.rules` 内容如下：

```text
KERNEL=="sd[a-z][0-9]", SUBSYSTEM=="block", ACTION=="add", RUN+="/etc/udev/rules.d/usb/usb-add.sh %k"
SUBSYSTEM=="block", ACTION=="remove", RUN+="/etc/udev/rules.d/usb/usb-remove.sh"
```

语法说明：

- `==` 是匹配操作符，左边是内核上报事件时携带的字段名，右边是期望匹配的值；一条规则中所有用 `==` 连接的条件都满足时，规则才生效。
- `KERNEL=="sd[a-z][0-9]"`：匹配设备节点名，正则形式的 `sd[a-z][0-9]` 只匹配形如 `sda1`、`sdb2` 这样"盘符+分区号"的分区设备节点，不匹配 `sda` 这样的整盘设备节点。
- `SUBSYSTEM=="block"`：只处理块设备子系统上报的事件，对应第二章介绍的 `SUBSYSTEM` 环境变量。
- `ACTION=="add"` / `ACTION=="remove"`：分别对应第二章 `enum kobject_action` 中的 `KOBJ_ADD`、`KOBJ_REMOVE`。
- `RUN+="..."`：`+=` 是追加操作符，表示向本条规则待执行的命令列表中追加一条命令，规则匹配成功后由 udev 守护进程执行。
- `%k`：udev 规则中的替换符，代表触发本次事件的内核设备节点名（`kernel name`），例如插入的分区被内核命名为 `sda1` 时，`%k` 就会被替换成 `sda1`。

第一条规则的效果是：当某个 `sd[a-z][0-9]` 形式的分区设备被添加时，调用 `usb-add.sh` 并把分区名作为参数传入；第二条规则的效果是：当任意块设备被移除时，调用 `usb-remove.sh`。

需要注意的是，第二条规则没有像第一条那样加上 `KERNEL=="sd[a-z][0-9]"` 的限定，因此任何 `block` 子系统的移除事件（包括整盘设备、其他分区、`loop` 设备等）都会触发 `usb-remove.sh`。这里为了简化案例，直接对所有块设备的移除事件做统一处理；在需要精确区分设备来源的场景中，应当为移除规则补上与添加规则对称的 `KERNEL` 限定。

在 `/etc/udev/rules.d/` 下创建 `usb` 目录，存放规则中引用的脚本：

```sh
mkdir -p /etc/udev/rules.d/usb
```

## 8.4 编写挂载与卸载脚本

### 8.4.1 usb-add.sh：插入时挂载

```sh
#!/bin/sh
/bin/mount -t vfat /dev/$1 /mnt
sync
```

`$1` 接收 `RUN+=` 中 `%k` 传入的设备节点名（如 `sda1`），因此实际执行的设备路径是 `/dev/sda1`。`mount -t vfat` 把该分区以 vfat 文件系统类型挂载到 `/mnt` 目录，设备路径与挂载点是两个独立参数，必须以空格分隔，不能写成一个路径。`sync` 用于把此前可能残留在内核缓冲区中的写操作刷新到存储设备。

### 8.4.2 usb-remove.sh：拔出时卸载

```sh
#!/bin/sh
sync
/bin/umount -l /mnt
```

拔出 U 盘或 TF 卡时，设备已经从物理上消失，此时若直接执行普通的 `umount`，可能因为设备已不可访问而卸载失败；`umount -l`（`--lazy`，惰性卸载）会立即将该挂载点从文件系统层级中摘除，实际的设备引用则等到不再被任何进程使用时才真正释放，因此更适合物理设备已经拔出这种场景。执行卸载前先 `sync`，是为了尽量减少设备拔出瞬间可能丢失的未落盘数据。

案例中挂载点固定为 `/mnt`，只适合同一时间只插入一个存储设备的场景；如果需要同时支持多个设备，需要按设备名（如 `%k`）生成不同的挂载点目录，避免多个设备共用同一个挂载点造成冲突。

### 8.4.3 赋予脚本可执行权限

```sh
chmod 777 /etc/udev/rules.d/usb/usb-add.sh
chmod 777 /etc/udev/rules.d/usb/usb-remove.sh
```

`RUN+=` 指定的脚本必须具备可执行权限，udev 守护进程才能够真正调用它们；`777` 赋予了所有用户读写执行权限，若不需要其他用户写入脚本内容，`755`（属主可读写执行，其余用户只读、可执行）已经足够，安全性更高。

## 8.5 案例验证

完成 8.2～8.4 节的配置后，插入 U 盘或 TF 卡，可观察到：

1. 内核为新分区创建设备节点（如 `/dev/sda1`），并生成 `SUBSYSTEM=block`、`ACTION=add` 的 uevent。
2. udev 守护进程通过 netlink 收到该事件，匹配 `001.rules` 中的第一条规则，执行 `usb-add.sh sda1`。
3. `/dev/sda1` 被挂载到 `/mnt`，此时执行 `mount` 或 `ls /mnt` 可以看到设备内容。

拔出设备时：

1. 内核生成 `SUBSYSTEM=block`、`ACTION=remove` 的 uevent。
2. udev 守护进程匹配第二条规则，执行 `usb-remove.sh`。
3. `/mnt` 被惰性卸载，之前挂载的内容不再可见。

## 8.6 挂载 TF 卡

### 8.6.1 系统自带的 TF 卡自动挂载规则

完成 8.1～8.5 节的 U 盘挂载配置后，插入 TF 卡（对应内核设备节点 `mmcblk0p1` 一类的 eMMC/SD 分区）会发现同样可以自动挂载，即便还没有为 TF 卡编写任何规则。原因是系统里除了 8.3 节新建的 `/etc/udev/rules.d/001.rules`，`/lib/udev/rules.d/` 目录下还预置了一批出厂规则文件：

```text
[root@RK356X:/lib/udev/rules.d]# ls
50-udev-default.rules
60-block.rules
60-cdrom_id.rules
60-drm.rules
60-evdev.rules
60-input-id.rules
60-persistent-alsa.rules
60-persistent-input.rules
60-persistent-storage-tape.rules
60-persistent-storage.rules
60-persistent-v4l.rules
60-sensor.rules
60-serial.rules
61-partition-init.rules
61-sd-cards-auto-mount.rules
```

其中 `61-sd-cards-auto-mount.rules` 就是负责 TF/SD 卡分区自动挂载的出厂规则，这也是没有专门配置 TF 卡规则时插卡依然能自动挂载的原因。

udev 在启动时会把 `/etc/udev/rules.d/`、`/run/udev/rules.d/`、`/lib/udev/rules.d/` 等多个目录下的规则文件合并成一份规则列表，并按文件名的字典序依次处理；只有当不同目录下存在同名规则文件时，`/etc/udev/rules.d/` 下的文件才会覆盖 `/lib/udev/rules.d/` 下的同名文件。本例中 `001.rules` 与 `61-sd-cards-auto-mount.rules` 文件名并不相同，二者能够共存、都会生效，只是凭借 `001` 在字典序上小于 `61`，`001.rules` 中的规则会先于出厂规则被处理。如果不希望出厂规则介入（例如希望完全由自己的脚本决定挂载点和文件系统类型），需要直接删除 `/lib/udev/rules.d/61-sd-cards-auto-mount.rules`，之后 TF 卡就不会再被这条出厂规则自动挂载，是否挂载完全由自己在 `/etc/udev/rules.d/` 下编写的规则决定。

### 8.6.2 在 001.rules 中追加 TF 卡规则

在已有的 `001.rules` 基础上，追加两条针对 TF 卡分区的规则（无需新建文件，与 8.3.2 节的 U 盘规则共存于同一份 `001.rules` 中）：

```text
KERNEL=="sd[a-z][0-9]", SUBSYSTEM=="block", ACTION=="add", RUN+="/etc/udev/rules.d/usb/usb-add.sh %k"
SUBSYSTEM=="block", ACTION=="remove", RUN+="/etc/udev/rules.d/usb/usb-remove.sh"
KERNEL=="mmcblk[0-9]p[0-9]", SUBSYSTEM=="block", ACTION=="add", RUN+="/etc/udev/rules.d/tf/tf-add.sh %k"
SUBSYSTEM=="block", ACTION=="remove", RUN+="/etc/udev/rules.d/tf/tf-remove.sh"
```

新增两条规则的含义与 8.3.2 节的 U 盘规则完全对应：`KERNEL=="mmcblk[0-9]p[0-9]"` 匹配 `mmcblk0p1`、`mmcblk1p2` 这类 TF/eMMC 分区设备节点（不匹配 `mmcblk0` 这样的整卡设备节点）；当这样的分区被添加时，调用 `tf-add.sh` 并传入分区名；当任意块设备被移除时，调用 `tf-remove.sh`。

需要注意的是，第二条 U 盘移除规则和第四条 TF 卡移除规则都只以 `SUBSYSTEM=="block"` 为条件，没有像各自的添加规则那样限定 `KERNEL`。这意味着无论拔出的是 U 盘分区还是 TF 卡分区，两条移除规则会同时匹配、`usb-remove.sh` 和 `tf-remove.sh` 都会被执行——多出的那一次 `umount -l /mnt` 只是对一个已经被卸载的挂载点重复操作，实际不会造成问题，但严谨的做法仍然是分别给两条移除规则加上对应的 `KERNEL` 限定，使其只处理各自类型的设备。

在 `/etc/udev/rules.d/` 目录下创建 `tf` 目录，存放 TF 卡规则引用的脚本：

```sh
mkdir -p /etc/udev/rules.d/tf
```

创建完成后，`/etc/udev/rules.d/` 下应当同时包含 8.3 节的 `usb` 目录、`001.rules` 文件和本节新建的 `tf` 目录。

### 8.6.3 编写 TF 卡的挂载与卸载脚本

`tf-add.sh`：

```sh
#!/bin/sh
/bin/mount -t vfat /dev/$1 /mnt
sync
```

`tf-remove.sh`：

```sh
#!/bin/sh
sync
/bin/umount -l /mnt
```

两个脚本的写法与 8.4 节的 `usb-add.sh`、`usb-remove.sh` 完全一致，只是服务对象换成了 TF 卡分区：`$1` 接收 `%k` 传入的 `mmcblk0p1` 一类分区名，`mount` 的设备路径与挂载点仍需作为两个独立参数传入；`umount -l` 同样用于应对设备已经物理拔出的场景。

赋予脚本可执行权限：

```sh
chmod 777 /etc/udev/rules.d/tf/tf-add.sh
chmod 777 /etc/udev/rules.d/tf/tf-remove.sh
```

### 8.6.4 验证

插入 TF 卡后：

1. 内核为其分区创建设备节点（如 `/dev/mmcblk0p1`），生成 `SUBSYSTEM=block`、`ACTION=add` 的 uevent。
2. udev 守护进程匹配 `001.rules` 中新增的 TF 卡添加规则，执行 `tf-add.sh mmcblk0p1`，将该分区挂载到 `/mnt`。

拔出 TF 卡后，内核生成 `ACTION=remove` 的 uevent，`usb-remove.sh` 与 `tf-remove.sh` 会一起被执行，`/mnt` 被惰性卸载。若此前没有删除 8.6.1 节提到的出厂规则 `61-sd-cards-auto-mount.rules`，出厂规则也会独立完成一次挂载/卸载，二者互不影响，但会各自尝试操作同一个流程，实际效果以先完成的一次为准。

## 8.7 小结

- Buildroot 需要在 `System configuration` → `/dev management` 中选择 `Dynamic using devtmpfs + eudev`，才会在 `/sbin` 下常驻一个基于 netlink 监听 uevent 的 udev 守护进程；`ps -aux | grep "udev"` 可用于确认该进程是否已经运行。
- udev 规则以 `KEY=="value"` 形式的匹配条件加上 `RUN+="脚本路径"` 形式的动作组成，存放在 `/etc/udev/rules.d/` 下，`%k` 代表触发事件的设备节点名，会作为参数传给 `RUN+=` 指定的脚本。
- 挂载脚本需要将设备路径与挂载点作为两个独立参数传给 `mount`；卸载脚本在设备已经物理拔出的场景下，使用 `umount -l` 惰性卸载比普通 `umount` 更可靠。
- 案例中的移除规则未对设备名做限定，会匹配所有块设备的移除事件；固定挂载点 `/mnt` 也只适合单设备场景，实际产品中需要按需加上更精确的匹配条件和动态挂载点。
- 规则引用的脚本必须具备可执行权限，`RUN+=` 才能够真正调用它们。
- udev 会合并 `/etc/udev/rules.d/`、`/lib/udev/rules.d/` 等多个目录下的规则文件并按文件名排序处理，仅当文件名相同时前者才会覆盖后者；`/lib/udev/rules.d/61-sd-cards-auto-mount.rules` 是系统自带的 TF/SD 卡自动挂载规则，删除它可以让 TF 卡的挂载行为完全由用户自定义的规则决定。
- U 盘规则与 TF 卡规则可以共存于同一份 `001.rules` 中，但由于两者的移除规则都未限定 `KERNEL`，拔出任意一种设备都会同时触发两个卸载脚本，需要据此评估是否要收紧匹配条件。

# 附录 A usbmount：Buildroot 中的自动挂载工具

## A.1 usbmount 与手工 udev 规则的关系

第 8 章通过在 `/etc/udev/rules.d/001.rules` 中手工编写 `KERNEL==`、`SUBSYSTEM==`、`ACTION==` 规则并配合自行编写的 `usb-add.sh`、`usb-remove.sh` 脚本，实现了 U 盘与 TF 卡插入时的自动挂载。这种做法直观，但存在几个局限：挂载点固定写死为 `/mnt`，无法同时挂载多个设备；文件系统类型固定写死为 `vfat`，遇到 `ext4`、`ntfs` 等其他格式的存储设备需要额外修改脚本；规则文件与脚本文件需要自行维护，缺乏统一的配置入口。

usbmount 是一个独立的、可直接在 Buildroot 中启用的软件包，功能上与第 8 章手工搭建的机制属于同一类问题的两种解决方案：二者都依赖 udev 守护进程监听 uevent、匹配规则、执行外部脚本完成挂载，只是 usbmount 把"匹配规则—探测文件系统—选择挂载点—执行挂载—调用扩展脚本"这一整套流程封装成了成熟的框架，使用者只需要安装并按需修改配置文件，不必再自己编写 udev 规则和挂载脚本。

需要注意的是，usbmount 面向的是 `ID_BUS` 属性为 `usb` 的块设备，即通过 USB 总线枚举出来的存储设备（U 盘、USB 读卡器等）。对于像 i.MX6ULL 这类 SoC 上直接挂在片上 MMC/SD 控制器下的 TF 卡（对应 `mmcblk0p1` 一类设备节点），usbmount 默认的匹配条件不会处理，这部分仍需依赖第 8.6 节手工编写的 `mmcblk` 规则，或者为 TF 卡场景单独编写规则。usbmount 与第 8 章的内容因此并非互斥替代关系，而是分别覆盖了"标准 USB 存储设备"和"任意块设备（含内部 MMC/SD）"两种场景，实际项目中可以按接入方式选择其一，也可以两者共存、分别处理各自负责的设备。

## A.2 在 Buildroot 中启用 usbmount

usbmount 依赖真正的 udev 实现来触发脚本执行，因此前提是 `System configuration` → `/dev management` 已经选择为 `Dynamic using devtmpfs + eudev`（对应第 8.1 节的配置），而不能是 `Dynamic using devtmpfs` 或基于 busybox `mdev` 的方案，因为 usbmount 的运行依赖 udev 规则文件与守护进程的配合，mdev 不具备与之等价的规则匹配和外部程序调用机制。

在 menuconfig 图形化配置界面中，进入：

```text
Target packages --->
    Filesystem and flash utilities --->
        [*] usbmount
```

勾选 `usbmount` 后保存配置并重新编译。usbmount 依赖 `mount`、`blkid` 等工具来探测分区的文件系统类型，Buildroot 在选中 `usbmount` 时会自动选中所需的 `util-linux` 相关组件，无需额外单独勾选。编译完成后，目标文件系统中会新增以下内容：

- 负责挂载与卸载主逻辑的脚本（安装在 `/usr/share/usbmount/` 或等价路径下）；
- 一份 udev 规则文件，负责在 USB 存储设备的分区/整盘出现或消失时调用上述脚本；
- 配置文件 `/etc/usbmount/usbmount.conf`；
- 两个用于扩展的空目录：`/etc/usbmount/mount.d/` 与 `/etc/usbmount/umount.d/`。

## A.3 配置文件 usbmount.conf

`/etc/usbmount/usbmount.conf` 是 usbmount 的核心配置入口，常用配置项包括：

- `FILESYSTEMS`：允许自动挂载的文件系统类型白名单，以空格分隔，例如 `"vfat ext2 ext3 ext4 ntfs hfsplus"`。usbmount 在挂载前会调用 `blkid` 探测分区的实际文件系统类型，只有类型出现在该白名单中才会被挂载，不会像第 8 章的脚本那样把文件系统类型写死为 `vfat`。
- `MOUNTOPTIONS`：传给 `mount` 的通用挂载选项，多个选项以逗号分隔，例如 `"sync,noexec,nodev,noatime"`。
- `FS_MOUNTOPTIONS`：针对特定文件系统追加的挂载选项，例如为 `vfat` 单独指定 `"-fstype=vfat,shortname=mixed,utf8"`，以处理长文件名和编码问题。
- `VERBOSE`：置为 `1` 时会把详细的挂载/卸载过程记录到系统日志中，便于排查规则未生效等问题；默认关闭。

usbmount 的挂载点无需在配置文件中手工指定：设备被识别后，会依次尝试挂载到 `/media/usb0`、`/media/usb1` … 直到 `/media/usb7`（默认最多支持 8 个并发挂载点），每个已经在用的挂载点会被跳过。这一机制直接解决了第 8.4.2 节中固定挂载点 `/mnt` 只能支持单一设备的局限，多个 U 盘同时插入时会被分别挂载到不同的 `/media/usbN` 目录下，互不冲突。

## A.4 扩展机制：mount.d 与 umount.d

除了修改配置文件之外，usbmount 还提供了脚本挂载点式的扩展机制：任何放入 `/etc/usbmount/mount.d/` 目录且具备可执行权限的脚本，都会在一次挂载成功完成之后被自动依次调用；任何放入 `/etc/usbmount/umount.d/` 目录的脚本，则会在对应挂载点被卸载之前被调用。usbmount 在调用这些脚本时会设置若干环境变量供脚本读取，常见的包括：

- `UM_DEVICE`：触发本次事件的设备节点路径，如 `/dev/sda1`；
- `UM_MOUNTPOINT`：本次挂载所使用的挂载点路径，如 `/media/usb0`；
- `UM_FILESYSTEM`：`blkid` 探测到的文件系统类型；
- `UM_MOUNTOPTIONS`：实际传给 `mount` 的挂载选项。

这一机制与第 8 章中直接把业务逻辑写进 `usb-add.sh`（由 `RUN+=` 直接调用）的做法相比，职责划分更清晰：规则匹配、文件系统探测、挂载点分配等通用逻辑由 usbmount 框架统一处理，使用者只需要按约定在 `mount.d`、`umount.d` 目录下编写"设备已挂载/即将卸载时要做什么"的钩子脚本，例如挂载完成后自动同步索引、点亮指示灯，或卸载前先停止正在读写该设备的应用进程。

## A.5 验证

完成 A.2～A.4 节的配置并重新编译烧写系统后，插入 USB 存储设备可观察到：

1. 内核为该设备创建对应的设备节点，并上报 `SUBSYSTEM=block`、`ACTION=add` 的 uevent；
2. udev 守护进程匹配 usbmount 自带的规则，调用其挂载脚本；
3. 挂载脚本通过 `blkid` 探测文件系统类型，确认其位于 `FILESYSTEMS` 白名单内后，将设备挂载到 `/media/usb0`（或下一个空闲的 `/media/usbN`）；
4. 若 `/etc/usbmount/mount.d/` 下存在可执行脚本，会在挂载完成后被依次调用。

执行 `mount` 命令或 `ls /media/usb0` 可以确认挂载结果；若 `VERBOSE` 已置为 `1`，还可以在系统日志中看到 usbmount 记录的详细执行过程。拔出设备时，`/media/usb0` 对应的挂载点会被自动卸载，`/etc/usbmount/umount.d/` 下的脚本先于卸载动作被调用。

## A.6 小结

- usbmount 与第 8 章手工编写的 udev 规则、挂载脚本解决的是同一类问题，二者都依赖 eudev 监听 uevent 并执行外部脚本完成挂载，因此 usbmount 同样要求 `/dev management` 配置为 `Dynamic using devtmpfs + eudev`。
- usbmount 默认只处理 `ID_BUS` 为 `usb` 的存储设备，SoC 片上 MMC/SD 控制器下的 TF 卡不在其默认覆盖范围内，第 8.6 节的 `mmcblk` 规则仍然需要单独维护。
- 在 Buildroot 的 `Target packages` → `Filesystem and flash utilities` 中勾选 `usbmount` 即可启用，相关的 `util-linux` 依赖会被自动选中。
- `/etc/usbmount/usbmount.conf` 通过 `FILESYSTEMS`、`MOUNTOPTIONS`、`FS_MOUNTOPTIONS`、`VERBOSE` 等配置项，解决了第 8 章示例中文件系统类型写死、缺少调试日志等局限。
- 设备默认按 `/media/usb0` ～ `/media/usb7` 顺序分配挂载点，天然支持多个存储设备同时挂载，不再需要像第 8.4.2 节那样自行处理挂载点冲突问题。
- `/etc/usbmount/mount.d/`、`/etc/usbmount/umount.d/` 提供了标准化的挂载后/卸载前钩子脚本机制，通过 `UM_DEVICE`、`UM_MOUNTPOINT`、`UM_FILESYSTEM` 等环境变量向钩子脚本传递上下文，无需像第 8 章那样把所有逻辑直接写在 udev 规则调用的脚本中。


