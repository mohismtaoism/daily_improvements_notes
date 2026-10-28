# 🎓 Linux 1-Wire Driver Training Course

## 嵌入式Linux驱动开发实战 — ROM-3310 1-Wire EEPROM驱动

---

**适用硬件：** ROM-3310 (AM3358) + Puhui Carrier Board
**内核版本：** Linux 3.2.0 (TI PSP 04.06.00.11)
**目标芯片：** GX2431 (0x2D, 1024-bit EEPROM), GXE00 (0x14, 256-bit EEPROM), DS28E25 (0x47, legacy)

---

# 课程概述

## 你将学到什么

| 模块 | 内容 | 难度 |
|------|------|------|
| 模块一 | Linux内核模块基础框架 | ⭐ |
| 模块二 | 1-Wire子系统架构与核心数据结构 | ⭐⭐ |
| 模块三 | 简单EEPROM驱动：DS2431源码精读 | ⭐⭐ |
| 模块四 | 从零编写GXE00驱动 | ⭐⭐⭐ |
| 模块五 | 集成与构建系统 | ⭐ |
| 模块六 | 安全认证型驱动：DS28E25架构分析 | ⭐⭐⭐⭐ |
| 模块七 | 独立编程练习 | ⭐⭐~⭐⭐⭐ |

---

# 模块一：Linux内核模块基础框架

## 1.1 什么是内核模块？

Linux内核有两种方式包含驱动代码：

- **Built-in (=y)**：编译时直接链接进内核镜像，系统启动时自动初始化
- **Module (=m)**：编译为独立的 `.ko` 文件，运行时动态加载/卸载

在这个BSP中，所有1-Wire驱动都是built-in (`=y`)。

## 1.2 最小的内核模块

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init my_driver_init(void)
{
    printk(KERN_INFO "Hello from my first driver!\n");
    return 0;   // 0 = success, negative = error
}

static void __exit my_driver_exit(void)
{
    printk(KERN_INFO "Goodbye from my first driver!\n");
}

module_init(my_driver_init);
module_exit(my_driver_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple hello world driver");
```

## 1.3 关键概念

### `__init` 和 `__exit` 标记

```c
static int __init my_func(void)  // → 这个函数在初始化完成后可以被内核释放
static void __exit my_func(void) // → 如果驱动是built-in，这个函数被丢弃以节省空间
```

- `__init`：标记初始化函数。内核在调用完所有init函数后，会回收这些代码所占的内存
- `__exit`：标记退出函数。对于built-in驱动，exit函数永远不会被调用，所以代码直接被丢弃

### 返回值约定

| 返回值 | 含义 |
|--------|------|
| `0` | 初始化成功 |
| `-ENOMEM` | 内存分配失败 |
| `-EIO` | I/O错误 |
| `-ENODEV` | 设备不存在或不支持 |
| `-EBUSY` | 资源忙 |

### MODULE_* 宏

```c
MODULE_LICENSE("GPL");        // 必须！内核不接受非GPL模块的某些API调用
MODULE_AUTHOR("Your Name");   // 作者信息
MODULE_DESCRIPTION("Desc");   // 描述
MODULE_VERSION("1.0");        // 版本号
```

## 1.4 编译内核模块的方式

对于built-in驱动：
```makefile
obj-$(CONFIG_W1_SLAVE_GXE00) += w1_gxe00.o
```

当 `CONFIG_W1_SLAVE_GXE00=y` 时，`w1_gxe00.c` 被编译并链接进内核。

## 1.5 内核的打印函数

```c
printk(KERN_EMERG "emerg\n");    // 0: 系统不可用
printk(KERN_ALERT "alert\n");    // 1: 需要立即处理
printk(KERN_CRIT "crit\n");      // 2: 严重情况
printk(KERN_ERR "error\n");      // 3: 错误
printk(KERN_WARNING "warn\n");   // 4: 警告
printk(KERN_INFO "info\n");      // 6: 信息
printk(KERN_DEBUG "debug\n");    // 7: 调试

// 快捷宏（推荐）
pr_emerg(), pr_alert(), pr_crit(), pr_err(), pr_warn(), pr_info(), pr_debug()
dev_info(&sl->dev, "device found\n");    // 设备相关输出，自动包含设备名
```

---

# 模块二：1-Wire子系统架构

## 2.1 整体架构层次

```
Userspace (应用程序)
     │
     │ sysfs (/sys/bus/w1/devices/14-xxxxxxxxxxxx/eeprom)
     │ ioctl (/dev/ds28e25)
     ▼
┌─────────────────────────────────────┐
│        W1 Slave Driver              │ ← 你的代码在这里
│  (w1_gxe00.c, w1_ds2431.c, ...)     │
└─────────────────────────────────────┘
     │ w1_family_ops.add_slave/remove_slave
     ▼
┌─────────────────────────────────────┐
│        W1 Core  (w1.c)              │ ← 内核W1框架
│  - 总线扫描 (枚举ROM ID)             │
│  - 设备匹配 (按family code)          │
│  - 读写接口 (w1_read_byte, ...)      │
└─────────────────────────────────────┘
     │ w1_bus_master 函数表
     ▼
┌─────────────────────────────────────┐
│     W1 Master Driver (ds2484.c)     │ ← I2C→1-Wire桥接芯片
│  - I2C设备驱动                       │
│  - 实现了reset_bit, touch_bit等      │
└─────────────────────────────────────┘
     │ I2C bus
     ▼
┌─────────────────────────────────────┐
│     硬件: DS2484 (I2C addr 0x18)    │
│           ↓                         │
│     1-Wire Bus                      │
│      ├── DS28E25 (0x47) [legacy]    │
│      ├── GX2431 (0x2D) [new]        │
│      └── GXE00 (0x14) [new]         │
└─────────────────────────────────────┘
```

## 2.2 核心数据结构

### w1_reg_num — 64位ROM标识

```c
struct w1_reg_num {
#if defined(__LITTLE_ENDIAN_BITFIELD)
    __u64   family:8,     // 第0字节：家族码 (0x14, 0x2D, 0x47...)
            id:48,        // 第1-6字节：48位唯一序列号
            crc:8;        // 第7字节：CRC校验
#elif defined(__BIG_ENDIAN_BITFIELD)
    __u64   crc:8,
            id:48,
            family:8;
#endif
};
```

所以一个GXE00芯片的ROM ID看起来像：

```
0x14 0xA1 0xB2 0xC3 0xD4 0xE5 0xF6 0xCRC
  ↑                                  ↑
 family=0x14   48-bit unique ID      CRC-8
```

在sysfs中，设备名就是 `{family_code}-{48bit_id_hex}`，例如 `14-a1b2c3d4e5f6`。

### w1_slave — 从设备表示

```c
struct w1_slave {
    struct module       *owner;
    unsigned char       name[W1_MAXNAMELEN];   // 设备名
    struct w1_reg_num   reg_num;               // 64位ROM ID
    u8                  rom[9];                // 原始ROM字节
    u32                 flags;
    int                 ttl;                   // 生存时间（用于老化检测）

    struct w1_master    *master;               // 所属总线主控
    struct w1_family    *family;               // 匹配的家族驱动
    void                *family_data;          // ← 你的私有数据指针！
    struct device       dev;                   // 内核设备结构
};
```

**关键理解 `family_data`**：这是框架留给驱动作者存放私有数据的地方。对于简单驱动可能不需要（DS2431就没用），但对于需要维护状态的复杂驱动（如DS28E25的SHA密钥），就在这里 `kmalloc` 一个结构体并挂载。

### w1_family — 家族驱动注册

```c
struct w1_family_ops {
    int  (* add_slave)(struct w1_slave *);     // 发现设备时调用
    void (* remove_slave)(struct w1_slave *);  // 设备移除时调用
};

struct w1_family {
    struct list_head    family_entry;  // 链表节点（内部使用）
    u8                  fid;          // 家族码：0x14, 0x2D, 0x47...
    struct w1_family_ops *fops;       // 操作函数表
    atomic_t            refcnt;       // 引用计数
};
```

## 2.3 W1总线的设备发现流程

```
系统启动
  │
  ▼
DS2484 I2C驱动probe()
  │  → i2c_board_info匹配 ("ds2484", 0x18)
  │  → 初始化DS2484芯片
  ▼
w1_add_master_device(&ds2484_master)
  │  → 创建struct w1_master
  │  → 启动工作队列，开始总线扫描
  ▼
w1_search()
  │  → 发送0xF0 (Search ROM)命令
  │  → 使用"三角搜索算法"枚举所有设备
  ▼
对于每个找到的ROM ID:
  │  (例如 0x14A1B2C3D4E5F6CRC)
  │
  ▼
w1_attach_slave_device()
  │  → 提取family = 0x14
  │  → 查找注册的 w1_family.fid == 0x14
  ▼
匹配成功!
  → gxe00_add_slave(sl)被调用
  → 创建sysfs属性文件 eeprom
  → 用户可以通过 cat eeprom 读取
```

## 2.4 1-Wire基本I/O操作

```c
// 复位总线 + 选择特定从设备
int w1_reset_select_slave(struct w1_slave *sl);
// 内部流程：
//   1. 发送复位脉冲 (w1_reset_bus)
//   2. 发送0x55 (Match ROM)
//   3. 发送sl->rom[8]的64位ROM ID
//   结果：总线上只有该设备响应后续命令

// 复位总线（不选择设备）
int w1_reset_bus(struct w1_master *master);

// 跳过ROM选择（只有总线上只有一个设备时可用）
int w1_reset_bus();
w1_write_8(master, 0xCC);   // Skip ROM
// 然后直接发送设备命令

// 读写单个字节
void w1_write_8(struct w1_master *, u8 byte);
u8   w1_read_8(struct w1_master *);

// 读写多字节
void w1_write_block(struct w1_master *, const u8 *buf, int len);
void w1_read_block(struct w1_master *, u8 *buf, int len);
```

## 2.5 互斥保护 — 总线共享

**所有1-Wire设备共享同一条总线！** 这意味着：

```c
// ✅ 正确：在操作前后加锁
mutex_lock(&sl->master->mutex);
// ... 一系列W1操作 ...
mutex_unlock(&sl->master->mutex);

// ❌ 错误：不加锁可能与其他设备操作交错
// 如果另一个进程同时操作DS28E25，两条命令会混在一起！
```

> 💡 在DS2431和GXE00的驱动中，read/write函数在加锁后执行所有的W1操作。这是因为一个"读"实际上涉及2次复位+选择+数据传输，如果不加锁，总线可能中途被其他设备占用。

---

# 模块三：简单EEPROM驱动精读 — w1_ds2431.c

## 3.0 准备工作

在阅读前，请打开 `source/linux-3.2.0-psp04.06.00.11/drivers/w1/slaves/w1_ds2431.c` 对照阅读。

## 3.1 芯片知识：DS2431 / GX2431

| 参数 | 值 |
|------|-----|
| 家族码 | 0x2D |
| EEPROM大小 | 1024 bits = 128 bytes |
| 页数 | 4页，每页32字节 |
| 页写缓冲区 | 8字节scratchpad |
| 命令集 | Read (0xF0), Write Scratch (0x0F), Read Scratch (0xAA), Copy Scratch (0x55) |
| 编程时间 | 11ms |

## 3.2 常量定义逐行分析

```c
#define W1_F2D_EEPROM_SIZE      128    // 总共128字节
#define W1_F2D_PAGE_COUNT       4      // 4个页
#define W1_F2D_PAGE_BITS        5      // 每页2^5=32字节
#define W1_F2D_PAGE_SIZE        (1<<W1_F2D_PAGE_BITS)  // 32字节
#define W1_F2D_PAGE_MASK        0x1F   // 页内偏移掩码

#define W1_F2D_SCRATCH_BITS     3      // 2^3=8字节scratchpad
#define W1_F2D_SCRATCH_SIZE     (1<<W1_F2D_SCRATCH_BITS) // 8字节
#define W1_F2D_SCRATCH_MASK     0x07   // scratchpad内偏移掩码

#define W1_F2D_READ_EEPROM      0xF0   // 读EEPROM命令
#define W1_F2D_WRITE_SCRATCH    0x0F   // 写scratchpad命令
#define W1_F2D_READ_SCRATCH     0xAA   // 读scratchpad命令
#define W1_F2D_COPY_SCRATCH     0x55   // 复制到EEPROM命令

#define W1_F2D_TPROG_MS         11     // EEPROM编程时间(ms)

#define W1_F2D_READ_RETRIES     10     // 读重试次数
#define W1_F2D_READ_MAXLEN      8      // 单次读取最大字节数
```

**命名惯例**：`F2D` 来自 Family 2D。这是W1驱动社区的习惯命名法。

## 3.3 读操作深度解析

### 通信协议：读EEPROM

```
主机操作                         1-Wire总线
─────────                       ──────────
Reset + Presence                 ───┐────
发送0xF0 (Read Memory)           ████
发送Address_Low                  ████
发送Address_High                 ████
读取Count字节数据                 ████→
(重复第二次做校验)
```

### `w1_f2d_readblock()` 逐行注释

```c
static int w1_f2d_readblock(struct w1_slave *sl, int off, int count, char *buf)
{
    u8 wrbuf[3];                         // 命令缓冲区：cmd + addrL + addrH
    u8 cmp[W1_F2D_READ_MAXLEN];         // 比对缓冲区
    int tries = W1_F2D_READ_RETRIES;    // 重试计数器

    do {
        // 构建命令包
        wrbuf[0] = W1_F2D_READ_EEPROM;  // 命令字节
        wrbuf[1] = off & 0xff;          // 地址低字节
        wrbuf[2] = off >> 8;            // 地址高字节

        // 第1次读
        if (w1_reset_select_slave(sl))  // 复位 + ROM匹配
            return -1;
        w1_write_block(sl->master, wrbuf, 3);  // 发送命令
        w1_read_block(sl->master, buf, count); // 读取数据

        // 第2次读（proof read）
        if (w1_reset_select_slave(sl))
            return -1;
        w1_write_block(sl->master, wrbuf, 3);
        w1_read_block(sl->master, cmp, count);

        // 比较两次结果
        if (!memcmp(cmp, buf, count))
            return 0;   // 一致 → 成功

    } while (--tries);  // 不一致 → 重试（最多10次）

    dev_err(&sl->dev, "proof reading failed %d times\n",
            W1_F2D_READ_RETRIES);
    return -1;
}
```

**为什么需要proof read？** 1-Wire是单线协议，没有独立的时钟线。噪声可能使主机或设备采样错误。通过两次独立读取并比较，可以大大降低误码率。

### `w1_f2d_read_bin()` — sysfs读入口

```c
static ssize_t w1_f2d_read_bin(struct file *filp, struct kobject *kobj,
                               struct bin_attribute *bin_attr,
                               char *buf, loff_t off, size_t count)
{
    struct w1_slave *sl = kobj_to_w1_slave(kobj);
    int todo = count;

    // 1. 边界修剪
    count = w1_f2d_fix_count(off, count, W1_F2D_EEPROM_SIZE);
    if (count == 0)
        return 0;

    // 2. 锁总线（防止与其他操作交错）
    mutex_lock(&sl->master->mutex);

    // 3. 分块读取（每块最多8字节）
    while (todo > 0) {
        int block_read;
        if (todo >= W1_F2D_READ_MAXLEN)
            block_read = W1_F2D_READ_MAXLEN;
        else
            block_read = todo;

        if (w1_f2d_readblock(sl, off, block_read, buf) < 0)
            count = -EIO;   // 记录错误，但继续读剩余部分

        todo -= W1_F2D_READ_MAXLEN;
        buf += W1_F2D_READ_MAXLEN;
        off += W1_F2D_READ_MAXLEN;
    }

    mutex_unlock(&sl->master->mutex);
    return count;
}
```

## 3.4 写操作深度解析

### 通信协议：写EEPROM

```
步骤1: 写Scratchpad
  Reset + Select → 0x0F → AddrL → AddrH → Data[8]

步骤2: 读Scratchpad并验证
  Reset + Select → 0xAA → 读回TA1, TA2, ES, Data[8]
  → 验证TA1, TA2, ES, Data是否与写入的一致

步骤3: 复制到EEPROM
  Reset + Select → 0x55 → AddrL → AddrH → ES
  → 等待10ms (tprog)
  → Reset总线唤醒设备
```

### Scratchpad验证的细节

```c
// 写入后读取scratchpad
w1_write_8(sl->master, W1_F2D_READ_SCRATCH);
w1_read_block(sl->master, rdbuf, len + 3);

// rdbuf[0] = TA1 (目标地址低字节) — 应与wrbuf[1]相同
// rdbuf[1] = TA2 (目标地址高字节) — 应与wrbuf[2]相同
// rdbuf[2] = E/S (结束偏移 + 状态)
//   - 低5位 = 最后写入字节在scratchpad中的偏移
//   - 应等于 ((addr + len - 1) % 8)
// rdbuf[3..3+len-1] = scratchpad中的数据

// 验证：
if ((rdbuf[0] != wrbuf[1]) ||         // TA1匹配？
    (rdbuf[1] != wrbuf[2]) ||         // TA2匹配？
    (rdbuf[2] != es) ||               // E/S匹配？
    (memcmp(data, &rdbuf[3], len)))   // 数据匹配？
{
    // 验证失败，重试
}
```

### `w1_f2d_write_bin()` — 处理非对齐写入

这是一个重要模式：**read-modify-write（读-改-写）**。

```c
if (len < W1_F2D_SCRATCH_SIZE || addr & W1_F2D_SCRATCH_MASK) {
    // 情况A：写入未对齐 → 需要读-改-写
    // 1. 先读回完整的scratchpad块
    // 2. 在内存中修改需要更新的部分
    // 3. 写回完整块
    char tmp[W1_F2D_SCRATCH_SIZE];
    w1_f2d_readblock(sl, addr & ~0x07, 8, tmp);  // 读回完整8字节
    memcpy(&tmp[addr & 0x07], buf, copy);         // 修改局部
    w1_f2d_write(sl, addr & ~0x07, 8, tmp);       // 写回
} else {
    // 情况B：对齐写入 → 直接写
    w1_f2d_write(sl, addr, 8, buf);
}
```

**为什么需要读-改-写？** 因为EEPROM以8字节(scratchpad大小)为编程单位。你不能只写3个字节到中间，必须凑满8字节。不在这8字节范围内的部分，先从EEPROM读出来保持原样。

## 3.5 sysfs属性文件注册

```c
static struct bin_attribute w1_f2d_bin_attr = {
    .attr = {
        .name = "eeprom",                    // 文件名
        .mode = S_IRUGO | S_IWUSR,          // 644权限
    },
    .size = W1_F2D_EEPROM_SIZE,             // 128字节
    .read = w1_f2d_read_bin,
    .write = w1_f2d_write_bin,
};

static int w1_f2d_add_slave(struct w1_slave *sl)
{
    return sysfs_create_bin_file(&sl->dev.kobj, &w1_f2d_bin_attr);
}

static void w1_f2d_remove_slave(struct w1_slave *sl)
{
    sysfs_remove_bin_file(&sl->dev.kobj, &w1_f2d_bin_attr);
}
```

## 3.6 家族注册总览

```c
static struct w1_family_ops w1_f2d_fops = {
    .add_slave    = w1_f2d_add_slave,
    .remove_slave = w1_f2d_remove_slave,
};

static struct w1_family w1_family_2d = {
    .fid = W1_EEPROM_DS2431,    // = 0x2D
    .fops = &w1_f2d_fops,
};

static int __init w1_f2d_init(void)
{
    return w1_register_family(&w1_family_2d);
}

module_init(w1_f2d_init);
```

---

# 模块四：从零编写GXE00驱动

## 4.1 GXE00芯片资料

| 参数 | 值 |
|------|-----|
| 制造商 | GXCAS (中科银河芯) |
| 家族码 | **0x14** |
| EEPROM大小 | 256 bits = **32字节** |
| 页数 | **1页** |
| Scratchpad | **8字节** |
| 命令集 | 与DS2431相同 (0xF0, 0x0F, 0xAA, 0x55) |
| 编程时间 | **10ms** |
| 工作温度 | -40°C ~ +85°C |

## 4.2 完整的GXE00驱动

```c
/*
 * w1_gxe00.c - w1 family 0x14 (GXCAS GXE00) driver
 *
 * GXE00: 256-bit 1-Wire EEPROM
 *   - Family code: 0x14
 *   - 1 page of 32 bytes
 *   - Compatible with DS2431 command set (subset)
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/device.h>
#include <linux/types.h>
#include <linux/delay.h>

#include "../w1.h"
#include "../w1_int.h"
#include "../w1_family.h"

// ============================================================
// 常量定义
// ============================================================

#define GXE00_EEPROM_SIZE       32    // 256 bits = 32 bytes
#define GXE00_PAGE_COUNT        1     // 1 page
#define GXE00_PAGE_SIZE         32    // 整片就是32字节

#define GXE00_READ_EEPROM       0xF0  // 读EEPROM
#define GXE00_WRITE_SCRATCH     0x0F  // 写scratchpad
#define GXE00_READ_SCRATCH      0xAA  // 读scratchpad
#define GXE00_COPY_SCRATCH      0x55  // 复制到EEPROM

#define GXE00_TPROG_MS          10    // 编程时间10ms

#define GXE00_READ_RETRIES      10    // 读重试次数
#define GXE00_READ_MAXLEN       8     // 单次最大读取长度

// ============================================================
// 辅助函数：边界裁剪
// ============================================================

static inline size_t gxe00_fix_count(loff_t off, size_t count, size_t size)
{
    if (off > size)
        return 0;
    if ((off + count) > size)
        return size - off;
    return count;
}

// ============================================================
// 读操作
// ============================================================

static int gxe00_readblock(struct w1_slave *sl, int off, int count, char *buf)
{
    u8 wrbuf[3];
    u8 cmp[GXE00_READ_MAXLEN];
    int tries = GXE00_READ_RETRIES;

    do {
        wrbuf[0] = GXE00_READ_EEPROM;
        wrbuf[1] = off & 0xff;
        wrbuf[2] = off >> 8;

        if (w1_reset_select_slave(sl))
            return -1;

        w1_write_block(sl->master, wrbuf, 3);
        w1_read_block(sl->master, buf, count);

        if (w1_reset_select_slave(sl))
            return -1;

        w1_write_block(sl->master, wrbuf, 3);
        w1_read_block(sl->master, cmp, count);

        if (!memcmp(cmp, buf, count))
            return 0;
    } while (--tries);

    dev_err(&sl->dev, "proof reading failed %d times\n",
            GXE00_READ_RETRIES);
    return -1;
}

static ssize_t gxe00_read_bin(struct file *filp, struct kobject *kobj,
                               struct bin_attribute *bin_attr,
                               char *buf, loff_t off, size_t count)
{
    struct w1_slave *sl = kobj_to_w1_slave(kobj);
    int todo = count;

    count = gxe00_fix_count(off, count, GXE00_EEPROM_SIZE);
    if (count == 0)
        return 0;

    mutex_lock(&sl->master->mutex);

    while (todo > 0) {
        int block_read;

        if (todo >= GXE00_READ_MAXLEN)
            block_read = GXE00_READ_MAXLEN;
        else
            block_read = todo;

        if (gxe00_readblock(sl, off, block_read, buf) < 0)
            count = -EIO;

        todo -= GXE00_READ_MAXLEN;
        buf += GXE00_READ_MAXLEN;
        off += GXE00_READ_MAXLEN;
    }

    mutex_unlock(&sl->master->mutex);

    return count;
}

// ============================================================
// 写操作
// ============================================================

static int gxe00_write(struct w1_slave *sl, int addr, int len, const u8 *data)
{
    int tries = GXE00_READ_RETRIES;
    u8 wrbuf[4];
    u8 rdbuf[8 + 3];
    u8 es = (addr + len - 1) % 8;

retry:
    // 步骤1: 写scratchpad
    if (w1_reset_select_slave(sl))
        return -1;

    wrbuf[0] = GXE00_WRITE_SCRATCH;
    wrbuf[1] = addr & 0xff;
    wrbuf[2] = addr >> 8;

    w1_write_block(sl->master, wrbuf, 3);
    w1_write_block(sl->master, data, len);

    // 步骤2: 读scratchpad验证
    if (w1_reset_select_slave(sl))
        return -1;

    w1_write_8(sl->master, GXE00_READ_SCRATCH);
    w1_read_block(sl->master, rdbuf, len + 3);

    if ((rdbuf[0] != wrbuf[1]) || (rdbuf[1] != wrbuf[2]) ||
        (rdbuf[2] != es) || (memcmp(data, &rdbuf[3], len) != 0)) {

        if (--tries)
            goto retry;

        dev_err(&sl->dev,
                "scratchpad compare failed %d times\n",
                GXE00_READ_RETRIES);
        return -1;
    }

    // 步骤3: 复制到EEPROM
    if (w1_reset_select_slave(sl))
        return -1;

    wrbuf[0] = GXE00_COPY_SCRATCH;
    wrbuf[3] = es;
    w1_write_block(sl->master, wrbuf, 4);

    msleep(GXE00_TPROG_MS);
    w1_reset_bus(sl->master);

    return 0;
}

static ssize_t gxe00_write_bin(struct file *filp, struct kobject *kobj,
                                struct bin_attribute *bin_attr,
                                char *buf, loff_t off, size_t count)
{
    struct w1_slave *sl = kobj_to_w1_slave(kobj);
    int addr, len, copy;

    count = gxe00_fix_count(off, count, GXE00_EEPROM_SIZE);
    if (count == 0)
        return 0;

    mutex_lock(&sl->master->mutex);

    addr = off;
    len = count;
    while (len > 0) {
        if (len < 8 || addr & 0x07) {
            // 未对齐：读-改-写
            char tmp[8];

            if (gxe00_readblock(sl, addr & ~0x07, 8, tmp)) {
                count = -EIO;
                goto out_up;
            }

            copy = 8 - (addr & 0x07);
            if (copy > len)
                copy = len;

            memcpy(&tmp[addr & 0x07], buf, copy);
            if (gxe00_write(sl, addr & ~0x07, 8, tmp) < 0) {
                count = -EIO;
                goto out_up;
            }
        } else {
            // 对齐：直接写
            copy = 8;
            if (gxe00_write(sl, addr, copy, buf) < 0) {
                count = -EIO;
                goto out_up;
            }
        }
        buf += copy;
        addr += copy;
        len -= copy;
    }

out_up:
    mutex_unlock(&sl->master->mutex);
    return count;
}

// ============================================================
// sysfs属性文件
// ============================================================

static struct bin_attribute gxe00_bin_attr = {
    .attr = {
        .name = "eeprom",
        .mode = S_IRUGO | S_IWUSR,
    },
    .size = GXE00_EEPROM_SIZE,
    .read = gxe00_read_bin,
    .write = gxe00_write_bin,
};

// ============================================================
// 家族驱动接口
// ============================================================

static int gxe00_add_slave(struct w1_slave *sl)
{
    return sysfs_create_bin_file(&sl->dev.kobj, &gxe00_bin_attr);
}

static void gxe00_remove_slave(struct w1_slave *sl)
{
    sysfs_remove_bin_file(&sl->dev.kobj, &gxe00_bin_attr);
}

static struct w1_family_ops gxe00_fops = {
    .add_slave    = gxe00_add_slave,
    .remove_slave = gxe00_remove_slave,
};

static struct w1_family gxe00_family = {
    .fid = 0x14,
    .fops = &gxe00_fops,
};

// ============================================================
// 模块入口/出口
// ============================================================

static int __init gxe00_init(void)
{
    pr_info("w1_gxe00: registering family 0x14 (GXE00 256-bit EEPROM)\n");
    return w1_register_family(&gxe00_family);
}

static void __exit gxe00_fini(void)
{
    w1_unregister_family(&gxe00_family);
}

module_init(gxe00_init);
module_exit(gxe00_fini);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Puhui/Advantech");
MODULE_DESCRIPTION("w1 family 0x14 driver for GXE00, 256-bit 1-Wire EEPROM");
```

## 4.3 差异分析：GXE00 vs DS2431

| 特性 | DS2431 (128B) | GXE00 (32B) | 原因 |
|------|--------------|-------------|------|
| EEPROM_SIZE | 128 | 32 | GXE00只有256位 |
| PAGE_COUNT | 4 | 1 | GXE00整片1页 |
| 命令集 | 相同 | 相同 | 兼容设计 |
| 编程时间 | 11ms | 10ms | 芯片工艺差异 |
| 读-改-写块大小 | 8字节 | 8字节 | 都是8字节scratchpad |

## 4.4 构建系统集成

### Kconfig 添加 (`drivers/w1/slaves/Kconfig`)

```kconfig
config W1_SLAVE_GXE00
    tristate "256-bit EEPROM family support (GXE00)"
    depends on W1
    help
      Say Y here if you want to use a 1-wire
      256-bit EEPROM family device (GXE00).
```

### Makefile 添加 (`drivers/w1/slaves/Makefile`)

```makefile
obj-$(CONFIG_W1_SLAVE_GXE00)    += w1_gxe00.o
```

### defconfig 添加 (`arch/arm/configs/am335x_rom3310_puhui_defconfig`)

```kconfig
CONFIG_W1_SLAVE_GXE00=y
CONFIG_W1_SLAVE_DS2431=y        # GX2431兼容DS2431
```

---

# 模块五：集成与构建

## 5.1 完整构建流程

```bash
# 1. 进入脚本目录
cd /path/to/AM335XLBV1120/scripts/

# 2. 设置环境变量
source setenv.sh
# 这会设置：CROSS_COMPILE, ARCH, KROOT, UBOOT_SOURCE, ROOTFS, LOG

# 3. 配置内核
./cfg_kernel.sh am335x_rom3310_puhui_defconfig

# 4. (可选) 用menuconfig检查配置
make -C $KROOT menuconfig
# 进入 Device Drivers → 1-Wire → 1-wire Slaves
# 确认 GXE00 和 DS2431 都已勾选

# 5. 构建内核
./mk_kernel.sh
# 输出：$KROOT/arch/arm/boot/uImage → copy到 image/

# 6. 烧写到SD卡启动
```

## 5.2 验证方法

启动新内核后：

```bash
# 查看所有1-Wire设备
ls /sys/bus/w1/devices/

# 期望输出示例：
# 14-000012345678  (GXE00)
# 2d-0000abcdef01  (GX2431)
# 47-0000deadbeef  (DS28E25, legacy)

# 读取GXE00内容
cat /sys/bus/w1/devices/14-*/eeprom | hexdump -C

# 写入GXE00
echo -ne '\x01\x02\x03\x04' > /sys/bus/w1/devices/14-*/eeprom

# 验证写入
cat /sys/bus/w1/devices/14-*/eeprom | hexdump -C
# 前4字节应为 01 02 03 04

# 检查内核日志
dmesg | tail -20
# 应看到 "w1_gxe00: registering family 0x14 (GXE00 256-bit EEPROM)"
```

---

# 模块六：安全认证型驱动 — DS28E25架构分析

> ⚠️ 此模块为进阶内容。在完成GX2431和GXE00驱动后，理解DS28E25可以帮助你处理未来更复杂的设备。

## 6.1 DS28E25 vs 普通EEPROM

| 特性 | DS2431/GX2431 (普通) | DS28E25 (安全型) |
|------|---------------------|-------------------|
| EEPROM大小 | 128字节 | 512字节 |
| 家族码 | 0x2D | 0x47 |
| 写操作 | 直接写入 | 需要MAC认证 |
| 读操作 | 直接读取 | 普通读或认证读 |
| 保护机制 | 页写保护 | SHA-256挑战-响应 |
| 密钥管理 | 无 | 32字节安全密钥 |
| 编程接口 | sysfs eeprom | /dev/ds28e25 + ioctl |
| 驱动复杂度 | ~310行 | ~2300行 |

## 6.2 DS28E25驱动架构

```
w1_ds28e25.c
├── 芯片命令定义 (~30个命令)
├── 写操作函数
│   ├── w1_ds28e25_write_segment()    ← 写4字节段
│   ├── w1_ds28e25_write_memory()     ← 写多段
│   ├── w1_ds28e25_write_scratchpad() ← 写32字节scratchpad
│   └── w1_ds28e25_write_authblock()  ← 带MAC认证的写
├── 读操作函数
│   ├── w1_ds28e25_read_memory()      ← 读内存
│   ├── w1_ds28e25_read_page()        ← 读整页(32字节)
│   └── w1_ds28e25_read_scratchpad()  ← 读scratchpad
├── 安全函数
│   ├── w1_ds28e25_load_secret()      ← 加载密钥
│   ├── w1_ds28e25_compute_secret()   ← 计算并锁定密钥
│   ├── w1_ds28e25_compute_read_pageMAC() ← 计算页面MAC
│   ├── w1_ds28e25_verifymac()        ← 完整验证流程
│   └── calculate_write_authMAC256()  ← 软件计算写入认证MAC
├── 状态函数
│   ├── w1_ds28e25_read_status()      ← 读状态寄存器
│   ├── w1_ds28e25_write_blockprotection() ← 写保护
│   └── w1_ds28e25_write_authblockprotection() ← 认证写保护
├── ioctl接口
│   └── 20+命令，通过 /dev/ds28e25 访问
├── sysfs接口
│   ├── read_eeprom   ← 读取全部16页
│   └── verify_mac    ← 触发认证验证
└── SHA-256软件实现
    └── w1_ds28e25_sha256.c/h
```

## 6.3 认证写流程

```
主机                                 DS28E25
────                                 ───────
1. 发送随机挑战(nonce)
2. 发送要写入的数据
                                    ──→ 计算 MAC = SHA256(数据 || 密钥 || 随机数)
3. 读取计算的MAC                     ←──
4. 主机用软件SHA256计算期望的MAC
5. 比较 → 如果一致，发送授权写命令
6. 设备写入EEPROM
```

---

# 模块七：独立编程练习

以下练习**不会影响当前代码库** —— 你可能需要创建临时文件或仅用纸笔完成。

---

## 📝 练习1：协议跟踪

**目标**：通过纸笔追踪了解1-Wire协议的数据流。

**任务**：给定一个GXE00芯片的ROM ID为 `0x14 0x11 0x22 0x33 0x44 0x55 0x66 0xCRC`，写出以下操作的完整1-Wire总线事务序列：

1. 从偏移量0读取8字节
2. 从偏移量8读取4字节
3. 写入4字节 `0xAA 0xBB 0xCC 0xDD` 到偏移量4

**提示格式**：
```
读操作序列:
  Reset + Presence
  → Match ROM: 0x55
  → ROM[8]: 0x14 0x11 ... 0xCRC
  → Read EEPROM: 0xF0
  → AddrL: 0x00
  → AddrH: 0x00
  → Read 8 bytes: <-- [0x?? 0x?? ...]
  Reset + Presence
  → Match ROM: 0x55
  → ROM[8]: ...
  → Read EEPROM: 0xF0
  → AddrL: 0x00
  → AddrH: 0x00
  → Read 8 bytes: <-- [0x?? 0x?? ...]
  (两次比较，如果一致则返回)
```

---

## 📝 练习2：代码填空

**目标**：在理解的基础上补全代码片段。

**任务**：下面是一个不完整的GXE00 sysfs读取函数，请在 `/* FILL */` 处补全正确的代码：

```c
static ssize_t gxe00_read_bin(
    struct file *filp, struct kobject *kobj,
    struct bin_attribute *bin_attr,
    char *buf, loff_t off, size_t count)
{
    struct w1_slave *sl = kobj_to_w1_slave(kobj);
    int todo = count;

    count = gxe00_fix_count(off, count, 32);
    if (count == 0)
        return 0;

    /* FILL: 锁总线 */
    // _____________________________________

    while (todo > 0) {
        int block_read = min(todo, 8);

        /* FILL: 调用readblock */
        // if (________________________________)  // (1)
        //     count = -EIO;

        /* FILL: 更新todo, buf指针, off */
        // todo -= ________________________________;  // (2)
        // buf  += ________________________________;  // (3)
        // off  += ________________________________;  // (4)
    }

    /* FILL: 解锁总线 */
    // _____________________________________

    return count;
}
```

**参考答案**在本文档末尾。

---

## 📝 练习3：添加新的sysfs属性

**目标**：在不影响现有代码的前提下，为GXE00驱动添加一个新功能。

**场景**：你的产品经理要求将GXE00的32字节分成两段暴露给用户：
- `eeprom`：只读，显示全部32字节
- `user`：读/写，仅前16字节

你需要：
1. 如何修改 `gxe00_read_bin()` 使其只读前16字节？—— 提示：改什么宏？
2. 如何添加一个新文件 `user` 来读写后16字节？
3. 为什么在这个方案中，写 `user` 的前16字节会影响 `eeprom` 的内容？

**思考**：这不是一个嵌套ioctl的方案，而是一个简单的sysfs多文件方案。你会如何设计？

---

## 📝 练习4：读-改-写流程跟踪

**目标**：深入理解写操作的读-改-写逻辑。

**任务**：假设GXE00的EEPROM当前内容为全0，用户执行了以下命令：

```bash
echo -ne '\xFF\xEE' > /sys/bus/w1/devices/14-*/eeprom
```

这会产生 `off=0, count=2` 的写请求。请按步骤描述 `gxe00_write_bin()` 内部发生了什么：

1. `addr` 和 `len` 的初始值是多少？
2. 第一次循环判断 `len < 8 || addr & 0x07` 的结果是什么？
3. `gxe00_readblock` 读回的 `tmp` 内容是什么？（假设EEPROM全是0）
4. `memcpy` 后的 `tmp` 内容是什么？
5. `gxe00_write` 被调用时传入的addr和len是什么？
6. 写完成后，EEPROM的0-7字节变成什么？

---

## 📝 练习5：驱动模块化改造

**目标**：理解模块化驱动的设计模式。

**任务**：假设你要把GXE00驱动改成一个可加载模块（`=m`），而不是built-in（`=y`）。你需要修改哪些地方？

思考以下问题：
1. Kconfig中的 `tristate` 和 `bool` 的区别是什么？
2. 如果GXE00是模块，而DS28E25是built-in，启动顺序会有什么问题？
3. 模块化驱动的初始化函数能否使用 `__init` 标记？
4. 驱动作为模块时，`module_exit()` 会在什么情况下被调用？

---

## 📝 练习6：错误注入调试

**目标**：理解驱动的错误处理路径。

**任务**：假设GXE00硬件存在以下故障，驱动的行为会是什么？

| 故障场景 | read_bin返回值 | write_bin返回值 | 内核日志 |
|----------|---------------|----------------|---------|
| 芯片从总线上移除 | ? | ? | ? |
| EEPROM写超时（20ms后仍忙） | — | ? | ? |
| 读数据时CRC校验（两次不一致）反复失败 | ? | — | ? |
| 写scratchpad后验证失败3次后成功 | — | ? | ? |

**注**：GXE00及DS2431没有硬件CRC校验，这里的"校验"指software proof reading。

---

## 📝 练习7（进阶）：实现IOCTL接口

**目标**：理解字符设备驱动与ioctl接口。

**任务**：阅读DS28E25驱动中关于miscdevice和ioctl的代码，然后设计一个简化版的GXE00 ioctl接口。要求仅实现两个命令：

```c
#define GXE00_IOCTL_MAGIC  0xE1
#define GXE00_IOCTL_READ   _IOR(GXE00_IOCTL_MAGIC, 1, struct gxe00_ioctl_data)
#define GXE00_IOCTL_WRITE  _IOW(GXE00_IOCTL_MAGIC, 2, struct gxe00_ioctl_data)

struct gxe00_ioctl_data {
    off_t   offset;     // 偏移
    size_t  length;     // 长度
    u8      *buffer;    // 数据缓冲区（用户空间指针）
};
```

你需要：
1. 写出miscdevice的注册代码
2. 写出unlocked_ioctl处理函数框架（注意copy_from_user/copy_to_user）
3. 说明为什么用户态用ioctl访问EEPROM比sysfs读写更高效（提示：减少上下文切换次数）

---

## 📝 练习8（思考题）：固件升级兼容性

**目标**：理解向后兼容的设计思路。

**场景**：你的公司在ROM-3310产品上已经部署了1000台设备，使用DS28E25作为认证EEPROM。现在要升级到GXE00 + GX2431的组合。你的驱动程序需要自动检测总线上有哪些设备并正确驱动它们。

问题：
1. W1子系统是如何自动完成这个任务的？为什么不需修改ds2484.c？
2. 如果同一个家族码（比如0x2D）有不同厂商的兼容芯片，驱动程序需要修改吗？
3. 如果GXE00的写时序从10ms增加到12ms（批号差异），你该修改哪个宏？对现有设备有影响吗？
4. 如果某台设备同时焊接了DS28E25和GXE00，总线上会出现什么情况？sysfs中能看到几个设备？

---

## 📝 练习9（代码审查）：你能发现这个驱动的bug吗？

**目标**：训练代码审查能力。

**任务**：以下是一个新手写的GXE00驱动片段，请找出所有问题：

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include "../w1.h"
#include "../w1_family.h"

#define SIZE 32

static ssize_t my_read(struct file *f, struct kobject *k,
        struct bin_attribute *a, char *buf, loff_t off, size_t cnt)
{
    struct w1_slave *sl = kobj_to_w1_slave(k);

    w1_reset_select_slave(sl);
    w1_write_8(sl->master, 0xF0);
    w1_write_8(sl->master, off & 0xFF);
    w1_write_8(sl->master, off >> 8);
    w1_read_block(sl->master, buf, cnt);

    return cnt;
}

static struct bin_attribute my_attr = {
    .attr = { .name = "data", .mode = 0666 },
    .read = my_read,
    .write = NULL,
};

static int my_add_slave(struct w1_slave *sl)
{
    return sysfs_create_bin_file(&sl->dev.kobj, &my_attr);
}

static struct w1_family my_fam = {
    .fid = W1_FAMILY_DS28E25,
    .fops = &(struct w1_family_ops){
        .add_slave = my_add_slave,
    },
};

module_init(my_init);
module_exit(my_fini);
```

**至少找出8个问题**（包括但不限于：缺少proof read、缺少互斥锁、缺少边界检查、错误回复用、权限错误、家族码错误、缺少模块元数据、缺少模块入口/出口函数声明等）。

---

## 📝 练习10（设计）：双芯片驱动共存方案

**目标**：设计一个真实世界的驱动共存方案。

**背景**：你的系统需要同时支持三个1-Wire EEPROM芯片：
1. DS28E25 (0x47) — 现有，安全认证
2. GX2431 (0x2D) — 新，128字节普通EEPROM
3. GXE00 (0x14) — 新，32字节普通EEPROM

**任务A**：画出W1子系统总线上三个设备的注册流程图，从 `w1_add_master_device()` 到三个 `add_slave()` 被调用。

**任务B**：假设你希望用户空间能通过统一路径 `/sys/bus/w1/devices/eeprom_all` 读取所有3个芯片的全部内容（拼接起来），但前提是不修改任何已存在的驱动文件。你该怎么做？
- 提示：在用户空间用一个脚本可以实现吗？
- 另一种思路：写一个"聚合驱动"，它监听三个家族码，然后在一个新的sysfs文件中聚合数据。这需要三个 `w1_register_family()` 调用，但只有一个 `add_slave` 实现——可以吗？（提示：看 `struct w1_family_ops` 是否可以复用？）

---

# 参考答案

## 练习2 参考答案

```c
/* FILL: 锁总线 */
mutex_lock(&sl->master->mutex);

/* FILL: 调用readblock */
if (gxe00_readblock(sl, off, block_read, buf) < 0)  // (1)
    count = -EIO;

/* FILL: 更新todo, buf指针, off */
todo -= block_read;         // (2)
buf  += block_read;         // (3)
off  += block_read;         // (4)

/* FILL: 解锁总线 */
mutex_unlock(&sl->master->mutex);
```

---

# 附录A：常见内核API速查

| API | 头文件 | 功能 |
|-----|--------|------|
| `kmalloc(size, GFP_KERNEL)` | `<linux/slab.h>` | 分配内核内存 |
| `kfree(ptr)` | `<linux/slab.h>` | 释放内核内存 |
| `memcpy(dst, src, n)` | `<linux/string.h>` | 内存拷贝 |
| `memset(ptr, c, n)` | `<linux/string.h>` | 内存设置 |
| `msleep(ms)` | `<linux/delay.h>` | 毫秒级休眠 |
| `pr_info(fmt, ...)` | `<linux/printk.h>` | 内核日志(info) |
| `dev_info(dev, fmt, ...)` | `<linux/device.h>` | 设备日志 |
| `mutex_lock(m)` | `<linux/mutex.h>` | 获取互斥锁 |
| `mutex_unlock(m)` | `<linux/mutex.h>` | 释放互斥锁 |

# 附录B：调试技巧

## 使用dev_dbg动态调试

```c
// 在驱动中使用
dev_dbg(&sl->dev, "Reading %d bytes from offset %lld\n", count, off);

// 启动后开启调试输出
echo 'file w1_gxe00.c +p' > /sys/kernel/debug/dynamic_debug/control
```

## 使用hexdump调试

```c
#include <linux/printk.h>

// 打印缓冲区内容
print_hex_dump(KERN_INFO, "gxe00: ", DUMP_PREFIX_OFFSET,
               16, 1, data, len, true);
```

## 使用GPIO调试（如果板上有LED或空闲GPIO）

```c
#include <linux/gpio.h>

gpio_request(debug_gpio, "gxe00-debug");
gpio_direction_output(debug_gpio, 0);

// 在关键路径上翻转GPIO → 用示波器观察
gpio_set_value(debug_gpio, 1);
// ... 操作 ...
gpio_set_value(debug_gpio, 0);
```

---

# 附录C：推荐阅读路线

1. **Linux Device Drivers, 3rd Edition** (O'Reilly) — 经典教材，虽然基于2.6内核，但概念通用
2. **Linux内核源码** — `Documentation/w1/` 目录下的文档
3. **内核源码树下的类似驱动** — `drivers/w1/slaves/` 目录下的所有文件
4. **本BSP的CLAUDE.md** — 记录了所有的板级配置和硬件信息

---

*课程结束 | 祝你在驱动开发的道路上越走越远！*s