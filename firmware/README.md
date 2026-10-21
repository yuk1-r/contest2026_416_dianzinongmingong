# firmware/ — 已验证可启动的 AP 镜像归档

本目录存放**经过真机验证**的 AP 固件镜像，用于回滚与复现。只归档 AP 镜像；
BL / OTA / APC1 / BTH / BTHCP / HIFI 使用组委会预编译件，不在此重复存放。

---

## 1. `nuttx_ap_v5_20260906.bin`

| 项目 | 值 |
|---|---|
| 用途 | AP（应用核）镜像 |
| 大小 | 2,127,680 字节 |
| SHA256 | `FF0042ED46ED1D5DB22C16477C414A4BA5CB93F0FDB614F59B9D666AE3F271D9` |
| 构建日期 | 2026-09-06 18:05:38（`uname` 输出一致） |
| NTC 补丁 | 已打（`pmu_open+0x1a78`，`0xb955 → 0xe011`） |
| 烧录地址 | `0x30190000`（双芯片模式） |
| 验证状态 | 已验证启动：NuttX 进 NSH、`zhaoxi_ui` 启动、`/dev/fb0` 464×454×32bpp、`/dev/input0` 打开成功、逐帧提交正常 |

镜像内含完整的 RM69330 屏驱动（`board_lcd_initialize` / `rm69330_lcdinitialize`
/ `rm69330_display_on`），相关证据见 `docs/newboard_bringup_20261021.md` 第四节。

---

## 2. 烧录方式

### 2.1 常规更新：AP-only

**前提**：板上已有完整的 BL / OTA 引导链。

```powershell
cd flash\vela_2800bp\vela_2800bp
.\dldtool.exe 20 --reboot .\programmer1700_dual.bin `
    --set-dual-chip 1 -M .\nuttx_ap.bin --pgm-rate 2000000
```

- `20` = COM 口号（对应 COM20）
- 板子正常运行时不需按键；若停在等待 SYNC，按一次 **RESET**
- `--reboot` 是「烧录完成后重启」，**不能**让运行中的系统进入下载模式

### 2.2 空板 / 出厂测试 BL：必须先整包

出厂 `noapp_test` 测试引导（串口出现 `REV_INFO=...:noapp_test`、满屏
`frame N transfer` 日志）只跑屏幕自检、**不加载 AP**，此时 AP-only 烧录无效，
必须先整包灌入一次：

```powershell
.\dldtool.exe 20 --reboot .\programmer1700_dual.bin `
    -M .\nuttx_bl.bin --set-dual-chip 1 `
    -M .\nuttx_ota.bin `
    -M .\nuttx_ap.bin `
    -M .\nuttx_apc1.bin `
    -M .\nuttx_bth.bin `
    -M .\nuttx_bthcp.bin `
    --addr 0x30D90000 .\nuttx_hifi.bin --pgm-rate 2000000
```

整包完成后，后续更新即可回到 2.1 的 AP-only。

### 2.3 烧录后重启

烧录结束 `dldtool` 会发 `SYS SHUTDOWN` / `SYS REBOOT`；若板子仍无输出，
**拔插 USB** 冷启动（不要只按 POWER）。

---

## 3. 校验

```powershell
Get-FileHash .\nuttx_ap_v5_20260906.bin -Algorithm SHA256
# 期望 FF0042ED46ED1D5DB22C16477C414A4BA5CB93F0FDB614F59B9D666AE3F271D9
```

或使用本目录的 `SHA256SUMS.txt`：

```bash
sha256sum -c SHA256SUMS.txt
```

---

## 4. 相关记录

- `docs/newboard_bringup_20261021.md` — 2026-10-21 新板 bring-up 全过程、
  测量数据、驱动反汇编证据、最终结论
- `docs/bringup_logs/` — 整包烧录日志与启动日志
