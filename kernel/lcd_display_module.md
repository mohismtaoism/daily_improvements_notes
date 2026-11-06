# The Light Path
## A Book about the LCD Display Module of the ROM-3310 / AM335x

> *From the moment a pixel is written into a memory buffer, to the moment a photon
> reaches your eye — this is the story of every one of them.*

---

## Prologue — How to read this book

This book describes one subsystem of a single embedded board: the **LCD display
module** of the **AM3358-based ROM-3310 COM Express module** on its Puhui carrier
board. It follows a strict physical order: hardware first, then the board support
code that wires the hardware to the kernel, then the framebuffer driver that turns
memory into a picture, then the light itself (backlight and the CH7055 bridge),
and finally the very first picture a user ever sees — the boot logo.

Every chapter ends with a "Reading the code" section that points at the exact
source file, function, and line so you can hold this book in one hand and the
source tree in the other.

The code under discussion lives in:

```
source/linux-3.2.0-psp04.06.00.11/
├── arch/arm/mach-omap2/
│   ├── board-advantech.c       # board file: registers the LCDC + backlight
│   ├── board-rom3310.h         # ROM-3310 pin mux, panel type, clock select
│   ├── devices.c               # am33xx_register_lcdc() / am33xx_register_ecap()
│   ├── clock33xx_data.c        # the display PLL and LCD clock tree
│   └── omap_hwmod_33xx_data.c  # LCDC hardware module (address, IRQ, clock)
├── drivers/video/
│   ├── da8xx-fb.c              # the framebuffer driver (the heart of this book)
│   ├── ch7055.c                # Chrontel CH7055 panel-interface bridge
│   ├── backlight/pwm_bl.c      # generic PWM backlight driver
│   ├── backlight/tlc59108.c    # TI TLC59108 I2C LED backlight driver
│   └── logo/                   # boot logo (PPM source → generated C)
├── include/video/da8xx-fb.h    # the driver's public structures
└── arch/arm/configs/am335x_rom3310_puhui_defconfig
```

> **Terminology note.** "LCDC" means the **LCD Controller** IP block inside the
> AM3358 SoC (its registers live at `0x4830_E000`). "DA8xx" is the name of the
> older DaVinci SoC family that first carried this controller; the AM335x carries
> the **second revision** of the same IP, and the Linux driver keeps the DA8xx
> name because it serves both. So when you read `da8xx-fb.c`, you are reading the
> AM335x LCDC driver.

---

# Part I — The Hardware

## Chapter 1 · The AM3358 LCD Controller

### 1.1 What an LCD controller does

A CPU is a terrible pixel-painter. It is excellent at *deciding* what should be
drawn, but it has better things to do than toggling 24 signal lines a million
times per second. So the AM3358 hands the actual drawing to a dedicated piece of
silicon: the **LCDC**.

The LCDC's job, stated plainly, is:

> *Read a rectangle of memory (the framebuffer), row by row, left to right, and
> push each pixel's red/green/blue values out of the chip one at a time, in lock
> step with a clock and two "sync" pulses that tell the panel where each line and
> each frame begins.*

It is a **pure DMA + raster engine**. The CPU only:

1. tells the LCDC where the framebuffer is (base + ceiling addresses),
2. programs the timing (resolution, porches, sync widths, pixel clock),
3. then lets the LCDC run autonomously, raising an interrupt at the end of each
   frame.

The CPU is then free to write new pixels into the *next* buffer while the LCDC is
still draining the *current* one — a technique called **double buffering**, which
is why the LCDC has two DMA channels (channel 0 and channel 1).

### 1.2 Where the LCDC lives in the SoC

From `arch/arm/mach-omap2/omap_hwmod_33xx_data.c`:

```
address range  0x4830_E000 .. 0x4830_FFFF   (8 KiB, L3 interconnect)
MPU IRQ        36
clock          lcdc_fck            (clock domain lcdc_clkdm)
module mode    MODULEMODE_SWCTRL   (software controls the module clock)
```

The LCDC is a master on the L3 interconnect: it can DMA straight out of SDRAM
without involving the CPU. Its OCP interface is declared as:

```c
struct omap_hwmod_addr_space am33xx_lcdc_addr_space[] = {
    {
        .pa_start   = 0x4830E000,
        .pa_end     = 0x4830E000 + SZ_8K - 1,
        .flags      = ADDR_MAP_ON_INIT | ADDR_TYPE_RT,
    },
    { }
};
```

The IRQ line is 36:

```c
static struct omap_hwmod_irq_info am33xx_lcdc_irqs[] = {
    { .irq = 36 },
    { .irq = -1 }
};
```

The module's system-config block (`SYSC`) at offset `0x54` supports idle modes
(force / no / smart), which the PM runtime framework uses to cut the module clock
when the display is off.

### 1.3 The raster engine and the panel interface

The LCDC's output is a set of **parallel signals** that leave the chip through the
`lcd_*` and (on this board) some remapped `gpmc_*` pins:

| Signal | Direction | Purpose |
|--------|-----------|---------|
| `LCD_PCLK`  | out | pixel clock — one new pixel per rising/falling edge |
| `LCD_HSYNC` | out | horizontal sync — marks the start of each scan line |
| `LCD_VSYNC` | out | vertical sync — marks the start of each frame |
| `LCD_AC_BIAS_EN` | out | alternating-current bias for passive panels |
| `LCD_DATA[23:0]` | out | 24-bit parallel RGB (8 bits per color) |

The panel controller you will meet in Chapter 2 — the **CH7055** — sits between
these pins and the actual glass panel.

### 1.4 The clock that makes pixels

A pixel is pushed out on every tick of the **pixel clock**. The pixel clock is not
a crystal; it is derived from a PLL and divided down inside the LCDC. The chain is:

```
sys_clkin (crystal, e.g. 24 MHz)
   └─ dpll_disp_ck         Display PLL  → set to 260 MHz by the board code
        └─ dpll_disp_m2_ck  /M2 divider (DPLL_CLKOUT)
             └─ disp_pll_clk
                  └─ lcd_gclk          (clksel: also ALT_CLK / PER_CLKOUTM2)
                       └─ lcdc_fck      the LCDC module clock (clk_get_rate)
                            └─ /CLK_DIVISOR  (inside the LCDC, LCD_CTRL_REG)
                                 └─ pixel clock → LCD_PCLK
```

The integer divider inside the LCDC is chosen so that `lcdc_fck / divider` lands
as close as possible to the panel's requested pixel clock. We will do this math in
Chapter 8. For now, note the important consequence of an **integer** divider:

> *If the panel asks for 40 MHz and the LCDC clock is 260 MHz, the divider is
> `260 / 40 = 6` (integer truncation), giving an actual pixel clock of
> `260 / 6 ≈ 43.3 MHz` — not exactly 40 MHz. The LCDC cannot make a perfect 40 MHz
> from 260 MHz, and the driver has to live with the quantization. This single
> fact explains a "+1" hack you will see in the driver, and why the reported
> frame rate is higher than the panel's nominal 60 Hz.*

### 1.5 The register map

From the `#define`s at the top of `drivers/video/da8xx-fb.c`:

| Offset | Name | What it controls |
|--------|------|------------------|
| 0x00 | `LCD_PID_REG` | Peripheral identification — the driver reads this to learn the IP revision |
| 0x04 | `LCD_CTRL_REG` | Clock divider, raster mode, clock enables |
| 0x08 | `LCD_STAT_REG` | Status (revision 1) |
| 0x28 | `LCD_RASTER_CTRL_REG` | Master switch: raster enable, TFT/passive mode, FDD, palette load mode |
| 0x2C | `LCD_RASTER_TIMING_0_REG` | Horizontal: pixels-per-line, hsync back/front porch + pulse width |
| 0x30 | `LCD_RASTER_TIMING_1_REG` | Vertical: lines-per-panel, vsync back/front porch + pulse width |
| 0x34 | `LCD_RASTER_TIMING_2_REG` | Clock polarity, AC bias, 24-bit mode, extra vertical bit |
| 0x40 | `LCD_DMA_CTRL_REG` | DMA burst size, FIFO threshold, double-buffer enable |
| 0x44 | `LCD_DMA_FRM_BUF_BASE_ADDR_0_REG` | Channel-0 DMA start address |
| 0x48 | `LCD_DMA_FRM_BUF_CEILING_ADDR_0_REG` | Channel-0 DMA end address |
| 0x4C | `LCD_DMA_FRM_BUF_BASE_ADDR_1_REG` | Channel-1 DMA start address |
| 0x50 | `LCD_DMA_FRM_BUF_CEILING_ADDR_1_REG` | Channel-1 DMA end address |
| 0x58 | `LCD_RAW_STAT_REG` | Raw interrupt status (revision 2) |
| 0x5C | `LCD_MASKED_STAT_REG` | Masked interrupt status (revision 2) |
| 0x60 | `LCD_INT_ENABLE_SET_REG` | Interrupt enable set (revision 2) |
| 0x64 | `LCD_INT_ENABLE_CLR_REG` | Interrupt enable clear (revision 2) |
| 0x68 | `LCD_END_OF_INT_IND_REG` | End-of-interrupt indicator |
| 0x6C | `LCD_CLK_ENABLE_REG` | DMA / LIDD / core clock enables (revision 2) |
| 0x70 | `LCD_CLK_RESET_REG` | Main reset bit (revision 2) |

The registers at offsets 0x44..0x50 are the reason the LCDC can do **double
buffering without a single CPU intervention**: channel 0 drains buffer 0 while
channel 1 drains buffer 1, and an end-of-frame interrupt tells the driver which
channel just finished so it can reprogram that channel's addresses for the next
swap.

**Reading the code:** `da8xx-fb.c:105-139` — the register and flag `#define`s
are the map you will want to keep open while reading Chapters 6-9.

---

## Chapter 2 · The board: ROM-3310 on the Puhui carrier

### 2.1 The physical data path

On the ROM-3310 + Puhui carrier, the LCD path is:

```
AM3358 LCDC                     CH7055 (I2C 0x76)          Panel glass
┌──────────────┐ 24-bit RGB ┌──────────────────┐  LVDS / ┌──────────┐
│ lcd_data0..15 │──────────▶│  Chrontel CH7055  │──▶ RGB? │  e.g.    │
│ gpmc_ad8..15  │            │  interface bridge │         │ 800×480  │
│ hsync/vsync   │            │  + brightness     │         │          │
│ pclk          │            └──────────────────┘         └──────────┘
└──────────────┘
       ▲
       │ power + backlight enable (GPIOs)
       │
   gpio3_3  = LCD_VDD_EN
   gpio3_16 = LCD_BKLT_EN
   gpio0_17 = VGA_LIGHT_EN
```

The CH7055 is a small I2C chip on bus **I2C0 at address `0x76`**. In this design
it performs two jobs: it acts as the **interface between the LCDC's parallel RGB
and the panel**, and it carries a **brightness** register that the kernel exposes
through sysfs. The board provides no dedicated backlight inverter for the CH7055
path; brightness is programmed directly into the chip's registers (Chapter 11).

The backlight on this design has a second, independent control: a **PWM signal**
generated by the ECAP (Enhanced Capture) peripheral, which feeds the panel's
backlight input. We will meet it in Chapter 10.

### 2.2 Pin muxing the 24-bit RGB bus

The AM3358 pins are highly multiplexed — one physical ball can serve many
functions. The board code must select the LCDC function for each pin. This lives
in `arch/arm/mach-omap2/board-rom3310.h`, in the `lcdc_pin_mux[]` table.

The lower 16 data lines use their native `lcd_data` pins in **mode 0**:

```c
{ "lcd_data0.lcd_data0",   OMAP_MUX_MODE0 | AM33XX_PIN_OUTPUT | AM33XX_PULL_DISA },
{ "lcd_data1.lcd_data1",   OMAP_MUX_MODE0 | AM33XX_PIN_OUTPUT | AM33XX_PULL_DISA },
   ... down to lcd_data15 ...
```

The upper 8 bits (`lcd_data16..23`) are *not* available on native LCD pins on this
SoC; they are remapped onto **GPMC address/data balls** in **mode 1**:

```c
{ "gpmc_ad15.lcd_data16", OMAP_MUX_MODE1 | AM33XX_PIN_OUTPUT },
{ "gpmc_ad14.lcd_data17", OMAP_MUX_MODE1 | AM33XX_PIN_OUTPUT },
   ... (ad13..ad10) ...
{ "gpmc_ad9.lcd_data22",  OMAP_MUX_MODE1 | AM33XX_PIN_OUTPUT },
{ "gpmc_ad8.lcd_data23",  OMAP_MUX_MODE1 | AM33XX_PIN_OUTPUT },
```

(If you ever wonder "why is my framebuffer only showing 16-bit color?", the first
thing to check is this table — every one of these 8 remapped lines must be present
and in mode 1.)

The control signals are their native functions, mode 0:

```c
{ "lcd_vsync.lcd_vsync",         OMAP_MUX_MODE0 | AM33XX_PIN_OUTPUT },
{ "lcd_hsync.lcd_hsync",         OMAP_MUX_MODE0 | AM33XX_PIN_OUTPUT },
{ "lcd_pclk.lcd_pclk",           OMAP_MUX_MODE0 | AM33XX_PIN_OUTPUT },
{ "lcd_ac_bias_en.lcd_ac_bias_en", OMAP_MUX_MODE0 | AM33XX_PIN_OUTPUT },
```

### 2.3 The power and backlight GPIOs

The table contains a long history of commented-out attempts (each line stamped
with `// modified by zhangjian`) showing that this design went through several
iterations of which GPIO powers what. The **active** configuration today is:

```c
{ "mii1_txen.gpio3_3", OMAP_MUX_MODE7 | AM33XX_PIN_OUTPUT_PULLUP }, /* LCD_VDD_EN */
```

That is the only power GPIO that is muxed in the table. The actual drive happens
later, in `lcdc_init()` in `board-advantech.c`:

```c
gpio_request(GPIO_TO_PIN(3, 3),  "LCD_VDD_EN");
gpio_direction_output(GPIO_TO_PIN(3, 3), 1);   /* panel power on */

gpio_request(GPIO_TO_PIN(3, 16), "LCD_BKLT_EN");
gpio_direction_output(GPIO_TO_PIN(3, 16), 1);  /* backlight enable on */

gpio_request(GPIO_TO_PIN(0, 17), "VGA_LIGHT_EN");
gpio_direction_output(GPIO_TO_PIN(0, 17), 1);
```

Three GPIOs, all driven high at initialization:

- **`gpio3_3` / `LCD_VDD_EN`** — panel power.
- **`gpio3_16` / `LCD_BKLT_EN`** — backlight enable.
- **`gpio0_17` / `VGA_LIGHT_EN`** — an auxiliary light/enable (VGA-related).

Because they are driven *before* the LCDC is registered, the panel is powered and
lit before the framebuffer even exists — which is exactly what you want at boot:
by the time the console appears, the hardware is already showing a picture.

**Reading the code:** `board-rom3310.h` — `lcdc_pin_mux[]` (around line 30);
`board-advantech.c:718` — `lcdc_init()`, the GPIO section at the top.

---

## Chapter 3 · The clock tree, precisely

### 3.1 The display PLL

The board code must bring the Display PLL to a frequency from which a usable
pixel clock can be divided. The board decides the PLL frequency from the panel
type (Chapter 5). For the default panel it is **260 MHz**.

The PLL node (`arch/arm/mach-omap2/clock33xx_data.c`) is a classic AM33xx
"non-core" DPLL with a M2 output divider:

```c
static struct dpll_data dpll_disp_dd = {
    .mult_div1_reg  = AM33XX_CM_CLKSEL_DPLL_DISP,
    .control_reg    = AM33XX_CM_CLKMODE_DPLL_DISP,
    .idlest_reg     = AM33XX_CM_IDLEST_DPLL_DISP,
    .mult_mask      = AM33XX_DPLL_MULT_MASK,
    .div1_mask      = AM33XX_DPLL_DIV_MASK,
    ...
};

static struct clk dpll_disp_ck = {
    .name      = "dpll_disp_ck",
    .parent    = &sys_clkin_ck,
    .dpll_data = &dpll_disp_dd,
    .set_rate  = &omap3_noncore_dpll_set_rate,
    ...
};

static struct clk dpll_disp_m2_ck = {
    .name        = "dpll_disp_m2_ck",
    .parent      = &dpll_disp_ck,
    .clksel_reg  = AM33XX_CM_DIV_M2_DPLL_DISP,
    ...
};
```

The M2 divider (`dpll_disp_m2_ck`) feeds `disp_pll_clk`, which is one of the
inputs to `lcd_gclk` (the pixel-clock mux). `lcdc_fck` then *follows* `lcd_gclk`:

```c
static struct clk lcdc_fck = {
    .name      = "lcdc_fck",
    .clkdm_name= "lcdc_clkdm",
    .parent    = &lcd_gclk,
    .enable_reg= AM33XX_CM_PER_LCDC_CLKCTRL,
    .enable_bit= AM33XX_MODULEMODE_SWCTRL,
    .recalc    = &followparent_recalc,
};
```

So `clk_get_rate(lcdc_fck)` returns whatever `disp_pll_clk` is after the M2
divider — the number the framebuffer driver will divide further inside the LCDC.

### 3.2 Who sets the PLL to 260 MHz

In `board-advantech.c`:

```c
static int __init conf_disp_pll(int rate)
{
    struct clk *disp_pll;
    disp_pll = clk_get(NULL, "dpll_disp_ck");
    ...
    ret = clk_set_rate(disp_pll, rate);
    ...
}
```

And the rate comes from the board's panel-type logic (`board-rom3310.h`):

```c
unsigned long panel_disp_clock_select(void)
{
    unsigned long disp_clock = 0;
    if (panel_type == 6)      disp_clock = 280000000;
    else if (panel_type == 7) disp_clock = 260000000;
    else if (panel_type == 8) disp_clock = 256500000;
    else                      disp_clock = 260000000;   /* default, panel_type 9 */
    return disp_clock;
}
```

So the default 800×480 panel runs the display PLL at 260 MHz; the panel's 40 MHz
request therefore needs a divider of 6 inside the LCDC, and the *actual* pixel
clock becomes 260/6 ≈ 43.3 MHz (see §1.4).

**Reading the code:** `clock33xx_data.c:397-473` (PLL + M2 + gclk + lcdc_fck);
`board-advantech.c:584` (`conf_disp_pll`); `board-rom3310.h:605`
(`panel_disp_clock_select`).

---

## Chapter 4 · What the board *tells* the kernel about the panel

### 4.1 The panel description structs

Every panel the kernel can drive is described by three nested structures in
`include/video/da8xx-fb.h`:

```c
struct display_panel {          /* the panel's electrical character */
    enum panel_type   panel_type;
    int               max_bpp;
    int               min_bpp;
    enum panel_shade  panel_shade;    /* COLOR_ACTIVE / COLOR_PASSIVE / MONOCHROME */
};

struct da8xx_lcdc_platform_data {   /* what the board passes to the driver */
    const char manu_name[10];
    void *controller_data;          /* → &lcd_cfg */
    const char type[25];            /* → "INNOLUX_VGA_800x480@60" */
    void (*panel_power_ctrl)(int);
    int  (*get_context_loss_count)(struct device *);
};

struct lcd_ctrl_config {            /* how the controller should drive it */
    const struct display_panel *p_disp_panel;
    int ac_bias;                    /* AC bias frequency */
    int ac_bias_intrpt;             /* transitions per interrupt */
    int dma_burst_sz;               /* 16 (16×32-bit words) */
    int bpp;                        /* 32 */
    int fdd;                        /* 0x80 — FIFO DMA request delay */
    int tft_alt_mode, stn_565_mode, mono_8bit_mode;
    int invert_line_clock;          /* 1 */
    int invert_frm_clock;           /* 1 */
    int sync_edge, sync_ctrl;       /* 0, 1 */
    int raster_order;               /* 0 */
};
```

The board's single instance of the display-panel description (in
`board-advantech.c`) is:

```c
static const struct display_panel disp_panel = {
    WVGA,          /* panel_type enum */
    32,            /* max_bpp */
    32,            /* min_bpp */
    COLOR_ACTIVE,  /* it is a color TFT */
};

static struct lcd_ctrl_config lcd_cfg = {
    &disp_panel,
    .ac_bias          = 255,
    .ac_bias_intrpt   = 0,
    .dma_burst_sz     = 16,
    .bpp              = 32,
    .fdd              = 0x80,
    .invert_line_clock = 1,
    .invert_frm_clock  = 1,
    .sync_ctrl         = 1,
};
```

Note `bpp = 32`: the framebuffer is **32 bits per pixel (XRGB8888)** in memory.
The 24-bit data bus carries only the RGB bytes; the extra 8 bits are a pad byte
that the LCDC simply ignores (more in Chapter 8).

### 4.2 The per-panel platform data blocks

For each named panel there is a `da8xx_lcdc_platform_data` block. The one that
matters for this board is:

```c
struct da8xx_lcdc_platform_data INNOLUX_VGA_800x480_pdata = {
    .manu_name       = "INNOLUX",
    .controller_data = &lcd_cfg,
    .type            = "INNOLUX_VGA_800x480@60",
};
```

The driver matches this `type` string against its own table of known panels
(Chapter 6). It is the *string* that binds board code to driver code.

### 4.3 panel_type and the kernel command line

The board keeps a module-global `panel_type` integer. Its default — and the value
for the ROM-3310 Puhui carrier — is **9**:

```c
int panel_type = 9;
#define PANEL_TYPE_DEFAULT 7
```

`panel_setup()` parses the kernel command line (`video_mode=...` on ROM-3310,
`lvds_panel=...` on other carriers) and maps a name to the number:

```c
static int __init panel_setup(char *panel)
{
    if      (!strcmp(panel, "VGA_800x480@60"))   panel_type = 9;
    else if (!strcmp(panel, "VGA_1024x768@60")) panel_type = 7;
    else if (!strcmp(panel, "VGA_1360x768@60")) panel_type = 8;
    ...
    else                                        panel_type = PANEL_TYPE_DEFAULT;
    return 1;
}

__setup("video_mode=", panel_setup);
```

So you can select a panel at boot without rebuilding:

```
video_mode=VGA_1024x768@60
```

**Reading the code:** `board-advantech.c:90-190` (structs + pdata blocks),
`board-advantech.c:602-651` (`panel_setup`), `board-rom3310.h` (`panel_type`).

---

# Part II — The Board Support Code

## Chapter 5 · Wiring it together: `lcdc_init()` and friends

### 5.1 The board init table

`board-advantech.c` declares a table of `(function, where, profile)` tuples. One
of them is `lcdc_init`:

```c
{ lcdc_init, DEV_ON_BASEBOARD, PROFILE_ALL },
```

During kernel init, the board machinery calls `lcdc_init()` once per match. What
it does, in order:

### 5.2 `lcdc_init()` step by step

```c
static void lcdc_init(int evm_id, int profile)
{
    struct da8xx_lcdc_platform_data *lcdc_pdata;
    unsigned long disp_clock = 0;

    setup_pin_mux(lcdc_pin_mux);              /* 1. mux the 24-bit RGB + syncs */

    /* 2. drive the power / backlight GPIOs high (Chapter 2) */
    gpio_request(GPIO_TO_PIN(3, 3), "LCD_VDD_EN");
    gpio_direction_output(GPIO_TO_PIN(3, 3), 1);
    gpio_request(GPIO_TO_PIN(3, 16), "LCD_BKLT_EN");
    gpio_direction_output(GPIO_TO_PIN(3, 16), 1);
    gpio_request(GPIO_TO_PIN(0, 17), "VGA_LIGHT_EN");
    gpio_direction_output(GPIO_TO_PIN(0, 17), 1);

    lcdc_pin_function_init();                 /* 3. (empty stub on this board) */

    disp_clock = panel_disp_clock_select();   /* 4. pick PLL rate from panel_type */

    if (conf_disp_pll(disp_clock)) {          /* 5. program the display PLL */
        pr_info("Failed configure display PLL, not attempting to register LCDC\n");
        return;
    }

    /* 6. pick the platform data block for this panel_type */
    switch (evm_id) {
    case GEN_PURP_EVM:
    case GEN_PURP_DDR3_EVM:
        if      (panel_type == 0) lcdc_pdata = &TFC_S9700RTWV35TR_01B_pdata;
        ...
        else if (panel_type == 9) {           /* ← default on ROM-3310 */
            lcdc_pdata = &INNOLUX_VGA_800x480_pdata;
            pr_info("lcdc_init  INNOLUX_VGA_800x480\n");
        }
        else if (panel_type == 10) lcdc_pdata = &HX8282_A11_1024x600_pdata;
        break;
    ...
    }

    lcdc_pdata->get_context_loss_count = omap_pm_get_dev_context_loss_count;

    /* 7. create the platform device that the framebuffer driver will probe */
    if (am33xx_register_lcdc(lcdc_pdata))
        pr_info("Failed to register LCDC device\n");
}
```

The three empty functions — `lcdc_pin_function_init()`, `lcd_vdd_config()`,
`lcd_bkl_config()` — exist in the header as hooks that other carriers fill in;
on the ROM-3310 Puhui board the work is done directly in `lcdc_init`.

### 5.3 `am33xx_register_lcdc()` — the handoff to the driver

In `arch/arm/mach-omap2/devices.c`, this function turns the platform data into a
real platform device using the SoC's hwmod infrastructure:

```c
int __init am33xx_register_lcdc(struct da8xx_lcdc_platform_data *pdata)
{
    struct omap_hwmod *oh;
    struct platform_device *pdev;

    oh = omap_hwmod_lookup("lcdc");          /* find the LCDC hwmod (Ch. 1.2) */
    if (!oh) { ... return -ENODEV; }

    pdev = omap_device_build("da8xx_lcdc", 0, oh, pdata,
            sizeof(struct da8xx_lcdc_platform_data), NULL, 0, 0);
    ...
}
```

This is the crucial handoff:

> The board file calls `am33xx_register_lcdc()` with a `da8xx_lcdc_platform_data`.
> That becomes **platform device `"da8xx_lcdc"` with id 0**. The framebuffer
> driver, whose `platform_driver.name` is `"da8xx_lcdc"` (Chapter 6), will match
> it and its `probe()` runs with the same platform data.

The clock lookup in the driver works because `clock33xx_data.c` maps the device
name `"da8xx_lcdc.0"` to `lcdc_fck`:

```c
CLK("da8xx_lcdc.0", NULL, &lcdc_fck, CK_AM33XX),
```

### 5.4 The backlight, registered later

Backlight is registered by a **late_initcall** so that it happens after the LCDC
is up:

```c
static int __init backlight_init(void)
{
    if (backlight_enable) {
        int ecap_index = 2;          /* CONFIG_MACH_ADVANTECH → ecap.2 */

        lcd_vdd_config(1);            /* power on (stub on this board) */

        am33xx_register_ecap(ecap_index, &pwm_pdata[ecap_index]);
        platform_device_register(&am335x_backlight);

        mdelay(15);
        lcd_bkl_config(1);            /* backlight on (stub) */
    }
}
late_initcall(backlight_init);
```

`am33xx_register_ecap()` is the ECAP twin of `am33xx_register_lcdc()`: it builds
platform device `"ecap.2"` (via `ecap.2` hwmod). The backlight platform device is
named `"pwm-backlight"` — matching `pwm_bl.c` (Chapter 10) — and carries
`am335x_backlight_data2`, which points at PWM `"ecap.2"`.

**Reading the code:** `board-advantech.c:718` (`lcdc_init`), `:706`
(`lcdc_uninit`), `:493-582` (`backlight_init` + pdata), `:584` (`conf_disp_pll`);
`devices.c:175` (`am33xx_register_lcdc`), `devices.c:582`
(`am33xx_register_ecap`).

---

# Part III — The Framebuffer Driver

## Chapter 6 · `fb_probe()` — waking the hardware

### 6.1 The platform driver and its match

At the bottom of `da8xx-fb.c`:

```c
static struct platform_driver da8xx_fb_driver = {
    .probe   = fb_probe,
    .remove  = fb_remove,
    .driver  = { .name = "da8xx_lcdc", ... },
    ...
};
```

This `"da8xx_lcdc"` string is exactly what `am33xx_register_lcdc()` built in
Chapter 5. When they meet, `fb_probe()` runs.

### 6.2 Probe, step by step

```c
static int __devinit fb_probe(struct platform_device *device)
{
    struct da8xx_lcdc_platform_data *fb_pdata = device->dev.platform_data;
    ...
    lcdc_regs = platform_get_resource(device, IORESOURCE_MEM, 0);
    ...
    da8xx_fb_reg_base = ioremap(lcdc_regs->start, len);   /* 0x4830E000 */

    fb_clk = clk_get(&device->dev, NULL);                  /* lcdc_fck */

    pm_runtime_enable(&device->dev);
    pm_runtime_get_sync(&device->dev);

    /* Determine the IP revision from the PID register */
    switch (lcdc_read(LCD_PID_REG)) {
    case 0x4C100102:    lcd_revision = LCD_VERSION_1; break;   /* old DA8xx */
    case 0x4F200800:
    case 0x4F201000:    lcd_revision = LCD_VERSION_2; break;   /* AM335x */
    default:            lcd_revision = LCD_VERSION_1;
    }
    ...
}
```

The **PID register** is the driver's way of asking "which revision of this IP am
I talking to?" The AM3358 returns `0x4F200800` (or `0x4F201000`), selecting
`LCD_VERSION_2`. Almost every later code path branches on `lcd_revision`:

- revision 1 uses `LCD_STAT_REG`; revision 2 uses `LCD_RAW_STAT_REG` /
  `LCD_MASKED_STAT_REG` + a separate `LCD_INT_ENABLE_SET_REG`.
- revision 2 additionally needs `LCD_CLK_ENABLE_REG` (DMA/LIDD/core clock enables)
  and a reset sequence through `LCD_CLK_RESET_REG` when enabling the raster.
- the horizontal resolution field is wider on revision 2 (up to 2048 px).

### 6.3 Finding the panel by name

The driver holds a table of all panels it knows how to drive. Probe scans it,
comparing each entry's `name` to the `type` string the board supplied:

```c
for (i = 0, lcdc_info = known_lcd_panels;
     i < ARRAY_SIZE(known_lcd_panels);
     i++, lcdc_info++) {
    if (strcmp(fb_pdata->type, lcdc_info->name) == 0)
        break;
}
if (i == ARRAY_SIZE(known_lcd_panels)) {
    dev_err(&device->dev, "GLCD: No valid panel found\n");
    ...
}
```

If no name matches, the driver refuses to load — the string in
`board-advantech.c` and the string in `known_lcd_panels[]` must match *exactly*
(Chapter 6.5).

### 6.4 Allocating the framebuffer and palette

The driver computes how much memory a full screen needs, doubles it for double
buffering, and allocates it as **coherent DMA memory**:

```c
par->vram_size = lcdc_info->width * lcdc_info->height * lcd_cfg->bpp;
ulcm = lcm((lcdc_info->width * lcd_cfg->bpp)/8, PAGE_SIZE);
par->vram_size = roundup(par->vram_size/8, ulcm);
par->vram_size = par->vram_size * LCD_NUM_BUFFERS;    /* × 2 */

par->vram_virt = dma_alloc_coherent(NULL, par->vram_size,
                    (resource_size_t *)&par->vram_phys, GFP_KERNEL | GFP_DMA);
```

For 800×480×32bpp that is 1.5 MiB × 2 = **3 MiB** of contiguous DMA memory. The
physical address (`vram_phys`) is what the LCDC's DMA channels will be pointed at.
`LCD_NUM_BUFFERS = 2` — the two DMA channels of Chapter 1.

A 256-entry hardware palette is allocated too (needed for the 8-bit pseudo-color
mode; on 24/32-bit mode it is unused but harmless):

```c
par->v_palette_base = dma_alloc_coherent(NULL, PALETTE_SIZE, ...);
```

### 6.5 The panel table — `known_lcd_panels[]`

The full table (from `da8xx-fb.c:237-404`). Fields: `width`, `height`, the
horizontal/vertical sync geometry (`hfp`/`hbp`/`hsw`/`vfp`/`vbp`/`vsw`), the
requested pixel clock `pxl_clk`, and `invert_pxl_clk` (polarity).

| idx | name | W×H | pxl_clk | notes |
|-----|------|-----|---------|-------|
| 0 | `Sharp_LCD035Q3DG01` | 320×240 | 4.608 MHz | small QVGA |
| 1 | `Sharp_LK043T1DG01` | 480×272 | 7.83 MHz | |
| 2 | `TFC_S9700RTWV35TR_01B` | 800×480 | 33 MHz | Ryan's 7" panel timings under CONFIG_MACH_ADVANTECH |
| 3 | `NHD-4.3-ATXI#-T-1` | 480×272 | 9 MHz | Newhaven |
| 4 | `AUO_G185XW01_V1` | 1360×768 | 60 MHz | |
| 5 | `INNOLUX_G150XGE_L04` | 1024×768 | 57 MHz | |
| 6 | `INNOLUX_VGA_800x600@60` | 800×600 | 40 MHz | |
| 7 | `INNOLUX_VGA_1024x768@60` | 1024×768 | 65 MHz | |
| 8 | `INNOLUX_VGA_1360x768@60` | 1360×768 | 85.5 MHz | |
| **9** | **`INNOLUX_VGA_800x480@60`** | **800×480** | **40 MHz** | **default (ROM-3310)** |
| 10 | `HX8282_A11_1024x600` | 1024×600 | 51.2 MHz | |

The entry the ROM-3310 uses:

```c
[9] = {
    .name           = "INNOLUX_VGA_800x480@60",
    .width          = 800,
    .height         = 480,
    .hfp = 50, .hbp = 179, .hsw = 27,     /* horizontal: front/back porch, sync */
    .vfp = 1,  .vbp = 24,  .vsw = 3,      /* vertical                               */
    .pxl_clk        = 40000000,
    .invert_pxl_clk = 1,
},
```

The blanking numbers encode the panel's scan-line geometry: each line is
`800 active + 50 front + 179 back + 27 sync = 1056` pixels wide; each frame is
`480 + 1 + 24 + 3 = 508` lines tall. With the *actual* pixel clock of ≈43.3 MHz,
the real frame rate is:

```
frame_rate = 43.3 MHz / (1056 × 508) ≈ 80.7 Hz
```

which is why the `@60` in the name is best read as "nominal 60 Hz panel" — the
hardware runs a bit faster than the panel's design point (this is normal for this
family of timings; the LCDC simply cannot make a clean 40.0 MHz from 260 MHz).

`invert_pxl_clk = 1` means the panel latches data on the **falling** edge of the
pixel clock — a common requirement, handled by `LCD_INVERT_PIXEL_CLOCK` in
`LCD_RASTER_TIMING_2_REG` (Chapter 7).

### 6.6 Finishing probe

Probe then:

- copies the panel geometry into `da8xx_fb_var` (the "variable screen info"),
- computes `pixclock` (the reported pixel-clock period in picoseconds) via
  `da8xxfb_pixel_clk_period()` (Chapter 8.4),
- sets `da8xx_fb_fix` ("fixed screen info": base address, line length),
- calls `fb_set_var()` to make the console framework adopt these defaults,
- **registers the framebuffer** (`register_framebuffer`), creating `/dev/fb0`,
- and requests the IRQ with the right handler for the revision:

```c
if (lcd_revision == LCD_VERSION_1)
    lcdc_irq_handler = lcdc_irq_handler_rev01;
else
    lcdc_irq_handler = lcdc_irq_handler_rev02;

ret = request_irq(par->irq, lcdc_irq_handler, 0, DRIVER_NAME, par);
```

**Reading the code:** `da8xx-fb.c:1447` (`fb_probe`), `:237` (`known_lcd_panels`),
`:206` (`da8xx_fb_var`), `:223` (`da8xx_fb_fix`).

---

## Chapter 7 · Programming the raster engine

### 7.1 `lcd_init()` — the config in one breath

When a mode is applied (`da8xxfb_set_par` → `lcd_init`), the driver programs the
whole LCDC. `lcd_init()` (`da8xx-fb.c:846`) orchestrates the steps:

```c
static int lcd_init(struct da8xx_fb_par *par, const struct lcd_ctrl_config *cfg,
                    struct da8xx_panel *panel)
{
    /* 1. pixel clock divider */
    lcd_calc_clk_divider(par);

    /* 2. pixel clock polarity */
    if (panel->invert_pxl_clk)
        lcdc_write(lcdc_read(LCD_RASTER_TIMING_2_REG) |
                   LCD_INVERT_PIXEL_CLOCK, LCD_RASTER_TIMING_2_REG);
    else
        lcdc_write(lcdc_read(LCD_RASTER_TIMING_2_REG) &
                   ~LCD_INVERT_PIXEL_CLOCK, LCD_RASTER_TIMING_2_REG);

    /* 3. DMA burst size and FIFO threshold */
    ret = lcd_cfg_dma(cfg->dma_burst_sz, cfg->fifo_th);

    /* 4. AC bias */
    lcd_cfg_ac_bias(cfg->ac_bias, cfg->ac_bias_intrpt);

    /* 5. sync geometry */
    lcd_cfg_vertical_sync(panel->vbp, panel->vsw, panel->vfp);
    lcd_cfg_horizontal_sync(panel->hbp, panel->hsw, panel->hfp);

    /* 6. display type (TFT / passive / mono) + clock polarity */
    ret = lcd_cfg_display(cfg);

    /* 7. validate panel type + pick bpp */
    if ((QVGA != cfg->p_disp_panel->panel_type) &&
        (WVGA != cfg->p_disp_panel->panel_type))
        return -EINVAL;
    ...
    ret = lcd_cfg_frame_buffer(par, panel->width, panel->height, bpp, cfg->raster_order);

    /* 8. FIFO DMA request delay */
    lcdc_write((lcdc_read(LCD_RASTER_CTRL_REG) & 0xfff00fff) |
               (cfg->fdd << 12), LCD_RASTER_CTRL_REG);
    ...
}
```

We follow each step.

### 7.2 The pixel clock divider — `lcd_calc_clk_divider()`

```c
static void lcd_calc_clk_divider(struct da8xx_fb_par *par)
{
    unsigned int lcd_clk, div;
    lcd_clk = clk_get_rate(par->lcdc_clk);      /* lcdc_fck, ≈ 260 MHz */
    div = lcd_clk / par->pxl_clk;               /* 260 / 40 = 6          */

    /* Configure the LCD clock divisor */
    lcdc_write(LCD_CLK_DIVISOR(div) |
               (LCD_RASTER_MODE & 0x1), LCD_CTRL_REG);

    if (lcd_revision == LCD_VERSION_2)
        lcdc_write(LCD_V2_DMA_CLK_EN | LCD_V2_LIDD_CLK_EN |
                   LCD_V2_CORE_CLK_EN, LCD_CLK_ENABLE_REG);
}
```

- `LCD_CLK_DIVISOR(div)` places `div` in bits 8.. of `LCD_CTRL_REG`.
- `LCD_RASTER_MODE` selects the raster (as opposed to LIDD) interface.
- On revision 2 the three internal clocks (DMA, LIDD, core) must be explicitly
  enabled — an AM335x-specific quirk.

### 7.3 DMA burst and FIFO — `lcd_cfg_dma()`

```c
switch (burst_size) {           /* cfg->dma_burst_sz = 16 */
case 16:  reg |= LCD_DMA_BURST_SIZE(LCD_DMA_BURST_16); break;   /* 16 × 32-bit */
...
}
reg |= (fifo_th << 8);          /* FIFO threshold */
lcdc_write(reg, LCD_DMA_CTRL_REG);
```

A burst of 16 × 32-bit words (64 bytes) per request, with the FIFO threshold at
`fifo_th` (0 in this config). This is the rate at which the LCDC pulls data from
SDRAM.

### 7.4 AC bias — `lcd_cfg_ac_bias()`

```c
reg = lcdc_read(LCD_RASTER_TIMING_2_REG) & 0xFFF00000;
reg |= LCD_AC_BIAS_FREQUENCY(period) |
       LCD_AC_BIAS_TRANSITIONS_PER_INT(transitions_per_int);
lcdc_write(reg, LCD_RASTER_TIMING_2_REG);
```

`ac_bias = 255`, `ac_bias_intrpt = 0`. AC bias matters for STN/passive panels;
for our active TFT it is programmed but of little consequence.

### 7.5 The sync registers — the geometry of a frame

Horizontal (`LCD_RASTER_TIMING_0_REG`):

```c
static void lcd_cfg_horizontal_sync(int back_porch, int pulse_width, int front_porch)
{
    reg = lcdc_read(LCD_RASTER_TIMING_0_REG) & 0xf;
    reg |= ((back_porch  & 0xff) << 24)
        |  ((front_porch & 0xff) << 16)
        |  ((pulse_width & 0x3f) << 10);
    lcdc_write(reg, LCD_RASTER_TIMING_0_REG);
}
```

Vertical (`LCD_RASTER_TIMING_1_REG`):

```c
static void lcd_cfg_vertical_sync(int back_porch, int pulse_width, int front_porch)
{
    reg = lcdc_read(LCD_RASTER_TIMING_1_REG) & 0x3ff;
    reg |= ((back_porch  & 0xff) << 24)
        |  ((front_porch & 0xff) << 16)
        |  ((pulse_width & 0x3f) << 10);
    lcdc_write(reg, LCD_RASTER_TIMING_1_REG);
}
```

With the default panel this writes:

```
horizontal:  back_porch=179<<24  front_porch=50<<16  pulse_width=27<<10
vertical:    back_porch=24<<24   front_porch=1<<16   pulse_width=3<<10
```

These porches create the blanking periods during which the panel's scan electronics
retrace — the same numbers that define the 1056×508 total frame of Chapter 6.5.

### 7.6 Display type and polarity — `lcd_cfg_display()`

```c
reg = lcdc_read(LCD_RASTER_CTRL_REG) & ~(LCD_TFT_MODE |
                                         LCD_MONO_8BIT_MODE |
                                         LCD_MONOCHROME_MODE);

switch (cfg->p_disp_panel->panel_shade) {
case COLOR_ACTIVE:
    reg |= LCD_TFT_MODE;                 /* active-matrix TFT */
    break;
...
}
/* revision 2: enable the underflow interrupt via INT_ENABLE_SET */
reg_int = lcdc_read(LCD_INT_ENABLE_SET_REG) | LCD_V2_UNDERFLOW_INT_ENA;
lcdc_write(reg_int, LCD_INT_ENABLE_SET_REG);

/* then the timing-2 polarity bits */
if (cfg->sync_ctrl)         reg |= LCD_SYNC_CTRL;
if (cfg->invert_line_clock) reg |= LCD_INVERT_LINE_CLOCK;   /* 1 */
if (cfg->invert_frm_clock)  reg |= LCD_INVERT_FRAME_CLOCK;  /* 1 */
lcdc_write(reg, LCD_RASTER_TIMING_2_REG);
```

For this board: **TFT mode on**, **line clock inverted**, **frame clock
inverted**, **sync control on**, **pixel clock inverted** (from Chapter 7.1, step
2). These polarity choices come from the panel's datasheet and are baked into the
board's `lcd_cfg` + the panel entry.

### 7.7 Frame buffer geometry — `lcd_cfg_frame_buffer()`

This programs the resolution fields:

```c
/* Panel width — pixels per line */
width &= 0x7f0;                       /* rev 2: up to 2048 */
reg |= ((width >> 4) - 1) << 4 | ...; /* 800 → (50-1) in bits 4..10 */
lcdc_write(reg, LCD_RASTER_TIMING_0_REG);

/* Panel height — lines per panel */
reg = ((height - 1) & 0x3ff) | (reg & 0xfffffc00);    /* 480-1 */
lcdc_write(reg, LCD_RASTER_TIMING_1_REG);
/* bit 10 of LPP lives in TIMING_2 on rev 2 */
reg |= ((height - 1) & 0x400) << 16;
lcdc_write(reg, LCD_RASTER_TIMING_2_REG);

/* 24/32 bpp → the 24-bit "unpacked" mode */
if (bpp == 24)
    reg |= LCD_TFT_MODE | LCD_V2_TFT_24BPP_MODE;
else if (bpp == 32)                    /* ← our case */
    reg |= LCD_TFT_MODE | LCD_V2_TFT_24BPP_MODE | LCD_V2_TFT_24BPP_UNPACK;
lcdc_write(reg, LCD_RASTER_CTRL_REG);
```

Here is the answer to the 32-bpp mystery:

> `LCD_V2_TFT_24BPP_MODE` tells the LCDC to push 24 bits per pixel on the data
> bus. `LCD_V2_TFT_24BPP_UNPACK` tells it the source buffer is *unpacked* — each
> pixel is 4 bytes in memory with a padding byte. The LCDC skips the pad byte and
> sends only the RGB bytes. Memory is 32-bit, the wire is 24-bit.

### 7.8 FDD — FIFO DMA request delay

```c
lcdc_write((lcdc_read(LCD_RASTER_CTRL_REG) & 0xfff00fff) |
           (cfg->fdd << 12), LCD_RASTER_CTRL_REG);
```

`fdd = 0x80`: the FIFO waits a number of cycles after the DMA request before
requesting, a tuning knob against FIFO overrun/underrun.

**Reading the code:** `da8xx-fb.c:846` (`lcd_init`), `:829`
(`lcd_calc_clk_divider`), `:532` (`lcd_cfg_dma`), `:564` (`lcd_cfg_ac_bias`),
`:575` (`lcd_cfg_horizontal_sync`), `:587` (`lcd_cfg_vertical_sync`), `:599`
(`lcd_cfg_display`), `:667` (`lcd_cfg_frame_buffer`).

---

## Chapter 8 · The clock math and the reported mode

### 8.1 The "+1" hack

The driver must *report* a `pixclock` (period in picoseconds) in
`fb_var_screeninfo`. If it reports a period the console framework deems
inconsistent with the actual hardware, mode-setting can fail or the console picks
a wrong timing. `da8xxfb_pixel_clk_period()` computes the period from the *actual*
post-divider clock and adds one picosecond on the ROM-3310:

```c
static unsigned int da8xxfb_pixel_clk_period(struct da8xx_fb_par *par)
{
    unsigned int lcd_clk, div, configured_pix_clk;
    unsigned long long pix_clk_period_picosec = 1000000000000ULL;

    lcd_clk = clk_get_rate(par->lcdc_clk);       /* 260 MHz   */
    div = lcd_clk / par->pxl_clk;                /* 260/40 = 6 */
    configured_pix_clk = lcd_clk / div;          /* 43.33 MHz  */

    do_div(pix_clk_period_picosec, configured_pix_clk);  /* 23077 ps */

#ifdef CONFIG_MACH_AM335X_ROM3310
    return pix_clk_period_picosec + 1;           /* 23078 ps — the "+1" */
#else
    return pix_clk_period_picosec;
#endif
}
```

So the framebuffer advertises a pixel-clock period of **23078 ps** (i.e.
≈43.3 MHz) — matching what the LCDC will actually produce. The `+1` is a
rounding/tolerance nudge so that the reported value stays inside the tolerance
windows of mode-comparison and console re-validation for this board.

### 8.2 The reported mode in `fb_var_screeninfo`

From probe, the driver fills the variable screen info with the panel's geometry:

```c
da8xx_fb_var.xres = da8xx_fb_var.xres_virtual = 800;
da8xx_fb_var.yres = 480;
da8xx_fb_var.yres_virtual = 480 * 2;      /* double buffer visible as 960 */
da8xx_fb_var.bits_per_pixel = 32;

da8xx_fb_var.hsync_len = 27;
da8xx_fb_var.vsync_len = 3;
da8xx_fb_var.pixclock = da8xxfb_pixel_clk_period(par);   /* 23078 ps */

da8xx_fb_var.right_margin  = 50;    /* hfp */
da8xx_fb_var.left_margin   = 179;   /* hbp */
da8xx_fb_var.lower_margin  = 1;     /* vfp */
da8xx_fb_var.upper_margin  = 24;    /* vbp */
```

Note how the framebuffer *naming* differs from the panel *naming*:

| Framebuffer field | Panel field | Value |
|-------------------|-------------|-------|
| `right_margin`    | hfp (front porch) | 50 |
| `left_margin`     | hbp (back porch)  | 179 |
| `lower_margin`    | vfp               | 1  |
| `upper_margin`    | vbp               | 24 |
| `hsync_len`       | hsw               | 27 |
| `vsync_len`       | vsw               | 3  |

This is why `da8xxfb_set_par()` copies the other way around:

```c
lcdc_info->hfp = info->var.right_margin;   /* fb → panel naming */
lcdc_info->hbp = info->var.left_margin;
lcdc_info->vfp = info->var.lower_margin;
lcdc_info->vbp = info->var.upper_margin;
...
ret = lcd_init(par, lcd_cfg, lcdc_info);   /* re-program with the new geometry */
```

So `set_par` doesn't just switch modes for fun — it re-runs the entire Chapter 7
programming sequence whenever the console changes resolution or depth.

### 8.3 `fb_check_var()` — policing what users can set

The console framework calls `fb_check_var` before accepting any mode change. It
validates the pixel format and clamps the virtual resolution to the VRAM:

```c
case 32:
    var->transp.offset = 24;  var->transp.length = 8;   /* XRGB */
    var->red.offset   = 16;   var->red.length   = 8;
    var->green.offset = 8;    var->green.length = 8;
    var->blue.offset  = 0;    var->blue.length  = 8;
    break;
```

So 32 bpp is always XRGB8888: 8 bits each of red, green, blue, and an ignored
high byte — matching the `LCD_V2_TFT_24BPP_UNPACK` layout of Chapter 7.7.

**Reading the code:** `da8xx-fb.c:1428` (`da8xxfb_pixel_clk_period`), `:1049`
(`fb_check_var`), `:1364` (`da8xxfb_set_par`).

---

## Chapter 9 · The rhythm of a frame: interrupts and double buffering

### 9.1 Enabling the raster

The raster engine is the part of the LCDC that actually emits pixels. It is
started by `lcd_enable_raster()`:

```c
static inline void lcd_enable_raster(void)
{
    /* Put LCDC in reset for several cycles, then bring it back */
    lcdc_write(LCD_CLK_MAIN_RESET, LCD_CLK_RESET_REG);
    mdelay(1);
    lcdc_write(0, LCD_CLK_RESET_REG);
    mdelay(1);

    reg = lcdc_read(LCD_RASTER_CTRL_REG);
    if (!(reg & LCD_RASTER_ENABLE))
        lcdc_write(reg | LCD_RASTER_ENABLE, LCD_RASTER_CTRL_REG);
}
```

The software reset (`LCD_CLK_MAIN_RESET`) before enabling is a revision-2
requirement: the LCDC must be cycled through reset with its register context kept
(the comment says so: *"Above reset sequence does not reset register context"*).

`lcd_disable_raster()` does the reverse — clears the enable bit and then waits up
to 5 seconds for the frame-done bit:

```c
reg = lcdc_read(LCD_RASTER_CTRL_REG);
if (reg & LCD_RASTER_ENABLE)
    lcdc_write(reg & ~LCD_RASTER_ENABLE, LCD_RASTER_CTRL_REG);

do {
    stat = (lcd_revision == LCD_VERSION_1) ? lcdc_read(LCD_STAT_REG)
                                          : lcdc_read(LCD_RAW_STAT_REG);
    mdelay(1);
} while (!(stat & BIT(0)) && (i++ < loop_cnt));   /* wait for frame done */
```

### 9.2 The end-of-frame dance

Now the beautiful part. Both DMA channels are programmed with the *same* buffer
range by `lcd_blit(LOAD_DATA, par)`:

```c
reg_ras |= LCD_PALETTE_LOAD_MODE(DATA_ONLY);
reg_dma |= LCD_DUAL_FRAME_BUFFER_ENABLE;
/* enable EOF0, EOF1, underflow, sync-lost interrupts */
...
lcdc_write(start, LCD_DMA_FRM_BUF_BASE_ADDR_0_REG);
lcdc_write(end,   LCD_DMA_FRM_BUF_CEILING_ADDR_0_REG);
lcdc_write(start, LCD_DMA_FRM_BUF_BASE_ADDR_1_REG);
lcdc_write(end,   LCD_DMA_FRM_BUF_CEILING_ADDR_1_REG);
lcdc_write(reg_dma, LCD_DMA_CTRL_REG);
lcdc_write(reg_ras, LCD_RASTER_CTRL_REG);
lcd_enable_raster();              /* must be last — raster enable after all fields */
```

Now the LCDC alternates between the two channels, one frame per channel. When a
channel finishes a frame it raises **END_OF_FRAME0** or **END_OF_FRAME1**. The
interrupt handler (`lcdc_irq_handler_rev02`) responds:

```c
if (stat & LCD_END_OF_FRAME0) {
    par->which_dma_channel_done = 0;
    lcdc_write(par->dma_start, LCD_DMA_FRM_BUF_BASE_ADDR_0_REG);
    lcdc_write(par->dma_end,   LCD_DMA_FRM_BUF_CEILING_ADDR_0_REG);
    par->vsync_flag = 1;
    wake_up_interruptible(&par->vsync_wait);
    if (vsync_cb_handler)
        vsync_cb_handler(vsync_cb_arg);
}
if (stat & LCD_END_OF_FRAME1) {
    par->which_dma_channel_done = 1;
    /* same, for channel 1 */
}
```

What is this telling the world?

- The channel that just finished can now be **re-pointed at the buffer the
  software is writing next** (via `da8xx_pan_display`, Chapter 12) — no tearing,
  because the LCDC is still draining the *other* channel.
- `wake_up_interruptible(&par->vsync_wait)` unblocks any userspace process waiting
  on `FBIO_WAITFORVSYNC` — the standard way for an application to draw only
  between frames.
- `vsync_cb_handler` is the callback the DA8xx "vsync" API exposes; the console
  (or an app) can register to get a callback at every frame boundary.

`lcd_blit()` also handles **palette loading** (`LOAD_PALETTE`), which DMA's the
256-entry palette to the LCDC's internal palette RAM — used when switching to
8-bit mode.

### 9.3 Interrupt hygiene and error recovery

The handler also watches for trouble:

```c
if ((stat & LCD_SYNC_LOST) && (stat & LCD_FIFO_UNDERFLOW)) {
    printk(KERN_ERR "LCDC sync lost or underflow error occured\n");
    lcd_disable_raster(NO_WAIT_FOR_FRAME_DONE);
    lcdc_write(stat, LCD_MASKED_STAT_REG);
    lcd_enable_raster();          /* try to restart cleanly */
}
```

A FIFO underflow (the DMA could not keep up) or a lost sync trigger a full
raster restart. In normal operation this should never happen — if it does, it is
a DMA-bandwidth or clock-margin problem, and the console will visibly stutter.

### 9.4 The palette-load interrupt

`LCD_PL_LOAD_DONE` marks the completion of a palette DMA. The handler disables the
raster, clears the flag, disables the PL interrupt, and starts data loading:

```c
else if (stat & LCD_PL_LOAD_DONE) {
    lcd_disable_raster(NO_WAIT_FOR_FRAME_DONE);
    lcdc_write(stat, LCD_MASKED_STAT_REG);
    reg_int = lcdc_read(LCD_INT_ENABLE_CLR_REG) | LCD_V2_PL_INT_ENA;
    lcdc_write(reg_int, LCD_INT_ENABLE_CLR_REG);
    lcd_blit(LOAD_DATA, par);      /* and the raster begins for real */
}
```

**Reading the code:** `da8xx-fb.c:411` (`lcd_enable_raster`), `:434`
(`lcd_disable_raster`), `:469` (`lcd_blit`), `:931` (`lcdc_irq_handler_rev02`),
`:993` (`lcdc_irq_handler_rev01`).

---

## Chapter 10 · Backlight by PWM — `pwm_bl.c`

### 10.1 The hardware side

The panel's backlight LED strip is dimmed by a **PWM** signal from the ECAP2
peripheral. The pin is muxed in `board-rom3310.h`:

```c
static struct pinmux_config ecap2_pin_mux[] = {
    {"mcasp0_ahclkr.ecap2_in_pwm2_out", OMAP_MUX_MODE4 | AM33XX_PIN_OUTPUT_PULLUP},
    {NULL, 0},
};
```

The board enables it (and records the fact) in `enable_ecap2()`:

```c
static void enable_ecap2(int evm_id, int profile)
{
    backlight_enable = true;
    setup_pin_mux(ecap2_pin_mux);        /* (under CONFIG_ECAP_PWM) */
}
```

### 10.2 The platform data

In `board-advantech.c` the backlight platform device is created with:

```c
#define AM335X_BACKLIGHT_MAX_BRIGHTNESS     100
#define AM335X_BACKLIGHT_DEFAULT_BRIGHTNESS 100
#define AM335X_PWM_PERIOD_NANO_SECONDS     (5000 * 10)     /* 50 000 ns = 20 kHz */

static struct platform_pwm_backlight_data am335x_backlight_data2 = {
    .pwm_id         = "ecap.2",
    .ch             = -1,
    .lth_brightness = 21,                 /* duty stays ≥ 21% below full dim */
    .max_brightness = AM335X_BACKLIGHT_MAX_BRIGHTNESS,    /* 100 */
    .dft_brightness = AM335X_BACKLIGHT_DEFAULT_BRIGHTNESS,/* 100 */
    .pwm_period_ns  = AM335X_PWM_PERIOD_NANO_SECONDS,     /* 50000 */
};
```

The **PWM period is 50 000 ns** (a 20 kHz carrier — above audible range) and the
backlight device is registered as `"pwm-backlight"` (matching `pwm_bl.c`).

### 10.3 The driver

`pwm_bl.c` is the generic kernel PWM-backlight driver. Its heart is
`pwm_backlight_update_status()`:

```c
static int pwm_backlight_update_status(struct backlight_device *bl)
{
    struct pwm_bl_data *pb = dev_get_drvdata(&bl->dev);
    int brightness = bl->props.brightness;
    int max = bl->props.max_brightness;

    if (bl->props.power != FB_BLANK_UNBLANK)
        brightness = 0;                    /* blanked → off */
    if (bl->props.fb_blank != FB_BLANK_UNBLANK)
        brightness = 0;

    if (brightness == 0) {
        pwm_set_duty_ns(pb->pwm, 0);       /* 0% duty */
#ifndef CONFIG_MACH_ADVANTECH
        pwm_stop(pb->pwm);
#endif
    } else {
        /* map 0..max onto duty with a minimum floor (lth_brightness) */
        brightness = pb->lth_brightness +
            (brightness * (pb->period - pb->lth_brightness) / max);
        pwm_set_period_ns(pb->pwm, pb->period);
        pwm_set_duty_ns(pb->pwm, brightness);
        pwm_start(pb->pwm);
    }
    ...
}
```

Three details worth noting for this board:

1. **`lth_brightness = 21`** — even at brightness 0 the PWM duty is not truly 0;
   the mapping is `duty = 21 + brightness × (50000 − 21)/100`. The floor keeps the
   backlight driver's switching circuitry (or the panel's minimum-on behavior)
   happy, and the "100% duty" case becomes `21 + 100×498.79 = 50000 ns` — a full
   period.

2. **`#ifndef CONFIG_MACH_ADVANTECH pwm_stop`** — on the Advantech boards the PWM
   is *never stopped*, only set to 0% duty. This is a deliberate choice: the
   inverter or panel on these carriers does not like the PWM signal disappearing
   entirely (some inverters re-ramp slowly or generate audible artifacts when the
   clock restarts). The PWM keeps running at 20 kHz and just goes to 0 duty.

3. Probe requests the PWM by name: `pwm_request("ecap.2", -1, "backlight")` —
   the `pwm_id` string and `ch = -1` from the platform data.

The `backlight` sysfs API that userspace sees is the standard one:
`/sys/class/backlight/` with `brightness`, `max_brightness`, `bl_power` files.

**Reading the code:** `drivers/video/backlight/pwm_bl.c` (whole file, 226 lines);
`board-advantech.c:111-133` (`am335x_backlight_data*`), `:510`
(`am335x_backlight` device), `:540` (`backlight_init`).

---

## Chapter 11 · The CH7055 bridge — `ch7055.c`

### 11.1 The chip and its place

The **Chrontel CH7055** sits between the LCDC and the panel (§2.1). It is on
**I2C0 at `0x76`**, declared in `board-rom3310.h`:

```c
I2C_BOARD_INFO("ch7055", 0x76),
```

It appears in the same I2C0 table as the TPS65910 PMIC (0x2D), the PCA9555 GPIO
expander (0x22), the S35390A RTC (0x30), the TLV320AIC3x audio codec (0x1B), and
the ADV430 watchdog (0x29) — a reminder that on this carrier a lot of small
peripherals share one I2C bus.

### 11.2 The driver is small on purpose

`ch7055.c` is only ~160 lines. It does two things: set a sensible default
brightness, and expose brightness through sysfs.

```c
static int __devinit ch7055_probe(struct i2c_client *client, ...)
{
    ch7055_client = client;

    /* default brightness 0x40 = 64 (of 0..127) into three registers */
    i2c_smbus_write_byte_data(ch7055_client, 0x04, 0x40);
    i2c_smbus_write_byte_data(ch7055_client, 0x05, 0x40);
    i2c_smbus_write_byte_data(ch7055_client, 0x06, 0x40);

    /* software mode */
    i2c_smbus_write_byte_data(ch7055_client, 0x02, 0x90);

    sysfs_create_group(&client->dev.kobj, &ch7055_attr_group);
    ...
}
```

The brightness sysfs attribute reads and writes registers `0x04..0x06`:

```c
static ssize_t ch7055_store_brightness(struct device *dev, ...,
                                       const char *buf, size_t count)
{
    if ((strict_strtoul(buf, 10, &val) < 0) || (val > 127))
        return -EINVAL;
    i2c_smbus_write_byte_data(ch7055_client, 0x04, val);
    i2c_smbus_write_byte_data(ch7055_client, 0x05, val);
    i2c_smbus_write_byte_data(ch7055_client, 0x06, val);
    return count;
}
```

So on this board, brightness is controlled from userspace like:

```sh
echo 100 > /sys/devices/.../0-0076/brightness
```

The register at `0x02` written with `0x90` selects **software mode** — i.e. the
chip's brightness is controlled by the I2C registers rather than by an external
hardware pin.

**Reading the code:** `drivers/video/ch7055.c` (whole file, 162 lines);
`board-rom3310.h:573` (the I2C entry).

---

## Chapter 12 · The framebuffer's user-facing face — `fb_ops`

### 12.1 The operation table

The driver's `fb_ops` connects the generic Linux framebuffer core to this
hardware:

```c
static struct fb_ops da8xx_fb_ops = {
    .owner          = THIS_MODULE,
    .fb_check_var   = fb_check_var,        /* Chapter 8.3 */
    .fb_set_par     = da8xxfb_set_par,     /* Chapter 8.2 */
    .fb_setcolreg   = fb_setcolreg,        /* palette (8-bit mode) */
    .fb_pan_display = da8xx_pan_display,   /* swap the DMA target mid-frame */
    .fb_ioctl       = fb_ioctl,            /* FBIO_WAITFORVSYNC etc. */
    .fb_fillrect    = cfb_fillrect,        /* software accel helpers */
    .fb_copyarea    = cfb_copyarea,
    .fb_imageblit   = cfb_imageblit,       /* ← this draws the boot logo */
    .fb_blank       = cfb_blank,
};
```

The `cfb_*` functions are the generic **software** drawing helpers — the AM335x
LCDC has no 2-D blitter, so rectangles, copies, and image blits (fonts, logos)
are done by the CPU, then the result is just displayed by the raster engine.

### 12.2 `cfb_blank()` — power states

```c
static int cfb_blank(int blank, struct fb_info *info)
{
    switch (blank) {
    case FB_BLANK_UNBLANK:
        if (par->panel_power_ctrl) par->panel_power_ctrl(1);
        lcd_enable_raster();
        break;
    case FB_BLANK_NORMAL:
    case FB_BLANK_VSYNC_SUSPEND:
    case FB_BLANK_HSYNC_SUSPEND:
    case FB_BLANK_POWERDOWN:
        if (par->panel_power_ctrl) par->panel_power_ctrl(0);
        lcd_disable_raster(WAIT_FOR_FRAME_DONE);
        break;
    }
}
```

Note `par->panel_power_ctrl` — the platform data has a function pointer for the
board to toggle the panel power. On this board `panel_power_ctrl` is not set in
the pdata (the GPIO power is managed directly in `lcdc_init`), so blanking here
only gates the raster; the backlight GPIOs stay as `lcdc_init` left them.

### 12.3 `fb_ioctl()` — waiting for the vertical sync

```c
case FBIO_WAITFORVSYNC:
    return fb_wait_for_vsync(info);
```

`fb_wait_for_vsync()` sleeps on the wait queue that the interrupt handler wakes
every frame (Chapter 9.2):

```c
par->vsync_flag = 0;
ret = wait_event_interruptible_timeout(par->vsync_wait,
                                       par->vsync_flag != 0,
                                       par->vsync_timeout);   /* HZ/5 */
```

The comment inside the function is a gem of embedded pragmatism — the race where
the ISR fires "just before or just after" the flag is cleared is explicitly
accepted, because the caller only needs a coarse "a frame just finished" signal.
This is exactly what an application needs to avoid tearing: draw, wait for vsync,
swap buffers, repeat.

### 12.4 `da8xx_pan_display()` — tear-free buffer swaps

Panning is how a double-buffered application shows its new frame. The app writes
into the *virtual* screen (`yres_virtual = 960`), then pans to the second half:

```c
start = fix->smem_start +
        new_var.yoffset * fix->line_length +
        new_var.xoffset * fbi->var.bits_per_pixel / 8;
end   = start + fbi->var.yres * fix->line_length - 1;
par->dma_start = start;
par->dma_end   = end;

/* program the channel that is *not* currently being drained */
if (par->which_dma_channel_done == 0) {
    lcdc_write(par->dma_start, LCD_DMA_FRM_BUF_BASE_ADDR_0_REG);
    lcdc_write(par->dma_end,   LCD_DMA_FRM_BUF_CEILING_ADDR_0_REG);
} else if (par->which_dma_channel_done == 1) {
    lcdc_write(par->dma_start, LCD_DMA_FRM_BUF_BASE_ADDR_1_REG);
    lcdc_write(par->dma_end,   LCD_DMA_FRM_BUF_CEILING_ADDR_1_REG);
}
```

This is the classic **triple-handed** dance: channel 0 is on screen, channel 1 is
being filled by the app, and the driver waits for the end-of-frame interrupt to
know *which* channel just freed up, then re-points exactly that one. The result:
no tearing, no flicker, at the cost of a slightly larger VRAM.

**Reading the code:** `da8xx-fb.c:1414` (`da8xx_fb_ops`), `:1280` (`cfb_blank`),
`:1243` (`fb_ioctl`), `:1206` (`fb_wait_for_vsync`), `:1316`
(`da8xx_pan_display`).

---

# Part IV — The Light, and the First Picture

## Chapter 13 · TLC59108 — the I2C backlight controller

The TI **TLC59108** is an I2C LED driver (8 constant-current channels) that this
BSP also wires up as a backlight device. On the ROM-3310 Puhui carrier it is not
declared in the ROM-3310 I2C table (it appears in the TI EVM board file at
`0x40`), but it is compiled in (`CONFIG_BACKLIGHT_TLC59108=y`) and available as
an alternative backlight path.

`tlc59108.c` registers a backlight device named `"tlc59108-bl"`:

```c
static void tlc59108_bl_set_backlight(struct tlc59108_bl *data, int brightness)
{
    i2c_smbus_write_byte_data(data->client, TLC59108_MODE1, 0x00);   /* normal */
    i2c_smbus_write_byte_data(data->client, TLC59108_LEDOUT0, 0x21); /* outputs */
    i2c_smbus_write_byte_data(data->client, TLC59108_PWM2,
                              brightness & 0xff);                    /* duty */
}
```

The architecture is the same as `pwm_bl.c`: a `backlight_device` whose
`update_status` pushes the brightness into the chip. The difference is that the
PWM is generated *inside* the TLC59108 (register `0x04`, 0..255) rather than by
the SoC's ECAP. `MAX_BRIGHTNESS = 0xFF`, and on AM335x it also forces LED4 fully
on via `LEDOUT1 = 0x01`.

**Reading the code:** `drivers/video/backlight/tlc59108.c` (whole file, 170
lines); `board-am335xevm.c:2665` (the `0x40` I2C entry).

---

## Chapter 14 · The first picture — the boot logo

### 14.1 From a PPM file to C

The boot logo begins its life as a Netpbm PPM file — 224-color (clut224) format —
in `drivers/video/logo/logo_linux_clut224.ppm`. This repository keeps alternates
in `source/logo/`:

| File | Brand |
|------|-------|
| `logo_linux_clut224.ppm-clemens` / `..._clemens.ppm` | Clemens |
| `logo_linux_clut224_purwell.ppm` | Purwell |
| `logo_linux_clut224_mesic.ppm` | Mesic |
| `logo_linux_clut224_lepuyiliao-poweron.ppm` | Le Pu Yi Liao (medical) |

At build time the kernel's `scripts/pnmtologo` converts the PPM into a C file
`logo_linux_clut224.c`:

```makefile
$(obj)/%_clut224.c: $(src)/%_clut224.ppm $(pnmtologo) FORCE
	$(call cmd_logo,clut224)
```

So "changing the boot logo" = replacing `logo_linux_clut224.ppm` and rebuilding.
The generated C file is just a byte array of pixels plus a 224-entry color
palette:

```c
static const struct linux_logo logo_linux_clut224 = {
    .type      = LINUX_LOGO_CLUT224,
    .width     = 640,
    .height    = 480,
    .clutsize  = 224,
    .clut      = (const unsigned char *){ ... },   /* palette */
    .data      = (const unsigned char[]){ ... },   /* pixel indices */
};
```

### 14.2 Who draws it

At console startup, the generic framebuffer console (`fbcon`) asks the logo layer
for an image and blits it to the screen via `fb_imageblit`:

```c
int fb_prepare_logo(struct fb_info *info, int rotate)
{
    ...
    fb_logo.logo = fb_find_logo(depth);     /* pick by color depth */
    ...
}
```

`fb_find_logo()` (`drivers/video/logo/logo.c`) picks the CLUT224 logo whenever
the display depth is ≥ 8:

```c
if (depth >= 8) {
#ifdef CONFIG_LOGO_LINUX_CLUT224
    logo = &logo_linux_clut224;
#endif
    ...
}
```

Then `fb_show_logo()` / `fb_show_logo_line()` draw it into the framebuffer, line
by line, using the framebuffer's `fb_imageblit` (which on this driver is the
generic `cfb_imageblit`). Since the logo is drawn into the same VRAM the LCDC is
DMA-ing, it simply appears — no special display code needed.

The config knobs:

```
CONFIG_LOGO=y
CONFIG_LOGO_LINUX_MONO=y
CONFIG_LOGO_LINUX_VGA16=y
CONFIG_LOGO_LINUX_CLUT224=y       # the one actually shown at ≥8 bpp
```

The `am335x_rom3310_puhui_no-logo_defconfig` simply disables these.

**Reading the code:** `drivers/video/logo/logo.c:32` (`fb_find_logo`),
`drivers/video/logo/Makefile` (the PPM→C rule), `drivers/video/fbcon.c:450,590`
(`fb_show_logo_line` / `fb_prepare_logo`).

---

## Chapter 15 · A boot-time walkthrough

Here is the whole story, in order, from power-on to a lit Linux console.

```
1. ROM bootloader → MLO → U-Boot → uImage → kernel starts.
2. Board setup runs the device table; lcdc_init() is invoked:
     a. setup_pin_mux(lcdc_pin_mux)          → 24-bit RGB + sync pins are LCDC.
     b. GPIOs driven high:                   → LCD_VDD_EN, LCD_BKLT_EN, VGA_LIGHT_EN.
     c. panel_disp_clock_select()            → 260 MHz (panel_type 9).
     d. conf_disp_pll(260e6)                 → dpll_disp_ck = 260 MHz.
     e. am33xx_register_lcdc(&INNOLUX_VGA_800x480_pdata)
                                            → platform device "da8xx_lcdc" born.
3. fb_probe() matches and runs:
     a. ioremap 0x4830E000; clk_get(lcdc_fck).
     b. LCD_PID_REG = 0x4F200800 → LCD_VERSION_2.
     c. string match → known_lcd_panels[9] = INNOLUX_VGA_800x480@60.
     d. 3 MiB coherent VRAM + palette; /dev/fb0 registered; IRQ 36 requested.
     e. fb_set_var() → da8xxfb_set_par() → lcd_init():
          divider = 260/40 = 6; all timing regs written; raster still off.
4. backlight_init() (late_initcall):
     a. am33xx_register_ecap(2)  → ECAP2 PWM running.
     b. platform_device_register(pwm-backlight) → backlight at 100/100.
5. fbcon comes up:
     a. fb_prepare_logo() → fb_find_logo(≥8bpp) → logo_linux_clut224.
     b. fb_show_logo() → cfb_imageblit draws the logo into VRAM.
     c. raster enabled → LCDC DMA begins → first frame out the pins.
     d. CH7055 probe (I2C) set default brightness 0x40.
6. The logo appears. The Puhui/medical brand name flashes, then the console
   text takes over, scrolling line by line into the same buffer.
7. Userspace starts. Apps can:
     - read/write /dev/fb0 directly,
     - use FBIO_WAITFORVSYNC + pan_display for tear-free animation,
     - set brightness via /sys/.../backlight or the CH7055 sysfs attribute.
```

Every one of the seven steps is a chapter of this book. If you hold this list
next to the source tree, you now know exactly where each piece lives.

---

## Appendix A · Register quick reference

For the default panel (800×480, 32 bpp, 260 MHz display PLL):

| Register | Value written | Meaning |
|----------|---------------|---------|
| `LCD_CTRL_REG` (0x04) | `div=6<<8 \| RASTER_MODE` | pixel clock = 260/6 ≈ 43.3 MHz |
| `LCD_CLK_ENABLE_REG` (0x6C) | `DMA\|LIDD\|CORE` clk en | rev-2 clocks on |
| `RASTER_TIMING_0` (0x2C) | `179<<24 \| 50<<16 \| 27<<10` + PPL(800) | horizontal geometry |
| `RASTER_TIMING_1` (0x30) | `24<<24 \| 1<<16 \| 3<<10` + LPP(480) | vertical geometry |
| `RASTER_TIMING_2` (0x34) | polarity + AC bias + 24bpp | invert pxl/line/frame clk, TFT-24bpp |
| `DMA_CTRL` (0x40) | burst 16, fifo_th, dual-buffer | DMA behavior |
| `RASTER_CTRL` (0x28) | TFT, 24bpp, unpack, FDD 0x80, raster-en | master control |
| FRM_BUF 0/1 base/ceil (0x44-0x50) | vram_phys → +screen_size-1 | DMA source ranges |

## Appendix B · Math sheet

```
Display PLL           = 260 MHz              (panel_type 9)
LCD divider           = 260/40              = 6      (integer)
Actual pixel clock    = 260/6               ≈ 43.33 MHz
Pixel clock period    = 1e12 / 43.33e6      ≈ 23077 ps   (reported 23078 ps)
Line (total)          = 800 + 50 + 179 + 27 = 1056
Frame (total)         = 480 + 1 + 24 + 3    = 508
Frame rate            = 43.33e6 / (1056·508) ≈ 80.7 Hz
VRAM (double-buffered)= 800·480·4 · 2        = 3 072 000 B (≈ 3 MiB)
Line length           = 800 · 4              = 3200 B
```

## Appendix C · Where everything lives (one-page map)

| Concern | File |
|---------|------|
| LCDC hardware description (addr/IRQ/clock) | `arch/arm/mach-omap2/omap_hwmod_33xx_data.c` |
| Display PLL + LCD clock tree | `arch/arm/mach-omap2/clock33xx_data.c` |
| Pin mux, panel_type, clock select, I2C tables | `arch/arm/mach-omap2/board-rom3310.h` |
| lcdc_init, panel pdata, backlight pdata, conf_disp_pll | `arch/arm/mach-omap2/board-advantech.c` |
| am33xx_register_lcdc / ecap | `arch/arm/mach-omap2/devices.c` |
| Framebuffer driver (the heart) | `drivers/video/da8xx-fb.c` |
| Driver structures | `include/video/da8xx-fb.h` |
| CH7055 bridge | `drivers/video/ch7055.c` |
| PWM backlight | `drivers/video/backlight/pwm_bl.c` |
| I2C backlight | `drivers/video/backlight/tlc59108.c` |
| Boot logo | `drivers/video/logo/` (+ `source/logo/` alternates) |
| Feature switches | `arch/arm/configs/am335x_rom3310_puhui_defconfig` |

---

## Afterword

The LCD subsystem of the AM3358 is, in the end, a very old idea executed very
well: *the CPU decides, the silicon draws.* The framebuffer is just a rectangle
of memory; the LCDC is just a very fast, very patient reader of that rectangle;
and everything else in this book — the PLL, the pin mux, the GPIOs, the CH7055,
the PWM — exists to make sure that when the LCDC pushes a pixel out of the chip,
there is a working, lit panel waiting to receive it.

Understanding it is mostly a matter of holding the right two things in your head
at once: **what the panel's datasheet requires** (timings, polarities, pixel
clock) and **what the LCDC is physically capable of** (integer dividers, 24-bit
bus, dual DMA channels). Every "weird" line in `da8xx-fb.c` — the `+1`, the
24bpp-unpack flag, the raster reset before enable, the pwm_stop guard — is a
small treaty between those two realities. This book has tried to show you the
treaties one by one.

*The pixel you write tonight is the pixel you see tomorrow.*
