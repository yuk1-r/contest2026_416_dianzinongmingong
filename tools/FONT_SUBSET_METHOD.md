# 运行时豆腐修复 — 方法说明与实测（2026-10-11，尺寸拆分版）

## 问题
commit 05b2ef3d 起对话日志会绘制自由 LLM 回复（CHAT_OUT.txt），语音也会创建提醒。
这些文本不可能被"扫源码/扫资源"的字模工具预测，超出子集的中文字在屏上显示为豆腐块。
旧 check 脚本仍 PASS，因为它检查的正是同一份源码字符集 —— 检查与真实威胁模型脱节。

## 关键观察：自由文本只走 font_20
读 zhaoxi_ui.c 的字体分配：
- font_20 = 天气/提醒/对话日志/输入框/设置正文/导航文字/状态/日期 → **所有自由文本**
- font_22 = 问候语，7 条硬编码模板（源码可提取）→ 窄子集足够
- font_24 = 设置标题"设备信息"+导航图标（FA 符号）→ 窄子集足够
- font_28 = 只 LV_FONT_DECLARE 从未赋值；时钟实际用 lv_font_montserrat_48 → **死重量，直接不生成**

因此基础字表只需进 font_20。

## 修复（加法式，不改团队既有默认行为）
- `tools/gen_cjk_fonts.ps1` 新增可选参数：
  `-BaseSetFile`（GB2312 一级字表 3755 字）、`-SamplesFile`（自由文本样例）、
  `-BaseSizes`（哪些尺寸加基础字表，交付值 "20"）、`-SizeList`（生成哪些尺寸，交付值 "20,22,24"）。
  默认空 → 不传参时行为与原版完全一致。
- `tools/check_cjk_coverage.ps1` 新增同名参数，按尺寸分别要求：
  base 尺寸须覆盖 源码∪资源∪余量∪基础字表∪样例；窄尺寸只须覆盖源码集。
- 修正字符范围过滤器：补 U+00B0–U+00B7 与 U+2014–U+2026（°·—… 等）。
  原过滤器只认 CJK 区，导致设置页"朝夕 AI 助手 · 演示版"的 `·`(U+00B7) 一直豆腐。
- 新增 `tools/gb2312_l1.txt`、`tools/runtime_samples.txt`。

## 交付配置与实测体积（bpp4, --no-compress）
生成命令：
```
gen_cjk_fonts.ps1 -BaseSetFile tools/gb2312_l1.txt -SamplesFile tools/runtime_samples.txt -BaseSizes 20 -SizeList 20,22,24
```
| 配置 | 编译 .o 合计 | 链接 bin/main | LV_FONT_FMT_TXT_LARGE |
|---|---|---|---|
| 原版窄子集（有豆腐）| 306 KB | 1.96 MB | 0 |
| 基础字表进全部 4 档 | 4.10 MB | 5.76 MB | **必须 1**（28px 超 20-bit 位域）|
| **交付：base 只进 20，弃 28** | **889,728 B ≈ 0.87 MB** | 2.54 MB | **0（不需要）** |

分档：font_20(base) 743,680 + font_22(narrow) 67,776 + font_24(narrow) 78,272。
比"全档基础字表"省 78%；仅为原版窄子集的约 2.9 倍，却彻底消除自由文本豆腐。

## 固件注意
- 交付配置下 **不需要** 改 LV_FONT_FMT_TXT_LARGE（font_20 的 bitmap 索引 < 2^20）。
  仅当把基础字表加到 24/28 档时才必须置 1。
- 板端 app 的 Makefile/源列表需去掉 zhaoxi_font_28.c（已不生成；zhaoxi_ui.c 里那条
  LV_FONT_DECLARE 是无害 extern，可顺手删）。

## 验证（PC 模拟器 release/v9.6）
- 覆盖检查：font_20 base PASS（3761/3761，含二级探针字 蜇 U+8707）；font_22/24 narrow PASS（99/99）。
- 渲染无豆腐：tile0 自由文本（骤/伞/带/喇/叭/蜇）、tile2 标题/设置/`·` 全部正常出字。
- 圆屏几何审计 ZHAOXI_GEO：objs=36 fail=0 warn=6 worst_r=220，与基线一致（字模变更未破坏布局）。

## 已知取舍
- GB2312 二级生僻字（除显式加入样例者外）仍会豆腐；全表覆盖需体积再翻倍，不建议。
- 6 个 GEO WARN 是元素侵入团队自设 SAFE_MARGIN=12（r>215），仍在物理圆内，非本次引入。

## 对话页输入改造：键盘 → 长按语音（PTT，2026-10-11 追加）
按需求移除对话页的屏幕键盘/输入框/发送按钮，改为整条"按住 说话"长按条（push-to-talk）。

- 交互：`LV_EVENT_PRESSED` 进入聆听（条变橙、文案"聆听中… 松开发送"）；
  `RELEASED` 提交转写并回到待机文案；`PRESS_LOST`（滑出条外）取消不提交。
- 与板端语音通道的文件契约（`/data/ai_agent/`）：
  - `PTT`：UI 按住期间写 "1"，松开/取消写 "0"。agent 的 ASR 据此开关麦克风。
  - `VOICE_IN.txt`：agent 把转写文本写入；UI 在松开时读取并删除，作为本次输入提交。
  - 提交文本沿用既有 `CHAT_IN.txt` 桥（"<seq>\n<text>"），agent 回复走 `CHAT_OUT.txt`。
- 模拟器（`#ifndef __NuttX__`）无真实 ASR：`voice_take_transcript()` 轮转 3 条样例句，
  便于无板验证整条链路；板端走真实文件读取。
- 布局：提醒条限高 2 行（220×48 @y36，超出打点），对话日志 300×108 @y88，
  长按条 292×56 圆角 @y200。键盘移除后日志区加高，仍与长按条/导航不重叠。
- 字模联动：新文案"按住 说话 / 聆听中… 松开发送"属源码可提取字符，
  **改 UI 字符串后必须重跑 gen_cjk_fonts.ps1**，否则窄子集（font_22）缺字显示豆腐。

### 验证（PC 模拟器 release/v9.6）
- 覆盖检查：font_20 base / font_22 / font_24 narrow 全 PASS（窄集 121/121，含新文案字）。
- 渲染：tile1 待机显示"按住 说话"无豆腐、无键盘残留；模拟一次按压后日志出现
  "> 明天傍晚有骤雨，记得带伞"，无豆腐、与提醒条不重叠。
- 链路：按压后 `/data/ai_agent/PTT`="0"、`CHAT_IN.txt`="1\n明天傍晚有骤雨，记得带伞"，
  即转写确实经既有桥提交给 agent。
- 圆屏几何审计 ZHAOXI_GEO：objs=33 fail=0 warn=4 worst_r=220，PASS
  （键盘三件移除、长按条加入后布局仍在圆内）。
