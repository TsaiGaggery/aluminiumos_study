# ES9356 SoundWire Codec Integration

## PTL (Panther Lake) - Full Driver Integration, Topology Build, and Audio HAL Configuration

### Intel Panther Lake Platform - Android GKI Kernel
### Everest Semiconductor ES9356 SDCA Codec (Multi-Function)
### SOF (Sound Open Firmware) + SoundWire Bus
### February 2026

---

## Table of Contents

1. [Background Concepts](#1-background-concepts)
2. [Hardware Identification](#2-hardware-identification)
3. [Kernel Driver Integration](#3-kernel-driver-integration)
4. [SOF Topology Creation](#4-sof-topology-creation)
5. [DRAS HAL Integration](#5-dras-hal-integration)
6. [ALSA Configuration Files](#6-alsa-configuration-files)
7. [Deployment and Verification](#7-deployment-and-verification)
8. [Known Issues and Workarounds](#8-known-issues-and-workarounds)
9. [Kernel Command Line](#9-kernel-command-line-audio-related)
10. [Complete File Reference](#10-complete-file-reference)

---

## 1. Background Concepts

This section explains the ES9356-specific concepts. For general background on SOF, SoundWire,
SDCA, ALH copiers, and the Frontend/Backend DAI architecture, refer to the
[PTL GCS RT712+RT1320 Audio Fix](PTL_GCS_RT712_RT1320_Audio_Fix.md) document, Section 1.

### 1.1 ES9356: A Multi-Function SDCA Codec

Unlike the RT712+RT1320 combination where functions are split across two chips, the ES9356
is a **single-chip multi-function** codec. It exposes **4 DAI endpoints** over a single
SoundWire link, handling headphone output, speaker amplification, headset microphone input,
and digital microphone capture all in one device:

```
  ES9356 Codec (Single Chip, 4 Functions)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Endpoint 0 (aif1): SimpleJack Playback ──► Headphone Output    │
  │                                              ┌───┐              │
  │                                              │DAC│──► HP OUT    │
  │                                              └───┘              │
  │                                                                  │
  │  Endpoint 2 (aif2): SimpleJack Capture  ◄── Headset Mic Input   │
  │                                              ┌───┐              │
  │                                              │ADC│◄── MIC IN    │
  │                                              └───┘              │
  │                                                                  │
  │  Endpoint 3 (aif3): SmartAmp Playback   ──► Speaker Amplifier   │
  │                                              ┌───┐              │
  │                                              │AMP│──► SPK OUT   │
  │                                              └───┘              │
  │                                                                  │
  │  Endpoint 1 (aif4): SmartMic Capture    ◄── Digital Microphone  │
  │                                              ┌───┐              │
  │                                              │PDM│◄── DMIC IN   │
  │                                              └───┘              │
  │                                                                  │
  │  SoundWire Slave Interface ◄───── SoundWire Link 3 ────── CPU   │
  └──────────────────────────────────────────────────────────────────┘
```

Each endpoint becomes a separate backend DAI link in the kernel, and each gets its own
frontend PCM device defined in the topology. This is analogous to having 4 separate codec
chips, but all addressed over a single SoundWire link.

### 1.2 Physical DMIC Architecture: PCH vs SoundWire DMIC

A critical discovery during integration: the physical DMIC microphones on the PTL evaluation
board are **NOT connected to the ES9356 codec**. Instead, they connect to the **PCH
(Platform Controller Hub)** via the camera module:

```
  Physical DMIC Wiring on PTL Eval Board:

  ┌────────────────────────────────────────────────────────────────────┐
  │  Intel PTL CPU                                                     │
  │                                                                    │
  │  Audio DSP                                                         │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │                                                              │  │
  │  │  SoundWire Link 3 ────────── ES9356 Codec                    │  │
  │  │    ├── BE 0: SimpleJack Playback  ──► Headphone              │  │
  │  │    ├── BE 1: SimpleJack Capture   ◄── Headset Mic            │  │
  │  │    ├── BE 2: SmartAmp Playback    ──► Speaker                │  │
  │  │    └── BE 4: SmartMic Capture     ◄── ES9356 PDM_DIN         │  │
  │  │                                        (no physical mic!)    │  │
  │  │                                                              │  │
  │  │  PCH DMIC Pins ◄──── Camera Module ◄──── Physical DMIC 🎤   │  │
  │  │    ├── BE 5: dmic01 (48kHz)                                  │  │
  │  │    └── BE 6: dmic16k (16kHz)                                 │  │
  │  │                                                              │  │
  │  │  HDA HDMI                                                    │  │
  │  │    ├── BE 7: iDisp1                                          │  │
  │  │    ├── BE 8: iDisp2                                          │  │
  │  │    └── BE 9: iDisp3                                          │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────────┘
```

This means the topology must include **both**:
- **SoundWire DMIC pipeline** (PCM 4): Captures from ES9356 aif4 endpoint (no physical mic)
- **PCH DMIC pipeline** (PCM 10): Captures from the physical microphones via PCH PDM pins

The Android `InternalMic` audio path is routed to PCM 10 (PCH DMIC), not PCM 4 (SDW DMIC).

### 1.3 Backend DAI ID Namespaces

When both SoundWire codec endpoints and PCH DMIC coexist, the backend DAI IDs must not
overlap. The IDs are assigned in a fixed order:

```
  Backend DAI ID Assignment:

  SDW endpoints (assigned first by sof_sdw machine driver):
    BE 0 ── SOC_SDW_JACK_OUT_DAI_ID  ── Headphone playback
    BE 1 ── SOC_SDW_JACK_IN_DAI_ID   ── Headset mic capture
    BE 2 ── SOC_SDW_AMP_OUT_DAI_ID   ── Speaker amplifier
    (BE 3 ── reserved for AMP feedback, unused here)
    BE 4 ── SOC_SDW_DMIC_DAI_ID      ── SoundWire DMIC capture

  PCH DMIC endpoints (assigned next):
    BE 5 ── DMIC0_ID  ── dmic01 (48kHz, primary)
    BE 6 ── DMIC1_ID  ── dmic16k (16kHz, secondary)

  HDMI endpoints (assigned last):
    BE 7 ── HDMI1_ID  ── iDisp1
    BE 8 ── HDMI2_ID  ── iDisp2
    BE 9 ── HDMI3_ID  ── iDisp3
```

**Critical**: If `DMIC0_ID` and `DMIC1_ID` are not explicitly set in the topology build
parameters, they default to 4 and 5, which **collides** with the SDW DMIC BE ID of 4. This
was solved by explicitly setting `DMIC0_ID=5,DMIC1_ID=6,HDMI1_ID=7,HDMI2_ID=8,HDMI3_ID=9`
in the build.

### 1.4 Topology Filename Suffix Convention

The kernel's SOF HDA driver automatically appends a DMIC channel suffix to the topology
filename based on the ACPI NHLT (Non-HD Audio Link Table) DMIC count:

```c
// sound/soc/sof/intel/hda.c (lines 1378-1391)
if (tplg_fixup && dmic_fixup && mach->mach_params.dmic_num) {
    tplg_filename = devm_kasprintf(sdev->dev, GFP_KERNEL,
                                   "%s%s%d%s",
                                   sof_pdata->tplg_filename,
                                   i2s_mach_found ? "-dmic" : "-",
                                   mach->mach_params.dmic_num,
                                   "ch");
}
```

For SoundWire machines (not I2S), the separator is `"-"` and suffix is `"{N}ch"`:

```
  ACPI match table base name:  sof-ptl-es9356.tplg
  NHLT reports 2 DMICs:        + "-" + "2" + "ch"
  Actual filename loaded:      sof-ptl-es9356-2ch.tplg
```

This is why the ACPI match table specifies `sof-ptl-es9356.tplg` but the actual topology
file deployed to the device must be named `sof-ptl-es9356-2ch.tplg`.

### 1.5 DRAS (Desktop Remote Audio Service) Architecture

DRAS is the Android Audio HAL for Intel desktop platforms. It is the bridge between Android's
AudioFlinger and the ALSA kernel subsystem. For the ES9356, DRAS must:

1. **Detect** which codec is present on the SoundWire bus
2. **Find** the correct configuration directory (`sof-soundwire_es9356/`)
3. **Read** `audio_config.xml` to learn PCM device mappings
4. **Apply** `mixer_paths.xml` defaults to ALSA mixer controls
5. **Route** audio streams to the correct PCM devices at runtime

```
  DRAS Audio Flow:

  ┌─────────────────────────────────────────────────────────────────────┐
  │  Android Framework                                                  │
  │                                                                     │
  │  App ──► AudioFlinger ──► DRAS HAL (com.android.hardware.audio.    │
  │                               │      desktop APEX)                  │
  │                               │                                     │
  │                               ▼                                     │
  │                     ┌──── environment.rs ────┐                      │
  │                     │                        │                      │
  │                     │  1. detect_audio_      │                      │
  │                     │     codecs_from_       │                      │
  │                     │     system()           │                      │
  │                     │     Scans:             │                      │
  │                     │     /sys/bus/          │                      │
  │                     │      soundwire/        │                      │
  │                     │      devices/*/        │                      │
  │                     │      modalias          │                      │
  │                     │     → finds "p9356"    │                      │
  │                     │     → pushes "es9356"  │                      │
  │                     │                        │                      │
  │                     │  2. get_audio_config_  │                      │
  │                     │     dir_from_detected_ │                      │
  │                     │     codecs()           │                      │
  │                     │     Builds candidate:  │                      │
  │                     │     "sof-soundwire_    │                      │
  │                     │      es9356"           │                      │
  │                     │     Finds match in     │                      │
  │                     │     /vendor/etc/audio/ │                      │
  │                     │                        │                      │
  │                     └────────────────────────┘                      │
  │                               │                                     │
  │                               ▼                                     │
  │                     /vendor/etc/audio/sof-soundwire_es9356/         │
  │                     ├── audio_config.xml  (PCM mapping)             │
  │                     └── mixer_paths.xml   (mixer defaults)          │
  │                               │                                     │
  │                               ▼                                     │
  │                     ALSA Kernel Interface                           │
  │                     ├── hw:sofsoundwire,0   (Jack Out)              │
  │                     ├── hw:sofsoundwire,1   (Jack In)               │
  │                     ├── hw:sofsoundwire,2   (Speaker)               │
  │                     ├── hw:sofsoundwire,10  (Internal Mic/PCH DMIC) │
  │                     ├── hw:sofsoundwire,5   (HDMI1)                 │
  │                     ├── hw:sofsoundwire,6   (HDMI2)                 │
  │                     └── hw:sofsoundwire,7   (HDMI3)                 │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Hardware Identification

### 2.1 Platform Summary

| Property | Value |
|----------|-------|
| Platform | Intel Panther Lake (PTL) Evaluation Board |
| Kernel | 6.18 (Android GKI, ww04 tree) |
| Codec | Everest Semi ES9356 SDCA (MFG_ID=0x04b3, PART_ID=0x9356, v03) |
| Bus Interface | SoundWire (SDCA compliant) |
| SoundWire Link | Link 3 (BIT(3)) -- verified from DSDT |
| ACPI ADR | 0x00033004b3935601 |
| Device Name on Bus | sdw:0:3:04b3:9356:01 |
| SoundWire Link Mask | 0xF (Links 0-3 enabled) |
| ALSA Card | sofsoundwire |
| Machine Driver | sof_sdw |
| NHLT DMICs | 2 (PCH DMIC, 2-channel) |
| Target Device | adb -s 10.239.136.143:5555 |

### 2.2 Multi-Function DAI Architecture

The ES9356 exposes 4 DAI (Digital Audio Interface) endpoints over SoundWire:

| DAI Name | Function | Direction | DAI Link ID Constant | Endpoint # |
|----------|----------|-----------|---------------------|------------|
| es9356-sdp-aif1 | Headphone Output (Jack) | Playback | SOC_SDW_JACK_OUT_DAI_ID (0) | EP 0 |
| es9356-sdp-aif2 | Headset Mic Input (Jack) | Capture | SOC_SDW_JACK_IN_DAI_ID (1) | EP 2 |
| es9356-sdp-aif3 | Speaker Amplifier | Playback | SOC_SDW_AMP_OUT_DAI_ID (2) | EP 3 |
| es9356-sdp-aif4 | Digital Microphone (DMIC) | Capture | SOC_SDW_DMIC_DAI_ID (4) | EP 1 |

### 2.3 SoundWire Device Enumeration

After boot, the kernel discovers the ES9356:

```
$ ls /sys/bus/soundwire/devices/
sdw-master-0-3
sdw:0:3:04b3:9356:01    # ES9356 on Link 3 (MFG=04b3, Part=9356)
```

SoundWire modalias string (used for driver matching):
```
sdw:m04B3p9356v03c01 -> sdw:0:3:04b3:9356:01   (ES9356)
```

### 2.4 SDCA Functions Detected

The BIOS DSDT declares these SDCA functions for the ES9356:

```
acpi device:XX: find_sdca_function: SDCA function SimpleJack (type ?) at 0xN
acpi device:XX: find_sdca_function: SDCA function SmartAmp (type 1) at 0xN
acpi device:XX: find_sdca_function: SDCA function HID (type 10) at 0xN
acpi device:XX: find_sdca_function: SDCA function SmartMic (type ?) at 0xN
```

**Important issue**: The BIOS reports the DMIC function with an HID type descriptor
instead of SmartMic/SimpleMic. This causes a kernel-side SDCA function type check to fail.
See Section 3.4 for the bypass workaround.

### 2.5 NHLT Configuration

```
ACPI: NHLT detected
sof-audio-pci-intel-ptl: DMICs detected in NHLT tables: 2
```

NHLT reports **2 PCH DMICs**, which means:
1. The kernel will append `-2ch` suffix to the topology filename
2. The topology must include PCH DMIC pipelines
3. The actual topology file loaded is `sof-ptl-es9356-2ch.tplg`

### 2.6 ADR Breakdown

The ACPI ADR value `0x00033004b3935601` encodes:

| Field | Value | Meaning |
|-------|-------|---------|
| Link ID | 3 | SoundWire Link 3 |
| Unique ID | 0 | First instance on this link |
| Version | 3 | SDCA version 3 |
| MFG_ID | 0x04b3 | Everest Semiconductor |
| PART_ID | 0x9356 | ES9356 |
| Class | 01 | SDW class |

---

## 3. Kernel Driver Integration

All kernel changes are in the ww04 kernel tree:
```
/home/gaggery/nvme/alos-ptl-ww04-kernel-6.18/common/
```

### 3.1 ES9356 Codec Driver

**Files**: `sound/soc/codecs/es9356.c`, `sound/soc/codecs/es9356.h`

The codec driver is provided by Everest Semiconductor. It implements the SoundWire slave
device interface with SDCA register access via regmap_soundwire.

#### 3.1.1 SoundWire Device ID Table

The driver matches MFG_ID=0x04b3, PART_ID=0x9356, versions 0x02 and 0x03:

```c
static const struct sdw_device_id es9356_sdw_id[] = {
    SDW_SLAVE_ENTRY_EXT(0x04b3, 0x9356, 0x02, 0, 0),
    SDW_SLAVE_ENTRY_EXT(0x04b3, 0x9356, 0x03, 0, 0),
    {},
};
MODULE_DEVICE_TABLE(sdw, es9356_sdw_id);
```

#### 3.1.2 Key Driver Components

- **DAI Definitions**: Registers 4 DAIs (aif1-aif4) via `snd_soc_es9356_sdw_component`
- **Jack Detection**: IRQ-based headset jack and button detection via delayed work queues
  (`jack_detect_work`, `button_detect_work`)
- **SDCA Register Map**: Uses `devm_regmap_init_sdw()` for register I/O over SoundWire

#### 3.1.3 Build Configuration

**Kconfig** (`sound/soc/codecs/Kconfig`):
```
config SND_SOC_ES9356
    tristate "Everest Semi ES9356 CODEC"
    depends on SOUNDWIRE
    select REGMAP_SOUNDWIRE
```

**Makefile** (`sound/soc/codecs/Makefile`):
```makefile
snd-soc-es9356-y := es9356.o
obj-$(CONFIG_SND_SOC_ES9356) += snd-soc-es9356.o
```

### 3.2 SDW Utils Helper

**File**: `sound/soc/sdw_utils/soc_sdw_es9356.c`

This file provides machine-driver-level helpers called by the `sof_sdw` generic machine
driver during card registration and PCM runtime initialization.

#### 3.2.1 DAPM Routes

Three sets of DAPM routes connect codec widgets to machine-level widgets:

```c
/* Headphone/Headset routes */
static const struct snd_soc_dapm_route es9356_map[] = {
    { "Headphone", NULL, "es9356 HP" },
    { "es9356 MIC1", NULL, "Headset Mic" },
};

/* Speaker route */
static const struct snd_soc_dapm_route es9356_spk_map[] = {
    { "Speaker", NULL, "es9356 SPK" },
};

/* DMIC route */
static const struct snd_soc_dapm_route es9356_dmic_map[] = {
    { "es9356 PDM_DIN", NULL, "DMIC" },
};
```

```
  DAPM Widget Connections:

  Machine-level widgets          ES9356 codec widgets
  (defined in sof_sdw)          (defined in es9356.c)

  ┌─────────────┐               ┌─────────────────┐
  │ "Headphone" │◄── route ────│ "es9356 HP"     │──► Physical HP out
  └─────────────┘               └─────────────────┘
  ┌──────────────┐              ┌─────────────────┐
  │ "Headset Mic"│── route ───►│ "es9356 MIC1"   │◄── Physical mic in
  └──────────────┘              └─────────────────┘
  ┌─────────────┐               ┌─────────────────┐
  │ "Speaker"   │◄── route ────│ "es9356 SPK"    │──► Physical speaker
  └─────────────┘               └─────────────────┘
  ┌─────────────┐               ┌─────────────────┐
  │ "DMIC"      │── route ───►│ "es9356 PDM_DIN"│◄── DMIC input
  └─────────────┘               └─────────────────┘
```

#### 3.2.2 Exported Functions

| Function | Purpose |
|----------|---------|
| `asoc_sdw_es9356_init()` | Headset codec init -- adds `everest,jd-src` device property, stores codec device reference |
| `asoc_sdw_es9356_exit()` | Headset codec cleanup -- removes software node |
| `asoc_sdw_es9356_amp_init()` | Speaker amp init -- tracks `amp_num` for multi-amp configurations |
| `asoc_sdw_es9356_amp_exit()` | Speaker amp cleanup -- removes software nodes for amp devices |
| `asoc_sdw_es9356_rtd_init()` | Jack runtime init -- creates headset jack with button support (BTN_0..5), adds DAPM routes, registers jack callback via `snd_soc_component_set_jack()` |
| `asoc_sdw_es9356_spk_rtd_init()` | Speaker runtime init -- adds speaker DAPM routes, registers `spk:es9356-spk` component string |
| `asoc_sdw_es9356_dmic_rtd_init()` | DMIC runtime init -- adds DMIC DAPM routes, registers `mic:es9356-dmic` component string |

#### 3.2.3 Jack Detection and Button Mapping

The `asoc_sdw_es9356_rtd_init()` function creates a headset jack with button support:

```c
ret = snd_soc_card_jack_new_pins(rtd->card, "Headset Jack",
                 SND_JACK_HEADSET | SND_JACK_BTN_0 |
                 SND_JACK_BTN_1 | SND_JACK_BTN_2 |
                 SND_JACK_BTN_3 | SND_JACK_BTN_4 |
                 SND_JACK_BTN_5,
                 &ctx->sdw_headset,
                 es9356_jack_pins,
                 ARRAY_SIZE(es9356_jack_pins));

snd_jack_set_key(jack->jack, SND_JACK_BTN_0, KEY_PLAYPAUSE);
snd_jack_set_key(jack->jack, SND_JACK_BTN_1, KEY_VOICECOMMAND);
snd_jack_set_key(jack->jack, SND_JACK_BTN_2, KEY_VOLUMEUP);
snd_jack_set_key(jack->jack, SND_JACK_BTN_3, KEY_VOLUMEDOWN);
snd_jack_set_key(jack->jack, SND_JACK_BTN_4, KEY_NEXTSONG);
snd_jack_set_key(jack->jack, SND_JACK_BTN_5, KEY_PREVIOUSSONG);
```

#### 3.2.4 SDW Utils Makefile

**File**: `sound/soc/sdw_utils/Makefile`

Added `soc_sdw_es9356.o` to the `snd-soc-sdw-utils` module:

```makefile
snd-soc-sdw-utils-y := soc_sdw_utils.o soc_sdw_dmic.o soc_sdw_rt_dmic.o \
                       soc_sdw_rt700.o soc_sdw_rt711.o                    \
                       soc_sdw_rt5682.o soc_sdw_rt_sdca_jack_common.o     \
                       soc_sdw_rt_amp.o soc_sdw_rt_mf_sdca.o              \
                       soc_sdw_bridge_cs35l56.o                            \
                       soc_sdw_cs42l42.o soc_sdw_cs42l43.o                \
                       soc_sdw_es9356.o                                   \
                       soc_sdw_cs_amp.o                                   \
                       soc_sdw_maxim.o \
                       soc_sdw_ti_amp.o
obj-$(CONFIG_SND_SOC_SDW_UTILS) += snd-soc-sdw-utils.o
```

#### 3.2.5 Header Declarations

**File**: `include/sound/soc_sdw_utils.h`

```c
/* ES9356 support */
int asoc_sdw_es9356_init(struct snd_soc_card *card,
                         struct snd_soc_dai_link *dai_links,
                         struct asoc_sdw_codec_info *info,
                         bool playback);
int asoc_sdw_es9356_exit(struct snd_soc_card *card,
                         struct snd_soc_dai_link *dai_link);
int asoc_sdw_es9356_amp_init(struct snd_soc_card *card,
                             struct snd_soc_dai_link *dai_links,
                             struct asoc_sdw_codec_info *info,
                             bool playback);
int asoc_sdw_es9356_amp_exit(struct snd_soc_card *card,
                             struct snd_soc_dai_link *dai_link);
int asoc_sdw_es9356_rtd_init(struct snd_soc_pcm_runtime *rtd,
                             struct snd_soc_dai *dai);
int asoc_sdw_es9356_spk_rtd_init(struct snd_soc_pcm_runtime *rtd,
                                 struct snd_soc_dai *dai);
int asoc_sdw_es9356_dmic_rtd_init(struct snd_soc_pcm_runtime *rtd,
                                  struct snd_soc_dai *dai);
```

### 3.3 Codec Info List Entry

**File**: `sound/soc/sdw_utils/soc_sdw_utils.c` (lines 654-701)

The `codec_info_list` array tells the `sof_sdw` machine driver how to handle each codec.
The ES9356 entry maps all 4 DAIs to their respective link IDs and callbacks:

```c
{
    .part_id = 0x9356,
    .version_id = 3,
    .dais = {
        {   /* aif1: Headphone Output */
            .direction = {true, false},          /* playback only */
            .dai_name = "es9356-sdp-aif1",
            .dai_type = SOC_SDW_DAI_TYPE_JACK,
            .dailink = {SOC_SDW_JACK_OUT_DAI_ID, SOC_SDW_UNUSED_DAI_ID},
            .init = asoc_sdw_es9356_init,        /* called during card registration */
            .exit = asoc_sdw_es9356_exit,
            .rtd_init = asoc_sdw_es9356_rtd_init, /* called during PCM runtime init */
            .controls = generic_jack_controls,
            .num_controls = ARRAY_SIZE(generic_jack_controls),
            .widgets = generic_jack_widgets,
            .num_widgets = ARRAY_SIZE(generic_jack_widgets),
        },
        {   /* aif4: SDW DMIC Capture */
            .direction = {false, true},          /* capture only */
            .dai_name = "es9356-sdp-aif4",
            .dai_type = SOC_SDW_DAI_TYPE_MIC,
            .dailink = {SOC_SDW_UNUSED_DAI_ID, SOC_SDW_DMIC_DAI_ID},
            .rtd_init = asoc_sdw_es9356_dmic_rtd_init,
            .widgets = generic_dmic_widgets,
            .num_widgets = ARRAY_SIZE(generic_dmic_widgets),
        },
        {   /* aif2: Headset Mic Capture */
            .direction = {false, true},          /* capture only */
            .dai_name = "es9356-sdp-aif2",
            .dai_type = SOC_SDW_DAI_TYPE_JACK,
            .dailink = {SOC_SDW_UNUSED_DAI_ID, SOC_SDW_JACK_IN_DAI_ID},
        },
        {   /* aif3: Speaker Amplifier */
            .direction = {true, false},          /* playback only */
            .dai_name = "es9356-sdp-aif3",
            .dai_type = SOC_SDW_DAI_TYPE_AMP,
            .dailink = {SOC_SDW_AMP_OUT_DAI_ID, SOC_SDW_UNUSED_DAI_ID},
            .init = asoc_sdw_es9356_amp_init,
            .exit = asoc_sdw_es9356_amp_exit,
            .rtd_init = asoc_sdw_es9356_spk_rtd_init,
            .controls = generic_spk_controls,
            .num_controls = ARRAY_SIZE(generic_spk_controls),
            .widgets = generic_spk_widgets,
            .num_widgets = ARRAY_SIZE(generic_spk_widgets),
        },
    },
    .dai_num = 4,
},
```

**Note**: The `ignore_internal_dmic` flag is **NOT set** for ES9356. This allows the kernel
to create PCH DMIC DAI links (`dmic01`, `dmic16k`), which are needed because the physical
microphones connect to PCH, not to the ES9356. If `ignore_internal_dmic` were set to `true`,
`mach_params.dmic_num` would be forced to 0, preventing PCH DMIC DAI link creation and
causing the topology filename suffix to be omitted.

### 3.4 SDCA Endpoint Bypass

**File**: `sound/soc/sdw_utils/soc_sdw_utils.c` (lines 1426-1434)

The `is_sdca_endpoint_present()` function checks whether a codec's SDCA function type
matches the expected DAI type before creating a DAI link. The ES9356 BIOS reports the DMIC
function (function type 0x4) with an HID type descriptor instead of SmartMic/SimpleMic,
causing this check to fail for the MIC endpoint.

```c
/*
 * ES9356 BIOS reports the DMIC function (0x4) as HID type instead of
 * SmartMic/SimpleMic, so the SDCA function type check fails for the
 * MIC endpoint. Bypass the check since the codec DAI is registered.
 */
if (codec_info->part_id == 0x9356) {
    ret = 1;
    goto put_device;
}
```

Without this bypass, the following would happen:

```
  Without bypass:

  sof_sdw machine driver
    │
    ├── Processing ES9356 aif1 (JACK type)
    │   └── is_sdca_endpoint_present() → checks SDCA functions
    │       └── Finds UAJ function → type matches JACK → OK ✓
    │
    ├── Processing ES9356 aif4 (MIC type)
    │   └── is_sdca_endpoint_present() → checks SDCA functions
    │       ├── Function 1: HID type → doesn't match MIC → skip
    │       ├── Function 2: SmartAmp → doesn't match MIC → skip
    │       └── No matching function found → SKIP endpoint ✗
    │           (DMIC DAI link NOT created, PCM 4 missing)
    │
    ...

  With bypass (part_id == 0x9356):

  sof_sdw machine driver
    │
    ├── Processing ES9356 aif4 (MIC type)
    │   └── is_sdca_endpoint_present() → part_id == 0x9356
    │       └── Bypass! Return 1 (present) → DMIC DAI link created ✓
    ...
```

### 3.5 ACPI Match Table

**File**: `sound/soc/intel/common/soc-acpi-intel-ptl-match.c`

The ACPI match table tells the SOF driver which codec is present on which SoundWire link
and which topology file to load.

#### 3.5.1 Endpoint Definitions

Four endpoints declared, one per ES9356 function:

```c
static const struct snd_soc_acpi_endpoint es9356_endpoints[] = {
    { /* Jack Playback Endpoint */
        .num = 0,              /* endpoint number within this device */
        .aggregated = 0,       /* NOT aggregated (standalone) */
        .group_position = 0,
        .group_id = 0,         /* group 0 = no aggregation */
    },
    { /* DMIC Capture Endpoint */
        .num = 1,
        .aggregated = 0,
        .group_position = 0,
        .group_id = 0,
    },
    { /* Jack Capture Endpoint */
        .num = 2,
        .aggregated = 0,
        .group_position = 0,
        .group_id = 0,
    },
    { /* Speaker Playback Endpoint */
        .num = 3,
        .aggregated = 0,
        .group_position = 0,
        .group_id = 0,
    },
};
```

**Note**: All endpoints are `aggregated = 0` because all 4 functions reside on a single
chip on a single SoundWire link. There is no multi-link aggregation needed.

#### 3.5.2 ADR Device and Link Address Arrays

```c
static const struct snd_soc_acpi_adr_device es9356_3_adr[] = {
    {
        .adr = 0x00033004b3935601ull,  /* Link 3, MFG 04b3, Part 9356 */
        .num_endpoints = ARRAY_SIZE(es9356_endpoints),  /* 4 endpoints */
        .endpoints = es9356_endpoints,
        .name_prefix = "es9356"
    }
};

static const struct snd_soc_acpi_link_adr ptl_es9356_l3[] = {
    {
        .mask = BIT(3),                            /* Link 3 */
        .num_adr = ARRAY_SIZE(es9356_3_adr),       /* 1 device */
        .adr_d = es9356_3_adr,                     /* ES9356 */
    },
    {}  /* terminator */
};
```

#### 3.5.3 Machine Table Entry

Added at the end of `snd_soc_acpi_intel_ptl_sdw_machines[]` before the terminator:

```c
{
    .link_mask = BIT(3),                         /* device on Link 3 */
    .links = ptl_es9356_l3,                      /* device descriptions */
    .drv_name = "sof_sdw",                       /* machine driver to use */
    .sof_tplg_filename = "sof-ptl-es9356.tplg",  /* topology to load (base name) */
},
```

The `link_mask = BIT(3)` means this entry matches when a SoundWire device is detected on
Link 3. The topology filename `sof-ptl-es9356.tplg` is the **base name** -- the kernel
appends `-2ch` at runtime based on NHLT DMIC count (see Section 1.4).

---

## 4. SOF Topology Creation

### 4.1 SOF Repository

```
/home/gaggery/nvme/sof/
```

Topology source files are in `tools/topology/topology2/`. The build uses the Docker image
`thesofproject/sof:latest` which includes `alsatplg 1.2.13` (the topology compiler).

### 4.2 Build Targets

**File**: `tools/topology/topology2/production/tplg-targets-ace3.cmake`

Two build targets were added for the ES9356:

#### 4.2.1 SDW-DMIC Only (sof-ptl-es9356)

This topology includes only the SoundWire codec pipelines (no PCH DMIC). Used for testing
the codec in isolation:

```cmake
# ES9356 eval board with SDW-DMIC only
"cavs-sdw\;sof-ptl-es9356\;PLATFORM=ptl,SDW_DMIC=1,NUM_SDW_AMP_LINKS=1,\
SDW_AMP_FEEDBACK=false,SDW_SPK_STREAM=Playback-SmartAmp,SDW_DMIC_STREAM=Capture-SmartMic,\
SDW_JACK_OUT_STREAM=Playback-SimpleJack,SDW_JACK_IN_STREAM=Capture-SimpleJack"
```

#### 4.2.2 SDW-DMIC + PCH-DMIC (sof-ptl-es9356-2ch) -- Production

This is the **production topology** that includes both SoundWire codec pipelines AND PCH
DMIC capture with TDFB beamforming and DRC processing:

```cmake
# ES9356 eval board with SDW-DMIC + PCH-DMIC (2ch)
"cavs-sdw\;sof-ptl-es9356-2ch\;PLATFORM=ptl,SDW_DMIC=1,NUM_SDW_AMP_LINKS=1,NUM_DMICS=2,\
PDM1_MIC_A_ENABLE=0,PDM1_MIC_B_ENABLE=0,DMIC0_ID=5,DMIC1_ID=6,HDMI1_ID=7,HDMI2_ID=8,HDMI3_ID=9,\
SDW_AMP_FEEDBACK=false,SDW_SPK_STREAM=Playback-SmartAmp,SDW_DMIC_STREAM=Capture-SmartMic,\
SDW_JACK_OUT_STREAM=Playback-SimpleJack,SDW_JACK_IN_STREAM=Capture-SimpleJack,\
PREPROCESS_PLUGINS=nhlt,NHLT_BIN=nhlt-sof-ptl-es9356-2ch.bin,DMIC0_ENHANCED_CAPTURE=true,\
EFX_DMIC0_TDFB_PARAMS=line2_generic_pm10deg,EFX_DMIC0_DRC_PARAMS=dmic_default"
```

### 4.3 Build Parameters Explained

| Parameter | Value | Purpose |
|-----------|-------|---------|
| PLATFORM | ptl | Panther Lake -- sets DMIC_DRIVER_VERSION=5, NUM_HDMIS=3 |
| SDW_DMIC | 1 | Enable SoundWire DMIC pipeline (captures from ES9356 aif4 via BE ID 4) |
| NUM_SDW_AMP_LINKS | 1 | One SoundWire amplifier link (ES9356 speaker, no aggregation) |
| NUM_DMICS | 2 | Enable PCH DMIC with 2 channels (PDM0 only) |
| PDM1_MIC_A/B_ENABLE | 0 | Disable PDM1 (only PDM0 used for 2ch DMIC) |
| DMIC0_ID | 5 | Backend DAI ID for PCH DMIC 48kHz (after SDW IDs 0-4) |
| DMIC1_ID | 6 | Backend DAI ID for PCH DMIC 16kHz |
| HDMI1_ID / 2 / 3 | 7 / 8 / 9 | HDMI backend DAI IDs (after DMIC IDs) |
| SDW_AMP_FEEDBACK | false | No amplifier feedback/sensing channel |
| SDW_SPK_STREAM | Playback-SmartAmp | SoundWire stream name for speaker (must match DSDT) |
| SDW_DMIC_STREAM | Capture-SmartMic | SoundWire stream name for DMIC (must match DSDT) |
| SDW_JACK_OUT_STREAM | Playback-SimpleJack | SoundWire stream name for headphone jack |
| SDW_JACK_IN_STREAM | Capture-SimpleJack | SoundWire stream name for headset mic |
| PREPROCESS_PLUGINS | nhlt | Generate NHLT blob embedded in topology binary |
| NHLT_BIN | nhlt-sof-ptl-es9356-2ch.bin | Also output standalone NHLT binary (for debug) |
| DMIC0_ENHANCED_CAPTURE | true | Enable TDFB beamforming + DRC processing on PCH DMIC |
| EFX_DMIC0_TDFB_PARAMS | line2_generic_pm10deg | TDFB config: 2-mic linear array, +/-10 degree |
| EFX_DMIC0_DRC_PARAMS | dmic_default | DRC parameters: default DMIC compression |

### 4.4 Build Command

```bash
cd /home/gaggery/nvme/sof

docker run -i \
  -v "$(pwd)":/home/sof/work/sof.git \
  -v "$(pwd)":/home/sof/work/sof-bind-mount-DO-NOT-DELETE \
  --user "$(id -u):$(id -g)" \
  thesofproject/sof:latest bash -c '
    SOF=/home/sof/work/sof.git
    TPLG2=$SOF/tools/topology/topology2
    BUILDDIR=/tmp/tplg_build
    mkdir -p $BUILDDIR

    # Step 1: Extract ABI version from SOF source
    bash $TPLG2/get_abi.sh $SOF ipc4 > $BUILDDIR/abi.conf

    # Step 2: Concatenate ABI header with base SoundWire topology template
    cat $BUILDDIR/abi.conf $TPLG2/cavs-sdw.conf \
      > $BUILDDIR/sof-ptl-es9356-2ch.conf

    # Step 3: Compile topology with all parameters
    ALSA_CONFIG_DIR=$TPLG2 \
    LD_LIBRARY_PATH=/home/sof/work/alsa/alsa-lib/src/.libs \
    /home/sof/work/tools/bin/alsatplg -v 1 \
      -I $TPLG2/ \
      -D "PLATFORM=ptl,SDW_DMIC=1,NUM_SDW_AMP_LINKS=1,\
NUM_DMICS=2,PDM1_MIC_A_ENABLE=0,PDM1_MIC_B_ENABLE=0,\
DMIC0_ID=5,DMIC1_ID=6,HDMI1_ID=7,HDMI2_ID=8,HDMI3_ID=9,\
SDW_AMP_FEEDBACK=false,\
SDW_SPK_STREAM=Playback-SmartAmp,\
SDW_DMIC_STREAM=Capture-SmartMic,\
SDW_JACK_OUT_STREAM=Playback-SimpleJack,\
SDW_JACK_IN_STREAM=Capture-SimpleJack,\
PREPROCESS_PLUGINS=nhlt,\
NHLT_BIN=nhlt-sof-ptl-es9356-2ch.bin,\
DMIC0_ENHANCED_CAPTURE=true,\
EFX_DMIC0_TDFB_PARAMS=line2_generic_pm10deg,\
EFX_DMIC0_DRC_PARAMS=dmic_default" \
      -p -c $BUILDDIR/sof-ptl-es9356-2ch.conf \
      -o $SOF/sof-ptl-es9356-2ch.tplg
  '
```

### 4.5 Build Steps Explained

```
  Build Pipeline:

  ┌──────────────┐     ┌──────────────────┐     ┌─────────────────────────────┐
  │ get_abi.sh   │     │ cavs-sdw.conf    │     │ alsatplg 1.2.13 compiler    │
  │              │     │ (base SoundWire  │     │ (inside Docker container)   │
  │ Extracts ABI │     │  topology        │     │                             │
  │ version from │     │  template with   │     │ Reads .conf input           │
  │ SOF source   │     │  conditional     │     │ Applies -D defines          │
  │              │     │  blocks)         │     │ Resolves includes (-I)      │
  └──────┬───────┘     └────────┬─────────┘     │ Outputs binary .tplg        │
         │                      │               │ + NHLT blob                 │
         │   abi.conf           │               └──────────────┬──────────────┘
         └──────────┐           │                               │
                    ▼           ▼                               │
              ┌─────────────────────┐                          │
              │ cat abi.conf        │     -D "PLATFORM=ptl,    │
              │     cavs-sdw.conf   │      SDW_DMIC=1,..."     │
              │  > combined.conf    │──────────────────────────►│
              └─────────────────────┘                          │
                                                               ▼
                                              ┌──────────────────────────┐
                                              │ sof-ptl-es9356-2ch.tplg │
                                              │ (90,007 bytes)           │
                                              │                          │
                                              │ + nhlt-sof-ptl-es9356-   │
                                              │   2ch.bin (6,211 bytes)  │
                                              └──────────────────────────┘
```

1. **get_abi.sh**: Extracts the SOF ABI version header (`ABI 3:29:1`) and writes it as a
   topology configuration snippet.

2. **cat abi.conf cavs-sdw.conf**: Concatenates the ABI header with the base SoundWire
   topology template. `cavs-sdw.conf` includes conditional blocks resolved by the
   preprocessor.

3. **alsatplg -p**: The `-p` flag enables Topology2 preprocessor mode. Key defines:
   - `NUM_SDW_AMP_LINKS=1` → 1 ALH copier for SmartAmp
   - `SDW_DMIC=1` → include SmartMic capture pipeline
   - `NUM_DMICS=2` → include PCH DMIC pipelines with NHLT blob
   - `DMIC0_ID=5` → PCH DMIC backend IDs start at 5 (after SDW IDs 0-4)

4. **Docker container**: Required because system `alsatplg 1.2.9` (from alsa-lib 1.2.11)
   fails on nested math expressions in the topology configuration. The Docker image has
   `alsatplg 1.2.13` with full support.

### 4.6 Topology Output: PCM and Backend DAI Layout

#### 4.6.1 Frontend PCM Devices

| PCM ID | Name | Direction | Source | Stream Name |
|--------|------|-----------|--------|-------------|
| 0 | Jack Out | Playback | ES9356 aif1 (SimpleJack) | Playback-SimpleJack |
| 1 | Jack In | Capture | ES9356 aif2 (SimpleJack) | Capture-SimpleJack |
| 2 | Speaker | Playback | ES9356 aif3 (SmartAmp) | Playback-SmartAmp |
| 4 | Microphone (SDW DMIC) | Capture | ES9356 aif4 (SmartMic) | Capture-SmartMic |
| 5 | HDMI1 | Playback | Intel HDA iDisp1 | iDisp1 |
| 6 | HDMI2 | Playback | Intel HDA iDisp2 | iDisp2 |
| 7 | HDMI3 | Playback | Intel HDA iDisp3 | iDisp3 |
| 10 | DMIC Raw (PCH DMIC) | Capture | PCH dmic01 | dmic01 |
| 31 | Deepbuffer Jack Out | Playback | ES9356 aif1 (deep buffer) | Playback-SimpleJack |

#### 4.6.2 Backend DAI Links

| BE ID | Name | Type | Stream Name | Codec |
|-------|------|------|-------------|-------|
| 0 | Playback-SimpleJack | SDW ALH | SDW0-Playback | ES9356 es9356-sdp-aif1 |
| 1 | Capture-SimpleJack | SDW ALH | SDW0-Capture | ES9356 es9356-sdp-aif2 |
| 2 | Playback-SmartAmp | SDW ALH | SDW2-Playback | ES9356 es9356-sdp-aif3 |
| 4 | Capture-SmartMic | SDW ALH | SDW4-Capture | ES9356 es9356-sdp-aif4 |
| 5 | dmic01 | PCH DMIC | dmic01 | (PCH built-in) |
| 6 | dmic16k | PCH DMIC | dmic16k | (PCH built-in) |
| 7 | iDisp1 | HDA HDMI | iDisp1 | ehdaudio0D2 |
| 8 | iDisp2 | HDA HDMI | iDisp2 | ehdaudio0D2 |
| 9 | iDisp3 | HDA HDMI | iDisp3 | ehdaudio0D2 |

```
  Complete Audio Pipeline:

  ┌─────────────────────────────────────────────────────────────────────┐
  │  Audio DSP                                                          │
  │                                                                     │
  │  PCM 0  ──► [Host Copier] ──► [Gain] ──► [ALH Copier] ──► BE 0    │  ← Jack Out
  │  PCM 1  ◄── [Host Copier] ◄── [Gain] ◄── [ALH Copier] ◄── BE 1    │  ← Jack In
  │  PCM 2  ──► [Host Copier] ──► [Gain] ──► [ALH Copier] ──► BE 2    │  ← Speaker
  │  PCM 4  ◄── [Host Copier] ◄── [Gain] ◄── [ALH Copier] ◄── BE 4    │  ← SDW DMIC
  │  PCM 10 ◄── [Host Copier] ◄── [TDFB] ◄── [DRC] ◄── [DMIC] ◄ BE 5 │  ← PCH DMIC
  │  PCM 5  ──► [Host Copier] ──► [ALH Copier] ──► BE 7                │  ← HDMI1
  │  PCM 6  ──► [Host Copier] ──► [ALH Copier] ──► BE 8                │  ← HDMI2
  │  PCM 7  ──► [Host Copier] ──► [ALH Copier] ──► BE 9                │  ← HDMI3
  │  PCM 31 ──► [Host Copier] ──► [Gain] ──────────┘ (deep to PCM 0)  │  ← Deepbuffer
  └─────────────────────────────────────────────────────────────────────┘
```

---

## 5. DRAS HAL Integration

### 5.1 Source Location and Build

```
Source: vendor/google/desktop/audio/dras/src/common/environment.rs
Build:  m com.android.hardware.audio.desktop
APEX:   com.android.hardware.audio.desktop → /vendor/apex/
```

DRAS is written in Rust and runs as an Android APEX service. The audio configuration
detection logic lives in `environment.rs`.

### 5.2 How DRAS Finds the Audio Config Directory

DRAS uses a priority chain to find the correct audio configuration directory. Three changes
were needed for ES9356 support.

#### 5.2.1 Priority Chain

```
  get_audio_config_dir() resolution order:

  Priority 1: hal_config.xml (SKU-based)
  ├── boards_uses_hal_config_xml(board) → check if board uses XML config
  └── get_audio_config_dir_from_hal_config() → read "audio-config-dir" value
      └── NOT used for PTL eval board (no hal_config.xml)

  Priority 2: Dynamic codec detection (Ubuntu-like approach)  ← THIS IS USED
  ├── get_primary_soundcard_name()
  │   ├── Try vendor.audio.soundcard.name system property → not set
  │   └── Fallback: read /proc/asound/card0/id → "sofsoundwire" ← CHANGE #3
  ├── detect_audio_codecs_from_system()
  │   ├── Method 1: Scan /sys/bus/soundwire/devices/*/modalias
  │   │   └── Found "p9356" → pushes "es9356"               ← CHANGE #1
  │   └── Method 2: Scan /proc/asound/card*/codec*
  │       └── Found "es9356" in codec info                    ← CHANGE #2
  └── get_audio_config_dir_from_detected_codecs("sofsoundwire")
      ├── Builds candidate: "sof-soundwire_es9356"
      ├── Scans /vendor/etc/audio/ for matching directory
      └── Finds: /vendor/etc/audio/sof-soundwire_es9356/
          └── Verifies audio_config.xml exists → MATCH ✓

  Priority 3: SKU-based directory naming
  └── (fallback, not reached)
```

#### 5.2.2 Change #1: SoundWire Modalias Detection

Added ES9356 to the SoundWire modalias codec detection in `detect_audio_codecs_from_system()`:

```rust
/// Detect audio codecs from the system by scanning SoundWire bus and ALSA card info.
fn detect_audio_codecs_from_system() -> Vec<String> {
    let mut codecs = Vec::new();

    // Method 1: Scan SoundWire bus for codec devices
    if let Ok(entries) = fs::read_dir("/sys/bus/soundwire/devices") {
        for entry in entries.flatten() {
            let modalias_path = entry.path().join("modalias");
            if let Ok(modalias) = fs::read_to_string(&modalias_path) {
                // SoundWire modalias format: sdw:mXXXXpYYYY (manufacturer/part)
                let modalias = modalias.trim().to_lowercase();
                if modalias.contains("p0712") || modalias.contains("p712") {
                    codecs.push("rt712".to_string());
                } else if modalias.contains("p1320") {
                    codecs.push("rt1320".to_string());
                // ... other codecs ...
                } else if modalias.contains("p4243") {
                    codecs.push("cs42l43".to_string());
                } else if modalias.contains("p3556") {
                    codecs.push("cs35l56".to_string());
+               } else if modalias.contains("p9356") {     // ← NEW
+                   codecs.push("es9356".to_string());
                }
            }
        }
    }
```

What this does at runtime on the ES9356 board:

```
  /sys/bus/soundwire/devices/sdw:0:3:04b3:9356:01/modalias
  → contains: "sdw:m04B3p9356v03c01"
  → modalias.contains("p9356") → true
  → codecs = ["es9356"]
```

#### 5.2.3 Change #2: ALSA Card Codec Scanning

Added "es9356" to the ALSA card codec info scanning list:

```rust
    // Method 2: Scan ALSA card components for codec info
    if let Ok(entries) = fs::read_dir("/proc/asound") {
        for entry in entries.flatten() {
            if name_str.starts_with("card") {
                if let Ok(codec_entries) = fs::read_dir(&card_path) {
                    for codec_entry in codec_entries.flatten() {
                        if codec_name_str.starts_with("codec") {
                            if let Ok(codec_info) = fs::read_to_string(codec_entry.path()) {
                                let codec_lower = codec_info.to_lowercase();
                                for codec in ["rt712", "rt721", "rt722", "rt1320",
+                                             "cs42l43", "cs35l56", "es9356",   // ← NEW
                                              "alc256", "alc5682"] {
                                    if codec_lower.contains(codec) {
                                        codecs.push(codec.to_string());
                                    }
                                }
```

This is a secondary detection method. If Method 1 (SoundWire modalias) misses the codec,
Method 2 reads `/proc/asound/card0/codec*` files and searches for codec name strings.

#### 5.2.4 Change #3: Soundcard Name Fallback

The `get_primary_soundcard_name()` function originally relied on the
`vendor.audio.soundcard.name` system property, which is not set on the PTL eval board.
Added a fallback to read the card name from `/proc/asound/card0/id`:

```rust
fn get_primary_soundcard_name(board_name: &Option<String>) -> Option<String> {
    // ... existing hal_config and system property checks ...

    match get_system_properties("vendor.audio.soundcard.name") {
        Ok(Some(card)) => return Some(card),
        Ok(None) => {}
        Err(e) => {
            error!("get vendor.audio.soundcard.name failed: {}", e);
        }
    }

+   // Fallback: read card name from /proc/asound/card0/id
+   fs::read_to_string("/proc/asound/card0/id")
+       .ok()
+       .map(|s| s.trim().to_string())
}
```

The same fallback was also added in `get_audio_config_dir()` for the Priority 2 detection
path:

```rust
    // Priority 2: Try dynamic detection based on actual hardware
    let soundcard_for_detection = primary_soundcard_name.clone().or_else(|| {
+       fs::read_to_string("/proc/asound/card0/id")
+           .ok()
+           .map(|s| s.trim().to_string())
    });
```

Without this fallback, `soundcard_for_detection` would be `None`, and the entire Priority 2
detection would be skipped, causing DRAS to fall through to Priority 3 (SKU-based) which
would also fail.

### 5.3 Audio Config Directory Resolution

Once codecs are detected, `get_audio_config_dir_from_detected_codecs()` searches for a
matching directory in `/vendor/etc/audio/`:

```rust
fn get_audio_config_dir_from_detected_codecs(soundcard_name: &str) -> Option<PathBuf> {
    let codecs = detect_audio_codecs_from_system();  // → ["es9356"]
    let basedir = PathBuf::from("/vendor/etc/audio");

    // Build candidate directory names
    // For SoundWire cards, try: "sof-soundwire_es9356"
    let mut candidates = Vec::new();
    if card_normalized.contains("soundwire") || card_normalized.contains("sof") {
        for codec in &codecs {
            candidates.push(format!("sof-soundwire_{}", codec));
            // → candidates = ["sof-soundwire_es9356"]
        }
    }

    // Search /vendor/etc/audio/ for matching directories
    // Exact match: "sof-soundwire_es9356" exists? → YES
    // Verify audio_config.xml exists inside? → YES
    // → Return /vendor/etc/audio/sof-soundwire_es9356/
}
```

```
  Directory matching on device:

  /vendor/etc/audio/
  ├── sof-soundwire_cs42l43/            ← CS42L43 config
  ├── sof-soundwire_rt712_rt1320/       ← RT712+RT1320 config
  ├── sof-soundwire_es9356/             ← ES9356 config ← MATCHED
  │   ├── audio_config.xml
  │   └── mixer_paths.xml
  └── for_all_boards/                   ← fallback
```

### 5.4 What Happens After Config Is Found

Once DRAS finds the config directory, it:

1. **Reads `audio_config.xml`**: Parses the PCM device mapping for each audio path
2. **Reads `mixer_paths.xml`**: Applies global default mixer settings, then path-specific
   overrides when audio paths are activated/deactivated
3. **Opens ALSA PCM devices**: When Android requests audio output/input, DRAS opens the
   corresponding ALSA PCM device (e.g., `hw:sofsoundwire,2` for Speaker)

```
  DRAS runtime behavior:

  1. Startup:
     audio_config.xml → learn PCM mappings
     mixer_paths.xml  → apply global defaults:
       Headphone Switch     = 0 (off)
       Headset Mic Switch   = 0 (off)
       Speaker Switch       = 0 (off)
       Dmic0 Capture Switch = 1 (on)  ← PCH DMIC always enabled

  2. Android requests Speaker playback:
     → DRAS activates "Speaker" path
     → mixer_paths.xml: Speaker Switch = 1
     → Opens hw:sofsoundwire,2
     → Audio flows: App → AudioFlinger → DRAS → PCM 2 → DSP → ES9356 aif3 → 🔊

  3. Speaker playback stops:
     → DRAS deactivates "Speaker" path
     → mixer_paths.xml: Speaker Switch = 0 (back to default)
     → Closes hw:sofsoundwire,2

  4. Android requests InternalMic capture (e.g., camera recording):
     → DRAS activates "InternalMic" path
     → mixer_paths.xml: Dmic0 Capture Switch = 1 (already on)
     → Opens hw:sofsoundwire,10
     → Audio flows: Physical DMIC 🎤 → PCH → DSP (TDFB+DRC) → PCM 10 → DRAS → App
```

---

## 6. ALSA Configuration Files

These files are deployed to the device at:
```
/vendor/etc/audio/sof-soundwire_es9356/
```

Source location for the build system:
```
alos_grub/new_files/device/google/desktop/fatcat/alsa/sof-soundwire_es9356/
```

### 6.1 audio_config.xml

```xml
<?xml version='1.0' encoding='utf-8'?>
<audio_config>
	<!--ES9356 SDCA codec on SoundWire-->
	<SoundCardName>hw:sofsoundwire</SoundCardName>
	<HiFi>
		<Speaker>
			<PlaybackPCM>hw:sofsoundwire,2</PlaybackPCM>
		</Speaker>
		<Headphone>
			<PlaybackPCM>hw:sofsoundwire,0</PlaybackPCM>
			<JackDev>sof-soundwire Headset Jack</JackDev>
			<JackSwitch>2</JackSwitch>
		</Headphone>
		<InternalMic>
			<CapturePCM>hw:sofsoundwire,10</CapturePCM>
			<IntrinsicSensitivity>-2600</IntrinsicSensitivity>
		</InternalMic>
		<Mic>
			<CapturePCM>hw:sofsoundwire,1</CapturePCM>
			<CaptureMixerElem>Headset Mic</CaptureMixerElem>
			<JackDev>sof-soundwire Headset Jack</JackDev>
		</Mic>
		<HDMI1>
			<PlaybackPCM>hw:sofsoundwire,5</PlaybackPCM>
			<JackDev>sof-soundwire HDMI/DP,pcm=5</JackDev>
			<PlaybackChannels>2</PlaybackChannels>
		</HDMI1>
		<HDMI2>
			<PlaybackPCM>hw:sofsoundwire,6</PlaybackPCM>
			<JackDev>sof-soundwire HDMI/DP,pcm=6</JackDev>
			<PlaybackChannels>2</PlaybackChannels>
		</HDMI2>
		<HDMI3>
			<PlaybackPCM>hw:sofsoundwire,7</PlaybackPCM>
			<JackDev>sof-soundwire HDMI/DP,pcm=7</JackDev>
			<PlaybackChannels>2</PlaybackChannels>
		</HDMI3>
	</HiFi>
</audio_config>
```

**How DRAS uses audio_config.xml:**

```
  audio_config.xml routing map:

  Android AudioFlinger
       │
       ▼
  DRAS HAL reads audio_config.xml to find:
       │
       ├── Speaker     → open hw:sofsoundwire,2  (PCM 2, ES9356 SmartAmp)
       ├── Headphone   → open hw:sofsoundwire,0  (PCM 0, ES9356 SimpleJack)
       ├── InternalMic → open hw:sofsoundwire,10 (PCM 10, PCH DMIC)
       ├── Mic         → open hw:sofsoundwire,1  (PCM 1, ES9356 SimpleJack capture)
       ├── HDMI1       → open hw:sofsoundwire,5  (PCM 5, HDA display)
       ├── HDMI2       → open hw:sofsoundwire,6  (PCM 6, HDA display)
       └── HDMI3       → open hw:sofsoundwire,7  (PCM 7, HDA display)
```

**Critical**: `InternalMic` uses **PCM 10** (PCH DMIC), **NOT** PCM 4 (SDW DMIC). The
physical microphones connect to PCH pins, not to the ES9356. If this were set to PCM 4,
camera recording and other internal mic use cases would capture from the wrong device
(no physical microphone connected).

### 6.2 mixer_paths.xml

```xml
<?xml version='1.0' encoding='utf-8'?>
<mixer enum_mixer_numeric_fallback="true">
	<!--ES9356 SDCA codec mixer paths-->
	<ctl name="Headphone Switch" value="0" />
	<ctl name="Headset Mic Switch" value="0" />
	<ctl name="Speaker Switch" value="0" />
	<ctl name="Dmic0 Capture Switch" value="1" />
	<path name="Speaker">
		<ctl name="Speaker Switch" value="1" />
	</path>
	<path name="Headphone">
		<ctl name="Headphone Switch" value="1" />
	</path>
	<path name="InternalMic">
		<ctl name="Dmic0 Capture Switch" value="1" />
	</path>
	<path name="Mic">
		<ctl name="Headset Mic Switch" value="1" />
	</path>
	<path name="HDMI1" />
	<path name="HDMI2" />
	<path name="HDMI3" />
</mixer>
```

**Understanding mixer_paths.xml structure:**

```
  mixer_paths.xml layout:

  ┌─────────────────────────────────────────────────────────────────┐
  │  <mixer>                                                        │
  │                                                                 │
  │    ── Global defaults (applied at DRAS startup) ──              │
  │    <ctl name="Headphone Switch" value="0" />   ← Off by default│
  │    <ctl name="Headset Mic Switch" value="0" /> ← Off by default│
  │    <ctl name="Speaker Switch" value="0" />     ← Off by default│
  │    <ctl name="Dmic0 Capture Switch" value="1" /> ← Always On   │
  │                                                                 │
  │    ── Path overrides (applied when path is activated) ──        │
  │    <path name="Speaker">                                        │
  │      <ctl name="Speaker Switch" value="1" />   ← On for speaker│
  │    </path>                                                      │
  │    <path name="Headphone">                                      │
  │      <ctl name="Headphone Switch" value="1" /> ← On for HP     │
  │    </path>                                                      │
  │    <path name="InternalMic">                                    │
  │      <ctl name="Dmic0 Capture Switch" value="1" /> ← On for mic│
  │    </path>                                                      │
  │    <path name="Mic">                                            │
  │      <ctl name="Headset Mic Switch" value="1" /> ← On for mic  │
  │    </path>                                                      │
  │    ...                                                          │
  │                                                                 │
  │  </mixer>                                                       │
  └─────────────────────────────────────────────────────────────────┘

  When DRAS starts:
    1. All global defaults applied (everything outside <path> blocks)
    2. Dmic0 Capture Switch stays On always (global default = 1)
    3. When "Speaker" path activates: Speaker Switch → On
    4. When "Speaker" path deactivates: Speaker Switch → back to default (Off)
```

**Important**: `Dmic0 Capture Switch` defaults to `value="1"` (enabled). Without this, PCH
DMIC capture produces silence after every reboot. This was discovered during testing -- the
ALSA control defaults to Off (muted), requiring manual unmuting with `tinymix`.

---

## 7. Deployment and Verification

### 7.1 Kernel Module Deployment

The ES9356 code compiles into two kernel modules:

| Module | Location on Device |
|--------|-------------------|
| `snd-soc-es9356.ko` | `/vendor_dlkm/lib/modules/` |
| `snd-soc-sdw-utils.ko` | `/vendor_dlkm/lib/modules/` |

```bash
adb -s <device_ip>:5555 root
adb -s <device_ip>:5555 remount
adb -s <device_ip>:5555 push snd-soc-sdw-utils.ko \
    /vendor_dlkm/lib/modules/snd-soc-sdw-utils.ko
adb -s <device_ip>:5555 push snd-soc-es9356.ko \
    /vendor_dlkm/lib/modules/snd-soc-es9356.ko
adb -s <device_ip>:5555 reboot
```

**Why a reboot is required**: Kernel modules are loaded during early boot. The SoundWire
enumeration and SOF topology loading happen during the audio PCI driver's probe function.

### 7.2 Topology File Deployment

```bash
adb -s <device_ip>:5555 remount
adb -s <device_ip>:5555 push sof-ptl-es9356-2ch.tplg \
    /vendor/firmware/intel/sof-ipc4-tplg/sof-ptl-es9356-2ch.tplg
adb -s <device_ip>:5555 reboot
```

**Why a reboot is required**: The topology is loaded by the kernel's firmware loader during
the audio PCI driver's probe function at boot time.

### 7.3 ALSA Config Deployment

```bash
adb -s <device_ip>:5555 remount
adb -s <device_ip>:5555 push audio_config.xml \
    /vendor/etc/audio/sof-soundwire_es9356/audio_config.xml
adb -s <device_ip>:5555 push mixer_paths.xml \
    /vendor/etc/audio/sof-soundwire_es9356/mixer_paths.xml

# Restart audio server to reload (no reboot needed)
adb -s <device_ip>:5555 shell kill $(pidof audioserver)
```

**Why no reboot is needed**: DRAS reads XML files when audioserver starts. Killing
audioserver causes Android's init to restart it, re-reading the XML files.

### 7.4 DRAS HAL Deployment

```bash
# Build (from Android source tree root)
source build/envsetup.sh
lunch firefly-userdebug
m com.android.hardware.audio.desktop

# Deploy
adb -s <device_ip>:5555 remount
adb -s <device_ip>:5555 push \
    out/target/product/firefly/vendor/apex/com.android.hardware.audio.desktop.apex \
    /vendor/apex/com.android.hardware.audio.desktop.apex
adb -s <device_ip>:5555 reboot
```

### 7.5 Verification: Topology Loading

```bash
adb shell dmesg | grep -E "Topology|can.t find BE|component load|DAI link"
```

Expected output (after successful deployment):
```
sof-audio-pci-intel-ptl: Topology file: intel/sof-ipc4-tplg/sof-ptl-es9356-2ch.tplg
sof-audio-pci-intel-ptl: Topology: ABI 3:29:1 Kernel ABI 3:23:1
sof_sdw: DAI link[0] 'SDW3-Playback-SimpleJack': codecs=1 cpus=1
sof_sdw: DAI link[1] 'SDW3-Capture-SimpleJack': codecs=1 cpus=1
sof_sdw: DAI link[2] 'SDW3-Playback-SmartAmp': codecs=1 cpus=1
sof_sdw: DAI link[3] 'SDW3-Capture-SmartMic': codecs=1 cpus=1
sof_sdw: DAI link[4] 'dmic01': codecs=1 cpus=1
sof_sdw: DAI link[5] 'dmic16k': codecs=1 cpus=1
sof_sdw: DAI link[6] 'iDisp1': codecs=1 cpus=1
sof_sdw: DAI link[7] 'iDisp2': codecs=1 cpus=1
sof_sdw: DAI link[8] 'iDisp3': codecs=1 cpus=1
```

**What to look for:**
- No "can't find BE" or "component load failed" errors
- DAI links for dmic01 and dmic16k are present (PCH DMIC enabled)
- All 9 DAI links created successfully

### 7.6 Verification: ALSA Card and PCM Devices

```bash
# Check ALSA card
adb shell cat /proc/asound/cards
# Expected:
#  0 [sofsoundwire   ]: sof-soundwire - sof-soundwire

# Check PCM devices
adb shell cat /proc/asound/pcm
# Expected:
# 00-00: Jack Out (*) :  : playback 1
# 00-01: Jack In (*) :  : capture 1
# 00-02: Speaker (*) :  : playback 1
# 00-04: Microphone (*) :  : capture 1
# 00-05: HDMI1 (*) :  : playback 1
# 00-06: HDMI2 (*) :  : playback 1
# 00-07: HDMI3 (*) :  : playback 1
# 00-10: DMIC Raw (*) :  : capture 1
# 00-31: Deepbuffer Jack Out (*) :  : playback 1
```

### 7.7 Verification: PCH DMIC Capture

```bash
# Record 5 seconds from PCH DMIC (PCM 10)
adb shell tinycap /data/local/tmp/test.wav \
    -D 0 -d 10 -c 2 -r 48000 -b 32 -T 5

# Pull and verify
adb pull /data/local/tmp/test.wav .
# Play back locally -- should hear ambient sound
```

**Verified result from this system:**

| Measurement | Value |
|-------------|-------|
| Peak sample | 35,471,936 |
| Peak dBFS | -20.8 dBFS |
| RMS dBFS | -39.7 dBFS |
| Duration | 5 seconds |
| Channels | 2 (stereo) |
| Sample rate | 48 kHz |

### 7.8 Verification: Mixer Controls

```bash
# List all mixer controls
adb shell tinymix -D 0

# Verify Dmic0 Capture Switch is ON
adb shell tinymix -D 0 'Dmic0 Capture Switch'
# Expected: On
```

---

## 8. Known Issues and Workarounds

### 8.1 SDCA Function Type Mismatch (BIOS Bug)

**Issue**: The BIOS reports the ES9356 DMIC function (function type 0x4) with an HID type
descriptor instead of SmartMic or SimpleMic.

**Impact**: Without the fix, the SDW DMIC endpoint (PCM 4) is not created because the
`is_sdca_endpoint_present()` function cannot find a matching SDCA function type.

**Workaround**: Added `part_id`-specific bypass in `is_sdca_endpoint_present()` for
`part_id == 0x9356` (see Section 3.4).

### 8.2 PCH DMIC Capture Initially Silent

**Issue**: After deploying the topology with PCH DMIC, the `Dmic0 Capture Switch` ALSA
control defaults to Off (0), causing PCH DMIC capture to return silence.

**Impact**: Internal microphone recording produces empty audio.

**Workaround**: Set `Dmic0 Capture Switch` default to `value="1"` in `mixer_paths.xml`.
DRAS reads this at startup and applies the initial mixer state.

### 8.3 InternalMic PCM Routing Error

**Issue**: Initial `audio_config.xml` had `InternalMic` pointing to PCM 4 (SDW DMIC)
instead of PCM 10 (PCH DMIC).

**Impact**: Camera recording captured from the wrong PCM device (no physical mic connected
to ES9356 DMIC input).

**Fix**: Changed `InternalMic` `CapturePCM` to `hw:sofsoundwire,10`.

### 8.4 Topology alsatplg Version Requirement

**Issue**: System `alsatplg 1.2.9` (from alsa-lib 1.2.11) cannot compile SOF Topology2
configurations due to unsupported nested math expressions like
`$[($in_channels * ($[($in_rate + 999)] / 1000)) * ($in_bit_depth / 8)]`.

**Impact**: Topology compilation fails with syntax errors.

**Workaround**: Use Docker image `thesofproject/sof:latest` which contains `alsatplg 1.2.13`
with full nested expression support (see Section 4.4).

### 8.5 ADR Link ID Must Match DSDT

**Issue**: Initial integration used Link 1 (`BIT(1)`) in the ACPI match table, but the DSDT
defines the ES9356 on Link 3 (`BIT(3)`).

**Impact**: The kernel cannot match the ACPI description to the match table entry, failing
to load the SOF topology. No sound card is created.

**Fix**: Corrected the ADR to `0x00033004b3935601` (link 3) and `link_mask` to `BIT(3)`.
This required rebuilding and reflashing the kernel.

### 8.6 DRAS Soundcard Name Detection Failure

**Issue**: The `vendor.audio.soundcard.name` system property is not set on the PTL eval
board. Without the `/proc/asound/card0/id` fallback, `primary_soundcard_name` is `None`,
causing DRAS to skip the Priority 2 codec detection path entirely.

**Impact**: DRAS cannot find the `sof-soundwire_es9356/` config directory. Audio HAL
initializes with wrong or no configuration.

**Fix**: Added `/proc/asound/card0/id` fallback in `get_primary_soundcard_name()` and
`get_audio_config_dir()` (see Section 5.2.4).

---

## 9. Kernel Command Line (Audio-Related)

```
snd_intel_dspcfg.dsp_driver=3
snd_sof_intel_hda_common.sof_use_tplg_nhlt=1
snd_sof_pci.ipc_type=1
snd_intel_sdw_acpi.sdw_link_mask=0xF
```

| Parameter | Value | Purpose |
|-----------|-------|---------|
| snd_intel_dspcfg.dsp_driver | 3 | Force SOF DSP driver (not legacy HDA or SST). Value 3 = SOF. |
| snd_sof_intel_hda_common.sof_use_tplg_nhlt | 1 | Use NHLT data embedded in topology for DMIC configuration |
| snd_sof_pci.ipc_type | 1 | Use IPC4 protocol for DSP communication (Panther Lake uses IPC4) |
| snd_intel_sdw_acpi.sdw_link_mask | 0xF | Enable SoundWire links 0-3 (binary 1111). Without this, SoundWire may not enumerate. |

---

## 10. Complete File Reference

### 10.1 New Files (Kernel)

| File Path | Description |
|-----------|-------------|
| `sound/soc/codecs/es9356.c` | ES9356 SoundWire codec driver (~1430 lines) |
| `sound/soc/codecs/es9356.h` | ES9356 codec header (registers, structs) |
| `sound/soc/sdw_utils/soc_sdw_es9356.c` | SDW utils helper for ES9356 (280 lines) |

### 10.2 Modified Files (Kernel)

| File Path | Change |
|-----------|--------|
| `sound/soc/codecs/Kconfig` | Added `SND_SOC_ES9356` config entry |
| `sound/soc/codecs/Makefile` | Added `es9356.o` build rules |
| `sound/soc/sdw_utils/Makefile` | Added `soc_sdw_es9356.o` to sdw-utils module |
| `sound/soc/sdw_utils/soc_sdw_utils.c` | Added `codec_info_list` entry (4 DAIs) + SDCA endpoint bypass |
| `include/sound/soc_sdw_utils.h` | Added 7 function declarations |
| `sound/soc/intel/common/soc-acpi-intel-ptl-match.c` | Added ACPI match entry (endpoints, ADR, link array, machine table) |

### 10.3 Topology and Configuration Files

| File | Description |
|------|-------------|
| `sof-ptl-es9356-2ch.tplg` | SOF topology binary (90,007 bytes) |
| `nhlt-sof-ptl-es9356-2ch.bin` | Standalone NHLT blob (6,211 bytes, for debug) |
| `sof-soundwire_es9356/audio_config.xml` | DRAS audio routing configuration |
| `sof-soundwire_es9356/mixer_paths.xml` | ALSA mixer initial state and path overrides |

### 10.4 DRAS HAL Files

| File | Change |
|------|--------|
| `vendor/google/desktop/audio/dras/src/common/environment.rs` | Added ES9356 modalias detection (`p9356`), ALSA codec scanning (`es9356`), and `/proc/asound/card0/id` fallback |

### 10.5 SOF Build Files

| File | Relevance |
|------|-----------|
| `tools/topology/topology2/production/tplg-targets-ace3.cmake` | Added 2 ES9356 build targets (SDW-DMIC only, SDW-DMIC + PCH-DMIC) |
| `tools/topology/topology2/cavs-sdw.conf` | Base SoundWire topology template |
| `tools/topology/topology2/get_abi.sh` | ABI version extraction script |

### 10.6 Device Deployment Paths

| Local Source | Device Path |
|-------------|-------------|
| `sof-ptl-es9356-2ch.tplg` | `/vendor/firmware/intel/sof-ipc4-tplg/sof-ptl-es9356-2ch.tplg` |
| `snd-soc-es9356.ko` | `/vendor_dlkm/lib/modules/snd-soc-es9356.ko` |
| `snd-soc-sdw-utils.ko` | `/vendor_dlkm/lib/modules/snd-soc-sdw-utils.ko` |
| `audio_config.xml` | `/vendor/etc/audio/sof-soundwire_es9356/audio_config.xml` |
| `mixer_paths.xml` | `/vendor/etc/audio/sof-soundwire_es9356/mixer_paths.xml` |
| `com.android.hardware.audio.desktop.apex` | `/vendor/apex/com.android.hardware.audio.desktop.apex` |

--- End of Document ---
