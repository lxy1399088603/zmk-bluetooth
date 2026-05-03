# Sofle QMK 迁移到 ZMK 蓝牙固件计划

## 1. 可行性结论

可行。

你的当前项目是基于 QMK 的 Sofle 分体键盘固件，PCB 已经按 Pro Micro ATmega32U4 引脚位置设计并使用了两年。现在的目标不是重新设计键盘，而是在保持 PCB、矩阵走线、键位肌肉记忆基本不变的前提下，把主控替换为 SuperMini NRF52840，并获得蓝牙或双模连接能力。

推荐路线：

- 使用两块 SuperMini NRF52840，左右手各一块。
- 固件从 QMK 迁移到 ZMK。
- 不再尝试在当前 QMK 固件中硬加蓝牙。
- 保留当前 VIA 导出的四层键位作为迁移来源。
- 先完成稳定的 BLE 分体键盘，再逐步恢复编码器、宏、RGB、OLED。

不推荐路线：

- 不推荐 Pro Micro RP2040。RP2040 本身没有蓝牙，不能直接满足 BLE 分体键盘需求。
- 不推荐继续以 QMK 作为蓝牙主方案。QMK 对有线键盘和 RP2040 很好，但 nRF52840 BLE 分体键盘更适合 ZMK。

## 2. 目标需求

迁移后的键盘应该满足：

- 保持当前物理按键位置不变。
- 保持当前 `sofle VIA layout.json` 中的四层键位布局。
- 支持蓝牙连接设备。
- 支持多蓝牙设备切换。
- 支持 USB 只作为供电/充电时，仍然通过蓝牙输出按键。
- 尽量支持 USB/BLE 双模输出切换。
- 支持左右手分体无线通信。
- 支持旋钮功能。
- 初始阶段不依赖 VIA，先用 ZMK 源码 keymap 固定布局。

## 3. 关键判断

### 3.1 为什么 SuperMini NRF52840 合适

SuperMini NRF52840 是 Pro Micro footprint 替代板，兼容 nice!nano 思路，具有：

- nRF52840 BLE 能力。
- 3.7V 锂电池接口。
- USB 供电/充电能力。
- 外部 VCC/LED 电源软件控制能力。
- 与 Pro Micro 键盘 PCB 的物理引脚位置兼容。

这正好匹配“PCB 不改，只换主控并增加蓝牙”的需求。

### 3.2 为什么要换 ZMK

ZMK 原生面向蓝牙键盘，尤其适合：

- nRF52840。
- BLE 分体键盘。
- 蓝牙 profile 切换。
- USB/BLE 输出切换。
- 低功耗无线键盘。

本项目后续需要用到的 ZMK 能力：

- `&bt BT_SEL 0..4`：选择指定蓝牙设备。
- `&bt BT_NXT` / `&bt BT_PRV`：切换下一个/上一个蓝牙设备。
- `&bt BT_CLR` / `&bt BT_CLR_ALL`：清除蓝牙配对。
- `&out OUT_BLE`：强制走蓝牙输出。
- `&out OUT_USB`：切换到 USB 输出。
- `&out OUT_TOG`：在 USB/BLE 间切换。
- split central/peripheral：左右手分体通信。
- sensor-bindings：旋钮功能。

参考文档：

- ZMK 蓝牙行为：https://zmk.dev/docs/keymaps/behaviors/bluetooth
- ZMK 输出选择：https://zmk.dev/docs/keymaps/behaviors/outputs
- ZMK keymap：https://zmk.dev/docs/keymaps
- ZMK 硬件说明：https://zmk.dev/docs/hardware

## 4. 当前 QMK 项目中的关键硬件信息

来自 `D:\cwork\keyboard\sofle\config.h`：

```c
#define MATRIX_ROW_PINS { C6, D7, E6, B4, B5 }
#define MATRIX_COL_PINS { B6, B2, B3, B1, F7, F6, C7 }
#define MATRIX_ROW_PINS_RIGHT { C6, D7, E6, B4, B5 }
#define MATRIX_COL_PINS_RIGHT { F6, F7, B1, B3, B2, B6, C7 }
#define ENCODERS_PAD_A { F5 }
#define ENCODERS_PAD_B { F4 }
#define ENCODERS_PAD_A_RIGHT { F4 }
#define ENCODERS_PAD_B_RIGHT { F5 }
#define RGB_DI_PIN D3
#define SOFT_SERIAL_PIN D2
```

注意：

- `C7` 在当前 QMK 中是为了 VIA 编码器映射而加入的假列。
- `C7` 不是实际 Pro Micro 常规可用引脚。
- ZMK 中不应该把 `C7` 当成真实矩阵列。
- ZMK 应该用 encoder sensor 来处理旋钮，而不是继续模拟矩阵按键。

## 5. Pro Micro 到 SuperMini NRF52840 引脚映射

基于你提供的 SuperMini NRF52840 引脚图，以及当前 QMK 的 Pro Micro/AVR pin 命名，可得到第一版迁移映射：

| 当前 QMK AVR pin | Pro Micro 标号 | SuperMini NRF52840 GPIO | 当前用途 |
|---|---:|---:|---|
| `C6` | `D5` | `P0.24` | 矩阵行 0 |
| `D7` | `D6` | `P1.00` | 矩阵行 1 |
| `E6` | `D7` | `P0.11` | 矩阵行 2 |
| `B4` | `D8` | `P1.04` | 矩阵行 3 |
| `B5` | `D9` | `P1.06` | 矩阵行 4 |
| `B6` | `D10` | `P0.09` | 左手列 0 / 右手列 5 |
| `B2` | `D16` | `P0.10` | 左手列 1 / 右手列 4 |
| `B3` | `D14` | `P1.11` | 左手列 2 / 右手列 3 |
| `B1` | `D15` | `P1.13` | 左手列 3 / 右手列 2 |
| `F7` | `D18` | `P1.15` | 左手列 4 / 右手列 1 |
| `F6` | `D19` | `P0.02` | 左手列 5 / 右手列 0 |
| `F5` | `D20` | `P0.29` | 旋钮 A/B 之一 |
| `F4` | `D21` | `P0.31` | 旋钮 A/B 之一 |
| `D3` | `D3` | `P0.20` | RGB 数据 |
| `D2` | `D2` | `P0.17` | 原 QMK 软串口，BLE 分体后不需要 |
| `C7` | 无常规 Pro Micro pin | 无 | QMK/VIA 假列，ZMK 不使用 |

OLED 可能使用的 Pro Micro I2C 脚位：

| Pro Micro 标号 | SuperMini NRF52840 GPIO | 可能用途 |
|---:|---:|---|
| `D1` | `P0.06` | I2C SDA |
| `D0` | `P0.08` | I2C SCL |

OLED 引脚需要根据 PCB 原理图或实测确认后再启用。

## 6. 推荐项目结构

目标目录：`D:\cwork\keyboard\zmk-solfe`

建议结构：

```text
D:\cwork\keyboard\zmk-solfe\
  README.md
  PLAN.md
  build.yaml
  .github\
    workflows\
      build.yml
  config\
    west.yml
    sofle_supermini.conf
    sofle_supermini.keymap
  boards\
    shields\
      sofle_supermini\
        Kconfig.defconfig
        Kconfig.shield
        sofle_supermini.conf
        sofle_supermini.dtsi
        sofle_supermini_left.overlay
        sofle_supermini_right.overlay
```

命名建议：

- 目录沿用你指定的 `zmk-solfe`。
- ZMK shield 内部建议使用 `sofle_supermini`，避免把 `solfe` 拼写带入固件标识。

## 7. 实施阶段

### 阶段 1：最小可用 BLE 分体键盘

目标：先让左右手通过蓝牙组成一个能正常打字的键盘。

任务：

- 创建 ZMK config 项目骨架。
- 创建 `sofle_supermini` shield。
- 定义左右手矩阵行列 pin。
- 去掉 QMK 中的假 `C7` 列。
- 建立 Sofle 的 matrix transform。
- 只迁移基础层，或先做一层用于按键测试的顺序 keymap。
- 编译左右手固件。
- 刷入两块 SuperMini NRF52840。
- 用键盘测试工具确认每个物理按键位置正确。

验收标准：

- 左右手可以组成一个 BLE 分体键盘。
- 所有物理按键都能触发。
- 基础层键位与当前使用习惯一致。
- 暂不要求 RGB、OLED、旋钮、复杂宏完全恢复。

### 阶段 2：完整四层键位迁移

目标：还原当前 VIA JSON 中的四层布局。

任务：

- 以 `sofle VIA layout.json` 为源，转换四层 keymap。
- `KC_TRNS` 转成 `&trans`。
- `KC_NO` 转成 `&none`。
- `TG(1)` 转成 `&tog 1`。
- `TT(2)`、`TT(3)` 第一版建议转成 `&mo 2`、`&mo 3`，保证稳定。
- 如果双击切层是强需求，后续再用 ZMK tap-dance 或自定义 behavior 还原。
- 在工具层加入蓝牙切换和输出切换键。

建议加入的蓝牙/输出按键：

```dts
&bt BT_SEL 0
&bt BT_SEL 1
&bt BT_SEL 2
&bt BT_SEL 3
&bt BT_SEL 4
&bt BT_NXT
&bt BT_PRV
&bt BT_CLR
&out OUT_BLE
&out OUT_USB
&out OUT_TOG
```

验收标准：

- 四层键位基本等价于当前 QMK/VIA 布局。
- 能切换蓝牙设备。
- USB 插着供电时，可以选择继续通过 BLE 输出。

### 阶段 3：旋钮迁移

目标：恢复左右旋钮。

当前 QMK 做法：

- 旋钮转动被模拟成矩阵按键事件。
- VIA 再把这些假按键映射成音量、翻页等功能。

ZMK 推荐做法：

- 把旋钮定义为 sensor。
- 使用 `sensor-bindings` 为每一层定义旋钮动作。
- 初始行为建议：
  - 左旋钮：音量加/减。
  - 右旋钮：Page Up / Page Down。

验收标准：

- 左右旋钮都可用。
- 方向正确。
- 如方向反了，优先通过交换 A/B pin 或调换 sensor binding 修正。

### 阶段 4：宏和特殊行为迁移

目标：恢复当前 QMK 中真正有价值的自定义功能。

当前 QMK 自定义 keycode：

- `ATABF`：Windows Alt-Tab 正向。
- `ATABR`：Windows Alt-Tab 反向。
- `NMR`：窗口移动到右侧显示器。
- `NML`：窗口移动到左侧显示器。
- `SBS`：Shift + Backspace 删除整词。

迁移建议：

- `NMR` / `NML` 先用 ZMK macro 实现：`LSHIFT + LGUI + RIGHT/LEFT`。
- `ATABF` / `ATABR` 先做简单宏版本，后续再考虑是否需要完全复刻 QMK 的 Alt 延迟释放逻辑。
- `SBS` 第一版可以先替换为显式的 `CTRL+BACKSPACE` 或 `CTRL+DELETE` 按键。
- 如果 `Shift + Backspace` 的左右方向删除整词对你非常重要，再单独设计 ZMK combo/hold-tap/custom behavior。

验收标准：

- 窗口移动宏可用。
- Alt-Tab 至少有可用替代。
- 整词删除有可用方案，是否完全复刻由后续体验决定。

### 阶段 5：RGB 和 OLED

目标：在核心输入稳定后恢复视觉功能。

建议顺序：

- 初始固件关闭 RGB 和 OLED。
- 先确认 BLE 分体、键位、旋钮、蓝牙切换都稳定。
- 再研究 RGB 数据脚 `P0.20`。
- 再研究 OLED I2C 脚位。
- 最后研究 SuperMini 的外部 VCC 控制。

电源建议：

- 如果要用电池，RGB/OLED 默认关闭。
- 如果你主要外接电源使用，可以再开启 RGB/OLED。
- 你提供的图中提到 `P0.13` 可控制 3.3V 到 VCC 的供电，后续可以作为 ZMK external power control 的调查对象。

验收标准：

- 关闭 RGB/OLED 时，键盘稳定可靠。
- 开启 RGB/OLED 后，不影响 BLE 分体连接。
- 功耗符合你的使用方式。

## 8. 主要风险

### 风险 1：SuperMini 与 nice!nano 并非固件层完全一致

虽然 SuperMini 是 Pro Micro footprint 兼容，但 bootloader、板级定义、电源控制脚可能与 nice!nano 存在差异。

处理方式：

- 第一版尝试使用 nice!nano 类 board target。
- 如果编译、刷机或 pin 行为不对，再创建 `supermini_nrf52840` 自定义 board。

### 风险 2：矩阵 transform 与当前 VIA keymap 顺序不一致

当前 VIA layout 是 10x7，其中包含假列。ZMK 需要物理按键矩阵和 layout transform 对齐。

处理方式：

- 第一版先做按键位置测试层。
- 不急着一次性迁移全部复杂键位。
- 每个键按实际位置校验。

### 风险 3：旋钮不能照搬 QMK/VIA 模型

ZMK 不需要用 fake matrix column 模拟旋钮。

处理方式：

- 旋钮按 sensor 处理。
- keymap 中用 `sensor-bindings` 设置每层行为。

### 风险 4：QMK 过程式逻辑不能全部直接等价迁移

例如 Alt-Tab 延迟释放、Shift-Backspace 判断左右 Shift，这些在 QMK C 代码里很好写，但 ZMK keymap 更偏声明式。

处理方式：

- 先完成 80% 高频使用能力。
- 对极少数复杂行为单独评估是否需要自定义 ZMK behavior。

### 风险 5：功耗和外设供电

你当前 QMK 固件有 OLED pet、RGB layer lighting 等功能。有线使用没问题，但蓝牙/电池场景下可能影响明显。

处理方式：

- 初始版本关闭 RGB/OLED。
- 后续根据外接电源或电池使用习惯逐项恢复。

## 9. 第一版实现清单

- [x] 创建 ZMK 项目骨架。
- [x] 创建 `sofle_supermini` shield。
- [x] 配置左右手构建目标。
- [x] 配置矩阵 pin。
- [x] 配置 matrix transform。
- [ ] 加入基础测试层。
- [ ] 编译左右手固件。
- [ ] 刷机并验证每个按键。
- [x] 迁移四层 keymap。
- [x] 加入蓝牙 profile 切换键。
- [x] 加入 USB/BLE 输出切换键。
- [x] 加入旋钮 sensor。
- [x] 迁移窗口移动宏。
- [ ] 评估 Alt-Tab 和 Shift-Backspace 是否需要精确复刻。
- [ ] 调查 RGB。
- [ ] 调查 OLED。
- [ ] 调查 `P0.13` 外部 VCC 控制。
- [ ] 写最终刷机、配对、使用说明。

## 10. 推荐的第一版固件范围

第一版固件应该尽量保守：

- 启用 BLE split。
- 启用 USB，方便刷机和必要时有线输出。
- 加入四层结构，但可以先只验证基础层。
- 加入蓝牙切换键。
- 加入输出切换键。
- 暂时关闭 RGB。
- 暂时关闭 OLED。
- 暂时不追求 VIA/ZMK Studio 动态改键。
- 旋钮可以在矩阵验证完成后第二步加入。

第一目标不是一次性复刻所有炫酷功能，而是先得到一把稳定、键位熟悉、能蓝牙连接和切换设备的 Sofle。


