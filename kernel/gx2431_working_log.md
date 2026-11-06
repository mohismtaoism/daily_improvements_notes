# 最终解决方案

## 原理

  修改 ds2484_w1_reset_bus 函数，每次复位都先试标准速度，不行再试 OverDrive：

| 步骤 |  速度 | 哪位芯片响应 |
|-----------|---------|--------|
| 1 | 标准速度 (1WS=0) | GX2431 ✅ |
| 2 | OverDrive (1WS=1) | DS28E25 ✅ |

## 最终代码

  probe 函数中的配置（保持原始 OverDrive 默认值）：

```c
  ds2484_write_config(data, (DS2484_REG_CFG_1WS | DS2484_REG_CFG_APU));

  ds2484_adjust_port(data, tRSTL_OD, 0x0A);
  ds2484_adjust_port(data, tMSP_OD, 0x08);
  ds2484_adjust_port(data, tW0L_OD, 0x0A);

  ds2484_w1_reset_bus(&data->w1_ch);
```

ds2484_w1_reset_bus 函数（核心改动）：

```c
  static u8 ds2484_w1_reset_bus(void *data)
  {
        struct ds2484_w1_chan *pchan = data;
        struct ds2484_data    *pdev = pchan->pdev;
        int err;
        u8 retval = 1;

        mutex_lock(&pdev->access_lock);

        /* 第一步：标准速度（兼容 GX2431） */
        ds2484_wait_1wire_idle(pdev);
        ds2484_write_config(pdev, DS2484_REG_CFG_APU);
        ds2484_wait_1wire_idle(pdev);

        err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
        if (err < 0)
                dev_warn(&pdev->client->dev, "ds2484_send_cmd failed\n");

        err = ds2484_wait_1wire_idle(pdev);
        retval = err & DS2484_REG_STS_PPD;

        /* 第二步：标准速度没检测到 → OverDrive（兼容 DS28E25） */
        if (!retval) {
                ds2484_write_config(pdev, DS2484_REG_CFG_APU | DS2484_REG_CFG_1WS);
                ds2484_wait_1wire_idle(pdev);

                err = ds2484_send_cmd(pdev, DS2484_CMD_1WIRE_RESET);
                if (err < 0)
                        dev_warn(&pdev->client->dev, "ds2484_send_cmd failed\n");

                err = ds2484_wait_1wire_idle(pdev);
                retval = err & DS2484_REG_STS_PPD;
        }

        mutex_unlock(&pdev->access_lock);

        return !retval;
  }
```

注意：切换速度后必须调用 ds2484_wait_1wire_idle(pdev) 等待配置生效，否则下一个复位命令可能会被忽略——这是之前动态切换失败的原因。

# 测试结果

| 芯片 |  速度 | 结果 |
|-----------|---------|--------|
| GX2431 | 标准速度 (1WS=0) | ✅ |
| DS28E25 | OverDrive (1WS=1) | ✅ |

两个芯片共用同一个 DS2484 1-Wire 主控，不需要改硬件，也不需要额外加 GPIO。

# w1 search disabled after inserting ds28e25

Root Cause Analysis

  Your scenario told me exactly where the problem was. There are two bugs:

  Bug 1: w1_ds28e25.c:2290 — Search disabled after DS28E25 loads

  sl->master->search_count = 0;   // ← THIS LINE
  The DS28E25 driver permanently disables the W1 bus scan after it registers. The W1 process loop at w1.c:995 checks if (dev->search_count) before each scan — when zero, no scans ever happen again. So a GX2431
  hotplugged later is never detected.

  Scenario A: DS28E25 found first → search_count = 0 → scans stop → GX2431 never found ❌
  Scenario B: GX2431 found first → w1_ds2431_add_slave doesn't touch search_count → scans continue → DS28E25 found next ✅ (but then search_count = 0 stops further scans)