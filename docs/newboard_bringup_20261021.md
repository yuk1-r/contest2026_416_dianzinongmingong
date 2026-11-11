# 新板 bring-up 记录（BES2800BP ZE7/JE6 EVB V1.2）

日期：2026-10-21
对象：用户新换的 BES2800BP_ZE7/JE6_EVB_V1.2 主板 + 新圆屏（RM69330 AMOLED 454×454）
目标：从零烧录、验证显示

---

## 一、结论摘要

| 问题 | 结论 |
|---|---|
| 旧板那个 `DSI0 read: RX TIMEOUT` | **新板上不存在**。DSI 读 ID、传帧全部正常 |
| 新板 AP-only 烧录后不启动 | **不是烧录失败**，是该板的 BL 是出厂 `noapp_test` 测试镜像，只跑屏幕自检、不加载 AP |
| 整包烧录后 | **成功**，NuttX + zhaoxi_ui 正常启动，fb0 / 触摸 / 逐帧提交全部正常 |
| 但屏幕完全不亮 | 软件侧一切正常仍无光 → 指向**面板或其发光供电**（未定论） |
| 当前状态 | 已恢复正常启动：VSYS 5V、串口正常、系统运行；**仅屏幕不发光** |

> 重要：中途曾出现“完全不启动”，根因是**旧跳帽接触不良**（UART/电源跳帽未接通），
> 更换全部新跳帽并冷启动后已完全恢复，与固件无关。**唯一遗留问题仍是屏幕不发光。**

---

## 二、时间线与证据

### 2.1 第一次 AP-only 烧录（17:00）

```
image : nuttx_ap.bin 3139224 bytes
sha256: B3B41FAFBDFCA326C4A98F3CA35505772D91BA32A9BB58701E4C0973D902FAD7
PROGRAMMING SUCCEEDED
```

烧录成功，但板子重启后**完全无输出、不进下载模式**。

### 2.2 发现真实原因：出厂测试 BL（17:21）

USB 拔插后串口出现大量输出，关键行：

```
CHIP=best1700
BUILD_DATE=Oct 12 2024 11:11:59
REV_INFO=a8ab17b8f-dirty:noapp_test
FLASH_ID: C8-63-17 SIZE: 0x00800000
TIP: Long press PwrKey to switch between I2C and UART0
dsi_lcdc_test
DSI0 init: width:454, dsimhz:393, lane:1, pixbit:24, pixmhz:34
LCD power status: 08
LCD id: 0x001000
LCD [RM69330] is initialized.
//LCDC: ready for frame transfer.
```

- 该板 BL 是 **`noapp_test`** 出厂测试镜像，只做 DSI 自检并无限刷帧，**不加载 AP**
- 这正对应仓库 `work/backup/v1_working_20260905/README.txt` 的记录：
  > 必须全量烧录（不能只烧 AP），否则 COM20 只有二进制帧

### 2.3 整包烧录（17:25 RESET 同步 → 17:29 完成）

```
nuttx_bl.bin    @ 0x30000000   125,440
nuttx_ota.bin   @ 0x30040000   646,132
nuttx_ap.bin    @ 0x30190000  2,127,680   ← 9/6 v5 已验证镜像
nuttx_apc1.bin  @ 0x30B10000 1,854,396
nuttx_bth.bin   @ 0x28000000   878,524
nuttx_bthcp.bin @ 0x28300000   374,856
nuttx_hifi.bin  @ 0x30D90000 1,543,644
PROGRAMMING SUCCEEDED
```

镜像来源：组委会预编译件（v1 备份与 flash 目录哈希一致）+ `backup/v5_zhaoxi_ui_20260906/nuttx_ap.bin`

### 2.4 启动成功，DSI 正常（17:29）

```
NuttX BES NuttX EVB 0.0.0 dd92bcf4257-dirty Sep  6 2026 18:05:38 arm ap
zhaoxi_ui [11:255]
lv_nuttx_fbdev_set_file: The framebuffer device was opened successfully
  fmt 21 / xres 464 / yres 454 / nplanes 1 / bpp 32
lv_nuttx_touchscreen_create: touchscreen /dev/input0 open success
zhaoxi_ui_main: Zhaoxi UI started!
PANDBG commit frame=1 … frame=7（每秒一帧）
```

对启动日志做 `TIMEOUT|DSI0|ERROR|Failed|assert` 全量扫描：**除一条无害的 fatfs 提示外无任何错误**。

→ **新板的 DSI/面板通信完全正常，旧板的 RX TIMEOUT 不复现。**

### 2.5 屏幕仍然完全不亮

- 用户确认：**完全没光**（不是暗、不是低亮度）
- 厂商 `dsi_lcdc_test` 阶段同样没光
- RM69330 是 **AMOLED**（像素自发光，无背光），需要 1.8V 逻辑 + 3.3V VDD + ELVDD
- 固件侧已在每 2 秒调用 `rm69330_display_on()` 强制点亮（`zhaoxi_ui.c:263`）

### 2.6 之后的硬件状态变化（关键）

1. 用户断开 USB 重插两根橙色 FPC 排线（照片确认已插到底，卡扣压紧）
2. 期间一度出现：**插上 `VSYS`–`VCC` 跳帽时 VSYS 被拉塌到 0V；拔掉才回到 5V**
3. `VCC`–`GND` 电阻（200Ω 档）读数为 `OL` → **开路，无硬短路**
4. 该现象最终定性为 **旧跳帽接触不良**，不是跳帽位置错误。
   **全部更换新跳帽后，`VSYS`–`VCC` 正常插上，VSYS 稳定 5V，板子正常启动。**
   （`VIN`→`VSYS`、`VSYS`→`VCC` 都是允许的配置，不要因为之前的塌压现象而拔掉。）
5. 在跳帽修复前，曾出现：

```
串口 60s 无输出（含 USB 重枚举）
dldtool --ramrun 探测：write=0，无 SYNC
```

→ 当时是 **UART / 电源跳帽未接通**导致"看起来不启动"，与固件无关。
换新跳帽 + 冷启动后系统完全恢复（见 2.7）。

### 2.7 跳帽修复后的正常启动（18:27）

```
CHIP=best1700  KERNEL=NUTTX  FLASH_DUAL_CHIP=1  SW_TYPE=OTA
BUILD_DATE=May 13 2026 10:41:11   REV_INFO=b17130c:nx_bestbsp_ap
DSI0 init: width:454, dsimhz:400, lane:1, pixbit:24, pixmhz:34
//LCDC: TEDBG TE IRQ got #0 / #1  (pin=47)
lv_nuttx_fbdev_set_file: opened successfully (464x454, 32bpp)
PANDBG commit frame=2 … frame=12
```

---

## 三、已排除项

| 怀疑点 | 排除依据 |
|---|---|
| 固件版本/UI 改动 | 9/6 已验证镜像同样不亮 |
| NTC 断言 | 启动日志无 assertion |
| DSI 参数/时序 | 读 ID、传帧、framedone 全部正常 |
| fb0 未注册 | `fbdev_set_file` 成功，464×454×32bpp |
| 触摸初始化 | `/dev/input0` 打开成功 |
| BL 缺失 | 整包烧录后正常启动过 |
| FPC 未插好 | 照片确认插到底、卡扣压紧 |
| VCC 对地短路 | 电阻档 `OL`（开路） |

## 四、未决问题

1. **屏为什么不发光**：软件链路全通但零发光。剩余可能：
   - 面板 VDD/ELVDD 供电路径异常（板上负载开关/LDO 未输出）
   - 面板本体不良（新屏也可能中招）
   - 需在**正常上电状态**下量 `R67`(3.3V)、`R313`、`R18`，与旧板 3.17V / 3.17V / 5.2V 对照
   - **注意**：VSYS=0 时测得的 R67=0V 无效，已作废
2. ~~当前板子不启动~~ → **已解决**：原因是旧跳帽接触不良，换新跳帽 + 冷启动后恢复正常
3. **跳帽配置无需特殊限制**：`VIN`→`VSYS`、`VSYS`→`VCC` 均可正常插上（已验证 VSYS 稳定 5V）

## 四之二、驱动反汇编证据（软件侧已排除）

从 `releases/20261016_003141_rollback_ui_uncond/nuttx_ap.elf` 反汇编 `rm69330_lcdinitialize`
（`0x102ed078`）得到的完整上电时序：

| 命令 | 参数 | 含义 |
|---|---|---|
| `0xF3` | 0x00 | 面板专用配置 |
| `0x35` | 0x00 | 开启 TE（对应日志 `TE IRQ got #n`） |
| `0x44` | 0xE300 | TE 扫描线 |
| **`0x51`** | **0xCF** | **亮度 = 207（约 81%），非 0** |
| `0x2A` / `0x2B` | 4 字节 | 行列地址窗口 |
| `0x11` | — | sleep out，随后 `usleep 200ms` |
| **`0x29`** | — | **display on**，随后 `usleep` |
| `0xDA`/`0xDB`/`0xDC` | 读 1 字节 | 读面板 ID，失败返回错误 |

`rm69330_display_on()`（`0x102ed244`）仅重复发送 `0x29`；`zhaoxi_ui.c` 的
`poll_timer_cb` 每 2 秒调用一次。

→ **亮度、display-on、TE 全部正常发出**，软件侧没有遗漏项。屏不亮只能归因于
面板发光电源（ELVDD/ELVSS）或面板本体。

## 四之三、屏使能到底在哪（回答"是否需要额外操作"）

全仓检索结论：

| 位置 | 是否包含显示使能 |
|---|---|
| `app/zhaoxi_ui/src/zhaoxi_ui.c` | 无（仅 `rm69330_display_on()` → 发 `0x29`） |
| `patches/` 全部补丁 | 无 |
| `vendor/bes/.../include/*.h` | 无（只有 `BOARD_LCDC_WIDTH/HEIGHT` 与时序参数） |
| `defconfig` | 无（仅 `CONFIG_BES_LCD_RM69330=y` / `CONFIG_LCD=y` / `CONFIG_LCDC_TE_IRQ_ENABLE=y`） |
| **预编译 `libbesboard_ap.a` → `board_lcd_initialize()`** | **有，真正的上电/复位时序** |

`board_lcd_initialize`（`0x102ee364`）反汇编：

```asm
bes_configgpio(pinA) ; bes_gpiowrite(pinA, 1)
bes_configgpio(pinB) ; bes_gpiowrite(pinB, 1)
bes_configgpio(pinC) ; bes_gpiowrite(pinC, 0)
bes_configgpio(pinD) ; bes_gpiowrite(pinD, 0)
usleep(100000)                       ; 100 ms 复位保持
bes_gpiowrite(pinC, 1)
bes_gpiowrite(pinD, 1)
bl rm69330_lcdinitialize
```

- 共 **4 个 GPIO**；按 BES 编码（bits0-5 引脚号、bit14 `GPIO_OUTPUT`）解出
  `0x1481A → GPIO26`、`0x1483A → GPIO58`（对应排针 G26 / G58）
- 另外两个由常量偏移得到，预编译对象无行号信息（addr2line 返回 `??:?`），无法反查宏名
- **结论：使能由固件显式完成，启动时自动执行，无需人工操作；但 AMOLED 的 ELVDD/ELVSS 发光线不在固件控制范围内**

由于日志中 `TE IRQ got #0/#1` 正常触发，说明面板已被上电、复位已释放、
MIPI 数据已送达——`board_lcd_initialize` 的 GPIO 序列确实生效了。
剩余缺口指向板上 AMOLED 发光升压电路。

## 四之四、预编译库是否需要单独烧录（验证）

**不需要。** `libbesboard_ap.a` 是静态库，编译期链接进 `nuttx_ap.bin`，
`board_lcd_initialize` / `rm69330_lcdinitialize` / `rm69330_display_on` 的机器码
已经是 AP 镜像的一部分。AP-only 烧录即包含。

用不依赖 PC 相对寻址的指令特征在各版镜像中检索（亮度写入 `0x51=0xCF`、
`display_on` 的 `0x29`）：

| 镜像 | `0x35` init | 亮度 `0x51=0xCF` | `0x29` display_on |
|---|---|---|---|
| v5 9/6（板上运行中） | 0x150422 | 0x150444 | 0x15059E |
| 10/16 release AP | 0x15D0D2 | 0x15D0F4 | 0x15D24E |
| v3 9/6 TLS fix | 0x154BA2 | 0x154BC4 | 0x154D1E |
| v1 9/5 lvgldemo（报告称已点亮） | 0x11053A | 0x11055C | 0x1106B6 |

→ 四版镜像的显示驱动代码一致。**同一份代码在 v1 上亮过、在 v5 上不亮**，
差异只能在硬件侧。

> 检索脚本：`D:\XI\openvela\work\find_lcd_code.py`

## 四之五、v5 AP-only 重烧复验（19:35）

为排除"整包烧录过程中 AP 损坏"的可能，单独重烧一次 v5：

```
image  : nuttx_ap.bin  2,127,680 bytes
sha256 : FF0042ED46ED1D5DB22C16477C414A4BA5CB93F0FDB614F59B9D666AE3F271D9
addr   : 0x30190000
Burn File consumes 12984 ms
PROGRAMMING SUCCEEDED → SYS REBOOT
```

重启后系统正常：

```
PANDBG commit frame=2 … frame=12（每秒一帧）
PANDBG framedone queue_cnt=1 state=4
```

**屏幕依然完全不亮。** 至此"镜像损坏 / 版本不对 / 烧录不完整"三种可能全部排除。

## 四之六、给供应商的问题清单（可直接转发）

1. 板型号 `BES2800BP_ZE7/JE6_EVB_V1.2`，圆屏 RM69330 AMOLED 454×454
2. 现象：**DSI 通信正常、面板 TE 信号正常、framebuffer 正常、逐帧提交正常，但屏幕零发光**
3. 已实测：VSYS 5V、屏侧逻辑电源 1.8V（R68）/3.3V（R67）均正常
4. 固件侧已确认：`0x51=0xCF`（亮度 207）、`0x29`（display on）均已发出；`0x11` 前有 200 ms 延时
5. 出厂 `dsi_lcdc_test` 阶段同样无光
6. **请确认：该板的 AMOLED ELVDD/ELVSS 发光供电是常开，还是由某个 GPIO 使能？若是 GPIO，是哪一个？**
7. 另一块同型板的现象是**读面板 ID（0xDA）超时**，请一并判断是否为屏批次问题

## 四之七、显示接口引脚定义与最终测量结论

从 V1.3 原理图提取的 `BM14B-24DS`（24 针显示屏接口）定义：

| 脚 | 信号 | 新板实测 |
|---|---|---|
| 2 | **VBAT**（屏内升压输入） | R18 ≈ 5V ✅ |
| 3 | **LCD_VDD**（3.3V） | R67 ≈ 3.3V ✅ |
| 4 | **LCD_VDDIO**（1.8V） | R68 ≈ 1.8V ✅ |
| 6/7 | MIPI_CLK_P/N | 数据在传 ✅ |
| 9/10 | MIPI_D0_P/N | 数据在传 ✅ |
| 14–20 | SPI（CS/CLK/DCX/D1–D3） | 未使用 |
| 22 | **RST** | R35 ≈ 1.8V（已释放）✅ |
| 23 | **TE** | IRQ 持续触发 ✅ |
| 24 | GND | — |

**关键：连接器上没有 ELVDD/ELVSS 引脚。** 这块 AMOLED 的发光升压在屏内部
（FPC 上）由 **VBAT** 升压产生，板上不提供、也无测点可量。

补充：`R42 / R64 / R65 / R66 / R76 / R77 / R79` 等是 **0Ω 串联选项电阻**
（同排还有 `R34/R41/R80/R85` 等 `NC` 未贴装），串在 SPI/MTP 等信号线上，
DSI 模式下信号不活动，两端 0V 属于**正常**，不是故障。

### 结论

| 层级 | 状态 |
|---|---|
| 主板供电（VSYS/VBAT/LCD_VDD/LCD_VDDIO） | 全部正常 |
| 复位释放 | 正常 |
| MIPI 数据 / TE | 正常 |
| 屏内逻辑 | 活着（能回 TE） |
| **屏内发光升压 / 面板本体** | **故障** |

板侧已无可排查项。**判定为屏幕侧不良（屏内升压或面板本体）**，建议换屏/返修。

> ⚠️ 供电提醒：EVB 用户手册写明 VSYS 推荐 **3.8V**（范围 3.6~5.5V）。
> 当前实际用 5V 供电。在更换新屏之前，建议按 3.8V 供电，避免对屏内升压电路
> 造成额外应力——尤其已经连续两块屏不亮的情况下。

## 五、建议的下一步

**优先（硬件）**

1. 确认跳帽全部插好（`VIN`→`VSYS`、`VSYS`→`VCC` 均可）；若 VSYS 被拉塌，先怀疑**跳帽接触不良**，换新跳帽而不是改位置
2. 上电后先确认串口有输出（`PANDBG commit` 恢复即系统正常）
3. 系统正常后立即量 `R67` / `R313` / `R18`（此时读数才有效）
4. 若 `R67` 无 3.3V → 板级显示供电故障；若三路电压都正常但屏仍不亮 → 面板本体故障
5. 交叉验证：把旧板那块屏接到新板上试（反之亦然），一次定死是屏还是板

**暂缓（软件）**

- 在屏不亮的前提下继续改固件没有意义，先解决发光
- 本次烧录用的镜像已归档到本仓 `firmware/`（见 `firmware/README.md`）
- 后续日常更新回到 AP-only：`dldtool.exe 20 --reboot ... -M nuttx_ap.bin`

## 六、本次改动与产物

| 文件 | 说明 |
|---|---|
| `firmware/nuttx_ap_v5_20260906.bin` | 本次烧录的 AP 镜像（2,127,680 字节，SHA256 `FF0042ED…`） |
| `firmware/README.md` | 镜像来源、整包烧录命令、AP-only 更新命令 |
| `firmware/SHA256SUMS.txt` | 校验值 |
| `docs/bringup_logs/fullflash_v5.log` | 整包烧录完整日志（`PROGRAMMING SUCCEEDED`） |
| `docs/bringup_logs/boot_after_fullflash.txt` | 整包烧录后成功启动日志 |
| `docs/bringup_logs/boot_after_jumper_fix.txt` | 跳帽修复后正常启动日志（含 DSI/TE/fb0） |
| `docs/bringup_logs/ap_only_v5_reflash.log` | AP-only 重烧日志 |

未修改任何源码、未改动 `flash/` 目录内文件。

> 说明：整包烧录中除 AP 外的 BL/OTA/APC1/BTH/BTHCP/HIFI 均使用组委会预编译件
> （与 `work/backup/v1_working_20260905/` 内文件 SHA256 一致），不在本仓重复存放。
