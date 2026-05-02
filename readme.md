# Yoga Book 9 13IRU8 (82YO) Linux Audio Fix / Linux 音频修复

是Deepseek V4搞定的修复，给了它windows提取的固件和`Linux on the Zenbook 14 OLED refresh (UM3402YAR)`提供的方法。提取的固件附带在仓库中了。

我自己不能搞定这些操作，因此亦不能验证修复在一般意义上的正确性，请谨慎参考。

感谢[Jichi Zhang](https://github.com/jichizhang/yoga-9-14imh9-linux?tab=readme-ov-file)提供的参考

This fix is done by deepseek V4 flash, given extracted windows firmware and `Linux on the Zenbook 14 OLED refresh (UM3402YAR)` as ref. Extracted Finwares are also in this repo.

I myself can't do such complex job, which also means I can't tell if this fix works for other devices. Use with caution.

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

### Issue 3: `V1 CRC error` = Missing UEFI Factory Calibration

**Update (2026-05-02):** The `V1 CRC error` is **NOT** caused by a missing/corrupt `.bin` firmware file. It comes from the kernel driver `sound/hda/codecs/side-codecs/tas2781_hda.c` trying to read per-unit speaker calibration data from UEFI variables:

```c
crc = crc32(~0, data, 84) ^ ~0;
if (crc != tmp_val[21]) {
    dev_err(p->dev, "%s: V1 CRC error\n", __func__);  // ← the error you see
    return;
}
```

Each Yoga Book 9 unit is calibrated at the factory (speaker impedance, resonant frequency f0, thermal protection thresholds). This calibration is stored as UEFI variables. If they are missing/corrupt (e.g. after a BIOS update or Linux-only install wiped them), the driver falls back to DSP built-in defaults. The DSP still works, but protection algorithms and impedance compensation may not be optimal.

**Current status:** UEFI calibration data not found on this system. This is likely unresolvable without a Windows dual-boot from the same machine.

### Issue 4: Missing Windows Audio Post-Processing Pipeline

Even with the correct TAS2781 firmware, Windows audio sounds different because the Windows driver stack includes additional APO (Audio Processing Object) layers:

| Windows Driver | Function | Linux Equivalent |
|---------------|----------|------------------|
| `dax3_ext_rtk.inf` | Dolby Atmos spatial audio, EQ, bass enhancement | ❌ None |
| `elevocapo64ext.inf` | Elevoc Whisper 3.0 voice/noise processing | ❌ None |
| `hdx_lenovoext_dolby_wrap...` | Lenovo custom audio config | ❌ None |

The Dolby driver contains per-device tuning XMLs with IEQ curves, volume-dependent loudness compensation, speaker protection thresholds, and virtual bass parameters. These were extracted from `DEV_0287_SUBSYS_17AA3843_PCI_SUBSYS_381E17AA.xml` and converted to EasyEffects presets.

## EasyEffects Presets (Dolby Audio Post-Processing)

To compensate for the missing Windows audio processing, EasyEffects presets have been created from the Dolby tuning data extracted from Windows drivers.

### Installation

```bash
sudo pacman -S easyeffects lsp-plugins-lv2
```

### Import Presets

1. Launch EasyEffects: `easyeffects`
2. **Presets → Import** → Select the JSON file from `easyeffects_presets/`
3. In the **Output** panel, enable all 5 plugins

### Preset Files

| File | Purpose | Dolby Source |
|------|---------|-------------|
| `YogaBook9_Dolby_Output.json` | Daily use / Music (dynamic mode) | IEQ Balanced + Audio Optimizer |
| `YogaBook9_Movie_Output.json` | Movies (less bass, clearer dialog) | IEQ Balanced only, RMS compressor |

### Plugin Chain (5 effects)

| # | Plugin | Dolby Equivalent | Key Parameter |
|---|--------|-----------------|---------------|
| 1 | Bass Loudness | Volume Leveler | `loudness=5.0`, `link=true` (volume-dependent bass compensation) |
| 2 | Bass Enhancer | Virtual Bass | `floor=35Hz`, `scope=200Hz` |
| 3 | Equalizer (20-band IIR) | IEQ + Audio Optimizer | Custom curve from Dolby XML (see below) |
| 4 | Compressor | Speaker Protection Regulator | `threshold=-25dB`, `ratio=4:1` |
| 5 | Crystalizer | IEQ treble harmonics | Adaptive intensity, 13 bands |

### EQ Curve (derived from Dolby IEQ Balanced + Audio Optimizer)

| Freq (Hz) | Gain (dB) | Freq (Hz) | Gain (dB) |
|-----------|----------|-----------|----------|
| 47 | -1.5 | 1688 | +5.6 |
| 141 | +3.4 | 2250 | +4.7 |
| 234 | +7.1 | 3000 | +7.2 |
| 328 | +5.9 | 3750 | +5.2 |
| 469 | +1.9 | 4688 | +2.2 |
| 656 | +0.6 | 5813 | +3.3 |
| 844 | +1.0 | 7125 | +1.1 |
| 1031 | +2.1 | 9000 | -0.4 |
| 1313 | +3.5 | 11250 | -2.8 |
| — | — | 13875 | -8.7 |
| — | — | 19688 | -15.0 |

### Volume-Dependent Bass (Loudness Compensation)

The **Bass Loudness** plugin with `link=true` automatically applies more bass boost at lower system volumes and less at higher volumes — matching the Dolby Volume Leveler behavior. This fixes the "low volume = weak bass" problem.

### Note on UEFI Calibration

The `V1 CRC error` in dmesg is expected and does **not** affect the EasyEffects preset functionality. It indicates missing per-unit factory calibration data in UEFI, which cannot be recreated without the original factory calibration. The DSP falls back to safe defaults.

## Before vs After

| Aspect | Before Fix | After Fix |
|--------|-----------|-----------|
| I2C Interrupt | ❌ `acpi=noirq` causes IRQ loss | ✅ I2C communication normal |
| DSP Coefficients | ❌ File missing, chip uses defaults | ✅ Lenovo custom tuning |
| TAS2781 firmware | ❌ Wrong filename | ✅ `TAS2XXX3843.bin.zst` loaded |
| Dolby EQ/Virtual Bass | ❌ Missing (no Linux equivalent) | ✅ EasyEffects preset |
| Volume-dependent bass | ❌ None | ✅ Bass Loudness plugin |
| Speaker protection | ❌ DSP defaults only | ✅ Compressor + Crystalizer |
| Audio Quality | Low volume, weak bass | Normal volume, punchy bass |

## References

- Gist: [Linux on the Zenbook 14 OLED refresh (UM3402YAR)](https://gist.github.com/masselstine/8fe9634b4c31cef07b8dfab089e4eb38#sound) — Cirrus CS35L41 version, similar naming convention
- TAS2781 driver source: `sound/hda/codecs/side-codecs/tas2781_hda_i2c.c`
- Firmware library: `sound/soc/codecs/tas2781-fmwlib.c`
- EasyEffects: <https://github.com/wwmm/easyeffects>

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

### 问题 3: `V1 CRC error` = UEFI 出厂校准缺失

**更新 (2026-05-02):** `V1 CRC error` **不是**由 `.bin` 固件文件缺失或损坏导致。它源于内核驱动 `sound/hda/codecs/side-codecs/tas2781_hda.c` 尝试从 UEFI 变量读取每台机器的扬声器校准数据：

```c
crc = crc32(~0, data, 84) ^ ~0;
if (crc != tmp_val[21]) {
    dev_err(p->dev, "%s: V1 CRC error\n", __func__);
    return;
}
```

工厂会对每台 Yoga Book 9 进行扬声器单独校准（阻抗、谐振频率 f0、热保护阈值等），校准数据保存在 UEFI 变量中。若这些变量缺失或损坏（如 BIOS 更新或纯 Linux 安装覆盖），驱动会回退到 DSP 内置默认值。DSP 仍能工作，但保护算法和阻抗补偿可能不是最优。

**当前状态：** 系统中未找到 UEFI 校准数据。若无同机器的 Windows 双系统，此问题难以解决。

### 问题 4: 缺失 Windows 音频后处理链路

即使 TAS2781 固件正确加载，Windows 上的音质也更好，因为 Windows 驱动栈有额外的 APO（音频处理对象）层：

| Windows 驱动 | 功能 | Linux 等效 |
|-------------|------|-----------|
| `dax3_ext_rtk.inf` | Dolby Atmos 空间音效、EQ、低音增强 | ❌ 无 |
| `elevocapo64ext.inf` | Elevoc Whisper 3.0 语音/噪声处理 | ❌ 无 |
| `hdx_lenovoext_dolby_wrap...` | Lenovo 定制音频配置 | ❌ 无 |

Dolby 驱动包含每设备的调音 XML（IEQ 曲线、音量依存等响度补偿、扬声器保护阈值、虚拟低音参数）。已从 `DEV_0287_SUBSYS_17AA3843_PCI_SUBSYS_381E17AA.xml` 提取这些数据并转换为 EasyEffects 预设。

## EasyEffects 预设（Dolby 音频后处理补偿）

从 Windows 驱动 Dolby 调音数据创建的 EasyEffects 预设，用于补偿缺失的音频后处理。

### 安装

```bash
sudo pacman -S easyeffects lsp-plugins-lv2
```

### 导入预设

1. 启动 EasyEffects: `easyeffects`
2. **Presets → Import** → 选择 `easyeffects_presets/` 目录下的 JSON 文件
3. 在 **Output** 面板中启用全部 5 个插件

### 预设文件

| 文件 | 用途 | Dolby 来源 |
|------|------|-----------|
| `YogaBook9_Dolby_Output.json` | 日常使用/音乐（动态模式） | IEQ Balanced + Audio Optimizer |
| `YogaBook9_Movie_Output.json` | 电影（低频收敛、对话清晰） | 仅 IEQ Balanced，RMS 压缩器 |

### 插件链（5 个效果器）

| # | 插件 | Dolby 对应功能 | 关键参数 |
|---|------|--------------|----------|
| 1 | Bass Loudness | Volume Leveler | `loudness=5.0`, `link=true`（随音量自动补偿低音） |
| 2 | Bass Enhancer | Virtual Bass | `floor=35Hz`, `scope=200Hz` |
| 3 | Equalizer (20段 IIR) | IEQ + Audio Optimizer | Dolby XML 定制曲线 |
| 4 | Compressor | Speaker Protection Regulator | `threshold=-25dB`, `ratio=4:1` |
| 5 | Crystalizer | IEQ 高频谐波增强 | 自适应强度，13 频段 |

### EQ 曲线（源自 Dolby IEQ Balanced + Audio Optimizer）

| 频率 (Hz) | 增益 (dB) | 频率 (Hz) | 增益 (dB) |
|-----------|----------|-----------|----------|
| 47 | -1.5 | 1688 | +5.6 |
| 141 | +3.4 | 2250 | +4.7 |
| 234 | +7.1 | 3000 | +7.2 |
| 328 | +5.9 | 3750 | +5.2 |
| 469 | +1.9 | 4688 | +2.2 |
| 656 | +0.6 | 5813 | +3.3 |
| 844 | +1.0 | 7125 | +1.1 |
| 1031 | +2.1 | 9000 | -0.4 |
| 1313 | +3.5 | 11250 | -2.8 |
| — | — | 13875 | -8.7 |
| — | — | 19688 | -15.0 |

### 音量依存低音补偿

**Bass Loudness** 插件设为 `link=true` 后，系统音量越低会自动补偿越多低音，音量越高补偿越少——与 Dolby Volume Leveler 行为一致。解决了"低音量 bass 效果差"的问题。

### 关于 UEFI 校准

dmesg 中的 `V1 CRC error` 是预期现象，**不影响** EasyEffects 预设的功能。它仅表示 UEFI 中缺少出厂校准数据，无法在无原始校准数据的情况下重建。

## 差异对比

| 方面 | 修复前 | 修复后 |
|------|--------|--------|
| I2C 中断 | ❌ `acpi=noirq` 导致 IRQ 丢失 | ✅ I2C 正常通信 |
| DSP 系数 | ❌ 文件不存在，芯片用默认值 | ✅ Lenovo 定制调音 |
| TAS2781 固件 | ❌ 文件名错误 | ✅ `TAS2XXX3843.bin.zst` 已加载 |
| Dolby EQ/虚拟低音 | ❌ 缺失（Linux 无等效） | ✅ EasyEffects 预设 |
| 音量依存低音补偿 | ❌ 无 | ✅ Bass Loudness 插件 |
| 扬声器保护 | ❌ 仅 DSP 默认 | ✅ Compressor + Crystalizer |
| 音质 | 低音量、低频不足 | 正常音量、低频有力 |

## 参考

- Gist: [Linux on the Zenbook 14 OLED refresh (UM3402YAR)](https://gist.github.com/masselstine/8fe9634b4c31cef07b8dfab089e4eb38#sound) — Cirrus CS35L41 版本，命名规则类似
- TAS2781 驱动源码: `sound/hda/codecs/side-codecs/tas2781_hda_i2c.c`
- 固件库: `sound/soc/codecs/tas2781-fmwlib.c`
- EasyEffects: <https://github.com/wwmm/easyeffects>
