# [黑苹果] HP 暗影精灵三 (OMEN 15-ce)

惠普暗影精灵三（OMEN 15-ce 系列）的 OpenCore EFI 引导配置，基于 Intel HD Graphics 630 核显运行，已屏蔽 NVIDIA 独显。适配并实测支持 **macOS Monterey (12) 至 macOS Tahoe (26.x)**。

> ⚠️ 本仓库的 EFI 来自具体的个人机器，内含的 SMBIOS 序列号、MLB、UUID 等信息**仅供参考**。直接套用前请务必生成属于你自己的三码，详见下文「使用方法」。

## 硬件配置

| 部件 | 型号 | 说明 |
| --- | --- | --- |
| CPU | Intel Core i7-7700HQ | Kaby Lake，第 7 代 |
| 核显 | Intel HD Graphics 630 | 主显示输出 |
| 独显 | NVIDIA GeForce GTX 1050Ti | **已屏蔽**（macOS 不支持） |
| 无线网卡 | Intel AX210 | 由板载 Intel 网卡更换而来 |
| 有线网卡 | Realtek RTL8111 | 板载 |
| 声卡 | Realtek ALC295 | **已被物理拆除**（建议外接 USB 声卡） |

## 系统与引导

| 项目 | 内容 |
| --- | --- |
| 引导器 | OpenCore |
| 机型 (SMBIOS) | `MacBookPro16,4` |
| 适配系统 | macOS Monterey (12) ~ macOS Tahoe (26.x)（实测 26.5） |
| SIP | 完全关闭（`csr-active-config = 0x0FFF`） |
| Secure Boot | 关闭（`SecureBootModel = Disabled`） |

### boot-args

```
-no_compat_check -wegnoegpu -igfxblr alcid=1 -ibtcompatbeta
```

| 参数 | 作用 |
| --- | --- |
| `-no_compat_check` | 跳过 macOS 对机型版本的兼容性检查 |
| `-wegnoegpu` | 通过 WhateverGreen 关闭并屏蔽独立显卡（GTX 1050Ti），仅用核显 |
| `-igfxblr` | 修复笔记本屏幕背光寄存器 |
| `alcid=1` | AppleALC 声卡注入布局 ID（layout-id = 1） |
| `-ibtcompatbeta` | 开启 Intel 蓝牙在新版 macOS（macOS 15+ / 26 Tahoe）上的兼容支持 |

## 驱动状态

| 功能 | 状态 | 说明 |
| --- | --- | --- |
| 核显加速 | ✅ | Intel HD 630，`AAPL,ig-platform-id` + framebuffer 补丁 |
| 独显 | 🚫 屏蔽 | macOS 无 NVIDIA 驱动，已通过 `-wegnoegpu` 关闭 |
| 硬盘 (SATA) | ✅ | 通过 DeviceProperties 伪装 Intel 100 系列 AHCI 控制器 |
| 有线网络 | ✅ | RealtekRTL8111 |
| 无线网络 | ✅ | Intel AX210，macOS 26 启用 `AirportItlwm`，支持原生 Wi-Fi 控制 |
| 蓝牙 | ✅ | IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup (v2.7.2) |
| USB 共享网络 | ✅ | 已启用 HoRNDIS，支持安卓手机通过 USB 共享网络 |
| USB | ✅ | USBInjectAll + SSDT-EC-USBX |
| 电池 | ✅ | ACPIBatteryManager + ECEnabler |
| 亮度按键 | ✅ | BrightnessKeys |
| 睡眠 / 唤醒 | ✅ | HibernationFixup |
| 声卡 | ❌ | 硬件已被物理拆除（AppleALC 配置仍保留，建议使用 USB 声卡） |

## EFI 结构

### Kexts（已启用）

```
基础 / SMC      Lilu · VirtualSMC · SMCProcessor · SMCSuperIO
显卡            WhateverGreen
声卡            AppleALC
电池 / EC       ACPIBatteryManager · ECEnabler
睡眠            HibernationFixup
有线网卡        RealtekRTL8111
无线网卡        AirportItlwm（macOS 26 原生 Wi-Fi）
USB 网络共享    HoRNDIS
蓝牙            IntelBluetoothFirmware · IntelBTPatcher · BlueToolFixup (v2.7.2)
USB             USBInjectAll
亮度按键        BrightnessKeys
```

> 仓库内还包含以下**未启用**的备用驱动，可按需在 `config.plist` 中启用：
> - **Wi-Fi 备选驱动**：
>   - `AirportItlwm_Monterey`：macOS Monterey (12) 原生 Wi-Fi 驱动。
>   - `itlwm`：通用驱动，需搭配 [HeliPort.app](https://github.com/OpenIntelWireless/HeliPort) 使用。
>   - `AMFIPass` / `IOSkywalkFamily` / `IO80211FamilyLegacy`：OCLP 旧版无线网络堆栈补丁驱动备用。
> - **其他网卡 / 外设驱动**：
>   - `IntelMausi` / `IntelMausiEthernet` / `LucyRTL8125Ethernet` / `RealtekRTL8100` / `NullEthernet`。
>   - `VoodooPS2Controller`（PS/2 键盘与触控板）。

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
4. **无线网络配置**：
   - **macOS 26 (Tahoe)**：默认已启用适配该版本的 `AirportItlwm.kext`，可直接使用原生系统菜单栏连接 Wi-Fi，无需第三方客户端。
   - **macOS 12 (Monterey)**：请在 `config.plist` 的 `Kernel > Add` 中将 `AirportItlwm` 设为 `Enabled = false`，并将 `AirportItlwm_Monterey` 设为 `Enabled = true`。
   - **HeliPort 方案**：若原生驱动连接不稳定，可将 `itlwm.kext` 设为 `Enabled = true`，并配合安装 [HeliPort.app](https://github.com/OpenIntelWireless/HeliPort)。
5. **USB 网络共享**：已预装并启用 `HoRNDIS.kext`，连接安卓手机并开启手机的「USB 网络共享」即可即插即用作为备用网络。

## 已知问题与注意事项

- **声卡无法使用**：本机 Realtek ALC295 已被物理拆除，因此没有内建音频输出；如需声音可外接 USB 声卡。
- **独显不可用**：NVIDIA GTX 1050Ti 已被屏蔽，无法用于显示或计算。
- **蓝牙缓存重置**：若更换系统版本后蓝牙无法识别或无法打开，建议在 OpenCore 启动菜单执行一次 `Reset NVRAM`（配置中已配置自动清理蓝牙控制器故障标志）。

## 致谢

感谢以下开源项目：

- [Acidanthera / OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)（Lilu、WhateverGreen、VirtualSMC、AppleALC 等）
- [OpenIntelWireless](https://github.com/OpenIntelWireless)（itlwm、AirportItlwm、IntelBluetoothFirmware、HeliPort）
- [laobamac / AirportItlwm](https://github.com/laobamac/AirportItlwm)（适配新版 macOS 的 AirportItlwm 分支）
- 各 Kext 与 SSDT 的原作者与社区贡献者
