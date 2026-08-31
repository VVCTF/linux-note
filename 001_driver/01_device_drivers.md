# 1 字符设备驱动

## 1.1 字符设备注册

### 1.1.1 申请设备号

字符设备驱动注册的第一步，是向内核申请一个或一段设备号（dev_t）。

常见函数有两个：

1. 自动分配主设备号（推荐）

```c
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```

参数说明：

- `dev`：输出参数，返回分配到的设备号
- `baseminor`：起始次设备号，通常填 0
- `count`：需要连续申请的设备个数
- `name`：设备名，显示在 `/proc/devices`

返回值说明：

- `0`：成功

- `< 0`：失败（负错误码）
2. 指定主设备号分配

```c
int register_chrdev_region(dev_t from, unsigned count, const char *name);
```

参数说明：

- `from`：起始设备号，通常由 `MKDEV(major, minor)` 生成
- `count`：设备个数
- `name`：设备名

### 1.1.2 初始化 cdev

申请到设备号后，需要初始化 `struct cdev`，并把它和文件操作集 `file_operations` 绑定。

```c
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```

### 1.1.3 设置 owner 字段

初始化后，通常还会设置：

```c
cdev.owner = THIS_MODULE;
```

这样在设备被打开期间，模块引用计数会被正确维护。

### 1.1.4 将 cdev 添加到内核

完成初始化后，需要把 cdev 注册到内核，建立设备号到 fops 的映射。

```c
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

参数说明：

- p：已初始化的 cdev
- dev：起始设备号
- count：设备个数

返回值说明：

- 0：成功
- 小于 0：失败

调用时机：

- cdev_init 之后
- class_create 和 device_create 之前

---

### 1.1.5 创建设备类 class

设备类用于在 sysfs 中建立目录，方便管理设备节点。     

新内核常用：

```c
struct class *class_create(const char *name);
```

示例：

```c
struct class *my_class;
my_class = class_create("my_chrdev_class");
if (IS_ERR(my_class))
        return PTR_ERR(my_class);
```

补充：

- 老内核可能是 class_create(THIS_MODULE, name)
- 新旧内核接口有差异，需要按目标内核版本写

调用时机：

- cdev_add 成功后调用

---

### 1.1.6 创建设备节点

创建具体设备节点，用户空间才能通过 /dev/xxx 打开并访问该驱动。

```c
struct device *device_create(struct class *class, struct device *parent,
                             dev_t devt, void *drvdata, const char *fmt, ...);
```

示例：

```c
struct device *my_device;
my_device = device_create(my_class, NULL, dev_num, NULL, "my_chrdev0");
if (IS_ERR(my_device))
    return PTR_ERR(my_device);
```

参数说明：

- class：设备类对象
- parent：父设备，通常填 NULL
- devt：设备号
- drvdata：驱动私有数据，简单驱动通常填 NULL
- fmt：设备名格式字符串

调用时机：

- class_create 成功后调用

---

### 1.1.7 错误回滚（非常关键）

字符设备注册是分步完成的，任意一步失败都要回滚已成功步骤。

推荐回滚顺序：

1. device_create 失败：先 class_destroy
2. class_create 失败：先 cdev_del
3. cdev_add 失败：unregister_chrdev_region

核心原则：

- 谁先创建，谁后销毁
- 销毁顺序与创建顺序相反

---

### 1.1.8 模块退出时注销资源

退出函数里必须按逆序释放资源，避免内核资源泄漏。

```c
static void __exit mydrv_exit(void)
{
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
}
```

对应关系：

- device_create 对应 device_destroy
- class_create 对应 class_destroy
- cdev_add 对应 cdev_del
- alloc_chrdev_region/register_chrdev_region 对应 unregister_chrdev_region

---

### 1.1.9 声明模块入口与出口

```c
module_init(mydrv_init);
module_exit(mydrv_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("your_name");
MODULE_DESCRIPTION("simple char device driver");
```

作用说明：

- module_init：指定加载模块时执行的注册函数
- module_exit：指定卸载模块时执行的注销函数
- MODULE_LICENSE：许可证声明，不写可能影响符号导出使用

## 1.2 文件私有数据

### 1.2.1 什么是文件私有数据

在内核中，每次用户进程 `open` 一个设备文件，都会创建一个 `struct file` 对象。`struct file` 里有一个通用指针成员：

```c
void *private_data;
```

这个成员就是文件私有数据，用来挂接驱动自己的上下文信息。它的核心作用是：

- 让同一套 `read/write/ioctl` 代码拿到当前打开实例对应的数据
- 避免在驱动中使用大量全局变量
- 支持同一驱动同时服务多个设备或多个打开实例

---

### 1.2.2 单设备模式：直接保存全局设备指针

**适用场景**：驱动只有一个设备实例（如你当前的 `chrdev.c`）

**第一步：定义设备结构体**

```c
struct my_dev {
    dev_t dev_num;              // 设备号
    int major;                  // 主设备号
    int minor;                  // 次设备号
    struct cdev cdev;           // 字符设备对象
    struct class *cls;          // 设备类
    struct device *device;      // 设备对象
    char buf[128];              // 设备数据缓冲区
    struct mutex lock;          // 互斥锁（并发保护）
};

static struct my_dev my_device;  // 全局声明唯一的设备实例
```

**第二步：在 open 中保存设备指针**

```c
static int my_open(struct inode *inode, struct file *file)
{
    // 将全局设备指针挂到 file->private_data
    file->private_data = &my_device;

    printk(KERN_INFO "Device opened\n");
    return 0;
}
```

**第三步：在 read 中取出并使用**

```c
static ssize_t my_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;  // 取出设备结构体指针
    int ret;

    mutex_lock(&dev->lock);

    // 从设备缓冲区复制数据到用户空间
    ret = copy_to_user(buf, dev->buf, count);

    mutex_unlock(&dev->lock);

    return count;
}
```

**第四步：在 write 中取出并使用**

```c
static ssize_t my_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;  // 取出设备结构体指针
    int ret;

    if (count > sizeof(dev->buf))
        count = sizeof(dev->buf);

    mutex_lock(&dev->lock);

    // 从用户空间复制数据到设备缓冲区
    ret = copy_from_user(dev->buf, buf, count);

    mutex_unlock(&dev->lock);

    return count;
}
```

**第五步：在 release 中清理**

```c
static int my_release(struct inode *inode, struct file *file)
{
    struct my_dev *dev = file->private_data;

    printk(KERN_INFO "Device closed\n");
    // 进行清理工作（如有必要）

    return 0;
}
```

---

### 1.2.3 多设备模式：用 container_of 反查设备结构体

**适用场景**：驱动支持多个设备实例（如 `/dev/mydev0`、`/dev/mydev1`），共享同一套 `file_operations`

**问题背景**：

单设备模式中，`open` 往往直接 `file->private_data = &my_device`。但多设备模式下，`open` 需要根据当前 inode 对应的 `cdev`，反查出“当前这个 fd 属于哪个设备实例”。

最常用的方法就是 `container_of`。

```c
#define MYDEV_COUNT 2

struct my_dev {
    dev_t dev_num;
    int major;
    int minor;
    struct cdev cdev;
    char buf[128];
    struct mutex lock;
};

static struct my_dev my_devices[MYDEV_COUNT];
static struct class *my_class;

static int my_open(struct inode *inode, struct file *file)
{
    struct my_dev *dev;

    /* 通过 inode->i_cdev 反查所属设备对象 */
    dev = container_of(inode->i_cdev, struct my_dev, cdev);
    file->private_data = dev;

    printk(KERN_INFO "open minor=%d\n", dev->minor);
    return 0;
}

static ssize_t my_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;

    if (count > sizeof(dev->buf))
        count = sizeof(dev->buf);

    mutex_lock(&dev->lock);
    if (copy_to_user(buf, dev->buf, count)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    mutex_unlock(&dev->lock);

    return count;
}

static ssize_t my_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;

    if (count > sizeof(dev->buf))
        count = sizeof(dev->buf);

    mutex_lock(&dev->lock);
    if (copy_from_user(dev->buf, buf, count)) {
        mutex_unlock(&dev->lock);
        return -EFAULT;
    }
    mutex_unlock(&dev->lock);

    return count;
}

static int my_release(struct inode *inode, struct file *file)
{
    return 0;
}

static const struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .read = my_read,
    .write = my_write,
    .release = my_release,
};

static int __init my_init(void)
{
    int i, ret;
    dev_t dev_num;

    ret = alloc_chrdev_region(&dev_num, 0, MYDEV_COUNT, "mydev");
    if (ret < 0)
        return ret;

    for (i = 0; i < MYDEV_COUNT; i++) {
        my_devices[i].dev_num = MKDEV(MAJOR(dev_num), i);
        my_devices[i].major = MAJOR(dev_num);
        my_devices[i].minor = i;

        mutex_init(&my_devices[i].lock);
        cdev_init(&my_devices[i].cdev, &my_fops);
        my_devices[i].cdev.owner = THIS_MODULE;

        ret = cdev_add(&my_devices[i].cdev, my_devices[i].dev_num, 1);
        if (ret)
            goto err_cdev;
    }

    my_class = class_create("mydev_class");
    if (IS_ERR(my_class)) {
        ret = PTR_ERR(my_class);
        goto err_cdev;
    }

    for (i = 0; i < MYDEV_COUNT; i++) {
        if (IS_ERR(device_create(my_class, NULL, my_devices[i].dev_num, NULL, "mydev%d", i))) {
            ret = -EINVAL;
            goto err_dev;
        }
    }

    return 0;

err_dev:
    while (--i >= 0)
        device_destroy(my_class, my_devices[i].dev_num);
    class_destroy(my_class);
err_cdev:
    for (i = 0; i < MYDEV_COUNT; i++)
        cdev_del(&my_devices[i].cdev);
    unregister_chrdev_region(dev_num, MYDEV_COUNT);
    return ret;
}

static void __exit my_exit(void)
{
    int i;

    for (i = 0; i < MYDEV_COUNT; i++) {
        device_destroy(my_class, my_devices[i].dev_num);
        cdev_del(&my_devices[i].cdev);
    }

    class_destroy(my_class);
    unregister_chrdev_region(MKDEV(my_devices[0].major, 0), MYDEV_COUNT);
}

module_init(my_init);
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

这个示例创建了 `/dev/mydev0` 和 `/dev/mydev1`，用户可以同时打开两个设备，驱动会通过 `container_of` 自动区分。

### 1.2.4 小结

多设备模式的关键是：

1. 每个设备实例都有自己的 `struct cdev` 和私有状态
2. `open` 用 `container_of(inode->i_cdev, struct my_dev, cdev)` 反查设备实例
3. 把反查结果放入 `file->private_data`，后续 `read/write` 就不用再判断设备号

---

### 1.2.5 使用注意事项

1. **类型转换**：`private_data` 是 `void *`，取出时要做正确类型转换
2. **释放时机**：若指向动态分配内存（如形式3），要在 `release` 中成对释放
3. **并发保护**：多线程/多进程并发访问时，要配合锁（mutex/spinlock）保护共享数据
4. **悬空指针**：不要把已释放的地址留在 `private_data` 中
5. **错误处理**：在 `open` 中分配资源失败时要及时返回错误码，不要留下半初始化状态
6. **多设备调试**：在多设备模式中，用 printk 打印设备信息，便于验证 container_of 是否反查正确

---

## 1.3 杂项设备驱动

### 1.3.1 什么是杂项设备

杂项设备（misc device）是 Linux 内核提供的一种简化版字符设备，所有杂项设备共享同一个主设备号 **10**，通过不同的次设备号区分。

相比普通字符设备驱动，杂项设备不需要手动：

- 申请设备号
- 初始化 cdev
- 创建 class
- 创建 device

这些工作由内核的 misc 子系统统一处理，驱动只需要填一个 `struct miscdevice` 结构体并调用一个注册函数。

---

### 1.3.2 与字符设备驱动的对比

| 对比项  | 字符设备驱动                                                              | 杂项设备驱动                 |
| ---- | ------------------------------------------------------------------- | ---------------------- |
| 主设备号 | 动态或静态分配                                                             | 固定为 10                 |
| 次设备号 | 手动管理                                                                | 自动分配（或手动指定）            |
| 注册步骤 | 6步（申请设备号、cdev_init、cdev_add、class_create、device_create、module_init） | 1步（misc_register）      |
| 设备类  | 需要手动创建                                                              | 自动创建在 /sys/class/misc/ |
| 设备节点 | 需要手动创建                                                              | 自动创建                   |
| 代码量  | 多                                                                   | 少                      |
| 适用场景 | 需要精细控制设备号、多设备实例                                                     | 简单单一功能设备、快速原型          |

---

### 1.3.3 核心数据结构 struct miscdevice

```c
// 定义在 include/linux/miscdevice.h
struct miscdevice {
    int minor;                         // 次设备号，填 MISC_DYNAMIC_MINOR 自动分配
    const char *name;                  // 设备名，对应 /dev/name
    const struct file_operations *fops; // 文件操作集
    struct list_head list;             // 内核链表，不需要手动填
    struct device *parent;             // 父设备，通常填 NULL
    struct device *this_device;        // 注册完成后内核填入，驱动可用
    const struct attribute_group **groups; // sysfs 属性组，通常填 NULL
    const char *nodename;              // 自定义 /dev 节点名，通常用 name 即可
    umode_t mode;                      // 设备节点权限，0 表示默认
};
```

关键字段只有三个需要驱动填写：

- `minor`：填 `MISC_DYNAMIC_MINOR` 自动分配次设备号，或指定固定值
- `name`：设备名，会直接对应 `/dev/<name>`
- `fops`：文件操作集，和字符设备一样

**MISC_DYNAMIC_MINOR 说明**：

```c
// 定义在 include/linux/miscdevice.h
#define MISC_DYNAMIC_MINOR  255
```

这个宏的值是 `255`，是一个约定好的特殊值。当 `minor` 字段填写 `255` 时，`misc_register` 内部会调用 `find_first_zero_bit` 从空闲的次设备号中自动挑选一个可用值分配给该设备。

注册完成后，实际分配到的次设备号会回写到 `my_misc_dev.minor`，可以通过它查看：

```c
printk(KERN_INFO "minor = %d\n", my_misc_dev.minor);
// 或者加载后查看
// cat /proc/misc
```

如果你有特殊需求，想固定次设备号，直接填入具体数字即可（0~254 范围），但要保证不和系统中其他杂项设备冲突：

```c
// 固定次设备号为 100
static struct miscdevice my_misc_dev = {
    .minor = 100,
    .name  = "my_misc_dev",
    .fops  = &misc_fops,
};
```

通常推荐使用 `MISC_DYNAMIC_MINOR`，让内核自动管理，避免冲突。

---

### 1.3.4 注册与注销函数

**注册**：

```c
int misc_register(struct miscdevice *misc);
```

- 调用后自动完成：申请次设备号、创建设备节点、在 sysfs 中创建条目
- 返回 0 表示成功，负数为错误码

**注销**：

```c
void misc_deregister(struct miscdevice *misc);
```

- 自动完成设备节点销毁、次设备号释放

---

### 1.3.5 注册流程

字符设备驱动需要 6 步，杂项设备驱动只需 3 步：

```
1. 定义 struct miscdevice（填 minor、name、fops）
         ↓
2. 在 init 中调用 misc_register()
         ↓
3. 在 exit 中调用 misc_deregister()
```

---

### 1.3.6 完整示例（含 file->private_data）

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/mutex.h>

#define BUF_SIZE 128

// 杂项设备的私有数据结构
struct misc_dev_data {
    char buf[BUF_SIZE];
    struct mutex lock;
};

static struct misc_dev_data misc_data;

static int misc_dev_open(struct inode *inode, struct file *file)
{
    // 在打开时保存私有数据指针
    file->private_data = &misc_data;

    printk(KERN_INFO "misc device opened\n");
    return 0;
}

static int misc_dev_release(struct inode *inode, struct file *file)
{
    printk(KERN_INFO "misc device released\n");
    return 0;
}

static ssize_t misc_dev_read(struct file *file, char __user *buf,
                              size_t count, loff_t *ppos)
{
    // 取出之前保存的私有数据
    struct misc_dev_data *data = file->private_data;
    int ret;

    if (count > BUF_SIZE)
        count = BUF_SIZE;

    mutex_lock(&data->lock);

    ret = copy_to_user(buf, data->buf, count);

    mutex_unlock(&data->lock);

    if (ret < 0)
        return -EFAULT;

    printk(KERN_INFO "misc device read %zu bytes\n", count);
    return count;
}

static ssize_t misc_dev_write(struct file *file, const char __user *buf,
                               size_t count, loff_t *ppos)
{
    // 取出之前保存的私有数据
    struct misc_dev_data *data = file->private_data;
    int ret;

    if (count > BUF_SIZE)
        count = BUF_SIZE;

    mutex_lock(&data->lock);

    ret = copy_from_user(data->buf, buf, count);

    mutex_unlock(&data->lock);

    if (ret < 0)
        return -EFAULT;

    printk(KERN_INFO "misc device write %zu bytes\n", count);
    return count;
}

static const struct file_operations misc_fops = {
    .owner   = THIS_MODULE,
    .open    = misc_dev_open,
    .release = misc_dev_release,
    .read    = misc_dev_read,
    .write   = misc_dev_write,
};

// 定义杂项设备
static struct miscdevice my_misc_dev = {
    .minor = MISC_DYNAMIC_MINOR,  // 自动分配次设备号
    .name  = "my_misc_dev",       // 对应 /dev/my_misc_dev
    .fops  = &misc_fops,
};

static int __init misc_dev_init(void)
{
    int ret;

    // 初始化私有数据
    mutex_init(&misc_data.lock);
    memset(misc_data.buf, 0, BUF_SIZE);

    ret = misc_register(&my_misc_dev);
    if (ret) {
        printk(KERN_ERR "misc_register failed: %d\n", ret);
        return ret;
    }

    printk(KERN_INFO "misc device registered, minor: %d\n",
           my_misc_dev.minor);
    return 0;
}

static void __exit misc_dev_exit(void)
{
    misc_deregister(&my_misc_dev);
    printk(KERN_INFO "misc device unregistered\n");
}

module_init(misc_dev_init);
module_exit(misc_dev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Driver Author");
MODULE_DESCRIPTION("Misc device driver with private_data");
```

加载后效果：

```bash
sudo insmod my_misc.ko
echo "Hello from user" > /dev/my_misc_dev
cat /dev/my_misc_dev
dmesg | tail -20  # 查看打印信息
```

---

### 1.3.7 何时选择杂项设备而不是字符设备

选杂项设备：

- 功能简单，只需要一个设备节点
- 快速验证驱动逻辑，不想写大量注册代码
- 嵌入式中常见的简单外设（LED、按键、传感器读取等）

选字符设备：

- 需要多个设备实例（如 `/dev/mydev0`、`/dev/mydev1`）
- 需要自定义主设备号
- 需要精细控制设备号分配策略

# 2 并发与竞争

Linux 驱动中的并发问题，本质是多个执行流同时访问共享资源。本章先讲 API，再给场景和案例。

并发来源：

- 多进程同时访问同一设备节点
- 多线程同时访问同一个文件描述符或多个文件描述符
- 中断上下文和进程上下文同时访问共享状态
- 多核 CPU 并行执行同一段驱动代码

## 2.1 原子操作（Atomic Operation）

### 2.1.1 核心 API

1. 定义与初始化

```c
atomic_t cnt = ATOMIC_INIT(0);
atomic64_t cnt64 = ATOMIC64_INIT(0);
```

2. 读写

```c
int v = atomic_read(&cnt);
atomic_set(&cnt, 10);
```

3. 加减

```c
atomic_inc(&cnt);
atomic_dec(&cnt);
atomic_add(5, &cnt);
atomic_sub(2, &cnt);
```

4. 读改写并返回新值

```c
int v1 = atomic_inc_return(&cnt);
int v2 = atomic_dec_return(&cnt);
int v3 = atomic_add_return(3, &cnt);
```

API 说明：

- 参数都是 `atomic_t *`
- 返回值通常是新的计数值或当前值
- 可以在中断上下文和进程上下文使用

### 2.1.2 使用场景

- 打开计数
- 错误计数
- 引用计数

### 2.1.3 最小案例

```c
struct my_dev {
    atomic_t open_cnt;
};

static int my_open(struct inode *inode, struct file *file)
{
    struct my_dev *dev = file->private_data;
    int cnt = atomic_inc_return(&dev->open_cnt);
    printk(KERN_INFO "opened, open_cnt=%d\n", cnt);
    return 0;
}

static int my_release(struct inode *inode, struct file *file)
{
    struct my_dev *dev = file->private_data;
    int cnt = atomic_dec_return(&dev->open_cnt);
    printk(KERN_INFO "released, open_cnt=%d\n", cnt);
    return 0;
}
```

### 2.1.4 注意事项

- 原子变量只保证单变量操作原子，不保证多变量一致性
- 无法替代临界区锁，保护复杂结构请用 spinlock/mutex

## 2.2 自旋锁（Spinlock）

### 2.2.1 核心 API

1. 定义与初始化

```c
spinlock_t lock;
spin_lock_init(&lock);
```

2. 普通加解锁

```c
spin_lock(&lock);
/* critical section */
spin_unlock(&lock);
```

3. 关本地中断版（最常用）

```c
unsigned long flags;

spin_lock_irqsave(&lock, flags);
/* critical section */
spin_unlock_irqrestore(&lock, flags);
```

API 说明：

- `flags` 用于保存并恢复中断状态
- `spin_lock_irqsave` 适合“该锁既在中断用，也在进程上下文用”的场景

### 2.2.2 使用场景

- ISR（中断处理）和 read/write 路径共享状态
- 临界区极短且绝不睡眠

### 2.2.3 最小案例

```c
struct my_dev {
    spinlock_t irq_lock;
    int data_ready;
    unsigned int irq_count;
};

static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    unsigned long flags;

    spin_lock_irqsave(&dev->irq_lock, flags);
    dev->data_ready = 1;
    dev->irq_count++;
    spin_unlock_irqrestore(&dev->irq_lock, flags);

    return IRQ_HANDLED;
}
```

### 2.2.4 注意事项

- 持有自旋锁期间不能调用可能睡眠的 API，例如 `copy_to_user`、`copy_from_user`、`msleep`
- 加锁和解锁函数必须严格配对
- 多把锁必须固定顺序，防止 ABBA 死锁

自旋锁死锁示例：

```text
CPU0: spin_lock(A) -> spin_lock(B)
CPU1: spin_lock(B) -> spin_lock(A)

双方互等，CPU 忙等，系统表现为卡死。
```

## 2.3 信号量（Semaphore）

### 2.3.1 核心 API

1. 定义与初始化

```c
struct semaphore sem;
sema_init(&sem, 1);   /* 1 表示二值信号量，N 表示计数信号量 */
```

2. 获取

```c
down(&sem);                 /* 不可中断等待 */
if (down_interruptible(&sem))
    return -ERESTARTSYS;    /* 可中断等待 */
```

3. 释放

```c
up(&sem);
```

API 说明：

- `sema_init` 第二个参数是并发配额
- `down_interruptible` 被信号打断时返回非 0

### 2.3.2 使用场景

- 限流：同时最多允许 N 个线程进入
- 设备后端有固定并发容量（如最多 2 路 DMA 通道）

### 2.3.3 最小案例

```c
struct my_dev {
    struct semaphore slots;
};

static int __init my_init(void)
{
    sema_init(&my_device.slots, 2);
    return 0;
}

static long my_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct my_dev *dev = file->private_data;

    if (down_interruptible(&dev->slots))
        return -ERESTARTSYS;

    /* 耗时操作 */

    up(&dev->slots);
    return 0;
}
```

### 2.3.4 注意事项

- 获取成功后必须保证所有退出路径都 `up`
- 推荐用 `goto out_up` 统一释放资源
- 如果只需要互斥，优先用 `mutex`，可读性更好

## 2.4 互斥锁（Mutex）

### 2.4.1 核心 API

1. 定义与初始化

```c
struct mutex lock;
mutex_init(&lock);
```

2. 获取

```c
mutex_lock(&lock);
if (mutex_lock_interruptible(&lock))
    return -ERESTARTSYS;
```

3. 释放

```c
mutex_unlock(&lock);
```

4. 非阻塞尝试

```c
if (!mutex_trylock(&lock))
    return -EBUSY;
```

API 说明：

- 仅可在进程上下文使用
- 可用于可能睡眠的临界区

### 2.4.2 使用场景

- 字符设备 read/write/ioctl 对共享缓冲区加锁
- 需要调用 `copy_to_user`/`copy_from_user` 的临界区

### 2.4.3 最小案例

```c
struct my_dev {
    char buf[128];
    size_t len;
    struct mutex buf_lock;
};

static ssize_t my_write(struct file *file, const char __user *ubuf,
                        size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;

    if (count > sizeof(dev->buf))
        count = sizeof(dev->buf);

    if (mutex_lock_interruptible(&dev->buf_lock))
        return -ERESTARTSYS;

    if (copy_from_user(dev->buf, ubuf, count)) {
        mutex_unlock(&dev->buf_lock);
        return -EFAULT;
    }
    dev->len = count;

    mutex_unlock(&dev->buf_lock);
    return count;
}
```

### 2.4.4 注意事项

- 中断上下文禁止使用 mutex
- 同线程重复锁同一 mutex 会死锁
- 错误路径忘记 unlock 会导致永久阻塞

## 2.5 四种机制对比

| 机制        | 可睡眠 | 中断上下文可用 | 典型用途     |
| --------- | --- | ------- | -------- |
| atomic    | 否   | 是       | 单变量计数    |
| spinlock  | 否   | 是       | 中断共享短临界区 |
| semaphore | 是   | 否       | 资源并发限流   |
| mutex     | 是   | 否       | 进程上下文互斥  |

## 2.6 API 选型步骤

1. 只改一个计数器，用 `atomic`
2. 中断会访问同一数据，用 `spinlock`
3. 需要同时允许 N 个访问者，用 `semaphore`
4. 普通读写并发保护且可能睡眠，用 `mutex`

## 2.7 当前驱动落地建议

建议在你的字符设备和杂项设备代码中组合使用：

- 缓冲区 `buf[]` 用 `mutex`
- 打开计数用 `atomic_t`
- 若后续加入中断，再加 `spinlock_t`

```c
struct my_dev {
    char buf[128];
    struct mutex buf_lock;
    atomic_t open_cnt;
    spinlock_t irq_lock;
    int irq_state;
};
```

# 3 IO 模型

本章按“概念 -> 驱动实现 -> 多路复用 -> 用户态案例”的顺序组织，重点回答两个问题：

- IO 行为如何分类
- 阻塞与非阻塞到底由谁决定、在驱动中如何实现

## 3.1 IO 分类

### 3.1.1 什么是 IO

IO（Input/Output）是数据在用户空间、内核空间与设备之间的传输过程。在字符设备驱动中，常见入口是：

- `read`
- `write`
- `ioctl`
- `poll/select/epoll`（事件等待）

一次 IO 调用通常包含：发起系统调用 -> 进入驱动回调 -> 检查条件 -> 传输数据或等待条件成立。

### 3.1.2 同步 IO

同步 IO 的定义：调用线程自己等待本次 IO 完成后再返回。

典型表现：线程执行 `read` 后，要么拿到数据返回，要么在内核中等待，直到数据就绪后返回。

### 3.1.3 异步 IO

异步 IO 的定义：发起 IO 后，调用线程不必一直等待；完成后通过通知机制获知结果。

在驱动实践中，常见工程形态是：非阻塞 fd + 事件通知（`select/poll/epoll`）+ 条件满足后再 `read/write`。

### 3.1.4 阻塞 IO

阻塞 IO 的定义：条件不满足时，调用线程睡眠等待。

示例：`read` 发现无数据，线程进入睡眠；数据到来后被唤醒继续执行。

### 3.1.5 非阻塞 IO

非阻塞 IO 的定义：条件不满足时，调用立即返回，不睡眠。

常见返回：`-EAGAIN`（或 `-EWOULDBLOCK`）。

### 3.1.6 同步/异步 与 阻塞/非阻塞的关系

两组概念维度不同：

| 维度       | 关注点       |
| -------- | --------- |
| 同步 / 异步  | 谁负责等待结果完成 |
| 阻塞 / 非阻塞 | 本次调用是否睡眠  |

## 3.2 驱动如何实现阻塞与非阻塞

驱动是阻塞/非阻塞行为的直接执行者。用户态通过打开方式表达意图，驱动在回调中落实策略。

### 3.2.1 谁决定阻塞与非阻塞

- 用户态：通过 `open(..., O_NONBLOCK)` 指定非阻塞意图
- 驱动态：在 `read/write` 中检查 `file->f_flags`，决定“立即返回”还是“进入等待”

典型判断：

```c
if (!data_ready) {
    if (file->f_flags & O_NONBLOCK)
        return -EAGAIN;  /* 非阻塞：立即返回 */

    /* 阻塞：进入等待 */
}
```

### 3.2.2 等待队列实现阻塞（核心）

等待队列用于让线程在条件不满足时睡眠，并在条件满足时被唤醒。

常用 API：

- `init_waitqueue_head`
- `wait_event_interruptible`
- `wake_up_interruptible`

最小模板：

```c
struct my_dev {
    char buf[128];
    int data_ready;
    wait_queue_head_t read_queue;
    struct mutex lock;
};

static ssize_t my_read(struct file *file, char __user *buf,
                       size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;

    if (!dev->data_ready) {
        if (file->f_flags & O_NONBLOCK)
            return -EAGAIN;

        if (wait_event_interruptible(dev->read_queue, dev->data_ready))
            return -ERESTARTSYS;
    }

    if (copy_to_user(buf, dev->buf, count))
        return -EFAULT;

    dev->data_ready = 0;
    return count;
}

static ssize_t my_write(struct file *file, const char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_dev *dev = file->private_data;

    if (copy_from_user(dev->buf, buf, count))
        return -EFAULT;

    dev->data_ready = 1;                      /* 先更新条件 */
    wake_up_interruptible(&dev->read_queue);  /* 再唤醒 */
    return count;
}
```

### 3.2.3 实现要点

- 等待队列只负责“睡眠/唤醒”，不保存业务条件
- 唤醒顺序必须是“先改状态，再唤醒”
- 非阻塞路径不能调用等待接口

## 3.3 多路复用

多路复用用于一个线程同时等待多个 fd 事件，避免每个 fd 启一个阻塞线程。

### 3.3.1 驱动如何对接 select/poll/epoll

驱动侧不需要三套实现，只需 `.poll` 一套实现。

```c
static const struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .read  = my_read,
    .write = my_write,
    .poll  = my_poll,
};
```

`.poll` 原型：

```c
unsigned int my_poll(struct file *file, poll_table *wait);
```

典型实现：

```c
static unsigned int my_poll(struct file *file, poll_table *wait)
{
    struct my_dev *dev = file->private_data;
    unsigned int mask = 0;

    poll_wait(file, &dev->read_queue, wait);

    if (dev->data_ready)
        mask |= POLLIN | POLLRDNORM;

    return mask;
}
```

### 3.3.2 select / poll / epoll 区别

| 机制     | 特点                            | 适用场景      |
| ------ | ----------------------------- | --------- |
| select | 位图 + 上限（`FD_SETSIZE`）+ 每次重置集合 | 小规模、兼容性优先 |
| poll   | 数组 + 线性扫描                     | 中等规模      |
| epoll  | 事件注册与等待分离 + 就绪队列              | 大规模、高并发   |

结论：用户态 API 不同，但驱动接入点统一为 `.poll`。

### 3.3.3 事件掩码速查

常用掩码：

- `POLLIN` / `POLLRDNORM`：可读
- `POLLOUT` / `POLLWRNORM`：可写
- `POLLERR`：错误
- `POLLHUP`：挂断
- `POLLPRI`：高优先级数据

对应关系：

| 语义  | 驱动返回      | poll 检查             | epoll 检查            | select 对应   |
| --- | --------- | ------------------- | ------------------- | ----------- |
| 可读  | `POLLIN`  | `revents & POLLIN`  | `events & EPOLLIN`  | `readfds`   |
| 可写  | `POLLOUT` | `revents & POLLOUT` | `events & EPOLLOUT` | `writefds`  |
| 异常  | `POLLPRI` | `revents & POLLPRI` | `events & EPOLLPRI` | `exceptfds` |

### 3.3.4 驱动实现注意事项

- `.poll` 不得阻塞；只登记等待队列并返回当前掩码
- 掩码必须与真实状态一致，避免误报
- 共享状态与 `read/write/poll` 并发访问时要加锁

## 3.4 用户态多路复用案例

### 3.4.1 select 案例

```c
int fd = open("/dev/mydev", O_RDWR | O_NONBLOCK);
fd_set rfds;
struct timeval tv;

FD_ZERO(&rfds);
FD_SET(fd, &rfds);
tv.tv_sec = 5;
tv.tv_usec = 0;

int ret = select(fd + 1, &rfds, NULL, NULL, &tv);
if (ret > 0 && FD_ISSET(fd, &rfds)) {
    char buf[128];
    read(fd, buf, sizeof(buf));
}
```

### 3.4.2 poll 案例

```c
int fd = open("/dev/mydev", O_RDWR | O_NONBLOCK);
struct pollfd pfd;

pfd.fd = fd;
pfd.events = POLLIN;

int ret = poll(&pfd, 1, 5000);
if (ret > 0 && (pfd.revents & POLLIN)) {
    char buf[128];
    read(fd, buf, sizeof(buf));
}
```

### 3.4.3 epoll 案例

```c
int fd = open("/dev/mydev", O_RDWR | O_NONBLOCK);
int epfd = epoll_create1(0);
struct epoll_event ev, events[16];

ev.events = EPOLLIN;
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

int n = epoll_wait(epfd, events, 16, 5000);
for (int i = 0; i < n; i++) {
    if (events[i].data.fd == fd && (events[i].events & EPOLLIN)) {
        char buf[128];
        read(fd, buf, sizeof(buf));
    }
}
```

### 3.4.4 socket 服务端多连接案例

本节给出三种用户态服务端模板，只关注事件监听与连接处理：

- `select`：维护 `fd_set`，每轮复制集合并全量扫描
- `poll`：维护 `pollfd` 数组，每轮线性扫描
- `epoll`：`epoll_ctl` 注册连接，`epoll_wait` 获取就绪队列

#### select 模板

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

int main(void)
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(9000),
        .sin_addr.s_addr = INADDR_ANY,
    };
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 8);

    fd_set allset, rset;
    FD_ZERO(&allset);
    FD_SET(lfd, &allset);
    int maxfd = lfd;

    while (1) {
        rset = allset;
        int nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
        if (nready < 0)
            continue;

        if (FD_ISSET(lfd, &rset)) {
            int cfd = accept(lfd, NULL, NULL);
            FD_SET(cfd, &allset);
            if (cfd > maxfd)
                maxfd = cfd;
            write(cfd, "welcome from select\n", 20);
            if (--nready == 0)
                continue;
        }

        for (int fd = lfd + 1; fd <= maxfd; fd++) {
            if (!FD_ISSET(fd, &rset))
                continue;
            char buf[128];
            int n = read(fd, buf, sizeof(buf));
            if (n <= 0) {
                close(fd);
                FD_CLR(fd, &allset);
            } else {
                write(fd, "event from select\n", 18);
            }
        }
    }
}
```

#### poll 模板

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <poll.h>

int main(void)
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(9000),
        .sin_addr.s_addr = INADDR_ANY,
    };
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 8);

    struct pollfd pfds[1024];
    pfds[0].fd = lfd;
    pfds[0].events = POLLIN;
    int nfds = 1;

    while (1) {
        int nready = poll(pfds, nfds, -1);
        if (nready < 0)
            continue;

        if (pfds[0].revents & POLLIN) {
            int cfd = accept(lfd, NULL, NULL);
            pfds[nfds].fd = cfd;
            pfds[nfds].events = POLLIN;
            pfds[nfds].revents = 0;
            nfds++;
            write(cfd, "welcome from poll\n", 18);
            if (--nready == 0)
                continue;
        }

        for (int i = 1; i < nfds; i++) {
            if (!(pfds[i].revents & POLLIN))
                continue;
            char buf[128];
            int n = read(pfds[i].fd, buf, sizeof(buf));
            if (n <= 0) {
                close(pfds[i].fd);
                for (int j = i; j < nfds - 1; j++)
                    pfds[j] = pfds[j + 1];
                nfds--;
                i--;
            } else {
                write(pfds[i].fd, "event from poll\n", 16);
            }
        }
    }
}
```

#### epoll 模板

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

int main(void)
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(9000),
        .sin_addr.s_addr = INADDR_ANY,
    };
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 8);

    int epfd = epoll_create1(0);
    struct epoll_event ev, events[64];
    ev.events = EPOLLIN;
    ev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);

    while (1) {
        int nready = epoll_wait(epfd, events, 64, -1);
        for (int i = 0; i < nready; i++) {
            int fd = events[i].data.fd;
            if (fd == lfd) {
                int cfd = accept(lfd, NULL, NULL);
                ev.events = EPOLLIN;
                ev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                write(cfd, "welcome from epoll\n", 19);
            } else {
                char buf[128];
                int n = read(fd, buf, sizeof(buf));
                if (n <= 0) {
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                } else {
                    write(fd, "event from epoll\n", 17);
                }
            }
        }
    }
}
```

---

## 3.5 异步通知

### 3.5.1 什么是异步通知

异步通知是指：设备事件发生时，内核主动通知用户进程，而不是用户进程主动轮询设备状态。

在字符设备驱动中，异步通知通常通过 `SIGIO` 信号完成。整体流程如下：

1. 用户态对设备 fd 开启 `O_ASYNC`
2. 内核调用驱动的 `.fasync`
3. 驱动把进程挂入异步通知链表
4. 设备事件发生时驱动调用 `kill_fasync`
5. 内核向目标进程发送 `SIGIO`

所以异步通知本质上是“事件触发 + 信号投递”的机制。

### 3.5.2 什么是信号

信号（signal）是内核向进程发送的软件中断通知，用于告知某个事件已经发生。

进程处理信号有三种方式：

1. 默认处理
2. 忽略处理（`SIG_IGN`）
3. 自定义处理函数（`signal` 或 `sigaction`）

异步通知通常使用 `SIGIO`。建议使用 `sigaction`，因为它比 `signal` 更稳定、可扩展：

```c
static void sigio_handler(int signo, siginfo_t *info, void *ctx)
{
    (void)signo;
    (void)info;
    (void)ctx;
    /* 处理异步事件 */
}

struct sigaction sa;
memset(&sa, 0, sizeof(sa));
sa.sa_sigaction = sigio_handler;
sa.sa_flags = SA_SIGINFO;
sigaction(SIGIO, &sa, NULL);
```

### 3.5.3 fcntl 函数与标志位说明

`fcntl` 原型：

```c
int fcntl(int fd, int cmd, ...);
```

参数语义：

1. `fd`：目标文件描述符
2. `cmd`：控制命令
3. 第三个参数：按 `cmd` 不同含义不同

#### 1) 与异步通知密切相关的 cmd

| cmd           | 作用         | 典型用途                      |
| ------------- | ---------- | ------------------------- |
| `F_GETFL`     | 获取文件状态标志   | 读取 `O_NONBLOCK`、`O_ASYNC` |
| `F_SETFL`     | 设置文件状态标志   | 设置 `O_NONBLOCK`、`O_ASYNC` |
| `F_GETOWN`    | 获取异步信号接收者  | 查询当前 owner                |
| `F_SETOWN`    | 设置异步信号接收者  | 设置 `SIGIO` 发给谁            |
| `F_GETOWN_EX` | 获取扩展 owner | 获取进程/进程组/线程信息             |
| `F_SETOWN_EX` | 设置扩展 owner | 指定进程/进程组/线程               |
| `F_GETFD`     | 获取 fd 标志   | 查询 `FD_CLOEXEC`           |
| `F_SETFD`     | 设置 fd 标志   | 设置 `FD_CLOEXEC`           |

#### 2) F_SETOWN 重点解释

```c
fcntl(fd, F_SETOWN, owner);
```

三个参数分别是：

1. 第 1 个参数 `fd`：目标设备 fd
2. 第 2 个参数 `F_SETOWN`：设置异步通知接收者
3. 第 3 个参数 `owner`：接收信号的目标

`owner` 取值语义：

1. `owner > 0`：发送给 PID 为 `owner` 的进程
2. `owner < 0`：发送给 PGID 为 `-owner` 的进程组
3. `owner == 0`：通常不使用

常见写法：

```c
fcntl(fd, F_SETOWN, getpid());
```

#### 3) 常见文件状态标志位（F_SETFL 使用）

| 标志位          | 含义     | 说明                  |
| ------------ | ------ | ------------------- |
| `O_ASYNC`    | 开启异步通知 | 触发驱动 `.fasync` 注册流程 |
| `O_NONBLOCK` | 非阻塞 IO | 无数据时立即返回            |
| `O_APPEND`   | 追加写    | 写入从文件尾追加            |
| `O_SYNC`     | 同步写    | 写操作等待同步完成           |

典型用户态启用代码：

```c
if (fcntl(fd, F_SETOWN, getpid()) == -1)
    perror("F_SETOWN");

int flags = fcntl(fd, F_GETFL);
if (flags == -1)
    perror("F_GETFL");

if (fcntl(fd, F_SETFL, flags | O_ASYNC) == -1)
    perror("F_SETFL(O_ASYNC)");
```

### 3.5.4 驱动如何实现异步通知

驱动侧推荐按下面步骤实现：

1. 设备结构体定义异步链表头

```c
struct fasync_struct *async_queue;
```

2. 实现 `.fasync` 回调，使用 `fasync_helper` 维护链表

```c
static int my_fasync(int fd, struct file *file, int on)
{
    struct my_dev *dev = file->private_data;
    return fasync_helper(fd, file, on, &dev->async_queue);
}
```

- `on = 1`：加入链表

- `on = 0`：从链表移除
3. 事件到来时调用 `kill_fasync` 发送 `SIGIO`

```c
kill_fasync(&dev->async_queue, SIGIO, POLL_IN);
```

4. 在 `.release` 里注销异步通知

```c
static int my_release(struct inode *inode, struct file *file)
{
    my_fasync(-1, file, 0);
    return 0;
}
```

5. 在 `file_operations` 挂接 `.fasync`

```c
static const struct file_operations my_fops = {
    .owner   = THIS_MODULE,
    .open    = my_open,
    .read    = my_read,
    .write   = my_write,
    .fasync  = my_fasync,
    .release = my_release,
};
```

实现注意点：

1. 仅在状态变化时调用 `kill_fasync`，避免信号风暴
2. 共享状态访问要加锁，避免并发冲突
3. 信号处理函数要尽量短，只做最小化操作

### 3.5.5 异步通知与多路复用对比

| 对比项   | 多路复用（select/poll/epoll） | 异步通知（fasync/SIGIO） |
| ----- | ----------------------- | ------------------ |
| 通知方式  | 用户主动等待                  | 内核主动投递信号           |
| 代码结构  | 集中式事件循环                 | 信号回调分散处理           |
| 典型场景  | 多 fd 并发监控               | 单/少量 fd 事件提醒       |
| 复杂度重点 | 等待队列与事件分发               | 信号安全与并发控制          |

---

# 4 内核定时器

内核定时器用于在未来某个时间点执行回调函数，典型场景包括超时检测、周期性轮询、延后重试等。Linux 内核定时器底层基于时间轮（timer wheel）管理，到期后由软中断上下文执行回调。

## 4.1 timer_list 结构体

定时器对象是 `struct timer_list`。不同内核版本内部字段会有差异，但核心语义一致：

- `expires`: 到期时间点，单位是 jiffies。
- `function`: 到期时执行的回调函数。
- `flags`: 定时器标志位（内部管理和行为控制）。
- `entry`: 挂入内核定时器链表的节点（内核内部使用）。

说明：

- 开发者主要关注 `expires` 和回调函数设置。
- 定时器回调运行在软中断上下文，不能睡眠，不能调用可能阻塞的 API。

```c
struct timer_list {
    struct hlist_node entry;  // 定时器链表节点
    unsigned long expires;    // 定时器到期时间（jiffies值）
    void (*function)(struct timer_list *);  // 到期回调函数
    u32 flags;                // 定时器标志
    ...
};
```

## 4.2 定时时间计算与 jiffies

`jiffies` 是内核全局节拍计数器，每个时钟中断加 1。

- `HZ`: 每秒节拍数（例如 100、250、1000）。
- 关系：秒与 jiffies 的换算是 `秒 * HZ`。

常见写法：

```c
unsigned long timeout = jiffies + 5 * HZ;   /* 5 秒后到期 */
```

为了可移植和避免溢出，推荐使用内核提供的转换函数。



全局变量jiffies用来记录自系统启动以来产生的节拍的总数。`jiffies = seconds*HZ`，jiffies定义在文件include/linux/jiffies.h中，定义如下：

```c
extern u64 _jiffy_data jiffies_64;
extern unsigned long volatile __jiffy_data jiffies;
```

## 4.3 jiffies 转换函数

常用函数如下：

- `msecs_to_jiffies(ms)`: 毫秒转 jiffies。
- `usecs_to_jiffies(us)`: 微秒转 jiffies。
- `jiffies_to_msecs(j)`: jiffies 转毫秒。
- `jiffies_to_usecs(j)`: jiffies 转微秒。
- `time_after(a, b)`, `time_before(a, b)`: 安全比较 jiffies 时间先后（处理回绕）。

示例：

```c
mod_timer(&my_timer, jiffies + msecs_to_jiffies(200));
```

## 4.4 内核定时器使用步骤

1. 定义 `struct timer_list` 变量。
2. 初始化定时器并绑定回调。
3. 设定到期时间并启动。
4. 在回调里执行任务，需要周期性就重装定时器。
5. 模块退出前调用 `del_timer_sync()` 删除定时器。

关键 API：

- `timer_setup(timer, callback, flags)`
- `add_timer(timer)`
- `mod_timer(timer, expires)`
- `del_timer(timer)`
- `del_timer_sync(timer)`

## 4.5 内核定时器初始化

Linux 内核定时器初始化方式与内核版本强相关。以 Linux 4.15 为分界点，定时器 API 和回调函数签名发生了明显变化。

内核定时器初始化有两类方式：

- 宏定义初始化：编译时静态初始化，代码简洁，适合全局静态定时器。
- 手动初始化：运行时初始化，灵活性高，适合结构体成员和动态对象。

### 4.5.1 宏定义初始化

宏定义初始化通过 `DEFINE_TIMER` 在定义变量时完成基础初始化。

#### 4.5.1.1 旧版本（Linux 4.14 及以下）

核心说明：

- 旧版 `DEFINE_TIMER` 常见为四参数形式。
- 回调签名为 `void (*)(unsigned long)`。

宏原型（教材常见）：

```c
#define DEFINE_TIMER(_name, _function, _expires, _data)
```

参数说明：

| 参数          | 说明                           |
| ----------- | ---------------------------- |
| `_name`     | 定时器变量名                       |
| `_function` | 到期回调函数                       |
| `_expires`  | 初始到期 jiffies 值（通常先写 0，启动时再设） |
| `_data`     | 传给回调函数的 `unsigned long` 参数   |

示例：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/timer.h>
#include <linux/jiffies.h>

static void old_timer_callback(unsigned long data)
{
    struct timer_list *timer = (struct timer_list *)data;
    pr_info("old timer callback\n");
    mod_timer(timer, jiffies + HZ);
}

DEFINE_TIMER(my_old_timer, old_timer_callback, 0, (unsigned long)&my_old_timer);

static int __init timer_init(void)
{
    mod_timer(&my_old_timer, jiffies + HZ);
    return 0;
}

static void __exit timer_exit(void)
{
    del_timer_sync(&my_old_timer);
}
```

注意事项：

- 回调参数必须是 `unsigned long`。
- `data` 常用于传递设备结构体指针（强转）。
- 类型安全较弱。

#### 4.5.1.2 新版本（Linux 4.15 及以上）

核心说明：

- 新版 `DEFINE_TIMER` 简化为两参数。
- 回调签名改为 `void (*)(struct timer_list *)`。

宏原型：

```c
#define DEFINE_TIMER(_name, _function)
```

参数说明：

| 参数          | 说明     |
| ----------- | ------ |
| `_name`     | 定时器变量名 |
| `_function` | 到期回调函数 |

示例：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/timer.h>
#include <linux/jiffies.h>

static void new_timer_callback(struct timer_list *t)
{
    pr_info("new timer callback\n");
    mod_timer(t, jiffies + msecs_to_jiffies(1000));
}

DEFINE_TIMER(my_new_timer, new_timer_callback);

static int __init timer_init(void)
{
    mod_timer(&my_new_timer, jiffies + msecs_to_jiffies(1000));
    return 0;
}

static void __exit timer_exit(void)
{
    del_timer_sync(&my_new_timer);
}
```

注意事项：

- 回调参数必须是 `struct timer_list *`。
- 宏内部默认 `flags = 0`。
- 业务上下文建议通过 `from_timer()` 从外层结构体获取。

### 4.5.2 手动初始化

手动初始化在运行时完成，适合设备驱动中的结构体成员场景。

#### 4.5.2.1 旧版本（Linux 4.14 及以下）

旧版常见两种方式：

- `init_timer()` + 手动赋值
- `setup_timer()` 宏

方式一示例（基础写法）：

```c
static struct timer_list my_old_timer;

static void old_timer_callback(unsigned long data)
{
    pr_info("old timer callback\n");
}

static int __init timer_init(void)
{
    init_timer(&my_old_timer);
    my_old_timer.function = old_timer_callback;
    my_old_timer.data = (unsigned long)&my_old_timer;
    my_old_timer.expires = jiffies + HZ;
    add_timer(&my_old_timer);
    return 0;
}
```

方式二示例（简化写法）：

```c
setup_timer(&my_old_timer, old_timer_callback, (unsigned long)&my_old_timer);
my_old_timer.expires = jiffies + HZ;
add_timer(&my_old_timer);
```

说明：

- `init_timer()` 仅做基础初始化，仍需手动设置 `function/data`。
- 实践中建议优先 `mod_timer()`，它对重复启动更安全。

#### 4.5.2.2 新版本（Linux 4.15 及以上）

新版本统一使用 `timer_setup()`：

```c
void timer_setup(struct timer_list *timer,
                 void (*callback)(struct timer_list *),
                 unsigned int flags);
```

参数说明：

| 参数         | 说明           |
| ---------- | ------------ |
| `timer`    | 要初始化的定时器对象   |
| `callback` | 到期回调函数       |
| `flags`    | 定时器标志，常用 `0` |

结构体成员场景示例：

```c
struct my_device {
    int id;
    struct timer_list timer;
};

static struct my_device dev;

static void device_timer_callback(struct timer_list *t)
{
    struct my_device *d = from_timer(d, t, timer);
    pr_info("device %d timer callback\n", d->id);
    mod_timer(t, jiffies + msecs_to_jiffies(2000));
}

static int __init timer_init(void)
{
    dev.id = 1;
    timer_setup(&dev.timer, device_timer_callback, 0);
    mod_timer(&dev.timer, jiffies + msecs_to_jiffies(2000));
    return 0;
}

static void __exit timer_exit(void)
{
    del_timer_sync(&dev.timer);
}
```

说明：

- `timer_setup()` 有类型检查，安全性更高。
- 可通过 `flags` 设置行为（如 `TIMER_DEFERRABLE`、`TIMER_PINNED`）。
- 新代码推荐统一使用 `timer_setup()` + `mod_timer()` + `del_timer_sync()`。

### 4.5.3 本节小结

- Linux 4.15 前后是定时器 API 的关键分界线。
- 旧版核心特征：`unsigned long data` 回调模型。
- 新版核心特征：`struct timer_list *` 回调模型。
- 生产代码建议采用新版接口，避免旧接口兼容性问题。

---

# 5 文件位置控制 lseek

## 5.1 用户空间 lseek 函数

### 5.1.1 lseek 原型与参数

在用户空间，应用程序通过 `lseek()` 系统调用改变打开文件的当前读写位置。

```c
off_t lseek(int fd, off_t offset, int whence);
```

参数说明：

| 参数       | 说明          |
| -------- | ----------- |
| `fd`     | 已打开文件的文件描述符 |
| `offset` | 位置偏移量，可正可负  |
| `whence` | 基准位置，三种选择   |

`whence` 参数可取值：

| 宏          | 说明                     |
| ---------- | ---------------------- |
| `SEEK_SET` | 从文件开头算，`offset` 为绝对位置  |
| `SEEK_CUR` | 从当前位置算，`offset` 为相对偏移  |
| `SEEK_END` | 从文件末尾算，通常 `offset` 为负数 |

返回值说明：

- `>= 0`：成功，返回新的文件位置（从文件开头算起的字节数）
- `-1`：出错，`errno` 被设置

### 5.1.2 lseek 使用示例

```c
#include <unistd.h>
#include <fcntl.h>

int main(void)
{
    int fd = open("/dev/my_device", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    /* 写入 12 字节 */
    write(fd, "hello world ", 12);

    /* 查看当前位置 */
    off_t pos = lseek(fd, 0, SEEK_CUR);
    printf("current position: %ld\n", pos);

    /* 回到开头 */
    lseek(fd, 0, SEEK_SET);

    /* 读取前 12 字节 */
    char buf[13] = {0};
    read(fd, buf, 12);
    printf("read: %s\n", buf);

    close(fd);
    return 0;
}
```

### 5.1.3 lseek 的意义

- 对于顺序设备（如串口、网卡）：lseek 无意义，内核驱动应返回 `-ESPIPE` 或不支持
- 对于随机访问设备（如块设备、内存设备、文件系统）：lseek 允许用户指定读写起点

---

## 5.2 内核空间 llseek 回调

### 5.2.1 llseek 函数原型

驱动通过 `file_operations` 中的 `llseek` 成员指定如何处理 lseek 系统调用。

```c
loff_t (*llseek)(struct file *file, loff_t offset, int whence);
```

参数说明：

| 参数       | 说明                                    |
| -------- | ------------------------------------- |
| `file`   | 代表打开的文件实例，包含 `f_pos` 成员表示当前位置         |
| `offset` | 用户传入的偏移量                              |
| `whence` | 用户传入的基准位置（SEEK_SET/SEEK_CUR/SEEK_END） |

返回值说明：

- `>= 0`：新的文件位置
- `< 0`：错误码（如 `-EINVAL`、`-ESPIPE`）

### 5.2.2 内核中文件位置的管理

- VFS 维护 `struct file` 的 `f_pos` 成员，记录当前读写位置
- 驱动的 `read()`、`write()` 接收 `loff_t *ppos` 指针，指向当前位置
- 驱动的 `read()`、`write()` 负责更新 `*ppos`
- 驱动的 `llseek()` 负责修改 `file->f_pos`

### 5.2.3 llseek 实现示例

```c
#define BUF_SIZE 1024U
static char mem[BUF_SIZE];

static loff_t demo_llseek(struct file *file, loff_t offset, int whence)
{
    loff_t new_pos = 0;

    switch (whence) {
        case SEEK_SET:
            /* 从文件开头算 */
            if (offset < 0 || offset > BUF_SIZE)
                return -EINVAL;
            new_pos = offset;
            break;

        case SEEK_CUR:
            /* 从当前位置算 */
            if (file->f_pos + offset < 0 || file->f_pos + offset > BUF_SIZE)
                return -EINVAL;
            new_pos = file->f_pos + offset;
            break;

        case SEEK_END:
            /* 从文件末尾算 */
            if (BUF_SIZE + offset < 0 || BUF_SIZE + offset > BUF_SIZE)
                return -EINVAL;
            new_pos = BUF_SIZE + offset;
            break;

        default:
            return -EINVAL;
    }

    file->f_pos = new_pos;
    pr_info("lseek to position: %lld\n", file->f_pos);
    return new_pos;
}
```

### 5.2.4 llseek 的三条原则

1. **验证位置合法**：确保新位置在 `[0, 设备大小]` 范围内
2. **更新 file->f_pos**：VFS 会用这个值更新应用层的位置
3. **返回新位置或错误码**：成功返回新位置，失败返回负错误码

---

## 5.3 read/write 与文件位置的关系

### 5.3.1 read 回调中的位置管理

`read` 回调接收 `loff_t *ppos` 指针，表示当前读取位置。

```c
static ssize_t demo_read(struct file *file, char __user *buf, 
                         size_t count, loff_t *ppos)
{
    loff_t pos = *ppos;
    size_t cnt = count;

    /* 检查越界 */
    if (pos >= BUF_SIZE)
        return 0;  /* EOF */

    /* 截断超出的部分 */
    if (pos + cnt > BUF_SIZE)
        cnt = BUF_SIZE - pos;

    /* 复制数据到用户空间 */
    if (copy_to_user(buf, mem + pos, cnt))
        return -EFAULT;

    /* 更新位置 */
    *ppos = pos + cnt;

    pr_info("read at pos %lld, cnt %zu\n", pos, cnt);
    return cnt;
}
```

关键点：

- 读取前检查 `*ppos` 是否超过设备大小
- 读取完成后，必须更新 `*ppos` 为新位置
- 返回实际读取字节数

### 5.3.2 write 回调中的位置管理

`write` 回调同样接收 `loff_t *ppos`，逻辑与 `read` 相似。

```c
static ssize_t demo_write(struct file *file, const char __user *buf,
                          size_t count, loff_t *ppos)
{
    loff_t pos = *ppos;
    size_t cnt = count;

    /* 检查写入位置是否超过缓冲区 */
    if (pos >= BUF_SIZE)
        return -ENOSPC;  /* 或返回 0 */

    /* 截断超出的部分 */
    if (pos + cnt > BUF_SIZE)
        cnt = BUF_SIZE - pos;

    /* 从用户空间复制数据 */
    if (copy_from_user(mem + pos, buf, cnt))
        return -EFAULT;

    /* 更新位置 */
    *ppos = pos + cnt;

    pr_info("write at pos %lld, cnt %zu\n", pos, cnt);
    return cnt;
}
```

关键点：

- 写入前检查写入位置和大小的合法性
- 写入完成后，必须更新 `*ppos`
- 返回实际写入字节数

### 5.3.3 read/write/llseek 的协作流程

```
用户调用 lseek(fd, 10, SEEK_SET)
    ↓
VFS 调用驱动的 llseek()
    ↓
驱动设置 file->f_pos = 10
    ↓
用户调用 read(fd, buf, 5)
    ↓
VFS 调用驱动的 read(file, buf, 5, &ppos)，其中 ppos 指向 file->f_pos
    ↓
驱动从 mem[10] 开始读 5 字节
    ↓
驱动更新 *ppos = 15
    ↓
返回 5 字节给用户
    ↓
用户再次调用 read(fd, buf, 5)
    ↓
VFS 再次调用 read，此时 ppos 已是 15
    ↓
驱动从 mem[15] 开始读 5 字节
```

---

## 5.4 file_operations 配置

要支持 lseek，需要在 `file_operations` 中注册 `llseek` 回调：

```c
static const struct file_operations fops = {
    .owner   = THIS_MODULE,
    .open    = demo_open,
    .release = demo_release,
    .read    = demo_read,
    .write   = demo_write,
    .llseek  = demo_llseek,
};
```

如果驱动不支持 lseek（如顺序设备），可以：

1. 不定义 `.llseek`，内核默认使用 `default_llseek`
2. 显式赋值为 `no_llseek`，返回 `-ESPIPE`

```c
.llseek = no_llseek,  /* 禁止 lseek */
```

---

## 5.5 注意事项

### 5.5.1 字符串与 `\0` 的陷阱

当用 `write()` 写入 C 字符串时，务必检查是否包含结束符：

```c
/* 错误：写入 13 字节，包括 \0 */
write(fd, "hello world ", 13);

/* 正确：写入 12 字节，不包括 \0 */
write(fd, "hello world ", 12);

/* 或使用 strlen() */
write(fd, "hello world ", strlen("hello world "));
```

如果不小心写进 `\0`，后续用 `printf("%s\n", buf)` 打印会提前截断。

### 5.5.2 越界检查

- llseek 中一定要验证新位置不超过设备大小
- read/write 中一定要截断超出范围的请求
- 避免用户通过恶意参数导致内核越界读写

### 5.5.3 位置同步

- 多进程或多线程访问同一设备时，位置是独立的（每个 `file` 结构有自己的 `f_pos`）
- 如果需要共享位置状态，应使用 `O_APPEND` 或在驱动中维护全局位置

---

## 5.6 本章小结

- `lseek()` 是改变文件位置的系统调用
- 驱动通过 `llseek()` 回调处理 lseek 请求，修改 `file->f_pos`
- `read()` 和 `write()` 回调通过 `*ppos` 参数获取/更新当前位置
- 三者协作完成随机访问的完整流程
- 注意越界检查、字符串 `\0` 陷阱、多进程位置独立性

---

# 6 Linux 内核打印等级

## 6.1 内核打印等级概述

### 6.1.1 什么是内核打印等级

Linux 内核提供的日志输出函数 `printk()` 支持按等级打印信息。等级用于区分日志的重要程度，内核和用户空间都可以设置"过滤等级"，决定哪些日志会在控制台显示，哪些只存入日志缓冲区。

### 6.1.2 内核打印的两个等级概念

- **日志等级**：每条打印信息自身携带的优先级
- **控制台等级**：内核当前允许显示在控制台上的最低等级

只有日志等级 <= 控制台等级的信息才会在控制台直接显示；其他信息仍存在内核日志缓冲区，可通过 `dmesg` 查看。

---

## 6.2 八个内核打印等级

### 6.2.1 完整的等级列表

Linux 内核定义了 8 个打印等级，从高优先级到低优先级：

| 等级数字 | 宏定义            | 说明           |
| ---- | -------------- | ------------ |
| 0    | `KERN_EMERG`   | 系统不可用，最严重的错误 |
| 1    | `KERN_ALERT`   | 必须立即采取行动     |
| 2    | `KERN_CRIT`    | 严重错误         |
| 3    | `KERN_ERR`     | 错误           |
| 4    | `KERN_WARNING` | 警告           |
| 5    | `KERN_NOTICE`  | 普通但重要的信息     |
| 6    | `KERN_INFO`    | 提示信息         |
| 7    | `KERN_DEBUG`   | 调试信息         |

等级越小，优先级越高；等级越大，优先级越低。

### 6.2.2 使用打印等级的示例

```c
#include <linux/kernel.h>

/* 紧急错误 */
printk(KERN_EMERG "System is unusable!\n");

/* 告警 */
printk(KERN_ALERT "Action must be taken!\n");

/* 严重错误 */
printk(KERN_CRIT "Critical condition!\n");

/* 普通错误 */
printk(KERN_ERR "An error occurred\n");

/* 警告 */
printk(KERN_WARNING "This is a warning\n");

/* 普通信息 */
printk(KERN_NOTICE "This is a notice\n");

/* 提示信息 */
printk(KERN_INFO "This is info\n");

/* 调试信息 */
printk(KERN_DEBUG "Debug message\n");
```

### 6.2.3 未指定等级的情况

如果 `printk()` 不指定等级（或指定了无效等级），内核使用默认等级 `DEFAULT_MESSAGE_LOGLEVEL`，通常为 **4 (KERN_WARNING)**。

```c
printk("This will use default log level\n");
```

---

## 6.3 内核控制台等级

### 6.3.1 四个关键的消息等级参数

内核维护四个重要的等级值，可通过 `/proc/sys/kernel/printk` 查看和修改：

```bash
$ cat /proc/sys/kernel/printk
4 4 1 7
```

输出四个数字的含义：

| 位置    | 名称                         | 含义                            |
| ----- | -------------------------- | ----------------------------- |
| 第 1 个 | `console_loglevel`         | 控制台等级（当前）：只有日志等级 <= 此值才显示在控制台 |
| 第 2 个 | `default_message_loglevel` | 默认日志等级：`printk()` 未指定等级时使用    |
| 第 3 个 | `minimum_console_loglevel` | 最小控制台等级：不能降低控制台等级到此值以下        |
| 第 4 个 | `default_console_loglevel` | 默认控制台等级：系统启动时设定为此值            |

### 6.3.2 实际含义解释

以 `4 4 1 7` 为例：

- **第 1 个数字 4**：当前控制台等级为 4（`KERN_WARNING` 及以上）
  
  - 日志等级为 0、1、2、3、4 的信息会显示在控制台
  - 日志等级为 5、6、7 的信息只存入缓冲区，不显示在控制台

- **第 2 个数字 4**：默认日志等级为 4（`KERN_WARNING`）
  
  - 调用 `printk("message")` 不指定等级时，视为 `KERN_WARNING`

- **第 3 个数字 1**：最小控制台等级为 1（`KERN_ALERT`）
  
  - 用户通过命令无法将控制台等级设置到 1 以下，防止低优先级信息淹没控制台

- **第 4 个数字 7**：默认控制台等级为 7（`KERN_DEBUG`）
  
  - 系统启动时初始化为此值

---

## 6.4 使用 dmesg 和 /proc 查看日志

### 6.4.1 查看内核日志缓冲区

所有内核打印信息（包括没有显示在控制台的）都存储在内核日志缓冲区中，可用 `dmesg` 查看：

```bash
$ dmesg
[    0.000000] Linux version 6.6.114.1-microsoft-standard-WSL2 (...)
[    0.000000] Command line: initrd=umd.cpio ...
...
```

常用选项：

```bash
/* 查看最后 20 行 */
$ dmesg | tail -20

/* 实时监看新日志 */
$ dmesg -f

/* 清空日志缓冲区 */
$ sudo dmesg -c
```

### 6.4.2 查看当前打印等级

```bash
$ cat /proc/sys/kernel/printk
4 4 1 7
```

---

## 6.5 使用 echo 修改控制台等级

### 6.5.1 修改控制台等级的方法

通过写入 `/proc/sys/kernel/printk` 可以改变当前控制台等级。

只需修改第 1 个数字即可。格式为：

```bash
echo <new_console_level> > /proc/sys/kernel/printk
```

### 6.5.2 常用场景

**场景 1：提高控制台等级，显示更多日志（包括 INFO 和 DEBUG）**

```bash
$ sudo echo 8 > /proc/sys/kernel/printk
```

此后，所有等级（0-7）的日志都会显示在控制台。

验证：

```bash
$ cat /proc/sys/kernel/printk
8 4 1 7
```

**场景 2：降低控制台等级，只显示严重错误**

```bash
$ sudo echo 3 > /proc/sys/kernel/printk
```

此后，只有等级为 0、1、2、3 的日志显示在控制台（KERN_EMERG、KERN_ALERT、KERN_CRIT、KERN_ERR）。

**场景 3：恢复默认值**

```bash
$ sudo echo 4 > /proc/sys/kernel/printk
```

### 6.5.3 修改完整的四个参数

可以一次修改所有四个参数：

```bash
/* 格式：console_loglevel default_message_loglevel minimum_console_loglevel default_console_loglevel */
$ sudo echo 8 4 1 7 > /proc/sys/kernel/printk
```

---

## 6.6 驱动开发中的打印实战

### 6.6.1 选择合适的等级

根据消息的重要程度选择等级：

```c
/* 模块加载失败，使用 ERROR */
if (alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME) < 0) {
    printk(KERN_ERR "Failed to allocate device number\n");
    return -1;
}

/* 正常的初始化流程，使用 INFO */
printk(KERN_INFO "Device %s loaded, major: %d\n", DEVICE_NAME, major);

/* 调试某个变量值，使用 DEBUG */
printk(KERN_DEBUG "Current state: %d\n", state);

/* 需要立即注意的问题，使用 WARNING */
if (buffer_overrun) {
    printk(KERN_WARNING "Buffer might overflow!\n");
}
```

### 6.6.2 调试技巧

开发时，可以临时提高控制台等级以查看 DEBUG 信息：

```bash
/* 1. 加载模块前先提高等级 */
$ sudo echo 8 > /proc/sys/kernel/printk

/* 2. 加载驱动模块 */
$ sudo insmod my_driver.ko

/* 3. 查看日志中的 DEBUG 信息 */
$ dmesg | tail -30

/* 4. 恢复等级 */
$ sudo echo 4 > /proc/sys/kernel/printk
```

### 6.6.3 推荐的打印等级用法

| 情形       | 推荐等级           | 示例                                |
| -------- | -------------- | --------------------------------- |
| 初始化成功    | `KERN_INFO`    | `Device registered successfully`  |
| 功能异常但可恢复 | `KERN_WARNING` | `Buffer nearly full, flushing...` |
| 致命错误     | `KERN_ERR`     | `Failed to allocate memory`       |
| 调试数据     | `KERN_DEBUG`   | `Reading 100 bytes from offset 0` |
| 需要立即处理   | `KERN_ALERT`   | `System overheating!`             |

---

## 6.7 本章小结

- 内核有 8 个打印等级（0-7），从 `KERN_EMERG` 到 `KERN_DEBUG`
- 日志等级决定信息的优先级；控制台等级决定哪些信息显示在控制台
- `/proc/sys/kernel/printk` 记录 4 个关键参数：当前控制台等级、默认日志等级、最小控制台等级、默认控制台等级
- 用 `echo` 可以修改第 1 个参数（当前控制台等级）以调整日志显示
- 开发驱动时应根据消息重要程度选择合适的等级，便于调试和监控
- 所有日志都保存在内核缓冲区，可用 `dmesg` 查看

---

# 7 IOCTL 控制接口

## 7.1 什么是 IOCTL

### 7.1.1 IOCTL 的作用

`ioctl`（input/output control）用于执行“读写之外”的设备控制操作。

典型场景：

- 设置设备参数（波特率、模式、阈值）
- 触发控制动作（复位、清缓冲、启动/停止）
- 获取状态信息（版本号、寄存器值、统计计数）

与 `read/write` 的区别：

- `read/write` 主要处理数据流
- `ioctl` 主要处理控制命令

---

## 7.2 用户态如何调用 ioctl

### 7.2.1 函数原型

```c
#include <sys/ioctl.h>

int ioctl(int fd, unsigned long cmd, ...);
```

参数说明：

| 参数    | 说明                    |
| ----- | --------------------- |
| `fd`  | 已打开设备节点的文件描述符         |
| `cmd` | 命令码，定义“做什么”           |
| `...` | 可选参数，通常是指针，给驱动传入/传出数据 |

返回值说明：

- `0` 或非负值：成功
- `-1`：失败，`errno` 被设置

### 7.2.2 用户态示例

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

#define DEMO_IOC_MAGIC  'D'
#define DEMO_IOC_SET_VAL _IOW(DEMO_IOC_MAGIC, 1, int)
#define DEMO_IOC_GET_VAL _IOR(DEMO_IOC_MAGIC, 2, int)

int main(void)
{
    int fd = open("/dev/demo-device", O_RDWR);
    int val = 123;

    ioctl(fd, DEMO_IOC_SET_VAL, &val);

    val = 0;
    ioctl(fd, DEMO_IOC_GET_VAL, &val);
    printf("val=%d\n", val);

    close(fd);
    return 0;
}
```

---

## 7.3 驱动如何实现 ioctl

### 7.3.1 file_operations 挂接

新内核推荐使用 `unlocked_ioctl`：

```c
static long demo_ioctl(struct file *file, unsigned int cmd, unsigned long arg);

static const struct file_operations fops = {
    .owner          = THIS_MODULE,
    .read           = demo_read,
    .write          = demo_write,
    .unlocked_ioctl = demo_ioctl,
};
```

### 7.3.2 驱动回调原型

```c
static long demo_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
```

参数说明：

| 参数     | 说明              |
| ------ | --------------- |
| `file` | 当前打开文件对象        |
| `cmd`  | 用户传入命令码         |
| `arg`  | 用户传入参数，常是用户指针地址 |

### 7.3.3 驱动实现示例

```c
#include <linux/uaccess.h>

#define DEMO_IOC_MAGIC   'D'
#define DEMO_IOC_SET_VAL _IOW(DEMO_IOC_MAGIC, 1, int)
#define DEMO_IOC_GET_VAL _IOR(DEMO_IOC_MAGIC, 2, int)

static int demo_val;

static long demo_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    int val;

    if (_IOC_TYPE(cmd) != DEMO_IOC_MAGIC)
        return -ENOTTY;

    switch (cmd) {
    case DEMO_IOC_SET_VAL:
        if (copy_from_user(&val, (int __user *)arg, sizeof(val)))
            return -EFAULT;
        demo_val = val;
        return 0;

    case DEMO_IOC_GET_VAL:
        val = demo_val;
        if (copy_to_user((int __user *)arg, &val, sizeof(val)))
            return -EFAULT;
        return 0;

    default:
        return -ENOTTY;
    }
}
```

---

## 7.4 命令码是什么

### 7.4.1 命令码的本质

`cmd` 不是随便写的数字，它是按位编码后的控制字，包含：

- 方向（读/写/无参数）
- 类型（magic）
- 序号（nr）
- 数据大小（size）

内核通过这些字段做校验与分发。

### 7.4.2 命令码构成

命令码由 `_IOC(dir, type, nr, size)` 组合而成：

```c
#define _IOC(dir,type,nr,size) ...
```

本章统一使用如下顺序描述命令码字段：

1. 设备类型（type）
2. 序列号（nr）
3. 方向（dir）
4. 数据尺寸（size）

字段位宽（常见定义）：

- `type`：8 位
- `nr`：8 位
- `dir`：2 位
- `size`：13/14 位（与架构相关，常见为 14 位）

字段意义：

| 字段     | 说明                |
| ------ | ----------------- |
| `type` | 设备类型标识（magic）     |
| `nr`   | 命令序号              |
| `dir`  | 方向：无参数/写入内核/从内核读出 |
| `size` | 数据类型大小（字节）        |

方向字段常见取值：

```c
#define _IOC_NONE   0U
#define _IOC_WRITE  1U
#define _IOC_READ   2U
```

方向含义（按用户视角）：

- `_IOC_WRITE`：用户把数据写给内核（常用于 `_IOW`）
- `_IOC_READ`：用户从内核读数据（常用于 `_IOR`）
- `_IOC_READ | _IOC_WRITE`：双向（`_IOWR`）

示例命令：

```c
#define DEMO_IOC_SET_VAL  _IOW('D', 1, int)
```

按本节顺序拆解：

- `type = 'D'`
- `nr = 1`
- `dir = _IOC_WRITE`
- `size = sizeof(int)`

分解命令码时可使用以下宏：

```c
_IOC_DIR(cmd)   /* 取出 2 位方向 */
_IOC_TYPE(cmd)  /* 取出 8 位 type */
_IOC_NR(cmd)    /* 取出 8 位 nr */
_IOC_SIZE(cmd)  /* 取出 14 位 size */
```

说明：少数体系结构在 `size` 位宽上可能不同，但驱动开发应始终使用 `_IO/_IOR/_IOW/_IOWR` 和 `_IOC_*` 宏，不要手写位运算常量。

---

## 7.5 命令如何组合（组合宏）

### 7.5.1 四个常用组合宏

```c
#define _IO(type,nr)           _IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)     _IOC(_IOC_READ,(type),(nr),sizeof(size))
#define _IOW(type,nr,size)     _IOC(_IOC_WRITE,(type),(nr),sizeof(size))
#define _IOWR(type,nr,size)    _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),sizeof(size))
```

用途说明：

| 宏       | 含义     | 场景            |
| ------- | ------ | ------------- |
| `_IO`   | 无数据参数  | 纯控制命令，如 reset |
| `_IOR`  | 内核读给用户 | 用户“获取”设备信息    |
| `_IOW`  | 用户写给内核 | 用户“设置”设备参数    |
| `_IOWR` | 双向读写   | 既输入又输出        |

### 7.5.2 命令定义建议

```c
#define DEMO_IOC_MAGIC   'D'

#define DEMO_IOC_RESET    _IO(DEMO_IOC_MAGIC, 0)
#define DEMO_IOC_SET_VAL  _IOW(DEMO_IOC_MAGIC, 1, int)
#define DEMO_IOC_GET_VAL  _IOR(DEMO_IOC_MAGIC, 2, int)
#define DEMO_IOC_XCHG_VAL _IOWR(DEMO_IOC_MAGIC, 3, int)
```

建议规则：

- `type` 在同一驱动中保持唯一且稳定
- `nr` 从小到大分配，避免复用
- `size` 填“类型名”，不要直接写数字

---

## 7.6 命令如何分解（分解宏）

### 7.6.1 常用分解宏

内核可用以下宏拆出命令字段：

```c
_IOC_DIR(cmd)
_IOC_TYPE(cmd)
_IOC_NR(cmd)
_IOC_SIZE(cmd)
```

### 7.6.2 分解与校验示例

```c
pr_info("dir=%u type=%u nr=%u size=%u\n",
        _IOC_DIR(cmd), _IOC_TYPE(cmd), _IOC_NR(cmd), _IOC_SIZE(cmd));

if (_IOC_TYPE(cmd) != DEMO_IOC_MAGIC)
    return -ENOTTY;

if (_IOC_NR(cmd) > 3)
    return -ENOTTY;
```

常见校验顺序：

1. 先校验 `type`
2. 再校验 `nr`
3. 需要时校验 `size`
4. 最后进入 `switch(cmd)`

---

## 7.7 IOCTL 实战案例

### 7.7.1 案例一：通过 ioctl 读写“地址-值”

这个案例模拟“寄存器读写”场景：

- 用户传入寄存器地址 `addr`
- 驱动按地址读/写对应值 `value`
- 读操作常用 `_IOWR`（地址传入、值传出）

数据结构与命令定义：

```c
#include <linux/types.h>

struct reg_access {
    __u32 addr;
    __u32 value;
};

#define DEMO_IOC_MAGIC      'D'
#define DEMO_IOC_REG_WRITE  _IOW(DEMO_IOC_MAGIC, 10, struct reg_access)
#define DEMO_IOC_REG_READ   _IOWR(DEMO_IOC_MAGIC, 11, struct reg_access)
```

驱动端处理示例：

```c
static __u32 fake_regs[256];

static long demo_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct reg_access ra;

    if (_IOC_TYPE(cmd) != DEMO_IOC_MAGIC)
        return -ENOTTY;

    switch (cmd) {
    case DEMO_IOC_REG_WRITE:
        if (copy_from_user(&ra, (void __user *)arg, sizeof(ra)))
            return -EFAULT;
        if (ra.addr >= 256)
            return -EINVAL;
        fake_regs[ra.addr] = ra.value;
        return 0;

    case DEMO_IOC_REG_READ:
        if (copy_from_user(&ra, (void __user *)arg, sizeof(ra)))
            return -EFAULT;
        if (ra.addr >= 256)
            return -EINVAL;
        ra.value = fake_regs[ra.addr];
        if (copy_to_user((void __user *)arg, &ra, sizeof(ra)))
            return -EFAULT;
        return 0;

    default:
        return -ENOTTY;
    }
}
```

用户态调用示例：

```c
struct reg_access ra;

/* 写：addr=0x20, value=0x12345678 */
ra.addr = 0x20;
ra.value = 0x12345678;
ioctl(fd, DEMO_IOC_REG_WRITE, &ra);

/* 读：从 addr=0x20 读回 value */
ra.addr = 0x20;
ra.value = 0;
ioctl(fd, DEMO_IOC_REG_READ, &ra);
printf("addr=0x%x value=0x%x\n", ra.addr, ra.value);
```

### 7.7.2 案例二：通过 ioctl 控制定时器

需求：

- 打开定时器（start）
- 关闭定时器（stop）
- 修改定时器周期（set period）

命令与全局状态定义：

```c
#define DEMO_IOC_TIMER_START   _IO(DEMO_IOC_MAGIC, 20)
#define DEMO_IOC_TIMER_STOP    _IO(DEMO_IOC_MAGIC, 21)
#define DEMO_IOC_TIMER_SET_MS  _IOW(DEMO_IOC_MAGIC, 22, unsigned int)

static struct timer_list demo_timer;
static unsigned int demo_period_ms = 1000;
static bool demo_timer_running;
```

驱动端实现示例：

```c
static void demo_timer_cb(struct timer_list *t)
{
    pr_info("timer tick, period=%u ms\n", demo_period_ms);

    if (demo_timer_running)
        mod_timer(&demo_timer, jiffies + msecs_to_jiffies(demo_period_ms));
}

static long demo_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    unsigned int ms;

    if (_IOC_TYPE(cmd) != DEMO_IOC_MAGIC)
        return -ENOTTY;

    switch (cmd) {
    case DEMO_IOC_TIMER_START:
        demo_timer_running = true;
        mod_timer(&demo_timer, jiffies + msecs_to_jiffies(demo_period_ms));
        return 0;

    case DEMO_IOC_TIMER_STOP:
        demo_timer_running = false;
        del_timer_sync(&demo_timer);
        return 0;

    case DEMO_IOC_TIMER_SET_MS:
        if (copy_from_user(&ms, (void __user *)arg, sizeof(ms)))
            return -EFAULT;
        if (ms < 10 || ms > 60000)
            return -EINVAL;
        demo_period_ms = ms;

        /* 如果定时器已开启，立即应用新周期 */
        if (demo_timer_running)
            mod_timer(&demo_timer, jiffies + msecs_to_jiffies(demo_period_ms));
        return 0;

    default:
        return -ENOTTY;
    }
}
```

用户态调用示例：

```c
unsigned int ms = 2000;

/* 修改为 2 秒 */
ioctl(fd, DEMO_IOC_TIMER_SET_MS, &ms);

/* 启动定时器 */
ioctl(fd, DEMO_IOC_TIMER_START);

sleep(5);

/* 停止定时器 */
ioctl(fd, DEMO_IOC_TIMER_STOP);
```

### 7.7.3 这两个案例的设计要点

- 地址读写类命令：优先用结构体封装参数，避免参数扩展困难
- 涉及用户内存传递：必须用 `copy_from_user/copy_to_user`
- 定时器控制类命令：`start/stop/set` 要拆成独立命令，语义清晰
- 周期修改命令建议做上下限检查，防止设置异常值

---

## 7.8 IOCTL 章节小结

- `ioctl` 用于设备控制，不替代 `read/write`
- 用户态通过 `ioctl(fd, cmd, arg)` 发命令，驱动在 `unlocked_ioctl` 中处理
- `cmd` 由方向、类型、序号、大小四部分编码构成
- 组合宏：`_IO/_IOR/_IOW/_IOWR`
- 分解宏：`_IOC_DIR/_IOC_TYPE/_IOC_NR/_IOC_SIZE`
- 实战中常见两类：地址读写控制、定时器启停与周期配置
- 推荐在驱动中做 `type`/`nr`/`size` 校验，非法命令返回 `-ENOTTY`

---

# 8 将驱动封装为 API 函数

## 8.1 为什么要把驱动封装为 API

在学习阶段，我们常直接在应用里调用 `open/read/write/ioctl/close`。

但在真实项目中，建议把这些系统调用封装成统一 API，主要原因是：

- 屏蔽设备节点路径、命令码细节，减少业务代码重复
- 统一错误处理和日志输出
- 便于版本升级（驱动变化时只改一处）
- 形成可复用组件，供多个应用共享

---

## 8.2 基础知识：什么是库

### 8.2.1 库的定义

库（Library）是可复用代码的集合，通常包含：

- 头文件：声明 API（给调用者看）
- 库文件：实现代码（给链接器和运行时用）

调用者只需包含头文件并链接库，即可使用已有能力。

### 8.2.2 常见库类型

1. 静态库（`.a`）
2. 动态库（`.so`）

对比：

| 类型        | 特点          | 优点                 | 缺点                |
| --------- | ----------- | ------------------ | ----------------- |
| 静态库 `.a`  | 编译时拷贝到可执行文件 | 部署简单、运行不依赖外部 `.so` | 可执行文件更大，升级需重编译    |
| 动态库 `.so` | 运行时装载       | 程序体积小、升级库可不重编应用    | 运行环境需保证 `.so` 可找到 |

### 8.2.3 如何使用库

头文件使用：

```c
#include "demo_api.h"
```

链接静态库示例：

```bash
gcc app.c -L. -ldemoapi -o app
```

链接动态库示例：

```bash
gcc app.c -L. -ldemoapi -o app
export LD_LIBRARY_PATH=.
./app
```

---

## 8.3 库的原理

### 8.3.1 编译与链接流程

应用使用库的大致流程：

1. 预处理：展开头文件，检查函数声明
2. 编译：把源文件变成目标文件（`.o`）
3. 链接：把应用目标文件和库合并
4. 运行：静态库直接运行，动态库在启动时装载

### 8.3.2 符号解析

链接器会根据函数名（符号）把调用点绑定到实现：

- 应用写 `demo_set_value(...)`
- 链接器在库中找到同名函数定义
- 生成可执行程序中的调用关系

如果找不到实现，会出现“undefined reference”链接错误。

### 8.3.3 驱动 API 库的本质

驱动 API 库并不是在用户态“重写驱动”，而是做协议适配层：

- 向下：调用 `open/read/write/ioctl/close`
- 向上：暴露业务可读的函数，例如 `timer_start()`

所以它的核心价值是“接口标准化”和“调用简化”。

---

## 8.4 封装驱动 API 的设计思路

### 8.4.1 先定义稳定接口

先写头文件 API，而不是先写应用。建议包含：

- 生命周期：`init/open/close`
- 基础数据接口：`read/write`
- 控制接口：`set/get/start/stop`
- 错误码约定：返回 `0` 成功，负值失败

### 8.4.2 统一上下文句柄

不要把 `fd` 暴露到各处，建议定义上下文结构：

```c
typedef struct {
    int fd;
} demo_handle_t;
```

这样后续扩展（锁、缓存、状态）更方便。

### 8.4.3 API 与 ioctl 命令映射

封装层应完成“函数语义 -> 命令码”的映射：

- `demo_timer_start()` -> `DEMO_IOC_TIMER_START`
- `demo_timer_stop()` -> `DEMO_IOC_TIMER_STOP`
- `demo_timer_set_ms(ms)` -> `DEMO_IOC_TIMER_SET_MS`

业务层无需感知命令码细节。

---

## 8.5 参考实现：头文件 + 实现文件

### 8.5.1 头文件示例（demo_api.h）

```c
#ifndef DEMO_API_H
#define DEMO_API_H

#include <stdint.h>

typedef struct {
    int fd;
} demo_handle_t;

typedef struct {
    uint32_t addr;
    uint32_t value;
} demo_reg_t;

int demo_open(demo_handle_t *h, const char *dev);
int demo_close(demo_handle_t *h);

int demo_reg_write(demo_handle_t *h, uint32_t addr, uint32_t value);
int demo_reg_read(demo_handle_t *h, uint32_t addr, uint32_t *value);

int demo_timer_start(demo_handle_t *h);
int demo_timer_stop(demo_handle_t *h);
int demo_timer_set_ms(demo_handle_t *h, uint32_t ms);

#endif
```

### 8.5.2 实现文件示例（demo_api.c）

```c
#include "demo_api.h"
#include <fcntl.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <errno.h>

#define DEMO_IOC_MAGIC        'D'
#define DEMO_IOC_REG_WRITE    _IOW(DEMO_IOC_MAGIC, 10, demo_reg_t)
#define DEMO_IOC_REG_READ     _IOWR(DEMO_IOC_MAGIC, 11, demo_reg_t)
#define DEMO_IOC_TIMER_START  _IO(DEMO_IOC_MAGIC, 20)
#define DEMO_IOC_TIMER_STOP   _IO(DEMO_IOC_MAGIC, 21)
#define DEMO_IOC_TIMER_SET_MS _IOW(DEMO_IOC_MAGIC, 22, uint32_t)

int demo_open(demo_handle_t *h, const char *dev)
{
    if (!h || !dev)
        return -EINVAL;

    h->fd = open(dev, O_RDWR);
    if (h->fd < 0)
        return -errno;

    return 0;
}

int demo_close(demo_handle_t *h)
{
    if (!h || h->fd < 0)
        return -EINVAL;

    if (close(h->fd) < 0)
        return -errno;

    h->fd = -1;
    return 0;
}

int demo_reg_write(demo_handle_t *h, uint32_t addr, uint32_t value)
{
    demo_reg_t reg = { .addr = addr, .value = value };

    if (!h || h->fd < 0)
        return -EINVAL;

    if (ioctl(h->fd, DEMO_IOC_REG_WRITE, &reg) < 0)
        return -errno;

    return 0;
}

int demo_reg_read(demo_handle_t *h, uint32_t addr, uint32_t *value)
{
    demo_reg_t reg;

    if (!h || h->fd < 0 || !value)
        return -EINVAL;

    reg.addr = addr;
    reg.value = 0;

    if (ioctl(h->fd, DEMO_IOC_REG_READ, &reg) < 0)
        return -errno;

    *value = reg.value;
    return 0;
}

int demo_timer_start(demo_handle_t *h)
{
    if (!h || h->fd < 0)
        return -EINVAL;
    if (ioctl(h->fd, DEMO_IOC_TIMER_START) < 0)
        return -errno;
    return 0;
}

int demo_timer_stop(demo_handle_t *h)
{
    if (!h || h->fd < 0)
        return -EINVAL;
    if (ioctl(h->fd, DEMO_IOC_TIMER_STOP) < 0)
        return -errno;
    return 0;
}

int demo_timer_set_ms(demo_handle_t *h, uint32_t ms)
{
    if (!h || h->fd < 0)
        return -EINVAL;
    if (ioctl(h->fd, DEMO_IOC_TIMER_SET_MS, &ms) < 0)
        return -errno;
    return 0;
}
```

### 8.5.3 业务调用示例

```c
#include "demo_api.h"
#include <stdio.h>

int main(void)
{
    demo_handle_t h = { .fd = -1 };
    uint32_t val;

    if (demo_open(&h, "/dev/demo-device") < 0)
        return -1;

    demo_reg_write(&h, 0x20, 0x12345678);
    demo_reg_read(&h, 0x20, &val);
    printf("reg=0x%x\n", val);

    demo_timer_set_ms(&h, 1000);
    demo_timer_start(&h);
    sleep(3);
    demo_timer_stop(&h);

    demo_close(&h);
    return 0;
}
```

---

## 8.6 如何把 API 做成静态库和动态库

### 8.6.1 生成静态库 `.a`

```bash
gcc -c demo_api.c -o demo_api.o
ar rcs libdemoapi.a demo_api.o
```

应用链接：

```bash
gcc app.c -L. -ldemoapi -o app
```

### 8.6.2 生成动态库 `.so`

```bash
gcc -fPIC -c demo_api.c -o demo_api.o
gcc -shared -o libdemoapi.so demo_api.o
```

应用链接与运行：

```bash
gcc app.c -L. -ldemoapi -o app
export LD_LIBRARY_PATH=.
./app
```

---

## 8.7 封装驱动 API 的工程规范

### 8.7.1 错误码规范

- API 对外统一返回 `0` 成功，负值失败
- 系统调用失败时返回 `-errno`
- 参数错误返回 `-EINVAL`

### 8.7.2 线程安全

如果多个线程共享同一个 `demo_handle_t`：

- 需要在 API 层加互斥锁
- 或规定“一线程一句柄”

### 8.7.3 版本兼容

当 ioctl 命令升级时：

- 保留旧命令一段时间
- 新增 API 用新命令，不要直接覆盖旧语义
- 通过版本号函数（如 `demo_get_version()`）做能力检测

### 8.7.4 文档与测试

至少准备：

- API 头文件注释（输入、输出、返回值）
- 最小可运行样例
- 回归测试脚本（覆盖 open/close/ioctl 常用路径）

---

## 8.8 本章小结

- 库是可复用代码集合，常见有静态库 `.a` 和动态库 `.so`
- 库的本质是“声明 + 实现 + 链接 + 运行时加载（动态库）”
- 驱动 API 封装层负责把系统调用和命令码细节隐藏在统一函数后面
- 推荐先设计稳定头文件，再实现 `ioctl` 映射函数
- 工程中要重视错误码、线程安全、版本兼容和测试

---

# 9 中断

## 9.1 Linux 下的中断与原理

### 9.1.1 什么是中断

中断是硬件或软件向 CPU 发出的“异步事件通知”。

当中断到来时，CPU 暂停当前执行流，进入中断处理流程，执行完后再返回原任务。

驱动里最常见的中断来源：

- GPIO 电平/边沿变化
- UART 收发事件
- 定时器到期
- DMA 传输完成

### 9.1.2 IRQ number、HW interrupt ID、irq domain

这三个概念是 Linux 中断子系统里最容易混淆的部分。

1. `HW interrupt ID`（硬件中断号）
2. `IRQ number`（Linux 虚拟中断号）
3. `irq domain`（映射域）

#### 1) 什么是 HW interrupt ID

`HW interrupt ID` 是中断控制器硬件视角的编号（也常叫 hwirq）。

例如在 GIC 里，控制器寄存器和设备树里描述的是这类编号。

它的特点：

- 与具体中断控制器实现有关
- 可能在不同控制器里重复
- 不适合直接作为 Linux 通用中断接口参数

#### 2) 什么是 IRQ number

`IRQ number` 是 Linux 内核分配给驱动使用的“逻辑中断号”（虚拟号）。

驱动调用 `request_irq()`、`enable_irq()`、`disable_irq()` 时使用的是这个号。

它的特点：

- 在内核中统一管理
- 对驱动层屏蔽硬件细节
- 与具体控制器可解耦

#### 3) 什么是 irq domain

`irq domain` 是内核用于建立 `hwirq -> irq` 映射关系的机制。

可理解为“中断号翻译器”：

- 输入：控制器/设备树给出的硬件号（hwirq）
- 输出：驱动能用的 Linux IRQ 号

为什么要有它：

- 现代 SoC 常有多级中断控制器（GPIO 控制器级联到 GIC）
- 单靠一个全局硬件号体系难以管理
- 通过 domain 分层可支持级联、层次化中断控制器

常见使用场景：

- GPIO 控制器把 pin 中断映射成父控制器 IRQ
- MSI/MSI-X（PCIe）中断动态分配
- 虚拟化场景下中断重映射

### 9.1.3 Linux 中断处理分层

Linux 常把中断处理分为两部分：

1. 顶半部（hardirq）
2. 底半部（softirq/tasklet）

顶半部特点：

- 立即响应，优先级高
- 执行时间要短
- 不能睡眠（不能调用可能阻塞的接口）

底半部特点：

- 用于延后处理耗时逻辑
- 可以减少中断关闭时间
- 本章重点使用 tasklet 机制

### 9.1.4 中断注册与释放流程

典型流程：

1. 获取中断号（IRQ number）
2. 调用 `request_irq()` 注册处理函数
3. 在中断函数中做最小化处理
4. 在卸载路径调用 `free_irq()` 释放

常用原型：

```c
int request_irq(unsigned int irq, irq_handler_t handler,
                unsigned long flags, const char *name, void *dev);

void free_irq(unsigned int irq, void *dev);
```

### 9.1.5 中断源类型：SGI/PPI/SPI/LPI

“中断源”就是能触发中断的来源。GIC 体系里常见四类源：

1. SGI（Software Generated Interrupt）
2. PPI（Private Peripheral Interrupt）
3. SPI（Shared Peripheral Interrupt）
4. LPI（Locality-specific Peripheral Interrupt）

#### SGI

- 软件触发（通常核间通信 IPI）
- 面向 CPU 内核间协作，不是普通外设中断
- 常用于调度、TLB 刷新、核间唤醒

#### PPI

- 每个 CPU 私有的外设中断
- 同一个中断号在不同 CPU 上各自独立
- 常见于本地定时器、性能监控等

#### SPI

- SoC 共享外设中断
- 由 GIC 分发到一个或多个 CPU
- 普通驱动开发最常接触（GPIO/UART/I2C/SPI 控制器等）

#### LPI（GICv3/v4 引入）

- 主要服务 MSI/MSI-X 等可扩展消息中断
- 数量大、可动态管理
- 常见于 PCIe 高并发设备场景

为什么要分这些类型：

- 区分“软件触发/硬件触发”来源
- 区分“CPU 私有/系统共享”分发模型
- 支持从传统 SoC 中断到海量消息中断（LPI）的扩展

怎么理解和使用：

- 普通驱动开发：多数处理 SPI（有时通过 GPIO 控制器级联）
- SMP 系统调优：会接触 SGI/PPI
- PCIe/MSI 大规模设备：要理解 LPI 与 ITS

---

## 9.2 GIC 控制器

### 9.2.1 GIC 是什么

GIC（Generic Interrupt Controller）是 ARM 架构中常见的通用中断控制器。

它负责：

- 接收外设中断请求
- 做优先级与屏蔽控制
- 把中断分发到某个 CPU 核心

### 9.2.2 GIC 基本组成

常见由两部分构成：

- Distributor（分发器）
- CPU Interface（CPU 接口）

简要理解：

- Distributor：管理全局中断使能、优先级、目标 CPU
- CPU Interface：把“已经选中的中断”送到当前 CPU

### 9.2.3 在驱动里与 GIC 的关系

驱动开发者通常不直接操作 GIC 寄存器（除极少数底层 BSP 场景）。

常规驱动只需要：

- 从设备树或 GPIO 子系统拿到 IRQ 号
- 调用内核中断 API（`request_irq`、`disable_irq`、`enable_irq` 等）

GIC 细节由中断子系统和控制器驱动统一处理。

### 9.2.4 GICv1 / GICv2 / GICv3 / GICv4 区别

> 注：工程里常见的是 GICv2、GICv3、GICv4，GICv1 主要在早期系统中出现。

| 版本    | 典型特性                          | 常见场景             |
| ----- | ----------------------------- | ---------------- |
| GICv1 | 早期基础版本，功能较简单                  | 老平台、历史系统         |
| GICv2 | 广泛使用；支持传统 IRQ 分发；虚拟化支持较基础     | 32 位/早期 64 位 SoC |
| GICv3 | 64 位系统主流；引入系统寄存器接口；支持 ITS/LPI | 现代 ARMv8 SoC、服务器 |
| GICv4 | 在 v3 基础上增强虚拟化中断注入效率           | 高性能虚拟化平台         |

重点差异可这样理解：

- v2 -> v3：从传统寄存器访问向系统寄存器接口演进，并引入 LPI/ITS，支持海量 MSI 类中断。
- v3 -> v4：进一步优化虚拟机中断路径，降低虚拟化中断注入开销。

### 9.2.5 驱动开发者应该关心什么

大多数驱动不需要分支判断 “我是 GICv2 还是 v3”。

优先关注：

- 正确从设备树/ACPI 获取 IRQ
- 正确设置触发类型（上升沿、下降沿、电平）
- 顶半部最小化，重处理放到底半部
- 对共享中断和 CPU 亲和性有清晰设计

当你处理 PCIe MSI、高并发网络/存储、虚拟化平台时，再深入关注 LPI/ITS/GICv4 细节。

---

## 9.3 如何将 GPIO 中断注册到内核

### 9.3.1 注册目标与标准流程

GPIO 中断注册到内核的目标是：

- 将 GPIO 引脚映射为 Linux IRQ 号
- 把中断处理函数绑定到该 IRQ
- 在模块退出时正确释放中断资源

标准流程如下：

1. `gpio_to_irq(gpio_num)`：GPIO 编号映射为 IRQ
2. `request_irq(...)` 或 `request_threaded_irq(...)`：向内核申请并注册中断
3. `free_irq(irq, dev_id)`：注销中断

### 9.3.2 核心 API 函数分析

#### 1) gpio_to_irq

```c
int gpio_to_irq(unsigned int gpio);
```

作用：把 GPIO 全局编号转换为 Linux IRQ 号。

返回值：

- `>= 0`：有效 IRQ 号
- `< 0`：错误码

#### 2) request_irq

```c
int request_irq(unsigned int irq, irq_handler_t handler,
                unsigned long flags, const char *name, void *dev);
```

作用：为 IRQ 注册顶半部处理函数。

参数要点：

- `irq`：Linux IRQ 号
- `handler`：中断处理函数
- `flags`：触发方式、共享标志等
- `name`：显示在 `/proc/interrupts` 的名字
- `dev`：共享中断识别指针（非共享可为 `NULL`）

#### 3) free_irq

```c
void free_irq(unsigned int irq, void *dev_id);
```

作用：释放已注册中断。

注意：若申请时使用共享中断，`dev_id` 必须与 `request_irq` 时一致。

### 9.3.3 request_irq 内核申请路径与 request_threaded_irq

`request_irq()` 是常用接口，但底层由 `request_threaded_irq()` 完成核心申请流程。

调用关系可概括为：

```text
request_irq()
  -> request_threaded_irq(irq, handler, NULL, flags, name, dev)
      -> __setup_irq(...)
          -> irq_domain 映射检查、irq_desc 查找
          -> 分配并挂接 irqaction
          -> 配置触发类型与共享策略
          -> enable_irq / 启用中断线路
```

`request_threaded_irq()` 原型：

```c
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,
                         irq_handler_t thread_fn,
                         unsigned long flags,
                         const char *name,
                         void *dev_id);
```

使用建议：

- 只需顶半部时，使用 `request_irq`。
- 需要把较重逻辑下放到线程上下文时，使用 `request_threaded_irq`。
- 若 `handler` 返回 `IRQ_WAKE_THREAD`，内核会调度 `thread_fn` 执行。

### 9.3.4 i.MX6UL 中 gpio_to_irq 引脚编号计算

在 i.MX6UL 平台，常见 GPIO 命名为 `GPIOx_IOy`，旧式全局 GPIO 编号常按以下公式计算：

```text
gpio_num = (bank - 1) * 32 + io_index
```

其中：

- `bank`：GPIO 组号（GPIO1、GPIO2、GPIO3...）
- `io_index`：组内引脚号（0~31）

示例：

- `GPIO1_IO13` -> `13`
- `GPIO2_IO00` -> `32`
- `GPIO3_IO05` -> `69`
- `GPIO5_IO01` -> `129`

因此 `gpio_to_irq(13)` 常对应 `GPIO1_IO13`。

> 注：在较新内核中 GPIO base 可能动态分配，工程实践更推荐通过设备树 + gpiod 接口获取 IRQ（如 `gpiod_to_irq()`）。

### 9.3.5 GPIO 中断注册案例（与代码截图一致）

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/gpio.h>

int irq;

irqreturn_t test_interrupt(int irq, void *args)
{
    printk("This is test_interrupt\n");
    return IRQ_HANDLED;
}

static int interrupt_irq_init(void)
{
    int ret;

    irq = gpio_to_irq(13);
    printk("irq is %d\n", irq);

    ret = request_irq(irq, test_interrupt, IRQF_TRIGGER_RISING, "test", NULL);
    if (ret < 0) {
        printk("request_irq is error\n");
        return -1;
    }

    return 0;
}

static void interrupt_irq_exit(void)
{
    free_irq(irq, NULL);
    printk("bye bye\n");
}

module_init(interrupt_irq_init);
module_exit(interrupt_irq_exit);

MODULE_LICENSE("GPL");
```

### 9.3.6 中断处理函数注意事项

在 hardirq 上下文中，应避免：

- 睡眠（如 `msleep`）
- 长时间循环
- 大量 `printk` 刷屏
- 复杂或可能阻塞的资源分配

推荐做法：

- 顶半部只做快速确认和状态记录
- 复杂处理下放到 `thread_fn` 或 tasklet

---

## 9.4 Tasklet 机制与注册

### 9.4.1 Tasklet 注册目标与执行流程

Tasklet 的目标是将中断顶半部中的后续处理延后到软中断上下文执行，从而缩短顶半部执行时间。

标准流程如下：

1. 定义 Tasklet 回调函数
2. 初始化 Tasklet（静态或动态）
3. 在 IRQ 顶半部调用 `tasklet_schedule()`
4. 需要时通过 `tasklet_disable/tasklet_enable` 控制执行
5. 模块退出前调用 `tasklet_kill()` 取消并回收

### 9.4.2 Tasklet 核心 API 分析

#### 1) 静态初始化函数一：DECLARE_TASKLET

```c
DECLARE_TASKLET(name, callback);
```

作用：静态定义并初始化一个 tasklet 对象（新式 callback 写法）。

> 说明：部分旧教材会写成 `DECLARE_TASKLET(name, func, data)`，那是旧式接口写法。

#### 2) 静态初始化函数二：DECLARE_TASKLET_DISABLED

```c
DECLARE_TASKLET_DISABLED(name, callback);
```

作用：静态定义一个“初始为禁用状态”的 tasklet。通常需要后续调用 `tasklet_enable()` 才能执行。

#### 3) 动态初始化函数：tasklet_init

```c
void tasklet_init(struct tasklet_struct *t,
                  void (*func)(unsigned long),
                  unsigned long data);
```

作用：动态初始化 tasklet，适合对象在运行时创建的场景。

补充：较新内核还提供 `tasklet_setup(struct tasklet_struct *t, void (*callback)(struct tasklet_struct *))`，用于新式 callback 风格初始化。

#### 4) tasklet_enable

```c
void tasklet_enable(struct tasklet_struct *t);
```

作用：使能 tasklet（配合 `tasklet_disable` 使用）。

#### 5) tasklet_disable

```c
void tasklet_disable(struct tasklet_struct *t);
```

作用：禁用 tasklet，常用于临界区保护，防止 tasklet 在关键数据修改期间运行。

常见成对写法：

```c
tasklet_disable(&test_tasklet);
/* 修改与 tasklet 共享的数据 */
tasklet_enable(&test_tasklet);
```

#### 6) tasklet_schedule（调度函数）

```c
void tasklet_schedule(struct tasklet_struct *t);
```

作用：把 tasklet 标记为待执行，由软中断机制在合适时机运行。

#### 7) tasklet_kill（取消调度/回收函数）

```c
void tasklet_kill(struct tasklet_struct *t);
```

作用：同步等待并停止 tasklet。可理解为“取消后续调度并回收”，常用于驱动卸载路径。

### 9.4.3 基于 9.3 GPIO 中断案例加入 Tasklet

下面示例在 9.3 的 GPIO 中断基础上，把“较重处理”下放到 tasklet 中执行。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/gpio.h>

static int irq;
static struct tasklet_struct test_tasklet;

static void test_tasklet_func(unsigned long data)
{
    printk("tasklet running, data=%lu\n", data);
    /* 这里执行延后处理（仍不可睡眠） */
}

static irqreturn_t test_interrupt(int irq, void *args)
{
    printk("This is test_interrupt\n");

    /* 顶半部只做快速处理，把后续工作下放 */
    tasklet_schedule(&test_tasklet);

    return IRQ_HANDLED;
}

static int interrupt_irq_init(void)
{
    int ret;

    /* 动态初始化 tasklet */
    tasklet_init(&test_tasklet, test_tasklet_func, 13);

    irq = gpio_to_irq(13);
    printk("irq is %d\n", irq);

    ret = request_irq(irq, test_interrupt, IRQF_TRIGGER_RISING, "test", NULL);
    if (ret < 0) {
        printk("request_irq is error\n");
        return -1;
    }

    return 0;
}

static void interrupt_irq_exit(void)
{
    free_irq(irq, NULL);
    tasklet_kill(&test_tasklet);
    printk("bye bye\n");
}

module_init(interrupt_irq_init);
module_exit(interrupt_irq_exit);

MODULE_LICENSE("GPL");
```

### 9.4.4 Tasklet 使用注意事项

- tasklet 运行在软中断上下文，不能睡眠。
- 同一个 tasklet 实例不会并发执行。
- 顶半部只负责快速确认，耗时逻辑通过 `tasklet_schedule` 延后。
- 卸载驱动前必须执行 `tasklet_kill`，避免回调访问已释放资源。

### 9.4.5 Tasklet 源码定义分析（基于当前内核 interrupt.h）

从当前内核头文件可以看到，Tasklet 接口包含“新式 callback 写法”和“旧式 func/data 写法”两套。

#### 1) struct tasklet_struct 的关键字段

```c
struct tasklet_struct {
    struct tasklet_struct *next;
    unsigned long state;
    atomic_t count;
    bool use_callback;
    union {
        void (*func)(unsigned long data);
        void (*callback)(struct tasklet_struct *t);
    };
    unsigned long data;
};
```

字段理解：

- `state`：调度/运行状态位（如 `TASKLET_STATE_SCHED`、`TASKLET_STATE_RUN`）
- `count`：禁用深度计数（`0` 表示可运行，`>0` 表示禁用）
- `func/callback`：旧式与新式回调入口
- `data`：旧式 `func(unsigned long)` 的参数

#### 2) 静态宏为什么和旧教程不一样

当前头文件中的新式宏是：

```c
DECLARE_TASKLET(name, callback)
DECLARE_TASKLET_DISABLED(name, callback)
```

旧教程常见：

```c
DECLARE_TASKLET(name, func, data)
DECLARE_TASKLET_DISABLED(name, func, data)
```

这是因为内核演进后引入了 callback 风格，同时保留了 `DECLARE_TASKLET_OLD` 兼容旧代码。

#### 3) tasklet_schedule 的“去重调度”特性

`tasklet_schedule()` 内部会先设置 `TASKLET_STATE_SCHED` 位，若该位已设置则不重复入队。

因此：

- 连续多次调用 `tasklet_schedule()`
- 在 tasklet 尚未开始执行前，通常只会执行一次

#### 4) tasklet_disable / tasklet_enable 的本质

从实现看：

- `tasklet_disable()` -> `atomic_inc(&t->count)`
- `tasklet_enable()` -> `atomic_dec(&t->count)`

这说明它们是“禁用深度计数”而非简单布尔开关。

使用规则：

- `disable` 与 `enable` 必须配对
- 未 `disable` 过不要随意 `enable`
- 退出路径一般不需要 `enable`，而是 `free_irq + tasklet_kill`

#### 5) 结合本章案例的接口选择建议

- 维护旧风格驱动：可继续用 `tasklet_init + func(unsigned long)`
- 写新代码：优先使用 callback 风格（`DECLARE_TASKLET` / `tasklet_setup`）
- 需要可睡眠下半部时，应考虑 threaded IRQ 方案

---

## 9.5 软中断

### 9.5.1 软中断基本概念

软中断（softirq）是 Linux 下半部机制之一，运行在软中断上下文。

它的定位是：

- 把顶半部中不适合立即做完的工作延后执行
- 保持较高吞吐，适合高频事件处理

软中断特点：

- 不能睡眠
- 可能在多 CPU 并发执行（同类型 softirq 在不同 CPU 上可并发）
- 处理函数要短小、无阻塞

### 9.5.2 软中断接口函数

结合 `include/linux/interrupt.h`，常见接口有：

```c
void open_softirq(int nr, void (*action)(struct softirq_action *));
void raise_softirq(unsigned int nr);
void raise_softirq_irqoff(unsigned int nr);
void __raise_softirq_irqoff(unsigned int nr);
asmlinkage void do_softirq(void);
asmlinkage void __do_softirq(void);
```

用途说明：

- `open_softirq`：注册某个 softirq 向量对应的处理函数（通常系统初始化阶段）
- `raise_softirq`：触发一个 softirq（会处理本地中断状态）
- `raise_softirq_irqoff`：在本地中断已关闭场景触发 softirq（如 hardirq 中）
- `do_softirq/__do_softirq`：执行软中断处理流程（通常由内核路径调用）

实践注意：

- 直接 `open_softirq` 通常是内核核心代码路径使用
- 普通可卸载驱动更常用 tasklet 或 threaded IRQ

### 9.5.3 软中断枚举类型

`interrupt.h` 中定义了软中断类型枚举：

```c
enum {
    HI_SOFTIRQ = 0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    IRQ_POLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ,
    RCU_SOFTIRQ,
    NR_SOFTIRQS
};
```

常见理解：

- `HI_SOFTIRQ`：高优先级 softirq
- `TIMER_SOFTIRQ/HRTIMER_SOFTIRQ`：定时器相关
- `NET_TX_SOFTIRQ/NET_RX_SOFTIRQ`：网络收发路径
- `TASKLET_SOFTIRQ`：tasklet 运行依赖的 softirq 类型
- `RCU_SOFTIRQ`：RCU 回调处理

### 9.5.4 基于 9.3 GPIO 中断案例的软中断示例

以下是教学示例：在 GPIO 中断顶半部中触发 softirq，把后续处理延后到 softirq 回调。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/gpio.h>

static int irq;

#define TEST_SOFTIRQ HI_SOFTIRQ

static void testsoft_func(struct softirq_action *softirq_action)
{
    printk("This is testsoft_func\n");
}

static irqreturn_t test_interrupt(int irq, void *args)
{
    printk("This is test_interrupt\n");
    raise_softirq(TEST_SOFTIRQ);
    return IRQ_HANDLED;
}

static int interrupt_irq_init(void)
{
    int ret;

    irq = gpio_to_irq(13);
    printk("irq is %d\n", irq);

    ret = request_irq(irq, test_interrupt, IRQF_TRIGGER_RISING, "test", NULL);
    if (ret < 0) {
        printk("request_irq is error\n");
        return -1;
    }

    open_softirq(TEST_SOFTIRQ, testsoft_func);

    return 0;
}

static void interrupt_irq_exit(void)
{
    free_irq(irq, NULL);
    printk("bye bye\n");
}

module_init(interrupt_irq_init);
module_exit(interrupt_irq_exit);
MODULE_LICENSE("GPL");
```

流程说明：

1. GPIO 上升沿触发硬中断，进入 `test_interrupt`
2. 中断顶半部调用 `raise_softirq(TEST_SOFTIRQ)` 触发软中断
3. 内核在软中断处理时机调用 `testsoft_func`
4. 在软中断回调中完成延后处理逻辑

### 9.5.5 工程建议

- 软中断回调中仍不能睡眠。
- 软中断适合高频、短小、无阻塞处理。
- 对于可卸载驱动，优先考虑 tasklet 或 threaded IRQ；直接 `open_softirq` 要特别注意生命周期管理。

---

## 9.6 工作队列专题

在 Linux 驱动里，工作队列相关对象常见有三类：

- 工作队列（共享队列，`system_wq`）：内核已经创建好，直接投递任务，使用简单。
- 自定义工作队列（private wq）：驱动自己创建/销毁，可控制并发策略和隔离性。
- 延迟队列（`delayed_work`）：本质是“工作项 + 延时触发”，适合定时延后执行。

它们的特点可总结为：

- 共同点：都运行在进程上下文，回调中可以睡眠。
- 共享队列：开发成本低，但与系统其它任务共享资源。
- 自定义队列：隔离性强，可做有序执行或限制并发，但需要完整生命周期管理。
- 延迟队列：天然支持“过一段时间再执行”，常用于消抖、重试、延时上报。

---

### 9.6.1 工作队列

工作队列（这里主要指共享队列 `system_wq`）是最基础的使用方式。
驱动只需要准备工作项和回调函数，然后通过调度 API 投递即可。

适用场景：

- 中断下半部里需要执行可睡眠逻辑
- 任务量不大，不需要独立线程池隔离
- 需要快速完成开发验证

#### 9.6.1.1 工作队列 API 函数

1. 工作项定义与初始化

```c
void my_work_func(struct work_struct *work);

DECLARE_WORK(my_work, my_work_func);     /* 静态初始化 */

struct work_struct dyn_work;
INIT_WORK(&dyn_work, my_work_func);      /* 动态初始化 */
```

2. 延迟工作定义与初始化

```c
void my_delay_func(struct work_struct *work);

DECLARE_DELAYED_WORK(my_dwork, my_delay_func);

struct delayed_work dyn_dwork;
INIT_DELAYED_WORK(&dyn_dwork, my_delay_func);
```

3. 调度与取消

```c
bool schedule_work(struct work_struct *work);
bool schedule_delayed_work(struct delayed_work *dwork, unsigned long delay);
bool cancel_work_sync(struct work_struct *work);
bool cancel_delayed_work_sync(struct delayed_work *dwork);
void flush_work(struct work_struct *work);
```

说明：

- `schedule_work`：立即调度到共享队列
- `schedule_delayed_work`：延时后调度
- `cancel_*_sync`：用于模块退出，保证回调不再运行
- `flush_work`：等待指定工作项执行完成

#### 9.6.1.2 工作队列案例（共享队列 + 延迟队列）

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/workqueue.h>
#include <linux/delay.h>

static struct work_struct demo_work;
static struct delayed_work demo_dwork;

static void demo_work_func(struct work_struct *work)
{
    msleep(200);
    printk("demo_work_func run\n");
}

static void demo_delay_func(struct work_struct *work)
{
    printk("demo_delay_func run after delay\n");
}

static int __init demo_init(void)
{
    INIT_WORK(&demo_work, demo_work_func);
    INIT_DELAYED_WORK(&demo_dwork, demo_delay_func);

    schedule_work(&demo_work);
    schedule_delayed_work(&demo_dwork, msecs_to_jiffies(1000));
    return 0;
}

static void __exit demo_exit(void)
{
    cancel_work_sync(&demo_work);
    cancel_delayed_work_sync(&demo_dwork);
    printk("demo exit\n");
}

module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

流程说明：

1. 初始化普通工作和延迟工作
2. 立即投递普通工作
3. 1 秒后执行延迟工作
4. 退出时同步取消，避免回调悬挂

---

### 9.6.2 自定义工作队列

自定义工作队列用于“把本驱动工作与系统公共工作隔离”，并可配置并发行为。

适用场景：

- 任务量较大，希望避免影响系统共享队列
- 需要严格顺序执行（ordered workqueue）
- 需要自定义并发上限或队列属性

#### 9.6.2.1 自定义工作队列 API 函数

1. 创建队列

```c
struct workqueustruct *create_workqueue(const char *name);
```

2. 投递与取消

```c
bool queue_work(struct workqueue_struct *wq, struct work_struct *work);
bool queue_delayed_work(struct workqueue_struct *wq,
                        struct delayed_work *dwork,
                        unsigned long delay);
bool cancel_work_sync(struct work_struct *work);
bool cancel_delayed_work_sync(struct delayed_work *dwork);
```

3. 同步与销毁

```c
void flush_workqueue(struct workqueue_struct *wq);
void destroy_workqueue(struct workqueue_struct *wq);
```

说明：

- `create_workqueue` 为历史接口，推荐优先使用 `alloc_workqueue` 系列
- `alloc_ordered_workqueue` 适合必须保序的业务
- 销毁前建议先 `cancel`，再 `flush`，最后 `destroy`

#### 9.6.2.2 自定义工作队列案例

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/interrupt.h>
#include <linux/gpio.h>
#include <linux/workqueue.h>
#include <linux/delay.h>

static int irq;

struct work_struct test_work;
struct workqueue_struct *test_workqueue;

static void test_work_func(struct work_struct *work)
{
    msleep(1000);
    printk(KERN_INFO "test_work_func called\n");
}

static irqreturn_t test_interrupt(int irq, void *args)
{
    printk(KERN_INFO "test_interrupt called, irq:%d\n", irq);
    queue_work(test_workqueue, &test_work);
    return IRQ_HANDLED;
}

static int __init interrupt_workqueue_init(void)
{
    int ret;

    irq = gpio_to_irq(13);
    printk(KERN_INFO "gpio_to_irq ret:%d\n", irq);

    ret = request_irq(irq, test_interrupt, IRQF_TRIGGER_RISING, "interrupt_workqueue_test", NULL);
    if (ret < 0)
    {
        printk(KERN_ERR "request_irq failed\n");
        return ret;
    }

    test_workqueue = create_workqueue("test_workqueue");

    INIT_WORK(&test_work, test_work_func);


    return 0;
}

static void __exit interrupt_workqueue_exit(void)
{
    free_irq(irq, NULL);
    cancel_work_sync(&test_work);
    flush_workqueue(test_workqueue);
    destroy_workqueue(test_workqueue);
}

module_init(interrupt_workqueue_init);
module_exit(interrupt_workqueue_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tengfei");
```

流程说明：

1. 初始化时创建自定义队列并初始化工作项
2. 中断触发后在 ISR 中调用 `queue_work`
3. 工作线程执行 `test_work`，可安全 `msleep`
4. 退出路径执行 `cancel` + `flush` + `destroy`，确保无悬挂回调

---

### 9.6.3 延迟工作队列

延迟工作队列基于 `struct delayed_work`，用于“不是立刻执行，而是延后一段时间再执行”的场景。

典型用途：

- 按固定延时执行一次任务
- 按条件重新投递，形成周期性轮询
- 中断后延时处理（例如按键消抖）

#### 9.6.3.1 延迟工作队列 API 函数

1. 初始化延迟工作项

```c
void delay_fn(struct work_struct *work);

DECLARE_DELAYED_WORK(my_dwork, delay_fn);

struct delayed_work dyn_dwork;
INIT_DELAYED_WORK(&dyn_dwork, delay_fn);
```

2. 在共享队列调度

```c
bool schedule_delayed_work(struct delayed_work *dwork, unsigned long delay);
bool mod_delayed_work(struct workqueue_struct *wq,
                      struct delayed_work *dwork,
                      unsigned long delay);
```

说明：

- `schedule_delayed_work`：把延迟工作投递到共享队列

- `mod_delayed_work(system_wq, ...)`：可修改下次触发时间；若未排队则直接排队
3. 在自定义工作队列调度

```c
bool queue_delayed_work(struct workqueue_struct *wq,
                        struct delayed_work *dwork,
                        unsigned long delay);
bool mod_delayed_work(struct workqueue_struct *wq,
                      struct delayed_work *dwork,
                      unsigned long delay);
```

4. 取消与同步

```c
bool cancel_delayed_work(struct delayed_work *dwork);
bool cancel_delayed_work_sync(struct delayed_work *dwork);
```

说明：

- `cancel_delayed_work`：只保证取消排队，不等待正在执行的回调
- `cancel_delayed_work_sync`：会等待回调结束，模块卸载更安全

#### 9.6.3.2 延迟工作队列案例

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/workqueue.h>
#include <linux/jiffies.h>

static struct workqueue_struct *delay_wq;
static struct delayed_work delay_work;

static void delay_work_func(struct work_struct *work)
{
    printk("delay_work_func run, jiffies=%lu\n", jiffies);

    /* 示例：每隔 2 秒再次执行，形成周期性任务 */
    queue_delayed_work(delay_wq, &delay_work, msecs_to_jiffies(2000));
}

static int __init delay_demo_init(void)
{
    delay_wq = create_workqueue("delay_wq");
    if (!delay_wq) {
        printk("alloc_ordered_workqueue failed\n");
        return -ENOMEM;
    }

    INIT_DELAYED_WORK(&delay_work, delay_work_func);

    /* 首次延时 1 秒触发 */
    queue_delayed_work(delay_wq, &delay_work, msecs_to_jiffies(1000));
    printk("delay_demo_init ok\n");
    return 0;
}

static void __exit delay_demo_exit(void)
{
    cancel_delayed_work_sync(&delay_work);

    if (delay_wq) {
        flush_workqueue(delay_wq);
        destroy_workqueue(delay_wq);
        delay_wq = NULL;
    }

    printk("delay_demo_exit\n");
}

module_init(delay_demo_init);
module_exit(delay_demo_exit);
MODULE_LICENSE("GPL");
```

流程说明：

1. 初始化时创建自定义队列并初始化 `delayed_work`
2. 首次通过 `queue_delayed_work` 延时 1 秒执行
3. 回调内部再次投递，实现周期性延时执行
4. 退出时 `cancel_delayed_work_sync` + `flush_workqueue` + `destroy_workqueue`

---

### 9.6.4 工作队列传参

在工作队列回调里，函数原型固定为：

```c
void func(struct work_struct *work);
```

因此不能像普通函数那样直接写多个参数。工程里常用做法是：

1. 自定义一个业务结构体
2. 把 `struct work_struct` 作为该结构体成员
3. 在回调中通过 `container_of` 从 `work` 反推到业务结构体

这样就能在回调中安全拿到 `a`、`b` 等自定义字段。

#### 9.6.4.1 关键 API 与宏

```c
void INIT_WORK(struct work_struct *work, work_func_t func);
bool queue_work(struct workqueue_struct *wq, struct work_struct *work);

#define container_of(ptr, type, member) ...
```

说明：

- `INIT_WORK`：绑定工作项与回调函数
- `queue_work`：把工作项放入指定工作队列
- `container_of`：由成员指针反推出宿主结构体指针

#### 9.6.4.2 案例：按图实现工作队列传参

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/gpio.h>

static int irq;

struct work_data {
    struct work_struct test_work;
    int a;
    int b;
};

static struct work_data test_workqueue_work;
static struct workqueue_struct *test_workqueue;

static void test_work(struct work_struct *work)
{
    struct work_data *pdata;

    pdata = container_of(work, struct work_data, test_work);

    printk("a is %d\n", pdata->a);
    printk("b is %d\n", pdata->b);
}

static irqreturn_t test_interrupt(int irq, void *args)
{
    printk("this is test_interrupt\n");
    queue_work(test_workqueue, &test_workqueue_work.test_work);
    return IRQ_RETVAL(IRQ_HANDLED);
}

static int __init interrupt_irq_init(void)
{
    int ret;

    irq = gpio_to_irq(13);
    printk("irq is %d\n", irq);

    ret = request_irq(irq, test_interrupt, IRQF_TRIGGER_RISING, "test", NULL);
    if (ret < 0) {
        printk("request_irq is error\n");
        return -1;
    }

    test_workqueue = create_workqueue("test_workqueue");
    if (!test_workqueue) {
        free_irq(irq, NULL);
        return -ENOMEM;
    }

    INIT_WORK(&test_workqueue_work.test_work, test_work);

    test_workqueue_work.a = 1;
    test_workqueue_work.b = 2;

    return 0;
}

static void __exit interrupt_irq_exit(void)
{
    free_irq(irq, NULL);

    cancel_work_sync(&test_workqueue_work.test_work);

    if (test_workqueue) {
        flush_workqueue(test_workqueue);
        destroy_workqueue(test_workqueue);
        test_workqueue = NULL;
    }

    printk("bye bye\n");
}

module_init(interrupt_irq_init);
module_exit(interrupt_irq_exit);
MODULE_LICENSE("GPL");
```

流程说明：

1. 定义 `struct work_data`，把工作项和业务参数放在同一结构体内
2. 中断触发后调用 `queue_work(test_workqueue, &test_workqueue_work.test_work)`
3. 回调函数中通过 `container_of` 还原 `struct work_data *`
4. 读取并打印 `a`、`b`，完成“工作队列传参”
5. 退出时按 `cancel` + `flush` + `destroy` 回收队列资源

---

### 9.6.5 CMWQ（并发管理工作队列）

CMWQ（Concurrency-Managed Workqueue）可以理解为 Linux 新一代工作队列实现。
它的目标是：在保证功能一致的前提下，把“线程池管理、并发控制、资源复用”统一交给内核，而不是每个 workqueue 都单独起大量内核线程。

核心原理：

- 内核维护 worker-pool，多个工作队列可共享底层 worker 资源
- 根据负载动态唤醒/回收 worker，避免线程数量失控
- 通过 `max_active` 和 flags 控制并发度、优先级、绑定策略
- 驱动仍按“提交 work -> worker 执行回调”的模型使用，使用方式基本不变

这也是你提到的关键点：
对驱动编写者来说，很多调度与回收 API 仍然类似，最明显变化主要在“创建队列的 API 和参数能力”上。

#### 9.6.5.1 CMWQ 并发管理机制

1. worker-pool 复用
- 旧模型倾向于每队列配线程，数量较多时开销大

- CMWQ 采用共享 worker-pool，降低线程创建和切换成本
2. 并发上限控制
- `alloc_workqueue(..., max_active, ...)` 可以设置并发上限

- `max_active = 1` 等价“串行化”执行，适合必须保序的场景
3. 执行属性控制
- 通过 flags 声明队列行为，例如是否高优先级、是否可冻结、是否内存回收路径可用

#### 9.6.5.2 CMWQ 常用 API 函数

1. 推荐创建接口

```c
struct workqueue_struct *alloc_workqueue(const char *fmt,
                                         unsigned int flags,
                                         int max_active, ...);
```

常见参数说明：

- `fmt`：队列名称格式串

- `flags`：队列属性

- `max_active`：并发工作项上限
2. 有序队列创建

```c
struct workqueue_struct *alloc_ordered_workqueue(const char *fmt,
                                                 unsigned int flags, ...);
```

说明：

- 有序队列本质是并发度为 1 的有序执行模型

- 适合寄存器时序、状态机推进等严格顺序业务
3. 常见 flags（按需选用）

```c
WQ_UNBOUND
WQ_HIGHPRI
WQ_MEM_RECLAIM
WQ_FREEZABLE
WQ_CPU_INTENSIVE
```

4. 调度与销毁接口（与前文一致）

```c
bool queue_work(struct workqueue_struct *wq, struct work_struct *work);
bool queue_delayed_work(struct workqueue_struct *wq,
                        struct delayed_work *dwork,
                        unsigned long delay);
void flush_workqueue(struct workqueue_struct *wq);
void destroy_workqueue(struct workqueue_struct *wq);
```

5. 兼容性说明
- `create_workqueue` 属于历史接口，教学中可见，但新代码更建议 `alloc_workqueue`
- 从迁移视角看，业务回调和提交流程通常不用大改，重点是按场景设置 `flags + max_active`

---

### 9.6.6 中断线程化

中断线程化（threaded IRQ）就是把中断处理拆成两部分：

- 顶半部（hardirq handler）：尽快返回，不做耗时工作
- 中断线程（thread_fn）：在内核线程上下文执行，可睡眠、可做较慢处理

它的目标是降低 hardirq 停留时间，提高系统实时性与稳定性。

典型适用场景：

- 中断后处理逻辑较长
- 需要 `msleep`、`mutex`、I2C/SPI 访问等可睡眠操作
- 不希望使用 workqueue 再二次转发

#### 9.6.6.1 中断线程化 API 函数

```c
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,
                         irq_handler_t thread_fn,
                         unsigned long flags,
                         const char *name,
                         void *dev);

void free_irq(unsigned int irq, void *dev_id);
```

参数说明：

- `irq`：中断号
- `handler`：顶半部函数（可为 `NULL`）
- `thread_fn`：中断线程函数（线程上下文执行）
- `flags`：触发方式和行为标志（如 `IRQF_TRIGGER_RISING`）
- `name`：中断名字
- `dev`：设备私有指针，`free_irq` 时需保持一致

返回值说明：

- `0`：注册成功
- `< 0`：注册失败（负错误码）

中断回调的返回值语义：

1. 顶半部 `handler` 返回值
- `IRQ_NONE`：该中断不是本设备产生
- `IRQ_HANDLED`：本设备已在顶半部处理完成，不需要线程化继续处理
- `IRQ_WAKE_THREAD`：请求内核唤醒对应 `thread_fn` 线程
2. 线程函数 `thread_fn` 返回值
- 通常返回 `IRQ_HANDLED`
- 也可写成 `IRQ_RETVAL(IRQ_HANDLED)`，语义是“线程阶段已完成处理中断”

为什么返回值可以唤醒中断线程：

- `request_threaded_irq` 在注册时把 `handler` 与 `thread_fn` 绑定到同一个 irq action
- 当中断发生时，内核先执行 `handler`
- 若 `handler` 返回 `IRQ_WAKE_THREAD`，中断子系统会检查该 action 存在 `thread_fn`
- 随后把该 irq 的线程置为可运行状态（wake up），由调度器安排执行 `thread_fn`

简化理解就是：`IRQ_WAKE_THREAD` 是“从 hardirq 向 irq 线程发出的唤醒信号”。

#### 9.6.6.2 中断线程化案例（按图）

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/gpio.h>
#include <linux/delay.h>

static int irq;

static irqreturn_t test_work(int irq, void *args)
{
    msleep(1000);
    printk("this is test_work\n");
    return IRQ_RETVAL(IRQ_HANDLED);
}

static irqreturn_t test_interrupt(int irq, void *args)
{
    printk("this is test_interrupt\n");
    return IRQ_WAKE_THREAD;
}

static int __init interrupt_irq_init(void)
{
    int ret;

    irq = gpio_to_irq(13);
    printk("irq is %d\n", irq);

    ret = request_threaded_irq(irq,
                               test_interrupt,
                               test_work,
                               IRQF_TRIGGER_RISING,
                               "test",
                               NULL);
    if (ret < 0) {
        printk("request_irq is error\n");
        return -1;
    }

    return 0;
}

static void __exit interrupt_irq_exit(void)
{
    free_irq(irq, NULL);
    printk("bye bye\n");
}

module_init(interrupt_irq_init);
module_exit(interrupt_irq_exit);
MODULE_LICENSE("GPL");
```

流程说明：

1. GPIO 上升沿触发中断，先进入 `test_interrupt`
2. 顶半部打印后返回 `IRQ_WAKE_THREAD`
3. 内核据此唤醒对应 irq 线程，执行 `test_work`
4. 线程函数中可 `msleep(1000)`，最后返回 `IRQ_RETVAL(IRQ_HANDLED)`
5. 模块退出时 `free_irq` 释放中断

工程建议：

- 顶半部只做最小化处理，把耗时逻辑放到 `thread_fn`
- 若设备不支持并行重入，常配合 `IRQF_ONESHOT` 使用（防止线程未完成时重复进中断）

---

## 9.7 本章小结

- Linux 中断处理建议采用“顶半部快处理 + 下半部延后处理”
- GIC 负责 ARM 平台中断分发，普通驱动通常通过统一 IRQ API 间接使用
- GPIO 中断注册核心步骤：`gpio_to_irq` + `request_irq` + `free_irq`
- tasklet 和 softirq 适合不可睡眠下半部；workqueue 适合可睡眠、可阻塞任务
- 工作队列常用 API 包括：初始化、调度、取消、`flush`、`destroy`
- 延迟工作队列可在共享队列或自定义队列中调度，适合消抖、重试、周期任务
- 工作队列传参推荐“结构体嵌入 `work_struct` + `container_of` 反向取参”模式
- CMWQ 通过 `alloc_workqueue(flags, max_active)` 提供并发管理能力，是当前推荐模型
- 中断线程化通过 `request_threaded_irq` 把耗时逻辑下放到 irq 线程，常用 `IRQ_WAKE_THREAD` 触发
- 可卸载驱动必须在退出路径中完整回收中断与工作队列资源
