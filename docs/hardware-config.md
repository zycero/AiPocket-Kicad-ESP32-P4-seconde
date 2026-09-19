# AI_Pocket 固件开发约定与 P4 引脚总表

> **面向固件 / 软件开发**：本文给出 ESP32-P4（U2）到所有外设的**完整引脚映射**，以及光看引脚表看不出的设计约定、时序、踩坑。
>
> **真相源**：所有 GPIO 编号、网络名、对端连接均**从最新 `AI_Pocket.kicad_sch` 网表 trace 得到，并用 ESP32-P4 datasheet（QFN104 管脚总览）逐脚交叉验证**。如本文与原理图网表不一致，**以原理图为准**。
>
> 信号统一用 **P4 的 GPIO 编号 + 网络名** 标识；外设位号（U4/U5…）会随原理图微调，仅作参考。

---

## 0. ★本版重大校正说明（看过旧版务必重读）

旧版本文档的 GPIO 编号**大面积错误**（把一套臆测/参考板编号当成了 P4 真实 GPIO）。本版按 **网表 + P4 datasheet 双源**全面校正。**若你已按旧表写过固件，下列每一项都要改**：

| 信号（网络名） | 旧文档（错） | 真实 GPIO | P4 物理 pin |
|---|---|---|---|
| `ESP_SDA` | GPIO7 | **GPIO8** | 8 |
| `ESP_SCL` | GPIO8 | **GPIO7** | 7 |
| `TP_INT` | GPIO4 | **GPIO15** | 16 |
| `TP_RESET` | GPIO5 | **GPIO23** | 25 |
| `LCD_RESET` | GPIO48 | **GPIO27** | 56 |
| `LCD_BL_PWM` | GPIO53 | **GPIO21** | 23 |
| `I2S_DOUT` | GPIO20 | **GPIO1** | 1 |
| `I2S_BCLK` | GPIO21 | **GPIO2** | 2 |
| `I2S_LRCLK` | GPIO22 | **GPIO12** | 13 |
| `SOUND_EN` | GPIO23 | **GPIO13** | 14 |
| `I2S_CLK`（PDM 时钟） | GPIO50 | **GPIO49** | 92 |
| `I2S_DATA`（PDM 数据） | GPIO49 | **GPIO50** | 93 |
| `SDIO_CMD` | GPIO19 | **GPIO39** | 80 |
| `SDIO_CLK` | GPIO18 | **GPIO40** | 81 |
| `SDIO_D0..D3` | GPIO14..17 | **GPIO41 / 42 / 43 / 44** | 82/83/84/86 |
| `SD1_CLK` | GPIO43 | **GPIO30** | 60 |
| `SD1_CMD` | GPIO44 | **GPIO29** | 58 |
| `SD1_D0..D3` | GPIO39..42 | **GPIO31 / 32 / 33 / 28** | 61/63/64/57 |

> 注：`SD1_*` 与 `LCD_RESET` 的「真实 GPIO」列已是**整体下移一位后**的最新值（为避开 GPIO34 JTAG strapping，详见下方风险点②与 §2.8）。

> 编号本来就对、无需改的：`User_LED=GPIO3`、`C6_WAKEUP=GPIO6`、`KEY_DET=GPIO10`、`SYS_HOLD=GPIO11`、`USB1_N=GPIO24`、`USB1_P=GPIO25`、`GPIO_VIB=GPIO26`、`BOOT=GPIO35`、`SD1_PWRN=GPIO45`、`C6_EN=GPIO54`。

**状态更新**（详见 §6、§8）：
1. ✅ **（已解决）** 触摸中断 `TP_INT` 已从 GPIO22 改接 **`GPIO15`（LP IO，pin16）**，现可作 deep-sleep RTC 唤醒源（触摸点屏唤醒），GPIO22 已腾空。
2. ✅ **（已解决）** 原 `SD1_D2` 复用 GPIO34（JTAG strapping）的问题，已通过「SD 总线 + LCD_RESET 整体下移一位」消除：SD 现占 GPIO28~33、`LCD_RESET`→GPIO27，**GPIO34 已腾空**。
3. ⚠ 另：全板**供电 / 上拉 / 电容审查**发现若干待决项（D16 原理图/BOM 型号不一致、晶振 CL 待定、SD 默认上电等；触摸 RESET 上拉已改好），见 **§8**。

---

## 1. 系统组成与电源链路

主控 **ESP32-P4NRW32（U2，32MB PSRAM）**，外挂一颗 **ESP32-C6-MINI-1 模组（U5）** 做 Wi-Fi/BLE 协处理器（经 **SDIO** + esp-hosted 与 P4 通信）。其余：16MB SPI Flash（U1，PY25Q128HA）、I2S 功放（U4，NS4168）、PDM 麦克风（MK1，MSM261DGT003）、MIPI-DSI 屏 + 电容触摸（屏 FPC = J1）、microSD（J5）、振动马达。充电 IC = U3（ETA6003）。

电源链路（按角色，不写位号）：

```
电池 / USB-C ──► 充电+路径管理IC(U3) ──► VSYS(≈3.7~4.4V，常带电)
                                         │
                ┌────────────────────────┼────────────────────────┐
                │                         │                        │
        电源键自锁(SW3)        主3V3 DCDC(EN=MAIN_EN)        背光升压(U7)
                                          ▼
                                    ESP_3V3 (3.3V，主电源轨)
                                          │
                          ├──► P4 的 IO/模拟域、C6、功放、屏 …
                          └──► 核心 DCDC(U6) ──► ESP_VDD_HP(≈1.1V，P4 内核)
```

| 电压轨 | 电压 | 说明 |
|---|---|---|
| `VSYS` | ≈3.7~4.4V | 系统总线，只要电池/USB 有电就带电 |
| `ESP_3V3` | 3.3V | **主电源轨，受电源键自锁电路开关**（EN=`MAIN_EN`）|
| `ESP_VDD_HP` | ≈1.1V | **P4 核心电压，由 P4 内部 DVFS + 外部 DCDC(U6) 闭环管理，固件不要干预** |

> ⚠ `ESP_VDD_HP` 由 P4 的 `FB_DCDC`(pin78) / `EN_DCDC`(pin79) 两脚控制外部 DCDC(U6) 闭环调节，**固件绝对不要去操作这两个信号**（它们不是普通 GPIO）。

---

## 2. ★P4 引脚总表（按外设分组）

> 方向均站在 **P4 视角**。供电域决定该 IO 的电平与是否可做 LP/RTC 唤醒（见 §5、§6）。

### 2.1 电源 / 开关机控制
| 信号 | GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `KEY_DET` | GPIO10 | 11 | 输入 | 电源键分压(R41/R42) | VDD_LP(LP IO) | 按下=**高**；可做 light-sleep 唤醒（LP IO）|
| `SYS_HOLD` | GPIO11 | 12 | 输出 | 经 D18 维持 `MAIN_EN` | VDD_LP(LP IO) | 上电后**第一件事拉高**自锁，否则松手断电（见 §3）|
| `RESET`(CHIP_PU) | — | 103 | — | 复位键 SW1 + 上拉 R2 | — | 非 GPIO，硬件复位脚 |
| `FB_DCDC` | — | 78 | — | U6:FB | — | P4 内核 DCDC 反馈，**勿动** |
| `EN_DCDC` | — | 79 | — | U6:EN | — | P4 内核 DCDC 使能，**勿动** |

### 2.2 ESP32-C6 协处理器（SDIO，对端 U5）
| 信号 | P4 GPIO | pin | 对端 C6 脚 | 供电域 | 固件注意 |
|---|---|---|---|---|---|
| `C6_EN` | GPIO54 | 98 | U5:EN | VDD_IO_6(3V3) | 输出，复位/使能 C6 |
| `C6_WAKEUP` | GPIO6 | 6 | U5:IO2 | VDD_LP(3V3) | 输出，P4 唤醒 C6 |
| `SDIO_CMD` | GPIO39 | 80 | C6 IO18 | VDD_IO_5 | 上拉 R36→3V3 |
| `SDIO_CLK` | GPIO40 | 81 | C6 IO19 | VDD_IO_5 | P4 为 SDIO host |
| `SDIO_D0` | GPIO41 | 82 | C6 IO20 | VDD_IO_5 | 上拉 R35→3V3 |
| `SDIO_D1` | GPIO42 | 83 | C6 IO21 | VDD_IO_5 | 上拉 R34→3V3 |
| `SDIO_D2` | GPIO43 | 84 | C6 IO22 | VDD_IO_5 | 上拉 R33→3V3 |
| `SDIO_D3` | GPIO44 | 86 | C6 IO23 | VDD_IO_5 | 上拉 R31→3V3 |

> C6 侧 `IO18~IO23` 正是 ESP32-C6 esp-hosted **SDIO slave 标准引脚**；C6 固件用 esp-hosted-mcu 默认配置即可对上。⚠ SDIO 所在 `VDD_IO_5` 域由 **P4 内部 LDO（VO4）** 供电（见 §5），固件须把 VO4 配成与 C6/上拉一致的 **3.3V**。

### 2.3 MIPI-DSI 屏 + 背光（屏 FPC = J1）
| 信号 | P4 GPIO/脚 | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `DSI_D0_P/N` | 专用 | 39/40 | — | J1:10/11 | VDD_MIPI_DPHY | lane 0；非 GPIO |
| `DSI_D1_P/N` | 专用 | 35/36 | — | J1:13/14 | VDD_MIPI_DPHY | lane 1；非 GPIO |
| `DSI_CLK_P/N` | 专用 | 38/37 | — | J1:16/17 | VDD_MIPI_DPHY | 时钟；非 GPIO |
| `LCD_RESET` | GPIO27 | 56 | 输出 | J1:31 | VDD_IO_4(3V3) | 屏复位 |
| `LCD_BL_PWM` | GPIO21 | 23 | 输出 | U7(SY7301A 升压):EN/PWM | VDD_IO_0(3V3) | 背光亮度 PWM；**上电默认低=背光关**，由固件点亮 |

> 屏 = D320C2403V1（驱动 IC JD9168S，**2-lane** DSI，需完整 init code）。🔴 **DSI 的 D-PHY 供电 `VDD_MIPI_DPHY`(VO3) 必须配 2.5V**（合法 2.25~2.75V），**绝不能 3.3V**，否则烧 PHY（见 §5、§7）。

### 2.4 触摸（I2C + INT/RESET，屏 FPC = J1）
| 信号 | P4 GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `TP_INT` | GPIO15 | 16 | 输入/中断 | J1:35 + R48(10K)→GND | VDD_LP(3V3) | **LP IO，可做 deep-sleep 触摸唤醒**（见 §6）|
| `TP_RESET` | GPIO23 | 25 | 输出 | J1:34 + R47(10K)→3V3 | VDD_IO_0(3V3) | 触摸复位（低有效）；R47 已上拉到 3V3，符合 GT967 ✅ |

> 触摸 IC = **GT967**（汇顶 GT9xx，I2C），与系统 I2C 共用 `ESP_SDA/ESP_SCL`。上电地址时序见 §4.2。

### 2.5 I2C 系统总线
| 信号 | P4 GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `ESP_SCL` | **GPIO7** | 7 | 双向 | J1:37 + 上拉 R10→3V3 | VDD_LP(3V3) | ⚠ 与旧文档相反 |
| `ESP_SDA` | **GPIO8** | 8 | 双向 | J1:36 + 上拉 R9→3V3 | VDD_LP(3V3) | ⚠ 与旧文档相反 |

> 已带上拉。触摸 GT967 挂在此总线（从机地址 0x14 或 0x5D，见 §4.2）。新增 I2C 设备注意地址不冲突。

### 2.6 I2S 功放（对端 U4 = NS4168）
| 信号 | P4 GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `I2S_DOUT` | GPIO1 | 1 | 输出 | U4:SDATA | VDD_LP(3V3) | 音频数据 → 功放 |
| `I2S_BCLK` | GPIO2 | 2 | 输出 | U4:BCLK | VDD_LP(3V3) | 位时钟（P4 为主）|
| `I2S_LRCLK` | GPIO12 | 13 | 输出 | U4:LRCLK | VDD_LP(3V3) | 帧时钟 |
| `SOUND_EN` | GPIO13 | 14 | 输出 | U4:CTRL + R18→GND | VDD_LP(3V3) | 功放使能/MUTE；**上电默认下拉=静音**，拉高开声 |

### 2.7 PDM 麦克风（对端 MK1 = MSM261DGT003）
| 信号 | P4 GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `I2S_CLK`（实 **PDM_CLK**） | GPIO49 | 92 | 输出 | MK1:CLK | VDD_IO_6(3V3) | ⚠ 网络名是历史命名 |
| `I2S_DATA`（实 **PDM_DATA**） | GPIO50 | 93 | 输入 | MK1:DATA | VDD_IO_6(3V3) | ⚠ 网络名是历史命名 |

> 🔴 **是 PDM，不是 I2S**：MK1:L/R 接地（左声道）。P4 必须配 **PDM-RX 模式**采集，不能按标准 I2S 三线配（见 §7）。

### 2.8 microSD（对端 J5，经 PMOS Q2 电源切换）
| 信号 | P4 GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `SD1_D3` | GPIO28 | 57 | 双向 | J5:CD/DAT3 + 上拉 R25 | VDD_IO_4(3V3) | 兼卡检测 |
| `SD1_CMD` | GPIO29 | 58 | 双向 | J5:CMD + 上拉 R26 | VDD_IO_4(3V3) | |
| `SD1_CLK` | GPIO30 | 60 | 输出 | J5:CLK | VDD_IO_4(3V3) | |
| `SD1_D0` | GPIO31 | 61 | 双向 | J5:DAT0 + 上拉 R21 | VDD_IO_4(3V3) | |
| `SD1_D1` | GPIO32 | 63 | 双向 | J5:DAT1 + 上拉 R22 | VDD_IO_4(3V3) | |
| `SD1_D2` | GPIO33 | 64 | 双向 | J5:DAT2 + 上拉 R24 | VDD_IO_4(3V3) | ✅ 已下移避开 GPIO34 strapping |
| `SD1_PWRN` | GPIO45 | 87 | 输出 | Q2(PMOS):G + R27 | VDD_IO_5 | **低有效导通**：拉低=给 SD 上电，拉高=断电 |

> SD 总线 6 根线已从 GPIO29-34 整体下移到 **GPIO28~33**（避开 GPIO34 JTAG strapping）；P4 SDMMC 走 GPIO matrix，软件按上表实际 GPIO 配置即可。

### 2.9 USB / 振动 / 指示灯 / 启动调试
| 信号 | P4 GPIO | pin | 方向 | 对端 | 供电域 | 固件注意 |
|---|---|---|---|---|---|---|
| `USB1_N` | GPIO24 | 52 | 双向 | USB D−(经 R15) | VDD_IO_4(3V3) | **Full-Speed OTG**（HS 专用脚 pin49/50 未用）|
| `USB1_P` | GPIO25 | 53 | 双向 | USB D+(经 R16) | VDD_IO_4(3V3) | 默认 USB_PU |
| `GPIO_VIB` | GPIO26 | 55 | 输出 | R19→NMOS(Q1) 栅 | VDD_IO_4(3V3) | 振动马达**低边驱动**，高=振动 |
| `User_LED` | GPIO3 | 3 | 输出 | R7→LED | VDD_LP(3V3) | 用户指示灯 |
| `BOOT` | GPIO35 | 66 | 输入 | BOOT 键 SW2 + 上拉 R8 | VDD_IO_4(3V3) | **strapping 启动模式**，非电源键；默认上拉=SPI Boot |
| `GPIO36`(strap) | GPIO36 | 68 | — | 上拉 R5→3V3 | VDD_IO_4 | strapping（启动模式+ROM log），上拉=1；原"待确认"已查明 |
| `GPIO0` | GPIO0 | 104 | 空闲 | 测试点 TP7 | VDD_LP | 未用，引出测试点 |
| `GPIO47` | GPIO47 | 89 | 空闲 | 测试点 TP6 | VDD_IO_5 | 未用，引出测试点 |
| `GPIO34` | GPIO34 | 65 | 空闲 | 悬空（原 SD1_D2 已移走）| VDD_IO_4 | **JTAG strapping**，腾空避坑；建议加 no-connect/测试点，勿接信号 |

### 2.10 P4 内部 Flash（专用脚，对端 U1 = PY25Q128HA）
`FLASH_CS/Q/WP/HOLD/CK/D` = pin 27/28/29/31/32/33，**非 GPIO**，由 P4 flash 控制器驱动；供电 `VDDO_FLASH/VDD_FLASHIO`(VO1)。晶振 `XTAL_N/P` = pin99/100 → Y1（40MHz）+ C7/C8。

---

## 3. 开关机与电源自锁时序 ★固件必读

整机是**软件自锁式电源开关**，被开关的是主电源轨 `ESP_3V3`（通过主 3V3 DCDC 的 EN 脚 `MAIN_EN`）：`MAIN_EN` 高 = 开机供电，低 = 关机断电。`MAIN_EN` 不直连 P4，由电源键(SW3) + `SYS_HOLD` 经二极管共同维持，默认下拉到关。

**时序**

1. **开机（纯硬件）**：按电源键 → VSYS 经二极管注入 `MAIN_EN` → DCDC 上电 → P4 启动。无需固件参与。
2. **自锁保持（固件第一要务）**：🔴 P4 启动后**最早**要做的事 —— 把 `GPIO11 (SYS_HOLD)` 配为输出并拉**高**，经 D18 维持 `MAIN_EN`。**不做这步，用户一松手就立刻断电。**
3. **读键**：自锁后轮询 `GPIO10 (KEY_DET)`，高=按住、低=松开。长按 → 关机；短按 → 自定义。
4. **软关机**：执行关机清理后把 `GPIO11 (SYS_HOLD)` 拉**低**；待电源键也松开，`MAIN_EN` 失去维持被下拉 → 整机下电。
5. **复位不掉电**：`MAIN_EN` 上有保持电容，软复位/看门狗期间短暂跳变不会掉电；复位后需在自锁窗口内重新拉高 `SYS_HOLD`。

```c
void app_main(void) {
    // 1. 上电后第一件事：自锁，否则松手即断电
    gpio_set_direction(GPIO_NUM_11, GPIO_MODE_OUTPUT);   // SYS_HOLD
    gpio_set_level(GPIO_NUM_11, 1);                       // 维持供电

    // 2. 配置电源键读取（高 = 按下）
    gpio_set_direction(GPIO_NUM_10, GPIO_MODE_INPUT);     // KEY_DET

    // ... 正常初始化 ...

    // 3. 主循环：长按关机
    if (key_pressed_for(GPIO_NUM_10, LONG_PRESS_MS)) {
        do_shutdown_cleanup();
        gpio_set_level(GPIO_NUM_11, 0);                   // 放开自锁 → 整机断电
    }
}
```

> **BOOT 键（GPIO35）不是电源键**，它是 P4 的下载/启动 strapping 脚。复位键独立接在 P4 的 `CHIP_PU`（非 GPIO）。

---

## 4. 各外设软件控制要点

### 4.1 I2C（SDA=GPIO8 / SCL=GPIO7）
系统主 I2C，已带上拉。⚠ 注意 **SDA/SCL 与旧文档相反**：`SDA=GPIO8`、`SCL=GPIO7`。

### 4.2 触摸 GT967（I2C）
- **上电地址锁定时序（固件必须做）**：RESET(`GPIO23`) 拉低期间，用 `TP_INT`(`GPIO15`) 电平选 I2C 地址（INT 高→0x14；INT 低→0x5D）；RESET 低 ≥100µs 后拉高，再等 ≥5ms 把 `TP_INT` 切回输入/中断。
- `TP_RESET`(GPIO23) 经 R47 **上拉到 3V3**（符合 GT967 /RSTB external pull-up）；`TP_INT`(GPIO15) 经 R48 对地下拉（地址由固件上电时序主动驱动选）。固件仍须上电后尽早把这两脚配成推挽主动驱动。
- 触摸中断沿由 GT967 寄存器配置；正常工作时 `GPIO15` 作输入读中断。**`GPIO15` 是 LP IO，可兼作 deep-sleep 触摸唤醒源**（见 §6）。

### 4.3 MIPI 屏（D320C2403V1 / JD9168S）
- 图像走 P4 **MIPI-DSI** 专用接口（2-lane：CLK+D0+D1），不占普通 GPIO。
- 复位 `LCD_RESET=GPIO28`；背光 `LCD_BL_PWM=GPIO21`（上电默认灭，固件点亮）。
- 🔴 DSI init 前先用 LDO API 把 `VDD_MIPI_DPHY`(VO3) 设为 **2.5V**（见 §5/§7）。

### 4.4 功放 NS4168（I2S）
标准 I2S 主模式输出：`BCLK=GPIO2`、`LRCLK=GPIO12`、`DOUT=GPIO1`。`SOUND_EN=GPIO13` 默认下拉静音，放音前拉高。

### 4.5 麦克风（PDM）
配 **PDM-RX**：`PDM_CLK=GPIO49`（输出）、`PDM_DATA=GPIO50`（输入）。MK1:L/R 接地=左声道。详见 §7。

### 4.6 SDIO ↔ C6
P4 为 SDIO host，4-bit 模式，引脚见 §2.2。C6 跑 esp-hosted-mcu slave（IO18-23）。上电先 `C6_EN(GPIO54)` 使能 C6，再起 SDIO + esp-hosted。

### 4.7 microSD
上电先把 `SD1_PWRN=GPIO45` 拉**低**给卡上电（PMOS 低有效导通），再初始化 SDMMC；下电拉高断电省电。`SD1_D3=GPIO29` 兼具卡检测。⚠ `SD1_D2=GPIO34` 见 §8。

### 4.8 USB / 振动
USB 走 GPIO24/25 的 **Full-Speed OTG**。振动 `GPIO_VIB=GPIO26` 高电平经 NMOS 低边驱动马达。

---

## 5. P4 内部 LDO 与 IO 供电域（固件配置点）

P4 多个 `VDD_IO_x` 由片内 LDO（VO1~VO4）或外部 `ESP_3V3` 供电。**软件用 `esp_ldo` API 配置 VO1~VO4 输出电压**：

| 内部 LDO | 供给 | 接到的 P4 域 | 目标电压 | 影响 |
|---|---|---|---|---|
| **VO1** | `ESP_LDO_VO1` | VDD_FLASHIO(pin30) / VDDO_FLASH(pin71) | 3.3V | 内部 Flash IO |
| **VO2** | `ESP_LDO_VO2` | VDD_PSRAM_0/1(pin59/67) / VDDO_PSRAM(pin72) | 1.8V | PSRAM |
| **VO3** | `ESP_LDO_VO3` | VDD_MIPI_DPHY(pin41) / VDDO_3(pin73) | 🔴 **2.5V** | **MIPI D-PHY，配错烧 PHY** |
| **VO4** | `ESP_LDO_VO4` | VDDO_4(pin74) / VDD_IO_5(pin85) | 3.3V | **SDIO(GPIO39-48) 的 IO 域**，需与 C6/上拉匹配 |

外部 3.3V 直供的 IO 域（无需软件配电压）：`VDD_IO_0`(GPIO16-23)、`VDD_IO_4`(GPIO24-35)、`VDD_IO_6`(GPIO49-54)、`VDD_LP`(GPIO0-15)。

> 实际电压以 ESP-IDF 该型号默认/示例为准；本表给设计意图。**VO3=2.5V 与 VO4 给 SDIO 域供电是两个最容易被忽略、且配错直接出问题的点。**

---

## 6. Deep-sleep / 唤醒源

- **deep-sleep ≠ 关机**：`ESP_3V3` 不掉电，触摸/外设持续供电。芯片级 deep-sleep ≈ 0.025mA（整板含 C6/LDO 实测更高）。
- **P4 的 RTC/deep-sleep GPIO 唤醒只支持 LP IO（GPIO0~GPIO15）**。本板已用到的 LP IO：
  `GPIO3`(LED)、`GPIO6`(C6_WAKEUP)、`GPIO7/8`(I2C)、`GPIO10`(KEY_DET)、`GPIO11`(SYS_HOLD)、`GPIO15`(TP_INT)、`GPIO1/2/12/13`(I2S/功放)。
- ✅ **触摸唤醒已可用**：`TP_INT` 已改接 `GPIO15`（LP IO），可配置为 deep-sleep RTC 唤醒源，实现「触摸点屏唤醒」。
  → 当前可作 deep-sleep 唤醒输入的有 **`GPIO10`(KEY_DET，电源键)** 与 **`GPIO15`(TP_INT，触摸)**。

---

## 7. 固件踩坑提示（硬件相关）

- **麦克风是 PDM，不是 I2S**：网络名虽叫 `I2S_CLK`/`I2S_DATA`，但必须配 **PDM-RX 模式**采集，不能按标准 I2S 三线配置。（后续可能把网络名改为 `PDM_CLK`/`PDM_DATA`。）
- **MIPI D-PHY 供电（VO3）必须 2.5V**，不能用 3.3V 默认值，否则超 PHY 上限损坏。
- **SDIO 的 IO 域（VDD_IO_5）由内部 LDO VO4 供电**，固件须确保 VO4 = 3.3V 与 C6/上拉匹配，否则 SDIO 电平异常。
- **QSPI Flash 需 QE=1**：若按 4 线 QSPI 跑，bootloader/eFuse 需保证 Flash 的 QE 位置 1（PY25Q128HA 出厂状态以 datasheet 为准）。
- **USB 走 Full-Speed OTG（GPIO24/25）**，高速 PHY 专用脚（pin49/50）未用。

---

## 8. 硬件待改 / 待确认

**风险（建议硬件评审）**
- ✅ **（已解决）触摸唤醒域**：`TP_INT` 已从 GPIO22 改接 `GPIO15`（LP IO），可做 deep-sleep 唤醒；GPIO22 已腾空。
- ✅ **（已解决）`SD1_D2` 不再用 GPIO34**：SD 总线 + `LCD_RESET` 已整体下移一位（SD→GPIO28~33、`LCD_RESET`→GPIO27），GPIO34 腾空，彻底避开 JTAG strapping。**建议给腾空的 GPIO34 加 no-connect 标志或留测试点，勿接信号。**
- ⚠ **GPIO37/38 strapping 脚悬空**（pin69/70 未连接）：默认 SPI Boot（GPIO35 上拉=1）不受影响，但浮空 strapping 建议确认无误采样风险。

**保留的电气待改项（详见评审）**
- [ ] ⚠ 晶振 C7/C8 当前 **15pF**（待定）：曾有换晶振计划 —— **YXC X252040MMB4SI(C2885592, CL=10pF)** + C7/C8 改 **12pF**（原 BOM_JLC.md 记录，该文件已删，决策保留于此），但**未落实**——原理图与 `production/bom.csv` 仍是 **Q22FA12800332 + 15pF**。需定方向：① 换 YXC(CL=10pF) → C7/C8 取 **12pF**；② 保持 Q22FA12800332 → 先查其 CL（CL=10pF 同样取 ~12pF，CL=15pF 才取 ~22pF）。**定案后把原理图 / production/bom.csv 一并对齐。**
- [x] ✅ **（已解决）触摸 `TP_RESET` 上拉**：R47 已从对地下拉改为 **上拉到 ESP_3V3**（=触摸 VDD 同轨，J1:39 TP_VDD 亦在此轨），符合 GT967 手册 p.5/p.20 的 /RSTB external pull-up 要求。`TP_INT`(GPIO15) R48 保留对地下拉（地址由固件主动驱动选）。

**供电 / 上拉 / 电容 全板审查产出（本轮）**
- ✅ **D16 已定 B5819W**：原理图与 `production/bom.csv` 一致（B5819W / SOD-123 / C8598，与振动续流 D8 共用 SKU）。屏背光 6 串 LED、boost 正常输出仅 ~18V（屏 SPEC §6.5），B5819W 40V 余量充足。〔曾考虑 SS36(60V) 在背光开路 OVP(37V) 工况余量更大，已弃用；正常工作用不到 OVP。〕
- 🟡 **VBUS TVS D5**：当前为 ESD 管(0402ESDA-05N)，浪涌钳位偏高；ETA6003 IN 绝对最大仅 6V。建议换工作电压 5~5.5V、钳位 <6V 的功率 TVS。
- 🟡 **SD 电源默认态**：Q2(PMOS) 栅极 R27=51K **对地下拉→默认导通→SD 默认上电**。若意图「默认断电省电」，栅极应改上拉到源极(3V3)、由 GPIO45 拉低导通；否则 GPIO45 复位/浮空时 SD 一直带电（deep-sleep 漏电）。**确认固件电源策略**。
- 🟡 **充电电流 213mA**：R13(ISET1)=4.7K → I≈1000/4700=213mA，大电池偏慢；USB_DET 接 GND 固定 AC 模式，R14(ISET2)=4.7K 成死电阻（可删省料）。确认是否刻意限流。
- 🟡 **充电状态无指示**：ETA6003 STATS_9 悬空；D2 红灯直连 3V3=常亮电源灯（非充电指示）；D3 绿灯=User_LED(GPIO3)。如需充电指示需把 STATS 引到 LED/GPIO。
- 🟢 **P4 复位 RC 偏短**：R2=10K + C12=100nF → τ=1ms；P4 strapping tH≥3ms，建议 C12 → **1µF**（τ=10ms）给足上电余量。
- 🟢 **NS4168(U4) VDD 储能**：仅 1µF+100nF，D 类功放峰值电流大，建议就近补 **10~22µF** MLCC。
- 🟢（可选）U8 FB 并 22pF 前馈电容优化负载瞬态；C6 的 EN 电容 C51 100nF→1µF 贴合 datasheet。

**Layout 必查（netlist 看不出就近性）**
- P4 各内部 LDO 输出电容(VO1~VO4) 与 VDD_HP core 电容必须**就近**对应供电脚。
- C6 模组 3V3(pin3) 旁需独立 **10µF+0.1µF** 就近去耦（WiFi TX 峰值 ~380mA），否则突发会拉低主 3V3。

**✅ 审查确认正确（无需改）**：主 3V3=3.316V(R45/R46=30.1K/6.65K)、背光 40mA(R38=7.5R)、P4 各供电域去耦完整无裸脚、Flash/C6 strapping、I2C 上拉 2.2K、USB Type-C CC 5.1K、SD 数据线 10K 上拉齐全、振动续流 D8 极性正确、振动栅极 R19=1K/R20=10K 下拉、NS4168 CTRL 默认静音下拉、电源自锁电路齐全。

**位号订正提示**：C6 模组实际位号为 **U5**（旧记 U3 有误，U3 是充电 IC ETA6003）。建议同步更新 `CLAUDE.md`。
