# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

`AI_Pocket` —— 基于 **ESP32-P4** 的口袋 AI 设备硬件设计（KiCad **10.0.3** 工程，无固件代码）。
主控 ESP32-P4（U2）+ ESP32-C6-MINI-1 Wi-Fi/BLE 协处理器（U3，经 SDIO）+ 16MB SPI Flash（U1）+ PDM 麦克风 + MIPI-DSI 屏与电容触摸 + I2S 功放 + microSD + 振动马达，双 DCDC（SY8113B ×2，3.3V/1.2V）供电，锂电充电 + 软件自锁式电源开关。最终面向 **JLCPCB（嘉立创）SMT 装配**。

## ★ 工作纪律（每次分析/改动前必读）

1. **先读手册，再给方案。** 任何涉及芯片/器件/屏/开发板的分析、选型、引脚连接、参数核对，**必须先读 `datasheets/` 下对应资料**（结构见下方「datasheets/ 手册库」），依据手册给结论，禁止凭记忆臆断。
   - 多数 `*.pdf` 旁有**同名 `.md` 速查文档**（引脚定义、关键参数、典型连接，逐节标注原文页码）。**先查 `.md` 速查**，需要更细的电气/时序参数再翻 PDF。
   - ⚠ W25Q128 速查的页码用 datasheet **印刷页码**（阅读器页序 = 印刷页码 + 1），其余分立芯片速查页码 = 阅读器页序。
   - 手册里没有的器件，先按"datasheet 提取"流程补一份速查（见 `hardware-doc-extract` skill），再继续。

2. **选型优先嘉立创（LCSC），优先便宜 + Basic 料。** 需要新器件或替换器件时：
   - 先在 LCSC 找**库存充足、单价低、最好是 Basic（免 feeder fee）** 的料；Extended 料每种约 +$3 一次性 feeder fee，能合并 SKU 就合并。
   - 选定后把 `Footprint` / `LCSC` / `MPN` / `Manufacturer` 等字段补进**原理图器件属性**（以原理图为准）；投产 BOM 由 KiCad 导出到 `production/bom.csv`。
   - 可用 `check-manufacturing` skill 实时核库存与可装配性。

3. **元件库选型顺序：先 KiCad 原生通用件，原生没有才用项目 LCSC 库。** 符号/封装/3D 尽量用 KiCad 内置通用件（便于后续换厂商采购）；原生确实没有的，才用 `easyeda2kicad` 从 LCSC 拉取并统一进 `libs/AI_Pocket_LCSC.*`。

4. **编辑边界：PCB 禁改，原理图只做简单改动，BOM 走 KiCad 插件。**
   - **禁止直接修改 PCB（`AI_Pocket.kicad_pcb`）。** 不在 PCB 编辑器里改布局/布线/铺铜，也不用任何工具（CLI / MCP / 文本 Edit）写 PCB 文件。需要动 PCB 的改动一律交回人工在 KiCad 里做；只读操作（DRC、统计、导出 Gerber、trace 网络）允许。
   - **原理图（`*.kicad_sch`）只允许简单改动**：改属性/值/位号、增删个别元件与连线。按内存约定**只用文本 Edit**，禁用 `mcp__kicad-sch__` 写接口（序列化会损坏 KiCad 10 文件）；复杂结构性重构交回人工。
   - **BOM 通过 KiCad 插件生成，不用 CLI 当出料口。** `kicad-cli sch export python-bom` 仅供核查比对，最终交付 BOM 以 KiCad 插件产物 / `production/bom.csv` 为准（器件字段以原理图属性为准）。

## datasheets/ 手册库

`datasheets/` 下全部条目都是**指向集中资料库 `~/hardware-docs` 的软链接**（库是唯一副本，本仓不版本化任何 datasheet 内容；软链换机/clone 后需库存在才有效，读不到时先 `ls -L` 确认链接有效，库缺失时跑 `hardware-doc-extract` skill 的 `setup_library.sh` 重建锚点）。三组内容：

| 路径 | 内容 | 关键件 |
|---|---|---|
| `_chips/`（逐芯片软链） | 本板分立芯片（13 颗），每颗 `*.pdf` + 同名 `.md` 速查，分别软链到库内 `01.MCU/ESP32-C6`、`03.内存与存储`、`04.电源`、`05.音频`、`06.分立器件` 等分类 | C6 模组、W25Q128 Flash、SY8113B/SY8089A DCDC、SY7301A 背光升压、ETA6003 充电、MAX98357A/NS4168 功放、AO3400/AO3401 MOS、肖特基/TVS |
| `3.2_D320C2403V1_1024x768_MIPI/`（软链） | **MIPI 屏模组**资料 | 屏规格 `D320C2403V1`（+ `.md` 速查）、屏驱动 IC **JD9168S**、**触摸 IC GT967**（`GT9xx` 系列，附 `gt9xx.c/.h` 驱动 + 英文手册）、接线图、RK 平台 timing 参考 |
| `board_FireBeetle2-ESP32-P4-DFR1172/`（软链） | **参考开发板**（DFRobot FireBeetle2 ESP32-P4）资料 | **ESP32-P4 主控手册** `01_ESP32-P4_Datasheet_CN.pdf` + `01_ESP32-P4-keypoints.md`、参考板**原理图 PDF**、板载 MIC（MSM261DGT003）手册、引脚定义图、P4↔屏飞线说明 `FLYWIRE-*.md` |

> ⚠ 易忽略点：**ESP32-P4 主控手册不在 `_chips/`，在 `board_FireBeetle2-*/` 里**（中文版 + keypoints 速查）；**触摸 IC 是 GT967、屏驱动是 JD9168S**——`docs/hardware-config.md` 第 7 节"触摸 IC 型号待确认"可据此落地（唤醒边沿/上拉极性查 GT967 手册）。参考板原理图是本板设计的对照基准。

## 真相源与文档分工（重要，避免改漂移）

- **PCB / 原理图网表是唯一真相源**：位号、阻值、具体引脚连接**一律以最新 `AI_Pocket.kicad_sch`（及其网表）为准**。
- **核查引脚/连接只认 netlist trace**，不要用坐标 flood-fill 猜连接（坐标会误判）。
- `docs/hardware-config.md` —— **只记固件需要、但光看原理图看不出的设计约定与时序**（电源自锁开关机时序、deep-sleep 唤醒源、电源管理纪律、PDM≠I2S、MIPI PHY 供电 2.5V 等踩坑）。**不复述易漂移的位号/阻值/连接**。改硬件后只在这里更新"约定/时序"类内容。
- `production/bom.csv` —— KiCad 导出的投产 BOM（位号/封装/数量/值/LCSC）；其余 BOM 字段（MPN/厂商/Basic-Extended/成本）以原理图器件属性为准。

## 原理图层次结构

根 `AI_Pocket.kicad_sch` 下挂 7 张子图；电源域再嵌套 3 张：

```
AI_Pocket.kicad_sch (根)
├── ESP32_C6        WiFi/BLE 协处理器（SDIO 接 P4）
├── LCD             MIPI-DSI 屏 + 电容触摸
├── MIC             PDM 麦克风
├── Power           电源域（充电/路径管理/自锁）
│   ├── Power_3V3   主 3V3 DCDC（EN=MAIN_EN）
│   ├── Power_1V1   P4 核心 1.1V DCDC
│   └── LCD_Power   背光升压驱动
├── SD              microSD（PMOS 电源切换，低有效导通）
├── Sound           I2S 功放
└── VIB             振动马达（NMOS 低边驱动）
```

> `untitled.kicad_sch` 是孤立文件（无人引用）；`*.bak` / `archive/` / `backups/` / `.history/` 是历史快照，**不要当现状读**。

## 元件库与选型流程

- 项目库：`libs/AI_Pocket_LCSC.{kicad_sym, pretty, 3dshapes}`（easyeda2kicad 拉取的 LCSC 件）+ `libs/Mic_LCSC.*`（麦克风）。
- 项目级 `sym-lib-table` / `fp-lib-table` 只注册了 `AI_Pocket_LCSC`；Espressif 官方库、`P4_LCSC:`、`Mic_LCSC:` 等命名空间注册在**全局** `~/Library/Preferences/kicad/10.0/fp-lib-table`（缺注册会报"找不到 footprint"）。
- 加 LCSC 料：`easyeda2kicad --full --lcsc_id=<Cxxxx>` 拉符号/封装/3D，再修焊盘/合并进 `AI_Pocket_LCSC`。datasheet 抓取：先用 LCSC 真链；本机对 analog.com / st.com 整域不通，需走 CDN 镜像。

## 常用命令（kicad-cli 10.0.3）

```bash
# ERC 电气规则检查
kicad-cli sch erc AI_Pocket.kicad_sch -o erc.rpt

# 导出网表（供核查 / netlist trace）
kicad-cli sch export netlist AI_Pocket.kicad_sch -o AI_Pocket.net

# ⚠ BOM 走 KiCad 插件生成，不用 CLI 当出料口；下面这条仅供核查比对
kicad-cli sch export python-bom AI_Pocket.kicad_sch -o BOM_for_JLC.xml

# PCB DRC
kicad-cli pcb drc AI_Pocket.kicad_pcb -o drc.rpt

# 导出 Gerber（量产）
kicad-cli pcb export gerbers AI_Pocket.kicad_pcb -o gerber/
```

> 也可用 KiCad MCP（`mcp__kicad__*` / `mcp__kicad-sch__*`）做**只读**结构化分析：跑 ERC/DRC、列元件/网络、trace 网表连接。**写操作受工作纪律 4 约束**——PCB 禁改、原理图改动只走文本 Edit（`mcp__kicad-sch__` 写接口会损坏文件）。
