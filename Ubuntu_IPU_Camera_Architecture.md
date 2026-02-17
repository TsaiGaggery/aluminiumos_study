# Ubuntu/Linux IPU Camera Software Architecture

## Intel IPU6/IPU7 Upstream Kernel Driver, libcamera, PipeWire Integration

### Intel Panther Lake / Lunar Lake / Arrow Lake Platforms
### Linux Mainline Kernel (v6.10+) IPU6 Upstream Driver
### libcamera + PipeWire Userspace Camera Stack
### February 2026

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [IPU Hardware Generations](#2-ipu-hardware-generations)
3. [Kernel Driver Architecture](#3-kernel-driver-architecture)
4. [ipu-bridge: ACPI Sensor Enumeration](#4-ipu-bridge-acpi-sensor-enumeration)
5. [ISYS (Input System) Architecture](#5-isys-input-system-architecture)
6. [V4L2 and Media Controller Framework](#6-v4l2-and-media-controller-framework)
7. [Firmware Loading and Management](#7-firmware-loading-and-management)
8. [libcamera Framework Architecture](#8-libcamera-framework-architecture)
9. [PipeWire Camera Integration](#9-pipewire-camera-integration)
10. [GStreamer Camera Pipeline](#10-gstreamer-camera-pipeline)
11. [Complete Data Flow: Application to Hardware](#11-complete-data-flow-application-to-hardware)
12. [Out-of-Tree vs Upstream Driver Comparison](#12-out-of-tree-vs-upstream-driver-comparison)
13. [IPU7 and Future Upstream Support](#13-ipu7-and-future-upstream-support)
14. [File Reference](#14-file-reference)

---

## 1. Architecture Overview

The Ubuntu/Linux camera stack for Intel IPU platforms follows a layered architecture that
is fundamentally different from Android's camera HAL approach. Linux uses the V4L2 (Video
for Linux 2) kernel API with the Media Controller framework, and relies on **libcamera** as
the primary userspace camera framework to replace the traditional direct V4L2 access model.

```
  Ubuntu/Linux IPU Camera Stack (Complete Architecture)

  ┌────────────────────────────────────────────────────────────────────────┐
  │  Applications                                                         │
  │                                                                       │
  │  ┌───────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────────┐     │
  │  │ GNOME     │  │ Firefox  │  │ Google Meet  │  │ OBS Studio   │     │
  │  │ Cheese    │  │ WebRTC   │  │ (browser)    │  │ (streaming)  │     │
  │  └─────┬─────┘  └────┬─────┘  └──────┬───────┘  └──────┬───────┘     │
  │        │              │               │                  │            │
  │        ▼              ▼               ▼                  ▼            │
  │  ┌─────────────────────────────────────────────────────────────┐      │
  │  │                    PipeWire (Camera Provider)               │      │
  │  │  ┌────────────────────────────────────────────────────┐    │      │
  │  │  │  libcamera-spa-plugin (PipeWire SPA module)        │    │      │
  │  │  │  Bridges libcamera cameras to PipeWire graph       │    │      │
  │  │  └──────────────┬─────────────────────────────────────┘    │      │
  │  └─────────────────┼──────────────────────────────────────────┘      │
  │                    │                                                  │
  │                    ▼                                                  │
  │  ┌─────────────────────────────────────────────────────────────┐      │
  │  │                    libcamera                                │      │
  │  │  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐    │      │
  │  │  │ Pipeline     │  │ IPA (Image  │  │ Camera Manager │    │      │
  │  │  │ Handler      │  │ Processing  │  │ (discovery &   │    │      │
  │  │  │ (IPU3/Simple)│  │ Algorithms) │  │  enumeration)  │    │      │
  │  │  └──────┬───────┘  └──────┬──────┘  └────────────────┘    │      │
  │  │         │                 │                                 │      │
  │  │         ▼                 ▼                                 │      │
  │  │  ┌──────────────────────────────────────────────────┐      │      │
  │  │  │  V4L2/Media Controller Abstraction Layer         │      │      │
  │  │  │  V4L2VideoDevice, V4L2Subdevice, MediaDevice     │      │      │
  │  │  └──────────────┬───────────────────────────────────┘      │      │
  │  └─────────────────┼──────────────────────────────────────────┘      │
  │                    │                                                  │
  ═══════════════════  │  ══════════════════════════════════ User/Kernel ═
  │                    │                                                  │
  │                    ▼                                                  │
  │  ┌─────────────────────────────────────────────────────────────┐      │
  │  │  Linux Kernel                                               │      │
  │  │                                                             │      │
  │  │  ┌─────────────────────────────────┐                       │      │
  │  │  │  V4L2 Core + Media Controller   │                       │      │
  │  │  │  /dev/video*, /dev/media*       │                       │      │
  │  │  │  /dev/v4l-subdev*               │                       │      │
  │  │  └────────────┬────────────────────┘                       │      │
  │  │               │                                             │      │
  │  │  ┌────────────┴────────────────────────────────────┐       │      │
  │  │  │  Intel IPU6 Driver (intel-ipu6 + intel-ipu6-isys)│       │      │
  │  │  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │       │      │
  │  │  │  │ PCI Probe│  │ ISYS     │  │ CSI-2        │  │       │      │
  │  │  │  │ + FW Load│  │ (Capture)│  │ Receivers    │  │       │      │
  │  │  │  └──────────┘  └──────────┘  └──────┬───────┘  │       │      │
  │  │  │  ┌──────────┐  ┌──────────┐         │          │       │      │
  │  │  │  │ MMU      │  │ DMA      │         │          │       │      │
  │  │  │  │ (IOMMU)  │  │ Engine   │         │          │       │      │
  │  │  │  └──────────┘  └──────────┘         │          │       │      │
  │  │  └─────────────────────────────────────┼──────────┘       │      │
  │  │               │                         │                   │      │
  │  │  ┌────────────┴──────┐   ┌─────────────┴───────────┐      │      │
  │  │  │ ipu-bridge        │   │ Sensor I2C Drivers      │      │      │
  │  │  │ (ACPI enumeration)│   │ (ov2740, ov8856, etc.)  │      │      │
  │  │  └───────────────────┘   └─────────────────────────┘      │      │
  │  └─────────────────────────────────────────────────────────────┘      │
  │                    │                                                  │
  ═══════════════════  │  ═════════════════════════════════════ Hardware ═
  │                    ▼                                                  │
  │  ┌─────────────────────────────────────────────────────────────┐      │
  │  │  Intel IPU6 Hardware                                        │      │
  │  │  ┌───────────────────────────────────────────────────┐     │      │
  │  │  │  ISYS (Input System)                               │     │      │
  │  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │     │      │
  │  │  │  │CSI-2 Rx 0│ │CSI-2 Rx 1│ │CSI-2 Rx N│          │     │      │
  │  │  │  └─────┬────┘ └─────┬────┘ └─────┬────┘          │     │      │
  │  │  │        │             │             │               │     │      │
  │  │  │        └─────────────┴─────────────┘               │     │      │
  │  │  │                      │                             │     │      │
  │  │  │                 ┌────┴────┐                        │     │      │
  │  │  │                 │ DMA to  │                        │     │      │
  │  │  │                 │ System  │                        │     │      │
  │  │  │                 │ Memory  │                        │     │      │
  │  │  │                 └─────────┘                        │     │      │
  │  │  └───────────────────────────────────────────────────┘     │      │
  │  │                                                             │      │
  │  │  ┌─────────────────┐  ┌─────────────────┐                 │      │
  │  │  │ Camera Sensor 0 │  │ Camera Sensor 1 │   (via CSI-2)   │      │
  │  │  │ (e.g., OV2740)  │  │ (e.g., OV8856)  │                 │      │
  │  │  └─────────────────┘  └─────────────────┘                 │      │
  │  └─────────────────────────────────────────────────────────────┘      │
  └────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Key Differences from Android Camera Stack

| Aspect | Android | Ubuntu/Linux |
|--------|---------|-------------|
| Kernel API | V4L2 (same) | V4L2 (same) |
| Userspace Framework | Camera HAL3 (camera3_device_t) | libcamera |
| Image Processing | IPU PSYS (hardware ISP) or vendor HAL | libcamera IPA (software) or simple passthrough |
| Session Management | Camera Service (Java/C++) | PipeWire (camera provider) |
| Application API | android.hardware.camera2 | PipeWire, GStreamer, or direct libcamera |
| Firmware | Intel proprietary (loaded by HAL) | Upstream firmware blobs (loaded by kernel) |
| PSYS (Processing) | Full hardware ISP pipeline | **Not used upstream** -- ISYS only |
| Sensor Enumeration | ACPI + Android HAL config | ACPI + ipu-bridge + V4L2 async |

**Critical distinction**: The upstream Linux IPU6 driver **only supports ISYS** (Input System),
which provides raw Bayer sensor data capture via CSI-2. It does **not** support PSYS (Processing
System), which is Intel's hardware ISP. This means all image processing (demosaicing, auto
exposure, auto white balance, noise reduction) must be done in software by libcamera's IPA
modules or by the application itself.

---

## 2. IPU Hardware Generations

### 2.1 IPU Version Matrix

The upstream Linux kernel IPU6 driver supports multiple IPU generations through a single
codebase with version-specific configurations:

```
  IPU6 Hardware Versions Supported by Upstream Driver:

  ┌──────────────┬──────────┬──────────────┬──────────────┬────────────┐
  │ IPU Version  │ hw_ver   │ Intel SoC    │ CSI-2 Ports  │ Firmware   │
  ├──────────────┼──────────┼──────────────┼──────────────┼────────────┤
  │ IPU6         │ VER_6    │ Tiger Lake   │ 8 ports      │ ipu6_fw    │
  │ IPU6SE       │ VER_6SE  │ Jasper Lake  │ 4 ports      │ ipu6se_fw  │
  │ IPU6EP       │ VER_6EP  │ Alder Lake / │ 4 ports      │ ipu6ep_fw  │
  │              │          │ Raptor Lake  │              │            │
  │ IPU6EP-MTL   │ VER_6EP  │ Meteor Lake  │ 6 ports      │ ipu6epmtl  │
  │              │ _MTL     │              │              │ _fw        │
  └──────────────┴──────────┴──────────────┴──────────────┴────────────┘

  IPU Versions NOT in upstream (as of kernel 6.18):

  ┌──────────────┬──────────────┬──────────────────────────────────────┐
  │ IPU Version  │ Intel SoC    │ Status                               │
  ├──────────────┼──────────────┼──────────────────────────────────────┤
  │ IPU7         │ Lunar Lake   │ Out-of-tree only (intel/ipu7-drivers)│
  │ IPU7 (PTL)   │ Panther Lake │ Out-of-tree only (Android IPU7 HAL) │
  │ IPU75        │ Arrow Lake   │ Not yet available in any public repo │
  └──────────────┴──────────────┴──────────────────────────────────────┘
```

### 2.2 PCI Device IDs

The driver identifies IPU hardware generation via PCI device ID in the probe function.

**Source**: `drivers/media/pci/intel/ipu6/ipu6.c` (line 532)

```c
switch (id->device) {
    case PCI_DEVICE_ID_INTEL_IPU6:         // Tiger Lake
        isp->hw_ver = IPU6_VER_6;
        isp->cpd_fw_name = "intel/ipu/ipu6_fw.bin";
        break;
    case PCI_DEVICE_ID_INTEL_IPU6SE:       // Jasper Lake
        isp->hw_ver = IPU6_VER_6SE;
        isp->cpd_fw_name = "intel/ipu/ipu6se_fw.bin";
        break;
    case PCI_DEVICE_ID_INTEL_IPU6EP_ADLP:  // Alder Lake P
    case PCI_DEVICE_ID_INTEL_IPU6EP_RPLP:  // Raptor Lake P
        isp->hw_ver = IPU6_VER_6EP;
        isp->cpd_fw_name = "intel/ipu/ipu6ep_fw.bin";
        break;
    case PCI_DEVICE_ID_INTEL_IPU6EP_ADLN:  // Alder Lake N
        isp->hw_ver = IPU6_VER_6EP;
        isp->cpd_fw_name = "intel/ipu/ipu6epadln_fw.bin";
        break;
    case PCI_DEVICE_ID_INTEL_IPU6EP_MTL:   // Meteor Lake
        isp->hw_ver = IPU6_VER_6EP_MTL;
        isp->cpd_fw_name = "intel/ipu/ipu6epmtl_fw.bin";
        break;
}
```

### 2.3 Version-Specific CSI-2 Port Counts

Each IPU generation has a different number of CSI-2 receiver ports:

**Source**: `drivers/media/pci/intel/ipu6/ipu6.c` (lines 286-289)

```c
#define IPU6_ISYS_CSI2_NPORTS           4   // IPU6 (TGL) base
#define IPU6SE_ISYS_CSI2_NPORTS         4   // IPU6SE (JSL)
#define IPU6_TGL_ISYS_CSI2_NPORTS       8   // IPU6 TGL (overrides base)
#define IPU6EP_MTL_ISYS_CSI2_NPORTS     6   // IPU6EP MTL
```

---

## 3. Kernel Driver Architecture

### 3.1 Module Structure

The upstream IPU6 driver compiles into two kernel modules:

```
  Kernel Module Build Structure:

  ┌──────────────────────────────────────────────────────────────────┐
  │  Module: intel-ipu6.ko (PCI device driver + core infrastructure)│
  │                                                                  │
  │  ipu6.o          ── PCI probe, firmware load, version detection  │
  │  ipu6-bus.o      ── Auxiliary bus for ISYS/PSYS sub-devices      │
  │  ipu6-dma.o      ── DMA memory allocation with IOMMU support    │
  │  ipu6-mmu.o      ── IPU internal MMU (L1/L2 TLB management)     │
  │  ipu6-buttress.o ── Power management, clock control, FW auth     │
  │  ipu6-cpd.o      ── CPD (Code Partition Directory) FW parsing    │
  │  ipu6-fw-com.o   ── Firmware communication (message queues)      │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  Module: intel-ipu6-isys.ko (V4L2 capture driver)               │
  │                                                                  │
  │  ipu6-isys.o         ── ISYS main: probe, V4L2/MC registration  │
  │  ipu6-isys-csi2.o    ── CSI-2 receiver V4L2 subdevice           │
  │  ipu6-fw-isys.o      ── ISYS firmware command interface          │
  │  ipu6-isys-video.o   ── V4L2 video device (capture nodes)       │
  │  ipu6-isys-queue.o   ── videobuf2 queue management              │
  │  ipu6-isys-subdev.o  ── V4L2 subdevice helpers                  │
  │  ipu6-isys-mcd-phy.o ── MCD PHY (Tiger Lake CSI-2 PHY)          │
  │  ipu6-isys-jsl-phy.o ── JSL PHY (Jasper Lake CSI-2 PHY)         │
  │  ipu6-isys-dwc-phy.o ── DWC PHY (Meteor Lake CSI-2 PHY)         │
  └──────────────────────────────────────────────────────────────────┘
```

**Source**: `drivers/media/pci/intel/ipu6/Makefile`

### 3.2 Kconfig Dependencies

**Source**: `drivers/media/pci/intel/ipu6/Kconfig`

```
config VIDEO_INTEL_IPU6
    tristate "Intel IPU6 driver"
    depends on ACPI || COMPILE_TEST
    depends on VIDEO_DEV
    depends on X86 && HAS_DMA
    depends on IPU_BRIDGE || !IPU_BRIDGE
    select AUXILIARY_BUS
    select IOMMU_IOVA
    select VIDEO_V4L2_SUBDEV_API
    select MEDIA_CONTROLLER
    select VIDEOBUF2_DMA_SG
    select V4L2_FWNODE
```

Key selections:
- **AUXILIARY_BUS**: Used for ISYS sub-device registration
- **MEDIA_CONTROLLER**: Required for media graph topology
- **VIDEOBUF2_DMA_SG**: DMA scatter-gather buffer management
- **V4L2_FWNODE**: Firmware node parsing for sensor discovery

### 3.3 PCI Probe Flow

The main entry point is `ipu6_pci_probe()` in `ipu6.c`. This function orchestrates the
entire hardware initialization sequence:

```
  PCI Probe Flow (ipu6_pci_probe):

  ┌─────────────────────────────────────────────────────────────┐
  │  1. PCI device enable and BAR mapping                       │
  │     pcim_enable_device(pdev)                                │
  │     isp->base = pcim_iomap_region(pdev, BAR0)              │
  │                                                             │
  │  2. Identify hardware version from PCI device ID            │
  │     isp->hw_ver = IPU6_VER_6EP_MTL  (etc.)                 │
  │     isp->cpd_fw_name = "intel/ipu/ipu6epmtl_fw.bin"        │
  │                                                             │
  │  3. Initialize platform data based on hw version            │
  │     ipu6_internal_pdata_init(isp)                           │
  │     Sets: CSI-2 port count, SRAM sizes, IRQ masks           │
  │                                                             │
  │  4. Set DMA mask (39-bit addressing)                        │
  │     dma_set_mask_and_coherent(dev, DMA_BIT_MASK(39))        │
  │                                                             │
  │  5. Initialize Buttress (power management)                  │
  │     ipu6_buttress_init(isp)                                 │
  │                                                             │
  │  6. Load and validate firmware                              │
  │     request_firmware(&isp->cpd_fw, fw_name, dev)            │
  │     ipu6_cpd_validate_cpd_file(isp, fw->data, fw->size)    │
  │                                                             │
  │  7. Initialize ISYS sub-device                              │
  │     isp->isys = ipu6_isys_init(pdev, ...)                   │
  │       ├── ipu_bridge_init()  ← ACPI sensor enumeration      │
  │       ├── ipu6_bus_initialize_device()                      │
  │       ├── ipu6_mmu_init() for ISYS MMU                      │
  │       └── ipu6_bus_add_device()                             │
  │                                                             │
  │  8. Initialize PSYS sub-device (minimal in upstream)        │
  │     isp->psys = ipu6_psys_init(pdev, ...)                   │
  │                                                             │
  │  9. Authenticate firmware via Buttress                      │
  │     ipu6_buttress_authenticate(isp)                         │
  │                                                             │
  │ 10. Configure VC arbitration mechanism                      │
  │     ipu6_configure_vc_mechanism(isp)                        │
  │                                                             │
  │ 11. Enable runtime PM                                       │
  │     pm_runtime_allow(dev)                                   │
  └─────────────────────────────────────────────────────────────┘
```

### 3.4 Key Data Structures

**`struct ipu6_device`** -- Top-level device structure:

```c
// drivers/media/pci/intel/ipu6/ipu6.h (line 73)
struct ipu6_device {
    struct pci_dev *pdev;
    struct list_head devices;
    struct ipu6_bus_device *isys;      // ISYS sub-device
    struct ipu6_bus_device *psys;      // PSYS sub-device
    struct ipu6_buttress buttress;     // Power management
    const struct firmware *cpd_fw;     // Firmware blob
    const char *cpd_fw_name;           // Firmware filename
    void __iomem *base;                // PCI BAR0 mapping
    bool secure_mode;                  // Secure boot mode
    u8 hw_ver;                         // IPU6_VER_* enum
};
```

**`struct ipu6_isys`** -- ISYS subsystem structure:

```c
// drivers/media/pci/intel/ipu6/ipu6-isys.h (line 127)
struct ipu6_isys {
    struct media_device media_dev;      // Media controller device
    struct v4l2_device v4l2_dev;        // V4L2 device
    struct ipu6_bus_device *adev;       // Auxiliary bus device
    int power;                          // Power state
    struct ipu6_isys_stream streams[IPU6_ISYS_MAX_STREAMS]; // FW streams
    void *fwcom;                        // FW communication context
    struct ipu6_isys_csi2 *csi2;        // CSI-2 receiver array
    struct v4l2_async_notifier notifier; // Async sensor notification
};
```

**`struct ipu6_isys_csi2`** -- CSI-2 receiver:

```c
// drivers/media/pci/intel/ipu6/ipu6-isys-csi2.h (line 37)
struct ipu6_isys_csi2 {
    struct ipu6_isys_subdev asd;        // V4L2 subdevice wrapper
    struct ipu6_isys *isys;             // Parent ISYS
    struct ipu6_isys_video av[NR_OF_CSI2_SRC_PADS]; // Video capture nodes
    void __iomem *base;                 // Register base
    unsigned int nlanes;                // Number of CSI-2 data lanes
    unsigned int port;                  // Port index (0..N)
};
```

---

## 4. ipu-bridge: ACPI Sensor Enumeration

### 4.1 The ACPI-to-V4L2 Gap Problem

Windows ACPI tables describe camera sensors using proprietary methods (SSDB buffers) that
are incompatible with the Linux device tree / fwnode approach used by V4L2 sensor drivers.
The **ipu-bridge** module bridges this gap by:

1. Scanning ACPI for known sensor Hardware IDs (HIDs)
2. Reading the SSDB (Sensor Specific Data Block) ACPI buffer
3. Creating **software fwnode** properties that V4L2 sensor drivers expect
4. Building a graph connection between sensor and IPU CSI-2 port

```
  ipu-bridge ACPI-to-V4L2 Translation:

  ACPI DSDT (Windows-oriented)              V4L2 fwnode (Linux-oriented)
  ┌────────────────────────────┐            ┌────────────────────────────┐
  │ Device (INT3474)           │            │ Software Node: "INT3474-2" │
  │   _HID: "INT3474"         │    ipu-    │   clock-frequency: 19200000│
  │   _STA: 0xF (enabled)     │   bridge   │   rotation: 0              │
  │   SSDB buffer:             │ ────────► │   orientation: BACK        │
  │     link = 2               │            │   port@0:                  │
  │     lanes = 2              │            │     endpoint@0:            │
  │     mclkspeed = 19200000   │            │       bus-type: CSI2-DPHY  │
  │     degree = 0             │            │       data-lanes: <1 2>    │
  │     vcmtype = 3 (dw9719)   │            │       remote-endpoint:     │
  │                            │            │         → IPU port@2 ep@0  │
  │   _PLD: panel=BACK         │            │   lens-focus: → dw9719-2  │
  └────────────────────────────┘            └────────────────────────────┘
```

### 4.2 Supported Sensor Table

The ipu-bridge maintains a table of supported sensors with their link frequencies:

**Source**: `drivers/media/pci/intel/ipu-bridge.c` (line 50)

```c
static const struct ipu_sensor_config ipu_supported_sensors[] = {
    IPU_SENSOR_CONFIG("HIMX11B1", 1, 384000000),     // Himax HM11B1
    IPU_SENSOR_CONFIG("HIMX2170", 1, 384000000),     // Himax HM2170
    IPU_SENSOR_CONFIG("HIMX2172", 1, 384000000),     // Himax HM2172
    IPU_SENSOR_CONFIG("INT0310",  1, 55692000),       // GalaxyCore GC0310
    IPU_SENSOR_CONFIG("INT33BE",  1, 419200000),      // OV5693
    IPU_SENSOR_CONFIG("INT3474",  1, 180000000),      // OV2740
    IPU_SENSOR_CONFIG("INT3479",  1, 422400000),      // OV5670
    IPU_SENSOR_CONFIG("INT347A",  1, 360000000),      // OV8865
    IPU_SENSOR_CONFIG("OVTI01A0", 1, 400000000),      // OV01A10
    IPU_SENSOR_CONFIG("OVTI02C1", 1, 400000000),      // OV02C10
    IPU_SENSOR_CONFIG("OVTI08F4", 3, 400000000,       // OV08x40 (3 freqs)
                                     749000000, 800000000),
    IPU_SENSOR_CONFIG("OVTI13B1", 1, 560000000),      // OV13B10
    IPU_SENSOR_CONFIG("OVTI8856", 3, 180000000,       // OV8856 (3 freqs)
                                     360000000, 720000000),
    // ... more sensors
};
```

### 4.3 SSDB Buffer Parsing

The SSDB (Sensor Specific Data Block) is a proprietary Intel ACPI buffer that encodes
sensor configuration. The bridge reads it via `ipu_bridge_parse_ssdb()`:

**Source**: `drivers/media/pci/intel/ipu-bridge.c` (line 294)

```c
int ipu_bridge_parse_ssdb(struct acpi_device *adev, struct ipu_sensor *sensor)
{
    struct ipu_sensor_ssdb ssdb = {};
    ret = ipu_bridge_read_acpi_buffer(adev, "SSDB", &ssdb, sizeof(ssdb));

    sensor->link = ssdb.link;           // CSI-2 port number
    sensor->lanes = ssdb.lanes;         // Number of data lanes (1-4)
    sensor->mclkspeed = ssdb.mclkspeed; // Master clock frequency
    sensor->rotation = ipu_bridge_parse_rotation(adev, &ssdb);
    sensor->orientation = ipu_bridge_parse_orientation(adev);

    if (ssdb.vcmtype)
        sensor->vcm_type = ipu_vcm_types[ssdb.vcmtype - 1];
}
```

### 4.4 Software Node Graph Construction

For each detected sensor, ipu-bridge creates a software node graph that mimics a device
tree structure:

```
  Software Node Graph Created by ipu-bridge:

  ┌─────────────────────────────────────────────────────────────┐
  │  SWNODE_SENSOR_HID: "INT3474-2"                             │
  │    Properties:                                               │
  │      clock-frequency = 19200000                              │
  │      rotation = 0                                            │
  │      orientation = V4L2_FWNODE_ORIENTATION_BACK               │
  │      lens-focus = → SWNODE_VCM                               │
  │                                                              │
  │    └── SWNODE_SENSOR_PORT: "port@0"                          │
  │          └── SWNODE_SENSOR_ENDPOINT: "endpoint@0"            │
  │                Properties:                                    │
  │                  bus-type = V4L2_FWNODE_BUS_TYPE_CSI2_DPHY    │
  │                  data-lanes = <1, 2>                          │
  │                  remote-endpoint = → SWNODE_IPU_ENDPOINT      │
  │                  link-frequencies = <180000000>               │
  │                                                              │
  │  SWNODE_IPU_PORT: "port@2" (under IPU HID node)             │
  │    └── SWNODE_IPU_ENDPOINT: "endpoint@0"                    │
  │          Properties:                                          │
  │            data-lanes = <1, 2>                                │
  │            remote-endpoint = → SWNODE_SENSOR_ENDPOINT         │
  │                                                              │
  │  SWNODE_VCM: "dw9719-2" (VCM lens focus motor)              │
  └─────────────────────────────────────────────────────────────┘
```

### 4.5 Bridge Initialization in PCI Probe

The bridge is initialized during ISYS device creation, **before** V4L2 registration:

**Source**: `drivers/media/pci/intel/ipu6/ipu6.c` (line 378)

```c
static struct ipu6_bus_device *
ipu6_isys_init(struct pci_dev *pdev, ...)
{
    // First: enumerate sensors via ACPI bridge
    ret = ipu_bridge_init(dev, ipu_bridge_parse_ssdb);
    if (ret)
        return ERR_PTR(ret);

    // Then: set up ISYS platform data and bus device
    pdata = kzalloc(sizeof(*pdata), GFP_KERNEL);
    isys_adev = ipu6_bus_initialize_device(pdev, parent, pdata, ...);
    isys_adev->mmu = ipu6_mmu_init(dev, base, ISYS_MMID, ...);
    ipu6_bus_add_device(isys_adev);
    return isys_adev;
}
```

### 4.6 IVSC (Intel Visual Sensing Controller) Support

Some platforms (11th/12th Gen+) include an Intel Visual Sensing Controller (IVSC) between
the camera sensor and the IPU. The ipu-bridge handles this by inserting additional software
nodes for the IVSC CSI-2 passthrough:

```
  Graph with IVSC:

  Sensor ──► IVSC (MEI CSI device) ──► IPU CSI-2 Port

  Without IVSC:

  Sensor ──────────────────────────► IPU CSI-2 Port
```

IVSC ACPI device IDs: `INTC1059`, `INTC1095`, `INTC100A`, `INTC10CF`

---

## 5. ISYS (Input System) Architecture

### 5.1 ISYS Overview

The ISYS (Input System) is the camera capture subsystem of the IPU. In the upstream driver,
ISYS is the **only** functional subsystem -- it handles raw frame capture from CSI-2
sensors and DMA transfer to system memory.

```
  ISYS Internal Architecture:

  ┌──────────────────────────────────────────────────────────────────┐
  │  IPU6 ISYS                                                       │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  CSI-2 Receivers (one per port)                            │  │
  │  │                                                            │  │
  │  │  ┌────────┐ ┌────────┐ ┌────────┐      ┌────────┐       │  │
  │  │  │ Port 0 │ │ Port 1 │ │ Port 2 │ ...  │ Port N │       │  │
  │  │  │ PHY    │ │ PHY    │ │ PHY    │      │ PHY    │       │  │
  │  │  │ lanes: │ │ lanes: │ │ lanes: │      │ lanes: │       │  │
  │  │  │ 1-4    │ │ 1-4    │ │ 1-4    │      │ 1-4    │       │  │
  │  │  └───┬────┘ └───┬────┘ └───┬────┘      └───┬────┘       │  │
  │  │      │           │           │               │            │  │
  │  │      ▼           ▼           ▼               ▼            │  │
  │  │  ┌────────────────────────────────────────────────────┐   │  │
  │  │  │  Stream Multiplexer (Virtual Channels 0-15)        │   │  │
  │  │  │  Each CSI-2 port can carry up to 16 VCs            │   │  │
  │  │  │  Each VC can have different data type (RAW, META)   │   │  │
  │  │  └──────────────────────────┬─────────────────────────┘   │  │
  │  └─────────────────────────────┼──────────────────────────────┘  │
  │                                │                                  │
  │  ┌─────────────────────────────┼──────────────────────────────┐  │
  │  │  Pixel Buffer (SRAM)        │                               │  │
  │  │  IPU6: 256KB (200KB usable) │                               │  │
  │  │  IPU6SE: 128KB (96KB usable)│                               │  │
  │  │                             │                               │  │
  │  │  Stores incoming pixel data before DMA to system memory     │  │
  │  │  Threshold-based watermark for DMA triggering               │  │
  │  └─────────────────────────────┼──────────────────────────────┘  │
  │                                │                                  │
  │  ┌─────────────────────────────┼──────────────────────────────┐  │
  │  │  DMA Engine                 │                               │  │
  │  │                             ▼                               │  │
  │  │  Transfers frames from SRAM to system memory                │  │
  │  │  Uses IPU MMU for virtual→physical address translation      │  │
  │  │  Scatter-gather DMA (videobuf2-dma-sg)                      │  │
  │  └────────────────────────────────────────────────────────────┘  │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  IPU MMU (Internal IOMMU)                                  │  │
  │  │  L1 Cache: 16 streams × up to 16 blocks each              │  │
  │  │  L2 Cache: 16 streams × 2 blocks each                     │  │
  │  │  Address space: 32-bit (31-bit in non-secure mode)         │  │
  │  │  ZLW pre-fetch mechanism for TLB warmup                    │  │
  │  └────────────────────────────────────────────────────────────┘  │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  ISYS Firmware (SP Processor)                              │  │
  │  │  Runs on dedicated scalar processor inside IPU             │  │
  │  │  Handles stream open/start/stop/close commands             │  │
  │  │  Reports SOF/EOF events and buffer completion              │  │
  │  │  Communication via shared memory message queues            │  │
  │  └────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.2 CSI-2 Receiver Implementation

Each CSI-2 port is represented as a V4L2 subdevice with 1 sink pad and up to 8 source pads
(one per virtual channel):

**Source**: `drivers/media/pci/intel/ipu6/ipu6-isys-csi2.h`

```c
#define NR_OF_CSI2_VC           16   // Max virtual channels per port
#define NR_OF_CSI2_SINK_PADS    1    // One sink (from sensor)
#define NR_OF_CSI2_SRC_PADS     8    // Up to 8 source pads
#define NR_OF_CSI2_PADS         (1 + 8)  // 9 total pads
```

```
  CSI-2 Subdevice Pad Layout:

  ┌──────────────────────────────────────────────────┐
  │  ipu6-isys-csi2 (V4L2 subdevice)                │
  │                                                  │
  │  Sink Pad 0 ◄──── Camera Sensor                 │
  │                                                  │
  │  Source Pad 1 ───► /dev/video0  (VC0, image)     │
  │  Source Pad 2 ───► /dev/video1  (VC0, metadata)  │
  │  Source Pad 3 ───► /dev/video2  (VC1, image)     │
  │  Source Pad 4 ───► /dev/video3  (VC1, metadata)  │
  │  Source Pad 5 ───► /dev/video4  (VC2, image)     │
  │  Source Pad 6 ───► /dev/video5  (VC2, metadata)  │
  │  Source Pad 7 ───► /dev/video6  (VC3, image)     │
  │  Source Pad 8 ───► /dev/video7  (VC3, metadata)  │
  └──────────────────────────────────────────────────┘
```

### 5.3 CSI-2 PHY Variants

Three different PHY implementations exist for different Intel platforms:

| PHY Type | File | Platforms | Description |
|----------|------|-----------|-------------|
| MCD PHY | `ipu6-isys-mcd-phy.c` | Tiger Lake (IPU6) | Multi-Channel DPHY |
| JSL PHY | `ipu6-isys-jsl-phy.c` | Jasper Lake (IPU6SE) | Simplified DPHY |
| DWC PHY | `ipu6-isys-dwc-phy.c` | Meteor Lake (IPU6EP-MTL) | Synopsys DesignWare DPHY |

The PHY is selected at runtime via function pointer in `struct ipu6_isys`:

```c
// ipu6-isys.h (line 153)
struct ipu6_isys {
    int (*phy_set_power)(struct ipu6_isys *isys,
                         struct ipu6_isys_csi2_config *cfg,
                         const struct ipu6_isys_csi2_timing *timing,
                         bool on);
};
```

### 5.4 Firmware Communication

The ISYS firmware runs on a dedicated scalar processor (SPC) inside the IPU. The kernel
communicates with it through shared memory message queues:

**Source**: `drivers/media/pci/intel/ipu6/ipu6-fw-isys.h`

```
  Firmware Message Queue Layout:

  ┌─────────────────────────────────────────────────────────┐
  │  Send Queues (Kernel → FW)                              │
  │                                                         │
  │  Queue 0: Proxy Send      (high-priority, direct)       │
  │  Queue 1: Device Send     (device-level commands)        │
  │  Queue 2..17: Message Send (per-stream commands)         │
  │                                                         │
  │  Commands: STREAM_OPEN, STREAM_START, STREAM_CAPTURE,    │
  │           STREAM_STOP, STREAM_FLUSH, STREAM_CLOSE        │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │  Receive Queues (FW → Kernel)                           │
  │                                                         │
  │  Queue 0: Proxy Recv      (proxy responses)              │
  │  Queue 1: Message Recv    (all stream responses)         │
  │                                                         │
  │  Responses: STREAM_OPEN_DONE, STREAM_START_ACK,          │
  │            PIN_DATA_READY, FRAME_SOF, FRAME_EOF,         │
  │            STREAM_STOP_ACK, STREAM_CLOSE_ACK             │
  └─────────────────────────────────────────────────────────┘
```

### 5.5 Stream Management

The ISYS supports up to 16 concurrent firmware streams. Each stream corresponds to a CSI-2
virtual channel and can produce multiple output pins (image + metadata):

**Source**: `drivers/media/pci/intel/ipu6/ipu6-isys-video.h` (line 44)

```c
struct ipu6_isys_stream {
    struct mutex mutex;
    struct media_entity *source_entity;
    atomic_t sequence;                  // Frame sequence counter
    int stream_source;                  // CSI-2 port number
    int stream_handle;                  // FW stream handle (0..15)
    unsigned int nr_output_pins;        // Number of capture outputs
    struct ipu6_isys_subdev *asd;       // Source subdevice
    int nr_queues;                      // Number of capture queues
    int streaming;                      // Streaming state
    struct list_head queues;            // List of video queues
    struct ipu6_isys_queue *output_pins_queue[11]; // Pin→queue mapping
    u8 vc;                              // Virtual channel ID
};
```

---

## 6. V4L2 and Media Controller Framework

### 6.1 Media Graph Topology

The IPU6 driver creates a media controller device (`/dev/mediaX`) with a graph that
represents the physical connections between sensors, CSI-2 receivers, and capture video
device nodes:

```
  Media Controller Graph (example: 2 sensors, 4 CSI-2 ports):

  ┌────────────────┐       ┌──────────────────────────────────────────┐
  │ OV2740 sensor  │       │  Intel IPU6 ISYS CSI-2 0                │
  │ (V4L2 subdev)  │       │  (V4L2 subdevice)                       │
  │                │       │                                          │
  │ pad 0 (src) ───┼──────►│ pad 0 (sink)                            │
  └────────────────┘       │                                          │
                           │ pad 1 (src) ───► /dev/video0  (capture)  │
                           │ pad 2 (src) ───► /dev/video1  (meta)     │
                           └──────────────────────────────────────────┘

  ┌────────────────┐       ┌──────────────────────────────────────────┐
  │ OV8856 sensor  │       │  Intel IPU6 ISYS CSI-2 2                │
  │ (V4L2 subdev)  │       │  (V4L2 subdevice)                       │
  │                │       │                                          │
  │ pad 0 (src) ───┼──────►│ pad 0 (sink)                            │
  └────────────────┘       │                                          │
                           │ pad 1 (src) ───► /dev/video4  (capture)  │
                           │ pad 2 (src) ───► /dev/video5  (meta)     │
                           └──────────────────────────────────────────┘

  Device nodes created:
    /dev/media0       ── Media controller (graph navigation)
    /dev/video0..N    ── V4L2 capture devices (VIDIOC_STREAMON)
    /dev/v4l-subdev0  ── Sensor subdevice (format/control setting)
    /dev/v4l-subdev1  ── CSI-2 subdevice (lane config)
```

### 6.2 V4L2 Video Device Implementation

Each CSI-2 source pad creates a V4L2 video capture device:

**Source**: `drivers/media/pci/intel/ipu6/ipu6-isys-video.h` (line 82)

```c
struct ipu6_isys_video {
    struct ipu6_isys_queue aq;          // videobuf2 queue
    struct mutex mutex;
    struct media_pad pad;               // Media entity pad
    struct video_device vdev;           // V4L2 video device
    struct v4l2_pix_format pix_fmt;     // Current pixel format
    struct v4l2_meta_format meta_fmt;   // Current meta format
    struct ipu6_isys *isys;
    struct ipu6_isys_csi2 *csi2;        // Parent CSI-2 receiver
    struct ipu6_isys_stream *stream;    // Active stream
    u32 source_stream;                  // Source stream ID
    u8 vc;                              // Virtual channel
    u8 dt;                              // Data type
};
```

### 6.3 Supported Pixel Formats

The ISYS supports raw Bayer formats (the sensor's native output) plus some packed formats:

```
  Supported Capture Formats:

  RAW Bayer (standard V4L2 fourcc):
    V4L2_PIX_FMT_SBGGR8    ── 8-bit Bayer BGGR
    V4L2_PIX_FMT_SGBRG8    ── 8-bit Bayer GBRG
    V4L2_PIX_FMT_SGRBG8    ── 8-bit Bayer GRBG
    V4L2_PIX_FMT_SRGGB8    ── 8-bit Bayer RGGB
    V4L2_PIX_FMT_SBGGR10   ── 10-bit Bayer BGGR
    V4L2_PIX_FMT_SGBRG10   ── 10-bit Bayer GBRG
    V4L2_PIX_FMT_SGRBG10   ── 10-bit Bayer GRBG
    V4L2_PIX_FMT_SRGGB10   ── 10-bit Bayer RGGB
    V4L2_PIX_FMT_SBGGR12   ── 12-bit Bayer BGGR
    ...and packed variants

  Metadata:
    V4L2_META_FMT_GENERIC_8    ── Generic 8-bit metadata
    V4L2_META_FMT_GENERIC_CSI2_10 ── CSI-2 10-bit metadata
```

### 6.4 Buffer Management (videobuf2)

The driver uses the videobuf2-dma-sg (scatter-gather) framework for buffer management:

```
  Buffer Flow:

  1. Application allocates buffers via VIDIOC_REQBUFS
     └── vb2_queue_init() with VB2_DMA_SG memory type

  2. Application queues buffer via VIDIOC_QBUF
     └── Buffer added to driver's active list

  3. VIDIOC_STREAMON triggers stream start:
     └── FW STREAM_OPEN → STREAM_START → STREAM_CAPTURE commands
     └── DMA address from sg_table sent to firmware

  4. Frame captured:
     └── FW sends PIN_DATA_READY response
     └── IRQ handler dequeues buffer
     └── Buffer returned to application via VIDIOC_DQBUF

  5. VIDIOC_STREAMOFF:
     └── FW STREAM_STOP → STREAM_CLOSE
     └── All queued buffers returned with error flag
```

### 6.5 V4L2 Async Sensor Registration

Sensor drivers register asynchronously with the ISYS V4L2 device using the V4L2 async
framework. The ipu-bridge creates fwnode connections, and when sensor drivers probe, they
register as V4L2 subdevices:

**Source**: `drivers/media/pci/intel/ipu6/ipu6-isys.c` (line 106)

```c
static int isys_complete_ext_device_registration(
    struct ipu6_isys *isys,
    struct v4l2_subdev *sd,              // Sensor subdevice
    struct ipu6_isys_csi2_config *csi2)  // CSI-2 port config
{
    // Find the sensor's source pad
    for (i = 0; i < sd->entity.num_pads; i++)
        if (sd->entity.pads[i].flags & MEDIA_PAD_FL_SOURCE)
            break;

    // Create immutable link: sensor src → CSI-2 sink
    ret = media_create_pad_link(
        &sd->entity, i,                              // Sensor source
        &isys->csi2[csi2->port].asd.sd.entity, 0,    // CSI-2 sink
        MEDIA_LNK_FL_ENABLED | MEDIA_LNK_FL_IMMUTABLE);

    isys->csi2[csi2->port].nlanes = csi2->nlanes;
}
```

---

## 7. Firmware Loading and Management

### 7.1 Firmware File Naming

Each IPU version requires its own firmware binary, stored in `/lib/firmware/intel/ipu/`:

| IPU Version | Firmware File | Approximate Size |
|-------------|--------------|-----------------|
| IPU6 (TGL) | `intel/ipu/ipu6_fw.bin` | ~3-5 MB |
| IPU6SE (JSL) | `intel/ipu/ipu6se_fw.bin` | ~2-3 MB |
| IPU6EP (ADL/RPL) | `intel/ipu/ipu6ep_fw.bin` | ~3-5 MB |
| IPU6EP (ADL-N) | `intel/ipu/ipu6epadln_fw.bin` | ~3-5 MB |
| IPU6EP-MTL | `intel/ipu/ipu6epmtl_fw.bin` | ~5-7 MB |

**Source**: `drivers/media/pci/intel/ipu6/ipu6.h` (lines 20-24)

### 7.2 CPD (Code Partition Directory) Format

The firmware uses Intel's CPD format, which is a container with multiple code/data entries:

**Source**: `drivers/media/pci/intel/ipu6/ipu6-cpd.c`

```
  CPD Firmware Structure:

  ┌─────────────────────────────────────────┐
  │  CPD Header                             │
  │    magic: "$CPD"                        │
  │    num_entries: N                        │
  │    header_version: 2                    │
  │                                         │
  │  Entry 0: ISYS Server FW               │
  │    ├── Metadata (authentication hash)   │
  │    └── Blob (SP processor code)         │
  │                                         │
  │  Entry 1: PSYS Server FW               │
  │    ├── Metadata (authentication hash)   │
  │    └── Blob (SP processor code)         │
  │                                         │
  │  Entry 2: Module Entries                │
  │    └── Various processing modules       │
  └─────────────────────────────────────────┘
```

### 7.3 Firmware Authentication

The Buttress hardware unit authenticates the firmware before it can execute:

**Source**: `drivers/media/pci/intel/ipu6/ipu6-buttress.c`

```
  Firmware Authentication Flow:

  1. Map firmware to IPU MMU address space
     ipu6_buttress_map_fw_image(isp->psys, cpd_fw, &sgt)

  2. Create package directory from CPD
     ipu6_cpd_create_pkg_dir(isp->psys, cpd_fw->data)

  3. Configure SPC (Scalar Processor Core)
     ipu6_configure_spc(isp, hw_variant, pkg_dir_idx, ...)
       - Writes firmware start PC to SPC registers
       - Writes package directory address to DMEM

  4. Authenticate via Buttress CSE IPC
     ipu6_buttress_authenticate(isp)
       - Sends authentication request to CSE (Converged Security Engine)
       - CSE verifies firmware signature
       - On success: SPC can start executing
```

### 7.4 Firmware Distribution on Ubuntu

On Ubuntu, IPU firmware files are distributed via the `linux-firmware` package:

```bash
# Ubuntu package providing IPU firmware
apt list --installed | grep linux-firmware
# linux-firmware/jammy-updates ... [installed]

# Firmware file locations
ls /lib/firmware/intel/ipu/
# ipu6_fw.bin  ipu6ep_fw.bin  ipu6epmtl_fw.bin  ipu6se_fw.bin ...
```

---

## 8. libcamera Framework Architecture

### 8.1 Overview

libcamera is the modern Linux camera framework that provides:
- Camera discovery and enumeration
- Pipeline configuration and stream management
- Image processing algorithms (IPA)
- Abstraction over complex V4L2/MC hardware topologies
- A stable C++ API for applications

```
  libcamera Internal Architecture:

  ┌──────────────────────────────────────────────────────────────────┐
  │  libcamera                                                       │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  CameraManager                                             │  │
  │  │  - Enumerates media devices via DeviceEnumerator           │  │
  │  │  - Matches devices to PipelineHandlers                     │  │
  │  │  - Creates Camera objects for each detected camera          │  │
  │  └──────────────────────────┬─────────────────────────────────┘  │
  │                             │                                    │
  │  ┌──────────────────────────┴─────────────────────────────────┐  │
  │  │  PipelineHandler (abstract base class)                     │  │
  │  │  - match(): Detect if this handler supports the hardware   │  │
  │  │  - generateConfiguration(): Create stream configurations   │  │
  │  │  - configure(): Apply configuration to hardware            │  │
  │  │  - start()/stop(): Begin/end streaming                     │  │
  │  │  - queueRequest(): Submit capture request                  │  │
  │  │                                                            │  │
  │  │  Implementations:                                          │  │
  │  │  ┌─────────────────┐  ┌──────────────────┐               │  │
  │  │  │ IPU3 Pipeline   │  │ Simple Pipeline  │               │  │
  │  │  │ Handler         │  │ Handler          │               │  │
  │  │  │ (Intel IPU3/6)  │  │ (Generic V4L2)   │               │  │
  │  │  └────────┬────────┘  └────────┬─────────┘               │  │
  │  └───────────┼────────────────────┼───────────────────────────┘  │
  │              │                    │                               │
  │  ┌───────────┼────────────────────┼───────────────────────────┐  │
  │  │  V4L2 Abstraction Layer        │                           │  │
  │  │                                │                           │  │
  │  │  MediaDevice      ── /dev/mediaX (media controller)       │  │
  │  │  V4L2VideoDevice  ── /dev/videoX (capture/output)         │  │
  │  │  V4L2Subdevice    ── /dev/v4l-subdevX (sensor/CSI-2)     │  │
  │  │  CameraSensor     ── Sensor abstraction (format, ctrls)   │  │
  │  └────────────────────────────────────────────────────────────┘  │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  IPA (Image Processing Algorithms)                         │  │
  │  │  - Runs in isolated process (or in-process for soft-ISP)  │  │
  │  │  - Provides: AE, AWB, AGC, lens shading correction        │  │
  │  │  - Interface defined per pipeline handler                  │  │
  │  └────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.2 Pipeline Handler: IPU3

The IPU3 pipeline handler in libcamera is designed for Intel IPU3 (Skylake/Kaby Lake) but
its architecture is instructive for understanding how IPU6 would be handled:

**Source**: `src/libcamera/pipeline/ipu3/ipu3.cpp`

```cpp
class PipelineHandlerIPU3 : public PipelineHandler {
public:
    // Match by looking for "ipu3-imgu" and "ipu3-cio2" media devices
    bool match(DeviceEnumerator *enumerator) override;

    // Generate config for up to 3 streams: output, viewfinder, raw
    std::unique_ptr<CameraConfiguration>
    generateConfiguration(Camera *camera, Span<const StreamRole> roles) override;

    // Configure CIO2 (CSI-2 receiver) and ImgU (image processing unit)
    int configure(Camera *camera, CameraConfiguration *config) override;

    // Queue capture request to CIO2 and ImgU
    int queueRequestDevice(Camera *camera, Request *request) override;
};
```

The IPU3 handler manages two media devices:
- **CIO2** (CSI-2 Input/Output): Raw frame capture from sensor
- **ImgU** (Image Processing Unit): Hardware ISP for demosaicing, scaling, etc.

```
  IPU3 Pipeline Handler Data Flow:

  Camera Sensor
       │
       ▼
  ┌─────────────┐
  │ CIO2 Device │  Captures raw Bayer frames
  │ (CSI-2 Rx)  │  V4L2 video device: /dev/videoX
  └──────┬──────┘
         │ raw Bayer frame
         ▼
  ┌─────────────┐
  │ ImgU Device │  Hardware ISP processing
  │ (Image Proc)│  - Demosaicing
  │             │  - Color correction
  │             │  - Noise reduction
  │             │  - Scaling
  └──────┬──────┘
         │ processed frame (NV12/YUV)
         ▼
  Application Buffer
```

### 8.3 Pipeline Handler: Simple (for IPU6 ISYS-only)

For IPU6 upstream (ISYS only, no PSYS), the **Simple pipeline handler** is used. It
works with any V4L2 device that provides a straightforward capture path:

**Source**: `src/libcamera/pipeline/simple/simple.cpp`

The Simple handler:
1. Discovers the media graph by traversing from sensor to video capture node
2. Configures formats along the entire pipeline (sensor → CSI-2 → video)
3. Captures raw frames directly (no hardware ISP)
4. Optionally applies software ISP via the SoftwareISP IPA module

```
  IPU6 with Simple Pipeline Handler:

  Camera Sensor (e.g., OV2740)
       │
       ▼
  ┌──────────────────────┐
  │ IPU6 ISYS CSI-2 Port │  V4L2 subdevice: /dev/v4l-subdevX
  │ (CSI-2 receiver)     │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ IPU6 ISYS Video      │  V4L2 video device: /dev/videoX
  │ (DMA capture)        │  Outputs: raw Bayer (SBGGR10, SGRBG10, etc.)
  └──────────┬───────────┘
             │ raw Bayer frame
             ▼
  ┌──────────────────────┐
  │ libcamera SoftwareISP│  Software processing (CPU-based)
  │ - Debayer            │  - Bayer demosaicing
  │ - AWB                │  - Auto white balance
  │ - Color correction   │  - Basic color processing
  └──────────┬───────────┘
             │ processed frame (NV12/RGB)
             ▼
  Application Buffer
```

### 8.4 Camera Discovery Process

```
  Camera Discovery Flow:

  1. CameraManager::start()
     └── DeviceEnumerator::enumerate()
         └── Scan /dev/media* devices
         └── Read media graph topology via MEDIA_IOC_G_TOPOLOGY

  2. For each PipelineHandler implementation:
     └── handler->match(enumerator)
         └── Search for matching media device by driver name
         └── If matched: create Camera objects

  3. IPU3 match example:
     └── Look for media device with driver "ipu3-imgu"
     └── Look for media device with driver "ipu3-cio2"
     └── For each CSI-2 port with a connected sensor:
         └── Create Camera(name, streams)

  4. Simple match example:
     └── Look for any media device with sensor entities
     └── Trace path: sensor → processing → video capture node
     └── For each complete path: Create Camera(name, streams)
```

### 8.5 Request/Buffer Model

libcamera uses a request-based model for frame capture:

```
  Request Lifecycle:

  Application                    libcamera                      Kernel
  ─────────────────────────────────────────────────────────────────────

  1. camera->createRequest()  → Request object created
                                 (contains control settings)

  2. request->addBuffer(      → Buffer associated with
      stream, framebuffer)       stream in request

  3. camera->queueRequest(    → PipelineHandler::
      request)                   queueRequestDevice()
                                    │
                                    ▼
                              V4L2VideoDevice::       → VIDIOC_QBUF
                                queueBuffer()            (kernel)
                                    │
                                    │  ← IRQ: frame complete
                                    ▼
                              V4L2VideoDevice::       ← VIDIOC_DQBUF
                                dequeueBuffer()
                                    │
                                    ▼
  4. requestComplete signal ← Request completed
     (application callback)    with filled buffers
```

---

## 9. PipeWire Camera Integration

### 9.1 PipeWire Camera Architecture

PipeWire is the modern multimedia server for Linux that handles both audio and video
(camera) streams. It replaces both PulseAudio (for audio) and provides camera access
that was previously done through direct V4L2:

```
  PipeWire Camera Stack:

  ┌────────────────────────────────────────────────────────────┐
  │  Application (e.g., GNOME Cheese, Firefox WebRTC)          │
  │  Uses: PipeWire API or GStreamer pipewiresrc element       │
  └──────────────┬─────────────────────────────────────────────┘
                 │ PipeWire protocol (Unix socket)
                 ▼
  ┌────────────────────────────────────────────────────────────┐
  │  PipeWire Session Manager (WirePlumber)                    │
  │  - Manages permissions (XDG Camera Portal)                 │
  │  - Routes camera streams to requesting applications        │
  │  - Handles camera sharing between applications             │
  └──────────────┬─────────────────────────────────────────────┘
                 │
                 ▼
  ┌────────────────────────────────────────────────────────────┐
  │  PipeWire Daemon (pipewire)                                │
  │                                                            │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │  SPA Plugin: libspa-libcamera                        │  │
  │  │  (spa/plugins/libcamera/)                            │  │
  │  │                                                      │  │
  │  │  - Wraps libcamera as PipeWire SPA (Simple Plugin API)│  │
  │  │  - Enumerates cameras via CameraManager               │  │
  │  │  - Creates PipeWire nodes for each camera             │  │
  │  │  - Handles buffer sharing via dmabuf                  │  │
  │  └──────────────────────────┬───────────────────────────┘  │
  └─────────────────────────────┼──────────────────────────────┘
                                │
                                ▼
  ┌────────────────────────────────────────────────────────────┐
  │  libcamera                                                  │
  │  (camera discovery, pipeline management, IPA)               │
  └────────────────────────────────────────────────────────────┘
```

### 9.2 SPA libcamera Plugin

The PipeWire SPA (Simple Plugin API) libcamera plugin creates a PipeWire node for each
camera discovered by libcamera:

```
  SPA libcamera Plugin Operation:

  1. Plugin loaded by PipeWire daemon at startup
     └── spa_libcamera_manager_open()
         └── CameraManager::start()

  2. For each camera discovered:
     └── Create PipeWire SPA node
     └── Register with PipeWire graph
     └── Expose as: "Camera: <name>" (e.g., "Camera: OV2740")

  3. When application requests camera stream:
     └── Camera::configure() with requested format
     └── Camera::start()
     └── Camera::queueRequest() for each frame

  4. Frame delivery:
     └── libcamera completes request
     └── SPA plugin pushes buffer to PipeWire graph
     └── PipeWire delivers buffer to application via dmabuf or memfd
```

### 9.3 XDG Camera Portal

On desktop Linux (GNOME, KDE), camera access is mediated by the XDG Desktop Portal:

```
  Camera Permission Flow:

  Application (Flatpak/Snap)
       │
       ├── Request: "I need camera access"
       │
       ▼
  XDG Desktop Portal
       │
       ├── Show permission dialog to user
       │   "Allow Firefox to access your camera?"
       │
       ├── If granted: Create PipeWire camera node access
       │
       ▼
  WirePlumber (Session Manager)
       │
       ├── Route camera PipeWire node to application
       │
       ▼
  Application receives camera frames via PipeWire
```

---

## 10. GStreamer Camera Pipeline

### 10.1 libcamerasrc Element

GStreamer provides a `libcamerasrc` element that uses libcamera for camera access:

```bash
# GStreamer pipeline: Camera preview with libcamera
gst-launch-1.0 libcamerasrc ! videoconvert ! autovideosink

# GStreamer pipeline: Camera capture to file
gst-launch-1.0 libcamerasrc ! video/x-raw,width=1920,height=1080 ! \
    videoconvert ! x264enc ! mp4mux ! filesink location=output.mp4

# GStreamer pipeline: Camera via PipeWire
gst-launch-1.0 pipewiresrc ! videoconvert ! autovideosink
```

### 10.2 V4L2 Direct Access (Legacy)

Applications can also access the camera directly via V4L2, bypassing libcamera. This works
for simple use cases but requires the application to handle raw Bayer data:

```bash
# Direct V4L2 capture (raw Bayer, no ISP)
v4l2-ctl --device=/dev/video0 --stream-mmap --stream-count=10 \
    --stream-to=raw_frames.bin

# FFmpeg with V4L2
ffmpeg -f v4l2 -input_format rawvideo -video_size 1920x1080 \
    -i /dev/video0 -vframes 100 output.mp4
```

---

## 11. Complete Data Flow: Application to Hardware

### 11.1 Full Stack Data Flow (PipeWire Path)

```
  Complete Data Flow: GNOME Cheese → Camera Sensor

  ┌─────────────────────────────────────────────────────────────────┐
  │  GNOME Cheese Application                                       │
  │  GStreamer pipeline: pipewiresrc ! videoconvert ! gtksink        │
  │                                                                 │
  │  Step 1: Request camera from PipeWire                           │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  PipeWire IPC (Unix domain socket)
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  PipeWire Daemon                                                 │
  │                                                                 │
  │  Step 2: SPA libcamera plugin activates camera                  │
  │  - Camera::configure(StreamRole::VideoRecording)                 │
  │  - Camera::start()                                               │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  libcamera C++ API
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  libcamera                                                       │
  │                                                                 │
  │  Step 3: Simple Pipeline Handler configures hardware             │
  │  - V4L2Subdevice::setFormat() on sensor subdev                  │
  │  - V4L2Subdevice::setFormat() on CSI-2 subdev                  │
  │  - V4L2VideoDevice::setFormat() on capture device               │
  │  - V4L2VideoDevice::allocateBuffers()                           │
  │                                                                 │
  │  Step 4: Start streaming                                        │
  │  - V4L2VideoDevice::streamOn() → VIDIOC_STREAMON                │
  │  - Queue initial buffers → VIDIOC_QBUF                          │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  V4L2 ioctls (system calls)
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Linux Kernel                                                    │
  │                                                                 │
  │  Step 5: ISYS driver processes STREAMON                         │
  │  - ipu6_isys_video_set_streaming(av, 1, bl)                     │
  │  - Send FW commands: STREAM_OPEN, STREAM_START, STREAM_CAPTURE   │
  │  - Configure DMA addresses from videobuf2 sg_table              │
  │                                                                 │
  │  Step 6: ISYS firmware configures hardware                      │
  │  - Program CSI-2 receiver: lane count, data type                │
  │  - Configure pixel buffer thresholds                            │
  │  - Start DMA engine                                             │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  Hardware bus (CSI-2 DPHY)
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Camera Sensor (e.g., OV2740)                                    │
  │                                                                 │
  │  Step 7: Sensor outputs pixel data                              │
  │  - I2C register writes configure: resolution, exposure, gain    │
  │  - Sensor starts streaming Bayer data over CSI-2 lanes          │
  │  - Each frame: SOF → line data → EOF                            │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  CSI-2 DPHY (differential pairs)
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  IPU6 ISYS Hardware                                              │
  │                                                                 │
  │  Step 8: CSI-2 receiver captures frame                          │
  │  - PHY deserializes DPHY signals                                │
  │  - Protocol decoder extracts pixel data from CSI-2 packets      │
  │  - Data flows to pixel buffer (SRAM)                            │
  │                                                                 │
  │  Step 9: DMA transfers frame to system memory                   │
  │  - IPU MMU translates virtual→physical addresses                │
  │  - DMA engine writes pixels to application's buffer             │
  │  - FW sends PIN_DATA_READY interrupt                            │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  IRQ + DMA completion
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Back to Kernel:                                                 │
  │  Step 10: ISR completes buffer                                  │
  │  - isys_isr_one() processes FW response                         │
  │  - vb2_buffer_done() marks buffer as ready                      │
  │  - Wake up poll/select on video device fd                       │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  VIDIOC_DQBUF / poll() wakeup
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Back to libcamera:                                              │
  │  Step 11: Frame processing                                      │
  │  - V4L2VideoDevice::dequeueBuffer()                             │
  │  - SoftwareISP (if enabled): Debayer raw → RGB/NV12             │
  │  - IPA algorithms: AWB, AE, AGC adjustments                    │
  │  - Request::complete() callback                                 │
  └──────────────┬──────────────────────────────────────────────────┘
                 │  PipeWire buffer delivery
                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Back to Application:                                            │
  │  Step 12: Display frame                                         │
  │  - PipeWire delivers processed buffer via dmabuf/memfd          │
  │  - GStreamer videoconvert: format conversion if needed          │
  │  - gtksink: render to GTK widget                                │
  └─────────────────────────────────────────────────────────────────┘
```

### 11.2 Timing and Performance

```
  Approximate Timing for Single Frame:

  Sensor exposure + readout:     ~33ms (at 30fps)
  CSI-2 transfer to SRAM:       ~1-5ms (depends on resolution/lanes)
  DMA to system memory:          ~0.5-2ms
  Software ISP (debayer+AWB):   ~5-15ms (CPU-dependent)
  PipeWire buffer delivery:      ~0.1ms
  ──────────────────────────────────────
  Total pipeline latency:        ~40-55ms (1-2 frame latency)
```

---

## 12. Out-of-Tree vs Upstream Driver Comparison

### 12.1 Feature Comparison

The Intel out-of-tree driver (`github.com/intel/ipu6-drivers`) includes significant
additional functionality compared to the upstream kernel driver:

```
  Feature Comparison:

  ┌──────────────────────────────┬────────────┬─────────────────┐
  │ Feature                      │ Upstream   │ Out-of-Tree     │
  │                              │ (mainline) │ (intel/ipu6-    │
  │                              │            │  drivers)        │
  ├──────────────────────────────┼────────────┼─────────────────┤
  │ ISYS (raw capture)           │ ✓          │ ✓               │
  │ PSYS (hardware ISP)          │ ✗          │ ✓               │
  │ CSI-2 BE (backend entities)  │ ✗          │ ✓               │
  │ TPG (test pattern generator) │ ✗          │ ✓               │
  │ Multiple output formats      │ Raw only   │ Raw + YUV + RGB │
  │ Sensor drivers (bundled)     │ ✗ (separate│ ✓ (18+ sensors) │
  │                              │  modules)  │                 │
  │ GPC (perf counters)          │ ✗          │ ✓               │
  │ ipu-trace (debug tracing)    │ ✗          │ ✓               │
  │ DKMS support                 │ N/A        │ ✓               │
  │ Kernel version support       │ 6.10+      │ 5.15 - 6.10    │
  └──────────────────────────────┴────────────┴─────────────────┘
```

### 12.2 Architectural Differences

```
  Out-of-Tree Driver Architecture (more complex):

  ┌────────────────────────────────────────────────────────────┐
  │  ipu.ko (main PCI driver)                                   │
  │  ├── ipu-bus.c    (custom bus, not auxiliary_bus)           │
  │  ├── ipu-dma.c    (DMA with custom IOMMU)                 │
  │  ├── ipu-mmu.c    (same MMU management)                   │
  │  ├── ipu-buttress.c (same power management)               │
  │  └── ipu-cpd.c    (same firmware parsing)                  │
  │                                                             │
  │  ipu6-isys.ko (input system - similar to upstream)          │
  │  ├── ipu-isys.c                                            │
  │  ├── ipu-isys-csi2.c                                       │
  │  ├── ipu-isys-csi2-be.c     ← ADDITIONAL: CSI-2 Backend   │
  │  ├── ipu-isys-csi2-be-soc.c ← ADDITIONAL: SoC Backend     │
  │  ├── ipu-isys-tpg.c         ← ADDITIONAL: Test Pattern Gen│
  │  ├── ipu-isys-video.c                                      │
  │  └── ipu-isys-queue.c                                      │
  │                                                             │
  │  ipu6-psys.ko (processing system - NOT in upstream)         │
  │  ├── ipu6-psys.c            ← Hardware ISP control         │
  │  ├── ipu-fw-psys.c          ← PSYS firmware interface      │
  │  ├── ipu-resources.c        ← Resource allocation          │
  │  ├── ipu6-ppg.c             ← Parameter Programming Groups │
  │  └── ipu6-l-scheduler.c     ← Work scheduling             │
  └────────────────────────────────────────────────────────────┘
```

### 12.3 Userspace Stack Differences

```
  Out-of-Tree Userspace Stack:

  ┌────────────────────────────────────────────────────────────┐
  │  Application                                                │
  │       │                                                     │
  │       ▼                                                     │
  │  icamerasrc (GStreamer plugin)                              │
  │       │                                                     │
  │       ▼                                                     │
  │  ipu6-camera-hal (Intel camera HAL)                        │
  │  ├── 3A algorithms (proprietary)                           │
  │  ├── ISP parameter tuning files                            │
  │  └── PSYS pipeline configuration                           │
  │       │                                                     │
  │       ▼                                                     │
  │  ipu6-camera-bins (proprietary libraries)                  │
  │  ├── libipa.so (Image Processing Accelerator)              │
  │  ├── lib3aiq.so (3A Intelligence)                          │
  │  └── libia_imaging.so (Intel imaging algorithms)           │
  │       │                                                     │
  │       ▼                                                     │
  │  Kernel: ISYS + PSYS drivers                               │
  └────────────────────────────────────────────────────────────┘

  Upstream Userspace Stack:

  ┌────────────────────────────────────────────────────────────┐
  │  Application                                                │
  │       │                                                     │
  │       ▼                                                     │
  │  PipeWire (or GStreamer libcamerasrc)                       │
  │       │                                                     │
  │       ▼                                                     │
  │  libcamera                                                  │
  │  ├── Simple pipeline handler                               │
  │  ├── SoftwareISP (CPU-based debayer)                       │
  │  └── Software 3A (basic AWB/AE)                            │
  │       │                                                     │
  │       ▼                                                     │
  │  Kernel: ISYS driver only (no PSYS)                        │
  └────────────────────────────────────────────────────────────┘
```

---

## 13. IPU7 and Future Upstream Support

### 13.1 IPU7 Status

As of February 2026, the IPU7 (found in Lunar Lake and Panther Lake) does **not** have
upstream Linux kernel support. The IPU7 driver exists only in:

1. **Android kernel trees** (Intel internal, used for Panther Lake Android)
2. **Out-of-tree repositories** (intel/ipu7-drivers, not yet publicly available)

The IPU7 shares the same general architecture as IPU6 (ISYS + CSI-2 + firmware) but has
significant register and firmware interface changes.

### 13.2 Upstream Upstreaming Path

The typical path for IPU driver upstreaming:

```
  IPU Upstream Timeline:

  IPU3 (Skylake/Kaby Lake)
    └── Upstream since kernel 4.16 (2018)
    └── Full ISYS + PSYS support via intel-ipu3 driver
    └── libcamera IPU3 pipeline handler

  IPU6 (Tiger Lake → Meteor Lake)
    └── Upstream ISYS since kernel 6.10 (2024)
    └── PSYS NOT upstream (proprietary algorithms)
    └── libcamera Simple pipeline handler

  IPU7 (Lunar Lake / Panther Lake)
    └── Expected upstream: kernel 6.14+ (estimated)
    └── Same ISYS-only approach as IPU6
    └── Will likely share ipu-bridge infrastructure
```

### 13.3 Challenges for IPU7 Upstream

1. **No PSYS**: Like IPU6, the PSYS (hardware ISP) relies on proprietary Intel firmware
   interfaces and algorithm libraries that cannot be open-sourced
2. **New PHY**: IPU7 uses a different CSI-2 PHY implementation
3. **Firmware format**: Different firmware binary format from IPU6
4. **Register changes**: New register layouts for ISYS configuration
5. **libcamera handler**: Will need a new or updated pipeline handler

---

## 14. File Reference

### 14.1 Upstream Kernel IPU6 Driver Files

**Location**: `drivers/media/pci/intel/ipu6/`

| File | Lines | Purpose |
|------|-------|---------|
| `ipu6.c` | ~847 | PCI driver: probe, FW load, ISYS/PSYS init, PM |
| `ipu6.h` | ~342 | Main header: `struct ipu6_device`, hw variants, MMU config |
| `ipu6-bus.c` | ~200 | Auxiliary bus device management for ISYS/PSYS |
| `ipu6-bus.h` | ~80 | Bus device structures |
| `ipu6-buttress.c` | ~700 | Power management, FW authentication, IPC with CSE |
| `ipu6-buttress.h` | ~100 | Buttress control structures, IPC definitions |
| `ipu6-cpd.c` | ~250 | CPD firmware container parsing and validation |
| `ipu6-cpd.h` | ~80 | CPD structures (header, entry, metadata) |
| `ipu6-dma.c` | ~300 | DMA allocation with IPU MMU support |
| `ipu6-dma.h` | ~30 | DMA function declarations |
| `ipu6-fw-com.c` | ~350 | Firmware message queue communication |
| `ipu6-fw-com.h` | ~50 | FW communication structures |
| `ipu6-fw-isys.c` | ~500 | ISYS firmware command interface |
| `ipu6-fw-isys.h` | ~300 | ISYS FW structures: stream config, pin config, responses |
| `ipu6-isys.c` | ~1200 | ISYS main: probe, V4L2/MC registration, ISR, power |
| `ipu6-isys.h` | ~203 | ISYS structures: `struct ipu6_isys`, stream config |
| `ipu6-isys-csi2.c` | ~600 | CSI-2 V4L2 subdevice implementation |
| `ipu6-isys-csi2.h` | ~79 | CSI-2 structures: `struct ipu6_isys_csi2` |
| `ipu6-isys-video.c` | ~900 | V4L2 video device: ioctls, streaming, format negotiation |
| `ipu6-isys-video.h` | ~136 | Video structures: `struct ipu6_isys_video`, stream |
| `ipu6-isys-queue.c` | ~500 | videobuf2 queue management, buffer handling |
| `ipu6-isys-queue.h` | ~80 | Queue structures |
| `ipu6-isys-subdev.c` | ~400 | V4L2 subdevice helpers, pad format operations |
| `ipu6-isys-subdev.h` | ~60 | Subdevice structures |
| `ipu6-isys-mcd-phy.c` | ~450 | MCD PHY for Tiger Lake |
| `ipu6-isys-jsl-phy.c` | ~200 | JSL PHY for Jasper Lake |
| `ipu6-isys-dwc-phy.c` | ~500 | DWC PHY for Meteor Lake |
| `ipu6-mmu.c` | ~800 | IPU internal MMU: L1/L2 TLB, ZLW mechanism |
| `ipu6-mmu.h` | ~60 | MMU structures |
| `Kconfig` | ~19 | Build configuration |
| `Makefile` | ~24 | Build rules for both modules |

### 14.2 ipu-bridge Files

**Location**: `drivers/media/pci/intel/`

| File | Purpose |
|------|---------|
| `ipu-bridge.c` | ACPI sensor enumeration, software fwnode creation |

**Location**: `include/media/`

| File | Purpose |
|------|---------|
| `ipu-bridge.h` | Bridge structures: `struct ipu_sensor`, SSDB, swnode enums |
| `ipu6-pci-table.h` | PCI device ID table for IPU6 variants |

### 14.3 Platform Register Headers

| File | Purpose |
|------|---------|
| `ipu6-platform-buttress-regs.h` | Buttress register offsets and bit definitions |
| `ipu6-platform-isys-csi2-reg.h` | CSI-2 receiver register offsets |
| `ipu6-platform-regs.h` | General platform registers (ISYS, PSYS offsets) |

### 14.4 libcamera Files (Key Components)

**Location**: `src/libcamera/pipeline/ipu3/`

| File | Purpose |
|------|---------|
| `ipu3.cpp` | IPU3 pipeline handler: camera creation, configure, start |
| `cio2.cpp` | CIO2 (CSI-2 I/O) device abstraction |
| `cio2.h` | CIO2 class definition |
| `imgu.cpp` | ImgU (Image Processing Unit) device abstraction |
| `imgu.h` | ImgU class definition |
| `frames.cpp` | Frame tracking between CIO2 and ImgU |

**Location**: `src/libcamera/pipeline/simple/`

| File | Purpose |
|------|---------|
| `simple.cpp` | Simple pipeline handler (used for IPU6 ISYS-only) |

**Location**: `src/libcamera/` (core)

| File | Purpose |
|------|---------|
| `camera.cpp` | Camera class: configure, start, stop, queueRequest |
| `camera_manager.cpp` | Camera discovery and lifecycle management |
| `pipeline_handler.cpp` | Base pipeline handler class |
| `media_device.cpp` | Media controller device abstraction |
| `v4l2_videodevice.cpp` | V4L2 video device wrapper |
| `v4l2_subdevice.cpp` | V4L2 subdevice wrapper |

### 14.5 Out-of-Tree Driver Files (github.com/intel/ipu6-drivers)

**Location**: `drivers/media/pci/intel/`

| File | Purpose |
|------|---------|
| `ipu.c` | Main PCI driver (equivalent to upstream `ipu6.c`) |
| `ipu-isys-csi2-be.c` | CSI-2 Backend entity (NOT in upstream) |
| `ipu-isys-csi2-be-soc.c` | SoC backend entity (NOT in upstream) |
| `ipu-isys-tpg.c` | Test Pattern Generator (NOT in upstream) |
| `ipu-trace.c` | Debug tracing (NOT in upstream) |

**Location**: `drivers/media/pci/intel/ipu6/psys/`

| File | Purpose |
|------|---------|
| `ipu6-psys.c` | PSYS device driver (hardware ISP) |
| `ipu-fw-psys.c` | PSYS firmware interface |
| `ipu6-ppg.c` | Parameter Programming Groups |
| `ipu6-l-scheduler.c` | Work item scheduling |
| `ipu-resources.c` | PSYS resource allocation |

### 14.6 Firmware Files (Ubuntu: /lib/firmware/)

| File | IPU Version |
|------|------------|
| `intel/ipu/ipu6_fw.bin` | IPU6 (Tiger Lake) |
| `intel/ipu/ipu6se_fw.bin` | IPU6SE (Jasper Lake) |
| `intel/ipu/ipu6ep_fw.bin` | IPU6EP (Alder Lake / Raptor Lake) |
| `intel/ipu/ipu6epadln_fw.bin` | IPU6EP (Alder Lake N) |
| `intel/ipu/ipu6epmtl_fw.bin` | IPU6EP-MTL (Meteor Lake) |

### 14.7 Ubuntu Userspace Packages

| Package | Purpose |
|---------|---------|
| `linux-firmware` | IPU firmware binaries |
| `libcamera0.3` | libcamera shared library |
| `libcamera-tools` | `cam` command-line tool for testing |
| `pipewire` | PipeWire daemon |
| `libspa-0.2-libcamera` | PipeWire SPA libcamera plugin |
| `wireplumber` | PipeWire session manager |
| `gstreamer1.0-pipewire` | GStreamer PipeWire element |
| `libgstreamer-plugins-bad1.0` | Contains `libcamerasrc` GStreamer element |

### 14.8 Device Nodes Created

| Device Node | Type | Purpose |
|-------------|------|---------|
| `/dev/media0` | Media Controller | Graph topology navigation |
| `/dev/video0..N` | V4L2 Video | Frame capture (one per CSI-2 source pad) |
| `/dev/v4l-subdev0..N` | V4L2 Subdevice | Sensor and CSI-2 configuration |

--- End of Document ---
