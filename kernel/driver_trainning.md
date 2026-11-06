# 🎓 Linux Driver Training for 1-Wire EEPROM on AM335x ROM-3310

> **Target Hardware**: ROM-3310 COM Express + Puhui carrier board
> **Kernel**: Linux 3.2.0 (TI PSP 04.06.00.11) — board-file based (no Device Tree)
> **1-Wire Master**: DS2484 I2C-to-1-Wire bridge (I2C1, addr 0x18)
> **Existing Devices**: DS28E25 (family 0x47, secure authenticator + 512B EEPROM)
> **New Devices**: GX2431 (family 0x2D, 1024-bit EEPROM, DS2431-compatible), GXE00 (family 0x14, 256-bit EEPROM)

---

## 📖 Table of Contents

1. [Overview: Where We're Going](#1-overview-where-were-going)
2. [Lesson 1: Kernel Module Basics](#2-lesson-1-kernel-module-basics)
3. [Lesson 2: The Linux Device Model & W1 Subsystem](#3-lesson-2-the-linux-device-model--w1-subsystem)
4. [Lesson 3: The 1-Wire Protocol & EEPROM Command Set](#4-lesson-3-the-1-wire-protocol--eeprom-command-set)
5. [Lesson 4: Anatomy of w1_ds2431.c (GX2431's Twin)](#5-lesson-4-anatomy-of-w1_ds2431c-gx2431s-twin)
6. [Lesson 5: Anatomy of w1_gxe00.c (New Driver)](#6-lesson-5-anatomy-of-w1_gxe00c-new-driver)
7. [Lesson 6: The DS28E25 — Why So Complex?](#7-lesson-6-the-ds28e25--why-so-complex)
8. [Lesson 7: Build System Integration](#8-lesson-7-build-system-integration)
9. [Lesson 8: Testing & Debugging](#9-lesson-8-testing--debugging)
10. [Reference: Quick Card](#10-reference-quick-card)

---

## 1. Overview: Where We're Going

### The Hardware Stack

```
┌──────────────────────────────────┐
│  Userspace (app, shell, ...)     │  ← cat /sys/bus/w1/devices/14-*/eeprom
├──────────────────────────────────┤
│  Sysfs bin_attribute "eeprom"    │  ← Your driver's interface to userspace
├──────────────────────────────────┤
│  W1 subsystem (w1 core)          │  ← Bus scanning, family matching, I/O primitives
├──────────────────────────────────┤
│  DS2484 master driver            │  ← I2C-to-1-Wire bridge (I2C1, addr 0x18)
├──────────────────────────────────┤
│  I2C core                        │
├──────────────────────────────────┤
│  Hardware (ROM-3310 + Puhui)     │  ← AM3358 + peripheral ICs
└──────────────────────────────────┘
```

### The Three Devices

| Chip | Manufacturer | Family Code | Memory | Existing Driver | What We Need |
|------|-------------|-------------|--------|-----------------|-------------|
| **DS28E25** | Maxim (existing) | 0x47 | 512B + SHA-256 | w1_ds28e25.c ✓ | Keep working (legacy) |
| **GX2431** | GXCAS (new) | 0x2D | 1024 bits (128B) | w1_ds2431.c exists but **not enabled** | Enable in defconfig |
| **GXE00** | GXCAS (new) | 0x14 | 256 bits (32B) | **none** — must write from scratch | New w1_gxe00.c driver |

> **Good News**: GX2431 is 100% pin/command-compatible with Maxim DS2431. The kernel already has w1_ds2431.c — just enable it.
> **GXE00** shares the same command set but with a different family code (0x14) and smaller memory (32 bytes vs 128).

### What a W1 Slave Driver Does

```
                  ┌──────────────────┐
                  │   w1_register_   │
                  │   family(&... )  │  ← Register family code at init
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │   add_slave()    │  ← Called when device found on bus
                  │   Create sysfs   │
                  │   files / dev    │
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │  read_bin() /    │  ← Userspace reads/writes through these
                  │  write_bin()     │
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │  remove_slave()  │  ← Called when device removed
                  │  Clean up        │
                  └──────────────────┘
```

---

## 2. Lesson 1: Kernel Module Basics

### 2.1 Module Skeleton

Every Linux driver has two entry points — init and exit:

```c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init my_driver_init(void)
{
    // Called at boot (built-in) or insmod (module)
    pr_info("Hello from my driver!\n");
    return 0;  // 0 = success, negative = error
}

static void __exit my_driver_fini(void)
{
    // Called at shutdown or rmmod
    pr_info("Goodbye from my driver!\n");
}

module_init(my_driver_init);
module_exit(my_driver_fini);

MODULE_LICENSE("GPL");           // Required! Kernel refuses proprietary modules
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("What this does");
```

### 2.2 __init, __exit, __initdata

```c
static int __init my_init(void);    // Code freed after boot (not needed in RAM)
static void __exit my_exit(void);   // Discarded if built-in (only used for modules)
static int my_data __initdata = 42; // Data freed after boot
```

### 2.3 Built-in vs Module

| | =y (built-in) | =m (module) |
|---|---|---|
| **When init runs** | At kernel boot | When insmod / modprobe is run |
| **When exit runs** | Never (discarded) | When mmod is run |
| **Size in RAM** | Always resident | Only when loaded |
| **In this BSP** | Everything built-in | Not used |

### 2.4 printk / dev_printk Logging

```c
// For early boot / no device context:
pr_info("format", ...);    // KERN_INFO
pr_debug("format", ...);   // KERN_DEBUG (need DEBUG defined or dynamic debug)
pr_err("format", ...);     // KERN_ERR
pr_warn("format", ...);    // KERN_WARNING

// For a device context (preferred in drivers):
dev_info(&sl->dev, "format", ...);    // Shows device path prefix
dev_err(&sl->dev, "format", ...);     // Error with device context
dev_dbg(&sl->dev, "format", ...);     // Debug with device context
```

View logs: dmesg | tail on the target.

---

## 3. Lesson 2: The Linux Device Model & W1 Subsystem

### 3.1 The Device Model Trinity

```
┌────────────────────────────────────────────────────────┐
│                    struct bus_type                     │
│  W1 bus (w1_bus_type)                                  │
│  - Matches devices to drivers by family code           │
│  - Scans for new devices periodically                  │
└────────────────────────┬───────────────────────────────┘
                         │
           ┌─────────────┴────────────┐
           │                          │
┌──────────▼──────────┐    ┌──────────▼───────────┐
│   struct device     │    │   struct device_     │
│   (w1_slave)        │    │   driver (w1_family) │
│                     │    │                      │
│  - reg_num (ROM ID) │    │  - fid (family code) │
│  - family_data *    │    │  - fops (add/remove) │
│  - dev (kobject)    │    │                      │
└─────────────────────┘    └──────────────────────┘
```

### 3.2 Key W1 Data Structures

From drivers/w1/w1_family.h:

```c
// The 64-bit ROM ID — first byte is the FAMILY CODE
struct w1_reg_num {
#if defined(__LITTLE_ENDIAN_BITFIELD)
    __u64   family:8,     // LSB: Device family (e.g., 0x14, 0x2D, 0x47)
            id:48,        // Unique 48-bit serial number
            crc:8;        // CRC8 of previous 56 bits
#else
    __u64   crc:8,
            id:48,
            family:8;
#endif
};

// The slave device — one per chip found on the bus
struct w1_slave {
    struct module       *owner;
    unsigned char       name[W1_MAXNAMELEN];   // e.g., "14-000000123456"
    struct w1_reg_num   reg_num;               // The 64-bit ROM ID
    u8                  rom[9];                // Raw ROM bytes
    struct w1_master    *master;               // Bus master (DS2484)
    struct w1_family    *family;               // Our registered family
    void                *family_data;          // ← YOUR private data buffer
    struct device       dev;                   // Kernel device (for sysfs)
};
```

### 3.3 Registration Flow

```c
// Step 1: Define operations
static struct w1_family_ops my_fops = {
    .add_slave    = my_add_slave,      // Called when device found
    .remove_slave = my_remove_slave,   // Called when device removed
};

// Step 2: Define family with family code
static struct w1_family my_family = {
    .fid  = 0x14,                      // The family code from ROM byte 0
    .fops = &my_fops,
};

// Step 3: Register at init
static int __init my_init(void) {
    return w1_register_family(&my_family);
}
module_init(my_init);

// Step 4: Unregister at exit
static void __exit my_exit(void) {
    w1_unregister_family(&my_family);
}
module_exit(my_exit);
```

### 3.4 The add_slave / remove_slave Lifecycle

```c
static int my_add_slave(struct w1_slave *sl)
{
    // Called when the W1 core scans the bus and finds a device
    // whose family code matches our .fid.
    //
    // At this point:
    //   - sl->reg_num.family == 0x14
    //   - sl->reg_num.id has the 48-bit unique serial
    //   - The device is on the bus and ready to talk
    //
    // Your job: create sysfs files for userspace access.
    int ret;

    ret = sysfs_create_bin_file(&sl->dev.kobj, &my_bin_attr);
    if (ret)
        dev_err(&sl->dev, "Failed to create sysfs file: %d\n", ret);

    return ret;  // 0 = success, negative = probe failure
}

static void my_remove_slave(struct w1_slave *sl)
{
    // Clean up what add_slave created
    sysfs_remove_bin_file(&sl->dev.kobj, &my_bin_attr);
}
```

### 3.5 Sysfs Binary Attributes — The Essential Pattern

```c
static ssize_t my_read_bin(struct file *filp, struct kobject *kobj,
                            struct bin_attribute *bin_attr,
                            char *buf, loff_t off, size_t count)
{
    struct w1_slave *sl = kobj_to_w1_slave(kobj);
    // ... read device into buf at offset off, up to count bytes ...
    return bytes_read;  // or negative error
}

static ssize_t my_write_bin(struct file *filp, struct kobject *kobj,
                             struct bin_attribute *bin_attr,
                             const char *buf, loff_t off, size_t count)
{
    struct w1_slave *sl = kobj_to_w1_slave(kobj);
    // ... write count bytes from buf to device at offset off ...
    return bytes_written;  // or negative error
}

static struct bin_attribute my_bin_attr = {
    .attr = {
        .name = "eeprom",           // Filename in sysfs
        .mode = S_IRUGO | S_IWUSR,  // 0444 read-all, 0200 write-owner
    },
    .size  = 32,                     // Total file size (0 = variable)
    .read  = my_read_bin,
    .write = my_write_bin,
};
```

### 3.6 The Master Mutex — Bus Sharing

The W1 bus is **shared** among all devices. You MUST hold the bus lock:

```c
mutex_lock(&sl->master->mutex);
// ... sequence of 1-Wire operations ...
mutex_unlock(&sl->master->mutex);
```

Forgetting this = corrupted transfers when two processes access the bus simultaneously.

### 4.3 Read Flow (for all three chips)

```
Master: [RESET] [MATCH_ROM: 0x55 + 8-byte ROM ID]
Device: [Presence pulse]
Master: [0xF0] [addr_low] [addr_high]
Device: sends data bytes

Then do it AGAIN for verification ("proof read"):
Master: [RESET] [MATCH_ROM: 0x55 + 8-byte ROM ID]
Device: [Presence pulse]
Master: [0xF0] [addr_low] [addr_high]
Device: sends data bytes

Compare both reads — if they match, data is valid.
```

What this looks like in code:

```c
static int my_readblock(struct w1_slave *sl, int off, int count, char *buf)
{
    u8 wrbuf[3] = { 0xF0, off & 0xff, off >> 8 };
    u8 cmp[8];
    // First read
    w1_reset_select_slave(sl);
    w1_write_block(sl->master, wrbuf, 3);
    w1_read_block(sl->master, buf, count);
    // Second read (proof read)
    w1_reset_select_slave(sl);
    w1_write_block(sl->master, wrbuf, 3);
    w1_read_block(sl->master, cmp, count);
    if (memcmp(cmp, buf, count) == 0)
        return 0;
    return -1;
}
```

### 4.4 Write Flow (Scratchpad Pattern)

EEPROM cannot be written directly — you must use the scratchpad:

```
STEP 1: Write data to scratchpad
  Master: [RESET] [SELECT] [0x0F] [addr_low] [addr_high] [data]

STEP 2: Read back scratchpad and verify
  Master: [RESET] [SELECT] [0xAA]
  Device: [TA1] [TA2] [ES] [data bytes...]
  Verify: (TA1 == addr_low) && (TA2 == addr_high) &&
          (ES == expected_end) && (data == what we wrote)

STEP 3: Copy to EEPROM
  Master: [RESET] [SELECT] [0x55] [addr_low] [addr_high] [ES]
  Wait: msleep(PROG_TIME)  <- 10-11ms

STEP 4: Release
  Master: [RESET]
```

### 4.5 Why the Scratchpad?

1. **1-Wire is half-duplex** — no separate clock. The device can't say "that byte was wrong."
2. **EEPROM writes take ~10ms** — verify before committing.
3. **Power loss during write** — the verification catches this before the EEPROM is touched.

### 4.6 Handle Partial / Misaligned Writes

Userspace may write to any offset, but the scratchpad is 8 bytes. For unaligned writes, use **read-modify-write**:

```c
if (len < 8 || addr & 0x07) {
    char tmp[8];
    my_readblock(sl, addr & ~0x07, 8, tmp);
    copy = 8 - (addr & 0x07);
    if (copy > len) copy = len;
    memcpy(&tmp[addr & 0x07], buf, copy);
    my_write(sl, addr & ~0x07, 8, tmp);
} else {
    my_write(sl, addr, 8, buf);
}
```

---

## 5. Lesson 4: Anatomy of w1_ds2431.c (GX2431's Twin)

The GX2431 is compatible with the Maxim DS2431. The kernel already has `drivers/w1/slaves/w1_ds2431.c` (310 lines).

### 5.1 Constants

```c
#define W1_F2D_EEPROM_SIZE      128     // 1024 bits = 128 bytes
#define W1_F2D_PAGE_COUNT       4       // 4 pages x 32 bytes
#define W1_F2D_PAGE_BITS        5       // 2^5 = 32 bytes per page
#define W1_F2D_PAGE_SIZE        32
#define W1_F2D_PAGE_MASK        0x1F
#define W1_F2D_SCRATCH_BITS     3       // 2^3 = 8 bytes scratchpad
#define W1_F2D_SCRATCH_SIZE     8
#define W1_F2D_SCRATCH_MASK     0x07
#define W1_F2D_READ_EEPROM      0xF0
#define W1_F2D_WRITE_SCRATCH    0x0F
#define W1_F2D_READ_SCRATCH     0xAA
#define W1_F2D_COPY_SCRATCH     0x55
#define W1_F2D_TPROG_MS         11
#define W1_F2D_READ_RETRIES     10
#define W1_F2D_READ_MAXLEN      8
```

**Naming**: `F2D` = Family 2D (0x2D). Historical W1 driver convention.

### 5.2 Bound Checking

```c
static inline size_t w1_f2d_fix_count(loff_t off, size_t count, size_t size)
{
    if (off > size) return 0;
    if ((off + count) > size) return size - off;
    return count;
}
```

### 5.3 The Read Function

```c
static int w1_f2d_readblock(struct w1_slave *sl, int off, int count, char *buf)
{
    u8 wrbuf[3];
    u8 cmp[W1_F2D_READ_MAXLEN];
    int tries = W1_F2D_READ_RETRIES;

    do {
        wrbuf[0] = W1_F2D_READ_EEPROM;
        wrbuf[1] = off & 0xff;
        wrbuf[2] = off >> 8;
        w1_reset_select_slave(sl);
        w1_write_block(sl->master, wrbuf, 3);
        w1_read_block(sl->master, buf, count);
        w1_reset_select_slave(sl);
        w1_write_block(sl->master, wrbuf, 3);
        w1_read_block(sl->master, cmp, count);
        if (!memcmp(cmp, buf, count))
            return 0;
    } while (--tries);
    dev_err(&sl->dev, "proof reading failed %d times\n", W1_F2D_READ_RETRIES);
    return -1;
}
```

Key: proof read with retry, max 8 bytes per call, sysfs handler loops.

### 5.4 The Write Function

```c
static int w1_f2d_write(struct w1_slave *sl, int addr, int len, const u8 *data)
{
    int tries = W1_F2D_READ_RETRIES;
    u8 wrbuf[4];
    u8 rdbuf[W1_F2D_SCRATCH_SIZE + 3];
    u8 es = (addr + len - 1) % W1_F2D_SCRATCH_SIZE;

retry:
    w1_reset_select_slave(sl);
    wrbuf[0] = W1_F2D_WRITE_SCRATCH;
    wrbuf[1] = addr & 0xff;
    wrbuf[2] = addr >> 8;
    w1_write_block(sl->master, wrbuf, 3);
    w1_write_block(sl->master, data, len);

    w1_reset_select_slave(sl);
    w1_write_8(sl->master, W1_F2D_READ_SCRATCH);
    w1_read_block(sl->master, rdbuf, len + 3);

    if ((rdbuf[0] != wrbuf[1]) || (rdbuf[1] != wrbuf[2]) ||
        (rdbuf[2] != es) || (memcmp(data, &rdbuf[3], len) != 0)) {
        if (--tries) goto retry;
        return -1;
    }

    w1_reset_select_slave(sl);
    wrbuf[0] = W1_F2D_COPY_SCRATCH;
    wrbuf[3] = es;
    w1_write_block(sl->master, wrbuf, 4);
    msleep(W1_F2D_TPROG_MS);
    w1_reset_bus(sl->master);
    return 0;
}
```

### 5.5 Slave Lifecycle

```c
static int w1_f2d_add_slave(struct w1_slave *sl)
{
    return sysfs_create_bin_file(&sl->dev.kobj, &w1_f2d_bin_attr);
}

static void w1_f2d_remove_slave(struct w1_slave *sl)
{
    sysfs_remove_bin_file(&sl->dev.kobj, &w1_f2d_bin_attr);
}
```

### 5.6 Family Registration

```c
static struct w1_family_ops w1_f2d_fops = {
    .add_slave    = w1_f2d_add_slave,
    .remove_slave = w1_f2d_remove_slave,
};
static struct w1_family w1_family_2d = {
    .fid  = W1_EEPROM_DS2431,
    .fops = &w1_f2d_fops,
};
static int __init w1_f2d_init(void) { return w1_register_family(&w1_family_2d); }
module_init(w1_f2d_init);
static void __exit w1_f2d_fini(void) { w1_unregister_family(&w1_family_2d); }
module_exit(w1_f2d_fini);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Bernhard Weirich <bernhard.weirich@riedel.net>");
MODULE_DESCRIPTION("w1 family 2d driver for DS2431, 1kb EEPROM");
```

---

## 6. Lesson 5: Anatomy of w1_gxe00.c (New Driver)

### 6.1 Key Differences from DS2431 Driver

| Parameter | DS2431 (GX2431) | GXE00 |
|-----------|-----------------|-------|
| Family code | 0x2D | **0x14** |
| Total size | 128 bytes | **32 bytes** |
| Pages | 4 x 32 bytes | **1 x 32 bytes** |
| Scratchpad | 8 bytes | 8 bytes (same) |
| Write time | 11ms | **10ms** |

### 6.2 Constants

```c
#define GXE00_EEPROM_SIZE       32
#define GXE00_PAGE_COUNT        1
#define GXE00_PAGE_SIZE         32

#define GXE00_READ_EEPROM       0xF0
#define GXE00_WRITE_SCRATCH     0x0F
#define GXE00_READ_SCRATCH      0xAA
#define GXE00_COPY_SCRATCH      0x55

#define GXE00_TPROG_MS          10
#define GXE00_READ_RETRIES      10
#define GXE00_READ_MAXLEN       8
```

### 6.3 Complete Driver (Summary)

The full driver is at `drivers/w1/slaves/w1_gxe00.c`. It follows the exact same pattern as `w1_ds2431.c`:

1. **Bound check**: `gxe00_fix_count()` — same logic, different size constant
2. **Read block**: `gxe00_readblock()` — proof read pattern, max 8 bytes
3. **Write**: `gxe00_write()` — scratchpad flow with 10ms program time
4. **Sysfs handlers**: `gxe00_read_bin()` / `gxe00_write_bin()` — with read-modify-write for partial writes
5. **Attribute**: `gxe00_bin_attr` — name "eeprom", size 32, mode S_IRUGO|S_IWUSR
6. **Registration**: family code `0x14`, `module_init(gxe00_init)`

### 6.4 Files to Create/Modify

```
source/linux-.../drivers/w1/slaves/w1_gxe00.c     # NEW - driver source
source/linux-.../drivers/w1/slaves/Kconfig          # ADD entry
source/linux-.../drivers/w1/slaves/Makefile         # ADD build rule
arch/arm/configs/am335x_rom3310_puhui_defconfig     # ADD CONFIG_W1_SLAVE_GXE00=y
```

### 6.5 Kconfig Entry

```kconfig
config W1_SLAVE_GXE00
    tristate "256-bit EEPROM family support (GXE00)"
    depends on W1
    help
      Say Y here if you want to use a 1-wire
      256-bit EEPROM family device (GXE00).
```

### 6.6 Makefile Rule

```makefile
obj-$(CONFIG_W1_SLAVE_GXE00)    += w1_gxe00.o
```

---

## 7. Lesson 6: The DS28E25 -- Why So Complex?

Your existing DS28E25 driver is **2333 lines** vs ~310 for DS2431/GXE00.

### Feature Comparison

| Feature | DS28E25 | DS2431/GXE00 |
|---------|---------|--------------|
| SHA-256 engine | Software SHA-256 (~400 lines) | Not needed |
| Challenge-response auth | Complex protocol | Not needed |
| Write authentication (MAC) | Every write needs a MAC | Not needed |
| Page protection | Per-page write/OTP protection | Not needed |
| Character device + ioctl | /dev/ds28e25, 20+ commands | sysfs only |
| Cover detection | Physical removal detection | Not needed |

### Architecture Comparison

**Simple EEPROM:**
```
Userspace -> sysfs eeprom -> read/write -> 1-Wire commands
```

**Complex Authenticator:**
```
Userspace -> ioctl(/dev/ds28e25) -> SHA-256 + 1-Wire -> response
```

For your new chips: **simple sysfs pattern** is all that's needed.

---

## 8. Lesson 7: Build System Integration

### 8.1 Kernel Build Flow

```
make menuconfig (cfg_kernel.sh)  ->  .config
                       |
      obj-$(CONFIG_W1_SLAVE_GXE00) += w1_gxe00.o
                    /               \
          CONFIG...=y               CONFIG...=m
          (built-in)                (module)
          obj-y += foo.o            obj-m += foo.o
          linked into vmlinux       standalone .ko
```

### 8.2 Build Sequence for This BSP

```bash
cd scripts/
source setenv.sh

# Configure
./cfg_kernel.sh am335x_rom3310_puhui_defconfig

# (Optional) Customize
make -C $KROOT menuconfig

# Build
./mk_kernel.sh
```

### 8.3 Defconfig Changes

```diff
 CONFIG_W1=y
 CONFIG_W1_MASTER_DS2484=y
 CONFIG_W1_SLAVE_DS28E25=y
+# CONFIG_W1_SLAVE_DS2431 was not set  ->  change to:
+CONFIG_W1_SLAVE_DS2431=y
+CONFIG_W1_SLAVE_GXE00=y
```

---

## 9. Lesson 8: Testing & Debugging

### 9.1 On-Target Testing

```bash
# Check devices
ls /sys/bus/w1/devices/
# Expected: 14-XXXXXXXXXXXX  (GXE00)
#           2d-XXXXXXXXXXXX  (GX2431)
#           47-XXXXXXXXXXXX  (DS28E25)

# Read EEPROM
cat /sys/bus/w1/devices/14-*/eeprom | hexdump -C
cat /sys/bus/w1/devices/2d-*/eeprom | hexdump -C

# Write to GXE00
echo -ne '\xDE\xAD\xBE\xEF' > /sys/bus/w1/devices/14-*/eeprom

# Verify
cat /sys/bus/w1/devices/14-*/eeprom | hexdump -C
```

### 9.2 Kernel Logs

```bash
dmesg | grep -i "w1\|gxe00\|ds2431\|ds28e25\|ds2484"
```

### 9.3 Common Issues

| Symptom | Likely Cause |
|---------|-------------|
| No 14-* or 2d-* devices | GXE00 not powered, DS2484 not init, driver not built |
| Read returns 0xFF | Blank chip, wrong family code, timing issue |
| Write doesn't persist | msleep too short, missing w1_reset_bus(), page protected |

### 9.4 Userspace Tools

```bash
hexdump -C /sys/bus/w1/devices/14-*/eeprom
echo -ne '\x01\x02\x03\x04' > /sys/bus/w1/devices/14-*/eeprom
dd if=/sys/bus/w1/devices/14-*/eeprom bs=1 count=1 | hexdump
dmesg -w
```

---

## 10. Reference: Quick Card

### W1 IO Primitives

```c
w1_reset_bus(master);              // Reset all devices
w1_reset_select_slave(slave);      // Reset + select specific device
w1_write_8(master, byte);          // Write 1 byte
w1_write_block(master, buf, n);    // Write n bytes
w1_read_8(master);                 // Read 1 byte
w1_read_block(master, buf, n);     // Read n bytes
```

### Standard 1-Wire EEPROM Commands

| Byte | Command | Direction |
|------|---------|-----------|
| 0xF0 | Read EEPROM | Master -> Device |
| 0x0F | Write Scratchpad | Master -> Device |
| 0xAA | Read Scratchpad | Device -> Master |
| 0x55 | Copy Scratchpad | Master -> Device |
| 0xCC | Skip ROM (broadcast) | Master -> All |
| 0x55 | Match ROM (select one) | Master -> One |

### Driver Template

```c
// 1. Constants
#define CHIP_EEPROM_SIZE    XX
// 2. Bound check
static inline size_t chip_fix_count(...) { ... }
// 3. Read block (proof read)
static int chip_readblock(...) { ... }
// 4. Write (scratchpad flow)
static int chip_write(...) { ... }
// 5. Sysfs handlers
static ssize_t chip_read_bin(...) { ... }
static ssize_t chip_write_bin(...) { ... }
static BIN_ATTR(chip_bin_attr, ..., CHIP_EEPROM_SIZE);
// 6. Slave lifecycle
static int chip_add_slave(struct w1_slave *sl) { ... }
static void chip_remove_slave(...) { ... }
// 7. Family registration
static struct w1_family chip_family = { .fid = 0xNN, ... };
module_init(chip_init);
module_exit(chip_fini);
```

### Linux Driver Error Codes

| Return | Meaning |
|--------|---------|
| 0 | Success |
| -EIO | I/O error |
| -ENOMEM | Out of memory |
| -ENODEV | No such device |
| -EINVAL | Invalid argument |
| -EFAULT | Bad address |
| -EPERM | Operation not permitted |

### Paths in This BSP

```
Kernel source:       source/linux-3.2.0-psp04.06.00.11/
Defconfig:           arch/arm/configs/am335x_rom3310_puhui_defconfig
Board header:        arch/arm/mach-omap2/board-rom3310.h
W1 core:             drivers/w1/w1.h, w1_family.h, w1_int.h
W1 master:           drivers/w1/masters/ds2484.c
W1 DS28E25 slave:    drivers/w1/slaves/w1_ds28e25.c (+sha256)
W1 DS2431 slave:     drivers/w1/slaves/w1_ds2431.c (needs enable)
W1 GXE00 slave:      drivers/w1/slaves/w1_gxe00.c (write this)
W1 slave Kconfig:    drivers/w1/slaves/Kconfig
W1 slave Makefile:   drivers/w1/slaves/Makefile
```

---

## Exercises

### Exercise 1: Trace the Flow

Read `w1_ds2431.c` and answer:
1. How many 1-Wire bus transactions for a single 8-byte read at offset 0?
2. What happens if the proof read fails 10 times?
3. What does `es = (addr + len - 1) % W1_F2D_SCRATCH_SIZE` calculate?

<details>
<summary>Answers</summary>
1. 2 transactions (first + proof read). 2 resets, 2 selects, 2 command writes, 2 data reads.
2. Returns -1, kernel error message. Sysfs handler returns count = -EIO.
3. End-of-Scratchpad marker -- byte offset within 8-byte scratchpad where write ends.
</details>

### Exercise 2: Compare the Two Drivers

1. Why is GXE00_READ_MAXLEN also 8 (same as DS2431)?
2. Why doesn't GXE00 have PAGE_COUNT, PAGE_BITS, PAGE_MASK, SCRATCH_MASK?

<details>
<summary>Answers</summary>
1. Both use 8-byte scratchpad. Max read length matches scratchpad size.
2. GXE00 only has 1 page (the whole 32-byte chip), so page-level concepts aren't needed.
</details>

### Exercise 3: Kconfig Dependency Chain

What happens if you add CONFIG_W1_SLAVE_GXE00=y but CONFIG_W1=n?

<details>
<summary>Answer</summary>
Kconfig dependency (`depends on W1`) prevents it. `make oldconfig` silently disables it.
Always verify W1 master is enabled first.
</details>

### Exercise 4: Modify the Code

Change the GXE00 driver to expose two sysfs files: `eeprom` (first 16 bytes) and `user` (last 16 bytes).

<details>
<summary>Hint</summary>
Two bin_attribute structs with different size fields and read/write handlers
that offset into the EEPROM address space at different fixed offsets.
</details>

---

*End of training document. For next steps, see the main CLAUDE.md in the kernel source directory.*
