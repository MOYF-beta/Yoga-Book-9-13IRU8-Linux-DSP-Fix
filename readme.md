# Yoga Book 9 13IRU8 (82YO) Linux Audio Fix / Linux 音频修复

是Deepseek V4搞定的修复，给了它windows提取的固件和`Linux on the Zenbook 14 OLED refresh (UM3402YAR)`提供的方法。提取的固件附带在仓库中了。

感谢[Jichi Zhang](https://github.com/jichizhang/yoga-9-14imh9-linux?tab=readme-ov-file)提供的参考

This fix is done by deepseek V4 flash, given extracted windows firmware and `Linux on the Zenbook 14 OLED refresh (UM3402YAR)` as ref. Extracted Finwares are also in this repo.

Thank [Jichi Zhang](https://github.com/jichizhang/yoga-9-14imh9-linux?tab=readme-ov-file)'s ref

**[English](#english) | [中文](#中文)**

---

<a id="english"></a>

## System Information

| Item | Value |
|------|-------|
| Model | Lenovo Yoga Book 9 13IRU8 (82YO) |
| BIOS | LENOVO-82YO-YogaBook913IRU8-LNVNB161216 |
| Kernel | 6.19.10-arch1-1 |
| HDA Codec | Realtek ALC287 (PCI SSID `17aa:3843`) |
| Smart Amplifier | TI TAS2781 (ACPI `TIAS2781`) |

## Hardware Architecture

```
PCI: 00:1f.3 Intel cAVS (8086:51ca)
  └─ HDA: Realtek ALC287
       ├─ Speaker out: 0x17 (line_out)
       ├─ Speaker out: 0x14 (line_out)  ← Subwoofer
       └─ I2C: TIAS2781 Smart Amplifier
            ├─ Firmware: /lib/firmware/ti/audio/tas2781/TAS2XXX*.bin
            └─ DSP tuning params: Embedded in firmware file
```

## Root Cause Analysis

### Issue 1: `acpi=noirq` Causes I2C Controller Unavailable

**in my case I used acpi=noirq, boot parameter includes is not enabled by default so this can be skipped.**

**Symptoms:**
```
intel-lpss 0000:00:15.1: can't find IRQ for PCI INT B
intel-lpss 0000:00:15.1: probe with driver intel-lpss failed with error -22
```

**Cause:** The kernel boot parameter includes `acpi=noirq`, which disables ACPI IRQ routing. The TAS2781 amplifier on I2C controller #1 (`8086:51e9`) relies on IRQ for communication.

**Solution:** Remove `acpi=noirq` and replace it with `pci=biosirq` (to avoid other potential interrupt issues).

### Issue 2: DSP Coefficient Firmware Filename Mismatch

**Symptoms:**
```
tas2781-hda i2c-TIAS2781:00: tas2781_apply_calib: V1 CRC error
```
No `fw_state = TASDEVICE_DSP_FW_ALL_OK` message in dmesg.

**Cause:** The TAS2781 HDA driver constructs the firmware filename using the following code:
```c
snprintf(tas_priv->coef_binaryname, sizeof(tas_priv->coef_binaryname),
         "TAS2XXX%04X.bin",
         lower_16_bits(codec->core.subsystem_id));
```
The HDA codec's PCI SSID is `17aa:3843`, so the driver actually requests `TAS2XXX3843.bin`.

However, the linux-firmware package only provides `TAS2XXX3881.bin` (named after the ACPI subsystem `17AA3881`). The driver cannot find the correct file and falls back to chip default parameters, resulting in poor audio quality.

**Solution:** Copy the `TAS2XXX3881.bin` (Lenovo custom tuning firmware) extracted from the Windows driver as `TAS2XXX3843.bin`.

## Instructions

### 1. Fix Kernel Boot Parameters

Edit `/etc/default/grub`:

```bash
sudo sed -i 's/acpi=noirq/pci=biosirq/g' /etc/default/grub
sudo update-grub
```

Verify the changes:
```bash
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub
# Confirm output no longer contains acpi=noirq
```

### 2. Replace DSP Coefficient Firmware

Extract the TAS2781 firmware files from the Windows driver (`C:\Windows\System32\csaudio`, `C:\Windows\System32\DriverStore`, etc.). For Yoga Book 9 13IRU8:

```bash
# Backup original firmware
cp /lib/firmware/TAS2XXX3881.bin.zst /home/user/TAS2XXX3881.bin.zst.linux-firmware-backup

# Install Windows firmware (extract beforehand)
sudo cp /path/to/extracted/TAS2XXX3881.bin /lib/firmware/ti/audio/tas2781/TAS2XXX3881.bin
sudo zstd --rm /lib/firmware/ti/audio/tas2781/TAS2XXX3881.bin

# Create the filename the driver actually requests
sudo cp /path/to/extracted/TAS2XXX3881.bin /lib/firmware/ti/audio/tas2781/TAS2XXX3843.bin
sudo zstd --rm /lib/firmware/ti/audio/tas2781/TAS2XXX3843.bin
```

### 3. Reboot and Verify

```bash
sudo reboot
```

After reboot, check:
```bash
sudo dmesg | grep -i tas2781
```

Expected output (no V1 CRC error):
```
snd_hda_codec_alc269 ehdaudio0D0: bound i2c-TIAS2781:00 (ops tas2781_hda_comp_ops)
```

For more debugging info:
```bash
sudo modprobe snd-hda-scodec-tas2781-i2c dyndbg=+p
sudo dmesg -w | grep tas2781
```

## Firmware Reference

| File | Purpose | Source |
|------|---------|--------|
| `TAS2XXX3843.bin.zst` | DSP coefficients (tuning params) | Extracted from Windows driver |
| `TAS2XXX3881.bin.zst` | DSP coefficients (alternate name) | Extracted from Windows driver |
| `TIAS2781RCA2.bin.zst` | RCA calibration/config data | linux-firmware package |
| `TAS2XXXFirmware.bin` | Base firmware | Windows driver |

`TAS2XXX3843.bin` contains Lenovo's custom DSP configuration for Yoga Book 9 13IRU8, including:
- Speaker EQ curves
- Protection algorithm parameters
- Gain/limiter settings
- Subwoofer crossover points

## Before vs After

| Aspect | Before Fix | After Fix |
|--------|-----------|-----------|
| I2C Interrupt | ❌ `acpi=noirq` causes IRQ loss | ✅ I2C communication normal |
| DSP Coefficients | ❌ File missing, chip uses defaults | ✅ Lenovo custom tuning |
| `V1 CRC error` | ❌ Present | ✅ Gone |
| Audio Quality | Low volume, weak bass | Normal volume, punchy bass |

## References

- Gist: [Linux on the Zenbook 14 OLED refresh (UM3402YAR)](https://gist.github.com/masselstine/8fe9634b4c31cef07b8dfab089e4eb38#sound) — Cirrus CS35L41 version, similar naming convention
- TAS2781 driver source: `sound/hda/codecs/side-codecs/tas2781_hda_i2c.c`
- Firmware library: `sound/soc/codecs/tas2781-fmwlib.c`

---
---

<a id="中文"></a>

## 系统信息

| 项目 | 值 |
|------|-----|
| 型号 | Lenovo Yoga Book 9 13IRU8 (82YO) |
| BIOS | LENOVO-82YO-YogaBook913IRU8-LNVNB161216 |
| 内核 | 6.19.10-arch1-1 |
| HDA Codec | Realtek ALC287 (PCI SSID `17aa:3843`) |
| 智能放大器 | TI TAS2781 (ACPI `TIAS2781`) |

## 硬件架构

```
PCI: 00:1f.3 Intel cAVS (8086:51ca)
  └─ HDA: Realtek ALC287
       ├─ Speaker out: 0x17 (line_out)
       ├─ Speaker out: 0x14 (line_out)  ← 低音扬声器
       └─ I2C: TIAS2781 Smart Amplifier
            ├─ 固件: /lib/firmware/ti/audio/tas2781/TAS2XXX*.bin
            └─ DSP 调音参数: 内嵌于固件文件中
```

## 根因分析

### 问题 1: `acpi=noirq` 导致 I2C 控制器不可用

**acpi=noirq 这个启动参数不是默认的，我自己添加了它。这一步应该可以跳过**

**症状：**
```
intel-lpss 0000:00:15.1: can't find IRQ for PCI INT B
intel-lpss 0000:00:15.1: probe with driver intel-lpss failed with error -22
```

**原因：** 内核启动参数包含 `acpi=noirq`，禁用 ACPI IRQ 路由。而 I2C 控制器 #1 (`8086:51e9`) 上的 TAS2781 放大器依赖 IRQ 进行通信。

**解决：** 移除 `acpi=noirq`，替换为 `pci=biosirq`（避免其他可能的中断问题）。

### 问题 2: DSP 系数固件文件名与驱动不匹配

**症状：**
```
tas2781-hda i2c-TIAS2781:00: tas2781_apply_calib: V1 CRC error
```
dmesg 中无 `fw_state = TASDEVICE_DSP_FW_ALL_OK` 信息。

**原因：** TAS2781 HDA 驱动通过以下代码构造固件文件名：
```c
snprintf(tas_priv->coef_binaryname, sizeof(tas_priv->coef_binaryname),
         "TAS2XXX%04X.bin",
         lower_16_bits(codec->core.subsystem_id));
```
HDA codec 的 PCI SSID 是 `17aa:3843`，所以驱动实际请求 `TAS2XXX3843.bin`。

但 linux-firmware 包只提供了 `TAS2XXX3881.bin`（命名来源于 ACPI 子系统 `17AA3881`），驱动根本找不到正确文件，回退到芯片默认参数，导致音质不佳。

**解决：** 将 Windows 驱动中提取的 `TAS2XXX3881.bin`（Lenovo 专用调音固件）复制为 `TAS2XXX3843.bin`。

## 操作步骤

### 1. 修复内核启动参数

编辑 `/etc/default/grub`：

```bash
sudo sed -i 's/acpi=noirq/pci=biosirq/g' /etc/default/grub
sudo update-grub
```

验证修改后参数：
```bash
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub
# 确认输出不再包含 acpi=noirq
```

### 2. 替换 DSP 系数固件

从 Windows 驱动（`C:\Windows\System32\csaudio`、`C:\Windows\System32\DriverStore` 等）提取 TAS2781 固件文件。对于 Yoga Book 9 13IRU8，需要：

```bash
# 备份原版固件
cp /lib/firmware/TAS2XXX3881.bin.zst /home/user/TAS2XXX3881.bin.zst.linux-firmware-backup

# 放入 Windows 版固件（需提前提取）
sudo cp /path/to/extracted/TAS2XXX3881.bin /lib/firmware/ti/audio/tas2781/TAS2XXX3881.bin
sudo zstd --rm /lib/firmware/ti/audio/tas2781/TAS2XXX3881.bin

# 创建驱动实际请求的文件名
sudo cp /path/to/extracted/TAS2XXX3881.bin /lib/firmware/ti/audio/tas2781/TAS2XXX3843.bin
sudo zstd --rm /lib/firmware/ti/audio/tas2781/TAS2XXX3843.bin
```

### 3. 重启验证

```bash
sudo reboot
```

重启后检查：
```bash
sudo dmesg | grep -i tas2781
```

正常输出应类似（无 V1 CRC error）：
```
snd_hda_codec_alc269 ehdaudio0D0: bound i2c-TIAS2781:00 (ops tas2781_hda_comp_ops)
```

如有更多调试信息，可开启驱动调试：
```bash
sudo modprobe snd-hda-scodec-tas2781-i2c dyndbg=+p
sudo dmesg -w | grep tas2781
```

## 固件说明

| 文件 | 用途 | 来源 |
|------|------|------|
| `TAS2XXX3843.bin.zst` | DSP 系数（调音参数） | Windows 驱动提取 |
| `TAS2XXX3881.bin.zst` | DSP 系数（备用命名） | Windows 驱动提取 |
| `TIAS2781RCA2.bin.zst` | RCA 校准/配置数据 | linux-firmware 包 |
| `TAS2XXXFirmware.bin` | 基础固件 | Windows 驱动 |

`TAS2XXX3843.bin` 包含 Lenovo 为 Yoga Book 9 13IRU8 定制的 DSP 配置，包括：
- 扬声器 EQ 曲线
- 保护算法参数
- 增益/限制器设置
- 低音扬声器分频点

## 差异对比

| 方面 | 修复前 | 修复后 |
|------|--------|--------|
| I2C 中断 | ❌ `acpi=noirq` 导致 IRQ 丢失 | ✅ I2C 正常通信 |
| DSP 系数 | ❌ 文件不存在，芯片用默认值 | ✅ Lenovo 定制调音 |
| `V1 CRC error` | ❌ 存在 | ✅ 消失 |
| 音质 | 低音量、低频不足 | 正常音量、低频有力 |

## 参考

- Gist: [Linux on the Zenbook 14 OLED refresh (UM3402YAR)](https://gist.github.com/masselstine/8fe9634b4c31cef07b8dfab089e4eb38#sound) — Cirrus CS35L41 版本，命名规则类似
- TAS2781 驱动源码: `sound/hda/codecs/side-codecs/tas2781_hda_i2c.c`
- 固件库: `sound/soc/codecs/tas2781-fmwlib.c`
