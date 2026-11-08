# ASUS 飞行堡垒6 (FX86FE) Hackintosh EFI

基于 OpenCore 引导的 **华硕飞行堡垒6 FX86FE** 黑苹果 EFI，已在 macOS Sonoma 14.8.9 与 Sequoia 15.8.9 上长期稳定使用，Tahoe 26 处于测试阶段。

![关于本机](assets/about-this-mac.jpg)

> ⚠️ **上传/使用前必读**：本 EFI 中带有我的 SMBIOS 序列号等个人化信息。**使用前请务必用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 重新生成** `PlatformInfo → Generic` 下的 `SystemSerialNumber` / `MLB` / `SystemUUID`（ROM 可用随机值），否则有被苹果封号风险且两台机器序列号冲突。

> 🔧 **UTBMap.kext（USB 端口映射）必须自行重做**：三份 EFI 里的 `UTBMap.kext` 都是我在**自己这台 FX86FE** 上逐口插拔实测出来的映射表，写死了本机的端口号与端口类型。**换任何一台机器（哪怕是同型号）直接套用，都会出现 USB 口失效、设备不识别、睡眠被 USB 唤醒等异常。** 正确做法：用 [USBToolBox](https://github.com/USBToolBox/tool) 在你自己的机器上重新生成 `UTBMap.kext` 并替换（保留 `USBToolBox.kext` 不变），详见 §四.7。

---

## 一、仓库结构（三个系统版本各一套）

| 目录 | 系统 | 状态 | Release |
| --- | --- | --- | --- |
| 仓库根目录（`/BOOT`、`/OC`） | **macOS Sequoia 15.8.9** | ✅ 长期验证，日常主力 | `v15.8.9` |
| `macos14/` | **macOS Sonoma 14.8.9** | ✅ 已验证 | `v14.8.9` |
| `macos26/` | **macOS Tahoe 26** | ⚠️ 测试版，未正式优化 | `v26.0-beta1` |

三套之间**不要混用 kext 与 config**，跨版本的关键差异：

- `AirportItlwm`：14 用 2.3.0（`MaxKernel 23.99.99`）、15 用 24.x（`MaxKernel 24.99.99`）、26 用 2.4.0-alpha（`MinKernel 25.0.0`），区间不重叠，拿错版本 Wi-Fi 直接不加载
- 声卡：26 起 `AppleHDA` 被移除，必须靠 `SimpleHDA.kext`，14/15 不需要（靠 AppleALC 即可）
- SMBIOS：14/15 用 `MacBookPro15,1` 或 `15,2`、26 必须 `MacBookPro16,1`（Tahoe 已淘汰 15,x 机型）
- boot-args：仅 26 需要 `revpatch=sbvmm` + `-ibtcompatbeta` + `-igfxblt` + `-igfxbls`

## 二、适配机型

| 项目   | 说明                                                                                                        |
| ---- | --------------------------------------------------------------------------------------------------------- |
| 实机验证 | 华硕飞行堡垒6 **FX86FE**（2018）                                                                                  |
| 理论适配 | 同模具 FX86FM/FX86FN，以及 8 代 Coffee Lake-H 标压 + UHD 630 + GTX 1050/1050Ti + Realtek 有线 + Intel 无线 的同结构机型（需微调） |
| 不适配  | 仅核显（i5-8250U 等 U 系低压）机型                                                                                   |

## 三、硬件配置

| 硬件   | 型号                                              | macOS 状态                                               |
| ---- | ----------------------------------------------- | ------------------------------------------------------ |
| CPU  | Intel Core i5-8300H（4 核 8 线程，45W，Coffee Lake-H） | ✅ 变频正常（CPUFriend 定制）                                   |
| 核显   | Intel UHD Graphics 630                          | ✅ 硬件加速（Metal 3）                                        |
| 独显   | NVIDIA GTX 1050 Ti 4GB（Pascal）                  | ❌ Sequoia 已无 Pascal 驱动，已隐藏（Windows 下正常）                |
| 内存   | 16GB DDR4 2667MHz                               | ✅                                                      |
| 硬盘   | Toshiba NVMe SSD                                | ✅                                                      |
| 无线网卡 | Intel Wireless-AC 9462                          | ✅ AirportItlwm（Wi-Fi 原生菜单）                             |
| 蓝牙   | Intel Bluetooth                                 | ✅ IntelBluetoothFirmware + BlueToolFixup               |
| 有线网卡 | Realtek RTL8111                                 | ✅                                                      |
| 声卡   | Realtek ALC 系列                                  | ✅ AppleALC                                             |
| 触控板  | I2C HID 触控板                                     | ✅ VoodooI2C + VoodooI2CHID                             |
| 键盘   | PS/2 内置键盘                                       | ✅ VoodooPS2 + BrightnessKeys + AsusSMC（Fn 功能键/背光调节全可用） |
| 电池   | SMCBatteryManager                               | ✅ 电量/健康度显示                                             |
| 屏幕   | 1920×1080 内置面板                                  | ✅ 亮度滑条 + 快捷键均可调                                        |

## 四、系统信息

- macOS **Sequoia 15.8.9**（Intel 机型最后支持的大版本）
- SMBIOS：**MacBookPro15,2**（MacBook Pro 13-inch 2018, Four Thunderbolt 3 Ports）
- 引导：OpenCore（config 已按规范精简）
- 本 EFI 基于板载真实电源表定制 CPUFriendDataProvider，参数按 **i5-8300H（45W）** 生成，而非照搬 MBP15,2 的 28W 表

> 各目录的 SMBIOS 并非统一：`macos14/` 为 `MacBookPro15,1`，根目录（15）为 `MacBookPro15,2`，`macos26/` 必须为 `MacBookPro16,1`（Tahoe 已淘汰 15,x 机型，用 15,x 会装不上）。换用任何一套都要自己生成三码。

## 五、主要修复与优化

### 1. 风扇满速修复（独显供电两全方案）

ASUS 笔记本风扇由 EC（嵌入式控制器）直控，**独显被 `_OFF` 断电后 EC 读不到 GPU 温度会进入保护模式，风扇直接拉满**。因此本 EFI 不采用断电屏蔽，而是：

- `SSDT-Disable_GPU_PEG0.aml` 保持 **Enabled=false**（独显持续供电，EC 可见）
- boot-args 保留 **`-wegnoegpu`**（macOS 图形栈忽略独显，不加载假显示头）

结果：风扇正常、亮度正常、系统干净。

### 2. 控制中心亮度滑条灰色修复

ASUS 的内屏连接器类型默认被核显固件报为 HDMI 类（`02000000`），导致系统把内置屏当外接屏、亮度滑条被禁用。修复：

```
framebuffer-con0-enable = 01000000
framebuffer-con0-type   = 00080000   # eDP 内屏
```

### 3. Fn 功能键 / 背光调节修复（自制 SSDT-ATKD）

ASUS 的 Fn 组合键事件由 EC 通过 ATKD（华硕 ACPI 通知接口）上报，原版 DSDT 的通知链在 macOS 下不完整，导致 **Fn+F6、Fn+F10、Fn+↑/↓ 调节背光全部失效**。修复方案：

- **SSDT-ATKD.aml（自制）**：在核显显示设备 `_SB.PCI0.GFX0.DD1F` 上挂载通知转发，将 EC 的背光事件（SKBV/SLKI）经 `ATKD.IANE` 上报给系统
- **AsusSMC.kext**：接收 ATKD 事件并映射为 macOS 功能键（风扇模式、飞行模式、触控板开关等）
- **BrightnessKeys.kext**：承接亮度调节事件
- **SSDT-ALSD.aml**：在 Darwin 下禁用假环境光传感器（ALSD `_STA=0`），防止"自动调节亮度"与手动调节互相干扰

### 4. 背光寄存器修复

- boot-args `-igfxblr` + DeviceProperties `enable-backlight-registers-fix`，修复 CFL 平台背光寄存器

### 5. DVMT 显存扩容

- `framebuffer-stolenmem 19MB / fbmem 9MB / unifiedmem 4GB`（DVMT 预分配不可改的 ASUS BIOS 通吃方案）

### 6. CPU 电源管理（按 i5-8300H 定制）

- **CPUFriend 1.3.0 + CPUFriendDataProvider**（用 CPUFriendFriend 基于本机 board-id 真实电源表生成）
- LFM 800MHz（0x08）、EPP 0x80（均衡省电）、CPUFloor 800
- 效果：空闲可降到 800MHz~1.5GHz，而不是一直顶着 3.9GHz

### 7. USB 端口定制（⚠️ 换机器必须重做 UTBMap）

三套 EFI 都用 **USBToolBox + UTBMap** 这套免改 SSDT 的方案，两个 kext 分工不同：

| kext | 作用 | 能否直接复用 |
| --- | --- | --- |
| `USBToolBox.kext` | 注入框架，把映射表喂给系统 | ✅ 通用，不用改 |
| `UTBMap.kext` | 真正的端口映射表（Port 号 / 连接器类型 / 内建标记） | ❌ **本机实测生成，必须自己重做** |

**为什么不能直接套用**：`UTBMap.kext` 里写死了每个端口的 Port 编号、类型（USB2 / USB3 / Type-C / Internal）与内建属性。端口号取决于 XHCI 控制器与 RHUB 的 `_ADR` 定义，换机型、换 BIOS 版本、甚至刷过 BIOS 都可能变。映射对不上的典型症状：

- 某个 USB 口完全无反应
- U 盘只认 USB 2.0、不认 3.0
- 蓝牙 / 摄像头 / 读卡器消失
- 合盖睡眠立刻被唤醒（内建设备被当外接）
- USB 网卡、外置声卡掉线

**重做步骤**：

1. 下载 [USBToolBox](https://github.com/USBToolBox/tool)（Windows 用 `USBToolBox.exe`，macOS/Linux 用 Python 版）
2. Windows 下选 `D. Select Ports` → **每个口都插一个 U 盘**（USB 2.0 与 3.0 设备各插一遍，Type-C 正反各一次），确认端口出现后按 `b` 返回
3. 选 `K. Build kext`，输出目录会生成新的 `UTBMap.kext`
4. 用它替换 `OC/Kexts/UTBMap.kext`；`config.plist` 里 `BundlePath` 不变，无需改配置
5. 加载顺序：`USBToolBox.kext` 在前、`UTBMap.kext` 在后（本仓库已排好）
6. 根目录（15）那套额外带 `XHCI-unsupported.kext`，若你的控制器无需兼容补丁可禁用；`macos14/`、`macos26/` 未包含该 kext

### 8. Intel 无网/蓝牙（Sequoia 方案）

- `IOSkywalkFamily + IO80211FamilyLegacy + AirportItlwm`（Wi-Fi 原生界面）
- `IntelBluetoothFirmware + BlueToolFixup + IntelBTPatcher`（蓝牙）

### 9. 其它

- `alcdelay=3000 alctcsel=1`：声卡冷启动/切换修复
- `XHCI-unsupported`：ASUS XHCI 控制器识别
- `ForgedInvariant`：CPU 型号名修正
- `RestrictEvents (revpatch=sbvmm)`：系统更新兼容

## 六、boot-args 现状

```
keepsyms=1 -igfxblr alcdelay=3000 alctcsel=1 -vi2c-force-polling -lilubetaall amfi=0x80 ipc_control_port_options=0 revpatch=sbvmm -wegnoegpu
```

## 七、安装说明

1. BIOS：**关闭 Secure Boot**；如可调，将 DVMT Pre-Allocated 设为 64M（不可调也没关系，已带内存补丁）
2. 将 `EFI` 文件夹放到安装 U 盘或硬盘 EFI 分区（FAT32）
3. **用 GenSMBIOS 重新生成三码**（重要，见顶部警告）
4. **重做 `UTBMap.kext`**（重要，见 §五.7）；不做也能开机，但 USB 端口会错乱
5. 安装后可选：用 [CPUFriendFriend](https://github.com/corpnewt/CPUFriendFriend) 重新生成适合自己 CPU 的 DataProvider
6. 亮度/多媒体快捷键若个别无效，重启一次即可（BrightnessKeys 在唤醒后偶发需重新匹配）

## 八、可选：Win/Ctrl 键互换（个人习惯）

macOS 复制粘贴是 ⌘+C/V，若想让两侧 Ctrl 键都能复制粘贴、Win 键当 Control 用：

```bash
hidutil property --set '{"UserKeyMapping":[{"HIDKeyboardModifierMappingSrc":0x7000000E0,"HIDKeyboardModifierMappingDst":0x7000000E3},{"HIDKeyboardModifierMappingSrc":0x7000000E4,"HIDKeyboardModifierMappingDst":0x7000000E7},{"HIDKeyboardModifierMappingSrc":0x7000000E3,"HIDKeyboardModifierMappingDst":0x7000000E0}]}'
```

持久化可写成 LaunchAgent（`RunAtLoad` 执行上述命令）。

## 九、已知问题

- **USB 端口映射（`UTBMap.kext`）不是通用的**：三套 EFI 里的映射表都是本机实测结果，换机器直接套用会出现端口失效、设备不识别、睡眠被唤醒等问题，必须按 §五.7 重新生成。
- **GTX 1050 Ti 无驱动**：macOS Sequoia 已移除 Pascal 架构驱动（Kepler 之后）。独显保持供电但被 `-wegnoegpu` 隐藏，性能由 UHD 630 承担。剪辑/轻度游戏可用，重度 3D 渲染请回 Windows。
- **风扇模式**：三档风扇（Fn+F5）由 EC 直控且跨系统记忆，macOS 不参与。若想切风扇模式，先回 Windows 切好再重启进 macOS。
- 内屏仅 eDP 输出，若 HDMI 外接出现异常可尝试在 con1/con2 上补 connector-type。

## 十、鸣谢

[OpenCorePkg (acidanthera)](https://github.com/acidanthera/OpenCorePkg) · [Lilu](https://github.com/acidanthera/Lilu) · [WhateverGreen](https://github.com/acidanthera/WhateverGreen) · [VirtualSMC](https://github.com/acidanthera/VirtualSMC) · [AppleALC](https://github.com/acidanthera/AppleALC) · [CPUFriend](https://github.com/acidanthera/CPUFriend) · [AsusSMC](https://github.com/hieplpvip/AsusSMC) · [BrightnessKeys](https://github.com/kprinssu/BrightnessKeys) · [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) · [USBToolBox](https://github.com/USBToolBox/kext) · [itlwm/AirportItlwm](https://github.com/OpenIntelWireless/itlwm) · [IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) · [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) · [CPUFriendFriend](https://github.com/corpnewt/CPUFriendFriend)
