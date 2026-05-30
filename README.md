# [黑苹果] HP 暗影精灵三 (OMEN 15-ce)

惠普暗影精灵三（OMEN 15-ce 系列）的 OpenCore EFI 引导配置，基于 Intel HD Graphics 630 核显运行，已屏蔽 NVIDIA 独显。

> ⚠️ 本仓库的 EFI 来自具体的个人机器，内含的 SMBIOS 序列号、MLB、UUID 等信息**仅供参考**。直接套用前请务必生成属于你自己的三码，详见下文「使用方法」。

## 硬件配置

| 部件 | 型号 | 说明 |
| --- | --- | --- |
| CPU | Intel Core i7-7700HQ | Kaby Lake，第 7 代 |
| 核显 | Intel HD Graphics 630 | 主显示输出 |
| 独显 | NVIDIA GeForce GTX 1050Ti | **已屏蔽**（macOS 不支持） |
| 无线网卡 | Intel AX210 | 由板载 Intel 网卡更换而来 |
| 有线网卡 | Realtek RTL8111 | 板载 |
| 声卡 | Realtek ALC295 | **已被物理拆除** |

## 系统与引导

| 项目 | 内容 |
| --- | --- |
| 引导器 | OpenCore |
| 机型 (SMBIOS) | `MacBookPro16,4` |
| 推荐系统 | macOS Monterey (12) 及更新版本 |
| SIP | 完全关闭（`csr-active-config = 0x0FFF`） |
| Secure Boot | 关闭（`SecureBootModel = Disabled`） |

### boot-args

```
-no_compat_check -wegnoegpu -igfxblr -igfxbls alcid=1
```

| 参数 | 作用 |
| --- | --- |
| `-no_compat_check` | 跳过 macOS 对机型版本的兼容性检查 |
| `-wegnoegpu` | 通过 WhateverGreen 关闭并屏蔽独立显卡（GTX 1050Ti），仅用核显 |
| `-igfxblr` | 修复笔记本屏幕背光寄存器 |
| `-igfxbls` | 平滑背光（亮度）调节曲线 |
| `alcid=1` | AppleALC 声卡注入布局 ID（layout-id = 1） |

## 驱动状态

| 功能 | 状态 | 说明 |
| --- | --- | --- |
| 核显加速 | ✅ | Intel HD 630，`AAPL,ig-platform-id` + framebuffer 补丁 |
| 独显 | 🚫 屏蔽 | macOS 无 NVIDIA 驱动，已通过 `-wegnoegpu` 关闭 |
| 硬盘 (SATA) | ✅ | 通过 DeviceProperties 伪装 Intel 100 系列 AHCI 控制器 |
| 有线网络 | ✅ | RealtekRTL8111 |
| 无线网络 | ⚠️ | Intel AX210，需配合 **HeliPort.app** 使用（itlwm 驱动） |
| 蓝牙 | ✅ | IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup |
| USB | ✅ | USBInjectAll + SSDT-EC-USBX |
| 电池 | ✅ | ACPIBatteryManager + ECEnabler |
| 亮度按键 | ✅ | BrightnessKeys |
| 睡眠 / 唤醒 | ✅ | HibernationFixup |
| 声卡 | ❌ | 硬件已被物理拆除（AppleALC 配置仍保留） |

## EFI 结构

### Kexts（已启用）

```
基础 / SMC      Lilu · VirtualSMC · SMCProcessor · SMCSuperIO
显卡            WhateverGreen
声卡            AppleALC
电池 / EC       ACPIBatteryManager · ECEnabler
睡眠            HibernationFixup
有线网卡        RealtekRTL8111
无线网卡        itlwm（需 HeliPort）
蓝牙            IntelBluetoothFirmware · IntelBTPatcher · BlueToolFixup
USB             USBInjectAll
亮度按键        BrightnessKeys
```

> 仓库内还包含一些**未启用**的备用驱动，可按需在 `config.plist` 中开启：
> `AirportItlwm_Monterey`（无线，原生菜单栏，无需 HeliPort）、`IntelMausi` / `LucyRTL8125Ethernet` / `RealtekRTL8100` / `NullEthernet`（其他有线网卡）、`VoodooPS2Controller`（PS/2 键鼠）、`HoRNDIS`（USB 网络共享）。

### ACPI (SSDT)

| 文件 | 作用 |
| --- | --- |
| `SSDT-PLUG.aml` | 启用 CPU 电源管理（XCPM plugin-type） |
| `SSDT-PNLF.aml` | 添加背光控制设备（亮度调节） |
| `SSDT-EC-USBX-LAPTOP.aml` | 笔记本嵌入式控制器 (EC) 与 USB 供电 (USBX) |
| `SSDT-XOSI.aml` | 伪装 `_OSI` 返回值，兼容 Windows ACPI 调用 |
| `SSDT-RMNE.aml` | 伪造内建网卡设备，确保网络相关服务可用 |

### Drivers

| 文件 | 作用 |
| --- | --- |
| `OpenRuntime.efi` | OpenCore 运行时支持（必需） |
| `OpenCanopy.efi` | 图形化启动菜单 |
| `HfsPlus.efi` | HFS+ 文件系统驱动 |
| `ResetNvramEntry.efi` | 在启动菜单中添加「重置 NVRAM」项 |

### Tools

| 文件 | 作用 |
| --- | --- |
| `OpenShell.efi` | UEFI Shell |

## 使用方法

1. **挂载 EFI 分区**：使用 [OpenCore Configurator](https://mackie100projects.altervista.org/) 或 `MountEFI` 挂载磁盘的 EFI 分区。
2. **替换 EFI**：将本仓库的 `EFI` 文件夹复制到 EFI 分区根目录。
3. **生成自己的三码（重要）**：用 [OpenCore 官方工具 `macserial`](https://github.com/acidanthera/OpenCorePkg) 或 OpenCore Configurator 为 `MacBookPro16,4` 生成新的序列号、主板序列号 (MLB)、SystemUUID 和 ROM，填入 `config.plist` 的 `PlatformInfo > Generic`，避免与他人冲突、影响 Apple 服务登录。
4. **无线网络**：本配置使用 `itlwm`，需安装 [HeliPort.app](https://github.com/OpenIntelWireless/HeliPort) 来连接 Wi-Fi。若想使用系统原生的 Wi-Fi 菜单，可改为启用 `AirportItlwm_Monterey`（注意与系统版本匹配）。

## 已知问题与注意事项

- **声卡无法使用**：本机 Realtek ALC295 已被物理拆除，因此没有内建音频输出；如需声音可外接 USB 声卡。
- **独显不可用**：NVIDIA GTX 1050Ti 已被屏蔽，无法用于显示或计算。
- **Wi-Fi 依赖 HeliPort**：`itlwm` 不接管系统原生 Wi-Fi 菜单，开机后需通过 HeliPort 连接网络。

## 致谢

感谢以下开源项目：

- [Acidanthera / OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)（Lilu、WhateverGreen、VirtualSMC、AppleALC 等）
- [OpenIntelWireless](https://github.com/OpenIntelWireless)（itlwm、AirportItlwm、IntelBluetoothFirmware、HeliPort）
- 各 Kext 与 SSDT 的原作者与社区贡献者
