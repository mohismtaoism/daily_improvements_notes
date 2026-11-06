# GXE00 / GX2431 热插拔遗留设备问题排查记录
# Hot-Plug Stale Device Troubleshooting Log

> **日期**: 2026-07-21
> **涉及芯片**: GXE00 (0x14), GX2431 (0x2D), DS28E25 (0x47)
> **涉及层**: Linux Kernel W1 驱动 → 用户态 App (`OneWireEEPROM` 类体系)
> **编写目标**: 让新同事和 10 岁的编程爱好者都能看懂

---

## 目录

1. [现象：发生了什么？](#1-现象发生了什么)
2. [为什么会有这个 Bug？——给小朋友的比喻](#2-为什么会有这个-bug给小朋友的比喻)
3. [硬件架构全景（简单版）](#3-硬件架构全景简单版)
4. [软件架构全景（完整版）](#4-软件架构全景完整版)
5. [Bug 分析 —— 两个问题](#5-bug-分析--两个问题)
6. [修复方案](#6-修复方案)
7. [代码运行流程（完整调用链）](#7-代码运行流程完整调用链)
8. [OneWireEEPROM 多态类体系](#8-onewireeeprom-多态类体系)
9. [测试验证方法](#9-测试验证方法)
10. [相关文件清单](#10-相关文件清单)

---

## 1. 现象：发生了什么？

**英文**: When a GX2431 chip is pulled out from the 1-Wire socket, the sysfs directory
`/sys/bus/w1/devices/2d-000022765b8` **still exists**. When a new chip (GXE00)
is inserted, the app sees **both** the old stale GX2431 folder and the new GXE00
folder, picks the wrong one, and fails.

**中文**: 当 GX2431 芯片从 1-Wire 插座拔出后，Linux 系统里的
`/sys/bus/w1/devices/2d-000022765b8` **文件夹仍然存在**。此时插入新的 GXE00，
应用程序会同时看到「旧的 GX2431 空壳」和「新的 GXE00 真身」，
如果选错就会读不到数据。

### 日志证据

```
[OneWireEEPROM::detect]  detected GX2431 at /sys/bus/w1/devices/2d-000022765b8/eeprom
[GX2431::readReg]        page 1 failed (got 0 bytes)   ← 芯片已拔出，读不到数据
[KeyInput::getHandleInfo] failed to read handle data    ← 识别失败
```

---

## 2. 为什么会有这个 Bug？——给小朋友的比喻

想象你有一个**对讲机**（1-Wire 总线），你和几个朋友（芯片）轮流用对讲机说话。

### 正常工作

1. 小明（GX2431）插入对讲机 → 老师（W1 内核线程）在点名册上写：**「小明在」**
2. 小明拔出来 → 老师每 2 秒喊一次：**「小明还在吗？」** 没人回答
3. 连续 10 次没人回答（20 秒后）→ 老师擦掉点名册上的名字

### Bug 情况

老师（内核线程）的喊话器坏了——**不管有没有人回答，他都当作有人在**。
这样点名册上的名字永远擦不掉，新同学（GXE00）来了也没位置记。

**为什么喊话器会坏？**
因为 `ds2484_w1_reset_bus()` 里有一个变量 `retval = 1`，默认值是 **"有人"**。
当喊话器超时（返回 -1）时，代码没有检查这个错误，直接用了默认值 `1`，
就误报成 **"有人在"**。

---

## 3. 硬件架构全景（简单版）

```
┌─────────────────────────────────────────────────────┐
│                   主板 (ROM-3310)                     │
│                                                       │
│  CPU (AM3358)                                        │
│    ↓ I2C 总线 (I2C1, 地址 0x18)                       │
│  DS2484 —— I2C 转 1-Wire 桥接芯片                      │
│    ↓ 1-Wire 总线 (两根线: 数据 + 地线)                  │
│  ┌──────┬──────┬──────┐                              │
│  │GX2431│GXE00 │DS28E25│  ← 可以同时插多个             │
│  │0x2D  │0x14  │0x47   │                              │
│  └──────┴──────┴──────┘                              │
└─────────────────────────────────────────────────────┘
```

### 各芯片规格对比

| 芯片 | 厂牌 | 家族码 | 容量 | 访问方式 |
|------|------|--------|------|----------|
| **GX2431** | GXCAS | 0x2D | 128 字节 (4 页) | 内核 `w1_ds2431.c` → sysfs `/sys/.../2d-*/eeprom` |
| **GXE00** | GXCAS | 0x14 | 32 字节 (1 页) | 内核 `w1_gxe00.c` → sysfs `/sys/.../14-*/eeprom` |
| **DS28E25** | Maxim | 0x47 | 512 字节 + SHA256 | 内核 `w1_ds28e25.c` → `/dev/ds28e25` (字符设备) |

**关键**: GX2431 和 GXE00 都用 **sysfs eeprom 文件** 暴露给用户态，
而 DS28E25 用 **字符设备 + ioctl**。前两个是纯 EEPROM，后一个是
带加密功能的认证芯片。

---

## 4. 软件架构全景（完整版）

```
┌──────────────────────────────────────────────────────────────────┐
│  用户态 (User Space)                                              │
│                                                                   │
│  IdentifyTask / TestTask / TraceTask                              │
│       │                                                          │
│       ▼                                                          │
│  KeyInput (继承 InputDevice)                                      │
│       │                                                          │
│       │  getHandleInfo()                                         │
│       │  writeRemainTimes()                                      │
│       ▼                                                          │
│  OneWireEEPROM  ←── 抽象基类 (纯虚接口)                            │
│       │                                                          │
│       ├── GXE00     ←── 直接继承 OneWireEEPROM                    │
│       │    路径: /sys/bus/w1/devices/14-*/eeprom                  │
│       │    大小: 32 字节                                          │
│       │    数据布局: 紧凑型 (id 4B + times 1B + type 1B + ac 1B)  │
│       │                                                          │
│       ├── GX2431    ←── 直接继承 OneWireEEPROM                    │
│       │    路径: /sys/bus/w1/devices/2d-*/eeprom                  │
│       │    大小: 128 字节                                         │
│       │    数据布局: 位打包型 (每 bit 占 1 byte)                    │
│       │                                                          │
│       └── DS28E25Chip  ←── 直接继承 OneWireEEPROM                 │
│            路径: /dev/ds28e25  (ioctl)                            │
│            大小: 512 字节                                         │
│            数据布局: 位打包型                                      │
│                                                                   │
│  工厂方法:                                                        │
│    OneWireEEPROM::detect()  →  返回第一个可用芯片                   │
│    OneWireEEPROM::listAll() →  返回所有可用芯片列表                  │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  内核态 (Kernel Space)                                            │
│                                                                   │
│  W1 子系统 (drivers/w1/)                                         │
│    │                                                             │
│    ├── w1.c           W1 核心框架：总线扫描、设备枚举、TTL 管理     │
│    ├── w1_io.c        W1 I/O 原语：reset, read, write, triplet     │
│    ├── w1_int.c       初始化：分配 master、创建内核线程             │
│    ├── w1_family.h    家族码定义                                  │
│    │                                                             │
│    ├── masters/                                                   │
│    │   └── ds2484.c   DS2484 I2C-to-1-Wire 桥接驱动              │
│    │         ├── ds2484_w1_reset_bus()  ←── Bug 所在地            │
│    │         ├── ds2484_w1_write_byte()                          │
│    │         ├── ds2484_w1_read_byte()                           │
│    │         └── ds2484_w1_triplet()    ←── 搜索算法的核心        │
│    │                                                             │
│    └── slaves/                                                   │
│          ├── w1_ds2431.c     0x2D: GX2431 驱动 (128 字节 EEPROM) │
│          ├── w1_gxe00.c      0x14: GXE00 驱动 (32 字节 EEPROM)   │
│          └── w1_ds28e25.c    0x47: DS28E25 驱动 (SHA256 认证)    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Bug 分析 —— 两个问题

### 问题一：内核 `ds2484_w1_reset_bus()` 超时误报

**文件**: `source/linux-3.2.0-psp04.06.00.11/drivers/w1/masters/ds2484.c`

**旧代码 (有 Bug)**:

```c
static u8 ds2484_w1_reset_bus(void *data)
{
    ...
    u8 retval = 1;   // ← Bug: 默认值 1, 意思是 "设备存在"
    
    err = ds2484_wait_1wire_idle(pdev);
    if (err >= 0)
        retval = err & DS2484_REG_STS_PPD;
    
    // 如果 wait_1wire_idle 超时返回 -1, err >= 0 不成立,
    // retval 仍保持 1, 函数最终返回 !retval = 0, 即 "设备存在"
    
    return !retval;   // 0 = 设备存在, 1 = 设备不存在
}
```

**为什么超时会发生**:
- DS2484 芯片的 I2C 总线可能暂时出错
- 芯片正忙（1-Wire 操作未完成）且超过 100 次轮询超时
- 此时 `ds2484_wait_1wire_idle()` 返回 -1

**修复**: 将 `retval = 1` 改成 `retval = 0`。

```c
u8 retval = 0;   // 修复: 默认 "设备不存在"，让 W1 子系统去确认
```

这样，当 `wait_1wire_idle` 超时时，`retval = 0` → `!retval = 1` → 正确报告 "设备不存在"。

### 问题二：用户态 `detect()` 没有验证路径是否健在

**文件**: `source/apps/app/platform/input/onewire_eeprom.cpp`

即使内核做了修复，内核的 TTL 清理机制也需要时间（默认 10 次扫描 × 2 秒 = 20 秒）。
如果用户在这个窗口内切换芯片，`glob()` 会同时返回新旧两个路径。

**旧代码**:

```cpp
// 旧: 找到第一个路径就相信它
glob_t g;
glob("/sys/bus/w1/devices/14-*/eeprom", ...);
if (ret == 0 && g.gl_pathc > 0) {
    // 直接使用 g.gl_pathv[0], 不做活跃性检查
    return new GXE00(g.gl_pathv[0]);
}
```

**修复**: 添加 `isEepromAlive()` 验证

```cpp
static bool isEepromAlive(const char *path)
{
    FILE *fp = fopen(path, "rb");
    if (!fp) return false;
    unsigned char byte;
    int got = fread(&byte, 1, 1, fp);
    fclose(fp);
    return got == 1;  // 活的芯片能读出 1 字节
}
```

`detect()` 现在遍历所有 glob 路径，**只接受能真正读出数据的路径**：

```cpp
for (size_t i = 0; i < g.gl_pathc; i++) {
    if (isEepromAlive(g.gl_pathv[i])) {
        return new GXE00(g.gl_pathv[i]);
    }
    // 跳过假路径
}
```

---

## 6. 修复方案

| 层 | 文件 | 修改内容 | 效果 |
|----|------|----------|------|
| ⚙️ 内核 | `ds2484.c` | `retval = 1` → `retval = 0` | 超时时正确报告"无设备" |
| 📱 应用 | `onewire_eeprom.cpp` | 添加 `isEepromAlive()` 验证 | 跳过 sysfs 中的僵尸路径 |

---

## 7. 代码运行流程（完整调用链）

### 7.1 应用启动 → 芯片识别

```
main.cpp:main()
  → TaskManager::start()
    → IdentifyTask::run()
      → KeyInput::getHandleInfo(&info)             [keyinput.cpp]
          │
          │  retry_times < 5  (最多重试 5 次)
          │
          ├─→ OneWireEEPROM::detect()              [onewire_eeprom.cpp]
          │     │
          │     ├─ glob("/sys/bus/w1/devices/14-*/eeprom")
          │     │   └─ isEepromAlive(path)  →  true → new GXE00(path)
          │     │
          │     ├─ glob("/sys/bus/w1/devices/2d-*/eeprom")
          │     │   └─ isEepromAlive(path)  →  true → new GX2431(path)
          │     │
          │     └─ stat("/dev/ds28e25")
          │         └─ 存在 → new DS28E25Chip()
          │
          └─→ chip->readHandleId(info->id)          [虚函数派发]
              │
              ├── GXE00:    read(0, buf, 4) → 小端解码 → id
              ├── GX2431:   read(0, pageBuf, 32) → 位解码 → id
              └── DS28E25Chip: read(0, pageBuf, 32) → 位解码 → id
               + chip->readHandleTimes(...)
               + chip->readHandleType(...)
               + chip->readHandleAC(...)
```

### 7.2 芯片拔出 → W1 总线扫描循环

```
内核线程 w1_process()  [w1.c]
  │
  │ 每 2 秒 (w1_timeout) 循环：
  │
  ├─→ ds2484_w1_reset_bus(&data->w1_ch)    [ds2484.c]
  │     ├─ 发送 DS2484_CMD_1WIRE_RESET
  │     ├─ 等待完成
  │     └─ 读取 PPD 位 (Presence Pulse Detected)
  │          ├─ 1 = 有设备应答
  │          └─ 0 = 无设备 (或超时)
  │
  ├─→ w1_search(dev, W1_SEARCH, callback)   [w1.c]
  │     └─ 通过 triplet 逐位搜索 ROM ID
  │
  └─→ w1_search_process_cb()                [w1.c]
        ├─ 清除所有 W1_SLAVE_ACTIVE 标记
        ├─ 执行搜索 → 找到的芯片标记为 ACTIVE
        └─ 遍历设备列表：
             ├─ 非 ACTIVE 且 ttl-- = 0 → 删除设备
             └─ ACTIVE → 重置 ttl = slave_ttl
```

### 7.3 应用层写数据流 (以 `writeRemainTimes` 为例)

```
WriteHandleTimesDialog::onSave()
  → KeyInput::writeRemainTimes(times)          [keyinput.cpp]
    → OneWireEEPROM::detect()                  发现芯片
    → chip->writeHandleTimes(times)            虚函数派发
        ├── GXE00:    write(4, &times, 1)      直接写 1 字节
        └── GX2431:   writeReg(1, times, 8)    写第 1 页, 8 位
    → delete chip
```

---

## 8. OneWireEEPROM 多态类体系

### 设计目标

让 `KeyInput` 层**完全不知道**具体是哪种 1-Wire 芯片。
它只跟基类 `OneWireEEPROM` 打交道的虚函数接口，实际的芯片差异被子类隐藏。

### 类图

```
┌──────────────────────────────┐
│   OneWireEEPROM (抽象基类)    │
│                              │
│  + read() = 0                │
│  + write() = 0               │
│  + readHandleId() = 0        │
│  + writeHandleId() = 0       │
│  + readHandleTimes() = 0     │
│  + writeHandleTimes() = 0    │
│  + readHandleType() = 0      │
│  + writeHandleType() = 0     │
│  + readHandleAC() = 0        │
│  + writeHandleAC() = 0       │
│                              │
│  + detect() [static]         │
│  + listAll() [static]        │
└──────────────┬───────────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
     ▼         ▼         ▼
┌────────┐┌────────┐┌───────────┐
│ GXE00  ││ GX2431 ││DS28E25Chip│
│        ││        ││           │
│ 32 字节││128字节 ││ 512 字节  │
│ sysfs  ││ sysfs  ││ /dev/     │
│ 紧凑型 ││ 位打包 ││ ioctl     │
└────────┘└────────┘└───────────┘
```

### 三种布局对比

| 方案 | ID (4B) | Times (1B) | Type (1B) | AC (1B) | 特点 |
|------|----------|------------|-----------|---------|------|
| **GXE00 紧凑型** | 偏移 0-3 (int32 LE) | 偏移 4 | 偏移 5 | 偏移 6 | 高效、可移植 |
| **GX2431 位打包** | 页 0 的 bit 0-31 | 页 1 bit 0-7 | 页 2 bit 0-7 | 页 3 bit 0-7 | 兼容 HcData 旧格式 |
| **DS28E25 位打包** | 页 0 的 bit 0-31 | 页 1 bit 0-7 | 页 2 bit 0-7 | 页 3 bit 0-7 | 同 GX2431 格式 |

---

## 9. 测试验证方法

### 9.1 内核层验证

```bash
# 检查 DS2484 复位是否正常
dmesg | grep DS2484
# 期望: 每次芯片插入都能看到 PPD=1

# 检查 W1 设备列表
ls /sys/bus/w1/devices/

# 拔出芯片后等待 30 秒，再检查
ls /sys/bus/w1/devices/
# 期望: 拔出的芯片条目已被清除

# 手动触发搜索
cat /sys/bus/w1/devices/w1_bus_master1/slaves
```

### 9.2 应用层验证

```bash
# 插入 GXE00，查看 app 日志，期望：
[OneWireEEPROM::detect] detected GXE00 at /sys/bus/w1/devices/14-*/eeprom
[GXE00::readHandleId]   OK
[KeyInput::getHandleInfo] OK

# 拔出 GXE00，插入 GX2431，期望：
[OneWireEEPROM::detect] detected GX2431 at /sys/bus/w1/devices/2d-*/eeprom

# 快速热切换（拔出旧芯片立即插入新芯片，不等 20 秒），期望：
# detect() 找到了 14-* 路径，但 isEepromAlive() 验证失败，
# 自动跳过，继续找 2d-* 路径 → 成功
```

### 9.3 边界情况

| 场景 | 预期行为 |
|------|----------|
| 没有插任何芯片 | `detect()` 返回 `nullptr` |
| 只插 GX2431 | 返回 GX2431 实例 |
| 只插 GXE00 | 返回 GXE00 实例 |
| 先拔 GX2431, 马上插 GXE00 | 跳过 2d-* 僵尸路径, 选中 14-* |
| 同时插多个同型号 | `detect()` 返回第一个活的, `listAll()` 返回全部 |

---

## 10. 相关文件清单

### 内核驱动

| 文件 | 路径 | 说明 |
|------|------|------|
| DS2484 Master | `source/linux-.../drivers/w1/masters/ds2484.c` | I2C-to-1-Wire 桥接芯片驱动, **Bug 修复点** |
| GX2431 Slave | `source/linux-.../drivers/w1/slaves/w1_ds2431.c` | GX2431 内核驱动 (复用 DS2431 驱动) |
| GXE00 Slave | `source/linux-.../drivers/w1/slaves/w1_gxe00.c` | GXE00 内核驱动 |
| DS28E25 Slave | `source/linux-.../drivers/w1/slaves/w1_ds28e25.c` | DS28E25 内核驱动 |
| W1 Core | `source/linux-.../drivers/w1/w1.c` | W1 子系统核心：扫描、TTL、设备生命周期 |
| Kconfig | `source/linux-.../drivers/w1/slaves/Kconfig` | 新增 `CONFIG_W1_SLAVE_GXE00` |
| Makefile | `source/linux-.../drivers/w1/slaves/Makefile` | 新增 `obj-$(CONFIG_W1_SLAVE_GXE00)` |
| 主 Defconfig | `source/linux-.../arch/arm/configs/am335x_rom3310_puhui_defconfig` | 启用 GXE00 |
| 辅助 Defconfig | `arch/arm/configs/am335x_rom3310_puhui_no-logo_defconfig` | 启用 GXE00 |

### 应用层 (C++)

| 文件 | 路径 | 说明 |
|------|------|------|
| 基类 | `source/apps/app/platform/input/onewire_eeprom.h` | 抽象基类 + 纯虚 handle-data 接口 |
| 基类实现 | `source/apps/app/platform/input/onewire_eeprom.cpp` | 工厂方法 + `isEepromAlive()`, **修复点** |
| GXE00 | `source/apps/app/platform/input/gxe00.h / .cpp` | GXE00 子类: 紧凑型数据布局 |
| GX2431 | `source/apps/app/platform/input/gx2431.h / .cpp` | GX2431 子类: 位打包数据布局 |
| DS28E25 | `source/apps/app/platform/input/ds28e25_chip.h / .cpp` | DS28E25 子类: ioctl 封装 |
| KeyInput | `source/apps/app/platform/input/keyinput.cpp` | 应用业务逻辑, 调用 `OneWireEEPROM` 接口 |
| 构建文件 | `source/apps/app/platform/platform.pri` | Qt .pro 包含文件, 列出所有源文件 |

### 文档

| 文件 | 说明 |
|------|------|
| `doc/gxe00_stale_device_troubleshooting.md` | **本文** |
| `doc/gx2431_working_log.md` | GX2431 双速复位调测记录 |
| `doc/w1_onewire_debug_record.md` | W1 调试记录 |
| `source/driver_trainning.md` | 内核驱动培训文档 |

---

*EOF — End of troubleshooting log*
