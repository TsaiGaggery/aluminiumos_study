# Android 與 Ubuntu IPU7 相機架構比較

## Intel Panther Lake (PTL) 平台 - 完整技術比較

### Android (ALOS) Camera2/HAL 堆疊 vs Ubuntu libcamera/PipeWire 堆疊
### IPU7 影像處理單元 - Kernel 6.18 / Kernel 6.10+
### 2026 年 2 月

---

## 目錄

1. [執行摘要](#1-執行摘要)
2. [並列架構總覽](#2-並列架構總覽)
3. [核心驅動程式層比較](#3-核心驅動程式層比較)
4. [使用者空間框架比較](#4-使用者空間框架比較)
5. [影像處理管線比較](#5-影像處理管線比較)
6. [緩衝區管理與 DMA 比較](#6-緩衝區管理與-dma-比較)
7. [3A 演算法比較](#7-3a-演算法比較)
8. [應用程式整合比較](#8-應用程式整合比較)
9. [感測器與硬體支援](#9-感測器與硬體支援)
10. [效能與功能矩陣](#10-效能與功能矩陣)
11. [移植考量](#11-移植考量)
12. [關鍵檔案參考](#12-關鍵檔案參考)

---

## 1. 執行摘要

本文件提供 Intel Panther Lake (PTL) 平台上相機子系統在兩個支援作業系統之間的全面技術比較：Android (ALOS) 與 Ubuntu/Linux。兩個平台皆以 Intel IPU7 影像處理單元為目標，但在相機擷取、影像處理及應用程式整合方面採用了根本不同的架構方式。

### 1.1 根本架構差異

兩個平台之間最重要的差異在於 PSYS（處理系統），即 IPU 的硬體 ISP 所扮演的角色：

```
  +----------------------------------------------------------------------+
  |  THE CRITICAL DIFFERENCE: PSYS (Hardware ISP)                        |
  |                                                                      |
  |  Android (ALOS):                                                     |
  |    ISYS (raw capture) --> PSYS (hardware ISP) --> Application        |
  |    - Full hardware-accelerated image processing pipeline             |
  |    - Intel CCA library for 3A (AE, AF, AWB)                         |
  |    - Graph-based processing model with PSysDevice API                |
  |    - Zero-copy DMA-BUF path between ISYS and PSYS                   |
  |                                                                      |
  |  Ubuntu/Linux (upstream):                                            |
  |    ISYS (raw capture) --> SoftwareISP (CPU) --> Application          |
  |    - PSYS is NOT available in the upstream kernel driver              |
  |    - All image processing done in software on CPU                    |
  |    - libcamera SoftwareISP for debayering                            |
  |    - Basic software 3A only (AE + AWB, no AF)                        |
  +----------------------------------------------------------------------+
```

此差異貫穿整個堆疊，影響影像品質、功耗、延遲及功能可用性。

### 1.2 平台摘要

| 面向 | Android (ALOS) | Ubuntu/Linux |
|--------|----------------|--------------|
| 平台 | Intel Panther Lake (PTL) | Intel Panther Lake (PTL) |
| IPU 世代 | IPU7 | IPU6/IPU7（驅動程式命名為 ipu6） |
| 核心 | 基於 6.18 | 6.10+ 主線 |
| 相機 API | Camera2 / CameraX | PipeWire + libcamera |
| HAL 層 | Camera HAL3 (AIDL) | libcamera pipeline handler |
| ISP 管線 | 硬體 PSYS | 軟體 ISP（CPU） |
| 3A 演算法 | Intel CCA（專有） | libcamera IPA（開源） |
| 輸出格式 | YUV/JPEG（已處理） | Raw Bayer -> 軟體去馬賽克 |
| 核心模組 | ipu7.ko, ipu7-isys.ko, ipu7-psys.ko | intel-ipu6.ko, intel-ipu6-isys.ko |
| 感測器探索 | ACPI + ipu-bridge + HAL 設定 | ACPI + ipu-bridge + V4L2 async |

### 1.3 為何這很重要

平台的選擇直接影響：

- **影像品質**：Android 的硬體 PSYS 管線搭配 Intel CCA 可提供量產等級的影像品質。Ubuntu 的軟體 ISP 僅提供基本去馬賽克處理，色彩處理有限且無硬體降噪。

- **效能**：Android 將 ISP 處理卸載至專用硬體，釋放 CPU 運算週期。Ubuntu 在 CPU 上執行所有影像處理，在較高解析度下會消耗大量運算資源。

- **功能集**：Android 支援自動對焦（AF）、進階 AWB、色調映射、邊緣增強及降噪。Ubuntu 僅支援基本的 AE 和 AWB。

- **功耗**：在相同吞吐量下，硬體 ISP 處理的功率效率遠高於軟體處理。

---

## 2. 並列架構總覽

### 2.1 完整堆疊比較圖

```
  Android (ALOS) Camera Stack              Ubuntu/Linux Camera Stack
  =============================            ============================

  +---------------------------+            +---------------------------+
  | Android Application       |            | Linux Application         |
  | (Camera2 API / CameraX)   |            | (Cheese, Firefox, OBS)    |
  +----------+----------------+            +----------+----------------+
             |                                        |
             | Binder IPC                             | PipeWire IPC
             |                                        | (Unix socket)
  +----------v----------------+            +----------v----------------+
  | CameraService             |            | PipeWire Daemon           |
  | (system_server process)   |            | + WirePlumber             |
  |                           |            |                           |
  | CameraProviderManager     |            | SPA libcamera plugin      |
  | Camera3Device             |            | (libspa-libcamera)        |
  +----------+----------------+            +----------+----------------+
             |                                        |
             | AIDL IPC                               | libcamera C++ API
             |                                        |
  +----------v----------------+            +----------v----------------+
  | Camera HAL (AIDL)         |            | libcamera                 |
  |                           |            |                           |
  | +-----+ +--------------+ |            | +---------------------+   |
  | |Google| |Intel IPU7    | |            | | SimplePipelineHandler|   |
  | |Desk  | |Camera HAL   | |            | | (generic V4L2)      |   |
  | |HAL   | |             | |            | +----------+----------+   |
  | |(Rust)| |(C++)        | |            |            |              |
  | |USB   | |MIPI CSI-2   | |            | +----------v----------+   |
  | +--+---+ +------+------+ |            | | SoftwareISP         |   |
  |    |            |         |            | | (CPU debayer)       |   |
  +----+------------+---------+            | +----------+----------+   |
       |            |                      |            |              |
       |            | +-- 3A Engine        | +----------v----------+   |
       |            | |   AiqUnit          | | IPA (soft)          |   |
       |            | |   Intel CCA        | | (AE + AWB only)     |   |
       |            | +-- IPA Client/      | +---------------------+   |
       |            |     Server           +----------+----------------+
       |            |                                 |
  =====|============|===== Kernel ========= =========|=================
       |            |                                 |
       | ioctl      | ioctl (V4L2)                    | ioctl (V4L2)
       |            |                                 |
  +----v---+  +-----v-------------------+  +----------v----------------+
  | UVC    |  | Intel IPU7 Driver       |  | Intel IPU6 Driver         |
  | Driver |  | (ipu7.ko)               |  | (intel-ipu6.ko)           |
  |        |  | +-------+ +----------+  |  | +-------+                 |
  |        |  | | ISYS  | | PSYS     |  |  | | ISYS  |  (no PSYS)     |
  |        |  | |(capture| |(process)|  |  | |(capture|                |
  |        |  | +---+---+ +----+-----+  |  | +---+---+                 |
  |        |  |     |          |        |  |     |                      |
  |        |  | +---v----------v-----+  |  | +---v------------------+  |
  |        |  | | IPU Firmware       |  |  | | IPU Firmware         |  |
  |        |  | +--------------------+  |  | +----------------------+  |
  +---+----+  +---------+----------+----+  +---------+-----------------+
      |                 |          |                  |
  ====|=================|==========|==================|=== Hardware ====
      |                 |          |                  |
  +---v----+     +------v----+    |           +------v-----------+
  | USB    |     | CSI-2     |    |           | CSI-2 Receiver   |
  | Camera |     | Receiver  |    |           | (ISYS only)      |
  +--------+     +-----+-----+    |           +------+-----------+
                       |          |                  |
                 +-----v-----+   |           +------v-----------+
                 | Camera     |   |           | Camera Sensor    |
                 | Sensor     |   |           | (MIPI CSI-2)     |
                 | (MIPI)     |   |           +------------------+
                 +------------+   |
                                  |
                            +-----v-----+
                            | PSYS HW   |
                            | ISP       |
                            | (Android  |
                            |  only)    |
                            +-----------+
```

### 2.2 元件對照表

此表格映射兩個平台之間的對等元件：

```
  +------------------------------+-----------------------------+-----------------------------+
  | Function                     | Android (ALOS)              | Ubuntu/Linux                |
  +------------------------------+-----------------------------+-----------------------------+
  | Application API              | Camera2 API (Java/Kotlin)   | PipeWire API (C)            |
  |                              | CameraX (Jetpack)           | GStreamer (C/Python)         |
  |                              |                             | libcamera API (C++)         |
  +------------------------------+-----------------------------+-----------------------------+
  | Session Management           | CameraService               | PipeWire Daemon             |
  |                              | (system_server, C++)        | + WirePlumber               |
  +------------------------------+-----------------------------+-----------------------------+
  | HAL / Pipeline Handler       | ICameraProvider (AIDL)      | PipelineHandler (C++)       |
  |                              | ICameraDeviceSession        | SimplePipelineHandler       |
  +------------------------------+-----------------------------+-----------------------------+
  | USB Camera HAL               | Google Desktop HAL (Rust)   | UVC + libcamera Simple      |
  +------------------------------+-----------------------------+-----------------------------+
  | MIPI Camera HAL              | Intel IPU7 HAL (C++)        | libcamera Simple + SoftISP  |
  +------------------------------+-----------------------------+-----------------------------+
  | Raw Capture (ISYS)           | CaptureUnit class           | V4L2VideoDevice             |
  |                              | DeviceBase / MainDevice     | (direct V4L2 capture)       |
  +------------------------------+-----------------------------+-----------------------------+
  | Image Processing             | ProcessingUnit (PSYS HW)    | SoftwareISP (CPU)           |
  |                              | PipeManager + PSysDevice    | Debayer algorithm            |
  +------------------------------+-----------------------------+-----------------------------+
  | 3A Algorithms                | AiqUnit + Intel CCA         | IPA module (soft)           |
  |                              | (AE, AF, AWB, ISP tuning)   | (AE, AWB only, no AF)       |
  +------------------------------+-----------------------------+-----------------------------+
  | 3A Execution Model           | IPA Client/Server           | IPA proxy (in-process       |
  |                              | (sandboxed process)         |  for SoftwareISP)           |
  +------------------------------+-----------------------------+-----------------------------+
  | Request Model                | RequestThread (event-driven)| Camera::queueRequest()      |
  |                              | NEW_REQUEST/FRAME/STATS/SOF | Request-complete signal     |
  +------------------------------+-----------------------------+-----------------------------+
  | Per-frame State              | CameraContext + DataContext  | Request controls            |
  |                              | AiqResultStorage            | (per-request metadata)      |
  +------------------------------+-----------------------------+-----------------------------+
  | Buffer Allocation            | Gralloc (ION/DMA-BUF)       | V4L2 MMAP or DMA-BUF        |
  |                              | CameraBuffer wrapper        | FrameBuffer (libcamera)     |
  +------------------------------+-----------------------------+-----------------------------+
  | Kernel V4L2 Driver           | ipu6-isys-video.c           | ipu6-isys-video.c (same)    |
  |                              | + ipu6-psys (PSYS device)   | (no PSYS device node)       |
  +------------------------------+-----------------------------+-----------------------------+
  | Firmware                     | ipu6_fw.bin (CPD format)    | ipu6_fw.bin (CPD format)    |
  |                              | Authenticated via Buttress  | Authenticated via Buttress  |
  +------------------------------+-----------------------------+-----------------------------+
  | Sensor Enumeration           | ipu-bridge (ACPI/SSDB)      | ipu-bridge (ACPI/SSDB)      |
  |                              | + HAL XML config files      | + V4L2 async notifier       |
  +------------------------------+-----------------------------+-----------------------------+
  | Permission Model             | Android permissions         | XDG Camera Portal           |
  |                              | (android.permission.CAMERA) | (Flatpak/Snap sandboxing)   |
  +------------------------------+-----------------------------+-----------------------------+
```

### 2.3 層級數量比較

```
  Android Camera Stack Layers:            Ubuntu Camera Stack Layers:

  Layer 8: Application (Java)             Layer 5: Application (C/Python)
  Layer 7: Camera2 API                    Layer 4: PipeWire
  Layer 6: CameraService                  Layer 3: libcamera + IPA
  Layer 5: AIDL HAL Interface             Layer 2: V4L2 / Media Controller
  Layer 4: Camera HAL (C++/Rust)          Layer 1: IPU6 Kernel Driver
  Layer 3: IPA Client/Server              Layer 0: IPU Hardware + Sensor
  Layer 2: V4L2 / Media Controller
  Layer 1: IPU Kernel Driver
  Layer 0: IPU Hardware + Sensor

  Total: 9 layers                         Total: 6 layers
  (More abstraction, more features)       (Thinner stack, less overhead)
```

---

## 3. 核心驅動程式層比較

### 3.1 模組結構

兩個平台在 V4L2/ISYS 層級共用相同的 IPU 核心驅動程式碼庫，但在 PSYS 支援和模組組織方面有顯著差異。

```
  Android Kernel Modules:                 Ubuntu Kernel Modules:
  ========================               ========================

  +---------------------------+          +---------------------------+
  | ipu7.ko                   |          | intel-ipu6.ko             |
  | (PCI driver + core)       |          | (PCI driver + core)       |
  |                           |          |                           |
  | - PCI probe & FW load     |          | - PCI probe & FW load     |
  | - Buttress power/IPC      |          | - Buttress power/IPC      |
  | - MMU (IOMMU)             |          | - MMU (IOMMU)             |
  | - CPD firmware parsing    |          | - CPD firmware parsing    |
  | - Auxiliary bus devices   |          | - Auxiliary bus devices   |
  | - DMA operations          |          | - DMA operations          |
  +---------------------------+          +---------------------------+

  +---------------------------+          +---------------------------+
  | ipu7-isys.ko              |          | intel-ipu6-isys.ko        |
  | (Input System)            |          | (Input System)            |
  |                           |          |                           |
  | - V4L2 video devices      |          | - V4L2 video devices      |
  | - CSI-2 receiver subdev   |          | - CSI-2 receiver subdev   |
  | - Media Controller graph  |          | - Media Controller graph  |
  | - FW stream commands      |          | - FW stream commands      |
  | - VB2 buffer queues       |          | - VB2 buffer queues       |
  | - ISR handling             |          | - ISR handling             |
  +---------------------------+          +---------------------------+

  +---------------------------+
  | ipu7-psys.ko              |          (NO EQUIVALENT IN UBUNTU)
  | (Processing System)       |
  |                           |          Ubuntu has NO PSYS kernel
  | - Hardware ISP control    |          module. All image processing
  | - Graph-based processing  |          is done in userspace via
  | - PSysDevice API          |          libcamera's SoftwareISP.
  | - Task submission/polling |
  | - Buffer registration     |
  | - PPG (Parameter          |
  |   Programming Groups)     |
  +---------------------------+
```

### 3.2 核心原始碼位置

```
  +------------------------------+------------------------------------------+
  | Component                    | Android                                  |
  +------------------------------+------------------------------------------+
  | Main PCI driver              | drivers/media/pci/intel/ipu6/ipu6.c      |
  | ISYS driver                  | drivers/media/pci/intel/ipu6/ipu6-isys.c |
  | ISYS video device            | ipu6-isys-video.c                        |
  | CSI-2 receiver               | ipu6-isys-csi2.c                         |
  | Buffer queue                 | ipu6-isys-queue.c                        |
  | Firmware communication       | ipu6-fw-com.c                            |
  | Firmware ABI                 | ipu6-fw-isys.h                           |
  | MMU                          | ipu6-mmu.c                               |
  | Buttress (power/security)    | ipu6-buttress.c                          |
  | CPD firmware parser          | ipu6-cpd.c                               |
  | Auxiliary bus                | ipu6-bus.c                               |
  | ipu-bridge (ACPI enum)       | drivers/media/pci/intel/ipu-bridge.c     |
  +------------------------------+------------------------------------------+
  | Component                    | Ubuntu                                   |
  +------------------------------+------------------------------------------+
  | Main PCI driver              | drivers/media/pci/intel/ipu6/ipu6.c      |
  | ISYS driver                  | drivers/media/pci/intel/ipu6/ipu6-isys.c |
  | ISYS video device            | ipu6-isys-video.c                        |
  | CSI-2 receiver               | ipu6-isys-csi2.c                         |
  | Buffer queue                 | ipu6-isys-queue.c                        |
  | Firmware communication       | ipu6-fw-com.c                            |
  | Firmware ABI                 | ipu6-fw-isys.h                           |
  | MMU                          | ipu6-mmu.c                               |
  | Buttress (power/security)    | ipu6-buttress.c                          |
  | CPD firmware parser          | ipu6-cpd.c                               |
  | Auxiliary bus                | ipu6-bus.c                               |
  | ipu-bridge (ACPI enum)       | drivers/media/pci/intel/ipu-bridge.c     |
  +------------------------------+------------------------------------------+

  NOTE: The ISYS-level kernel code is essentially identical between platforms.
  The critical difference is that Android includes PSYS kernel support (the
  ipu7-psys.ko module) while Ubuntu does not.
```

### 3.3 Kconfig 依賴性比較

```
  Android Kernel Config:                   Ubuntu Kernel Config:

  CONFIG_VIDEO_INTEL_IPU6=m               CONFIG_VIDEO_INTEL_IPU6=m
  CONFIG_VIDEO_INTEL_IPU6_ISYS=m          (ISYS built into intel-ipu6-isys)
  CONFIG_VIDEO_INTEL_IPU6_PSYS=m          (NO PSYS config option upstream)
  CONFIG_MEDIA_CONTROLLER=y               CONFIG_MEDIA_CONTROLLER=y
  CONFIG_VIDEO_V4L2_SUBDEV_API=y          CONFIG_VIDEO_V4L2_SUBDEV_API=y
  CONFIG_VIDEOBUF2_DMA_SG=y              CONFIG_VIDEOBUF2_DMA_SG=y
  CONFIG_V4L2_FWNODE=y                   CONFIG_V4L2_FWNODE=y
  CONFIG_AUXILIARY_BUS=y                 CONFIG_AUXILIARY_BUS=y
  CONFIG_IOMMU_IOVA=y                    CONFIG_IOMMU_IOVA=y
  CONFIG_IPU_BRIDGE=y                    CONFIG_IPU_BRIDGE=y
```

### 3.4 PCI 裝置 ID 與硬體版本偵測

兩個平台都透過 probe 函式中的 PCI 裝置 ID 來識別 IPU 硬體世代。上游驅動程式（Ubuntu 使用）目前僅支援 IPU6 變體。IPU7 支援尚未進入上游。

```
  PCI Device ID Mapping:

  +----------------+-------------+-------------------+------------------+
  | PCI Device ID  | IPU Version | Intel SoC         | Platform Support |
  +----------------+-------------+-------------------+------------------+
  | 0x9a19         | IPU6        | Tiger Lake        | Ubuntu           |
  | 0x4e19         | IPU6SE      | Jasper Lake       | Ubuntu           |
  | 0x465d         | IPU6EP      | Alder/Raptor Lake | Ubuntu           |
  | 0x7d19         | IPU6EP-MTL  | Meteor Lake       | Ubuntu           |
  | (TBD)          | IPU7        | Lunar Lake        | Android only*    |
  | (TBD)          | IPU7-PTL    | Panther Lake      | Android only*    |
  +----------------+-------------+-------------------+------------------+

  * IPU7 entered Linux staging in kernel 6.17 with 17,000+ lines of code,
    but is not yet fully upstream in production kernels as of February 2026.
```

### 3.5 ISYS 驅動程式架構（共用）

ISYS（輸入系統）架構在兩個平台之間基本上是共用的。兩者使用相同的韌體通訊協定、相同的 CSI-2 接收器實作，以及相同的 VB2 緩衝區佇列管理。

```
  ISYS Architecture (Common to Both Platforms):

  +------------------------------------------------------------------+
  |  ISYS (Input System) -- Shared Between Android and Ubuntu         |
  |                                                                    |
  |  +------------------------------------------------------------+  |
  |  |  CSI-2 Receivers (one V4L2 subdevice per port)              |  |
  |  |                                                              |  |
  |  |  +---------+ +---------+ +---------+       +---------+     |  |
  |  |  | Port 0  | | Port 1  | | Port 2  |  ...  | Port N  |     |  |
  |  |  | DPHY/   | | DPHY/   | | DPHY/   |       | DPHY/   |     |  |
  |  |  | DWC     | | DWC     | | DWC     |       | DWC     |     |  |
  |  |  +----+----+ +----+----+ +----+----+       +----+----+     |  |
  |  |       |            |           |                 |          |  |
  |  +-------+------------+-----------+-----------------+----------+  |
  |          |            |           |                 |              |
  |  +-------v------------v-----------v-----------------v----------+  |
  |  |  Stream Multiplexer (Virtual Channels 0-15)                  |  |
  |  |  Up to 16 concurrent firmware streams                        |  |
  |  +---------------------------+----------------------------------+  |
  |                              |                                     |
  |  +---------------------------v----------------------------------+  |
  |  |  ISYS Pixel Buffer (SRAM)                                     |  |
  |  |  Threshold-based watermark for DMA triggering                 |  |
  |  +---------------------------+----------------------------------+  |
  |                              |                                     |
  |  +---------------------------v----------------------------------+  |
  |  |  DMA Engine -> IPU MMU -> System Memory                       |  |
  |  +--------------------------------------------------------------+  |
  +------------------------------------------------------------------+
```

### 3.6 PSYS 驅動程式架構（僅限 Android）

PSYS（處理系統）提供硬體 ISP 功能，僅在 Android 上可用。它公開一個字元裝置（`/dev/ipu-psys0`）用於基於圖形的影像處理。

```
  PSYS Architecture (Android Only):

  +------------------------------------------------------------------+
  |  PSYS (Processing System) -- Android Only                         |
  |                                                                    |
  |  Userspace API: /dev/ipu-psys0                                     |
  |                                                                    |
  |  +------------------------------------------------------------+  |
  |  |  PSysDevice (HAL-side)                                       |  |
  |  |                                                              |  |
  |  |  addGraph(PSysGraph)     -- Define processing topology       |  |
  |  |  registerBuffer(buf)     -- Register DMA-BUF for processing  |  |
  |  |  addTask(PSysTask)       -- Submit processing work item      |  |
  |  |  poll()                  -- Wait for task completion         |  |
  |  +------------------------------------------------------------+  |
  |                                                                    |
  |  PSysGraph Structure:                                              |
  |  +------------------+    +------------------+                      |
  |  | PSysNode         |    | PSysNode         |                      |
  |  | (ISP algorithm)  |--->| (ISP algorithm)  |                      |
  |  | nodeCtxId: 0     |    | nodeCtxId: 1     |                      |
  |  | terminals: [     |    | terminals: [     |                      |
  |  |   input,         |    |   input,         |                      |
  |  |   output,        |    |   output,        |                      |
  |  |   params ]       |    |   params ]       |                      |
  |  +------------------+    +------------------+                      |
  |                                                                    |
  |  PSysTask:                                                         |
  |  - nodeCtxId (which node to execute)                               |
  |  - sequence (frame sequence number)                                |
  |  - terminalBuffers (input/output DMA-BUF handles)                  |
  |                                                                    |
  |  Kernel PSYS Module (ipu7-psys.ko):                                |
  |  - ipu6-psys.c          : Device driver, ioctl interface           |
  |  - ipu-fw-psys.c        : Firmware command interface               |
  |  - ipu6-ppg.c           : Parameter Programming Groups             |
  |  - ipu6-l-scheduler.c   : Task scheduling                         |
  |  - ipu-resources.c      : Resource allocation                      |
  +------------------------------------------------------------------+

  Ubuntu has NO equivalent component. Raw Bayer frames go directly
  to libcamera's SoftwareISP for CPU-based processing.
```

### 3.7 韌體載入與驗證（共用）

兩個平台使用相同的韌體載入與驗證機制，透過 Buttress 硬體單元進行：

```
  Firmware Authentication Flow (Shared):

  1. request_firmware("intel/ipu/ipu6_fw.bin")
     +-- Firmware loaded from /lib/firmware/ (Ubuntu)
     +-- Firmware loaded from /vendor/firmware/ (Android)

  2. ipu6_cpd_validate_cpd_file()
     +-- Parse CPD (Code Partition Directory) header
     +-- Validate magic: "$CPD" (0x44504324)
     +-- Extract manifest, metadata, module data

  3. ipu6_buttress_map_fw_image()
     +-- Map firmware pages through IPU MMU
     +-- Create scatter-gather table for DMA

  4. ipu6_cpd_create_pkg_dir()
     +-- Extract package directory from CPD container
     +-- Identify component entries (ISYS FW, PSYS FW)

  5. ipu6_configure_spc()
     +-- Write firmware start PC to SPC registers
     +-- Write package directory address to DMEM

  6. ipu6_buttress_authenticate()
     +-- Send authentication request to CSE
         (Converged Security Engine)
     +-- CSE verifies firmware digital signature
     +-- On success: SPC starts executing firmware
     +-- Bootload timeout: 5 seconds
     +-- Authentication timeout: 10 seconds
```

### 3.8 ipu-bridge ACPI 感測器列舉（共用）

兩個平台使用相同的 ipu-bridge 模組從 ACPI 表格中探索相機感測器。此橋接器將 Windows 導向的 ACPI 感測器描述符（SSDB 緩衝區）轉譯為 Linux V4L2 fwnode 圖形連接。

```
  ipu-bridge Translation (Common to Both Platforms):

  ACPI DSDT (Windows-oriented)             V4L2 fwnode (Linux-oriented)
  +----------------------------+          +----------------------------+
  | Device (INT3474)           |          | Software Node: "INT3474-2" |
  |   _HID: "INT3474"         |  ipu-    |   clock-frequency: 19200000|
  |   _STA: 0xF               |  bridge  |   rotation: 0              |
  |   SSDB buffer:             | ------> |   orientation: BACK        |
  |     link = 2               |          |   port@0:                  |
  |     lanes = 2              |          |     endpoint@0:            |
  |     mclkspeed = 19200000   |          |       bus-type: CSI2-DPHY  |
  |     degree = 0             |          |       data-lanes: <1 2>    |
  |                            |          |       remote-endpoint:     |
  |   _PLD: panel=BACK         |          |         -> IPU port@2 ep@0 |
  +----------------------------+          +----------------------------+

  After bridge creates software nodes:
  - Ubuntu: V4L2 async notifier binds sensor driver to CSI-2 port
  - Android: HAL reads sensor config + bridge fwnode for pipeline setup
```

---

## 4. 使用者空間框架比較

### 4.1 Android 相機框架

Android 相機框架是一個深度分層的架構，透過 AIDL（Android Interface Definition Language）IPC 強制執行嚴格的介面邊界：

```
  Android Camera Framework Layers:

  +------------------------------------------------------------------+
  |  Layer 1: Application                                              |
  |  android.hardware.camera2.CameraManager                            |
  |  android.hardware.camera2.CameraCaptureSession                     |
  |  android.hardware.camera2.CaptureRequest / CaptureResult           |
  +------------------------------+-----------------------------------+
                                 | Binder IPC
  +------------------------------v-----------------------------------+
  |  Layer 2: CameraService (frameworks/av/services/camera/)           |
  |                                                                    |
  |  CameraProviderManager:                                            |
  |    +-- Discovers HAL providers via AIDL ServiceManager             |
  |    +-- "android.hardware.camera.provider.ICameraProvider/internal/0"|
  |    |   (Google Desktop HAL - USB cameras)                          |
  |    +-- "android.hardware.camera.provider.ICameraProvider/intel/0"  |
  |        (Intel IPU7 HAL - MIPI cameras)                             |
  |                                                                    |
  |  Camera3Device:                                                    |
  |    +-- State machine: CLOSED -> UNCONFIGURED -> CONFIGURED         |
  |    |                  -> ACTIVE -> CLOSED                          |
  |    +-- processCaptureRequest() / processCaptureResult()            |
  +------------------------------+-----------------------------------+
                                 | AIDL IPC
  +------------------------------v-----------------------------------+
  |  Layer 3: Camera HAL Implementations                               |
  |                                                                    |
  |  +---------------------------+  +------------------------------+  |
  |  | Google Desktop HAL (Rust) |  | Intel IPU7 HAL (C++)         |  |
  |  |                           |  |                              |  |
  |  | - USB/UVC cameras only    |  | - MIPI CSI-2 cameras         |  |
  |  | - V4L2 direct capture     |  | - Full ISP pipeline          |  |
  |  | - MJPEG decode            |  | - CaptureUnit (ISYS)         |  |
  |  | - No ISP needed           |  | - ProcessingUnit (PSYS)      |  |
  |  | - No 3A algorithms        |  | - AiqUnit (3A / Intel CCA)   |  |
  |  |                           |  | - RequestThread (orchestrate)|  |
  |  | hal_usb/src/              |  | - IPA Client/Server          |  |
  |  |   v4l2_device.rs          |  | - CameraContext (state)      |  |
  |  |   uvc_device.rs           |  |                              |  |
  |  |   session/worker.rs       |  | vendor/intel/camera/hal/ipu7/|  |
  |  +---------------------------+  +------------------------------+  |
  +------------------------------------------------------------------+
```

### 4.2 Ubuntu/Linux 相機框架

Ubuntu 相機框架明顯更為精簡，應用程式與核心之間的抽象層較少：

```
  Ubuntu Camera Framework Layers:

  +------------------------------------------------------------------+
  |  Layer 1: Application                                              |
  |  GNOME Cheese, Firefox WebRTC, OBS Studio                         |
  |  GStreamer: pipewiresrc ! videoconvert ! autovideosink             |
  +------------------------------+-----------------------------------+
                                 | PipeWire protocol (Unix socket)
  +------------------------------v-----------------------------------+
  |  Layer 2: PipeWire                                                 |
  |                                                                    |
  |  PipeWire Daemon:                                                  |
  |    +-- SPA libcamera plugin (libspa-libcamera)                     |
  |    |   +-- Bridges libcamera cameras to PipeWire graph             |
  |    |   +-- Enumerates cameras via CameraManager                    |
  |    |   +-- Creates PipeWire node per camera                        |
  |    |   +-- Handles buffer sharing via dmabuf/memfd                 |
  |    |                                                               |
  |  WirePlumber (Session Manager):                                    |
  |    +-- Manages permissions (XDG Camera Portal)                     |
  |    +-- Routes camera streams to requesting applications            |
  +------------------------------+-----------------------------------+
                                 | libcamera C++ API
  +------------------------------v-----------------------------------+
  |  Layer 3: libcamera                                                |
  |                                                                    |
  |  CameraManager:                                                    |
  |    +-- DeviceEnumerator scans /dev/media* devices                  |
  |    +-- Matches devices to PipelineHandlers                         |
  |                                                                    |
  |  SimplePipelineHandler (for IPU6 ISYS-only):                       |
  |    +-- match(): Look for media device with sensor entities         |
  |    +-- Trace path: sensor -> CSI-2 subdev -> video capture node    |
  |    +-- configure(): Set formats along entire pipeline              |
  |    +-- start()/stop(): VIDIOC_STREAMON/OFF                         |
  |                                                                    |
  |  SoftwareISP (when swIspEnabled=true):                             |
  |    +-- CPU-based Bayer demosaicing                                 |
  |    +-- Basic color correction                                      |
  |    +-- Output: NV12 or RGB from raw Bayer input                    |
  |                                                                    |
  |  IPA (Image Processing Algorithms):                                |
  |    +-- Soft IPA module (for SoftwareISP)                           |
  |    +-- AE (Auto Exposure): basic metering                          |
  |    +-- AWB (Auto White Balance): gray world algorithm              |
  |    +-- NO Auto Focus support                                       |
  +------------------------------------------------------------------+
```

### 4.3 請求/擷取模型比較

```
  Android Request Model:                   Ubuntu Request Model:

  App creates CaptureRequest               App creates libcamera::Request
    |                                        |
    | Binder IPC                             | direct call
    v                                        v
  Camera3Device::processCaptureRequest()   Camera::queueRequest(Request*)
    |                                        |
    | AIDL IPC                               | V4L2 ioctl
    v                                        v
  HAL: RequestThread::processRequest()     PipelineHandler::
    |                                        queueRequestDevice()
    | Event-driven orchestration             |
    |   - Wait for NEW_SOF event             | V4L2VideoDevice::
    |   - Run 3A via IPA Client/Server       |   queueBuffer()
    |   - Queue buffer to CaptureUnit        |   VIDIOC_QBUF
    |   - Wait for ISYS frame                |
    |   - Submit to ProcessingUnit (PSYS)    |
    v                                        v
  ICameraDeviceCallback::                  Request::complete() signal
    processCaptureResult()                   |
    notify(SHUTTER)                          | PipeWire buffer delivery
    |                                        v
    | Binder callback                      Application receives frame
    v
  App: onCaptureCompleted()

  Key differences:
  - Android: 18-step pipeline with 3A, ISYS, and PSYS stages
  - Ubuntu:  Direct V4L2 capture -> software ISP -> done
  - Android: Event-driven with SOF/STATS/FRAME triggers
  - Ubuntu:  Simple poll/dequeue model
```

### 4.4 Intel IPU7 HAL 內部架構（僅限 Android）

Intel IPU7 HAL 是 Android 獨有的最複雜元件。它協調原始擷取、硬體 ISP 處理及 3A 演算法：

```
  Intel IPU7 HAL Internal Architecture:

  +------------------------------------------------------------------+
  |  RequestThread (event loop)                                        |
  |  Events: NEW_REQUEST, NEW_FRAME, NEW_STATS, NEW_SOF               |
  |                                                                    |
  |  threadLoop():                                                     |
  |    Wait for trigger event                                          |
  |    fetchNextRequest()                                              |
  |    handleRequest():                                                |
  |      1. Get DataContext from CameraContext                         |
  |      2. m3AControl->run3A() via AiqUnit                            |
  |      3. Queue buffer to CaptureUnit                                |
  |      4. Track sequence numbers                                     |
  +----------+------------------+------------------+------------------+
             |                  |                  |
  +----------v-------+  +------v---------+  +-----v--------------+
  |  CaptureUnit     |  | ProcessingUnit |  | AiqUnit            |
  |  (ISYS)          |  | (PSYS)         |  | (3A Engine)        |
  |                  |  |                |  |                    |
  | StreamSource     |  | IProcessingUnit|  | AiqEngine          |
  | DeviceBase       |  | PipeManager    |  | Intel CCA library  |
  | V4L2 capture     |  | PSysDevice     |  | (AE, AF, AWB)      |
  |                  |  | Graph model    |  | SensorHwCtrl       |
  | States:          |  |                |  | LensHw (VCM)       |
  | UNINIT->INIT     |  | Raw frame in   |  |                    |
  | ->CONFIGURE      |  | YUV/JPEG out   |  | CCA init requires: |
  | ->START->STOP    |  |                |  | - AIQB tuning data |
  +------------------+  +----------------+  | - NVM calibration  |
                                            | - CMC color matrix |
                                            +--------------------+

  Buffer flow through CaptureUnit:
  +------------------+
  | mAllocatedBuffers| (allocated pool)
  +--------+---------+
           | addPending()
  +--------v---------+
  | mPendingBuffers  | (ready to queue)
  +--------+---------+
           | queueBuffer() / VIDIOC_QBUF
  +--------v---------+
  | mBuffersInDevice | (in kernel driver)
  +--------+---------+
           | dequeueBuffer() / VIDIOC_DQBUF
  +--------v---------+
  | Consumers        | (ProcessingUnit)
  +------------------+
```

---

## 5. 影像處理管線比較

### 5.1 Android ISP 管線（硬體 PSYS）

在 Android 上，IPU7 硬體 ISP 將原始 Bayer 影格處理為量產品質的 YUV 輸出。處理基於圖形模型，以可設定的節點透過連結相連：

```
  Android PSYS Hardware ISP Pipeline:

  Raw Bayer Frame (from ISYS CaptureUnit)
       |
       v
  +------------------------------------------------------------------+
  |  PSYS Processing Graph (hardware accelerated)                      |
  |                                                                    |
  |  +-------------+    +-------------+    +-------------+            |
  |  | Node 0:     |    | Node 1:     |    | Node 2:     |            |
  |  | BLC         |--->| Demosaic    |--->| CCM         |            |
  |  | (Black Level|    | (Bayer to   |    | (Color      |            |
  |  |  Correct)   |    |  RGB)       |    |  Correction |            |
  |  +-------------+    +-------------+    |  Matrix)    |            |
  |                                        +------+------+            |
  |                                               |                    |
  |  +-------------+    +-------------+    +------v------+            |
  |  | Node 5:     |    | Node 4:     |    | Node 3:     |            |
  |  | Output      |<---| Scaling     |<---| NR + Edge   |            |
  |  | Format      |    | (resize)    |    | (Noise Reduc|            |
  |  | Conversion  |    |             |    |  + Sharpen) |            |
  |  +------+------+    +-------------+    +-------------+            |
  |         |                                                          |
  +---------|----------------------------------------------------------+
            |
            v
  YUV/JPEG Output (to CameraStream -> Application)

  ISP parameters controlled by Intel CCA:
  - Exposure/gain from AE
  - Color correction matrix from AWB
  - Gamma curve from tone mapping
  - Noise reduction filter coefficients
  - Edge enhancement kernel
  - Lens shading correction tables
```

### 5.2 Ubuntu ISP 管線（軟體）

在 Ubuntu 上，所有影像處理都在 CPU 上使用 libcamera 的 SoftwareISP 完成：

```
  Ubuntu SoftwareISP Pipeline:

  Raw Bayer Frame (from ISYS V4L2 capture device)
       |
       v
  +------------------------------------------------------------------+
  |  SoftwareISP (CPU-based, libcamera)                                |
  |                                                                    |
  |  Step 1: Debayering                                                |
  |  +--------------------------------------------------------------+  |
  |  | Bayer pattern interpolation (e.g., BGGR -> RGB)               |  |
  |  | Simple bilinear or more advanced algorithms                   |  |
  |  | CPU-intensive: processes every pixel on CPU                   |  |
  |  +--------------------------------------------------------------+  |
  |                                                                    |
  |  Step 2: AWB Correction                                            |
  |  +--------------------------------------------------------------+  |
  |  | Gray-world white balance algorithm                             |  |
  |  | Calculates per-channel gains                                   |  |
  |  | Applies gain correction to RGB channels                        |  |
  |  +--------------------------------------------------------------+  |
  |                                                                    |
  |  Step 3: Color Space Conversion                                    |
  |  +--------------------------------------------------------------+  |
  |  | RGB -> NV12 (YUV 4:2:0) conversion                            |  |
  |  | Or RGB output directly depending on application request        |  |
  |  +--------------------------------------------------------------+  |
  |                                                                    |
  |  NOT included (compared to Android PSYS):                          |
  |  x  No black level correction                                     |
  |  x  No lens shading correction                                    |
  |  x  No hardware noise reduction                                   |
  |  x  No edge enhancement / sharpening                              |
  |  x  No gamma / tone mapping                                       |
  |  x  No advanced color correction matrix                           |
  |  x  No auto focus support                                         |
  +------------------------------------------------------------------+
            |
            v
  NV12/RGB Output (to PipeWire -> Application)
```

### 5.3 處理比較表

```
  +----------------------------+-----------------------+------------------------+
  | ISP Processing Stage       | Android (PSYS HW)     | Ubuntu (SoftwareISP)   |
  +----------------------------+-----------------------+------------------------+
  | Black Level Correction     | Yes (hardware)        | No                     |
  | Lens Shading Correction    | Yes (hardware)        | No                     |
  | Bayer Demosaicing          | Yes (hardware)        | Yes (CPU, basic)       |
  | White Balance Correction   | Yes (CCA-tuned)       | Yes (gray-world)       |
  | Color Correction Matrix    | Yes (per-sensor CMC)  | No                     |
  | Gamma / Tone Mapping       | Yes (hardware LUT)    | No                     |
  | Noise Reduction (spatial)  | Yes (hardware filter) | No                     |
  | Noise Reduction (temporal) | Yes (multi-frame)     | No                     |
  | Edge Enhancement           | Yes (hardware)        | No                     |
  | Scaling / Cropping         | Yes (hardware scaler) | No (app must scale)    |
  | JPEG Encoding              | Yes (hardware)        | No (software, if any)  |
  | Color Space Conversion     | Yes (hardware)        | Yes (CPU)              |
  | HDR Processing             | Yes (multi-exposure)  | No                     |
  +----------------------------+-----------------------+------------------------+
  | Execution                  | Dedicated IPU HW      | CPU cores              |
  | Power Impact               | Low (HW accelerated)  | High (CPU load)        |
  | Typical Latency            | ~5-15ms per frame     | ~5-15ms per frame*     |
  +----------------------------+-----------------------+------------------------+

  * Ubuntu latency is similar at lower resolutions but scales worse at higher
    resolutions due to CPU processing overhead.
```

---

## 6. 緩衝區管理與 DMA 比較

### 6.1 Android 緩衝區架構

Android 使用多層緩衝區管理系統，以 Gralloc 用於應用程式緩衝區，DMA-BUF 用於核心/HAL 互動：

```
  Android Buffer Management:

  +------------------------------------------------------------------+
  |  Application Layer                                                 |
  |  +------------------------------------------------------------+  |
  |  | Gralloc Buffers (Android graphics allocator)                |  |
  |  | - Backed by ION or DMA-BUF heap                            |  |
  |  | - Shared via NativeHandle (fd + metadata)                   |  |
  |  | - Format: YCBCR_420_888, BLOB (JPEG), RAW16                |  |
  |  | - Usage flags: GPU, CAMERA, CPU_READ, etc.                  |  |
  |  +---------------------------+--------------------------------+  |
  +------------------------------------------------------------------+
                                 | importBuffer()
  +------------------------------------------------------------------+
  |  HAL Layer                                                         |
  |  +------------------------------------------------------------+  |
  |  | CameraBuffer (icamera internal representation)               |  |
  |  | - Wraps Gralloc handle or internal allocation                |  |
  |  | - Tracks: DMA address, sequence, port UUID                   |  |
  |  | - CameraStream: maps app stream ID to internal port          |  |
  |  +---------------------------+--------------------------------+  |
  +------------------------------------------------------------------+
                                 | VIDIOC_QBUF / DQBUF
  +------------------------------------------------------------------+
  |  Kernel Layer                                                      |
  |  +------------------------------------------------------------+  |
  |  | VB2 DMA-SG Buffers (videobuf2-dma-sg)                       |  |
  |  | - Scatter-gather list for physical pages                     |  |
  |  | - Mapped through IPU MMU for firmware DMA                    |  |
  |  |                                                              |  |
  |  | buf_init():                                                  |  |
  |  |   sg = vb2_dma_sg_plane_desc(vb, 0)                         |  |
  |  |   ipu6_dma_map_sgtable(adev, sg)                             |  |
  |  |   ivb->dma_addr = sg_dma_address(sg->sgl)  // IOVA          |  |
  |  +---------------------------+--------------------------------+  |
  +------------------------------------------------------------------+
                                 | IOVA address to firmware
  +------------------------------------------------------------------+
  |  ISYS -> PSYS Zero-Copy Path                                       |
  |  +------------------------------------------------------------+  |
  |  | ISYS captures raw frame to DMA-BUF buffer                    |  |
  |  | Same DMA-BUF fd passed to PSYS as input terminal              |  |
  |  | PSysDevice::registerBuffer(TerminalBuffer{fd, ...})           |  |
  |  | --> NO memory copy between ISYS capture and PSYS processing   |  |
  |  +------------------------------------------------------------+  |
  +------------------------------------------------------------------+
```

### 6.2 Ubuntu 緩衝區架構

Ubuntu 使用標準 V4L2 緩衝區管理搭配 libcamera 的 FrameBuffer 抽象。值得注意的是，在 SoftwareISP 路徑中有一個 CPU 複製步驟：

```
  Ubuntu Buffer Management:

  +------------------------------------------------------------------+
  |  Application Layer                                                 |
  |  +------------------------------------------------------------+  |
  |  | PipeWire Buffer                                              |  |
  |  | - Delivered via dmabuf or memfd (shared memory)              |  |
  |  | - PipeWire SPA plugin manages buffer lifecycle               |  |
  |  | - Format: NV12, RGB (after SoftwareISP processing)           |  |
  |  +---------------------------+--------------------------------+  |
  +------------------------------------------------------------------+
                                 | PipeWire protocol
  +------------------------------------------------------------------+
  |  libcamera Layer                                                   |
  |  +------------------------------------------------------------+  |
  |  | FrameBuffer (libcamera internal)                             |  |
  |  | - Wraps V4L2 buffers or DMA-BUF                              |  |
  |  | - Plane descriptors: fd, offset, length                       |  |
  |  | - Request: groups buffers with control metadata               |  |
  |  +---------------------------+--------------------------------+  |
  +------------------------------------------------------------------+
                                 | VIDIOC_QBUF / DQBUF
  +------------------------------------------------------------------+
  |  Kernel Layer                                                      |
  |  +------------------------------------------------------------+  |
  |  | VB2 DMA-SG Buffers (videobuf2-dma-sg)                       |  |
  |  | - Same VB2 framework as Android (shared kernel code)         |  |
  |  | - Scatter-gather DMA through IPU MMU                         |  |
  |  | - No PSYS device node -- data goes directly to userspace     |  |
  |  +---------------------------+--------------------------------+  |
  +------------------------------------------------------------------+
                                 | raw Bayer frame in user memory
  +------------------------------------------------------------------+
  |  SoftwareISP Processing (CPU copy required)                        |
  |  +------------------------------------------------------------+  |
  |  | Raw Bayer buffer from V4L2 capture device                    |  |
  |  | CPU reads raw pixels, processes (debayer + AWB)              |  |
  |  | CPU writes processed NV12/RGB to output buffer               |  |
  |  | --> CPU COPY between input and output buffers                 |  |
  |  | --> No DMA-BUF zero-copy path (unlike Android PSYS)           |  |
  |  +------------------------------------------------------------+  |
  +------------------------------------------------------------------+
```

### 6.3 緩衝區路徑比較

```
  +----------------------------+-------------------------------+-------------------------------+
  | Buffer Stage               | Android                       | Ubuntu                        |
  +----------------------------+-------------------------------+-------------------------------+
  | App buffer allocation      | Gralloc (ION/DMA-BUF heap)    | V4L2 MMAP or DMA-BUF export   |
  | App buffer format          | YCBCR_420_888, BLOB, RAW16    | NV12, RGB (post-SoftISP)      |
  | HAL internal buffer        | CameraBuffer (wraps Gralloc)  | FrameBuffer (wraps V4L2)      |
  | Kernel buffer type         | VB2 DMA-SG                    | VB2 DMA-SG                    |
  | DMA mapping                | IPU MMU (IOVA)                | IPU MMU (IOVA)                |
  | ISYS -> ISP transfer       | Zero-copy DMA-BUF             | CPU copy (SoftwareISP)        |
  | ISP -> App transfer        | Zero-copy (same Gralloc buf)  | CPU copy to output buffer     |
  | Total memory copies        | 0 (all DMA/zero-copy)         | 1-2 (CPU copies in SoftISP)   |
  +----------------------------+-------------------------------+-------------------------------+
```

---

## 7. 3A 演算法比較

### 7.1 Android 3A 架構（Intel CCA）

Android 使用 Intel CCA（Camera Control Algorithm）函式庫，這是一個提供量產等級 3A 處理的專有實作：

```
  Android 3A Architecture:

  +------------------------------------------------------------------+
  |  AiqUnit (3A Control)                                              |
  |  vendor/intel/camera/hal/ipu7/src/3a/AiqUnit.h                     |
  |                                                                    |
  |  States: NOT_INIT -> INIT -> CONFIGURED -> START -> STOP           |
  |                                                                    |
  |  run3A(ccaId, applyingSeq, frameNumber, *effectSeq):               |
  |    +-- IPAClient::runAec()   -> Auto Exposure Control              |
  |    |   +-- Exposure time, analog gain, digital gain, flash         |
  |    |                                                               |
  |    +-- IPAClient::runAiq()   -> Auto White Balance + ISP params    |
  |    |   +-- Color correction matrix, gamma, tone mapping            |
  |    |                                                               |
  |    +-- IPAClient::runAic()   -> ISP Configuration                  |
  |        +-- PAL binary for PSYS (filter coefficients, LUTs)         |
  +------------------------------------------------------------------+

  IPA Client/Server Architecture:

  +----------------------------+     +-------------------------------+
  |  HAL Process               |     |  IPA Process (sandboxed)      |
  |                            |     |                               |
  |  IPAClient (singleton)     |     |  IPAServer                    |
  |  +-- initCca()             | <-- |  +-- CcaWorker instances      |
  |  +-- runAec()              | --> |  |   (per camera + tuning)    |
  |  +-- runAiq()              |     |  |                            |
  |  +-- configAic()           | Shared |  +-- Intel CCA library     |
  |  +-- runAic()              | Memory |  |   (libcca.so)           |
  |  +-- decodeStats()         |     |  |   +-- AE algorithm        |
  |  +-- setStats()            |     |  |   +-- AF algorithm        |
  |  +-- updateTuning()        |     |  |   +-- AWB algorithm       |
  |  +-- getCmc()              |     |  |   +-- ISP tuning          |
  |  +-- getMkn()              |     |  +-- Per-sensor tuning data  |
  |  +-- getAiqd()             |     |      (AIQB, NVM, CMC)        |
  +----------------------------+     +-------------------------------+

  3A Processing Flow:

  Frame N arrives from sensor
    |
    v
  ISYS captures raw frame + embedded statistics
    |
    v
  Statistics decoded: IPAClient::decodeStats()
    |-- IPAServer::sendRequest(DECODE_STATS)
    |-- CcaWorker processes statistics
    v
  AiqUnit::run3A(ccaId, applyingSeq, frameNumber)
    |
    +-- runAec() -> exposure, gain, digital gain
    +-- runAiq() -> CCM, gamma, tone map
    +-- runAic() -> PSYS node parameters
    |
    v
  Results stored in AiqResultStorage
    |
    v
  ProcessingUnit reads results for frame N+latency
    |
    v
  PSYS processes raw -> YUV with 3A-tuned parameters
```

### 7.2 Ubuntu 3A 架構（libcamera Soft IPA）

Ubuntu 使用 libcamera 的軟體 IPA 模組，僅提供基本的自動曝光和自動白平衡：

```
  Ubuntu 3A Architecture:

  +------------------------------------------------------------------+
  |  libcamera IPA Module (soft)                                       |
  |  src/ipa/simple/                                                   |
  |                                                                    |
  |  Runs in-process with libcamera (not sandboxed for SoftwareISP)    |
  |                                                                    |
  |  Algorithms:                                                       |
  |                                                                    |
  |  +--------------------------------------------------------------+  |
  |  |  Auto Exposure (AE)                                           |  |
  |  |  - Calculates mean luminance from captured frame              |  |
  |  |  - Adjusts sensor exposure time and gain via V4L2 controls    |  |
  |  |  - Simple target-brightness convergence loop                  |  |
  |  +--------------------------------------------------------------+  |
  |                                                                    |
  |  +--------------------------------------------------------------+  |
  |  |  Auto White Balance (AWB)                                     |  |
  |  |  - Gray-world algorithm: assumes average scene is gray        |  |
  |  |  - Calculates per-channel gains (R, G, B)                     |  |
  |  |  - Applied during SoftwareISP debayering                      |  |
  |  +--------------------------------------------------------------+  |
  |                                                                    |
  |  NOT available:                                                    |
  |  x  Auto Focus (AF) -- no AF algorithm                            |
  |  x  Advanced AWB (illuminant estimation)                           |
  |  x  Tone mapping / gamma control                                  |
  |  x  Noise profiling / sensor characterization                      |
  |  x  Per-sensor tuning data                                         |
  |  x  ISP parameter generation                                      |
  +------------------------------------------------------------------+
```

### 7.3 3A 功能比較

```
  +-------------------------------+--------------------+--------------------+
  | 3A Feature                    | Android (CCA)      | Ubuntu (Soft IPA)  |
  +-------------------------------+--------------------+--------------------+
  | Auto Exposure (AE)            | Yes - advanced     | Yes - basic        |
  |   Metering modes              | Center/Spot/Matrix | Mean luminance     |
  |   Convergence speed           | Fast (tuned)       | Moderate           |
  |   Flash metering              | Yes                | No                 |
  +-------------------------------+--------------------+--------------------+
  | Auto Focus (AF)               | Yes                | No                 |
  |   CDAF (Contrast Detect AF)   | Yes                | No                 |
  |   PDAF (Phase Detect AF)      | Sensor-dependent   | No                 |
  |   Continuous AF               | Yes                | No                 |
  |   Touch-to-focus              | Yes (via regions)  | No                 |
  +-------------------------------+--------------------+--------------------+
  | Auto White Balance (AWB)      | Yes - advanced     | Yes - basic        |
  |   Illuminant estimation       | Yes (multi-illum)  | No (gray-world)    |
  |   Per-sensor calibration      | Yes (CMC data)     | No                 |
  |   Manual white balance        | Yes                | No                 |
  +-------------------------------+--------------------+--------------------+
  | ISP Tuning                    | Yes                | No                 |
  |   Per-sensor tuning files     | AIQB binary        | None               |
  |   Calibration data            | NVM / CMC          | None               |
  |   Runtime tuning updates      | Yes (updateTuning) | No                 |
  +-------------------------------+--------------------+--------------------+
  | Statistics Decode              | Yes (HW stats)     | No (frame-based)   |
  |   Grid-based statistics       | Yes (PSYS output)  | No                 |
  |   Histogram analysis          | Yes                | No                 |
  +-------------------------------+--------------------+--------------------+
  | Execution Model               | IPA Client/Server  | In-process         |
  |   Process isolation           | Yes (sandboxed)    | No                 |
  |   Shared memory for params    | Yes                | N/A                |
  +-------------------------------+--------------------+--------------------+
```

---

## 8. 應用程式整合比較

### 8.1 Android 應用程式整合

```
  Android Camera Application Integration:

  +------------------------------------------------------------------+
  |  Application Code (Java/Kotlin):                                   |
  |                                                                    |
  |  // 1. Get CameraManager                                          |
  |  val cameraManager = getSystemService(CAMERA_SERVICE)              |
  |                                                                    |
  |  // 2. Open camera device                                         |
  |  cameraManager.openCamera(cameraId, stateCallback, handler)        |
  |                                                                    |
  |  // 3. Create capture session                                     |
  |  cameraDevice.createCaptureSession(outputs, sessionCallback, ...)  |
  |                                                                    |
  |  // 4. Build capture request                                      |
  |  val request = cameraDevice.createCaptureRequest(TEMPLATE_PREVIEW) |
  |  request.addTarget(previewSurface)                                 |
  |  request.set(CaptureRequest.CONTROL_AF_MODE, AF_MODE_CONTINUOUS)   |
  |  request.set(CaptureRequest.CONTROL_AE_MODE, AE_MODE_ON)          |
  |                                                                    |
  |  // 5. Submit repeating request                                    |
  |  captureSession.setRepeatingRequest(request, captureCallback, ...)  |
  |                                                                    |
  |  // 6. Receive results                                             |
  |  captureCallback.onCaptureCompleted(session, request, result)      |
  |  // result contains: AE state, AF state, AWB state,                |
  |  //   exposure time, sensitivity, focus distance, etc.             |
  +------------------------------------------------------------------+

  Supported stream combinations on PTL:
  +---------------------+----------+-----------+
  | Stream Configuration | Format   | Max Size  |
  +---------------------+----------+-----------+
  | Preview              | NV21     | 1920x1080 |
  | Still Capture        | JPEG     | 4672x3416 |
  | Video Recording      | NV21     | 1920x1080 |
  | RAW Capture          | RAW16    | Sensor max |
  | Preview + Still      | NV21+JPEG| Sensor max |
  | Preview + Video      | NV21+NV21| 1920x1080 |
  +---------------------+----------+-----------+
```

### 8.2 Ubuntu 應用程式整合

```
  Ubuntu Camera Application Integration:

  Path 1: PipeWire (recommended for desktop applications)
  +------------------------------------------------------------------+
  |  Application uses PipeWire API or GStreamer pipewiresrc:            |
  |                                                                    |
  |  # GStreamer pipeline (most common)                                |
  |  gst-launch-1.0 pipewiresrc ! videoconvert ! autovideosink         |
  |                                                                    |
  |  # Camera preview via libcamerasrc                                 |
  |  gst-launch-1.0 libcamerasrc ! videoconvert ! autovideosink        |
  |                                                                    |
  |  # Video recording                                                |
  |  gst-launch-1.0 libcamerasrc ! video/x-raw,width=1920,height=1080 |
  |    ! videoconvert ! x264enc ! mp4mux ! filesink location=out.mp4   |
  +------------------------------------------------------------------+

  Path 2: libcamera API (for custom camera applications)
  +------------------------------------------------------------------+
  |  Application Code (C++):                                           |
  |                                                                    |
  |  // 1. Create and start CameraManager                              |
  |  CameraManager cm;                                                 |
  |  cm.start();                                                       |
  |                                                                    |
  |  // 2. Get camera                                                  |
  |  auto camera = cm.cameras()[0];                                    |
  |  camera->acquire();                                                |
  |                                                                    |
  |  // 3. Generate and apply configuration                            |
  |  auto config = camera->generateConfiguration({StreamRole::VideoRecording}); |
  |  camera->configure(config.get());                                  |
  |                                                                    |
  |  // 4. Allocate buffers                                            |
  |  FrameBufferAllocator allocator(camera);                            |
  |  allocator.allocate(config->at(0).stream());                       |
  |                                                                    |
  |  // 5. Queue requests                                              |
  |  auto request = camera->createRequest();                           |
  |  request->addBuffer(stream, buffers[0].get());                     |
  |  camera->queueRequest(request.get());                              |
  |                                                                    |
  |  // 6. Handle completed requests                                   |
  |  camera->requestCompleted.connect(handleRequestComplete);          |
  +------------------------------------------------------------------+

  Path 3: Direct V4L2 (legacy, raw Bayer only)
  +------------------------------------------------------------------+
  |  # Direct V4L2 capture - no ISP, raw Bayer output                  |
  |  v4l2-ctl --device=/dev/video0 --stream-mmap --stream-count=10     |
  |    --stream-to=raw_frames.bin                                      |
  |                                                                    |
  |  NOTE: This gives raw Bayer data. Application must handle           |
  |  demosaicing, white balance, and color correction itself.          |
  +------------------------------------------------------------------+

  Permission model:
  +------------------------------------------------------------------+
  |  XDG Camera Portal Flow:                                           |
  |                                                                    |
  |  Application (Flatpak/Snap sandbox)                                |
  |    --> Request camera access via xdg-desktop-portal                |
  |    --> Desktop shows permission dialog                             |
  |    --> If granted: PipeWire creates camera node access              |
  |    --> WirePlumber routes camera to application                     |
  |                                                                    |
  |  Native applications can access /dev/video* directly               |
  |  (if user has appropriate group permissions)                       |
  +------------------------------------------------------------------+
```

### 8.3 應用程式 API 功能比較

```
  +-------------------------------+-----------------------+------------------------+
  | Application Feature           | Android (Camera2 API) | Ubuntu (PipeWire/      |
  |                               |                       |  libcamera)            |
  +-------------------------------+-----------------------+------------------------+
  | Camera enumeration             | getCameraIdList()     | CameraManager::cameras |
  | Camera characteristics        | getCameraCharacteristics | Camera::properties  |
  | Stream configuration          | createCaptureSession  | generateConfiguration  |
  | Per-frame controls            | CaptureRequest        | Request::controls      |
  | Per-frame results              | CaptureResult         | Request::metadata      |
  | AE control                    | CONTROL_AE_MODE       | ExposureTime control   |
  | AF control                    | CONTROL_AF_MODE       | Not available          |
  | AWB control                   | CONTROL_AWB_MODE      | ColourGains control    |
  | Zoom                          | SCALER_CROP_REGION    | Not available          |
  | Flash control                 | FLASH_MODE            | Not available          |
  | Face detection                | STATISTICS_FACE_DETECT| Not available          |
  | RAW capture                   | RAW_SENSOR format     | Direct V4L2 (raw Bayer)|
  | JPEG capture                  | JPEG format           | Software encoding      |
  | Multi-camera                  | Physical camera IDs   | Multiple Camera objects|
  | Concurrent streams            | Up to 3 simultaneous  | Typically 1            |
  | Reprocessing                  | INPUT stream type     | Not available          |
  +-------------------------------+-----------------------+------------------------+
```

---

## 9. 感測器與硬體支援

### 9.1 感測器探索機制

兩個平台都使用 ipu-bridge 模組進行基於 ACPI 的感測器探索。此橋接器從 ACPI 讀取 SSDB（Sensor Specific Data Block）緩衝區並建立 V4L2 fwnode 屬性。

```
  Supported Sensors (ipu-bridge table):

  +-----------+-----------+--------+------------------+---------+---------+
  | ACPI HID  | Sensor    | Lanes  | Link Frequency   | Android | Ubuntu  |
  +-----------+-----------+--------+------------------+---------+---------+
  | INT3474   | OV2740    | 1-2    | 180 MHz          | Yes     | Yes     |
  | INT33BE   | OV5693    | 1-4    | 419.2 MHz        | Yes     | Yes     |
  | INT3479   | OV5670    | 1-4    | 422.4 MHz        | Yes     | Yes     |
  | INT347A   | OV8865    | 1-4    | 360 MHz          | Yes     | Yes     |
  | OVTI01A0  | OV01A10   | 1      | 400 MHz          | Yes     | Yes     |
  | OVTI02C1  | OV02C10   | 1-2    | 400 MHz          | Yes     | Yes     |
  | OVTI08F4  | OV08X40   | 1-4    | 400/749/800 MHz  | Yes     | Yes     |
  | OVTI13B1  | OV13B10   | 1-4    | 560 MHz          | Yes     | Yes     |
  | OVTI8856  | OV8856    | 1-4    | 180/360/720 MHz  | Yes     | Yes     |
  | HIMX11B1  | HM11B1    | 1      | 384 MHz          | Yes     | Yes     |
  | HIMX2170  | HM2170    | 1      | 384 MHz          | Yes     | Yes     |
  | HIMX2172  | HM2172    | 1      | 384 MHz          | Yes     | Yes     |
  +-----------+-----------+--------+------------------+---------+---------+

  NOTE: ipu-bridge sensor detection is shared between platforms. However,
  individual sensor I2C drivers must be present in the kernel. Ubuntu's
  mainline kernel has all standard OmniVision and Himax drivers. Android
  may bundle additional sensor drivers in the vendor tree.
```

### 9.2 CSI-2 PHY 支援

```
  CSI-2 PHY Variants:

  +------------+----------------------+--------------+--------------------+
  | PHY Type   | Source File          | Platforms    | Both Platforms?    |
  +------------+----------------------+--------------+--------------------+
  | MCD PHY    | ipu6-isys-mcd-phy.c  | Tiger Lake   | Yes (shared code)  |
  | JSL PHY    | ipu6-isys-jsl-phy.c  | Jasper Lake  | Yes (shared code)  |
  | DWC PHY    | ipu6-isys-dwc-phy.c  | Meteor Lake+ | Yes (shared code)  |
  +------------+----------------------+--------------+--------------------+

  The PHY implementation is selected at runtime via function pointer:

  struct ipu6_isys {
      int (*phy_set_power)(struct ipu6_isys *isys,
                           struct ipu6_isys_csi2_config *cfg,
                           const struct ipu6_isys_csi2_timing *timing,
                           bool on);
  };

  IPU7 (Panther Lake) is expected to use a new or modified PHY variant.
```

### 9.3 Media Controller 拓撲

兩個平台為 ISYS 建立相同的 media controller 圖形拓撲：

```
  Media Controller Graph (Shared Between Platforms):

  media-ctl -p -d /dev/media0

  +----------------+       +------------------------------+
  | OV2740 sensor  |       | Intel IPU6 ISYS CSI-2 0      |
  | (V4L2 subdev)  |       | (V4L2 subdevice)             |
  |                |       |                              |
  | pad 0 (src) ---+------>| pad 0 (sink)                 |
  +----------------+       |                              |
                            | pad 1 (src) --> /dev/video0  |
                            | pad 2 (src) --> /dev/video1  |
                            +------------------------------+

  +----------------+       +------------------------------+
  | OV8856 sensor  |       | Intel IPU6 ISYS CSI-2 2      |
  | (V4L2 subdev)  |       | (V4L2 subdevice)             |
  |                |       |                              |
  | pad 0 (src) ---+------>| pad 0 (sink)                 |
  +----------------+       |                              |
                            | pad 1 (src) --> /dev/video4  |
                            | pad 2 (src) --> /dev/video5  |
                            +------------------------------+

  Device nodes created on both platforms:
    /dev/media0         -- Media controller (graph navigation)
    /dev/video0..N      -- V4L2 capture devices (VIDIOC_STREAMON)
    /dev/v4l-subdev0..N -- Sensor and CSI-2 subdevices

  Android additionally creates:
    /dev/ipu-psys0      -- PSYS processing device (Android only)
```

### 9.4 支援的像素格式（ISYS 擷取）

ISYS 驅動程式在兩個平台上支援相同的像素格式，因為核心程式碼是共用的：

```
  ISYS Capture Formats (Both Platforms):

  +------------------------+------+--------+--------------------+
  | V4L2 Format            | BPP  | Bits   | Description        |
  +------------------------+------+--------+--------------------+
  | V4L2_PIX_FMT_SBGGR8   |  8   |  8     | Bayer BGGR 8-bit   |
  | V4L2_PIX_FMT_SGBRG8   |  8   |  8     | Bayer GBRG 8-bit   |
  | V4L2_PIX_FMT_SGRBG8   |  8   |  8     | Bayer GRBG 8-bit   |
  | V4L2_PIX_FMT_SRGGB8   |  8   |  8     | Bayer RGGB 8-bit   |
  | V4L2_PIX_FMT_SBGGR10  | 16   | 10     | Bayer BGGR 10-bit  |
  | V4L2_PIX_FMT_SGBRG10  | 16   | 10     | Bayer GBRG 10-bit  |
  | V4L2_PIX_FMT_SGRBG10  | 16   | 10     | Bayer GRBG 10-bit  |
  | V4L2_PIX_FMT_SRGGB10  | 16   | 10     | Bayer RGGB 10-bit  |
  | V4L2_PIX_FMT_SBGGR12  | 16   | 12     | Bayer BGGR 12-bit  |
  | V4L2_PIX_FMT_SGBRG12  | 16   | 12     | Bayer GBRG 12-bit  |
  | V4L2_PIX_FMT_SGRBG12  | 16   | 12     | Bayer GRBG 12-bit  |
  | V4L2_PIX_FMT_SRGGB12  | 16   | 12     | Bayer RGGB 12-bit  |
  | V4L2_PIX_FMT_SBGGR10P | 10   | 10     | Bayer BGGR packed  |
  | V4L2_PIX_FMT_SBGGR12P | 12   | 12     | Bayer BGGR packed  |
  | V4L2_PIX_FMT_UYVY     | 16   | 16     | YUV 4:2:2 UYVY     |
  | V4L2_PIX_FMT_YUYV     | 16   | 16     | YUV 4:2:2 YUYV     |
  | V4L2_PIX_FMT_RGB565   | 16   | 16     | RGB 5:6:5           |
  | V4L2_PIX_FMT_BGR24    | 24   | 24     | BGR 8:8:8           |
  +------------------------+------+--------+--------------------+

  Resolution range: 2x1 to 4672x3416 (step: 2x2)

  Key difference in output formats:
  - Android: PSYS converts raw Bayer -> NV21/JPEG/YUV (hardware)
  - Ubuntu:  SoftwareISP converts raw Bayer -> NV12/RGB (CPU)
  - Ubuntu (direct V4L2): raw Bayer output only
```

---

## 10. 效能與功能矩陣

### 10.1 完整功能矩陣

```
  +-------------------------------------+-------------------+--------------------+
  | Feature                             | Android (ALOS)    | Ubuntu/Linux       |
  +-------------------------------------+-------------------+--------------------+
  |                                     |                   |                    |
  | --- CAPTURE ---                     |                   |                    |
  | Raw Bayer capture (ISYS)            | Yes               | Yes                |
  | YUV output (processed)             | Yes (PSYS HW)     | Yes (SoftISP CPU)  |
  | JPEG capture                        | Yes (PSYS HW)     | Software only      |
  | RAW16 output to app                | Yes               | Yes (via V4L2)     |
  | Multi-stream capture                | Yes (up to 3)     | Typically 1        |
  | Virtual channel support             | Yes (VC 0-15)     | Yes (VC 0-15)     |
  | Metadata capture                    | Yes               | Yes                |
  |                                     |                   |                    |
  | --- IMAGE PROCESSING ---            |                   |                    |
  | Hardware ISP (PSYS)                 | Yes               | No                 |
  | Software ISP                        | No (uses HW)      | Yes (SoftwareISP)  |
  | Bayer demosaicing                   | HW (PSYS)         | CPU (SoftISP)      |
  | Black level correction              | Yes               | No                 |
  | Lens shading correction             | Yes               | No                 |
  | Noise reduction (spatial)           | Yes (HW)          | No                 |
  | Noise reduction (temporal)          | Yes (HW)          | No                 |
  | Edge enhancement                    | Yes (HW)          | No                 |
  | Gamma / tone mapping                | Yes (HW LUT)      | No                 |
  | Color correction matrix             | Yes (per-sensor)  | No                 |
  | HDR processing                      | Yes               | No                 |
  | Hardware scaling / cropping         | Yes (PSYS)        | No                 |
  |                                     |                   |                    |
  | --- 3A ALGORITHMS ---               |                   |                    |
  | Auto Exposure (AE)                  | Yes (CCA)         | Yes (basic)        |
  | Auto Focus (AF)                     | Yes (CCA)         | No                 |
  | Auto White Balance (AWB)            | Yes (CCA)         | Yes (gray-world)   |
  | Flash metering                      | Yes               | No                 |
  | Face detection                      | Yes               | No                 |
  | Statistics decode (HW)              | Yes               | No                 |
  | Per-sensor tuning (AIQB/NVM/CMC)    | Yes               | No                 |
  |                                     |                   |                    |
  | --- KERNEL DRIVER ---               |                   |                    |
  | ISYS driver                         | Yes               | Yes                |
  | PSYS driver                         | Yes               | No                 |
  | ipu-bridge ACPI enumeration         | Yes               | Yes                |
  | Firmware authentication             | Yes (Buttress)    | Yes (Buttress)     |
  | IPU MMU management                  | Yes               | Yes                |
  | iWake power management              | Yes               | Yes                |
  | Runtime PM                          | Yes               | Yes                |
  |                                     |                   |                    |
  | --- APPLICATION ---                 |                   |                    |
  | Camera API                          | Camera2/CameraX   | PipeWire/libcamera |
  | Per-frame control                   | Yes (rich)        | Limited            |
  | Per-frame metadata results          | Yes (rich)        | Limited            |
  | Zoom control                        | Yes               | No                 |
  | Multiple simultaneous streams       | Yes (3)           | No (typically 1)   |
  | Reprocessing                        | Yes               | No                 |
  | Permission model                    | Android perms     | XDG Portal         |
  +-------------------------------------+-------------------+--------------------+
```

### 10.2 效能比較

```
  Performance Characteristics:

  +-------------------------------+--------------------+---------------------+
  | Metric                        | Android (ALOS)     | Ubuntu/Linux        |
  +-------------------------------+--------------------+---------------------+
  | Capture-to-display latency    | 2-3 frames         | 1-2 frames          |
  |   (30fps capture)             | (~66-100ms)        | (~33-66ms)          |
  |                               |                    |                     |
  | ISYS capture latency          | ~1ms (DMA)         | ~1ms (DMA)          |
  | ISP processing latency        | ~5-15ms (HW PSYS)  | ~5-15ms (CPU)       |
  | 3A processing latency         | ~5-10ms (parallel) | ~1-2ms (basic)      |
  | Total per-frame pipeline      | ~11-26ms           | ~7-18ms             |
  |                               |                    |                     |
  | CPU utilization during        | Low (HW ISP)       | High (CPU ISP)      |
  |   1080p capture               | ~5-10% CPU         | ~30-50% CPU         |
  |                               |                    |                     |
  | CPU utilization during        | Low-Medium          | Very High            |
  |   4K capture                  | ~10-15% CPU        | ~60-90% CPU         |
  |                               |                    |                     |
  | Power consumption (camera     | Lower (HW ISP     | Higher (CPU ISP     |
  |   active, 1080p)              |  offload)          |  processing)        |
  |                               |                    |                     |
  | Memory copies per frame       | 0 (zero-copy       | 1-2 (CPU copies     |
  |                               |  DMA-BUF path)     |  in SoftwareISP)    |
  |                               |                    |                     |
  | Max practical resolution      | Sensor max         | Limited by CPU      |
  |   at 30fps                    | (4672x3416)        |  throughput          |
  +-------------------------------+--------------------+---------------------+

  Notes:
  - Android's higher capture-to-display latency is due to the additional PSYS
    processing stage and 3A pipeline depth. However, the resulting image quality
    is significantly better.
  - Ubuntu's lower latency comes at the cost of reduced image quality and
    higher CPU utilization.
  - The CPU utilization figures are approximate and depend on the specific CPU,
    resolution, and frame rate.
```

### 10.3 時序圖比較

```
  Android Frame Pipeline:

  Sensor          ISYS           3A             PSYS           App
    |               |              |              |              |
    | Frame N       |              |              |              |
    +--SOF--------->|              |              |              |
    |               | SOF event    |              |              |
    |               +------------->|              |              |
    |               |              | run3A(N)     |              |
    |               |              | (for N+1)    |              |
    | pixel data    |              |              |              |
    +-------------->|              |              |              |
    |               | DMA to DRAM  |              |              |
    +--EOF--------->|              |              |              |
    |               | PIN_DATA_RDY |              |              |
    |               +----------------------------->|              |
    |               |              |              | Process raw  |
    |               |              |              | -> YUV/JPEG  |
    |               |              |              |              |
    |               |              |              | Task done    |
    |               |              |              +------------->|
    |               |              |              | Result       |


  Ubuntu Frame Pipeline:

  Sensor          ISYS           SoftISP                       App
    |               |              |                              |
    | Frame N       |              |                              |
    +--SOF--------->|              |                              |
    |               |              |                              |
    | pixel data    |              |                              |
    +-------------->|              |                              |
    |               | DMA to DRAM  |                              |
    +--EOF--------->|              |                              |
    |               | PIN_DATA_RDY |                              |
    |               +------------->|                              |
    |               |              | Debayer+AWB                  |
    |               |              | (CPU processing)             |
    |               |              |                              |
    |               |              | Request complete             |
    |               |              +----------------------------->|
    |               |              | NV12 frame                   |

  Key timing differences:
  - Android: 3A runs in parallel during ISYS capture (pipelined)
  - Android: PSYS adds processing stage but uses dedicated hardware
  - Ubuntu:  No 3A pipeline depth, simpler but lower quality
  - Ubuntu:  SoftwareISP CPU time varies with resolution
```

---

## 11. 移植考量

### 11.1 從 Android 移植到 Ubuntu

將相機解決方案從 Android 移植到 Ubuntu 時，工程師必須解決以下差距：

```
  Android -> Ubuntu Porting Challenges:

  +------------------------------------------------------------------+
  |  1. PSYS Replacement                                               |
  |     Android: Full hardware ISP via ipu7-psys.ko + PSysDevice       |
  |     Ubuntu:  Must use SoftwareISP or implement custom pipeline     |
  |                                                                    |
  |     Options:                                                       |
  |     a) Accept SoftwareISP quality (simplest, lowest quality)       |
  |     b) Use out-of-tree ipu6-drivers + ipu6-camera-hal              |
  |        (proprietary, DKMS, requires ipu6-camera-bins)              |
  |     c) Develop custom libcamera pipeline handler with GPU ISP      |
  |        (significant engineering effort)                            |
  +------------------------------------------------------------------+
  |  2. 3A Algorithm Replacement                                       |
  |     Android: Intel CCA (AE, AF, AWB, ISP tuning)                   |
  |     Ubuntu:  libcamera soft IPA (AE, AWB only)                     |
  |                                                                    |
  |     Impact:                                                        |
  |     - No auto-focus support on Ubuntu                              |
  |     - Basic white balance (gray-world vs CCA multi-illuminant)     |
  |     - No per-sensor tuning data (AIQB, NVM, CMC)                   |
  |     - No ISP parameter generation                                  |
  +------------------------------------------------------------------+
  |  3. HAL Interface Translation                                      |
  |     Android: AIDL ICameraProvider / ICameraDeviceSession            |
  |     Ubuntu:  libcamera PipelineHandler                             |
  |                                                                    |
  |     The interface contracts are fundamentally different:           |
  |     - Android: Rich per-frame metadata (100+ keys)                 |
  |     - libcamera: Simpler control/property model                    |
  +------------------------------------------------------------------+
  |  4. Buffer Management Translation                                  |
  |     Android: Gralloc + DMA-BUF + zero-copy ISYS->PSYS              |
  |     Ubuntu:  V4L2 MMAP/DMA-BUF + CPU copy in SoftwareISP          |
  |                                                                    |
  |     Performance impact: Additional memory copies on Ubuntu          |
  +------------------------------------------------------------------+
  |  5. Application API Translation                                    |
  |     Android: Camera2 API (Java/Kotlin, rich controls)              |
  |     Ubuntu:  PipeWire/libcamera/GStreamer (C/C++, basic controls)  |
  |                                                                    |
  |     Many Camera2 features have no Ubuntu equivalent:               |
  |     - AF modes, zoom, face detection, reprocessing                 |
  +------------------------------------------------------------------+
```

### 11.2 從 Ubuntu 移植到 Android

從 Ubuntu 移植到 Android 時，主要挑戰在相反方向：

```
  Ubuntu -> Android Porting Challenges:

  +------------------------------------------------------------------+
  |  1. Camera HAL Implementation                                      |
  |     Ubuntu:  libcamera SimplePipelineHandler handles everything     |
  |     Android: Must implement full ICameraProvider / ICameraDevice    |
  |              / ICameraDeviceSession AIDL interfaces                |
  |                                                                    |
  |     The Intel IPU7 HAL (vendor/intel/camera/hal/ipu7/) provides    |
  |     a reference implementation with 50+ source files.              |
  +------------------------------------------------------------------+
  |  2. PSYS Integration                                               |
  |     Ubuntu:  No PSYS driver or interface                           |
  |     Android: Must integrate ipu7-psys.ko kernel module             |
  |              + PSysDevice userspace interface                       |
  |              + Graph-based processing model                        |
  +------------------------------------------------------------------+
  |  3. CCA Library Integration                                        |
  |     Ubuntu:  No CCA library                                        |
  |     Android: Must integrate Intel CCA with:                        |
  |              - IPA Client/Server architecture                      |
  |              - Shared memory for parameters/statistics              |
  |              - Per-sensor tuning data (AIQB, NVM, CMC)             |
  |              - CcaWorker instances per camera+tuning mode          |
  +------------------------------------------------------------------+
  |  4. Android Framework Integration                                  |
  |     Ubuntu:  N/A                                                   |
  |     Android: Must integrate with CameraService, Camera3Device,     |
  |              Binder IPC, Gralloc buffer management, FMQ            |
  |              (Fast Message Queue), and SELinux policies.           |
  +------------------------------------------------------------------+
```

### 11.3 IPU7 上游時間表

```
  IPU Upstream Status Timeline:

  +--------------------------------------------------------------------+
  |                                                                      |
  |  IPU3 (Skylake/Kaby Lake):                                          |
  |    Upstream since kernel 4.16 (2018)                                 |
  |    Full ISYS + ImgU (PSYS equivalent) support                       |
  |    libcamera IPU3 pipeline handler available                         |
  |                                                                      |
  |  IPU6 (Tiger Lake -> Meteor Lake):                                   |
  |    Upstream ISYS since kernel 6.10 (2024)                            |
  |    PSYS NOT upstream (proprietary algorithms)                        |
  |    libcamera SimplePipelineHandler + SoftwareISP                     |
  |    Out-of-tree: intel/ipu6-drivers + ipu6-camera-hal                 |
  |                                                                      |
  |  IPU7 (Lunar Lake / Panther Lake):                                   |
  |    Entered Linux staging in kernel 6.17 (17,000+ lines)              |
  |    Same ISYS-only approach as IPU6 (no PSYS upstream)                |
  |    Will reuse ipu-bridge ACPI infrastructure                         |
  |    Out-of-tree: intel/ipu7-drivers + ipu7-camera-hal                 |
  |      + ipu7-camera-bins (proprietary libraries)                      |
  |                                                                      |
  |  Pattern: Each IPU generation follows the same upstream model:       |
  |    1. Out-of-tree driver for early platform enablement               |
  |    2. ISYS-only upstream submission (1-2 years after silicon)         |
  |    3. PSYS remains out-of-tree due to proprietary firmware/algos     |
  +--------------------------------------------------------------------+
```

### 11.4 樹外驅動程式路徑（Ubuntu 替代方案）

對於需要比上游 SoftwareISP 更高影像品質的 Ubuntu 使用者，存在一個樹外路徑：

```
  Out-of-Tree Ubuntu Camera Stack:

  +------------------------------------------------------------------+
  |  Application                                                       |
  |       |                                                            |
  |       v                                                            |
  |  icamerasrc (GStreamer plugin)                                     |
  |       |                                                            |
  |       v                                                            |
  |  ipu6-camera-hal / ipu7-camera-hal (Intel Camera HAL)              |
  |  +-- 3A algorithms (proprietary libraries)                         |
  |  +-- ISP parameter tuning files                                    |
  |  +-- PSYS pipeline configuration                                   |
  |       |                                                            |
  |       v                                                            |
  |  ipu6-camera-bins / ipu7-camera-bins (proprietary)                 |
  |  +-- libipa.so (Image Processing Accelerator)                      |
  |  +-- lib3aiq.so (3A Intelligence library)                          |
  |  +-- libia_imaging.so (Intel imaging algorithms)                   |
  |       |                                                            |
  |       v                                                            |
  |  Kernel: ipu6-drivers / ipu7-drivers (DKMS)                        |
  |  +-- ISYS module (similar to upstream)                             |
  |  +-- PSYS module (hardware ISP -- not upstream)                    |
  +------------------------------------------------------------------+

  Trade-offs vs upstream:
  +-------------------+----------------------------+----------------------+
  | Aspect            | Upstream                   | Out-of-Tree          |
  +-------------------+----------------------------+----------------------+
  | Image quality     | Basic (SoftwareISP)        | Full (hardware ISP)  |
  | Kernel updates    | Automatic with distro      | Manual DKMS rebuild  |
  | Maintenance       | Community maintained       | Intel maintained     |
  | License           | GPL (fully open)           | Mixed (proprietary   |
  |                   |                            |  binaries required)  |
  | PipeWire support  | Yes (via libcamera)        | GStreamer only       |
  |                   |                            |  (icamerasrc)        |
  | Sensor support    | Via mainline drivers       | Bundled (18+ sensors)|
  +-------------------+----------------------------+----------------------+
```

---

## 12. 關鍵檔案參考

### 12.1 Android 相機堆疊檔案

```
  Android Framework:
  +-------------------------------------------------------------------+
  | Path                                                    | Purpose  |
  +-------------------------------------------------------------------+
  | frameworks/av/services/camera/libcameraservice/         |          |
  |   common/CameraProviderManager.h      | HAL provider discovery    |
  |   device3/Camera3Device.cpp           | Device state machine      |
  | frameworks/av/camera/                 | Camera2 API native impl   |
  +-------------------------------------------------------------------+

  AIDL Camera Interfaces:
  +-------------------------------------------------------------------+
  | hardware/interfaces/camera/                             |          |
  |   provider/aidl/ICameraProvider.aidl  | Provider discovery        |
  |   device/aidl/ICameraDevice.aidl      | Device open/char          |
  |   device/aidl/ICameraDeviceSession.aidl| Stream config, capture   |
  |   device/aidl/ICameraDeviceCallback.aidl| Result/notify callbacks |
  +-------------------------------------------------------------------+

  Google Desktop Camera HAL (Rust):
  +-------------------------------------------------------------------+
  | vendor/google/desktop/camera/                           |          |
  |   hal_usb/src/v4l2_device.rs          | V4L2 capture wrapper      |
  |   hal_usb/src/uvc_device.rs           | UVC camera discovery      |
  |   hal_usb/src/session/worker.rs       | Capture worker thread     |
  |   hal_usb/src/session/stream.rs       | Stream management         |
  |   common/src/lib.rs                   | Shared library            |
  +-------------------------------------------------------------------+

  Intel IPU7 Camera HAL (C++):
  +-------------------------------------------------------------------+
  | vendor/intel/camera/hal/ipu7/                           |          |
  |   src/core/CaptureUnit.h              | ISYS abstraction          |
  |   src/core/ProcessingUnit.h           | PSYS abstraction          |
  |   src/core/PSysDevice.h              | PSYS kernel interface     |
  |   src/core/DeviceBase.h              | V4L2 device base class    |
  |   src/core/CameraStream.h           | App stream mapping        |
  |   src/core/RequestThread.h           | Request orchestration     |
  |   src/core/CameraContext.h           | Per-camera state          |
  |   src/core/CameraBuffer.h           | Buffer representation     |
  |   src/core/IProcessingUnit.h         | Processing interface      |
  |   src/core/StreamSource.h            | Buffer producer           |
  |   src/core/SofSource.h              | SOF event source          |
  |   src/core/IspSettings.h            | ISP parameters            |
  |   src/core/IpuPacAdaptor.h          | PAC adaptor               |
  |   src/core/SensorHwCtrl.h           | Sensor I2C control        |
  |   src/core/LensHw.h                 | VCM lens focus            |
  |   src/3a/AiqUnit.h                  | 3A control unit           |
  |   ipa/IPAServer.h                   | IPA algorithm server      |
  |   ipa/IPAHeader.h                   | IPA command definitions   |
  |   pipeline/ipaclient/IPAClient.h    | IPA client (HAL-side)     |
  |   pipeline/ipaclient/IPAClientWorker.h| IPA worker threads      |
  |   pipeline/ipaclient/IPAMemory.h    | Shared memory mgmt        |
  |   src/v4l2/V4l2DeviceFactory.h      | V4L2 device creation      |
  +-------------------------------------------------------------------+
```

### 12.2 Ubuntu 相機堆疊檔案

```
  libcamera:
  +-------------------------------------------------------------------+
  | Path                                                    | Purpose  |
  +-------------------------------------------------------------------+
  | src/libcamera/pipeline/simple/simple.cpp                |          |
  |   SimplePipelineHandler               | IPU6 pipeline handler     |
  | src/libcamera/pipeline/ipu3/ipu3.cpp  | IPU3 pipeline handler     |
  | src/libcamera/camera.cpp              | Camera class              |
  | src/libcamera/camera_manager.cpp      | Camera discovery          |
  | src/libcamera/pipeline_handler.cpp    | Base pipeline handler     |
  | src/libcamera/media_device.cpp        | Media controller          |
  | src/libcamera/v4l2_videodevice.cpp    | V4L2 video device         |
  | src/libcamera/v4l2_subdevice.cpp      | V4L2 subdevice            |
  | src/libcamera/software_isp/          | SoftwareISP implementation|
  | src/ipa/simple/                       | Soft IPA (AE, AWB)        |
  +-------------------------------------------------------------------+

  PipeWire:
  +-------------------------------------------------------------------+
  | spa/plugins/libcamera/               | SPA libcamera plugin       |
  | src/modules/module-adapter/          | SPA module adapter         |
  +-------------------------------------------------------------------+

  Ubuntu Packages:
  +-------------------------------------------------------------------+
  | Package                              | Purpose                    |
  +-------------------------------------------------------------------+
  | linux-firmware                       | IPU firmware binaries      |
  | libcamera0.3                         | libcamera shared library   |
  | libcamera-tools                      | cam command-line tool      |
  | pipewire                             | PipeWire daemon            |
  | libspa-0.2-libcamera                 | PipeWire SPA plugin        |
  | wireplumber                          | Session manager            |
  | gstreamer1.0-pipewire                | GStreamer PipeWire element  |
  | libgstreamer-plugins-bad1.0          | libcamerasrc element       |
  +-------------------------------------------------------------------+
```

### 12.3 共用核心驅動程式檔案

```
  Kernel Driver (Both Platforms):
  +-------------------------------------------------------------------+
  | drivers/media/pci/intel/ipu6/                           |          |
  +-------------------------------------------------------------------+
  | ipu6.c                               | Main PCI driver, probe     |
  | ipu6.h                               | Device structures          |
  | ipu6-isys.c                          | ISYS: init, ISR, power     |
  | ipu6-isys.h                          | ISYS structures            |
  | ipu6-isys-video.c                    | V4L2 video: formats, ioctl |
  | ipu6-isys-video.h                    | Video/stream structures    |
  | ipu6-isys-csi2.c                     | CSI-2 receiver             |
  | ipu6-isys-csi2.h                     | CSI-2 structures           |
  | ipu6-isys-queue.c                    | VB2 buffer queue           |
  | ipu6-fw-isys.h                       | Firmware ABI definitions   |
  | ipu6-fw-com.c / .h                   | Firmware syscom IPC        |
  | ipu6-mmu.c                           | IPU internal MMU           |
  | ipu6-buttress.c / .h                 | Power, CSE IPC, TSC sync   |
  | ipu6-bus.c / .h                      | Auxiliary bus devices       |
  | ipu6-cpd.c / .h                      | CPD firmware parsing       |
  | ipu6-dma.h                           | Custom DMA ops             |
  | ipu6-isys-mcd-phy.c                  | MCD PHY (Tiger Lake)       |
  | ipu6-isys-jsl-phy.c                  | JSL PHY (Jasper Lake)      |
  | ipu6-isys-dwc-phy.c                  | DWC PHY (Meteor Lake)      |
  | ipu6-isys-subdev.c                   | V4L2 subdevice helpers     |
  +-------------------------------------------------------------------+

  ipu-bridge (Both Platforms):
  +-------------------------------------------------------------------+
  | drivers/media/pci/intel/ipu-bridge.c | ACPI sensor enumeration    |
  | include/media/ipu-bridge.h           | Bridge structures          |
  | include/media/ipu6-pci-table.h       | PCI device ID table        |
  +-------------------------------------------------------------------+

  Platform Register Headers:
  +-------------------------------------------------------------------+
  | ipu6-platform-buttress-regs.h        | Buttress register offsets  |
  | ipu6-platform-isys-csi2-reg.h        | CSI-2 register offsets     |
  | ipu6-platform-regs.h                 | General platform registers |
  +-------------------------------------------------------------------+

  Device Nodes Created:
  +-------------------------------------------------------------------+
  | /dev/media0                          | Media controller (both)    |
  | /dev/video0..N                       | V4L2 capture (both)        |
  | /dev/v4l-subdev0..N                  | Sensor/CSI-2 subdev (both) |
  | /dev/ipu-psys0                       | PSYS device (Android only) |
  +-------------------------------------------------------------------+
```

### 12.4 關鍵常數（共用）

```
  IPU6_ISYS_MAX_STREAMS         = 16     (max firmware streams)
  IPU6_ISYS_SIZE_SEND_QUEUE     = 40     (command queue depth)
  IPU6_ISYS_SIZE_RECV_QUEUE     = 40     (response queue depth)
  IPU6_ISYS_MIN_WIDTH           = 2      (min capture width)
  IPU6_ISYS_MAX_WIDTH           = 4672   (max capture width)
  IPU6_ISYS_MIN_HEIGHT          = 1      (min capture height)
  IPU6_ISYS_MAX_HEIGHT          = 3416   (max capture height)
  IPU6_MAX_SRAM_SIZE            = 200KB  (IPU6 pixel buffer)
  IPU6SE_MAX_SRAM_SIZE          = 96KB   (IPU6SE pixel buffer)
  IPU6_DEVICE_GDA_NR_PAGES     = 128    (GDA physical pages)
  IPU6_MMU_MAX_DEVICES          = 4      (max MMU units)
  ISP_PAGE_SIZE                 = 4KB    (MMU page size)
  ISP_L1PT_PTES                 = 1024   (L1 page table entries)
  ISP_L2PT_PTES                 = 1024   (L2 page table entries)
  ISYS_PM_QOS_VALUE             = 300us  (CPU latency QoS)
  BUTTRESS_POWER_TIMEOUT_US     = 200ms  (power state timeout)
  BUTTRESS_CSE_BOOTLOAD_TIMEOUT = 5s     (firmware boot timeout)
  BUTTRESS_CSE_AUTHENTICATE_TIMEOUT = 10s (firmware auth timeout)

  Android PSYS Constants (not applicable to Ubuntu):
  MAX_NODE_NUM                  = 5      (max processing nodes)
  MAX_LINK_NUM                  = 10     (max node links)
  MAX_TASK_NUM                  = 8      (max concurrent tasks)
  MAX_TERMINAL_NUM              = 26     (max terminals per node)
```

---

## 文件結尾

本文件提供了 Intel Panther Lake (PTL) 搭配 IPU7 上 Android 與 Ubuntu 相機堆疊的完整技術比較。根本差異在於 Android 利用完整的 IPU 硬體 ISP（PSYS）搭配專有的 Intel CCA 演算法來實現量產品質的影像處理，而 Ubuntu 則依賴 libcamera 的 SoftwareISP 進行軟體影像處理，並搭配基本的開源 3A 演算法。

兩個平台共用相同的核心 ISYS 驅動程式碼、ipu-bridge ACPI 感測器列舉、透過 Buttress 的韌體載入/驗證，以及 V4L2/Media Controller 介面。分歧發生在 PSYS 邊界，並向上傳播至整個使用者空間堆疊。

相關文件請參閱：
- `Android_IPU7_Camera_Architecture.md` - 完整 Android IPU7 相機堆疊
- `Ubuntu_IPU_Camera_Architecture.md` - 完整 Ubuntu/Linux IPU 相機堆疊
- `ES9356_Integration_Technical_Document.md` - PTL 平台整合參考
