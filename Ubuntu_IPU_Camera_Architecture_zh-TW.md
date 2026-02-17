# Ubuntu/Linux IPU 相機軟體架構

## Intel IPU6/IPU7 上游核心驅動程式、libcamera、PipeWire 整合

### Intel Panther Lake / Lunar Lake / Arrow Lake 平台
### Linux 主線核心 (v6.10+) IPU6 上游驅動程式
### libcamera + PipeWire 使用者空間相機堆疊
### 2026 年 2 月

---

## 目錄

1. [架構概覽](#1-架構概覽)
2. [IPU 硬體世代](#2-ipu-硬體世代)
3. [核心驅動程式架構](#3-核心驅動程式架構)
4. [ipu-bridge：ACPI 感測器列舉](#4-ipu-bridgeacpi-感測器列舉)
5. [ISYS（輸入系統）架構](#5-isys輸入系統架構)
6. [V4L2 與 Media Controller 框架](#6-v4l2-與-media-controller-框架)
7. [韌體載入與管理](#7-韌體載入與管理)
8. [libcamera 框架架構](#8-libcamera-框架架構)
9. [PipeWire 相機整合](#9-pipewire-相機整合)
10. [GStreamer 相機管線](#10-gstreamer-相機管線)
11. [完整資料流：從應用程式到硬體](#11-完整資料流從應用程式到硬體)
12. [樹外驅動與上游驅動比較](#12-樹外驅動與上游驅動比較)
13. [IPU7 與未來上游支援](#13-ipu7-與未來上游支援)
14. [檔案參考](#14-檔案參考)

---

## 1. 架構概覽

Ubuntu/Linux 中 Intel IPU 平台的相機堆疊採用分層架構，與 Android 的 Camera HAL 方式有根本性的不同。Linux 使用 V4L2（Video for Linux 2）核心 API 搭配 Media Controller 框架，並依賴 **libcamera** 作為主要的使用者空間相機框架，以取代傳統的直接 V4L2 存取模式。

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

### 1.1 與 Android 相機堆疊的主要差異

| 面向 | Android | Ubuntu/Linux |
|--------|---------|-------------|
| 核心 API | V4L2（相同） | V4L2（相同） |
| 使用者空間框架 | Camera HAL3 (camera3_device_t) | libcamera |
| 影像處理 | IPU PSYS（硬體 ISP）或供應商 HAL | libcamera IPA（軟體）或簡單直通 |
| 工作階段管理 | Camera Service (Java/C++) | PipeWire（相機提供者） |
| 應用程式 API | android.hardware.camera2 | PipeWire、GStreamer 或直接使用 libcamera |
| 韌體 | Intel 專有（由 HAL 載入） | 上游韌體 blob（由核心載入） |
| PSYS（處理系統） | 完整硬體 ISP 管線 | **上游不使用** -- 僅 ISYS |
| 感測器列舉 | ACPI + Android HAL 組態 | ACPI + ipu-bridge + V4L2 async |

**關鍵區別**：上游 Linux IPU6 驅動程式**僅支援 ISYS**（輸入系統），其透過 CSI-2 提供原始 Bayer 感測器資料擷取。它**不**支援 PSYS（處理系統），即 Intel 的硬體 ISP。這意味著所有影像處理（去馬賽克、自動曝光、自動白平衡、雜訊抑制）必須由 libcamera 的 IPA 模組或應用程式本身以軟體方式完成。

---

## 2. IPU 硬體世代

### 2.1 IPU 版本矩陣

上游 Linux 核心 IPU6 驅動程式透過單一程式碼庫搭配版本特定組態來支援多個 IPU 世代：

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

### 2.2 PCI 裝置 ID

驅動程式在 probe 函式中透過 PCI 裝置 ID 識別 IPU 硬體世代。

**來源**：`drivers/media/pci/intel/ipu6/ipu6.c`（第 532 行）

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

### 2.3 各版本特定的 CSI-2 埠數量

每個 IPU 世代具有不同數量的 CSI-2 接收埠：

**來源**：`drivers/media/pci/intel/ipu6/ipu6.c`（第 286-289 行）

```c
#define IPU6_ISYS_CSI2_NPORTS           4   // IPU6 (TGL) base
#define IPU6SE_ISYS_CSI2_NPORTS         4   // IPU6SE (JSL)
#define IPU6_TGL_ISYS_CSI2_NPORTS       8   // IPU6 TGL (overrides base)
#define IPU6EP_MTL_ISYS_CSI2_NPORTS     6   // IPU6EP MTL
```

---

## 3. 核心驅動程式架構

### 3.1 模組結構

上游 IPU6 驅動程式編譯為兩個核心模組：

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

**來源**：`drivers/media/pci/intel/ipu6/Makefile`

### 3.2 Kconfig 相依性

**來源**：`drivers/media/pci/intel/ipu6/Kconfig`

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

關鍵選項：
- **AUXILIARY_BUS**：用於 ISYS 子裝置註冊
- **MEDIA_CONTROLLER**：媒體圖拓撲所需
- **VIDEOBUF2_DMA_SG**：DMA 散集式緩衝區管理
- **V4L2_FWNODE**：用於感測器發現的韌體節點解析

### 3.3 PCI Probe 流程

主要進入點為 `ipu6.c` 中的 `ipu6_pci_probe()`。此函式協調整個硬體初始化序列：

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

### 3.4 關鍵資料結構

**`struct ipu6_device`** -- 頂層裝置結構：

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

**`struct ipu6_isys`** -- ISYS 子系統結構：

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

**`struct ipu6_isys_csi2`** -- CSI-2 接收器：

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

## 4. ipu-bridge：ACPI 感測器列舉

### 4.1 ACPI 到 V4L2 的差異問題

Windows ACPI 表使用專有方法（SSDB 緩衝區）來描述相機感測器，這些方法與 Linux V4L2 感測器驅動程式所使用的裝置樹 / fwnode 方式不相容。**ipu-bridge** 模組透過以下方式橋接此差異：

1. 掃描 ACPI 以尋找已知的感測器硬體 ID (HID)
2. 讀取 SSDB（感測器特定資料區塊）ACPI 緩衝區
3. 建立 V4L2 感測器驅動程式所預期的**軟體 fwnode** 屬性
4. 建構感測器與 IPU CSI-2 埠之間的圖形連接

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

### 4.2 支援的感測器表

ipu-bridge 維護一個支援感測器表，其中包含各感測器的鏈路頻率：

**來源**：`drivers/media/pci/intel/ipu-bridge.c`（第 50 行）

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

### 4.3 SSDB 緩衝區解析

SSDB（感測器特定資料區塊）是 Intel 專有的 ACPI 緩衝區，用於編碼感測器組態。橋接模組透過 `ipu_bridge_parse_ssdb()` 讀取它：

**來源**：`drivers/media/pci/intel/ipu-bridge.c`（第 294 行）

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

### 4.4 軟體節點圖形建構

對於每個偵測到的感測器，ipu-bridge 會建立一個模擬裝置樹結構的軟體節點圖形：

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

### 4.5 PCI Probe 中的橋接初始化

橋接在 ISYS 裝置建立期間初始化，**在** V4L2 註冊**之前**：

**來源**：`drivers/media/pci/intel/ipu6/ipu6.c`（第 378 行）

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

### 4.6 IVSC（Intel Visual Sensing Controller）支援

某些平台（第 11/12 代以上）在相機感測器與 IPU 之間包含 Intel 視覺感測控制器（IVSC）。ipu-bridge 透過為 IVSC CSI-2 直通插入額外的軟體節點來處理此情況：

```
  Graph with IVSC:

  Sensor ──► IVSC (MEI CSI device) ──► IPU CSI-2 Port

  Without IVSC:

  Sensor ──────────────────────────► IPU CSI-2 Port
```

IVSC ACPI 裝置 ID：`INTC1059`、`INTC1095`、`INTC100A`、`INTC10CF`

---

## 5. ISYS（輸入系統）架構

### 5.1 ISYS 概覽

ISYS（輸入系統）是 IPU 的相機擷取子系統。在上游驅動程式中，ISYS 是**唯一**的功能子系統 -- 它處理來自 CSI-2 感測器的原始影格擷取，並透過 DMA 傳輸到系統記憶體。

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

### 5.2 CSI-2 接收器實作

每個 CSI-2 埠以 V4L2 子裝置表示，具有 1 個接收端（sink pad）和最多 8 個來源端（source pad，每個虛擬通道一個）：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-isys-csi2.h`

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

### 5.3 CSI-2 PHY 變體

針對不同的 Intel 平台存在三種不同的 PHY 實作：

| PHY 類型 | 檔案 | 平台 | 描述 |
|----------|------|-----------|-------------|
| MCD PHY | `ipu6-isys-mcd-phy.c` | Tiger Lake (IPU6) | 多通道 DPHY |
| JSL PHY | `ipu6-isys-jsl-phy.c` | Jasper Lake (IPU6SE) | 簡化 DPHY |
| DWC PHY | `ipu6-isys-dwc-phy.c` | Meteor Lake (IPU6EP-MTL) | Synopsys DesignWare DPHY |

PHY 在執行時期透過 `struct ipu6_isys` 中的函式指標選擇：

```c
// ipu6-isys.h (line 153)
struct ipu6_isys {
    int (*phy_set_power)(struct ipu6_isys *isys,
                         struct ipu6_isys_csi2_config *cfg,
                         const struct ipu6_isys_csi2_timing *timing,
                         bool on);
};
```

### 5.4 韌體通訊

ISYS 韌體運行在 IPU 內部的專用純量處理器 (SPC) 上。核心透過共用記憶體訊息佇列與其通訊：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-fw-isys.h`

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

### 5.5 串流管理

ISYS 支援最多 16 個並行韌體串流。每個串流對應一個 CSI-2 虛擬通道，可產生多個輸出接腳（影像 + 元資料）：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-isys-video.h`（第 44 行）

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

## 6. V4L2 與 Media Controller 框架

### 6.1 媒體圖拓撲

IPU6 驅動程式建立一個 Media Controller 裝置（`/dev/mediaX`），其圖形表示感測器、CSI-2 接收器和擷取影像裝置節點之間的實體連接：

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

### 6.2 V4L2 影像裝置實作

每個 CSI-2 來源端會建立一個 V4L2 影像擷取裝置：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-isys-video.h`（第 82 行）

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

### 6.3 支援的像素格式

ISYS 支援原始 Bayer 格式（感測器的原生輸出）及部分打包格式：

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

### 6.4 緩衝區管理（videobuf2）

驅動程式使用 videobuf2-dma-sg（散集式）框架進行緩衝區管理：

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

### 6.5 V4L2 非同步感測器註冊

感測器驅動程式使用 V4L2 非同步框架與 ISYS V4L2 裝置進行非同步註冊。ipu-bridge 建立 fwnode 連接，當感測器驅動程式 probe 時，它們會註冊為 V4L2 子裝置：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-isys.c`（第 106 行）

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

## 7. 韌體載入與管理

### 7.1 韌體檔案命名

每個 IPU 版本需要其專屬的韌體二進位檔，儲存在 `/lib/firmware/intel/ipu/`：

| IPU 版本 | 韌體檔案 | 大約大小 |
|-------------|--------------|-----------------|
| IPU6 (TGL) | `intel/ipu/ipu6_fw.bin` | ~3-5 MB |
| IPU6SE (JSL) | `intel/ipu/ipu6se_fw.bin` | ~2-3 MB |
| IPU6EP (ADL/RPL) | `intel/ipu/ipu6ep_fw.bin` | ~3-5 MB |
| IPU6EP (ADL-N) | `intel/ipu/ipu6epadln_fw.bin` | ~3-5 MB |
| IPU6EP-MTL | `intel/ipu/ipu6epmtl_fw.bin` | ~5-7 MB |

**來源**：`drivers/media/pci/intel/ipu6/ipu6.h`（第 20-24 行）

### 7.2 CPD（Code Partition Directory）格式

韌體使用 Intel 的 CPD 格式，這是一種包含多個程式碼/資料條目的容器：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-cpd.c`

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

### 7.3 韌體驗證

Buttress 硬體單元在韌體可以執行之前對其進行驗證：

**來源**：`drivers/media/pci/intel/ipu6/ipu6-buttress.c`

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

### 7.4 Ubuntu 上的韌體發行

在 Ubuntu 上，IPU 韌體檔案透過 `linux-firmware` 套件發行：

```bash
# Ubuntu package providing IPU firmware
apt list --installed | grep linux-firmware
# linux-firmware/jammy-updates ... [installed]

# Firmware file locations
ls /lib/firmware/intel/ipu/
# ipu6_fw.bin  ipu6ep_fw.bin  ipu6epmtl_fw.bin  ipu6se_fw.bin ...
```

---

## 8. libcamera 框架架構

### 8.1 概覽

libcamera 是現代 Linux 相機框架，提供：
- 相機發現與列舉
- 管線組態與串流管理
- 影像處理演算法（IPA）
- 對複雜 V4L2/MC 硬體拓撲的抽象化
- 穩定的 C++ API 供應用程式使用

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

### 8.2 Pipeline Handler：IPU3

libcamera 中的 IPU3 管線處理器是為 Intel IPU3（Skylake/Kaby Lake）設計的，但其架構對於理解 IPU6 的處理方式具有參考價值：

**來源**：`src/libcamera/pipeline/ipu3/ipu3.cpp`

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

IPU3 處理器管理兩個媒體裝置：
- **CIO2**（CSI-2 Input/Output）：從感測器擷取原始影格
- **ImgU**（Image Processing Unit）：用於去馬賽克、縮放等的硬體 ISP

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

### 8.3 Pipeline Handler：Simple（用於僅有 ISYS 的 IPU6）

對於上游 IPU6（僅 ISYS，無 PSYS），使用 **Simple 管線處理器**。它適用於任何提供簡單擷取路徑的 V4L2 裝置：

**來源**：`src/libcamera/pipeline/simple/simple.cpp`

Simple 處理器：
1. 透過從感測器到影像擷取節點的遍歷來發現媒體圖
2. 沿整條管線（感測器 → CSI-2 → 影像）組態格式
3. 直接擷取原始影格（無硬體 ISP）
4. 可選擇透過 SoftwareISP IPA 模組套用軟體 ISP

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

### 8.4 相機發現流程

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

### 8.5 請求/緩衝區模型

libcamera 使用基於請求的模型進行影格擷取：

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

## 9. PipeWire 相機整合

### 9.1 PipeWire 相機架構

PipeWire 是 Linux 的現代多媒體伺服器，同時處理音訊和影像（相機）串流。它取代了 PulseAudio（用於音訊），並提供先前透過直接 V4L2 完成的相機存取：

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

### 9.2 SPA libcamera 外掛程式

PipeWire SPA（Simple Plugin API）libcamera 外掛程式為 libcamera 發現的每個相機建立一個 PipeWire 節點：

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

在桌面 Linux（GNOME、KDE）上，相機存取由 XDG Desktop Portal 管控：

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

## 10. GStreamer 相機管線

### 10.1 libcamerasrc 元素

GStreamer 提供 `libcamerasrc` 元素，使用 libcamera 進行相機存取：

```bash
# GStreamer pipeline: Camera preview with libcamera
gst-launch-1.0 libcamerasrc ! videoconvert ! autovideosink

# GStreamer pipeline: Camera capture to file
gst-launch-1.0 libcamerasrc ! video/x-raw,width=1920,height=1080 ! \
    videoconvert ! x264enc ! mp4mux ! filesink location=output.mp4

# GStreamer pipeline: Camera via PipeWire
gst-launch-1.0 pipewiresrc ! videoconvert ! autovideosink
```

### 10.2 V4L2 直接存取（傳統方式）

應用程式也可以繞過 libcamera 直接透過 V4L2 存取相機。這適用於簡單的使用情境，但需要應用程式自行處理原始 Bayer 資料：

```bash
# Direct V4L2 capture (raw Bayer, no ISP)
v4l2-ctl --device=/dev/video0 --stream-mmap --stream-count=10 \
    --stream-to=raw_frames.bin

# FFmpeg with V4L2
ffmpeg -f v4l2 -input_format rawvideo -video_size 1920x1080 \
    -i /dev/video0 -vframes 100 output.mp4
```

---

## 11. 完整資料流：從應用程式到硬體

### 11.1 完整堆疊資料流（PipeWire 路徑）

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

### 11.2 時序與效能

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

## 12. 樹外驅動與上游驅動比較

### 12.1 功能比較

Intel 樹外驅動程式（`github.com/intel/ipu6-drivers`）相較於上游核心驅動程式包含大量額外功能：

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

### 12.2 架構差異

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

### 12.3 使用者空間堆疊差異

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

## 13. IPU7 與未來上游支援

### 13.1 IPU7 現狀

截至 2026 年 2 月，IPU7（見於 Lunar Lake 和 Panther Lake）**尚未**獲得上游 Linux 核心支援。IPU7 驅動程式僅存在於：

1. **Android 核心樹**（Intel 內部，用於 Panther Lake Android）
2. **樹外儲存庫**（intel/ipu7-drivers，尚未公開發布）

IPU7 與 IPU6 共享相同的通用架構（ISYS + CSI-2 + 韌體），但暫存器和韌體介面有重大變更。

### 13.2 上游化路徑

IPU 驅動程式上游化的典型路徑：

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

### 13.3 IPU7 上游化的挑戰

1. **無 PSYS**：與 IPU6 相同，PSYS（硬體 ISP）依賴無法開源的 Intel 專有韌體介面和演算法函式庫
2. **新 PHY**：IPU7 使用不同的 CSI-2 PHY 實作
3. **韌體格式**：與 IPU6 不同的韌體二進位格式
4. **暫存器變更**：ISYS 組態的新暫存器佈局
5. **libcamera 處理器**：將需要新的或更新的管線處理器

---

## 14. 檔案參考

### 14.1 上游核心 IPU6 驅動程式檔案

**位置**：`drivers/media/pci/intel/ipu6/`

| 檔案 | 行數 | 用途 |
|------|-------|---------|
| `ipu6.c` | ~847 | PCI 驅動程式：probe、韌體載入、ISYS/PSYS 初始化、電源管理 |
| `ipu6.h` | ~342 | 主標頭檔：`struct ipu6_device`、硬體變體、MMU 組態 |
| `ipu6-bus.c` | ~200 | ISYS/PSYS 的輔助匯流排裝置管理 |
| `ipu6-bus.h` | ~80 | 匯流排裝置結構 |
| `ipu6-buttress.c` | ~700 | 電源管理、韌體驗證、與 CSE 的 IPC |
| `ipu6-buttress.h` | ~100 | Buttress 控制結構、IPC 定義 |
| `ipu6-cpd.c` | ~250 | CPD 韌體容器解析與驗證 |
| `ipu6-cpd.h` | ~80 | CPD 結構（標頭、條目、元資料） |
| `ipu6-dma.c` | ~300 | 搭配 IPU MMU 支援的 DMA 分配 |
| `ipu6-dma.h` | ~30 | DMA 函式宣告 |
| `ipu6-fw-com.c` | ~350 | 韌體訊息佇列通訊 |
| `ipu6-fw-com.h` | ~50 | 韌體通訊結構 |
| `ipu6-fw-isys.c` | ~500 | ISYS 韌體命令介面 |
| `ipu6-fw-isys.h` | ~300 | ISYS 韌體結構：串流組態、接腳組態、回應 |
| `ipu6-isys.c` | ~1200 | ISYS 主要：probe、V4L2/MC 註冊、ISR、電源 |
| `ipu6-isys.h` | ~203 | ISYS 結構：`struct ipu6_isys`、串流組態 |
| `ipu6-isys-csi2.c` | ~600 | CSI-2 V4L2 子裝置實作 |
| `ipu6-isys-csi2.h` | ~79 | CSI-2 結構：`struct ipu6_isys_csi2` |
| `ipu6-isys-video.c` | ~900 | V4L2 影像裝置：ioctl、串流、格式協商 |
| `ipu6-isys-video.h` | ~136 | 影像結構：`struct ipu6_isys_video`、串流 |
| `ipu6-isys-queue.c` | ~500 | videobuf2 佇列管理、緩衝區處理 |
| `ipu6-isys-queue.h` | ~80 | 佇列結構 |
| `ipu6-isys-subdev.c` | ~400 | V4L2 子裝置輔助程式、pad 格式操作 |
| `ipu6-isys-subdev.h` | ~60 | 子裝置結構 |
| `ipu6-isys-mcd-phy.c` | ~450 | Tiger Lake 的 MCD PHY |
| `ipu6-isys-jsl-phy.c` | ~200 | Jasper Lake 的 JSL PHY |
| `ipu6-isys-dwc-phy.c` | ~500 | Meteor Lake 的 DWC PHY |
| `ipu6-mmu.c` | ~800 | IPU 內部 MMU：L1/L2 TLB、ZLW 機制 |
| `ipu6-mmu.h` | ~60 | MMU 結構 |
| `Kconfig` | ~19 | 建置組態 |
| `Makefile` | ~24 | 兩個模組的建置規則 |

### 14.2 ipu-bridge 檔案

**位置**：`drivers/media/pci/intel/`

| 檔案 | 用途 |
|------|---------|
| `ipu-bridge.c` | ACPI 感測器列舉、軟體 fwnode 建立 |

**位置**：`include/media/`

| 檔案 | 用途 |
|------|---------|
| `ipu-bridge.h` | 橋接結構：`struct ipu_sensor`、SSDB、swnode 列舉 |
| `ipu6-pci-table.h` | IPU6 變體的 PCI 裝置 ID 表 |

### 14.3 平台暫存器標頭檔

| 檔案 | 用途 |
|------|---------|
| `ipu6-platform-buttress-regs.h` | Buttress 暫存器偏移量與位元定義 |
| `ipu6-platform-isys-csi2-reg.h` | CSI-2 接收器暫存器偏移量 |
| `ipu6-platform-regs.h` | 一般平台暫存器（ISYS、PSYS 偏移量） |

### 14.4 libcamera 檔案（關鍵元件）

**位置**：`src/libcamera/pipeline/ipu3/`

| 檔案 | 用途 |
|------|---------|
| `ipu3.cpp` | IPU3 管線處理器：相機建立、組態、啟動 |
| `cio2.cpp` | CIO2（CSI-2 I/O）裝置抽象 |
| `cio2.h` | CIO2 類別定義 |
| `imgu.cpp` | ImgU（Image Processing Unit）裝置抽象 |
| `imgu.h` | ImgU 類別定義 |
| `frames.cpp` | CIO2 與 ImgU 之間的影格追蹤 |

**位置**：`src/libcamera/pipeline/simple/`

| 檔案 | 用途 |
|------|---------|
| `simple.cpp` | Simple 管線處理器（用於僅有 ISYS 的 IPU6） |

**位置**：`src/libcamera/`（核心）

| 檔案 | 用途 |
|------|---------|
| `camera.cpp` | Camera 類別：configure、start、stop、queueRequest |
| `camera_manager.cpp` | 相機發現與生命週期管理 |
| `pipeline_handler.cpp` | 基礎管線處理器類別 |
| `media_device.cpp` | Media Controller 裝置抽象 |
| `v4l2_videodevice.cpp` | V4L2 影像裝置包裝器 |
| `v4l2_subdevice.cpp` | V4L2 子裝置包裝器 |

### 14.5 樹外驅動程式檔案（github.com/intel/ipu6-drivers）

**位置**：`drivers/media/pci/intel/`

| 檔案 | 用途 |
|------|---------|
| `ipu.c` | 主要 PCI 驅動程式（等同上游的 `ipu6.c`） |
| `ipu-isys-csi2-be.c` | CSI-2 後端實體（不在上游中） |
| `ipu-isys-csi2-be-soc.c` | SoC 後端實體（不在上游中） |
| `ipu-isys-tpg.c` | 測試圖案產生器（不在上游中） |
| `ipu-trace.c` | 偵錯追蹤（不在上游中） |

**位置**：`drivers/media/pci/intel/ipu6/psys/`

| 檔案 | 用途 |
|------|---------|
| `ipu6-psys.c` | PSYS 裝置驅動程式（硬體 ISP） |
| `ipu-fw-psys.c` | PSYS 韌體介面 |
| `ipu6-ppg.c` | 參數程式群組 |
| `ipu6-l-scheduler.c` | 工作項目排程 |
| `ipu-resources.c` | PSYS 資源分配 |

### 14.6 韌體檔案（Ubuntu：/lib/firmware/）

| 檔案 | IPU 版本 |
|------|------------|
| `intel/ipu/ipu6_fw.bin` | IPU6 (Tiger Lake) |
| `intel/ipu/ipu6se_fw.bin` | IPU6SE (Jasper Lake) |
| `intel/ipu/ipu6ep_fw.bin` | IPU6EP (Alder Lake / Raptor Lake) |
| `intel/ipu/ipu6epadln_fw.bin` | IPU6EP (Alder Lake N) |
| `intel/ipu/ipu6epmtl_fw.bin` | IPU6EP-MTL (Meteor Lake) |

### 14.7 Ubuntu 使用者空間套件

| 套件 | 用途 |
|---------|---------|
| `linux-firmware` | IPU 韌體二進位檔 |
| `libcamera0.3` | libcamera 共用函式庫 |
| `libcamera-tools` | `cam` 命令列測試工具 |
| `pipewire` | PipeWire 守護程式 |
| `libspa-0.2-libcamera` | PipeWire SPA libcamera 外掛程式 |
| `wireplumber` | PipeWire 工作階段管理員 |
| `gstreamer1.0-pipewire` | GStreamer PipeWire 元素 |
| `libgstreamer-plugins-bad1.0` | 包含 `libcamerasrc` GStreamer 元素 |

### 14.8 建立的裝置節點

| 裝置節點 | 類型 | 用途 |
|-------------|------|---------|
| `/dev/media0` | Media Controller | 圖形拓撲導航 |
| `/dev/video0..N` | V4L2 Video | 影格擷取（每個 CSI-2 來源端一個） |
| `/dev/v4l-subdev0..N` | V4L2 Subdevice | 感測器與 CSI-2 組態 |

--- 文件結束 ---
