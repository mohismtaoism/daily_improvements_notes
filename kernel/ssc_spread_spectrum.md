# Spread Spectrum Clocking (SSC) for ROM-3310 (AM3352)

*From the EMC problem, through the AM335x hardware, to the kernel code changes you make by hand.*

**Target platform:** ROM-3310 COM Express module + Puhui carrier, AM3352, Linux 3.2.0 (TI PSP 04.06.00.11)
**Goal:** Reduce conducted/radiated EMC noise by spreading the display clock frequency with SSC.
**Audience:** an embedded engineer who reads datasheets, holds a soldering iron, and edits kernel code by hand.
**Reference:** AM335x TRM SPRUH73*, TI U-Boot `set_mpu_spreadspectrum()`, linux-omap SSC patch series.

---

## Table of Contents

1. [Why we need this: the EMC problem](#1-why-we-need-this-the-emc-problem)
2. [What spread spectrum clocking actually does](#2-what-spread-spectrum-clocking-actually-does)
3. [The AM335x SSC hardware](#3-the-am335x-ssc-hardware)
4. [The SSC math](#4-the-ssc-math)
5. [How to adjust SSC (with the idea behind it in one story)](#5-how-to-adjust-ssc-with-the-idea-behind-it-in-one-story)
6. [The display PLL on your board](#6-the-display-pll-on-your-board)
7. [Worked example: 0.5% and 1% down-spread on DPLL_DISP](#7-worked-example-05-and-1-down-spread-on-dpll_disp)
8. [The reference implementation (U-Boot) — annotated](#8-the-reference-implementation-u-boot--annotated)
9. [The kernel integration design](#9-the-kernel-integration-design)
10. [Manual code changes, step by step](#10-manual-code-changes-step-by-step)
11. [Build and run](#11-build-and-run)
12. [Verification on hardware](#12-verification-on-hardware)
13. [Tuning and troubleshooting](#13-tuning-and-troubleshooting)
14. [Risks, limitations, and notes](#14-risks-limitations-and-notes)
15. [Appendix A — Register quick reference](#appendix-a--register-quick-reference)
16. [Appendix B — The complete diff](#appendix-b--the-complete-diff)
17. [Appendix C — Optional: SSC on the MPU PLL](#appendix-c--optional-ssc-on-the-mpu-pll)
18. [References](#18-references)

---

## 1. Why we need this: the EMC problem

Every digital system switches signals at a clock frequency. A 260 MHz display clock is a very
efficient radiator: all its energy sits in a single, very narrow spectral line at 260 MHz, plus
strong harmonics at 520 MHz, 780 MHz, 1040 MHz, …

For regulatory testing (CISPR 22 / EN 55022 / EN 55011), an emission *peak* at one of those lines
frequently exceeds the limit, even though the **total** radiated energy is small. The classic
countermeasure is **spreading** that single line into a band, so the energy that was concentrated
at 260.000 MHz is redistributed across, say, 260.000 MHz → 258.700 MHz. The peak drops by several dB,
the total energy stays the same, and the test passes.

Your board is a medical/industrial product (the "Le Pu Yi Liao" boot logo and the surgical-handle
hardware suggest a medical device). Those products are certified to strict EMC limits, and the
display ribbon/LVDS cable is usually the loudest radiator. SSC on the display PLL is the most
targeted, lowest-cost fix.

---

## 2. What spread spectrum clocking actually does

A DPLL multiplies a reference frequency to a high-frequency clock. Its output frequency is
`Fdco = Fref × M / N`. If we slowly vary the multiplier `M` back and forth, the output frequency
is modulated — it "sweeps" continuously between a low and a high value.

There are two modulation shapes:

| Mode | Frequency range | Effect |
|------|----------------|--------|
| **Center spread** | nominal ± Δ | clock is sometimes *above* nominal |
| **Down spread** (recommended) | nominal − 2Δ … nominal | clock never exceeds nominal |

**Why down-spread is the right choice for a display:**

1. **Timing margin.** The display timing (sync pulses, porches) is derived from the nominal clock.
   With down-spread the clock is *never faster* than nominal, so the tightest timing edges are not
   stressed. A slow drift is absorbed by the panel's internal PLL/link.
2. **Range.** The TRM says down-spread gives a 2× effective spread on the low side, which means you
   can use a smaller `ΔM` for the same suppression.
3. **Interoperability.** Many downstream PHYs (LVDS transmitters, HDMI bridges like your CH7055)
   tolerate slow down-spread well, but are pickier about overshooting their nominal input range.

**The modulation waveform** is triangular. The period of the triangle is the **modulation
frequency** `Fm` (typically a few tens to a few hundred kHz). It must be:
- fast enough that the emissions peak reduction is effective (a slow sweep is still a peak at any
  instant — the analyser's detector integrates over time, so ~30 kHz and up is sensible);
- slow enough that the display PLL locks and tracks the sweep (a few hundred kHz is fine).

**The two parameters you program are therefore:**

- `MODFREQDIV` — sets the **modulation frequency** `Fm`;
- `DELTAMSTEP` — sets the **modulation depth** (how far the frequency swings = the spread %).

---

## 3. The AM335x SSC hardware

### 3.1 Which PLLs support SSC

From the AM335x TRM (SPRUH73*):

> *"SSC is functionally supported only for the **LCD (DISP)** and **MPU** PLLs."*

The `CM_SSC_DELTAMSTEP_DPLL_*` and `CM_SSC_MODFREQDIV_DPLL_*` registers (and the SSC enable
field in `CM_CLKMODE_DPLL_*`) are *mapped* for all five PLLs (CORE, MPU, DDR, PER, DISP) — but on
the **CORE, DDR and PER** PLLs the silicon does not actually implement spreading. **Programming SSC
on those is pointless (and possibly ignored).**

This is convenient for us: the DISP PLL is one of the two PLLs SSC genuinely works on, and it is
precisely the PLL that drives the display — your loudest emitter.

### 3.2 The three registers you touch

Each PLL has a `CM_SSC_DELTAMSTEP`, a `CM_SSC_MODFREQDIV`, and the SSC bits live in its
`CM_CLKMODE`. All live in the CM_WKUP register block (base `0x44E0_0000`).

| PLL | CLKMODE | SSC_DELTAMSTEP | SSC_MODFREQDIV | CLKSEL (M/N) |
|-----|---------|----------------|----------------|--------------|
| MPU | 0x0488 | 0x0424 | 0x0428 | 0x042C |
| DDR | 0x0494 | 0x0438 | 0x043C | 0x0440 |
| **DISP** | **0x0498** | **0x044C** | **0x0450** | **0x0454** |
| CORE | 0x0490 | 0x0460 | 0x0464 | 0x0468 |
| PER | 0x048C | 0x0474 | 0x0478 | (periph) 0x049C |

*(Full 32-bit absolute addresses: 0x44E0_0000 + offset. In the kernel these are already defined as
`AM33XX_CM_CLKMODE_DPLL_DISP`, `AM33XX_CM_SSC_DELTAMSTEP_DPLL_DISP`, etc.)*

### 3.3 `CM_CLKMODE_DPLL_*` — the SSC control bits

Bit fields (identical layout in all five PLLs):

| Bit | Field | Meaning |
|-----|-------|---------|
| 12 | `SSC_EN` | 1 = spread spectrum enabled |
| 13 | `SSC_ACK` | **status** (read-only): 1 = SSC active / acknowledged |
| 14 | `SSC_DOWNSPREAD` | 1 = down-spread; 0 = center-spread |
| 15 | (reserved) | — |

The kernel already defines these correctly in `arch/arm/mach-omap2/cm-regbits-33xx.h`:

```c
#define AM33XX_DPLL_SSC_EN_SHIFT        12
#define AM33XX_DPLL_SSC_EN_MASK         (1 << 12)
#define AM33XX_DPLL_SSC_ACK_SHIFT       13
#define AM33XX_DPLL_SSC_ACK_MASK        (1 << 13)
#define AM33XX_DPLL_SSC_DOWNSPREAD_SHIFT 14
#define AM33XX_DPLL_SSC_DOWNSPREAD_MASK (1 << 14)
```

### 3.4 `CM_SSC_MODFREQDIV_DPLL_*` — modulation frequency

Per the TRM:

```
Fm = (DPLL_REFCLK / 4) / MODFREQDIV
MODFREQDIV = MODFREQDIV_MANTISSA × 2^MODFREQDIV_EXPONENT
```

Where `DPLL_REFCLK` is the reference clock **after** the DPLL pre-divider
(`= OSCIN / (N+1)` — for you: 24 MHz / predivider).

Register field layout (all five PLLs identical):

| Bits | Field | Width |
|------|-------|-------|
| 10–8 | `MODFREQDIV_EXPONENT` | 3 bits |
| 6–0  | `MODFREQDIV_MANTISSA` | 7 bits |

**TI's recommendation: use the maximum mantissa and the minimum exponent.** A value of
`mantissa = 17, exponent = 0` is what the reference implementation always arrives at (see §8).

> **⚠️ WARNING — wrong masks in this kernel's headers.**
> `arch/arm/mach-omap2/cm-regbits-33xx.h` contains these definitions, which are **wrong**
> (a header-generation bug — they do not match the TRM):
>
> ```c
> #define AM33XX_MODFREQDIV_EXPONENT_MASK  (0x10 << 8)   /* WRONG, TRM says 0x7 << 8 */
> #define AM33XX_MODFREQDIV_MANTISSA_MASK  (0x06 << 0)   /* WRONG, TRM says 0x7f << 0 */
> #define AM33XX_DELTAMSTEP_MASK           (0x19 << 0)   /* WRONG, TRM says 0xfffff << 0 */
> ```
>
> **Do not use these masks in your code.** The guide in §10 avoids them entirely by writing the
> complete registers. The SSC_EN/ACK/DOWNSPREAD single-bit masks above are correct.

### 3.5 `CM_SSC_DELTAMSTEP_DPLL_*` — modulation depth

`DELTAMSTEP` controls how far the multiplier `M` swings each modulation step. It is a **20-bit
fixed-point** value: a 2-bit integer part (bits 19–18) and an 18-bit fractional part (bits 17–0).

The relationship to the M-step is:

```
ΔM = 2^EXPONENT × MANTISSA × (DELTAMSTEP / 2^18)        [EXPONENT ≤ 3]
ΔM = 8 × MANTISSA × (DELTAMSTEP / 2^18)                 [EXPONENT > 3]
```

Since `MODFREQDIV = MANTISSA × 2^EXPONENT`, the first case collapses to:

```
ΔM = MODFREQDIV × (DELTAMSTEP / 2^18)
```

So `DELTAMSTEP = (ΔM × 2^18) / MODFREQDIV`, and the spread in percent is `100 × ΔM / M`.

**Range limits the TRM imposes (so the DPLL stays in a valid lock):**

| Constraint | Condition |
|-----------|-----------|
| `M − ΔM ≥ 20` | always (center or down) |
| `M + ΔM ≤ 2045` | center-spread only |
| `M − 2ΔM ≥ 20` | down-spread |
| `M ≤ 2045` | down-spread |

Your DISP PLL uses `M = 65`, so these are trivially satisfied.

---

## 4. The SSC math

We can now assemble the complete, proven algorithm. It comes from TI's own U-Boot code
(`set_mpu_spreadspectrum` in `arch/arm/cpu/armv7/am33xx/dpll.c`), which is the only public,
hardware-validated implementation of AM335x SSC. The Linux `clk-ti` series by Dario Binacchi
(2021) is a direct port of the same algorithm.

**Inputs:**
- `M`, `N` — read from `CM_CLKSEL_DPLL_DISP` (M = bits 17:8, N = bits 6:0, actual predivider = N+1);
- `OSCIN` — the crystal / sys_clkin frequency (24 MHz on your board);
- `permille` — the desired down-spread in **tenths of a percent** (5 → 0.5 %, 10 → 1 %, 20 → 2 %).

**Step 1 — reference clock and modulation-frequency divider:**

```
ref_clock        = OSCIN / (N + 1)          // reference into the PLL after pre-divider
pll_bandwidth    = ref_clock / 70           // ~loop bandwidth, TI rule of thumb
mod_freq_divider = ref_clock / (4 × pll_bandwidth)   // always ≈ 17 in practice
```

**Step 2 — MODFREQDIV (mantissa/exponent, max-mantissa rule):**

```
exponent = 0, mantissa = mod_freq_divider
while mantissa > 127 and exponent < 7:  mantissa >>= 1; exponent++
if mantissa > 127:  mantissa = 127
mod_freq_divider = mantissa << exponent          // final value for the register
```

**Step 3 — DELTAMSTEP (fixed point):**

```
delta_m_step = ((M × permille) / 10 / 2) << 18   // *2^18 = fractional bits
delta_m_step /= 100
delta_m_step /= mod_freq_divider
if delta_m_step > 0xFFFFF:  delta_m_step = 0xFFFFF    // clamp to 20 bits
```

The three scalings deserve explanation:
- `/ 10` — converts *tenths of a percent* (permille) into *percent*;
- `/ 2` — down-spread produces 2× the deviation on the low side (per the TRM), so we ask for half
  the ΔM to obtain exactly the requested spread;
- `/ 100` — converts percent into a fraction.

The `<< 18` turns the integer result into a 2.18 fixed-point number so the fractional arithmetic
of `ΔM / MODFREQDIV` isn't lost.

**Step 4 — program and enable:**

```
write delta_m_step                → CM_SSC_DELTAMSTEP_DPLL_DISP
write (exponent<<8) | mantissa    → CM_SSC_MODFREQDIV_DPLL_DISP
clkmode = read CM_CLKMODE_DPLL_DISP
clkmode &= ~(SSC_EN | SSC_ACK | SSC_DOWNSPREAD)     // clear all SSC flags
clkmode |= SSC_EN | SSC_DOWNSPREAD
write clkmode                     → CM_CLKMODE_DPLL_DISP
poll CM_CLKMODE_DPLL_DISP.SSC_ACK until 1           // SSC active
```

That is the whole implementation. It is ~40 lines of C.

---

## 5. How to adjust SSC (with the idea behind it in one story)

*If the formulas in §4 made your eyes glaze over, read this chapter instead — it is the same
content in a different language. If you already understand §4, jump straight to the knobs in
§5.3 and the adjust-loop in §5.6.*

### 5.1 The whole idea in one story

> Imagine a **drummer** in a big empty hall. He hits his drum once every second, always at
> exactly the same pace: *Boom. Boom. Boom.* The sound is very strong — at that one speed the
> whole hall shakes.
>
> Now the drummer learns a trick. Instead of always the same pace, he swings it a tiny bit,
> like a smooth wave: *slow… fast… slow… fast…* — but every second he still hits the drum once.
> He is not working less, not hitting quieter. He just never hits two booms at exactly the same
> speed.
>
> A friend stands at the far end of the hall with a listening machine that can only hear **one
> speed at a time**. Before the trick, the machine finds one very loud speed. After the trick,
> the sound is spread across many speeds, and at **any single speed** it is much, much quieter —
> even though the drum itself is just as loud overall.

**That is all SSC does.** It is the drummer's trick.

- Your **260 MHz display clock** is the drummer.
- The **EMC test machine** is the friend with the listening machine.
- Without SSC: all the radio energy sits at one frequency → the test fails.
- With SSC: the clock's speed wobbles a tiny bit, so the energy spreads over a small band →
  every single frequency is quiet enough → the test passes.

### 5.2 What's really happening (the honest version)

Your DISP PLL multiplies the 24 MHz crystal up to 260 MHz using `Fdco = 24 MHz × M / predivider`,
with **M = 65, predivider = 6**. SSC slowly wobbles M up and down, so the output frequency
sweeps in a smooth triangle:

```
frequency
    │  ┌─────────┐       ┌─────────┐
    │  │         │       │         │
260 ───┘         └───────┘         └───   (down-spread: never above 260)
    │
    └───────────────────────────────────── time
         └── one wobble = one Fm period ──┘
```

Three things describe the wobble, and each maps to a knob in the code:

| What the wobble has | What it's called | Which knob |
|---|---|---|
| How wide it is | spread % | `SSC_PERMILLE` (Knob 1) |
| How fast it waves | modulation frequency `Fm` | `mod_freq_divider` (Knob 2) |
| Which way it swings | center vs down | `SSC_DOWNSPREAD` bit (Knob 3) |

### 5.3 Knob 1 — how much to spread (the one you'll use 90% of the time)

One number in the helper function in `board-advantech.c` (§10 Step 2):

```c
#define SSC_PERMILLE        5        /* ← change this number */
```

The number is the spread in **tenths of a percent**. For your 260 MHz display PLL:

| `SSC_PERMILLE` | Down-spread | DELTAMSTEP written | Effect on EMC |
|---|---|---|---|
| 3 | 0.3 % | 0x56B | smallest wobble, smallest gain |
| **5** | **0.5 %** | **0x9A3** | **recommended starting point** |
| 10 | 1 % | 0x1346 | ~3 dB more suppression than 0.5 % |
| 20 | 2 % | 0x2727 | maximum recommended for a display |

**Rule of thumb:** bigger number → quieter radio peak, but wobblier clock (more pixel jitter).
Start at 5, measure, go to 10 if the peak is still too high, stop at 20. If the screen starts
flickering, go back down.

*(These register values assume the default 260 MHz panel — §7 has the worked math. The boot
message tells you the exact values your board produced.)*

### 5.4 Knob 2 — how fast the wobble is (rarely needs touching)

The formula picks **Fm ≈ 59 kHz** for your DISP PLL automatically — a good middle: fast enough
to fool the test machine, slow enough for the panel to follow. To slow the wobble down, override
the divider after it is computed:

```c
mod_freq_divider = ref_clock / (4 * pll_bandwidth);   /* ≈17 → 59 kHz */
mod_freq_divider = 34;    /* ← force it: Fm drops to ≈ 29 kHz */
```

Because the DELTAMSTEP formula divides by the same `mod_freq_divider`, **the spread % stays
exactly the same** — you are only changing the speed, not the width. ~30–60 kHz is a safe range
for a display.

### 5.5 Knob 3 — which direction (leave as-is, but know it exists)

Down-spread is enabled by the `SSC_DOWNSPREAD` bit. It is the safe choice: the clock never runs
faster than 260 MHz, so the panel's timing margin is never stressed. Center-spread would give the
same peak reduction with half the width, but the clock goes above nominal — for a display, stay
with down-spread.

### 5.6 The adjust → measure → adjust loop

```
1.  Edit SSC_PERMILLE in board-advantech.c
2.  ./mk_kernel.sh            (or your normal build)
3.  Flash uImage, boot, read the console line:
        SSC: DPLL_DISP spread enabled: M=65 N=6 Fm=58823 Hz deltamstep=0x9a3
4.  Confirm SSC is active:    devmem 0x44e00498   (bit 13 = 1)
5.  Measure on the spectrum analyser (or near-field probe over the LVDS cable)
6.  Still too loud  → raise SSC_PERMILLE
    Screen unhappy  → lower it
```

That is the whole loop — one number, rebuild, measure. For anything that misbehaves, the
troubleshooting checklist in §13.

---

## 6. The display PLL on your board

### 6.1 The clock path

```
OSC0 (24 MHz)
   └─ sys_clkin_ck            (24 MHz)
        └─ dpll_disp_ck       (= Fdco = 24 MHz × M / (N+1))
             └─ dpll_disp_m2_ck   (÷ M2 from CM_DIV_M2_DPLL_DISP)
                  └─ disp_pll_clk
                       └─ (LCD controller pixel clock path)
```

Defined in `arch/arm/mach-omap2/clock33xx_data.c` (lines ~397–451).

### 6.2 Where the frequency is set

In the **compiled** board file `arch/arm/mach-omap2/board-advantech.c`:

1. `lcdc_init()` (line 718) runs at boot;
2. it calls `disp_clock = panel_disp_clock_select();` (defined in the included
   `board-rom3310.h`, line 630);
3. then `conf_disp_pll(disp_clock)` (line 742 / function at line 584), which does:
   `clk_get(NULL, "dpll_disp_ck")` + `clk_set_rate(disp_pll, disp_clock)`;
4. `clk_set_rate` → `omap3_noncore_dpll_set_rate()` → `omap2_dpll_round_rate()` computes M and N,
   then `omap3_noncore_dpll_program()` **bypasses the PLL, writes M and N−1, and re-locks it**.

`panel_disp_clock_select()` returns the target DISP PLL frequency per panel type:

```c
unsigned long panel_disp_clock_select(void)
{
    unsigned long disp_clock = 0;
    if ( panel_type == 6 )
        disp_clock = 280000000;
    else if(panel_type == 7)
        disp_clock = 260000000;
    else if(panel_type == 8)
        disp_clock = 256500000;
    else
        disp_clock = 260000000;      // default for all other panels
    return disp_clock;
}
```

### 6.3 The exact M and N the kernel programs

`omap2_dpll_round_rate()` (in `arch/arm/mach-omap2/clkt_dpll.c`) searches for the (M, N) that makes
`Fdco = 24 MHz × M / N` land exactly on the target. For the default **260 MHz** target:

```
n=1: m=10  → 240 MHz      n=2: m=21 → 252 MHz      n=3: m=32 → 256 MHz
n=4: m=43  → 258 MHz      n=5: m=54 → 259.2 MHz    n=6: m=65 → 260 MHz  ✔ exact
```

So the register ends up as **M = 65, N-register = 5** (the predivider field stores `N−1`,
so the actual predivider is 6). `CM_CLKSEL_DPLL_DISP` will read `0x0000_4105`
(`(65 << 8) | 5`).

For the other panel types the values differ — that is exactly why the SSC function in §10 **reads M
and N from the register** instead of hard-coding them:

| Target | M | N-reg | Fdco |
|--------|----|----|------|
| 260 MHz (default) | 65 | 5 | 24×65/6 = 260.000 MHz |
| 280 MHz (panel 6) | 35 | 2 | 24×35/3 = 280.000 MHz |
| 256.5 MHz (panel 8) | 171 | 15 | 24×171/16 = 256.500 MHz |

---

## 7. Worked example: 0.5% and 1% down-spread on DPLL_DISP

Default configuration: **M = 65, N-register = 5**, OSCIN = 24 MHz.

```
ref_clock        = 24000000 / (5+1)  = 4,000,000 Hz
pll_bandwidth    = 4000000 / 70      = 57,142
mod_freq_divider = 4000000 / (4×57142) = 17.50 → 17
mantissa = 17, exponent = 0
Fm = 4000000 / 4 / 17 ≈ 58.8 kHz
```

**0.5 % down-spread (permille = 5):**

```
delta_m_step = ((65 × 5)/10 /2) << 18  / 100 / 17
             = (16 << 18)              / 100 / 17
             = 4194304                 / 100 / 17
             = 41943                   / 17
             = 2467          = 0x9A3
ΔM = 17 × (2467 / 2^18) = 0.160   →  spread = 2×ΔM/M = 0.49 % ✔
```

**1 % down-spread (permille = 10):**

```
delta_m_step = ((65 × 10)/10 /2) << 18  / 100 / 17
             = (32 << 18)               / 100 / 17
             = 8388608                  / 100 / 17
             = 83886                    / 17
             = 4934          = 0x1346
ΔM = 17 × (4934 / 2^18) = 0.320   →  spread = 2×ΔM/M = 0.98 % ✔
```

**2 % down-spread (permille = 20) — the practical maximum:**

```
delta_m_step = ((65 × 20)/10 /2) << 18  / 100 / 17
             = (65 << 18)               / 100 / 17
             = 17039360                 / 100 / 17
             = 170393                   / 17
             = 10023         = 0x2727
spread = 2×ΔM/M ≈ 1.97 % ✔
```

All three values fit comfortably in the 20-bit field (max 0xFFFFF) and the range checks
(`M−2ΔM = 65−0.64 = 64.36 ≥ 20`) pass easily.

**Register values to expect after the change (default 260 MHz panel):**

| Spread | SSC_DELTAMSTEP (0x44C) | SSC_MODFREQDIV (0x450) | CLKMODE SSC bits (0x498) |
|--------|------------------------|------------------------|--------------------------|
| 0.5 %  | `0x000009A3` | `0x00000011` (exp 0, mant 17) | bits 12,14 set → 0x5000 |
| 1 %    | `0x00001346` | `0x00000011` | 0x5000 |
| 2 %    | `0x00002727` | `0x00000011` | 0x5000 |

*(Note: the SSC bits are OR-ed into whatever CLKMODE already contains — `0x5000` is only the SSC
contribution; the DPLL enable bits in the low 3 bits remain.)*

---

## 8. The reference implementation (U-Boot) — annotated

This is the canonical TI function, in full, with comments. Keep it as your mental model:

```c
void set_mpu_spreadspectrum(int permille)
{
    u32 multiplier_m;
    u32 predivider_n;
    u32 cm_clksel_dpll_mpu;
    u32 cm_clkmode_dpll_mpu;
    u32 ref_clock;
    u32 pll_bandwidth;
    u32 mod_freq_divider;
    u32 exponent;
    u32 mantissa;
    u32 delta_m_step;

    /* Read PLL parameter m and n */
    cm_clksel_dpll_mpu = readl(&cmwkup->clkseldpllmpu);
    multiplier_m = (cm_clksel_dpll_mpu >> 8) & 0x3FF;
    predivider_n = cm_clksel_dpll_mpu & 0x7F;

    /* reference clock after pre-divider, its max. PLL bandwidth,
       and resulting modulation-frequency divider */
    ref_clock        = V_OSCK / (predivider_n + 1);
    pll_bandwidth    = ref_clock / 70;
    mod_freq_divider = ref_clock / (4 * pll_bandwidth);

    /* Calculate Mantissa/Exponent  (max mantissa, min exponent) */
    exponent = 0;
    mantissa = mod_freq_divider;
    while ((mantissa > 127) && (exponent < 7)) {
        exponent++;
        mantissa /= 2;
    }
    if (mantissa > 127)
        mantissa = 127;
    mod_freq_divider = mantissa << exponent;

    /*
     * Modulation steps: we use down-spread only, so the spread is twice
     * the value of permille → div by 2. The register takes the value in
     * percent → div by ten.
     */
    delta_m_step = ((u32)((multiplier_m * permille) / 10 / 2)) << 18;
    delta_m_step /= 100;
    delta_m_step /= mod_freq_divider;
    if (delta_m_step > 0xFFFFF)
        delta_m_step = 0xFFFFF;

    /* Setup spread spectrum */
    writel(delta_m_step,           &cmwkup->sscdeltamstepdllmpu);
    writel((exponent << 8) | mantissa, &cmwkup->sscmodfreqdivdpllmpu);

    cm_clkmode_dpll_mpu = readl(&cmwkup->clkmoddpllmpu);
    /* clear all SSC flags (bits 12-15) */
    cm_clkmode_dpll_mpu &= ~(0xF << CM_CLKMODE_DPLL_SSC_EN_SHIFT);
    /* enable SSC with down-spread only */
    cm_clkmode_dpll_mpu |=  CM_CLKMODE_DPLL_SSC_EN_MASK |
                            CM_CLKMODE_DPLL_SSC_DOWNSPREAD_MASK;
    writel(cm_clkmode_dpll_mpu, &cmwkup->clkmoddpllmpu);
    while (!(readl(&cmwkup->clkmoddpllmpu) & 0x2000))   /* SSC_ACK = bit 13 */
        ;
}
```

Notes on adapting it to the kernel DISP PLL:
- the register accesses are redirected to the DISP PLL's CM registers (see §3.2);
- `V_OSCK` (U-Boot's name for the 24 MHz crystal) becomes a kernel constant;
- the endless `while` ACK poll gets a bounded timeout;
- everything runs through `__raw_readl`/`__raw_writel`.

---

## 9. The kernel integration design

### 9.1 Where the hook goes

`lcdc_init()` already contains the exact point where the DISP PLL has just been configured **and
locked** — immediately after `conf_disp_pll(disp_clock)` succeeds, and before the LCDC device is
registered:

```c
    if (conf_disp_pll(disp_clock)) {
        pr_info("Failed configure display PLL, not attempting to"
                "register LCDC\n");
        return;
    }
    /* ← our call goes here: am33xx_disp_pll_set_ssc(); */
```

Why this is the right place:
- the PLL is already **locked** (SSC must be enabled while locked);
- it runs exactly once, after M/N are final — the SSC values are computed from the register, so they
  match the actual configuration;
- it is before `am33xx_register_lcdc()`, so no display activity is disturbed.

### 9.2 Design decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Target PLL | **DISP only** | TRM says SSC works only on DISP and MPU; DISP drives the loudest emitter. MPU is optional (§Appendix C). |
| Spread type | **down-spread** | clock never exceeds nominal → no timing-margin stress. |
| Default depth | **0.5 %** (`SSC_PERMILLE 5`) | safe start; you tune up after measuring. |
| M/N source | **read from `CM_CLKSEL_DPLL_DISP`** | robust against any panel_type / future rate change. |
| Masks | **write whole registers** | the kernel's DELTAMSTEP/MODFREQDIV masks are wrong (§3.4). |
| ACK wait | **bounded loop** | don't hang the boot if SSC fails. |
| Header | **`#include "cm33xx.h"`** | brings the `AM33XX_CM_SSC_*` address macros and the (correct) SSC_EN/ACK/DOWNSPREAD masks. |

---

## 10. Manual code changes, step by step

> **All changes are in ONE file: `arch/arm/mach-omap2/board-advantech.c`**
> (the only AM335x board file that is actually compiled — `board-am335xevm.c` is not linked, per the
> Makefile). You do **not** need to touch the clock framework, `dpll3xxx.c`, or any header.

### Step 1 — add the `cm33xx.h` include

Find the block of machine includes near the top of `board-advantech.c`:

```c
#include "common.h"
```

Add right below it (the file already sits in `mach-omap2/`, so the quoted include resolves):

```c
#include "cm33xx.h"
```

`cm33xx.h` pulls in `cm-regbits-33xx.h` itself, so you get both the address macros
(`AM33XX_CM_SSC_DELTAMSTEP_DPLL_DISP`, `AM33XX_CM_CLKMODE_DPLL_DISP`, `AM33XX_CM_CLKSEL_DPLL_DISP`)
and the single-bit SSC masks in one go.

### Step 2 — add the SSC helper function

Insert the following function **immediately before** `static int __init conf_disp_pll(int rate)`
(around line 584). Putting it there keeps it close to its only caller.

```c
/*
 * AM335x DPLL_DISP spread-spectrum clocking.
 *
 * Implements the TI reference algorithm (U-Boot set_mpu_spreadspectrum(),
 * arch/arm/cpu/armv7/am33xx/dpll.c), redirected to the display PLL.  Reads
 * the current M/N values from CM_CLKSEL_DPLL_DISP, programs
 * CM_SSC_DELTAMSTEP and CM_SSC_MODFREQDIV, then enables SSC with
 * down-spread and waits for SSC_ACK.
 *
 * Must be called AFTER the DPLL is locked (i.e. after conf_disp_pll()).
 *
 * NOTE: the DELTAMSTEP / MODFREQDIV *_MASK defines in cm-regbits-33xx.h
 * are wrong (truncated header-generation bug).  We therefore write the
 * whole registers and use no masks for them.
 *
 * NOTE: TRM SPRUH73* — SSC is functionally supported only on the LCD
 * (DISP) and MPU PLLs.
 */
#define SSC_OSCIN_HZ        24000000 /* OSC0 / sys_clkin, Hz (verify for your board) */
#define SSC_PERMILLE        5        /* desired down-spread in 0.1 % steps
                                        (5 = 0.5 %, 10 = 1 %, 20 = 2 %) */
#define SSC_MAX_ACK_WAIT    1000     /* bounded poll of SSC_ACK */

static void __init am33xx_disp_pll_set_ssc(void)
{
	u32 clksel, clkmode;
	u32 mult_m, div_n;              /* div_n = predivider field (N-1) */
	u32 ref_clock;                  /* OSCIN after pre-divider */
	u32 pll_bandwidth;
	u32 mod_freq_divider;
	u32 exponent, mantissa;
	u32 delta_m_step;
	int i;

	/* 1. Read current M and N from CM_CLKSEL_DPLL_DISP */
	clksel = __raw_readl(AM33XX_CM_CLKSEL_DPLL_DISP);
	mult_m = (clksel & AM33XX_DPLL_MULT_MASK) >> AM33XX_DPLL_MULT_SHIFT;
	div_n  = (clksel & AM33XX_DPLL_DIV_MASK)  >> AM33XX_DPLL_DIV_SHIFT;

	if (!mult_m) {
		pr_warn("SSC: DPLL_DISP not configured (M=0), skipped\n");
		return;
	}

	/* 2. Reference clock after pre-divider, PLL bandwidth, modulation-
	 *    frequency divider (TI rule of thumb: bw ~= ref/70) */
	ref_clock        = SSC_OSCIN_HZ / (div_n + 1);
	pll_bandwidth    = ref_clock / 70;
	mod_freq_divider = ref_clock / (4 * pll_bandwidth);

	/* 3. MODFREQDIV = mantissa x 2^exponent (max mantissa, min exponent) */
	exponent = 0;
	mantissa = mod_freq_divider;
	while ((mantissa > 127) && (exponent < 7)) {
		exponent++;
		mantissa >>= 1;
	}
	if (mantissa > 127)
		mantissa = 127;
	mod_freq_divider = mantissa << exponent;

	/* 4. DELTAMSTEP in 2.18 fixed point.
	 *    /10 : permille -> percent ; /2 : down-spread doubles deviation */
	delta_m_step = ((mult_m * SSC_PERMILLE) / 10 / 2) << 18;
	delta_m_step /= 100;
	delta_m_step /= mod_freq_divider;
	if (delta_m_step > 0xFFFFF)      /* 20-bit field */
		delta_m_step = 0xFFFFF;

	/* 5. Program the SSC parameters (whole-register writes) */
	__raw_writel(delta_m_step,           AM33XX_CM_SSC_DELTAMSTEP_DPLL_DISP);
	__raw_writel((exponent << 8) | mantissa,
	             AM33XX_CM_SSC_MODFREQDIV_DPLL_DISP);

	/* 6. Clear SSC flags, then enable SSC with down-spread */
	clkmode = __raw_readl(AM33XX_CM_CLKMODE_DPLL_DISP);
	clkmode &= ~(AM33XX_DPLL_SSC_EN_MASK | AM33XX_DPLL_SSC_ACK_MASK |
		     AM33XX_DPLL_SSC_DOWNSPREAD_MASK);
	clkmode |= AM33XX_DPLL_SSC_EN_MASK | AM33XX_DPLL_SSC_DOWNSPREAD_MASK;
	__raw_writel(clkmode, AM33XX_CM_CLKMODE_DPLL_DISP);

	/* 7. Wait for the hardware to acknowledge SSC */
	for (i = 0; i < SSC_MAX_ACK_WAIT; i++) {
		if (__raw_readl(AM33XX_CM_CLKMODE_DPLL_DISP) &
		    AM33XX_DPLL_SSC_ACK_MASK)
			break;
		cpu_relax();
	}

	if (i == SSC_MAX_ACK_WAIT)
		pr_warn("SSC: DPLL_DISP did not acknowledge SSC enable\n");
	else
		pr_info("SSC: DPLL_DISP spread enabled: M=%u N=%u Fm=%u Hz "
			"deltamstep=0x%x\n", mult_m, div_n + 1,
			ref_clock / 4 / mod_freq_divider, delta_m_step);
}
```

### Step 3 — call the helper from `lcdc_init()`

In `lcdc_init()` (line ~718), after the successful PLL configuration:

```c
	if (conf_disp_pll(disp_clock)) {
		pr_info("Failed configure display PLL, not attempting to"
				"register LCDC\n");
		return;
	}
	/* NEW: spread the display clock to reduce EMC emissions */
	am33xx_disp_pll_set_ssc();
```

That is the entire code change.

### Step 4 — optional: make the spread % a boot parameter

If you want to tune the spread without re-flashing, replace the `SSC_PERMILLE` macro with a global:

```c
static unsigned int ssc_permille = 5;          /* module-ish knob */

/* in __init section */
module_param(ssc_permille, uint, 0644);
```

and use `ssc_permille` in the DELTAMSTEP line. (Note: this is a built-in board file, so
`module_param` works only if `CONFIG_MODULES` — you can equally expose it via a `sysfs` file or a
`cmdline` parse. For a first bring-up, the macro is simpler.)

---

## 11. Build and run

Rebuild exactly as you normally do:

```bash
cd scripts
source setenv.sh
./cfg_kernel.sh am335x_rom3310_puhui_defconfig
./mk_kernel.sh
```

Boot the new `uImage`. On the console you should see, near the LCD init messages:

```
lcdc_init  start
SSC: DPLL_DISP spread enabled: M=65 N=6 Fm=58823 Hz deltamstep=0x9a3
lcdc_init  evm_id:0
lcdc_init  end
```

*(`N=6` is the actual predivider; the register holds 5. `Fm ≈ 58.8 kHz`, `deltamstep = 0x9a3` for
0.5 %.)*

If the kernel was built without `CONFIG_DEBUG` the `pr_info` still shows — `pr_info` is always
compiled in.

---

## 12. Verification on hardware

### 12.1 Register read-back (fast, no equipment)

On the target, dump the three registers (devmem or busybox):

```sh
# CM_WKUP base 0x44e00000
devmem 0x44e0044c        # SSC_DELTAMSTEP_DPLL_DISP  → 0x000009a3 (0.5%)
devmem 0x44e00450        # SSC_MODFREQDIV_DPLL_DISP  → 0x00000011
devmem 0x44e00498        # CLKMODE_DPLL_DISP         → bit13 (SSC_ACK) = 1
```

Checklist:
- `SSC_ACK` (bit 13 of 0x44e00498) reads **1**. If it stays 0, SSC did not engage (see §13).
- `SSC_EN` (bit 12) reads **1**.
- `SSC_DOWNSPREAD` (bit 14) reads **1**.

### 12.2 Functional check

- The LCD picture must remain stable, no flicker, no rolling.
- The HDMI bridge (CH7055) output must stay stable.
- Run the system for a few hours; watch for any panel lock-loss.

### 12.3 EMC measurement (the real test)

With a spectrum analyser on the radiated/conducted setup (or a near-field probe over the LVDS
ribbon / the board's clock area):

1. **Baseline**: measure the 260 MHz line and its harmonics **without** SSC. Note the peak levels.
2. **With 0.5 % SSC**: the 260 MHz line should spread into a ~1.3 MHz-wide hump and the *peak* level
   should drop — typically **3–6 dB**.
3. **With 1 % SSC**: expect roughly double the width and another ~3 dB.
4. If you have a scope, trigger on the pixel clock and confirm it sweeps at ~59 kHz between ~260 MHz
   and ~258.7 MHz (down-spread → never above nominal).

> **Tip**: don't be surprised if the drop at the *fundamental* is modest but the **harmonics**
> (520/780/1040 MHz) drop more. Harmonics of a spread clock spread proportionally, so the 2nd
> harmonic spreads twice as wide — usually the biggest win is exactly where your fail is.

---

## 13. Tuning and troubleshooting

### 13.1 Spread too small / EMC still fails

Increase `SSC_PERMILLE` from 5 → 10 → 20 (0.5 % → 1 % → 2 %) and re-measure. Practical ceiling for
a display: **1–2 %**. Above that, pixel-clock jitter and panel PLL tracking become risks.

### 13.2 `SSC_ACK` never asserts

- Confirm the DPLL is actually **locked**: `devmem 0x44e00448` (IDLEST_DPLL_DISP) bit 0 should be 1
  (CLK active / locked).
- Confirm you are reading the DISP registers, not MPU. Offset mix-ups here are easy.
- If `M=0` was read (our guard), the PLL was not configured before the call — the call order is wrong.
- Re-check `CM_SSC_MODFREQDIV` value: `0x11` means exponent 0, mantissa 17. A 0 there means the
  formula produced a bad divider (suspect `SSC_OSCIN_HZ`).

### 13.3 LCD flickers or loses lock after enabling SSC

- **Too much spread.** Drop back to 0.5 % or reduce to permille 3.
- **Modulation too fast.** Our `Fm ≈ 59 kHz` comes from the reference formula. If the panel's link
  can't track it, you can slow it by raising `mod_freq_divider` (the register is easy to
  experiment with — e.g. write mantissa 34/exp 0 → Fm ≈ 29 kHz). This is a tuning knob, not part of
  the shipped algorithm.
- **Panel timing generated from the PLL.** Some LCDC configurations derive horizontal/vertical
  timing from the same PLL and a wide spread can show as line jitter. 1 % is the safe ceiling.

### 13.4 Suspect the pixel clock isn't the only emitter

SSC only helps the **DISP and MPU** clocks. If the failing frequency isn't a multiple of the display
clock, look at other clocks (DDR, PER, ethernet). For the DDR clock there is no SSC support — use
shielding/grounding there.

### 13.5 I want the boot logo to still be clean

SSC is enabled in `lcdc_init`, which runs **before** the framebuffer/logo is drawn, so the whole
display path (including the logo) runs spread from the start. Good — no mid-stream switch.

---

## 14. Risks, limitations, and notes

1. **SSC is supported only on the DISP and MPU PLLs** (TRM). Do not waste effort trying it on
   CORE/PER/DDR — the registers exist but the function is not implemented in silicon for those.
2. **The +20 % inaccuracy.** The TRM warns that the actual spread has up to ~20 % error on the
   programmed value, and there is some overshoot. Treat the programmed % as approximate and verify
   with a measurement.
3. **Pixel-clock jitter.** Down-spread modulates the pixel clock by the chosen percent at ~59 kHz.
   For 0.5–1 % this is imperceptible on virtually all panels and the CH7055. Above 2 % the risk rises.
4. **Reprogramming the DISP PLL later.** If anything else ever calls `clk_set_rate(dpll_disp_ck)`
   after `lcdc_init`, the PLL is re-programmed (bypass → new M/N → lock) while the SSC registers
   still hold the *old* computed values. The SSC would then be slightly off-target. Currently
   `lcdc_init` is the only place the DISP PLL rate is set, so this is a latent, not active, issue.
   If you ever add a dynamic display-rate change, call `am33xx_disp_pll_set_ssc()` again after it.
5. **The DELTAMSTEP/MODFREQDIV masks in `cm-regbits-33xx.h` are wrong.** Documented in §3.4. Our code
   avoids them; if you later write your own SSC code, use the TRM field values
   (`0xFFFFF<<0`, `0x7<<8`, `0x7F<<0`).
6. **This is a per-board change** (`board-advantech.c`). It is deliberately *not* a clock-framework
   change, so it affects only the ROM-3310/Advantech boards and cannot destabilise other targets.
7. **OSCIN must match your crystal.** The formula assumes 24 MHz. If a future board uses 25 or
   26 MHz, update `SSC_OSCIN_HZ`. (It does not affect the PLL itself — only the modulation
   frequency and DELTAMSTEP scaling.)

---

## Appendix A — Register quick reference

Base of CM_WKUP = **0x44E0_0000**. All offsets in hex.

| Register | Offset | Access | Purpose |
|----------|--------|--------|---------|
| `CM_SSC_DELTAMSTEP_DPLL_DISP` | 0x044C | RW | modulation depth, 2.18 fixed point (20 bits) |
| `CM_SSC_MODFREQDIV_DPLL_DISP` | 0x0450 | RW | exponent bits 10–8, mantissa bits 6–0 |
| `CM_CLKSEL_DPLL_DISP` | 0x0454 | RW | M bits 17–8, N (N−1) bits 6–0, bypass sel bit 23 |
| `CM_CLKMODE_DPLL_DISP` | 0x0498 | RW | SSC_EN=12, SSC_ACK=13 (ro), SSC_DOWNSPREAD=14 |

Kernel macro names (use these — no magic numbers):

| Register | Kernel macro |
|----------|--------------|
| DELTAMSTEP | `AM33XX_CM_SSC_DELTAMSTEP_DPLL_DISP` |
| MODFREQDIV | `AM33XX_CM_SSC_MODFREQDIV_DPLL_DISP` |
| CLKSEL | `AM33XX_CM_CLKSEL_DPLL_DISP` |
| CLKMODE | `AM33XX_CM_CLKMODE_DPLL_DISP` |
| M field | `AM33XX_DPLL_MULT_MASK` / `AM33XX_DPLL_MULT_SHIFT` |
| N field | `AM33XX_DPLL_DIV_MASK` / `AM33XX_DPLL_DIV_SHIFT` |
| SSC_EN / ACK / DOWNSPREAD | `AM33XX_DPLL_SSC_EN_MASK` / `..._ACK_MASK` / `..._DOWNSPREAD_MASK` |

Key formulas (recap):

```
Fm          = (OSCIN / (N+1) / 4) / MODFREQDIV
MODFREQDIV  = MANTISSA × 2^EXPONENT            (max mantissa, min exponent)
ΔM          = MODFREQDIV × DELTAMSTEP / 2^18
down-spread = 2 × ΔM / M × 100  [%]
range: M−2ΔM ≥ 20, M ≤ 2045
```

---

## Appendix B — The complete diff

```diff
--- a/arch/arm/mach-omap2/board-advantech.c
+++ b/arch/arm/mach-omap2/board-advantech.c
@@ include block
+#include "cm33xx.h"
+
+/* ... am33xx_disp_pll_set_ssc() ... (see §10 Step 2) */
+
 static int __init conf_disp_pll(int rate)
 {
@@ lcdc_init()
        if (conf_disp_pll(disp_clock)) {
                pr_info("Failed configure display PLL, not attempting to"
                        "register LCDC\n");
                return;
        }
+       /* NEW: spread the display clock to reduce EMC emissions */
+       am33xx_disp_pll_set_ssc();
        pr_info("lcdc_init  evm_id:%d\n",evm_id);
```

Two insertions in one file. That is the entire patch.

---

## Appendix C — Optional: SSC on the MPU PLL

If after fixing the display you still have margin problems from the CPU clock, the same algorithm
works on the MPU PLL. This time you also need to consider that the CPU is performance-sensitive —
a large down-spread lowers the CPU's peak frequency. **0.5 % down-spread costs 0.5 % peak
performance**, which is negligible.

The changes mirror §10 but with the MPU register macros and a new call site (e.g. near the end of
`am335x_evm_init()`, after `clk_set_rate` on `dpll_mpu_ck`):

| Item | DISP version | MPU version |
|------|--------------|-------------|
| CLKSEL | `AM33XX_CM_CLKSEL_DPLL_DISP` | `AM33XX_CM_CLKSEL_DPLL_MPU` |
| DELTAMSTEP | `AM33XX_CM_SSC_DELTAMSTEP_DPLL_DISP` | `AM33XX_CM_SSC_DELTAMSTEP_DPLL_MPU` |
| MODFREQDIV | `AM33XX_CM_SSC_MODFREQDIV_DPLL_DISP` | `AM33XX_CM_SSC_MODFREQDIV_DPLL_MPU` |
| CLKMODE | `AM33XX_CM_CLKMODE_DPLL_DISP` | `AM33XX_CM_CLKMODE_DPLL_MPU` |

Check the MPU's M/N (for ~600–720 MHz on a 24 MHz crystal, M≈25–30 with the predivider at 1,
i.e. N-register 0 — verify with the same register-read trick: `devmem 0x44e0042c`). The same
range rules apply (`M−2ΔM ≥ 20`).

---

## 18. References

1. **AM335x Sitara™ Processors Technical Reference Manual (SPRUH73\*)** — sections on *DPLL
   Configuration*, *CM_SSC_DELTAMSTEP_DPLL_* / *CM_SSC_MODFREQDIV_DPLL_* / *CM_CLKMODE_DPLL_*;
   "SSC is functionally supported only for the LCD and MPU PLLs."
2. **TI U-Boot**, `arch/arm/cpu/armv7/am33xx/dpll.c` — `set_mpu_spreadspectrum()` (the algorithm
   this guide ports). Also `arch/arm/include/asm/arch-am33xx/cpu.h` for the CM_WKUP struct.
3. **linux-omap patch series, D. Binacchi, 2021**: *"clk: ti: add am33xx spread spectrum clock
   support"* (`[PATCH v6 3/5] ARM: dts: am33xx-clocks: add spread spectrum support`) — the accepted
   Linux port of the same algorithm, confirming the register map for all five PLLs and the
   field masks `ssc_enable_mask = 1<<12`, `ssc_downspread_mask = 1<<14`,
   `ssc_deltam_int_mask = 0x3<<18`, `ssc_deltam_frac_mask = 0x3FFFF<<0`,
   `ssc_modfreq_mant_mask = 0x7F`, `ssc_modfreq_exp_mask = 0x7<<8`.
4. This tree's own headers: `arch/arm/mach-omap2/cm33xx.h` (register addresses),
   `cm-regbits-33xx.h` (bit fields — note the erroneous DELTAMSTEP/MODFREQDIV masks, §3.4).

---

*Document written for the ROM-3310 / Puhui carrier BSP. All code changes are manual-guide only;
nothing in this document modifies the tree on its own.*
