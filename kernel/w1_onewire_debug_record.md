# AM335X 1-Wire 子系统 DS28E25 和 GX2431 调试全记录

```text
文档版本: 1.0
最后更新: 2026-07-18
适用硬件: AM335X + ROM-3310 + Puhui 载板
适用内核: Linux 3.2.0 (TI PSP 04.06.00.11)
```

---

## 目录

1. [总体架构概述](#1-总体架构概述)
2. [关键数据结构与定义](#2-关键数据结构与定义)
3. [DS28E25 芯片插入—完整代码流程](#3-ds28e25-芯片插入完整代码流程)
4. [GX2431 芯片插入—完整代码流程](#4-gx2431-芯片插入完整代码流程)
5. [核心问题与最终解决方案](#5-核心问题与最终解决方案)
6. [附录：相关源码文件索引](#6-附录相关源码文件索引)

---

## 1. 总体架构概述

### 1.1 硬件连接

```
+-------------+      I2C2 Bus      +-------------+      1-Wire Bus      +-----------+
| AM3358 SoC  | =================> | DS2484 Chip | ==================> | DS28E25   |
| (I2C-2)     |    (addr 0x18)     | I2C-to-1-Wire|        |           | (0x47)    |
|             |                     | Bridge      |        |           |           |
| board-      |                     |             |        |           | GX2431    |
| advantech.c |                     |             |        |           | (0x2D)    |
+-------------+                     +-------------+        |           +-----------+
                                                          Common Handle
```

- **CPU**: AM3358 通过 I2C-2 总线与 DS2484 通信
- **DS2484**: I2C 到 1-Wire 桥接芯片，I2C 地址 0x18
- **DS28E25**: Maxim SHA-256 安全认证芯片，**家族码 0x47**，支持 OverDrive
- **GX2431**: GXCAS 产 EEPROM（DS2431 克隆），**家族码 0x2D**，仅支持标准速度
- 两个芯片通过同一个接插件手柄插入，共用同一条 1-Wire 总线、同一个 DS2484

### 1.2 软件层次结构

```
+---------------------------------------------+
|           应用层 (用户空间)                    |
|   /sys/bus/w1/devices/2d-xxxx/eeprom         |
|   /sys/bus/w1/devices/47-xxxx/sysfs attrs    |
|   /dev/ds28e25 (misc char device)             |
+---------------------------------------------+
|         W1 从设备驱动 (slave drivers)         |
|  +-----------------+ +------------------+    |
|  | w1_ds2431.c     | | w1_ds28e25.c     |    |
|  | (GX2431=0x2D 用) | | (DS28E25=0x47 用)|    |
|  +--------+--------+ +--------+---------+    |
+-----------+-------------------+--------------+
|           |    W1 核心层       |              |
|           |  drivers/w1/w1.c  |              |
|           | 搜索 注册 通信      |              |
|           +--------+----------+              |
+--------------------+-------------------------+
|     W1 主控制器驱动 (master driver)            |
|  +-----------------+                          |
|  | ds2484.c         | [核心改动文件]           |
|  | (I2C->1-Wire桥)  |                         |
|  +-----------------+                          |
+---------------------------------------------+
|            I2C 子系统                          |
+---------------------------------------------+
```

### 1.3 两个芯片的关键差异

| 特性 | DS28E25 (Maxim) | GX2431 (GXCAS) |
|------|:-:|:-:|
| 家族码 (Family Code) | 0x47 | 0x2D |
| 存储容量 | 512 字节 (16页) | 128 字节 (4页) |
| SHA-256 安全认证 | 支持 | 不支持 |
| 标准速度 (15 kbps) | 支持 | 支持 |
| OverDrive 高速 (125 kbps) | **支持** | **不支持** |
| 对应驱动文件 | w1_ds28e25.c | w1_ds2431.c |
| 用户空间接口 | /dev/ds28e25 + sysfs | /sys/bus/w1/.../eeprom |

---

## 2. 关键数据结构与定义

### 2.1 1-Wire 设备地址结构 w1_reg_num

```c
// drivers/w1/w1.h 第25行
struct w1_reg_num {
#if defined(__LITTLE_ENDIAN_BITFIELD)
    __u64 family:8,   // 家族码 (最低8位)
          id:48,      // 唯一序列号 (中间48位)
          crc:8;      // CRC8 校验 (最高8位)
#endif
};
```

**64-bit ROM 布局:**

```
MSB                          LSB
+--------+------------------+--------+
| CRC-8  |  48-bit 序列号   | Family |
| (8bit) |     (48bit)      | (8bit) |
+--------+------------------+--------+
  bit63-56    bit55-8        bit7-0
```

DS28E25 的 ROM 示例: `47-000000557843`
- Family = 0x47 (DS28E25)
- ID = 000000557843
- CRC = 自动计算的 8-bit CRC

GX2431 的 ROM 示例: `2d-000022765b82`
- Family = 0x2D (DS2431/GX2431)
- ID = 000022765b82
- CRC = 自动计算的 8-bit CRC

### 2.2 W1 主控制器结构 w1_bus_master

```c
// drivers/w1/w1.h 第90行
struct w1_bus_master {
    void *data;                     // 私有数据
    u8  (*read_bit)(void *);        // 读一位
    void (*write_bit)(void *, u8);  // 写一位
    u8  (*touch_bit)(void *, u8);   // 触摸位
    u8  (*read_byte)(void *);       // 读字节
    void (*write_byte)(void *, u8); // 写字节
    u8  (*read_block)(void *, u8 *, int);     // 读多字节
    void (*write_block)(void *, const u8 *, int); // 写多字节
    u8  (*triplet)(void *, u8);     // 三重操作 (ROM搜索)
    u8  (*reset_bus)(void *);       // 总线复位 [改动的函数]
    u8  (*set_pullup)(void *, int); // 强上拉
};
```

### 2.3 家族驱动结构 w1_family

```c
// drivers/w1/w1_family.h
struct w1_family {
    struct list_head family_entry;
    u8  fid;                   // 家族码 (0x47 或 0x2D)
    struct w1_family_ops *fops; // add_slave / remove_slave
    atomic_t refcnt;
};

struct w1_family_ops {
    int  (* add_slave)(struct w1_slave *);    // 设备发现时调用
    void (* remove_slave)(struct w1_slave *); // 设备移除时调用
};
```

### 2.4 DS2484 配置寄存器

```c
// drivers/w1/masters/ds2484.c
#define DS2484_REG_CFG_1WS  0x08   // 1-Wire Speed: 1=OverDrive 0=Standard
#define DS2484_REG_CFG_SPU  0x04   // Strong Pull-Up
#define DS2484_REG_CFG_PDN  0x02   // 1-Wire Power Down
#define DS2484_REG_CFG_APU  0x01   // Active Pull-Up
```

**1WS 位是问题的关键:**
- `1WS=1` (0x08) -> OverDrive 高速 (125 kbps) -> DS28E25 能工作
- `1WS=0` (不设置) -> 标准速度 (15 kbps) -> GX2431 能工作
- 原始代码: `0x08 | 0x01 = 0x09`，即 **OverDrive 模式**

## 3. DS28E25 芯片插入—完整代码流程

### 3.1 总体流程

```
插入 DS28E25
    |
    v
[步骤1] DS2484 probe (内核启动时)
    |
    v
[步骤2] w1 内核线程启动 (w1_process)
    |
    v
[步骤3] 复位总线 -> ds2484_w1_reset_bus()
    |        标准速度 PPD=0, 换 OverDrive PPD=1
    v
[步骤4] ROM 搜索 -> w1_search()
    |
    v
[步骤5] 发现设备 -> w1_slave_found()
    |
    v
[步骤6] 注册设备 -> w1_attach_slave_device()
    |        匹配家族码 0x47 -> w1_ds28e25.c
    v
[步骤7] DS28E25 初始化 -> w1_ds28e25_add_slave()
    |
    v
[步骤8] 用户访问 /dev/ds28e25 + sysfs
```

### 3.2 步骤1: DS2484 probe (内核启动时)

**文件**: drivers/w1/masters/ds2484.c : 546

```c
static int ds2484_probe(struct i2c_client *client,
                const struct i2c_device_id *id)
{
    // 1. 分配 DS2484 私有数据
    data = kzalloc(sizeof(struct ds2484_data), GFP_KERNEL);

    // 2. 挂接所有 1-Wire 操作函数到 w1_bus_master 结构体
    data->w1_ch.w1_bm.data       = &data->w1_ch;
    data->w1_ch.w1_bm.read_byte  = ds2484_w1_read_byte;
    data->w1_ch.w1_bm.write_byte = ds2484_w1_write_byte;
    data->w1_ch.w1_bm.touch_bit  = ds2484_w1_touch_bit;
    data->w1_ch.w1_bm.triplet    = ds2484_w1_triplet;
    data->w1_ch.w1_bm.reset_bus  = ds2484_w1_reset_bus;  // 核心改动!
    data->w1_ch.w1_bm.set_pullup = ds2484_w1_set_pullup;

    // 3. 复位 DS2484 芯片本身
    err = ds2484_reset(data);

    // 4. 配置 DS2484: OverDrive (1WS=1) + Active Pull-Up
    ds2484_write_config(data, (DS2484_REG_CFG_1WS | DS2484_REG_CFG_APU));

    // 5. 设置 OverDrive 时序参数
    ds2484_adjust_port(data, tRSTL_OD, 0x0A);
    ds2484_adjust_port(data, tMSP_OD, 0x08);
    ds2484_adjust_port(data, tW0L_OD, 0x0A);

    // 6. 首次复位 1-Wire 总线
    ds2484_w1_reset_bus(&data->w1_ch);

    // 7. 注册 w1 master -> 启动搜索线程
    err = w1_add_master_device(&data->w1_ch.w1_bm);
}
```

### 3.3 步骤2: w1 内核线程启动

**文件**: drivers/w1/w1_int.c : 105

```c
int w1_add_master_device(struct w1_bus_master *master)
{
    // 分配 w1_master 结构体 (设备名 w1_bus_master1)
    dev = w1_alloc_dev(id, w1_max_slave_count, w1_max_slave_ttl, ...);

    // 复制函数指针表 (ds2484 的 reset_bus 等)
    memcpy(dev->bus_master, master, sizeof(struct w1_bus_master));

    // 启动内核线程 w1_process -> 开始循环搜索
    dev->thread = kthread_run(&w1_process, dev, "%s", dev->name);

    list_add(&dev->w1_master_entry, &w1_masters);
}
```

w1_process 循环:

```c
int w1_process(void *data)
{
    while (!kthread_should_stop()) {
        if (dev->search_count) {
            mutex_lock(&dev->mutex);
            w1_search_process(dev, W1_SEARCH);  // 执行搜索
            mutex_unlock(&dev->mutex);
        }
        schedule_timeout(jtime);  // 每 w1_timeout 秒搜一次
    }
}
```

### 3.4 步骤3: 复位总线 (核心改动)

**文件**: drivers/w1/masters/ds2484.c : 471

当 DS28E25 插入时:

```c
static u8 ds2484_w1_reset_bus(void *data)
{
    struct ds2484_w1_chan *pchan = data;
    struct ds2484_data    *pdev = pchan->pdev;
    int err;
    u8 retval = 1;

    mutex_lock(&pdev->access_lock);

    // ===== 第一步: 标准速度复位 (1WS=0) =====
    ds2484_wait_1wire_idle(pdev);
    ds2484_write_config(pdev, DS2484_REG_CFG_APU);  // 1WS=0
    ds2484_wait_1wire_idle(pdev);   // 必须等待! 否则被忽略

    err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
    if (err < 0)
        dev_warn(&pdev->client->dev, "ds2484_send_cmd failed\n");

    err = ds2484_wait_1wire_idle(pdev);
    retval = err & DS2484_REG_STS_PPD;
    // DS28E25: PPD=0 (标准速度不响应)

    // ===== 第二步: 标准速度失败 -> OverDrive (1WS=1) =====
    if (!retval) {
        ds2484_write_config(pdev,
            DS2484_REG_CFG_APU | DS2484_REG_CFG_1WS);  // 1WS=1
        ds2484_wait_1wire_idle(pdev);  // 必须等待!

        err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
        if (err < 0)
            dev_warn(&pdev->client->dev, "ds2484_send_cmd failed\n");

        err = ds2484_wait_1wire_idle(pdev);
        retval = err & DS2484_REG_STS_PPD;
        // DS28E25: PPD=1 (OverDrive 响应)
    }

    mutex_unlock(&pdev->access_lock);
    return !retval;  // 0=有设备, 1=无设备
}
```

**实际运行日志:**
```
[  167.908] DS2484: STD speed reset, PPD=0   <- 标准速度没人
[  167.910] DS2484: OD speed reset, PPD=1    <- OD 检测到 DS28E25
```

### 3.5 步骤4: ROM 搜索

**文件**: drivers/w1/w1.c : 865

```c
void w1_search(struct w1_master *dev, u8 search_type,
               w1_slave_found_callback cb)
{
    while (slave_count < max_slave_count) {
        // 1. 复位总线 [调用 ds2484_w1_reset_bus]
        if (w1_reset_bus(dev)) break;

        // 2. 发送搜索命令 0xF0 (Search ROM)
        w1_write_8(dev, search_type);

        // 3. 逐位遍历 64-bit ROM 地址
        for (i = 0; i < 64; i++) {
            // triplet: 读 bit1 + 读 bit2 + 写方向
            triplet_ret = w1_triplet(dev, search_bit);
            // [调用 ds2484_w1_triplet -> DS2484 硬件执行]

            rn |= (triplet_ret >> 2) << i;
        }

        // 4. 找到设备, 调用回调
        cb(dev, rn);  // -> w1_slave_found()
    }
}
```

**DS2484 的 triplet 硬件实现:**
```c
static u8 ds2484_w1_triplet(void *data, u8 dbit)
{
    mutex_lock(&pdev->access_lock);
    ds2484_wait_1wire_idle(pdev);
    ds2484_send_cmd_data(pdev,
        DS2484_CMD_1WIRE_TRIPLET, dbit ? 0xFF : 0);
    status = ds2484_wait_1wire_idle(pdev);
    mutex_unlock(&pdev->access_lock);
    return (status >> 5);
}
```

### 3.6 步骤5: 发现设备 - w1_slave_found

**文件**: drivers/w1/w1.c : 830

```c
void w1_slave_found(struct w1_master *dev, u64 rn)
{
    struct w1_reg_num *tmp = (struct w1_reg_num *) &rn;
    sl = w1_slave_search_device(dev, tmp);
    if (sl) {
        set_bit(W1_SLAVE_ACTIVE, &sl->flags);  // 已存在
    } else {
        if (rn && tmp->crc == w1_calc_crc8((u8 *)&rn_le, 7))
            w1_attach_slave_device(dev, tmp);  // family=0x47
    }
}
```

### 3.7 步骤6: 注册设备

**文件**: drivers/w1/w1.c : 667

```c
static int w1_attach_slave_device(struct w1_master *dev,
                                   struct w1_reg_num *rn)
{
    sl = kzalloc(sizeof(struct w1_slave), GFP_KERNEL);
    memcpy(&sl->reg_num, rn, sizeof(sl->reg_num));

    // 查找家族驱动 (通过 family 码匹配)
    spin_lock(&w1_flock);
    f = w1_family_registered(rn->family);
    // rn->family = 0x47 -> 返回 w1_ds28e25 注册的驱动
    if (!f) {
        f = &w1_default_family;
    }
    sl->family = f;
    spin_unlock(&w1_flock);

    // 调用 __w1_attach_slave_device
    // 内部会调用 sl->family->fops->add_slave(sl)
    err = __w1_attach_slave_device(sl);
}
```

**家族驱动注册:**
```c
// w1_ds28e25.c - module_init 时注册
w1_register_family(&(struct w1_family){
    .fid = 0x47,
    .fops = &w1_ds28e25_fops,
});

// w1_ds2431.c - module_init 时注册
w1_register_family(&(struct w1_family){
    .fid = 0x2D,
    .fops = &w1_f2d_fops,
});
```

### 3.8 步骤7: DS28E25 add_slave

**文件**: drivers/w1/slaves/w1_ds28e25.c : 2194

```c
static int w1_ds28e25_add_slave(struct w1_slave *sl)
{
    // 1. 创建 sysfs 属性文件
    device_create_file(&sl->dev, &w1_read_user_eeprom_attr);
    device_create_file(&sl->dev, &w1_verifymac_attr);

    // 2. 分配私有数据, 注册 misc 设备 /dev/ds28e25
    data = kzalloc(sizeof(struct ds28e25_data), GFP_KERNEL);
    data->mdev.name = "ds28e25";
    data->mdev.fops = &w1_ds28e25_ops;
    misc_register(&data->mdev);

    // 3. ROM ID 备份与读取
    memcpy(rom_no, (u8 *)&sl->reg_num, sizeof(sl->reg_num));
    w1_ds28e25_read_and_update_romid(sl, rom_buf);

    // 4. 可选: 初始化 + 验证 (init_verify=1 时执行)
    if (init_verify) {
        if (skip_setup == 0) {
            w1_ds28e25_setup_device(sl);
            // 写 scratchpad -> load secret
            // 读状态 -> read-auth 验证
            skip_setup = 1;
            w1_ds28e25_verifymac(sl);
        }
    }

    sl->master->search_count = 0;  // 停止搜索
    return 0;
}
```

### 3.9 步骤8: 用户空间访问

```bash
/sys/bus/w1/devices/47-000000557843/
    |-- name           # 设备名
    |-- id             # ROM ID
    |-- read_eeprom    # 读 EEPROM
    |-- verifymac      # MAC 验证

/dev/ds28e25           # misc char device (open/read/write/ioctl)
```

## 4. GX2431 芯片插入—完整代码流程

### 4.1 总体流程

```
插入 GX2431 (family=0x2D)
    |
    v
[步骤1] DS2484 probe (同 DS28E25)
    |
    v
[步骤2] w1 内核线程启动 (同 DS28E25)
    |
    v
[步骤3] 复位总线 -> ds2484_w1_reset_bus()
    |        标准速度 PPD=1, 直接成功!
    v
[步骤4] ROM 搜索 -> w1_search() (同 DS28E25)
    |
    v
[步骤5] 发现设备 -> w1_slave_found()
    |
    v
[步骤6] 匹配家族驱动 0x2D -> w1_ds2431.c
    |
    v
[步骤7] GX2431 初始化 -> w1_f2d_add_slave()
    |        只创建 eeprom 文件
    v
[步骤8] 通过 /sys/.../eeprom 读写
```

### 4.2 步骤1-2: 与 DS28E25 完全相同

DS2484 probe 和 w1 线程启动的代码流程完全一样，请参看 [3.2] 和 [3.3]。

### 4.3 步骤3: 复位总线 - 标准速度直接成功

**文件**: drivers/w1/masters/ds2484.c : 471

```c
static u8 ds2484_w1_reset_bus(void *data)
{
    mutex_lock(&pdev->access_lock);

    // ===== 第一步: 标准速度复位 (1WS=0) =====
    ds2484_wait_1wire_idle(pdev);
    ds2484_write_config(pdev, DS2484_REG_CFG_APU);
    ds2484_wait_1wire_idle(pdev);

    err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
    err = ds2484_wait_1wire_idle(pdev);
    retval = err & DS2484_REG_STS_PPD;
    // GX2431: PPD=1 (标准速度响应!)

    // ===== retval != 0, 不执行第二步 =====
    // 函数直接返回, DS2484 保持标准速度

    mutex_unlock(&pdev->access_lock);
    return !retval;  // 0=有设备
}
```

**实际运行日志:**
```
[  283.246] DS2484: STD speed reset, PPD=1   <- 标准速度直接检到 GX2431
[  283.266] DS2484: STD speed reset, PPD=1   <- 后续每次复位都是标准速度
```

### 4.4 步骤4-5: 搜索与发现

`w1_search()` 和 `w1_slave_found()` 流程与 DS28E25 完全一致。
区别在于返回的 ROM 中 family=0x2D。

### 4.5 步骤6: 家族驱动匹配 - w1_ds2431

**文件**: drivers/w1/w1_family.c

```
w1_family_registered(0x2D)
    |
    v
在全局链表 w1_families 中查找 fid=0x2D 的家族驱动
    |
    v
找到 w1_ds2431.c 注册的驱动
    |  .fid = W1_EEPROM_DS2431 = 0x2D
    |  .fops = &w1_f2d_fops
    v
sl->family = w1_ds2431 的家族驱动
```

### 4.6 步骤7: GX2431(DS2431) add_slave

**文件**: drivers/w1/slaves/w1_ds2431.c : 277

```c
static int w1_f2d_add_slave(struct w1_slave *sl)
{
    // 唯一的工作: 创建 eeprom sysfs 文件
    return sysfs_create_bin_file(&sl->dev.kobj, &w1_f2d_bin_attr);
}
```

与 DS28E25 的 `add_slave` 相比，GX2431 的 add_slave **非常简单**:

| 对比项 | w1_ds28e25_add_slave | w1_f2d_add_slave |
|--------|:-:|:-:|
| 创建 sysfs 属性 | 3 个文件 | 1 个文件 |
| 分配内存 | kzalloc | 不需要 |
| 注册字符设备 | /dev/ds28e25 | 不需要 |
| 设备初始化 | setup_device + verifymac | 不需要 |
| 失败可能性 | **高** (初始化失败导致移除) | **极低** |

**GX2431 注册永不会失败** — 这是它和 DS28E25 的一个关键行为差异。

**eeprom 文件定义:**
```c
static struct bin_attribute w1_f2d_bin_attr = {
    .attr = {
        .name = "eeprom",
        .mode = S_IRUGO | S_IWUSR,  // 所有人可读, 仅 owner 可写
    },
    .size = W1_F2D_EEPROM_SIZE,   // = 128 字节
    .read = w1_f2d_read_bin,      // 读回调
    .write = w1_f2d_write_bin,    // 写回调
};
```

### 4.7 步骤8: 用户空间访问

```bash
# 查看设备
$ ls /sys/bus/w1/devices/
2d-000022765b82  w1_bus_master1

# 读 EEPROM (128 字节)
$ hexdump -C /sys/bus/w1/devices/2d-000022765b82/eeprom
00000000  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
00000010  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
00000020  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
...

# 写 EEPROM
$ echo -ne "\x01\x02\x03\x04" > /sys/bus/w1/devices/2d-000022765b82/eeprom
```

**GX2431 读操作流程 (w1_f2d_read_bin):**

```
用户读 eeprom
    |
    v
w1_f2d_read_bin()
    |
    v
循环调用 w1_f2d_readblock(sl, off, count, buf)
    |
    v
do {    // 最多重试 10 次
    w1_reset_select_slave(sl)   // 复位 + 选中芯片
    w1_write_block(READ_EEPROM=0xF0, addr_L, addr_H)  // 发读命令+地址
    w1_read_block(buf, count)   // 读数据
    
    w1_reset_select_slave(sl)   // 再次复位 + 选中
    w1_write_block(READ_EEPROM=0xF0, addr_L, addr_H)  // 发读命令+地址
    w1_read_block(cmp, count)   // 再次读数据 (比对)
    
    if (memcmp(cmp, buf) == 0)  // 两次读取一致 -> 成功
        return 0;
} while (--tries);
```

**GX2431 写操作流程 (w1_f2d_write_bin):**

```
用户写 eeprom
    |
    v
w1_f2d_write_bin()
    |
    v
对于每个 8 字节块:
    |
    v
w1_f2d_write(sl, addr, 8, data)
    |
    +-- w1_reset_select_slave(sl)
    +-- w1_write_block(WRITE_SCRATCH=0x0F, addr_L, addr_H)
    +-- w1_write_block(data, 8)           // 写暂存器
    |
    +-- w1_reset_select_slave(sl)
    +-- w1_write_8(READ_SCRATCH=0xAA)
    +-- w1_read_block(rdbuf, 8+3)          // 读暂存器 + 验证
    |   if (数据不匹配) goto retry;
    |
    +-- w1_reset_select_slave(sl)
    +-- w1_write_block(COPY_SCRATCH=0x55, ES)
    +-- msleep(11)                         // 等待 EEPROM 写入完成
    +-- w1_reset_bus(sl->master)           // 释放总线
```

## 5. 核心问题与最终解决方案

### 5.1 问题本质

两个芯片通过同一个手柄插入，共用：
- 同一条 1-Wire 总线
- 同一个 DS2484 主控制器
- 同一个 I2C 地址

但对速度的要求互相矛盾:

| 芯片 | OverDrive (125kbps) | 标准速度 (15kbps) |
|:----:|:-------------------:|:-----------------:|
| DS28E25 | 正常 | 不响应 |
| GX2431 | 不响应 | 正常 |

DS2484 的 1WS 位只能设置一个速度, 无法同时满足两个芯片。

### 5.2 尝试过的错误方案

| 方案 | 结果 | 失败原因 |
|:----|:----:|:---------|
| 固定标准速度 (1WS=0) | GX2431 OK, DS28E25 失败 | DS28E25 需要 OverDrive |
| 固定 OverDrive (1WS=1) | DS28E25 OK, GX2431 失败 | GX2431 不支持 OverDrive |
| 加 P0 (tRSTL) 标准速度参数 | 同上 | 不是复位长度问题 |
| 加 P2 (tMSP) 标准速度参数 | 同上 | 不是采样点问题 |
| 所有 P0/P2/P4 设最大 | 同上 | DS28E25 仍不认标准速度 |

### 5.3 最终方案: 动态速度切换

修改 ds2484_w1_reset_bus 函数, 在每次复位时依次尝试两种速度:

```c
static u8 ds2484_w1_reset_bus(void *data)
{
    struct ds2484_w1_chan *pchan = data;
    struct ds2484_data    *pdev = pchan->pdev;
    int err;
    u8 retval = 1;

    mutex_lock(&pdev->access_lock);

    // ===== 第一步: 标准速度 (1WS=0) =====
    ds2484_wait_1wire_idle(pdev);
    ds2484_write_config(pdev, DS2484_REG_CFG_APU);
    ds2484_wait_1wire_idle(pdev);   // [关键] 必须等待配置生效!

    err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
    if (err < 0)
        dev_warn(&pdev->client->dev, "ds2484_send_cmd failed\n");
    err = ds2484_wait_1wire_idle(pdev);
    retval = err & DS2484_REG_STS_PPD;

    // 如果 GX2431 在总线上, PPD=1, 直接结束
    // (后续通信继续用标准速度)

    // ===== 第二步: 没人 -> OverDrive (1WS=1) =====
    if (!retval) {
        ds2484_write_config(pdev,
            DS2484_REG_CFG_APU | DS2484_REG_CFG_1WS);
        ds2484_wait_1wire_idle(pdev);   // [关键] 必须等待配置生效!

        err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
        if (err < 0)
            dev_warn(&pdev->client->dev, "ds2484_send_cmd failed\n");
        err = ds2484_wait_1wire_idle(pdev);
        retval = err & DS2484_REG_STS_PPD;

        // 如果 DS28E25 在总线上, PPD=1, 结束
        // (后续通信继续用 OverDrive)
    }

    mutex_unlock(&pdev->access_lock);
    return !retval;
}
```

### 5.4 为什么之前动态切换失败了?

**关键: ds2484_wait_1wire_idle 必须在每次配置切换后调用!**

ds2484_write_config() 通过 I2C 写 DS2484 的配置寄存器后:
1. DS2484 内部需要时间处理新配置
2. 如果不等就立刻发下一个 I2C 命令, DS2484 可能忽略它
3. 被忽略的复位命令返回 PPD=0, 导致动态切换失败

正确序列:
```
ds2484_write_config(1WS=0)     // 写配置
ds2484_wait_1wire_idle()       // [必须] 等待配置生效
ds2484_send_cmd(CMD_RESET)     // 发复位命令
ds2484_wait_1wire_idle()       // 等待复位完成
```

### 5.5 probe 中的配置 (保持原样)

probe 函数中**不需要**任何特殊配置, 保持原始的 OverDrive 配置:

```c
ds2484_write_config(data, (DS2484_REG_CFG_1WS | DS2484_REG_CFG_APU));

ds2484_adjust_port(data, tRSTL_OD, 0x0A);
ds2484_adjust_port(data, tMSP_OD, 0x08);
ds2484_adjust_port(data, tW0L_OD, 0x0A);
```

### 5.6 最终测试结果

```
只插 DS28E25:
[  167.908] DS2484: STD speed reset, PPD=0   <- 标准速度没人
[  167.910] DS2484: OD speed reset, PPD=1    <- OD 检测到
$ ls /sys/bus/w1/devices/
47-000000557843  w1_bus_master1                <- DS28E25 出现 ✅

只插 GX2431:
[  283.246] DS2484: STD speed reset, PPD=1   <- 标准速度直接检测到
$ ls /sys/bus/w1/devices/
2d-000022765b82  w1_bus_master1                <- GX2431 出现 ✅
```

### 5.7 总结

| 项目 | 说明 |
|:----|:-----|
| 问题 | DS28E25 和 GX2431 需要不同的 1-Wire 速度 |
| 方案 | 动态速度切换: 每次复位先试标准速度, 不行再试 OverDrive |
| 关键 | 速度切换后必须等 DS2484 空闲再发命令 |
| 改动 | ds2484.c 的 ds2484_w1_reset_bus 一个函数 |
| 优点 | 不需要改硬件, 不影响其他功能 |

---

## 6. 附录: 相关源码文件索引

### W1 核心层

| 文件 | 路径 | 作用 |
|:----|:-----|:-----|
| w1.h | drivers/w1/w1.h | W1 核心数据结构定义 |
| w1.c | drivers/w1/w1.c | 总线管理、搜索算法、设备注册 |
| w1_int.h | drivers/w1/w1_int.h | 内部接口定义 |
| w1_int.c | drivers/w1/w1_int.c | master 设备分配/添加/移除 |
| w1_family.h | drivers/w1/w1_family.h | 家族驱动接口定义 |
| w1_family.c | drivers/w1/w1_family.c | 家族驱动注册/注销管理 |
| w1_io.c | drivers/w1/w1_io.c | 底层 IO 操作 |

### DS2484 主控制器驱动

| 文件 | 路径 | 作用 |
|:----|:-----|:-----|
| ds2484.c | drivers/w1/masters/ds2484.c | DS2484 I2C-to-1-Wire **核心改动文件** |

### W1 从设备驱动

| 文件 | 路径 | 作用 |
|:----|:-----|:-----|
| w1_ds2431.c | drivers/w1/slaves/w1_ds2431.c | GX2431 (family 0x2D) EEPROM 驱动 |
| w1_ds28e25.c | drivers/w1/slaves/w1_ds28e25.c | DS28E25 (family 0x47) 安全认证驱动 |
| w1_ds28e25_sha256.c | drivers/w1/slaves/w1_ds28e25_sha256.c | DS28E25 SHA-256 算法实现 |
| w1_ds28e25_sha256.h | drivers/w1/slaves/w1_ds28e25_sha256.h | SHA-256 头文件 |
| w1_gx2431.c | drivers/w1/slaves/w1_gx2431.c | GX2431 GPIO 驱动 (本次未使用) |

### 板级文件

| 文件 | 路径 | 作用 |
|:----|:-----|:-----|
| board-advantech.c | arch/arm/mach-omap2/board-advantech.c | 板级初始化, 注册 I2C 设备 |
| board-rom3310.h | arch/arm/mach-omap2/board-rom3310.h | I2C 设备表 (含 DS2484) |

### 配置文件

| 文件 | 路径 | 作用 |
|:----|:-----|:-----|
| am335x_rom3310_puhui_defconfig | arch/arm/configs/ | 内核配置 |

### 代码调用关系总图

```
board-advantech.c
    |
    +-- i2c_register_board_info(2, am335x_i2c1_boardinfo)
    |       |
    |       +-- {"ds2484", 0x18}
    |
    +--> ds2484.c: ds2484_probe()
            |
            +-- w1_bus_master 挂接函数指针
            |   reset_bus = ds2484_w1_reset_bus  [核心改动]
            |
            +-- w1_add_master_device()
                |
                +-- kthread_run(w1_process)
                    |
                    +-- w1_search_process()
                        |
                        +-- w1_search()
                            |
                            +-- w1_reset_bus() -> ds2484_w1_reset_bus()
                            |   [标准速度: GX2431]
                            |   [OverDrive:  DS28E25]
                            |
                            +-- w1_triplet()  -> ds2484_w1_triplet()  (x64)
                            |
                            +-- w1_slave_found()
                                |
                                +-- w1_attach_slave_device()
                                    |
                                    +-- w1_family_registered(family)
                                    |   |-- 0x47 -> w1_ds28e25.c
                                    |   |-- 0x2D -> w1_ds2431.c
                                    |
                                    +-- __w1_attach_slave_device()
                                        +-- add_slave()
                                            |-- w1_ds28e25_add_slave()
                                            |-- w1_f2d_add_slave()
```

---

*本文档基于 AM335XLBV1120 BSP 调试过程编写*
*记录了 DS2484 1-Wire 主控 + w1 子系统从驱动初始化到芯片识别的完整代码流程*
*以及 DS28E25 和 GX2431 两芯片共存问题的分析与解决方案*
