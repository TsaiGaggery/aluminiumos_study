# RT712 + RT1320 SoundWire Audio Fix

## PTL GCS (Panther Lake Client Platform) - Topology Rebuild and ALSA Configuration

### Intel Panther Lake Client Platform - Android GKI Kernel
### Realtek RT712 SDCA Codec + RT1320 Smart Amplifier
### SOF (Sound Open Firmware) + SoundWire Bus
### February 2026

---

## Table of Contents

1. [Background Concepts](#1-background-concepts)
2. [Hardware Identification](#2-hardware-identification)
3. [Root Cause Analysis](#3-root-cause-analysis)
4. [SOF Topology Rebuild](#4-sof-topology-rebuild)
5. [ALSA Configuration Fix](#5-alsa-configuration-fix)
6. [Deployment and Verification](#6-deployment-and-verification)
7. [Known Issues and Notes](#7-known-issues-and-notes)
8. [Kernel Command Line](#8-kernel-command-line-audio-related)
9. [Complete File Reference](#9-complete-file-reference)

---

## 1. Background Concepts

This section explains the key technologies involved. If you are already familiar with SOF,
SoundWire, and ASoC, you can skip to Section 2.

### 1.1 What is SOF (Sound Open Firmware)?

SOF is an open-source audio DSP (Digital Signal Processor) firmware that runs on Intel platforms.
Modern Intel CPUs have a dedicated audio DSP built into the chip. Instead of sending raw audio
data directly to codecs, the CPU sends audio to this DSP first, which can apply processing like
equalization (EQ), dynamic range compression (DRC), volume control, and mixing -- all in
hardware, offloading work from the main CPU.

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │                        Intel CPU (Panther Lake)                     │
 │                                                                     │
 │   ┌─────────────┐         ┌──────────────────────────────────────┐  │
 │   │  Android OS  │         │         Audio DSP (built into CPU)  │  │
 │   │  AudioFlinger│  PCM    │  ┌─────┐   ┌────┐   ┌───────────┐  │  │
 │   │  + DRAS HAL  │────────►│  │Mixer│──►│ EQ │──►│ALH Copier │──┼──┼──► to codec
 │   │             │  data    │  └─────┘   └────┘   └───────────┘  │  │
 │   └─────────────┘         │                                      │  │
 │                            │  SOF firmware runs here              │  │
 │                            └──────────────────────────────────────┘  │
 └──────────────────────────────────────────────────────────────────────┘
```

SOF firmware is loaded by the Linux kernel driver (`sof-audio-pci-intel-ptl`) at boot time.
Along with the firmware, the driver also loads a **topology file** (`.tplg`) that tells the
DSP how to wire its internal processing blocks together.

### 1.2 What is a SOF Topology (.tplg)?

A topology file is a pre-compiled binary that defines the audio pipeline structure inside the
DSP. Think of it as a "wiring diagram" that tells the DSP:

- **What PCM devices to create** (e.g., "Speaker", "Jack Out", "HDMI1") -- these are the
  user-facing endpoints that Android apps play audio to.
- **What processing widgets to instantiate** (e.g., mixer, EQ, DRC, volume controls).
- **What DAI (Digital Audio Interface) links to create** -- these connect the DSP to physical
  audio hardware (codecs, amplifiers, HDMI outputs).
- **How to route audio** between all these components.

```
  Topology defines this pipeline inside the DSP:

  ┌─────────────────────────────────────────────────────────────────┐
  │  Audio DSP                                                      │
  │                                                                 │
  │  PCM 0 ──► [Host Copier] ──► [Gain] ──► [ALH Copier] ──► BE 0  │  ← Jack Out
  │  PCM 1 ◄── [Host Copier] ◄── [Gain] ◄── [ALH Copier] ◄── BE 1  │  ← Jack In
  │  PCM 2 ──► [Host Copier] ──► [Gain] ──► [ALH Copier] ──► BE 2  │  ← Speaker
  │  PCM 4 ◄── [Host Copier] ◄── [Gain] ◄── [ALH Copier] ◄── BE 4  │  ← SmartMic
  │  PCM 5 ──► [Host Copier] ──► [ALH Copier] ──► BE 7             │  ← HDMI1
  │  ...                                                            │
  └─────────────────────────────────────────────────────────────────┘
```

**Frontend (FE) PCM devices** are what the OS sees. When Android wants to play speaker audio,
it opens PCM device 2. The audio data enters the DSP through a "Host Copier" widget, flows
through processing, and exits through an "ALH Copier" widget to reach the physical codec.

**Backend (BE) DAI links** connect the DSP to physical codecs. Each BE link maps to a specific
codec on a specific bus. The kernel creates these BE links at boot time by scanning the hardware.

The topology file is compiled from human-readable configuration files (Topology2 format) using
the `alsatplg` tool. The compilation process takes parameters like `NUM_SDW_AMP_LINKS` that
control how many widgets are instantiated.

### 1.3 What is SoundWire?

SoundWire is a serial bus standard (MIPI SoundWire) for connecting audio devices to a
processor. It is the successor to older audio buses like I2S and HDA for codec connectivity.
Key characteristics:

- **Multi-drop bus**: Multiple devices can share a single SoundWire link (wire pair).
- **Multiple links**: A processor can have multiple independent SoundWire links (Link 0, 1, 2, 3...).
- **Device addressing**: Each device is identified by its Manufacturer ID (MFG_ID) and Part ID
  (PART_ID), similar to USB vendor/product IDs.
- **Stream synchronization**: Devices on different links can be synchronized for multi-channel audio.

On this PTL GCS board:

```
  Intel PTL CPU
  ┌──────────────────────────────────────────────────────┐
  │  Audio DSP                                           │
  │                                                      │
  │  SoundWire Link 0 ──────── (nothing connected)       │
  │  SoundWire Link 1 ──────── (nothing connected)       │
  │  SoundWire Link 2 ──────── RT1320 Smart Amplifier    │  ← drives the speaker
  │  SoundWire Link 3 ──────── RT712 SDCA Codec          │  ← headphone jack + mic
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

The kernel discovers SoundWire devices through ACPI tables in the BIOS. The `sdw_link_mask=0xF`
kernel parameter enables Links 0-3 (binary 1111). After enumeration, devices appear in sysfs:

```
/sys/bus/soundwire/devices/
  sdw:0:2:025d:1320:01    # RT1320 on Link 2 (MFG=025d, Part=1320)
  sdw:0:3:025d:0712:01    # RT712 on Link 3  (MFG=025d, Part=0712)
```

### 1.4 What is SDCA?

SDCA (SoundWire Device Class for Audio) is a standard that defines how audio devices on
SoundWire describe their capabilities. Each SDCA device exposes one or more **functions**:

| Function Type | Type ID | Purpose |
|---------------|---------|---------|
| SmartAmp | 1 | Speaker amplifier with protection (current/voltage sensing) |
| SmartMic | 3 | Digital microphone array with noise cancellation |
| UAJ | 6 | Universal Audio Jack (headphone + headset mic combo) |
| HID | 10 | Human Interface Device (jack detect button events) |

On this board, the ACPI DSDT declares these SDCA functions:

```
  RT1320 (Link 2):
    └── SmartAmp (type 1, address 0x4)     ← drives the physical speaker

  RT712 (Link 3):
    ├── UAJ (type 6, address 0x1)          ← headphone/headset jack
    ├── SmartMic (type 3, address 0x2)     ← DMIC capture
    └── HID (type 10, address 0x3)         ← jack insertion events
```

### 1.5 What is an ALH Copier?

ALH stands for **Audio Link Hub**. It is the hardware block inside the Intel audio DSP that
bridges the DSP's internal processing pipelines to external buses like SoundWire.

An **ALH Copier** is a widget (processing block) in the SOF topology that represents one
connection from the DSP to one SoundWire link. Each ALH copier widget must be paired with
exactly one **Backend (BE) DAI link** that the kernel creates.

```
  Inside the DSP:                          Outside the DSP:

  ┌─────────────────────────┐              ┌───────────────┐
  │  [EQ] ──► [ALH Copier]  │──── ALH ────►│  SoundWire    │──► RT1320 ──► Speaker
  │           (widget in     │   hardware   │  Link 2       │
  │            topology)     │   bridge     │               │
  └─────────────────────────┘              └───────────────┘

  The ALH Copier is the DSP's           The BE DAI link is the kernel's
  "output port" to a SoundWire          representation of this same
  link. Defined in the .tplg file.      connection. Created at runtime.
```

**Critical rule**: The number of ALH copier widgets in the topology **must exactly match**
the number of BE DAI links the kernel creates for that stream. If the topology has more ALH
copiers than the kernel has BE links, the topology will fail to load.

### 1.6 What is SmartAmp Aggregation?

SoundWire supports **aggregation**, where multiple devices on different links participate in
a single synchronized audio stream. This is commonly used for stereo speaker setups where the
left and right speakers are driven by separate amplifier chips on separate SoundWire links.

When aggregation is configured:
- Devices are assigned to the same **group_id** (e.g., `group_id = 1`).
- Each device in the group gets a **group_position** (0 = left, 1 = right).
- The kernel creates a single BE DAI link with **multiple CPU DAI slots** -- one per device
  in the aggregation group.
- The topology must create **one ALH copier widget per CPU DAI slot** in that aggregated link.

```
  Aggregated stereo SmartAmp (2 amps, 2 links):

  ┌─────────────────────────────────┐
  │  DSP                            │
  │                                 │       SoundWire
  │  PCM 2 (Speaker)                │       Link 2
  │    │                            │    ┌──────────┐
  │    ▼                            │    │          │
  │  [Host Copier]                  │    │  RT1320  │──► Left Speaker
  │    │                            │    │ (group 1 │
  │    ├──► [ALH Copier .0] ────────┼───►│  pos 0)  │
  │    │                            │    └──────────┘
  │    │                            │
  │    │                            │       SoundWire
  │    │                            │       Link 3
  │    │                            │    ┌──────────┐
  │    │                            │    │          │
  │    └──► [ALH Copier .1] ────────┼───►│  RT712   │──► Right Speaker
  │                                 │    │ (group 1 │
  │                                 │    │  pos 1)  │
  └─────────────────────────────────┘    └──────────┘

  BE DAI link: 1 link with 2 CPU DAI slots (aggregated)
  Topology:    2 ALH copier widgets (NUM_SDW_AMP_LINKS=2)
```

In the aggregated case, audio data is split across two SoundWire links simultaneously, with
the SoundWire controller keeping them in sync at the hardware level.

### 1.7 Frontend/Backend DAI Architecture (ASoC)

The Linux ASoC (ALSA System on Chip) framework uses a frontend/backend architecture to
separate user-facing PCM devices from physical codec connections:

```
  User Space (Android)
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Kernel Space
                              ┌──────────────────────────┐
  FE PCM 0 (Jack Out)   ────►│                          │────► BE 0 (SDW3 SimpleJack)  ──► RT712
  FE PCM 1 (Jack In)    ◄────│                          │◄──── BE 1 (SDW3 SimpleJack)  ◄── RT712
  FE PCM 2 (Speaker)    ────►│      SOF DSP Driver      │────► BE 2 (SDW2 SmartAmp)    ──► RT1320
  FE PCM 4 (Microphone) ◄────│   (topology defines the  │◄──── BE 4 (SDW3 SmartMic)    ◄── RT712
  FE PCM 5 (HDMI1)      ────►│    routing between       │────► BE 7 (HDA iDisp1)
  FE PCM 6 (HDMI2)      ────►│    FE and BE)            │────► BE 8 (HDA iDisp2)
  FE PCM 7 (HDMI3)      ────►│                          │────► BE 9 (HDA iDisp3)
                              └──────────────────────────┘

  FE (Frontend) = what apps see          BE (Backend) = physical hardware
  Created by topology file               Created by machine driver at boot
```

**FE PCM devices** are created by the topology file. Android apps (via AudioFlinger + DRAS HAL)
open these PCM devices to play or record audio.

**BE DAI links** are created by the **machine driver** (`sof_sdw`) at boot time. The machine
driver scans the ACPI tables to discover what codecs are present and creates one BE link per
codec endpoint. BE links are invisible to user-space applications.

The SOF DSP driver sits in the middle. When loading the topology, it must **connect each ALH
copier widget in the topology to a corresponding BE DAI link**. If this connection fails, the
entire topology fails to load, and no sound card is created.

### 1.8 DRAS (Desktop Remote Audio Service)

DRAS is the Android Audio HAL (Hardware Abstraction Layer) used on Intel desktop platforms.
It reads two XML configuration files:

- **`audio_config.xml`**: Maps logical audio paths (Speaker, Headphone, Mic, HDMI) to
  specific ALSA PCM device numbers and jack detection events.
- **`mixer_paths.xml`**: Defines ALSA mixer control defaults and path-specific overrides.
  Controls listed outside any `<path>` block are global defaults applied at startup.
  Controls inside a `<path>` block are applied when that audio path is activated.

These files are located at `/vendor/etc/audio/<card-name>/` on the device.

---

## 2. Hardware Identification

### 2.1 Platform Summary

| Property | Value |
|----------|-------|
| Platform | Intel Panther Lake Client Platform |
| Board Name | PTL GCS Evaluation Board |
| Kernel | 6.18.2-android17-0-maybe-dirty |
| SOF Firmware | 2.14.1.1 |
| Codec (Link 3) | Realtek RT712 SDCA (MFG_ID=0x025d, PART_ID=0x0712, v03) |
| Amplifier (Link 2) | Realtek RT1320 (MFG_ID=0x025d, PART_ID=0x1320, v03) |
| SoundWire Link Mask | 0xF (Links 0-3 enabled) |
| ALSA Card | sofsoundwire |
| Machine Driver | sof_sdw |

### 2.2 SoundWire Device Enumeration

After boot, the kernel discovers the SoundWire devices and registers them:

```
$ ls /sys/bus/soundwire/devices/
sdw-master-0-2
sdw-master-0-3
sdw:0:2:025d:1320:01    # RT1320 on Link 2
sdw:0:3:025d:0712:01    # RT712 on Link 3
```

The `sdw-master-0-2` and `sdw-master-0-3` entries are the SoundWire master controllers
(one per active link). The `sdw:0:2:025d:1320:01` entries are the discovered slave devices.

Device address format: `sdw:<manager>:<link>:<mfg_id>:<part_id>:<instance>`

SoundWire modalias strings (used for driver matching):
```
sdw:m025Dp0712v03c01 -> sdw:0:3:025d:0712:01   (RT712)
sdw:m025Dp1320v03c01 -> sdw:0:2:025d:1320:01   (RT1320)
```

### 2.3 SDCA Functions Detected

The kernel log shows the SDCA functions discovered from the ACPI DSDT:

```
acpi device:24: find_sdca_function: SDCA function SmartAmp (type 1) at 0x4
acpi device:20: find_sdca_function: SDCA function UAJ (type 6) at 0x1
acpi device:21: find_sdca_function: SDCA function SmartMic (type 3) at 0x2
acpi device:22: find_sdca_function: SDCA function HID (type 10) at 0x3
```

| Device | Function Type | SDCA Type ID | Function Address |
|--------|--------------|--------------|-----------------|
| RT1320 (Link 2) | SmartAmp | 1 | 0x4 |
| RT712 (Link 3) | UAJ (Universal Audio Jack) | 6 | 0x1 |
| RT712 (Link 3) | SmartMic | 3 | 0x2 |
| RT712 (Link 3) | HID | 10 | 0x3 |

**Important**: The SmartAmp function (type 1) is on the RT1320 only. The RT712 VB variant
is detected via `snd_soc_acpi_intel_sdca_is_device_rt712_vb()` which checks for the presence
of SmartMic (type 3), NOT SmartAmp. This means the RT712 is recognized as a codec (jack +
mic), not as a speaker amplifier.

### 2.4 NHLT Configuration

```
ACPI: NHLT 0x000000006FF74000 00053B (v00 INTEL PTL 00000002 01000013)
sof-audio-pci-intel-ptl: NHLT device BT(0) detected, ssp_mask 0x4
sof-audio-pci-intel-ptl: BT link detected in NHLT tables: 0x4
sof-audio-pci-intel-ptl: DMICs detected in NHLT tables: 0
```

NHLT (Non-HD Audio Link Table) is an ACPI table that describes the audio hardware
configuration. Key finding: **No PCH DMICs** are detected (DMICs = 0). This means the
kernel will NOT append a `-Nch` suffix to the topology filename. The topology file loaded
is exactly `sof-ptl-rt712-l3-rt1320-l2.tplg` as specified in the ACPI match table.

---

## 3. Root Cause Analysis

Two issues prevented audio from working on this board:

1. **Issue 1**: The SOF topology file failed to load because it expected 2 SmartAmp ALH
   copiers but the kernel only created 1 SmartAmp BE DAI link.
2. **Issue 2**: Even after fixing the topology, the RT1320 amplifier output stage was muted
   by default, so no sound reached the physical speaker.

### 3.1 Issue 1: Topology Mismatch (NUM_SDW_AMP_LINKS=2 vs 1)

#### 3.1.1 How the Kernel Selects a Topology

The kernel's `sof_sdw` machine driver uses an ACPI match table to select the correct topology
file. The match table is defined in:

**File**: `sound/soc/intel/common/soc-acpi-intel-ptl-match.c`

The relevant entry in `snd_soc_acpi_intel_ptl_sdw_machines[]`:

```c
{
    .link_mask = BIT(2) | BIT(3),                              // devices on Links 2 and 3
    .links = ptl_sdw_rt712_vb_l3_rt1320_l2,                    // device descriptions
    .drv_name = "sof_sdw",                                     // machine driver to use
    .machine_check = snd_soc_acpi_intel_sdca_is_device_rt712_vb,  // RT712 VB detection
    .sof_tplg_filename = "sof-ptl-rt712-l3-rt1320-l2.tplg",   // topology to load
    .get_function_tplg_files = sof_sdw_get_tplg_files,
},
```

The `link_mask = BIT(2) | BIT(3)` means this entry matches when SoundWire devices are detected
on Links 2 and 3. The `machine_check` function verifies that the device on Link 3 is
specifically an RT712 VB variant (by checking for SmartMic function type 3).

#### 3.1.2 Endpoint Definitions and Aggregation Groups

The match table entry references `ptl_sdw_rt712_vb_l3_rt1320_l2`, which describes what
endpoints each device has and whether they participate in aggregation.

**RT712 (Link 3)** uses `jack_amp_g1_dmic_endpoints` -- **3 endpoints**:

```c
static const struct snd_soc_acpi_endpoint jack_amp_g1_dmic_endpoints[] = {
    /* Jack Endpoint */
    {
        .num = 0,              // endpoint number within this device
        .aggregated = 0,       // NOT aggregated (standalone)
        .group_position = 0,
        .group_id = 0,         // group 0 = no aggregation
    },
    /* Amp Endpoint, work as spk_l_endpoint */
    {
        .num = 1,              // endpoint number 1
        .aggregated = 1,       // IS aggregated
        .group_position = 0,   // position 0 = LEFT speaker
        .group_id = 1,         // aggregation group 1
    },
    /* DMIC Endpoint */
    {
        .num = 2,              // endpoint number 2
        .aggregated = 0,       // NOT aggregated
        .group_position = 0,
        .group_id = 0,
    },
};
```

**RT1320 (Link 2)** uses `spk_r_endpoint` -- **1 endpoint**:

```c
static const struct snd_soc_acpi_endpoint spk_r_endpoint = {
    .num = 0,              // endpoint number 0
    .aggregated = 1,       // IS aggregated
    .group_position = 1,   // position 1 = RIGHT speaker
    .group_id = 1,         // aggregation group 1
};
```

Notice that **both** the RT712's amp endpoint (#1) and the RT1320's endpoint are in
**aggregation group 1**, forming a stereo SmartAmp pair:

```
  Aggregation Group 1 (as defined in the ACPI match table):

  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  RT712 (Link 3), Endpoint #1:    group_id=1, group_position=0   │
  │  "Left speaker amp"              (spk_l_endpoint equivalent)    │
  │                                                                  │
  │  RT1320 (Link 2), Endpoint #0:   group_id=1, group_position=1   │
  │  "Right speaker amp"             (spk_r_endpoint)               │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

This definition tells the kernel: "These two devices should work together as a stereo
speaker pair, synchronized across SoundWire Links 2 and 3."

#### 3.1.3 Link Address Arrays

The full link description ties everything together:

```c
static const struct snd_soc_acpi_adr_device rt712_vb_3_group1_adr[] = {
    {
        .adr = 0x000330025D071201ull,  /* Link 3, MFG 025D, Part 0712 */
        .num_endpoints = 3,            /* jack + amp + dmic */
        .endpoints = jack_amp_g1_dmic_endpoints,
        .name_prefix = "rt712"
    }
};

static const struct snd_soc_acpi_adr_device rt1320_2_group1_adr[] = {
    {
        .adr = 0x000230025D132001ull,  /* Link 2, MFG 025D, Part 1320 */
        .num_endpoints = 1,            /* amp only */
        .endpoints = &spk_r_endpoint,
        .name_prefix = "rt1320-1"
    }
};

static const struct snd_soc_acpi_link_adr ptl_sdw_rt712_vb_l3_rt1320_l2[] = {
    {
        .mask = BIT(3),                               /* Link 3 */
        .num_adr = ARRAY_SIZE(rt712_vb_3_group1_adr), /* 1 device */
        .adr_d = rt712_vb_3_group1_adr,               /* RT712 */
    },
    {
        .mask = BIT(2),                                /* Link 2 */
        .num_adr = ARRAY_SIZE(rt1320_2_group1_adr),    /* 1 device */
        .adr_d = rt1320_2_group1_adr,                  /* RT1320 */
    },
    {}  /* terminator */
};
```

#### 3.1.4 What the Original Topology Expected

The **original topology** was built with `NUM_SDW_AMP_LINKS=2`:

```
# From tplg-targets-ace3.cmake (line 127-129):
"cavs-sdw\;sof-ptl-rt712-l3-rt1320-l2\;PLATFORM=ptl,SDW_DMIC=1,NUM_SDW_AMP_LINKS=2,\
SDW_AMP_FEEDBACK=false,SDW_SPK_STREAM=Playback-SmartAmp,SDW_DMIC_STREAM=Capture-SmartMic,\
SDW_JACK_OUT_STREAM=Playback-SimpleJack,SDW_JACK_IN_STREAM=Capture-SimpleJack"
```

With `NUM_SDW_AMP_LINKS=2`, the topology instantiates **2 ALH copier widgets** for the
SmartAmp speaker pipeline:

- `alh-copier.Playback-SmartAmp.0` -- intended for the left amp (RT712 amp endpoint)
- `alh-copier.Playback-SmartAmp.1` -- intended for the right amp (RT1320)

This is the design for a stereo aggregated SmartAmp setup:

```
  What the topology (NUM_SDW_AMP_LINKS=2) expected:

  ┌─────────────────────────────────────────────────────────────┐
  │  DSP                                                        │
  │                                                             │
  │  PCM 2 (Speaker)                                            │
  │    │                                                        │
  │    ▼                                                        │
  │  [Host Copier]                                              │
  │    │                                                        │
  │    ├──► [ALH Copier .0]  "Playback-SmartAmp" ──► SDW Link 3 ──► RT712 (amp endpoint)
  │    │     stream name matches ──────────────────┐             │     └──► Left Speaker
  │    │                                           │             │
  │    └──► [ALH Copier .1]  "Playback-SmartAmp" ──► SDW Link 2 ──► RT1320
  │          stream name matches ──────────────────┘             │     └──► Right Speaker
  │                                                             │
  │  Expected BE DAI link: 1 aggregated link with 2 CPU slots   │
  └─────────────────────────────────────────────────────────────┘
```

#### 3.1.5 What the Kernel Actually Created

However, when the `sof_sdw` machine driver examines the hardware at runtime, it only creates
**1 SmartAmp BE DAI link** with **1 CPU DAI slot** (for the RT1320 only):

```
DAI link[2] 'SDW2-Playback-SmartAmp': codecs=1 cpus=1 plats=1 no_pcm=1 ignore=0
  codec[0]: name='sdw:0:2:025d:1320:01' dai='rt1320-aif1'
  cpu[0]: name='(null)' dai='SDW2 Pin2'
```

The RT712's amp endpoint (endpoint #1 from `jack_amp_g1_dmic_endpoints`) is **NOT** used to
create a second SmartAmp CPU DAI slot. The kernel decides this based on the actual SDCA
functions it discovers -- the RT712 has UAJ, SmartMic, and HID functions, but no SmartAmp
function. The SmartAmp function only exists on the RT1320.

This is the actual hardware reality:

```
  What actually exists on this board:

  ┌─────────────────────────────────────────┐
  │  DSP                                    │
  │                                         │       SoundWire Link 2
  │  PCM 2 (Speaker)                        │    ┌──────────────────┐
  │    │                                    │    │                  │
  │    ▼                                    │    │     RT1320       │
  │  [Host Copier]                          │    │   SmartAmp func  │──► Speaker
  │    │                                    │    │                  │    (mono or
  │    └──► [ALH Copier .0] ────────────────┼───►│  (the ONLY amp)  │     stereo on
  │                                         │    │                  │     one chip)
  │                                         │    └──────────────────┘
  │  BE link: 1 link, 1 CPU slot            │
  │  Topology needs: 1 ALH copier           │       SoundWire Link 3
  └─────────────────────────────────────────┘    ┌──────────────────┐
                                                 │     RT712        │
                                                 │   UAJ + SmartMic │
                                                 │   + HID          │
                                                 │   (NO SmartAmp)  │
                                                 └──────────────────┘
```

#### 3.1.6 The Failure Mechanism in topology.c

When the SOF driver loads the topology, it processes each widget and connects DAI widgets to
BE links. The critical function is `sof_connect_dai_widget()` in
`sound/soc/sof/topology.c` (line 1068):

```c
static int sof_connect_dai_widget(struct snd_soc_component *scomp,
                                  struct snd_soc_dapm_widget *w,
                                  struct snd_soc_tplg_dapm_widget *tw,
                                  struct snd_sof_dai *dai)
{
    struct snd_soc_card *card = scomp->card;
    struct snd_soc_pcm_runtime *rtd;
    int i;

    /* Step 1: Find the BE DAI link whose stream_name matches
     *         this widget's sname ("Playback-SmartAmp") */
    list_for_each_entry(rtd, &card->rtd_list, list) {
        if (rtd->dai_link->stream_name) {
            if (!strcmp(rtd->dai_link->stream_name, w->sname)) {
                /* exact match found */
                break;
            }
        }
    }

    /* Step 2: Iterate through the BE link's CPU DAI slots
     *         looking for an unassigned slot */
    for_each_rtd_cpu_dais(rtd, i, cpu_dai) {
        if (!snd_soc_dai_get_widget(cpu_dai, stream)) {
            snd_soc_dai_set_widget(cpu_dai, stream, w);
            break;  /* found a free slot, assign this widget */
        }
    }

    /* Step 3: If we exhausted all CPU DAI slots without finding
     *         a free one, ERROR */
    if (i == rtd->dai_link->num_cpus) {
        dev_err(scomp->dev, "error: can't find BE for DAI %s\n", w->name);
        return -EINVAL;
    }
}
```

Here is exactly what happens during topology loading on this board:

```
  Timeline of widget processing:

  1. ALH Copier "alh-copier.Playback-SmartAmp.0" is processed:
     - sname = "Playback-SmartAmp"
     - Searches BE DAI links for stream_name containing "Playback-SmartAmp"
     - Finds: DAI link[2] "SDW2-Playback-SmartAmp" (num_cpus=1)
     - Iterates CPU DAI slots:
       - cpu[0] (SDW2 Pin2): unassigned → ASSIGN widget .0 here
     - Success ✓

  2. ALH Copier "alh-copier.Playback-SmartAmp.1" is processed:
     - sname = "Playback-SmartAmp"
     - Searches BE DAI links → same link[2] "SDW2-Playback-SmartAmp"
     - Iterates CPU DAI slots:
       - cpu[0] (SDW2 Pin2): already assigned to .0 → skip
       - i reaches num_cpus (1) → NO FREE SLOT
     - ERROR: "can't find BE for DAI alh-copier.Playback-SmartAmp.1"
     - Topology loading ABORTS ✗
```

The topology expected 2 CPU DAI slots in the SmartAmp BE link (one for each amp in the
aggregation group), but the kernel only created 1 slot (for the RT1320 only).

#### 3.1.7 Failure Kernel Log (Before Fix)

```
sof-audio-pci-intel-ptl 0000:00:1f.3: Topology file: intel/sof-ipc4-tplg/sof-ptl-rt712-l3-rt1320-l2.tplg
sof-audio-pci-intel-ptl 0000:00:1f.3: Topology: ABI 3:29:1 Kernel ABI 3:23:1

sof_sdw sof_sdw: DAI link[0] 'SDW3-Playback-SimpleJack': codecs=1 cpus=1
  codec[0]: name='sdw:0:3:025d:0712:01' dai='rt712-sdca-aif1'
sof_sdw sof_sdw: DAI link[1] 'SDW3-Capture-SimpleJack': codecs=1 cpus=1
  codec[0]: name='sdw:0:3:025d:0712:01' dai='rt712-sdca-aif1'
sof_sdw sof_sdw: DAI link[2] 'SDW2-Playback-SmartAmp': codecs=1 cpus=1
  codec[0]: name='sdw:0:2:025d:1320:01' dai='rt1320-aif1'
sof_sdw sof_sdw: DAI link[3] 'SDW3-Capture-SmartMic': codecs=1 cpus=1
  codec[0]: name='sdw:0:3:025d:0712:01' dai='rt712-sdca-aif3'

sof-audio-pci-intel-ptl 0000:00:1f.3: error: can't find BE for DAI alh-copier.Playback-SmartAmp.1
sof-audio-pci-intel-ptl 0000:00:1f.3: failed to add widget type 27 name : alh-copier.Playback-SmartAmp.1 stream Playback-SmartAmp
sof_sdw sof_sdw: ASoC: failed to load widget alh-copier.Playback-SmartAmp.1
sof_sdw sof_sdw: ASoC: topology: could not load header: -22
sof-audio-pci-intel-ptl 0000:00:1f.3: tplg intel/sof-ipc4-tplg/sof-ptl-rt712-l3-rt1320-l2.tplg component load failed -22
sof-audio-pci-intel-ptl 0000:00:1f.3: error: failed to load DSP topology -22
sof_sdw sof_sdw: ASoC: failed to instantiate card -22
sof_sdw sof_sdw: error -EINVAL: snd_soc_register_card failed -22
```

Key observations from this log:
- **DAI link[2]**: `codecs=1 cpus=1` -- only 1 CPU slot, not 2 as aggregation would require.
- **"can't find BE for DAI alh-copier.Playback-SmartAmp.1"**: The second ALH copier widget
  cannot find a free CPU DAI slot.
- **Error -22 (-EINVAL)**: The entire sound card registration fails. No ALSA card is created.
  The system has no audio at all.

#### 3.1.8 The Fix: NUM_SDW_AMP_LINKS=1

The fix is to rebuild the topology with `NUM_SDW_AMP_LINKS=1`, which creates only 1 ALH
copier widget for the SmartAmp pipeline:

```
  Fixed topology (NUM_SDW_AMP_LINKS=1):

  ┌─────────────────────────────────────────┐
  │  DSP                                    │
  │                                         │       SoundWire Link 2
  │  PCM 2 (Speaker)                        │    ┌──────────────────┐
  │    │                                    │    │                  │
  │    ▼                                    │    │     RT1320       │
  │  [Host Copier]                          │    │   SmartAmp       │──► Speaker
  │    │                                    │    │                  │
  │    └──► [ALH Copier .0] ────────────────┼───►│                  │
  │                                         │    └──────────────────┘
  │  1 ALH copier ← matches → 1 CPU slot   │
  │  Topology loads successfully ✓          │
  └─────────────────────────────────────────┘
```

### 3.2 Issue 2: Amplifier Output Terminal Muted

Even after the topology loaded successfully and all PCM devices appeared, playing audio
produced no sound from the physical speaker. Investigation revealed that the RT1320 and
RT712 both have SDCA Output Terminal (OT23) switches that control the output stage.

#### 3.2.1 What are OT23 Switches?

In the SDCA specification, audio processing inside a codec follows a chain of **Functional
Units (FU)** and **Terminals**:

```
  Inside the RT1320 codec:

  SoundWire bus                                            Physical speaker
  ───────────►  [Input Terminal] ──► [FU: Volume] ──► [OT23 Switch] ──► [Amplifier] ──► 🔊
                                                         │
                                                    If OFF: audio
                                                    is blocked here
                                                    (no signal to amp)
```

OT23 (Output Terminal 23) is the final gate before the physical amplifier stage. When the
OT23 switch is Off, the entire output is muted at the hardware level, regardless of volume
settings or path configuration.

#### 3.2.2 Default State Problem

The RT1320 driver initializes these switches to Off (0) by default:

```
tinymix -D 0 'rt1320-1 OT23 L Switch'   →  Off
tinymix -D 0 'rt1320-1 OT23 R Switch'   →  Off
tinymix -D 0 'rt712 OT23 L Switch'      →  Off
tinymix -D 0 'rt712 OT23 R Switch'      →  Off
```

DRAS (the Android Audio HAL) does not manage these codec-specific switches. DRAS only
controls higher-level path switches like `Speaker Switch` and `Headphone Switch` (defined
in the `<path>` blocks of `mixer_paths.xml`). The OT23 switches are codec-internal
controls that DRAS has no knowledge of.

#### 3.2.3 The Fix: Add OT23 Defaults to mixer_paths.xml

The fix is to add the OT23 switches as global defaults in `mixer_paths.xml` (outside any
`<path>` block), so they are always set to On at DRAS startup:

```diff
 	<ctl name="Speaker Switch" value="0" />
 	<ctl name="Dmic0 Capture Switch" value="0" />
 	<ctl name="rt712 FU0F Capture Switch" value="0" />
+	<ctl name="rt1320-1 OT23 L Switch" value="1" />
+	<ctl name="rt1320-1 OT23 R Switch" value="1" />
+	<ctl name="rt712 OT23 L Switch" value="1" />
+	<ctl name="rt712 OT23 R Switch" value="1" />
 	<path name="Speaker">
```

With this change, the audio signal chain becomes:

```
  After fix:

  Android App ──► AudioFlinger ──► DRAS HAL ──► ALSA PCM 2 ──► SOF DSP
       │
       └──► [Host Copier] ──► [Gain] ──► [ALH Copier .0] ──► SoundWire Link 2
                                                                     │
                                                                     ▼
                                                              RT1320 codec
                                                    [Input] ──► [Volume] ──► [OT23: ON ✓] ──► [Amp] ──► 🔊
```

---

## 4. SOF Topology Rebuild

### 4.1 SOF Repository

```
/home/gaggery/nvme/sof/
```

Topology source files are in `tools/topology/topology2/`. The build uses the Docker image
`thesofproject/sof:latest` which includes `alsatplg 1.2.13` (the topology compiler).

### 4.2 Original Build Target (Broken)

From `tools/topology/topology2/production/tplg-targets-ace3.cmake` (line 127):

```cmake
"cavs-sdw\;sof-ptl-rt712-l3-rt1320-l2\;PLATFORM=ptl,SDW_DMIC=1,NUM_SDW_AMP_LINKS=2,\
SDW_AMP_FEEDBACK=false,SDW_SPK_STREAM=Playback-SmartAmp,SDW_DMIC_STREAM=Capture-SmartMic,\
SDW_JACK_OUT_STREAM=Playback-SimpleJack,SDW_JACK_IN_STREAM=Capture-SimpleJack"
```

The key parameter is **`NUM_SDW_AMP_LINKS=2`**, which creates 2 ALH copier widgets for
the SmartAmp pipeline.

### 4.3 Fixed Build Parameters

Changed `NUM_SDW_AMP_LINKS=2` to `NUM_SDW_AMP_LINKS=1`:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| PLATFORM | ptl | Panther Lake -- sets platform-specific defaults (DMIC_DRIVER_VERSION=5, NUM_HDMIS=3) |
| SDW_DMIC | 1 | Enable SoundWire DMIC pipeline (captures from RT712 SmartMic via BE ID 4) |
| **NUM_SDW_AMP_LINKS** | **1** | **One SoundWire amplifier link (RT1320 only, no aggregation)** |
| SDW_AMP_FEEDBACK | false | No amplifier feedback/sensing channel (disables IV sense pipeline) |
| SDW_SPK_STREAM | Playback-SmartAmp | SoundWire stream name for speaker output (must match BE link name) |
| SDW_DMIC_STREAM | Capture-SmartMic | SoundWire stream name for DMIC capture |
| SDW_JACK_OUT_STREAM | Playback-SimpleJack | SoundWire stream name for headphone jack output |
| SDW_JACK_IN_STREAM | Capture-SimpleJack | SoundWire stream name for headset mic input |

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
      > $BUILDDIR/sof-ptl-rt712-l3-rt1320-l2.conf

    # Step 3: Compile topology with NUM_SDW_AMP_LINKS=1
    ALSA_CONFIG_DIR=$TPLG2 \
    LD_LIBRARY_PATH=/home/sof/work/alsa/alsa-lib/src/.libs \
    /home/sof/work/tools/bin/alsatplg -v 1 \
      -I $TPLG2/ \
      -D "PLATFORM=ptl,SDW_DMIC=1,\
NUM_SDW_AMP_LINKS=1,\
SDW_AMP_FEEDBACK=false,\
SDW_SPK_STREAM=Playback-SmartAmp,\
SDW_DMIC_STREAM=Capture-SmartMic,\
SDW_JACK_OUT_STREAM=Playback-SimpleJack,\
SDW_JACK_IN_STREAM=Capture-SimpleJack" \
      -p -c $BUILDDIR/sof-ptl-rt712-l3-rt1320-l2.conf \
      -o $SOF/sof-ptl-rt712-l3-rt1320-l2.tplg
  '
```

### 4.5 Build Steps Explained

The topology build process works as follows:

```
  Build Pipeline:

  ┌──────────────┐     ┌──────────────────┐     ┌─────────────────────────────┐
  │ get_abi.sh   │     │ cavs-sdw.conf    │     │ alsatplg compiler           │
  │              │     │ (base SoundWire  │     │                             │
  │ Extracts ABI │     │  topology        │     │ Reads .conf input           │
  │ version from │     │  template with   │     │ Applies -D defines          │
  │ SOF source   │     │  conditional     │     │ Resolves includes (-I)      │
  │              │     │  #ifdef blocks)  │     │ Outputs binary .tplg        │
  └──────┬───────┘     └────────┬─────────┘     └──────────────┬──────────────┘
         │                      │                               │
         │   abi.conf           │                               │
         └──────────┐           │                               │
                    ▼           ▼                               │
              ┌─────────────────────┐                          │
              │ cat abi.conf        │     -D "NUM_SDW_AMP_     │
              │     cavs-sdw.conf   │      LINKS=1,..."        │
              │  > combined.conf    │──────────────────────────►│
              └─────────────────────┘                          │
                                                               ▼
                                                    ┌────────────────────┐
                                                    │ .tplg binary file  │
                                                    │ (65,722 bytes)     │
                                                    └────────────────────┘
```

1. **get_abi.sh**: Extracts the SOF ABI version header from the source tree (`ABI 3:29:1`)
   and writes it as a topology configuration snippet. This version is embedded in the `.tplg`
   binary so the kernel can check compatibility.

2. **cat abi.conf cavs-sdw.conf**: Concatenates the ABI header with the base SoundWire
   topology template. The `cavs-sdw.conf` file contains conditional blocks (`#ifdef`) that
   are resolved by the preprocessor based on the `-D` defines.

3. **alsatplg -p**: The `-p` flag enables Topology2 preprocessor mode. The preprocessor
   expands the human-readable configuration into ALSA topology format, then compiles it to
   binary. The `-D` flag passes defines that control which pipelines are instantiated:
   - `NUM_SDW_AMP_LINKS=1` → instantiate 1 ALH copier for SmartAmp
   - `SDW_DMIC=1` → include the SmartMic capture pipeline
   - `PLATFORM=ptl` → use Panther Lake platform defaults

4. **-I $TPLG2/**: Include search path for topology configuration fragments. The `cavs-sdw.conf`
   template includes many sub-files for pipeline definitions, widget definitions, etc.

5. **Docker container**: The build runs inside `thesofproject/sof:latest` because it contains
   the correct version of `alsatplg` (1.2.13) and alsa-lib with Topology2 support. Building
   with a system-installed `alsatplg` may produce incorrect results or fail if the version
   doesn't match.

### 4.6 Build Output Comparison

| | Original (broken) | Fixed |
|---|---|---|
| NUM_SDW_AMP_LINKS | 2 | 1 |
| SmartAmp ALH copiers | `alh-copier.Playback-SmartAmp.0` and `.1` | `alh-copier.Playback-SmartAmp.0` only |
| File size | N/A (stock from sof-bin) | 65,722 bytes |

### 4.7 Topology Output: PCM and Backend DAI Layout

#### 4.7.1 Frontend PCM Devices

These are the user-facing PCM devices created by the fixed topology:

| PCM ID | Name | Direction | Source | Stream Name |
|--------|------|-----------|--------|-------------|
| 0 | Jack Out | Playback | RT712 aif1 (SimpleJack) | Playback-SimpleJack |
| 1 | Jack In | Capture | RT712 aif1 (SimpleJack) | Capture-SimpleJack |
| 2 | Speaker | Playback | RT1320 aif1 (SmartAmp) | Playback-SmartAmp |
| 4 | Microphone | Capture | RT712 aif3 (SmartMic) | Capture-SmartMic |
| 5 | HDMI1 | Playback | Intel HDA iDisp1 | iDisp1 |
| 6 | HDMI2 | Playback | Intel HDA iDisp2 | iDisp2 |
| 7 | HDMI3 | Playback | Intel HDA iDisp3 | iDisp3 |
| 31 | Deepbuffer Jack Out | Playback | RT712 aif1 (deep buffer) | Playback-SimpleJack |

#### 4.7.2 Backend DAI Links

These are the hardware-facing DAI links created by the kernel:

| BE ID | Name | Type | Stream Name | Codec |
|-------|------|------|-------------|-------|
| 0 | Playback-SimpleJack | SDW ALH | SDW3-Playback | RT712 rt712-sdca-aif1 |
| 1 | Capture-SimpleJack | SDW ALH | SDW3-Capture | RT712 rt712-sdca-aif1 |
| 2 | Playback-SmartAmp | SDW ALH | SDW2-Playback | RT1320 rt1320-aif1 |
| 4 | Capture-SmartMic | SDW ALH | SDW3-Capture | RT712 rt712-sdca-aif3 |
| 7 | iDisp1 | HDA HDMI | iDisp1 | ehdaudio0D2 |
| 8 | iDisp2 | HDA HDMI | iDisp2 | ehdaudio0D2 |
| 9 | iDisp3 | HDA HDMI | iDisp3 | ehdaudio0D2 |

The stream names in the FE PCM devices and the BE DAI links must match for the topology
loader to connect them. For example, the Speaker FE (PCM 2) uses stream "Playback-SmartAmp"
which matches BE link[2] "SDW2-Playback-SmartAmp" (substring match).

---

## 5. ALSA Configuration Fix

### 5.1 mixer_paths.xml: Output Terminal Switches

As explained in Section 3.2, the RT1320 and RT712 OT23 switches default to Off. The fix
adds them as global defaults that are always On.

### 5.2 mixer_paths.xml Changes

File: `new_files/device/google/desktop/fatcat/alsa/sof-soundwire_rt712_rt1320/mixer_paths.xml`

Added 4 default controls (always On at boot):

```diff
 	<ctl name="Speaker Switch" value="0" />
 	<ctl name="Dmic0 Capture Switch" value="0" />
 	<ctl name="rt712 FU0F Capture Switch" value="0" />
+	<ctl name="rt1320-1 OT23 L Switch" value="1" />
+	<ctl name="rt1320-1 OT23 R Switch" value="1" />
+	<ctl name="rt712 OT23 L Switch" value="1" />
+	<ctl name="rt712 OT23 R Switch" value="1" />
 	<path name="Speaker">
```

These are set as global defaults (outside any `<path>` block), so they are always applied
at DRAS startup regardless of which audio path is selected.

### 5.3 Complete mixer_paths.xml

```xml
<?xml version='1.0' encoding='utf-8'?>
<mixer enum_mixer_numeric_fallback="true">
	<!--RT712 SDCA codec + RT1320 smart amplifier mixer paths-->
	<ctl name="Headphone Switch" value="0" />
	<ctl name="Headset Mic Switch" value="0" />
	<ctl name="rt712 ADC 23 Mux" value="MIC2" />
	<ctl name="rt712 FU44 Boost Volume" value="3" />
	<ctl name="rt712 FU1E Capture Volume" value="63" />
	<ctl name="rt712 FU0F Capture Volume" value="63" />
	<ctl name="rt712 FU05 Playback Volume" value="87" />
	<ctl name="rt712 FU06 Playback Volume" value="87" />
	<ctl name="Speaker Switch" value="0" />
	<ctl name="Dmic0 Capture Switch" value="0" />
	<ctl name="rt712 FU0F Capture Switch" value="0" />
	<ctl name="rt1320-1 OT23 L Switch" value="1" />
	<ctl name="rt1320-1 OT23 R Switch" value="1" />
	<ctl name="rt712 OT23 L Switch" value="1" />
	<ctl name="rt712 OT23 R Switch" value="1" />
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
		<ctl name="rt712 FU0F Capture Switch" value="1" />
	</path>
	<path name="HDMI1" />
	<path name="HDMI2" />
	<path name="HDMI3" />
	<path name="HDMI4" />
	<path name="SCOLineOut" />
	<path name="SCOLineIn" />
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
  │    <ctl name="rt1320-1 OT23 L Switch" value="1" /> ← Always On │
  │    ...                                                          │
  │                                                                 │
  │    ── Path overrides (applied when path is activated) ──        │
  │    <path name="Speaker">                                        │
  │      <ctl name="Speaker Switch" value="1" />   ← On for speaker│
  │    </path>                                                      │
  │    <path name="Headphone">                                      │
  │      <ctl name="Headphone Switch" value="1" /> ← On for HP     │
  │    </path>                                                      │
  │    ...                                                          │
  │                                                                 │
  │  </mixer>                                                       │
  └─────────────────────────────────────────────────────────────────┘

  When DRAS starts:
    1. All global defaults are applied (everything outside <path> blocks)
    2. When "Speaker" path activates: Speaker Switch → On
    3. When "Speaker" path deactivates: Speaker Switch → back to default (Off)
    4. OT23 switches stay On always (they're global, not in any path)
```

### 5.4 Mixer Control Reference

All 62 mixer controls registered for this sound card are categorized below:

| Control | Type | Default | Purpose |
|---------|------|---------|---------|
| **Playback Path Controls** | | | |
| Headphone Switch | bool | Off | Enable headphone output path |
| Speaker Switch | bool | Off | Enable speaker output path |
| rt712 FU05 Playback Volume | int | 87 | RT712 headphone DAC volume |
| rt712 FU06 Playback Volume | int | 87 | RT712 headphone DAC volume (ch2) |
| **Capture Path Controls** | | | |
| Headset Mic Switch | bool | Off | Enable headset mic input path |
| Dmic0 Capture Switch | bool | Off | Enable DMIC capture path |
| rt712 FU0F Capture Switch | bool | Off | Enable RT712 headset mic ADC |
| rt712 FU1E Capture Volume | int | 63 | RT712 DMIC capture volume |
| rt712 FU0F Capture Volume | int | 63 | RT712 headset mic capture volume |
| rt712 FU44 Boost Volume | int | 3 | RT712 mic boost gain |
| rt712 ADC 23 Mux | enum | MIC2 | RT712 ADC input source selection |
| **Output Terminal Controls (OT23)** | | | |
| rt1320-1 OT23 L Switch | bool | **On** | RT1320 left output terminal gate |
| rt1320-1 OT23 R Switch | bool | **On** | RT1320 right output terminal gate |
| rt712 OT23 L Switch | bool | **On** | RT712 left output terminal gate |
| rt712 OT23 R Switch | bool | **On** | RT712 right output terminal gate |

### 5.5 audio_config.xml

The `audio_config.xml` for this board maps audio paths to ALSA PCM devices. No changes
were needed:

```xml
<?xml version='1.0' encoding='utf-8'?>
<audio_config>
	<!--RT712 SDCA codec + RT1320 smart amplifier on SoundWire-->
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
       ├── Speaker     → open hw:sofsoundwire,2  (PCM 2, SmartAmp)
       ├── Headphone   → open hw:sofsoundwire,0  (PCM 0, SimpleJack)
       ├── InternalMic → open hw:sofsoundwire,10 (PCM 10, not available*)
       ├── Mic         → open hw:sofsoundwire,1  (PCM 1, SimpleJack capture)
       ├── HDMI1       → open hw:sofsoundwire,5  (PCM 5, HDA display)
       ├── HDMI2       → open hw:sofsoundwire,6  (PCM 6, HDA display)
       └── HDMI3       → open hw:sofsoundwire,7  (PCM 7, HDA display)

  * PCM 10 does not exist in current topology (no PCH DMIC, see Section 7.2)
```

---

## 6. Deployment and Verification

### 6.1 Topology Deployment

```bash
adb -s <device_ip>:5555 root
adb -s <device_ip>:5555 remount
adb -s <device_ip>:5555 push sof-ptl-rt712-l3-rt1320-l2.tplg \
    /vendor/firmware/intel/sof-ipc4-tplg/sof-ptl-rt712-l3-rt1320-l2.tplg
adb -s <device_ip>:5555 reboot
```

**Why a reboot is required**: The SOF firmware and topology are loaded during the audio PCI
driver's probe function, which runs at boot time. The topology binary is read from
`/vendor/firmware/` by the kernel's firmware loader. Replacing the file on disk requires a
reboot for the kernel to read the new version.

### 6.2 ALSA Config Deployment

```bash
adb -s <device_ip>:5555 remount
adb -s <device_ip>:5555 push mixer_paths.xml \
    /vendor/etc/audio/sof-soundwire_rt712_rt1320/mixer_paths.xml

# Restart audio server to reload mixer paths (no reboot needed)
adb -s <device_ip>:5555 shell kill $(pidof audioserver)
```

**Why no reboot is needed**: DRAS reads `mixer_paths.xml` when the audioserver process starts.
Killing audioserver causes Android's init system to restart it, which re-reads the XML files.
The ALSA kernel driver and SOF topology remain loaded.

### 6.3 Verification: Topology Loading

```bash
adb shell dmesg | grep -E "Topology|can.t find BE|component load|DAI link"
```

Expected output (after fix):
```
sof-audio-pci-intel-ptl: Topology file: intel/sof-ipc4-tplg/sof-ptl-rt712-l3-rt1320-l2.tplg
sof-audio-pci-intel-ptl: Topology: ABI 3:29:1 Kernel ABI 3:23:1
sof_sdw: DAI link[0] 'SDW3-Playback-SimpleJack': codecs=1 cpus=1
sof_sdw:   codec[0]: name='sdw:0:3:025d:0712:01' dai='rt712-sdca-aif1'
sof_sdw:   cpu[0]: dai='SDW3 Pin2'
sof_sdw: DAI link[1] 'SDW3-Capture-SimpleJack': codecs=1 cpus=1
sof_sdw:   codec[0]: name='sdw:0:3:025d:0712:01' dai='rt712-sdca-aif1'
sof_sdw:   cpu[0]: dai='SDW3 Pin3'
sof_sdw: DAI link[2] 'SDW2-Playback-SmartAmp': codecs=1 cpus=1
sof_sdw:   codec[0]: name='sdw:0:2:025d:1320:01' dai='rt1320-aif1'
sof_sdw:   cpu[0]: dai='SDW2 Pin2'
sof_sdw: DAI link[3] 'SDW3-Capture-SmartMic': codecs=1 cpus=1
sof_sdw:   codec[0]: name='sdw:0:3:025d:0712:01' dai='rt712-sdca-aif3'
sof_sdw:   cpu[0]: dai='SDW3 Pin4'
sof_sdw: DAI link[4] 'iDisp1': codecs=1 cpus=1
sof_sdw: DAI link[5] 'iDisp2': codecs=1 cpus=1
sof_sdw: DAI link[6] 'iDisp3': codecs=1 cpus=1
```

**What to look for:**
- No "can't find BE" or "component load failed" errors.
- DAI link[2] shows `codecs=1 cpus=1` -- matches our topology's single ALH copier.
- All 7 DAI links are successfully created.

### 6.4 Verification: ALSA Card and PCM Devices

```bash
# Check ALSA card
adb shell cat /proc/asound/cards
# Expected:
#  0 [sofsoundwire   ]: sof-soundwire - sof-soundwire
#                       IntelCorporation-PantherLakeClientPlatform-0.1-TBD

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
# 00-31: Deepbuffer Jack Out (*) :  : playback 1
```

### 6.5 Verification: OT23 Switches

```bash
adb shell tinymix -D 0 'rt1320-1 OT23 L Switch'
# Expected: On
adb shell tinymix -D 0 'rt1320-1 OT23 R Switch'
# Expected: On
adb shell tinymix -D 0 'rt712 OT23 L Switch'
# Expected: On
adb shell tinymix -D 0 'rt712 OT23 R Switch'
# Expected: On
```

### 6.6 Verification: Speaker Playback (Kernel Level)

This tests the speaker output path at the ALSA kernel level, bypassing Android audio:

```bash
# Enable speaker path and play a tone directly via ALSA
adb shell tinymix -D 0 'Speaker Switch' 1
adb shell tinyplay /data/local/tmp/tone.wav -D 0 -d 2
```

If audio is heard from the speaker, the kernel-level path is working.

### 6.7 Verification: OS Audio Playback + Kernel Mic Capture

This is the full end-to-end test. It plays audio through the Android audio stack (OS level)
while simultaneously recording from the kernel-level DMIC. This verifies the complete signal
chain:

```
  Full signal path tested:

  Android App (stagefright)
       │
       ▼
  AudioFlinger (Android audio mixer)
       │
       ▼
  DRAS HAL (opens hw:sofsoundwire,2)
       │
       ▼
  ALSA PCM 2 (Speaker)
       │
       ▼
  SOF DSP (Host Copier → Gain → ALH Copier)
       │
       ▼
  SoundWire Link 2
       │
       ▼
  RT1320 SmartAmp → Physical Speaker → 🔊 Sound waves through air
                                              │
                                              ▼
                                        DMIC microphone (on RT712)
                                              │
                                              ▼
                                        RT712 SmartMic
                                              │
                                              ▼
                                        SoundWire Link 3
                                              │
                                              ▼
                                        SOF DSP (ALH Copier → Gain → Host Copier)
                                              │
                                              ▼
                                        ALSA PCM 4 (Microphone)
                                              │
                                              ▼
                                        tinycap (kernel-level recording)
```

Test commands:

```bash
# Push a test tone (1kHz sine, 48kHz, 16-bit stereo, 3 seconds)
adb push tone1k.wav /data/local/tmp/

# Start kernel-level mic capture in background, then play from OS
adb shell "
tinycap /data/local/tmp/record.wav -D 0 -d 4 -c 2 -r 48000 -b 16 -T 8 &
sleep 1
stagefright -o -a /data/local/tmp/tone1k.wav &
sleep 5
wait
"

# Pull and verify recording
adb pull /data/local/tmp/record.wav .
```

**Expected**: The recording should show clear 1kHz tone signal during the playback window,
with quiet noise floor before and after.

**Verified result from this system:**

| Time Window | RMS Level | Peak | Frequency | Description |
|-------------|-----------|------|-----------|-------------|
| 0.0 - 1.0s | ~90 | ~280 | noise | Before playback (noise floor) |
| 1.0 - 4.0s | ~10,500 | ~16,800 | 1000 Hz | During OS playback (1kHz tone) |
| 4.5 - 8.0s | ~80 | ~270 | noise | After playback (noise floor) |

The ~130x increase in RMS during playback (90 → 10,500) and clear 1kHz frequency detection
confirm that the full audio path is working correctly.

**DRAS log confirmation:**

```
dras: [GraphKey("hw:0,2")] Start the GraphThread loop
dras: Enable UCM config for Speaker
dras: Set GraphThread stream state to Active
dras: [GraphKey("hw:0,2")] GraphThread block_size: 480
AudioTrack: stop(31): called with 144000 frames delivered
dras: [GraphKey("hw:0,2")] No streams left, shutting down graph thread.
dras: Set GraphThread stream state to Inactive
dras: Disable UCM config for Speaker
```

This shows DRAS opened `hw:0,2` (Speaker PCM), activated the Speaker UCM config (which
sets `Speaker Switch = 1` via mixer_paths.xml), streamed 144,000 frames (3 seconds at
48kHz), then shut down.

---

## 7. Known Issues and Notes

### 7.1 Topology ABI Version Mismatch Warning

The kernel logs a warning about ABI version mismatch:

```
Topology: ABI 3:29:1 Kernel ABI 3:23:1
```

The topology was compiled with SOF source that reports ABI 3.29.1 (the version in the
`/home/gaggery/nvme/sof` repository), while the kernel expects ABI 3.23.1 (the version
built into the 6.18.2 kernel). Despite this mismatch, the topology loads and functions
correctly because the SOF topology loader has backward compatibility for minor version
differences within the same major version (3.x). This is a warning, not an error.

### 7.2 No PCH DMIC

The NHLT tables report 0 DMICs (`DMICs detected in NHLT tables: 0`). The current topology
does not include PCH DMIC pipelines. The `InternalMic` path in `audio_config.xml` points to
PCM 10 (PCH DMIC), which does not exist in the current topology. The RT712's SmartMic
function (accessible via PCM 4) captures from the SoundWire DMIC, which is separate from
PCH DMIC.

If PCH DMIC is needed in the future, the topology would need to be rebuilt with
`NUM_DMICS=2` (or 4) and appropriate DMIC ID offsets.

### 7.3 SmartAmp Aggregation: Design vs Reality

The upstream kernel ACPI match table defines the RT712 VB with an aggregated amp endpoint
(`jack_amp_g1_dmic_endpoints[1]` with `group_id=1`), and the stock SOF topology uses
`NUM_SDW_AMP_LINKS=2` to support this aggregation.

```
  Design intent (upstream kernel match table):

  ┌──────────────────────────────────────────────────────────┐
  │  A board with BOTH RT712 and RT1320 acting as            │
  │  speaker amplifiers in a stereo aggregation group:       │
  │                                                          │
  │  RT712 amp endpoint ──► Left Speaker  (group 1, pos 0)   │
  │  RT1320 endpoint    ──► Right Speaker (group 1, pos 1)   │
  │                                                          │
  │  Topology: NUM_SDW_AMP_LINKS=2 (2 ALH copiers)          │
  │  Kernel: 1 aggregated BE link with 2 CPU DAI slots       │
  └──────────────────────────────────────────────────────────┘

  Actual hardware on PTL GCS board:

  ┌──────────────────────────────────────────────────────────┐
  │  Only RT1320 acts as speaker amplifier.                   │
  │  RT712 has NO SmartAmp function (only UAJ + SmartMic).    │
  │                                                          │
  │  RT1320 endpoint ──► Speaker  (sole amp)                  │
  │                                                          │
  │  Topology: NUM_SDW_AMP_LINKS=1 (1 ALH copier)           │
  │  Kernel: 1 non-aggregated BE link with 1 CPU DAI slot    │
  └──────────────────────────────────────────────────────────┘
```

This discrepancy exists because the kernel match table entry is designed to support multiple
board configurations that share the same RT712+RT1320 combination. On boards where the RT712
actually participates in SmartAmp aggregation (driving a second speaker), the original
`NUM_SDW_AMP_LINKS=2` topology would be correct. On this PTL GCS board, only the RT1320
drives the speaker, so `NUM_SDW_AMP_LINKS=1` is required.

### 7.4 Keyboard Fix (i8042.dumbkbd)

This board also required `i8042.dumbkbd` in the kernel command line to fix keyboard input.
The EC does not properly ACK the PS/2 enable command (0xF4), causing the `atkbd` driver to
repeatedly fail with "Failed to enable keyboard on isa0060/serio0". The `dumbkbd` parameter
skips the enable/disable commands.

---

## 8. Kernel Command Line (Audio-Related)

```
snd_intel_dspcfg.dsp_driver=3
snd_sof_intel_hda_common.sof_use_tplg_nhlt=1
snd_sof_pci.ipc_type=1
snd_intel_sdw_acpi.sdw_link_mask=0xF
i8042.dumbkbd
```

| Parameter | Value | Purpose |
|-----------|-------|---------|
| snd_intel_dspcfg.dsp_driver | 3 | Force SOF DSP driver (not legacy HDA or SST). Value 3 = SOF. |
| snd_sof_intel_hda_common.sof_use_tplg_nhlt | 1 | Use NHLT data embedded in topology for DMIC configuration |
| snd_sof_pci.ipc_type | 1 | Use IPC4 protocol for DSP communication (Panther Lake uses IPC4) |
| snd_intel_sdw_acpi.sdw_link_mask | 0xF | Enable SoundWire links 0-3 (binary 1111). Without this, SoundWire may not enumerate. |
| i8042.dumbkbd | (flag) | Skip PS/2 enable/disable commands for keyboard EC compatibility |

---

## 9. Complete File Reference

### 9.1 Modified Files in alos_grub

| File | Change |
|------|--------|
| `new_files/firmware/intel/sof-ipc4-tplg/sof-ptl-rt712-l3-rt1320-l2.tplg` | Rebuilt with `NUM_SDW_AMP_LINKS=1` (65,722 bytes) |
| `new_files/device/google/desktop/fatcat/alsa/sof-soundwire_rt712_rt1320/mixer_paths.xml` | Added OT23 L/R switch defaults for RT1320 and RT712 |

### 9.2 Kernel Source Files (Reference)

| File | Relevance |
|------|-----------|
| `sound/soc/intel/common/soc-acpi-intel-ptl-match.c` | ACPI match table with `ptl_sdw_rt712_vb_l3_rt1320_l2` entry and endpoint definitions |
| `sound/soc/sof/topology.c` | SOF topology loader; `sof_connect_dai_widget()` where "can't find BE" error originates (line 1119) |
| `sound/soc/sdw_utils/soc_sdw_utils.c` | `sof_sdw` machine driver; creates BE DAI links from ACPI device descriptions |
| `sound/soc/sdca/sdca_device.c` | RT712 VB detection (`sdca_device_quirk_rt712_vb()`) checks for SmartMic function |

### 9.3 SOF Build Files (Reference)

| File | Relevance |
|------|-----------|
| `tools/topology/topology2/production/tplg-targets-ace3.cmake` | Build target definitions for PTL topologies (line 127-129 for this board) |
| `tools/topology/topology2/cavs-sdw.conf` | Base SoundWire topology template (uses `NUM_SDW_AMP_LINKS` to control ALH copier count) |
| `tools/topology/topology2/get_abi.sh` | ABI version extraction script |

### 9.4 Device Deployment Paths

| Local Source | Device Path |
|-------------|-------------|
| `sof-ptl-rt712-l3-rt1320-l2.tplg` | `/vendor/firmware/intel/sof-ipc4-tplg/sof-ptl-rt712-l3-rt1320-l2.tplg` |
| `mixer_paths.xml` | `/vendor/etc/audio/sof-soundwire_rt712_rt1320/mixer_paths.xml` |
| `audio_config.xml` | `/vendor/etc/audio/sof-soundwire_rt712_rt1320/audio_config.xml` |

### 9.5 Git Commit

```
commit 8830676
audio: Fix RT712+RT1320 speaker output and topology mismatch

Rebuild sof-ptl-rt712-l3-rt1320-l2.tplg with NUM_SDW_AMP_LINKS=1
instead of 2. The original topology expected SmartAmp aggregation
with 2 ALH copiers (RT712 amp + RT1320 amp), but the kernel only
creates 1 SmartAmp BE link for RT1320, causing topology load failure:
"error: can't find BE for DAI alh-copier.Playback-SmartAmp.1"

Also enable RT1320 and RT712 output terminal switches (OT23 L/R) by
default in mixer_paths.xml. Without these, the RT1320 amplifier
output stage is muted and no sound reaches the physical speaker.
```

--- End of Document ---
