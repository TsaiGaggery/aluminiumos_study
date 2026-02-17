# Android IPU7 相機架構 - 完整技術文件

## 文件資訊
- **平台**: Intel Panther Lake (PTL) 搭配 IPU7
- **Android 版本**: ALOS (Android Linux on x86)
- **核心**: 基於 6.18 的核心，支援 IPU6/IPU7 驅動程式
- **日期**: 2026 年 2 月
- **範圍**: 完整的相機堆疊，從 Android 應用程式到硬體暫存器

---

## 目錄

1. [執行摘要](#1-執行摘要)
2. [系統架構概覽](#2-系統架構概覽)
3. [Android Camera2 API 與 CameraService](#3-android-camera2-api-與-cameraservice)
4. [AIDL Camera Provider/Device 介面](#4-aidl-camera-providerdevice-介面)
5. [Google Desktop Camera HAL (Rust/V4L2)](#5-google-desktop-camera-hal-rustv4l2)
6. [Intel IPU7 Camera HAL 架構](#6-intel-ipu7-camera-hal-架構)
7. [IPU7 3A 引擎與 IPA 子系統](#7-ipu7-3a-引擎與-ipa-子系統)
8. [IPU 核心驅動程式架構](#8-ipu-核心驅動程式架構)
9. [V4L2 與 Media Controller 整合](#9-v4l2-與-media-controller-整合)
10. [韌體通訊層](#10-韌體通訊層)
11. [完整資料流程：從應用程式到硬體](#11-完整資料流程從應用程式到硬體)
12. [緩衝區管理與 DMA](#12-緩衝區管理與-dma)
13. [電源管理與效能](#13-電源管理與效能)
14. [關鍵檔案參考索引](#14-關鍵檔案參考索引)

---

## 1. 執行摘要

本文件提供了在執行 Android (ALOS) 的 Intel Panther Lake (PTL) 平台上，相機子系統的全面技術分析。相機管線從 Android Camera2 應用程式 API 開始，經過 CameraService、AIDL HAL 介面、兩個不同的 Camera HAL 實作（Google Desktop HAL 和 Intel IPU7 HAL）、IPU 核心驅動程式，最終到達 IPU 硬體本身。

系統支援兩條 Camera HAL 路徑：

1. **Google Desktop Camera HAL** - 基於 Rust 的 HAL，用於 USB/UVC 相機，直接與 V4L2 視訊擷取裝置通訊。適用於桌面級 USB 網路攝影機。

2. **Intel IPU7 Camera HAL** - 基於 C++ 的 HAL，用於透過 MIPI CSI-2 連接的相機，使用 Intel IPU7 影像處理單元。此 HAL 提供完整的 ISP 管線支援，包括 3A 演算法（AE、AF、AWB）、影像處理（PSYS）和原始擷取（ISYS）。

兩個 HAL 都註冊為 AIDL camera provider，並由 Android CameraService 的 CameraProviderManager 管理。

### 高階架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Android Application                         │
│                     (Camera2 API / CameraX)                        │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ Binder IPC
┌─────────────────────────▼───────────────────────────────────────────┐
│                     CameraService (system_server)                   │
│              frameworks/av/services/camera/libcameraservice/        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │CameraProviderMgr │  │  Camera3Device   │  │  StatusTracker   │  │
│  │  (discovers HALs)│  │ (request/result) │  │  (state machine) │  │
│  └────────┬─────────┘  └────────┬─────────┘  └──────────────────┘  │
└───────────┼──────────────────────┼──────────────────────────────────┘
            │                      │ AIDL IPC
┌───────────▼──────────────────────▼──────────────────────────────────┐
│                    AIDL Camera HAL Interface                        │
│           hardware/interfaces/camera/provider/aidl/                 │
│           hardware/interfaces/camera/device/aidl/                   │
│  ┌────────────────────┐          ┌──────────────────────┐          │
│  │  ICameraProvider   │          │ ICameraDeviceSession  │          │
│  │  ICameraDevice     │          │ ICameraDeviceCallback │          │
│  └────────┬───────────┘          └───────────┬──────────┘          │
└───────────┼──────────────────────────────────┼──────────────────────┘
            │                                  │
    ┌───────▼──────────┐              ┌────────▼─────────────┐
    │ Google Desktop   │              │   Intel IPU7         │
    │ Camera HAL       │              │   Camera HAL         │
    │ (Rust, V4L2/UVC) │              │   (C++, libcamera)   │
    │                  │              │                      │
    │ vendor/google/   │              │ vendor/intel/camera/ │
    │ desktop/camera/  │              │ hal/ipu7/            │
    └───────┬──────────┘              └────────┬─────────────┘
            │                                  │
            │ ioctl(V4L2)                      │ ioctl(V4L2) + IPA
            │                                  │
┌───────────▼──────────────────────────────────▼──────────────────────┐
│                     Linux Kernel                                    │
│  ┌────────────────┐    ┌─────────────────────────────────────────┐  │
│  │  UVC Driver     │    │  Intel IPU6/IPU7 Driver                │  │
│  │  (uvcvideo)     │    │  drivers/media/pci/intel/ipu6/         │  │
│  │                 │    │  ┌──────────┐  ┌──────────┐            │  │
│  │  USB Camera     │    │  │  ISYS    │  │  PSYS    │            │  │
│  │  Capture        │    │  │(capture) │  │(process) │            │  │
│  └────────┬────────┘    │  └─────┬────┘  └─────┬────┘            │  │
│           │             │        │             │                  │  │
│           │             │  ┌─────▼─────────────▼────┐            │  │
│           │             │  │  IPU Firmware (FW)      │            │  │
│           │             │  │  intel/ipu/ipu6_fw.bin  │            │  │
│           │             │  └────────────┬───────────┘            │  │
│           │             └───────────────┼────────────────────────┘  │
└───────────┼─────────────────────────────┼──────────────────────────┘
            │                             │
┌───────────▼────────┐    ┌───────────────▼──────────────────────────┐
│   USB Bus          │    │   PCIe Bus → IPU7 Hardware               │
│   UVC Camera       │    │   MIPI CSI-2 → Camera Sensor             │
└────────────────────┘    └──────────────────────────────────────────┘
```

---

## 2. 系統架構概覽

### 2.1 元件關係

Android 相機堆疊由以下主要層級組成，每個層級都有不同的職責：

| 層級 | 元件 | 位置 | 語言 |
|-------|-----------|----------|----------|
| 應用程式 | Camera2 API | android.hardware.camera2 | Java/Kotlin |
| 框架 | CameraService | frameworks/av/services/camera/ | C++ |
| 介面 | AIDL HAL | hardware/interfaces/camera/ | AIDL→C++ |
| HAL (USB) | Google Desktop HAL | vendor/google/desktop/camera/ | Rust |
| HAL (IPU) | Intel IPU7 HAL | vendor/intel/camera/hal/ipu7/ | C++ |
| 核心 | IPU Driver | drivers/media/pci/intel/ipu6/ | C |
| 韌體 | IPU FW | intel/ipu/ipu6_fw.bin | Binary |
| 硬體 | IPU7 + Sensor | PCIe + MIPI CSI-2 | Silicon |

### 2.2 雙 HAL 架構

ALOS 平台獨特地支援兩個同時運作的 Camera HAL 提供者：

```
                CameraProviderManager
                ┌──────────┴──────────┐
                │                     │
    ┌───────────▼──────────┐  ┌───────▼────────────────┐
    │ Google Desktop HAL   │  │  Intel IPU7 HAL         │
    │                      │  │                         │
    │ - USB/UVC cameras    │  │ - MIPI CSI-2 cameras    │
    │ - Written in Rust    │  │ - Written in C++        │
    │ - Direct V4L2        │  │ - Full ISP pipeline     │
    │ - No ISP needed      │  │ - 3A algorithms (CCA)   │
    │ - JPEG from sensor   │  │ - libcamera-based IPA   │
    │                      │  │ - Graph-based PSYS      │
    └──────────────────────┘  └─────────────────────────┘
```

CameraService 中的 CameraProviderManager 透過 AIDL service manager 發現兩個提供者。每個提供者列舉其相機（Desktop HAL 的 USB 裝置、IPU7 HAL 的 MIPI 感測器），並將合併後的相機列表呈現給應用程式。

### 2.3 原始碼結構

```
ww07-fatcat-bkc/
├── frameworks/av/
│   ├── camera/                          # Camera2 API native implementation
│   └── services/camera/
│       └── libcameraservice/            # CameraService daemon
│           ├── common/
│           │   └── CameraProviderManager.h  # HAL provider discovery
│           └── device3/
│               └── Camera3Device.cpp        # Camera device implementation
│
├── hardware/interfaces/camera/
│   ├── provider/aidl/                   # ICameraProvider AIDL interface
│   │   └── ICameraProvider.aidl
│   └── device/aidl/                     # ICameraDevice AIDL interfaces
│       ├── ICameraDevice.aidl
│       ├── ICameraDeviceSession.aidl
│       └── ICameraDeviceCallback.aidl
│
├── vendor/google/desktop/camera/        # Google Desktop Camera HAL
│   ├── hal_usb/src/
│   │   ├── v4l2_device.rs               # V4L2 capture device wrapper
│   │   ├── uvc_device.rs                # UVC device discovery
│   │   └── session/
│   │       ├── worker.rs                # Capture worker thread
│   │       ├── stream.rs                # Stream management
│   │       └── capture_config.rs        # Capture configuration
│   └── common/src/lib.rs                # Shared library (AIDL, FMQ, etc.)
│
├── vendor/intel/camera/hal/ipu7/        # Intel IPU7 Camera HAL
│   ├── src/
│   │   ├── core/
│   │   │   ├── CaptureUnit.h            # ISYS abstraction
│   │   │   ├── ProcessingUnit.h         # PSYS abstraction
│   │   │   ├── PSysDevice.h             # PSYS device interface
│   │   │   ├── CameraStream.h           # App stream management
│   │   │   ├── RequestThread.h          # Request handling thread
│   │   │   ├── CameraContext.h          # Per-camera state
│   │   │   └── DeviceBase.h             # V4L2 device base class
│   │   ├── 3a/
│   │   │   └── AiqUnit.h                # 3A algorithm unit
│   │   └── v4l2/
│   │       └── V4l2DeviceFactory.h      # V4L2 device creation
│   ├── ipa/
│   │   ├── IPAServer.h                  # IPA algorithm server
│   │   └── IPAHeader.h                  # IPA command definitions
│   └── pipeline/ipaclient/
│       └── IPAClient.h                  # IPA client (HAL side)
│
└── desktop-kernel-source/drivers/       # Kernel source (also at
    └── media/pci/intel/ipu6/            # alos-ptl-ww04-kernel-6.18)
        ├── ipu6.c / ipu6.h             # Main PCI driver
        ├── ipu6-isys.c / .h            # Input System (ISYS)
        ├── ipu6-isys-video.c / .h      # V4L2 video device
        ├── ipu6-isys-csi2.c / .h       # CSI-2 receiver
        ├── ipu6-isys-queue.c            # Buffer queue management
        ├── ipu6-fw-isys.h              # Firmware ABI definitions
        ├── ipu6-fw-com.h               # Firmware communication
        ├── ipu6-mmu.c                  # MMU (IOMMU) for IPU
        ├── ipu6-buttress.c             # Buttress (power/IPC)
        ├── ipu6-bus.c                  # Auxiliary bus devices
        ├── ipu6-cpd.c                  # Firmware CPD parsing
        └── ipu6-dma.h                  # DMA operations
```

---

## 3. Android Camera2 API 與 CameraService

### 3.1 CameraService 架構

CameraService 是管理所有相機操作的核心常駐程式。它在 system_server 行程中執行，負責處理：

- 相機裝置列舉與發現
- HAL 提供者生命週期管理
- 應用程式與 HAL 之間的請求/結果路由
- 多客戶端存取控制與驅離策略

**關鍵原始碼**: `frameworks/av/services/camera/libcameraservice/`

### 3.2 CameraProviderManager

CameraProviderManager 發現並管理所有註冊為 AIDL 服務的 Camera HAL 提供者。

**原始碼**: `frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.h`

```
CameraProviderManager
├── initialize()
│   └── Queries ServiceManager for all ICameraProvider services
│       ├── "android.hardware.camera.provider.ICameraProvider/internal/0"
│       │   (Google Desktop HAL)
│       └── "android.hardware.camera.provider.ICameraProvider/intel/0"
│           (Intel IPU7 HAL)
│
├── getCameraDeviceInterface()
│   └── Returns ICameraDevice for a specific camera ID
│
├── getCameraIdList()
│   └── Aggregates camera IDs from all providers
│
└── onRegistration() / isValidDevice()
    └── Dynamic provider registration callbacks
```

提供者管理器維護一個內部資料結構對映：

```
┌─────────────────────────────────────────────────────────┐
│                 CameraProviderManager                    │
│                                                          │
│  mProviders[]                                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │ ProviderInfo                                       │  │
│  │  ├── mProviderName: "internal/0"                   │  │
│  │  ├── mInterface: ICameraProvider (AIDL proxy)      │  │
│  │  └── mDevices[]                                    │  │
│  │      ├── DeviceInfo { cameraId: "0", ... }         │  │
│  │      └── DeviceInfo { cameraId: "1", ... }         │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │ ProviderInfo                                       │  │
│  │  ├── mProviderName: "intel/0"                      │  │
│  │  ├── mInterface: ICameraProvider (AIDL proxy)      │  │
│  │  └── mDevices[]                                    │  │
│  │      └── DeviceInfo { cameraId: "2", ... }         │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 3.3 Camera3Device

Camera3Device 是實作相機裝置狀態機和請求/結果管線的核心類別。

**原始碼**: `frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp`

關鍵操作：

1. **initializeCommonLocked()** - 開啟 HAL 裝置，建立裝置會話
2. **configureStreamsLocked()** - 將串流配置傳送至 HAL
3. **processCaptureRequest()** - 提交擷取請求
4. **processCaptureResult()** - 從 HAL 回呼接收結果

```
Camera3Device State Machine:

    ┌───────────┐   open()    ┌──────────────┐
    │  CLOSED   ├────────────►│ UNCONFIGURED │
    └───────────┘             └──────┬───────┘
                                     │ configureStreams()
                              ┌──────▼───────┐
                              │  CONFIGURED   │◄─────────────────┐
                              └──────┬───────┘                   │
                                     │ processCaptureRequest()   │
                              ┌──────▼───────┐                   │
                              │   ACTIVE      │                  │
                              │  (streaming)  ├──────────────────┘
                              └──────┬───────┘  configureStreams()
                                     │ flush() / disconnect()
                              ┌──────▼───────┐
                              │    CLOSED     │
                              └──────────────┘
```

### 3.4 請求處理管線

```
Application                 CameraService              HAL
    │                            │                      │
    │ submitCaptureRequest()     │                      │
    ├───────────────────────────►│                      │
    │                            │ processCaptureRequest│
    │                            ├─────────────────────►│
    │                            │                      │ Queue to ISYS
    │                            │                      │ Run 3A
    │                            │                      │ Process in PSYS
    │                            │                      │
    │                            │  processCaptureResult│
    │                            │◄─────────────────────┤
    │  onCaptureCompleted()      │                      │
    │◄───────────────────────────┤                      │
    │                            │                      │
    │                            │  notify(SHUTTER)     │
    │                            │◄─────────────────────┤
    │  onCaptureStarted()        │                      │
    │◄───────────────────────────┤                      │
```

---

## 4. AIDL Camera Provider/Device 介面

### 4.1 介面層次結構

AIDL 相機介面定義了 CameraService 與 Camera HAL 實作之間的合約。

**原始碼**: `hardware/interfaces/camera/`

```
hardware/interfaces/camera/
├── provider/aidl/
│   └── android/hardware/camera/provider/
│       └── ICameraProvider.aidl
│           ├── setCallback(ICameraProviderCallback)
│           ├── getVendorTags() → VendorTagSection[]
│           ├── getCameraIdList() → String[]
│           ├── getCameraDeviceInterface(String) → ICameraDevice
│           └── notifyDeviceStateChange(long)
│
├── device/aidl/
│   └── android/hardware/camera/device/
│       ├── ICameraDevice.aidl
│       │   ├── getCameraCharacteristics() → CameraMetadata
│       │   ├── open(ICameraDeviceCallback) → ICameraDeviceSession
│       │   ├── getPhysicalCameraCharacteristics(String) → CameraMetadata
│       │   └── isStreamCombinationSupported(StreamConfiguration) → bool
│       │
│       ├── ICameraDeviceSession.aidl
│       │   ├── configureStreams(StreamConfiguration) → ConfigureStreamsRet
│       │   ├── processCaptureRequest(CaptureRequest[], ...) → int
│       │   ├── flush() → void
│       │   ├── close() → void
│       │   └── signalStreamFlush(int[], int)
│       │
│       └── ICameraDeviceCallback.aidl
│           ├── processCaptureResult(CaptureResult[]) → void
│           └── notify(NotifyMsg[]) → void
│
└── metadata/aidl/
    └── CameraMetadata.aidl              # Key-value metadata pairs
```

### 4.2 關鍵資料型別

#### 串流配置

```
StreamConfiguration {
    Stream[] streams;           // Array of stream definitions
    StreamConfigurationMode mode; // NORMAL or CONSTRAINED_HIGH_SPEED
    CameraMetadata sessionParams;
}

Stream {
    int32_t id;                 // Unique stream identifier
    StreamType streamType;      // OUTPUT or INPUT
    int32_t width;
    int32_t height;
    PixelFormat format;         // YCBCR_420_888, BLOB, RAW16, etc.
    int32_t usage;              // Gralloc usage flags
    StreamRotation rotation;
    int32_t dataSpace;
    int32_t groupId;
}
```

#### 擷取請求 / 結果

```
CaptureRequest {
    int32_t frameNumber;           // Monotonically increasing
    CameraMetadata settings;       // 3A settings, crop region, etc.
    StreamBuffer[] outputBuffers;  // Buffers to fill
    StreamBuffer[] inputBuffers;   // For reprocessing
}

CaptureResult {
    int32_t frameNumber;           // Matches request
    CameraMetadata result;         // Actual 3A results, timestamps
    StreamBuffer[] outputBuffers;  // Filled buffers
    bool partialResult;            // Partial metadata delivery
}

StreamBuffer {
    int32_t streamId;
    long bufferId;                 // Buffer cache ID
    NativeHandle buffer;           // Gralloc buffer handle
    BufferStatus status;           // OK or ERROR
    NativeHandle acquireFence;
    NativeHandle releaseFence;
}
```

#### HAL 串流回應

```
ConfigureStreamsRet {
    HalStream[] streams;
}

HalStream {
    int32_t id;                    // Matches Stream.id
    PixelFormat overrideFormat;    // HAL may override format
    int32_t overrideDataSpace;
    int32_t producerUsage;         // Additional usage flags
    int32_t maxBuffers;            // Max buffers HAL needs
}
```

#### 通知訊息

```
NotifyMsg {
    MsgType type;                  // ERROR or SHUTTER

    // For SHUTTER:
    ShutterMsg shutter {
        int32_t frameNumber;
        long timestamp;            // Sensor start-of-exposure (ns)
    }

    // For ERROR:
    ErrorMsg error {
        int32_t frameNumber;
        int32_t errorStreamId;
        ErrorCode errorCode;       // DEVICE, REQUEST, RESULT, BUFFER
    }
}
```

### 4.3 會話生命週期

```
┌──────────────────────────────────────────────────────────────────┐
│                    ICameraDeviceSession Lifecycle                 │
│                                                                  │
│  1. open(callback) → session                                     │
│     └─ HAL creates internal pipeline objects                     │
│                                                                  │
│  2. configureStreams(config) → halStreams                         │
│     └─ HAL configures ISYS/PSYS pipeline for requested streams   │
│     └─ Returns max buffer counts and format overrides            │
│                                                                  │
│  3. processCaptureRequest(requests) [repeated]                   │
│     └─ HAL queues buffers to ISYS, triggers 3A, processes PSYS   │
│     └─ Calls callback.processCaptureResult() when done           │
│     └─ Calls callback.notify(SHUTTER) at start of frame          │
│                                                                  │
│  4. flush()                                                      │
│     └─ HAL returns all in-flight buffers with ERROR status       │
│                                                                  │
│  5. close()                                                      │
│     └─ HAL tears down pipeline and releases resources            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Google Desktop Camera HAL (Rust/V4L2)

### 5.1 概覽

Google Desktop Camera HAL 是一個基於 Rust 的實作，專為 USB/UVC 相機設計。它直接使用 V4L2 API 從 USB 網路攝影機擷取影格，無需 ISP 管線。

**原始碼**: `vendor/google/desktop/camera/`

### 5.2 架構

```
┌─────────────────────────────────────────────────┐
│          Google Desktop Camera HAL               │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │            AIDL Provider Layer             │   │
│  │  ICameraProvider / ICameraDevice          │   │
│  │  ICameraDeviceSession                     │   │
│  └─────────────────┬────────────────────────┘   │
│                    │                             │
│  ┌─────────────────▼────────────────────────┐   │
│  │            Session Layer                   │   │
│  │  ┌──────────┐  ┌──────────┐              │   │
│  │  │  Worker   │  │  Stream  │              │   │
│  │  │ (capture  │  │  (per-   │              │   │
│  │  │  thread)  │  │  stream  │              │   │
│  │  │          │  │  config)  │              │   │
│  │  └────┬─────┘  └──────────┘              │   │
│  └───────┼──────────────────────────────────┘   │
│          │                                       │
│  ┌───────▼──────────────────────────────────┐   │
│  │          Device Layer                      │   │
│  │  ┌────────────────┐  ┌─────────────────┐ │   │
│  │  │ V4L2Capture    │  │  UVCDevice      │ │   │
│  │  │ Device         │  │  (discovery)    │ │   │
│  │  │ (v4l2r crate)  │  │  (vendor_id,    │ │   │
│  │  │                │  │   product_id)   │ │   │
│  │  └────────────────┘  └─────────────────┘ │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 5.3 V4L2 擷取裝置

**原始碼**: `vendor/google/desktop/camera/hal_usb/src/v4l2_device.rs`

V4L2CaptureDevice 使用 `v4l2r` Rust crate 封裝了 Linux V4L2 視訊擷取介面。它負責處理：

- 透過 `/dev/videoN` 開啟/關閉裝置
- 格式協商（MJPEG、YUYV、NV12）
- 透過 MMAP 或 DMABUF 進行緩衝區分配
- 透過 VIDIOC_STREAMON/STREAMOFF 進行串流啟動/停止
- 透過 VIDIOC_QBUF/DQBUF 進行緩衝區排隊/出隊

該裝置需要 `VIDEO_CAPTURE` 能力並使用單平面緩衝區。

### 5.4 UVC 裝置發現

**原始碼**: `vendor/google/desktop/camera/hal_usb/src/uvc_device.rs`

UVCDevice 結構表示一個已發現的 USB 相機，包含：

```rust
struct UVCDevice {
    vendor_id: u16,      // USB vendor ID
    product_id: u16,     // USB product ID
    device_path: String, // e.g., "/dev/video0"
    characteristics: UVCCharacteristics,
}
```

發現過程掃描 `/sys/class/video4linux/` 以尋找具有 `VIDEO_CAPTURE` 能力的視訊裝置，並將其與已知的 USB 相機描述符進行比對。

### 5.5 Worker 執行緒架構

**原始碼**: `vendor/google/desktop/camera/hal_usb/src/session/worker.rs`

Worker 作為一個專用擷取執行緒運作，處理兩種類型的任務：

```
Worker Thread Event Loop:
┌─────────────────────────────────────────┐
│                                         │
│   loop {                                │
│       match receiver.recv() {           │
│           Configure(config) => {        │
│               // Set V4L2 format        │
│               // Allocate buffers       │
│               // STREAMON               │
│           }                             │
│           Capture(request) => {         │
│               // DQBUF from V4L2        │
│               // Process frame:         │
│               //   - MJPEG decode       │
│               //   - YUV conversion     │
│               //   - JPEG encode        │
│               // Send result callback   │
│           }                             │
│           Stop => break                 │
│       }                                 │
│   }                                     │
│                                         │
└─────────────────────────────────────────┘
```

Worker 使用 crossbeam channels 與會話層進行通訊。在可能的情況下，使用 DMA 平面佈局進行零拷貝緩衝區傳遞。

### 5.6 共用程式庫

**原始碼**: `vendor/google/desktop/camera/common/src/lib.rs`

共用程式庫提供以下共用模組：

- **aidl** - Rust 的 AIDL 型別序列化
- **fmq** - Fast Message Queue (FMQ) 整合
- **format** - 像素格式轉換工具
- **geometry** - 影像尺寸計算
- **metadata** - 相機中繼資料處理

---

## 6. Intel IPU7 Camera HAL 架構

### 6.1 概覽

Intel IPU7 Camera HAL 是一個功能完整的 C++ Camera HAL，用於 MIPI CSI-2 連接的影像感測器。它提供完整的影像管線，包括原始擷取（ISYS）、影像訊號處理（PSYS）以及透過 Intel CCA（Camera Control Algorithm）的 3A 演算法整合。

**原始碼**: `vendor/intel/camera/hal/ipu7/`

此 HAL 在 `icamera` 命名空間中運作，並使用生產者/消費者緩衝區管線架構。

### 6.2 頂層架構

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Intel IPU7 Camera HAL                              │
│                    vendor/intel/camera/hal/ipu7/                      │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                     AIDL Provider Layer                          │ │
│  │           (ICameraProvider / ICameraDeviceSession)               │ │
│  └────────────────────────────┬────────────────────────────────────┘ │
│                               │                                      │
│  ┌────────────────────────────▼────────────────────────────────────┐ │
│  │                      RequestThread                               │ │
│  │  src/core/RequestThread.h                                        │ │
│  │  - Accepts capture requests from framework                       │ │
│  │  - Triggers 3A algorithm execution                               │ │
│  │  - Manages per-frame control sequencing                          │ │
│  │  - Event-driven: NEW_REQUEST, NEW_FRAME, NEW_STATS, NEW_SOF     │ │
│  └───────┬──────────────────┬──────────────────┬───────────────────┘ │
│          │                  │                  │                      │
│  ┌───────▼───────┐  ┌──────▼───────┐  ┌───────▼──────────┐         │
│  │  CaptureUnit  │  │ ProcessingUnit│  │    AiqUnit       │         │
│  │  (ISYS)       │  │ (PSYS)       │  │   (3A Engine)    │         │
│  │               │  │              │  │                  │         │
│  │ StreamSource  │  │IProcessingUnit│  │ AiqEngine        │         │
│  │ DeviceBase    │  │PipeManager   │  │ IntelCca         │         │
│  │ V4L2 capture  │  │PSysDevice    │  │ SensorHwCtrl     │         │
│  │               │  │              │  │ LensHw           │         │
│  └───────┬───────┘  └──────┬───────┘  └───────┬──────────┘         │
│          │                 │                   │                      │
│  ┌───────▼─────────────────▼───────────────────▼───────────────────┐ │
│  │                    CameraContext (per-camera)                     │ │
│  │  src/core/CameraContext.h                                        │ │
│  │  - AiqResultStorage: stores 3A results                           │ │
│  │  - DataContext pool: per-frame parameters                        │ │
│  │  - GraphConfig: pipeline graph configuration                     │ │
│  │  - Sequence→DataContext mapping                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                    IPA Client/Server                              │ │
│  │  pipeline/ipaclient/IPAClient.h   ipa/IPAServer.h                │ │
│  │  - Sandboxed 3A algorithm execution                              │ │
│  │  - Shared memory for parameters/statistics                       │ │
│  │  - libcamera IPA proxy interface                                 │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.3 CaptureUnit（ISYS 抽象層）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/CaptureUnit.h`

CaptureUnit 抽象化了 IPU 的 ISYS（輸入系統）功能。它是影像管線的來源——從相機感測器產生原始影格。

```cpp
class CaptureUnit : public StreamSource, public DeviceCallback {
    // Key interfaces:
    int qbuf(uuid port, const shared_ptr<CameraBuffer>& camBuffer);
    int allocateMemory(uuid port, const shared_ptr<CameraBuffer>& camBuffer);
    void addFrameAvailableListener(BufferConsumer* listener);
    int init();
    void deinit();
    int start();    // STREAMON + poll thread
    int stop();     // STREAMOFF + release buffers
    int configure(const map<uuid, stream_t>& outputFrames);
};
```

內部狀態機：

```
CAPTURE_UNINIT → CAPTURE_INIT → CAPTURE_CONFIGURE → CAPTURE_START
                                                          │
                                                          ▼
                                                    CAPTURE_STOP
```

CaptureUnit 管理 DeviceBase 物件，每個 V4L2 視訊節點各一個：

```
CaptureUnit
├── mDevices[] : vector<DeviceBase*>
│   ├── MainDevice (VIDEO_CAPTURE for pixel data)
│   └── MainDevice (META_CAPTURE for embedded data)
│
├── mPollThread : PollThread<CaptureUnit>
│   └── Calls poll() → dequeues buffers from devices
│
└── mOutputFrameInfo : map<uuid, stream_t>
    └── Per-port output frame configurations
```

### 6.4 DeviceBase（V4L2 裝置封裝）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/DeviceBase.h`

DeviceBase 是 V4L2 裝置互動的基礎類別：

```cpp
class DeviceBase : public EventSource {
    int configure(uuid port, const stream_t& config, uint32_t bufferNum);
    int openDevice();
    int streamOn();
    int streamOff();
    int queueBuffer(int64_t sequence);
    int dequeueBuffer();

    // Buffer management:
    vector<shared_ptr<CameraBuffer>> mAllocatedBuffers;  // Internal pool
    list<shared_ptr<CameraBuffer>> mPendingBuffers;      // Ready to queue
    list<shared_ptr<CameraBuffer>> mBuffersInDevice;     // Queued to driver
};
```

DeviceBase 中的緩衝區流程：

```
                    ┌──────────────────┐
  allocate ────────►│ mAllocatedBuffers │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
  addPending ──────►│  mPendingBuffers  │ (ready to queue)
                    └────────┬─────────┘
                             │ queueBuffer()
                    ┌────────▼─────────┐
  VIDIOC_QBUF ────►│ mBuffersInDevice  │ (in kernel)
                    └────────┬─────────┘
                             │ dequeueBuffer()
                    ┌────────▼─────────┐
  notify listeners ►│   Consumers       │ (ProcessingUnit)
                    └──────────────────┘
```

MainDevice 是像素產生視訊節點的主要子類別：

```cpp
class MainDevice : public DeviceBase {
    int createBufferPool(const stream_t& config);
    int onDequeueBuffer(shared_ptr<CameraBuffer> buffer);
    bool needQueueBack(shared_ptr<CameraBuffer> buffer);
};
```

### 6.5 ProcessingUnit（PSYS 抽象層）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/ProcessingUnit.h`

ProcessingUnit 在 PSYS（處理系統）中執行影像訊號處理演算法。它從 CaptureUnit 接收原始影格，並產生經過處理的輸出影格（YUV、JPEG 等）。

```cpp
class ProcessingUnit : public IProcessingUnit, public PipeManagerCallback {
    int configure(const map<uuid, stream_t>& inputInfo,
                  const map<uuid, stream_t>& outputInfo,
                  const ConfigMode configModes);
    int start();
    void stop();

    // PipeManagerCallback:
    void onTaskDone(const PipeTaskData& result);
    void onBufferDone(int64_t sequence, uuid port, shared_ptr<CameraBuffer>& buf);
    void onMetadataReady(int64_t sequence, const CameraBufferPortMap& outBuf);
    void onStatsReady(EventData& eventData);
};
```

關鍵內部元件：

```
ProcessingUnit
├── mPipeManager : unique_ptr<IPipeManager>
│   └── Manages PSYS processing graph
│       └── Uses PSysDevice for kernel interaction
│
├── mIspSettings : IspSettings
│   └── Current ISP parameter block (edge, NR, etc.)
│
├── mScheduler : shared_ptr<CameraScheduler>
│   └── Schedules PSYS tasks across nodes
│
├── mSequencesInflight : multiset<int64_t>
│   └── Tracks in-flight frame sequences
│
└── mConfigMode / mTuningMode
    └── Active pipeline configuration
```

處理流程：

```
CaptureUnit                ProcessingUnit               PSysDevice
    │                           │                            │
    │  onBufferAvailable()      │                            │
    ├──────────────────────────►│                            │
    │                           │ processNewFrame()          │
    │                           │ ┌─────────────────┐       │
    │                           │ │prepareTask()    │       │
    │                           │ │ - get src bufs  │       │
    │                           │ │ - get dst bufs  │       │
    │                           │ │ - get settings  │       │
    │                           │ └────────┬────────┘       │
    │                           │          │                 │
    │                           │ dispatchTask()             │
    │                           │ ┌────────▼────────┐       │
    │                           │ │ PipeManager      │       │
    │                           │ │ addTask()        │──────►│ addTask()
    │                           │ └─────────────────┘       │ (ioctl)
    │                           │                            │
    │                           │          ◄─────────────────┤ bufferDone()
    │                           │ onTaskDone()               │
    │                           │ onBufferDone()             │
    │                           │ ┌─────────────────┐       │
    │                           │ │return buffers to│       │
    │                           │ │CameraStream     │       │
    │                           │ └─────────────────┘       │
```

### 6.6 PSysDevice（PSYS 核心介面）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/PSysDevice.h`

PSysDevice 提供了 IPU PSYS 核心裝置的使用者空間介面。它使用基於圖的處理模型運作：

```cpp
class PSysDevice {
    int init();           // Opens /dev/ipu-psys0
    int addGraph(const PSysGraph& graph);   // Sets up processing graph
    int closeGraph();                        // Tears down graph
    int addTask(const PSysTask& task);       // Submits processing task
    int registerBuffer(TerminalBuffer* buf); // Registers DMA buffer
    void unregisterBuffer(const TerminalBuffer* buf);
    int poll();           // Waits for task completion
};
```

關鍵資料結構：

```
PSysGraph                          PSysTask
┌─────────────────────┐           ┌──────────────────────┐
│ nodes: list<PSysNode>│          │ nodeCtxId: uint8     │
│ links: list<PSysLink>│          │ sequence: int64      │
└──────┬──────────────┘          │ terminalBuffers:     │
       │                          │  map<id, TermBuf>    │
       ▼                          └──────────────────────┘
PSysNode
┌──────────────────────┐          TerminalBuffer
│ nodeCtxId: uint8     │          ┌──────────────────────┐
│ nodeRsrcId: uint8    │          │ userPtr: void*       │
│ bitmaps: PSysBitmap  │          │ handle: uint64       │
│ terminalConfig:      │          │ size: uint32         │
│  map<id, TermConfig> │          │ flags: uint32        │
└──────────────────────┘          │ psysBuf: ipu_psys_buf│
                                  │ isExtDmaBuf: bool    │
PSysLink                          └──────────────────────┘
┌──────────────────────┐
│ srcNodeCtxId         │
│ srcTermId            │
│ dstNodeCtxId         │
│ dstTermId            │
│ streamingMode        │
│ delayedLink          │
└──────────────────────┘
```

PSYS 圖定義了由連結串接的處理節點。每個節點具有帶有關聯緩衝區的終端（輸入/輸出埠）。任務指定要執行哪個節點，以及給定影格序列要使用哪些緩衝區。

### 6.7 CameraStream（應用程式串流）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/CameraStream.h`

CameraStream 將應用程式串流橋接到內部管線：

```cpp
class CameraStream : public BufferConsumer, public EventSource {
    void setPort(uuid port);    // Maps to internal pipeline port
    int start();
    int stop();
    int qbuf(camera_buffer_t* ubuffer, int64_t sequence, bool addExtraBuf);
    int allocateMemory(camera_buffer_t* ubuffer);
    int onBufferAvailable(uuid port, const shared_ptr<CameraBuffer>& camBuffer);
};
```

從應用程式到硬體的緩衝區流程：

```
App buffer (camera_buffer_t)
    │
    │ userBufferToCameraBuffer()
    ▼
CameraBuffer (internal representation)
    │
    │ BufferProducer::qbuf()
    ▼
CaptureUnit → V4L2 ISYS device
    │
    │ ISYS dequeue → onBufferAvailable
    ▼
ProcessingUnit → PSYS pipeline
    │
    │ onBufferDone → onBufferAvailable
    ▼
CameraStream → return to app
```

### 6.8 RequestThread（請求協調）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/RequestThread.h`

RequestThread 管理請求佇列，並協調 3A 執行與影格擷取：

```cpp
class RequestThread : public Thread, public EventSource, public EventListener {
    int processRequest(int bufferNum, camera_buffer_t **ubuffer);
    int waitFrame(int streamId, camera_buffer_t **ubuffer);
    int wait1stRequestDone();
    int configure(const stream_config_t *streamList);
    void handleEvent(EventData eventData);
};
```

事件驅動處理模型：

```
RequestThread Event Loop:

    ┌──────────────────────────────────┐
    │         threadLoop()              │
    │                                   │
    │  Wait for trigger event:          │
    │  ┌─────────────────────────┐     │
    │  │ NEW_REQUEST (from app)  │     │
    │  │ NEW_FRAME  (from ISYS)  │     │
    │  │ NEW_STATS  (from PSYS)  │     │
    │  │ NEW_SOF    (from sensor)│     │
    │  └────────────┬────────────┘     │
    │               │                   │
    │  fetchNextRequest()               │
    │  ┌────────────▼────────────┐     │
    │  │ handleRequest():         │     │
    │  │  1. Get DataContext       │     │
    │  │  2. m3AControl->run3A()  │     │
    │  │  3. Queue to CaptureUnit │     │
    │  │  4. Track sequences      │     │
    │  └──────────────────────────┘     │
    │                                   │
    └──────────────────────────────────┘
```

序列追蹤欄位：

```
mLastCcaId      - CCA algorithm instance ID
mLastEffectSeq  - Sequence where last 3A results took effect
mLastAppliedSeq - Sequence where last 3A results were applied
mLastSofSeq     - Last Start-of-Frame sequence received
mBlockRequest   - Blocks 2nd/3rd request until 1st 3A event
```

---

## 7. IPU7 3A 引擎與 IPA 子系統

### 7.1 AiqUnit（3A 演算法控制）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/3a/AiqUnit.h`

AiqUnit 管理 Intel CCA（Camera Control Algorithm）程式庫，用於 3A 處理（自動曝光、自動對焦、自動白平衡）。

```cpp
class AiqUnit : public AiqUnitBase {
    void init();
    void deinit();
    int configure(const stream_config_t *streamList);
    int start();
    void stop();
    int run3A(int64_t ccaId, int64_t applyingSeq,
              int64_t frameNumber, int64_t* effectSeq);
    vector<EventListener*> getSofEventListener();
    vector<EventListener*> getStatsEventListener();
};
```

AiqUnit 狀態機：

```
AIQ_UNIT_NOT_INIT → AIQ_UNIT_INIT → AIQ_UNIT_CONFIGURED
                                          │
                                    AIQ_UNIT_START
                                          │
                                    AIQ_UNIT_STOP
```

內部元件：

```
AiqUnit
├── mAiqEngine : AiqEngine*
│   └── Orchestrates 3A algorithm execution
│       ├── AEC (Auto Exposure Control)
│       ├── AWB (Auto White Balance)
│       ├── AF  (Auto Focus)
│       └── ISP parameter generation
│
├── mTuningModes : vector<TuningMode>
│   └── Active tuning mode per stream configuration
│
├── mCcaInitialized : bool
│   └── Whether CCA library is initialized
│
└── initIntelCcaHandle()
    └── Initializes CCA with:
        - AIQB (tuning data binary)
        - NVM (sensor calibration data)
        - CMC (color matrix characterization)
```

### 7.2 IPA 架構（Image Processing Algorithm）

IPA 子系統提供沙箱化的 3A 演算法執行，將演算法程式庫與主 HAL 行程分離以確保安全性。

```
┌──────────────────────────────────────────────────────────────────┐
│                     IPA Architecture                              │
│                                                                  │
│  HAL Process                         IPA Process                 │
│  ┌────────────────────┐             ┌──────────────────────┐    │
│  │                    │             │                      │    │
│  │    IPAClient       │  libcamera  │    IPAServer         │    │
│  │    (singleton)     │◄───IPA────►│    (per-camera)      │    │
│  │                    │   proxy     │                      │    │
│  │  Functions:        │             │  Functions:          │    │
│  │  - initCca()       │  Shared     │  - init()            │    │
│  │  - runAec()        │  Memory     │  - sendRequest()     │    │
│  │  - runAiq()        │  (params/   │                      │    │
│  │  - configAic()     │   stats)    │  ┌────────────────┐  │    │
│  │  - runAic()        │             │  │   CcaWorker    │  │    │
│  │  - decodeStats()   │             │  │  (per camera+  │  │    │
│  │  - setStats()      │             │  │   tuning mode) │  │    │
│  │  - updateTuning()  │             │  │                │  │    │
│  │  - getCmc()        │             │  │  Intel CCA lib │  │    │
│  │  - getMkn()        │             │  │  (AE,AF,AWB)   │  │    │
│  │  - getAiqd()       │             │  └────────────────┘  │    │
│  │                    │             │                      │    │
│  └────────────────────┘             └──────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 7.3 IPAClient

**原始碼**: `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAClient.h`

IPAClient 是一個單例模式，透過 libcamera IPA proxy 機制與 IPA 伺服器行程進行通訊：

```cpp
class IPAClient : public IAlgoClient, public Thread {
    static IPAClient* getInstance();
    static IPAClient* createInstance(PipelineHandler* handler);

    bool setupIPA();    // Establish IPA connection
    bool loadIPA();     // Load IPA module
    bool init(uint32_t bufferId);

    // CCA operations (sent to IPAServer):
    int initCca(int cameraId, int tuningMode, uint32_t bufferId);
    int runAec(int cameraId, int tuningMode, uint32_t bufferId);
    int runAiq(int cameraId, int tuningMode, uint32_t bufferId);
    int configAic(int cameraId, int tuningMode, uint32_t bufferId);
    int runAic(int cameraId, int tuningMode, uint32_t bufferId);
    int decodeStats(int cameraId, int tuningMode, uint32_t bufferId);
    int setStats(int cameraId, int tuningMode, uint32_t bufferId);

    // Shared memory management:
    bool allocMem(const string& name, int size, void** addr, uint32_t& handle);
    void freeMem(const string& name, void* addr, uint32_t handle);
};
```

### 7.4 IPAServer

**原始碼**: `vendor/intel/camera/hal/ipu7/ipa/IPAServer.h`

IPAServer 在 IPA 行程（或沙箱化的執行緒）中執行，並託管實際的 CCA 演算法實例：

```cpp
class IPAServer {
    IPAServer(IPAIPUCallback* callback);
    int init(uint8_t* data);
    int sendRequest(int cameraId, int tuningMode,
                    uint32_t cmd, const Span<uint8_t>& mem);

    // Internal:
    map<pair<int,int>, unique_ptr<CcaWorker>> mCcaWorkers;
    // Key: (cameraId, tuningMode) → CcaWorker
};
```

每個 CcaWorker 封裝一個 Intel CCA 程式庫實例，並處理特定相機和調校模式組合的演算法請求。

### 7.5 3A 處理流程

```
Frame N arrives from sensor
    │
    ▼
ISYS captures raw frame + embedded statistics
    │
    ▼
Statistics decoded: IPAClient::decodeStats()
    │                   │
    │          IPAServer::sendRequest(DECODE_STATS)
    │                   │
    │          CcaWorker processes statistics
    │                   │
    ▼                   ▼
AiqUnit::run3A(ccaId, applyingSeq, frameNumber)
    │
    ├── IPAClient::runAec()  → AE parameters
    │       └── exposure time, analog gain, digital gain
    │
    ├── IPAClient::runAiq()  → AWB + other parameters
    │       └── color correction, gamma, tone map
    │
    └── IPAClient::runAic()  → ISP configuration
            └── PSYS node parameters, filter coefficients
    │
    ▼
Results stored in AiqResultStorage
    │
    ▼
ProcessingUnit reads results for frame N+latency
    │
    ▼
PSYS processes raw→YUV with 3A-tuned parameters
```

### 7.6 CameraContext（每個相機的狀態）

**原始碼**: `vendor/intel/camera/hal/ipu7/src/core/CameraContext.h`

CameraContext 是每個相機 ID 的單例模式，儲存所有每影格的狀態：

```cpp
class CameraContext {
    static CameraContext* getInstance(int cameraId);

    AiqResultStorage* getAiqResultStorage();

    DataContext* acquireDataContext();
    void updateDataContextMapByFn(int64_t frameNumber, DataContext* context);
    void updateDataContextMapBySeq(int64_t sequence, DataContext* context);
    void updateDataContextMapByCcaId(int64_t ccaId, DataContext* context);

    const DataContext* getDataContextBySeq(int64_t sequence);
    const DataContext* getDataContextByCcaId(int64_t ccaId);

    shared_ptr<GraphConfig> getGraphConfig(ConfigMode configMode);
};
```

DataContext 擷取所有每影格的參數：

```cpp
class DataContext {
    int64_t mFrameNumber;    // App frame number
    int64_t mSequence;       // Kernel buffer sequence
    int64_t mCcaId;          // CCA algorithm instance ID

    aiq_parameter_t mAiqParams;   // 3A input parameters
    IspParameters mIspParams;      // ISP settings (edge, NR, etc.)
    camera_crop_region_t cropRegion;
    camera_zoom_region_t zoomRegion;
};
```

---
## 8. IPU 核心驅動程式架構

### 8.1 概述

IPU 核心驅動程式位於：
`drivers/media/pci/intel/ipu6/`

儘管命名為 "ipu6"，此驅動程式支援 IPU6 至 IPU7 的硬體變體。該驅動程式是一個 PCI 裝置驅動程式，會建立兩個 auxiliary bus 裝置：ISYS（輸入系統）和 PSYS（處理系統）。

### 8.2 驅動程式模組結構

```
drivers/media/pci/intel/ipu6/
│
├── ipu6.c / ipu6.h              # 主要 PCI 驅動程式，裝置初始化
│   └── ipu6_pci_probe()          # PCI probe 進入點
│   └── ipu6_configure_spc()      # Signal Processing Cell 設定
│
├── ipu6-isys.c / ipu6-isys.h    # 輸入系統 (ISYS)
│   └── isys_isr()                # ISYS 中斷處理常式
│   └── isys_setup_hw()           # 硬體初始化
│   └── update_watermark_setting() # iWake 電源管理
│
├── ipu6-isys-video.c / .h       # V4L2 視訊裝置節點
│   └── ipu6_isys_pfmts[]        # 支援的像素格式
│   └── video_open/close/ioctl    # V4L2 檔案操作
│
├── ipu6-isys-csi2.c / .h        # CSI-2 接收器子裝置
│   └── CSI-2 時序計算
│   └── DPHY/DWC PHY 控制
│   └── 錯誤偵測與回報
│
├── ipu6-isys-queue.c             # VB2 緩衝區佇列管理
│   └── ipu6_isys_buf_init()      # 緩衝區的 DMA 映射
│   └── buffer_list_get()          # 收集擷取用緩衝區
│
├── ipu6-fw-isys.h                # 韌體 ABI 定義
│   └── 命令/回應列舉
│   └── 串流設定結構
│   └── 影格緩衝區結構
│
├── ipu6-fw-com.h / .c           # 韌體通訊層
│   └── 基於 token 的訊息傳遞
│
├── ipu6-mmu.c                    # IPU 的 MMU (IOMMU)
│   └── 兩級頁表 (L1/L2)
│   └── TLB 無效化
│   └── IOVA 管理
│
├── ipu6-buttress.c / .h         # Buttress（電源/安全/IPC）
│   └── CSE IPC 用於韌體認證
│   └── 電源狀態管理
│   └── TSC 同步
│
├── ipu6-bus.c / .h              # Auxiliary bus 裝置管理
│   └── ISYS/PSYS 的 PM domain
│
├── ipu6-cpd.c / .h             # CPD 韌體解析
│   └── Package directory 解析
│   └── Manifest/metadata 擷取
│
└── ipu6-dma.h                   # DMA 操作
    └── IPU bus 的自訂 DMA alloc/map
```

### 8.3 PCI Probe 與裝置初始化

**原始碼**：`ipu6.c:ipu6_pci_probe()`

```
ipu6_pci_probe()
    │
    ├── 1. PCI 設定
    │   ├── pci_enable_device()
    │   ├── pci_set_master()
    │   ├── pci_request_regions()
    │   └── pci_ioremap_bar()  → isp->base
    │
    ├── 2. 判斷硬體版本
    │   └── 根據 PCI device ID：
    │       IPU6_VER_6      (0x9a19) - Tiger Lake
    │       IPU6_VER_6SE    (0x4e19) - Jasper Lake
    │       IPU6_VER_6EP    (0x465d) - Alder/Raptor Lake
    │       IPU6_VER_6EP_MTL(0x7d19) - Meteor Lake
    │
    ├── 3. 初始化內部平台資料
    │   └── ipu6_internal_pdata_init()
    │       ├── 設定 CSI-2 連接埠數量（每個版本 4-8 個）
    │       ├── 設定 SRAM 粒度和最大大小
    │       ├── 設定 LTR/memopen 門檻值
    │       └── 設定增強型 iWake 參數
    │
    ├── 4. 初始化 buttress（電源/IPC）
    │   └── ipu6_buttress_init()
    │       ├── 設定到 CSE 的 IPC 通道
    │       ├── 註冊中斷處理常式
    │       └── 初始化 mutex/completion 結構
    │
    ├── 5. 初始化 ISYS
    │   └── ipu6_isys_init()
    │       ├── 設定 ISYS MMU（3 個 MMU 單元）
    │       ├── ipu_bridge_init() → ACPI 感測器列舉
    │       ├── ipu6_bus_initialize_device("isys")
    │       └── ipu6_bus_add_device()
    │
    ├── 6. 初始化 PSYS
    │   └── ipu6_psys_init()
    │       ├── 設定 PSYS MMU（4 個 MMU 單元）
    │       ├── ipu6_bus_initialize_device("psys")
    │       └── ipu6_bus_add_device()
    │
    └── 7. 載入並認證韌體
        └── 請求韌體：intel/ipu/ipu6_fw.bin
            └── ipu6_cpd_parse()  # 解析 CPD 韌體容器
```

### 8.4 ISYS（輸入系統）架構

**原始碼**：`ipu6-isys.c`、`ipu6-isys.h`

ISYS 透過 MIPI CSI-2 從攝影機感測器擷取原始影像資料。

```
┌──────────────────────────────────────────────────────────────┐
│                        ISYS Hardware                          │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ CSI-2    │  │ CSI-2    │  │ CSI-2    │  │ CSI-2    │   │
│  │ Port 0   │  │ Port 1   │  │ Port 2   │  │ Port 3   │   │
│  │ (4 lane) │  │ (4 lane) │  │ (4 lane) │  │ (4 lane) │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │          │
│  ┌────▼──────────────▼──────────────▼──────────────▼─────┐   │
│  │              ISYS Pixel Buffer (SRAM)                  │   │
│  │              Max: 200KB (IPU6) / 96KB (IPU6SE)        │   │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌───────────────────────▼───────────────────────────────┐   │
│  │              ISYS DMA Engine                           │   │
│  │              (up to 16 concurrent streams)             │   │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌───────────────────────▼───────────────────────────────┐   │
│  │              GDA (Global Data Allocator)               │   │
│  │              128 pages × 2KB = 256KB                   │   │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│  ┌───────────────────────▼───────────────────────────────┐   │
│  │              IOSF Fabric Interface                     │   │
│  │              (VC0, VC1, VC2 ports with CDC FIFOs)     │   │
│  └───────────────────────┬───────────────────────────────┘   │
│                          │                                    │
└──────────────────────────┼────────────────────────────────────┘
                           │ PCIe / System Memory
                    ┌──────▼──────┐
                    │ Frame Buffer │
                    │ (DDR)        │
                    └─────────────┘
```

#### struct ipu6_isys

**原始碼**：`ipu6-isys.h:127`

```c
struct ipu6_isys {
    struct media_device media_dev;     // Media controller 裝置
    struct v4l2_device v4l2_dev;       // V4L2 裝置
    struct ipu6_bus_device *adev;      // Auxiliary bus 裝置

    int power;                         // 電源狀態
    spinlock_t power_lock;
    u32 isr_csi2_bits;                 // 活躍的 CSI-2 IRQ 位元
    u32 csi2_rx_ctrl_cached;

    spinlock_t streams_lock;
    struct ipu6_isys_stream streams[16]; // 最多 16 個韌體串流
    int streams_ref_count[16];

    void *fwcom;                       // 韌體通訊控制代碼
    u32 phy_termcal_val;               // PHY 終端校準值

    bool need_reset;
    bool icache_prefetch;
    unsigned int ref_count;
    unsigned int stream_opened;

    struct mutex mutex;
    struct mutex stream_mutex;

    struct ipu6_isys_pdata *pdata;     // 平台資料
    struct ipu6_isys_csi2 *csi2;       // CSI-2 接收器陣列

    struct isys_iwake_watermark iwake_watermark; // 電源管理
};
```

#### 串流初始化

**原始碼**：`ipu6-isys.c:144`

```c
static void isys_stream_init(struct ipu6_isys *isys)
{
    for (i = 0; i < IPU6_ISYS_MAX_STREAMS; i++) {
        mutex_init(&isys->streams[i].mutex);
        init_completion(&isys->streams[i].stream_open_completion);
        init_completion(&isys->streams[i].stream_close_completion);
        init_completion(&isys->streams[i].stream_start_completion);
        init_completion(&isys->streams[i].stream_stop_completion);
        INIT_LIST_HEAD(&isys->streams[i].queues);
        isys->streams[i].isys = isys;
        isys->streams[i].stream_handle = i;
        isys->streams[i].vc = INVALID_VC_ID;
    }
}
```

每個串流都有用於韌體命令同步的 completion 物件。
串流生命週期遵循：OPEN → START → CAPTURE（重複）→ STOP → CLOSE。

#### ISR（中斷服務常式）

**原始碼**：`ipu6-isys.c:344`

ISYS ISR 處理兩種類型的中斷：

1. **CSI-2 中斷**（每個連接埠）：影格開始 (SOF)、影格結束 (EOF)、錯誤
2. **軟體中斷**：韌體回應就緒

```
isys_isr(adev)
    │
    ├── 讀取 ctrl0_status（CSI-2 IRQ 狀態）
    ├── 讀取 UNISPART IRQ 狀態（SW IRQ）
    │
    ├── 對每個有待處理 IRQ 的 CSI-2 連接埠：
    │   └── ipu6_isys_csi2_isr(csi2)
    │       ├── 讀取每個 VC 的狀態位元
    │       ├── FS_VC(i) → ipu6_isys_csi2_sof_event_by_stream()
    │       └── FE_VC(i) → ipu6_isys_csi2_eof_event_by_stream()
    │
    └── 若 SW IRQ 待處理：
        └── isys_isr_one(adev)
            └── 從 syscom 佇列處理韌體回應
```

### 8.5 支援的像素格式

**原始碼**：`ipu6-isys-video.c:39`

驅動程式支援以下 V4L2 像素格式：

```
┌────────────────────────┬─────┬──────┬──────────────────────────┐
│ V4L2 Format            │ BPP │ Bits │ FW Format                │
├────────────────────────┼─────┼──────┼──────────────────────────┤
│ V4L2_PIX_FMT_SBGGR12  │  16 │   12 │ RAW16                    │
│ V4L2_PIX_FMT_SGBRG12  │  16 │   12 │ RAW16                    │
│ V4L2_PIX_FMT_SGRBG12  │  16 │   12 │ RAW16                    │
│ V4L2_PIX_FMT_SRGGB12  │  16 │   12 │ RAW16                    │
│ V4L2_PIX_FMT_SBGGR10  │  16 │   10 │ RAW16                    │
│ V4L2_PIX_FMT_SGBRG10  │  16 │   10 │ RAW16                    │
│ V4L2_PIX_FMT_SGRBG10  │  16 │   10 │ RAW16                    │
│ V4L2_PIX_FMT_SRGGB10  │  16 │   10 │ RAW16                    │
│ V4L2_PIX_FMT_SBGGR8   │   8 │    8 │ RAW8                     │
│ V4L2_PIX_FMT_SGBRG8   │   8 │    8 │ RAW8                     │
│ V4L2_PIX_FMT_SGRBG8   │   8 │    8 │ RAW8                     │
│ V4L2_PIX_FMT_SRGGB8   │   8 │    8 │ RAW8                     │
│ V4L2_PIX_FMT_SBGGR12P │  12 │   12 │ RAW12 (packed)           │
│ V4L2_PIX_FMT_SBGGR10P │  10 │   10 │ RAW10 (packed)           │
│ V4L2_PIX_FMT_UYVY     │  16 │   16 │ UYVY                     │
│ V4L2_PIX_FMT_YUYV     │  16 │   16 │ YUYV                     │
│ V4L2_PIX_FMT_RGB565   │  16 │   16 │ RGB565                   │
│ V4L2_PIX_FMT_BGR24    │  24 │   24 │ RGBA888                  │
│ V4L2_META_FMT_*       │ 8-16│  var │ RAW8/10/12/16 (metadata) │
└────────────────────────┴─────┴──────┴──────────────────────────┘

解析度範圍：2×1 至 4672×3416（步進：2×2）
```

### 8.6 MMU（記憶體管理單元）

**原始碼**：`ipu6-mmu.c`

IPU 擁有獨立於系統 IOMMU 的自有 IOMMU。它使用兩級頁表：

```
                    Virtual Address (32-bit)
    ┌──────────┬──────────┬──────────────┐
    │ L1 Index │ L2 Index │ Page Offset  │
    │ [31:22]  │ [21:12]  │   [11:0]     │
    └────┬─────┴────┬─────┴──────────────┘
         │          │
         │    ┌─────▼─────────────────┐
         │    │  L2 Page Table        │
         │    │  1024 entries × 4B    │
         │    │  Maps to 4KB pages    │
         │    └───────────────────────┘
         │
    ┌────▼────────────────────┐
    │  L1 Page Table          │
    │  1024 entries × 4B      │
    │  Each points to L2 PT   │
    └─────────────────────────┘

ISP_PAGE_SIZE  = 4KB  (1 << 12)
ISP_L1PT_PTES  = 1024
ISP_L2PT_PTES  = 1024
Address space  = 4GB  (32-bit)
```

主要操作：

```
tlb_invalidate(mmu)
    └── 對每個 MMU 單元：
        ├── 讀取 REG_L1_PHYS（硬體錯誤的替代方案）
        ├── 寫入 0xFFFFFFFF 到 REG_TLB_INVALIDATE
        └── wmb()（寫入記憶體屏障）

map_single(mmu_info, ptr)
    └── dma_map_single() 用於頁表項目

get_dummy_page(mmu_info)
    └── 為未映射的 IOVA 區域分配零化頁面
        └── 所有 L2 項目最初指向 dummy page
```

MMU 設定因硬體版本而異。ISYS 使用 3 個 MMU 單元，PSYS 使用 4 個 MMU 單元。每個 MMU 具有可設定的 L1/L2 串流快取，並支援 ZLW（Zero Length Write）預取。

### 8.7 Buttress（電源與安全）

**原始碼**：`ipu6-buttress.c`

Buttress 是 IPU 與 SoC 其餘部分之間的介面。它負責電源管理、CSE IPC（用於韌體認證）以及中斷路由。

```
┌──────────────────────────────────────────────────────────┐
│                    Buttress                                │
│                                                          │
│  ┌─────────────────┐  ┌────────────────────────────────┐ │
│  │ Power Control   │  │ CSE IPC                         │ │
│  │                 │  │ (Converged Security Engine)     │ │
│  │ buttress_power()│  │                                │ │
│  │ - ISYS power    │  │ ipc_reset() → ipc_open()      │ │
│  │ - PSYS power    │  │ ipc_send() / ipc_recv()       │ │
│  │                 │  │                                │ │
│  │ Timeout: 200ms  │  │ Firmware authentication:       │ │
│  └─────────────────┘  │ - Bootload: 5s timeout        │ │
│                       │ - Authenticate: 10s timeout    │ │
│  ┌─────────────────┐  │ - FW reset: 100ms timeout     │ │
│  │ IRQ Routing     │  └────────────────────────────────┘ │
│  │                 │                                      │
│  │ BUTTRESS_ISR_   │  ┌────────────────────────────────┐ │
│  │ IS_IRQ → ISYS   │  │ TSC Synchronization            │ │
│  │ PS_IRQ → PSYS   │  │                                │ │
│  │                 │  │ Syncs IPU timestamp counter    │ │
│  └─────────────────┘  │ with system TSC                │ │
│                       │ Max 10 retries, 5ms timeout    │ │
│                       └────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

IPC 重設協定（CSE 通訊）：

```
Phase 1: 設定 ENTRY → 等待 ENTRY|EXIT 回應
Phase 2: 清除 ENTRY|EXIT → 設定 QUERY
Phase 3: 等待 EXIT → 清除全部 → 設定 EXIT
    └── 成功：IPC 通道就緒
    └── 最多重試 2000 次（每次迭代 400-500us）
```

### 8.8 Auxiliary Bus 裝置模型

**原始碼**：`ipu6-bus.c`

ISYS 和 PSYS 作為 auxiliary bus 裝置註冊，各自擁有自己的 PM domain：

```
PCI Device (ipu6)
    │
    ├── Auxiliary Device: "intel-ipu6.isys"
    │   ├── PM domain: ipu6_bus_pm_domain
    │   │   ├── runtime_suspend → buttress_power(ISYS, false)
    │   │   └── runtime_resume  → buttress_power(ISYS, true)
    │   └── Platform data: ipu6_isys_pdata
    │
    └── Auxiliary Device: "intel-ipu6.psys"
        ├── PM domain: ipu6_bus_pm_domain
        │   ├── runtime_suspend → buttress_power(PSYS, false)
        │   └── runtime_resume  → buttress_power(PSYS, true)
        └── Platform data: ipu6_psys_pdata
```

### 8.9 CPD 韌體容器

**原始碼**：`ipu6-cpd.c`

IPU 韌體使用 CPD（Code Partition Directory）容器格式：

```
┌────────────────────────────────────────┐
│  CPD Header ($CPD)                     │
│  ├── hdr_mark: 0x44504324              │
│  ├── ent_cnt: number of entries        │
│  └── hdr_len: header length            │
├────────────────────────────────────────┤
│  CPD Entry [0] - Manifest              │
│  CPD Entry [1] - Metadata              │
│  CPD Entry [2] - Module Data           │
├────────────────────────────────────────┤
│                                        │
│  Module Data contains:                 │
│  ├── Package Directory (_IUPKDR_)      │
│  │   ├── Header: PKG_DIR_HDR_MARK     │
│  │   ├── Entry count                   │
│  │   └── Per-component entries:        │
│  │       ├── DMA address (offset)      │
│  │       ├── Component ID              │
│  │       ├── Version                   │
│  │       └── Size                      │
│  │                                     │
│  └── Firmware binary data              │
│                                        │
└────────────────────────────────────────┘
```

---

## 9. V4L2 與 Media Controller 整合

### 9.1 Media Controller 拓撲

ISYS 驅動程式建立以下 media controller 拓撲：

```
                        Media Device: "ipu6"
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
  ┌─────▼──────┐       ┌─────▼──────┐       ┌─────▼──────┐
  │ Sensor 0   │       │ Sensor 1   │       │ Sensor N   │
  │ (i2c subdev)│       │ (i2c subdev)│       │            │
  │ [src pad 0]│       │ [src pad 0]│       │            │
  └─────┬──────┘       └─────┬──────┘       └────────────┘
        │ IMMUTABLE          │
  ┌─────▼──────┐       ┌─────▼──────┐
  │ CSI-2 Rx 0 │       │ CSI-2 Rx 1 │
  │ (subdev)   │       │ (subdev)   │
  │ [sink 0]   │       │ [sink 0]   │
  │ [src 1]    │       │ [src 1]    │
  │ [src 2]    │       │ [src 2]    │
  └──┬─────┬───┘       └──┬─────┬───┘
     │     │               │     │
┌────▼──┐┌─▼───────┐ ┌────▼──┐┌─▼───────┐
│Capture││Capture  │ │Capture││Capture  │
│ 0     ││ 1      │ │ 2     ││ 3      │
│(video)││(video) │ │(video)││(video) │
│(/dev/ ││(/dev/  │ │(/dev/ ││(/dev/  │
│video0)││video1) │ │video2)││video3) │
└───────┘└────────┘ └───────┘└────────┘
```

每個 CSI-2 連接埠建立 `NR_OF_CSI2_SRC_PADS` 個視訊擷取裝置：
- 一個用於像素資料 (V4L2_BUF_TYPE_VIDEO_CAPTURE)
- 一個用於中繼資料 (V4L2_BUF_TYPE_META_CAPTURE)

### 9.2 Media Link 建立

**原始碼**：`ipu6-isys.c:197`

```c
// CSI-2 subdevice → Video capture device 連結
for (i = 0; i < csi2_pdata->nports; i++) {
    for (j = 0; j < NR_OF_CSI2_SRC_PADS; j++) {
        struct ipu6_isys_video *av = &isys->csi2[i].av[j];
        media_create_pad_link(sd, CSI2_PAD_SRC + j,
                              &av->vdev.entity, 0, 0);
        av->csi2 = &isys->csi2[i];
    }
}

// Sensor → CSI-2 subdevice 連結（在 notifier bound 回呼中）
media_create_pad_link(&sd->entity, src_pad_idx,
                      &isys->csi2[csi2->port].asd.sd.entity, 0,
                      MEDIA_LNK_FL_ENABLED | MEDIA_LNK_FL_IMMUTABLE);
```

### 9.3 V4L2 視訊裝置操作

**原始碼**：`ipu6-isys-video.c`

視訊裝置註冊：

```c
// 裝置命名："Intel IPU6 ISYS Capture N"
snprintf(av->vdev.name, sizeof(av->vdev.name),
         IPU6_ISYS_ENTITY_PREFIX " ISYS Capture %u",
         i * NR_OF_CSI2_SRC_PADS + j);
```

V4L2 ioctl 操作：

```
VIDIOC_QUERYCAP      → ipu6_isys_vidioc_querycap
VIDIOC_ENUM_FMT      → ipu6_isys_vidioc_enum_fmt
VIDIOC_ENUM_FRAMESIZES → ipu6_isys_vidioc_enum_framesizes
VIDIOC_G_FMT         → 標準 V4L2 格式取得
VIDIOC_S_FMT         → 標準 V4L2 格式設定
VIDIOC_REQBUFS       → VB2 queue_setup
VIDIOC_QBUF          → VB2 queue/prepare
VIDIOC_DQBUF         → VB2 dequeue
VIDIOC_STREAMON      → stream_start（韌體 STREAM_OPEN + START）
VIDIOC_STREAMOFF     → stream_stop（韌體 STOP + CLOSE）
```

### 9.4 CSI-2 接收器

**原始碼**：`ipu6-isys-csi2.c`

CSI-2 接收器處理 MIPI CSI-2 協定，具備：

- **支援的匯流排格式**：RGB565、RGB888、UYVY、YUYV、Bayer 8/10/12、META 8/10/12/16/24
- **時序計算**基於鏈路頻率：
  ```
  register_value = (A/1e9 + B * UI) / COUNT_ACC
  where UI = 1 / (2 * F), COUNT_ACC = 0.125ns
  ```
- **錯誤偵測**：20 種錯誤類型，包括：
  - 單一/多重封包標頭錯誤
  - 負載 CRC 錯誤
  - FIFO 溢位
  - 影格/行同步錯誤
  - DPHY 致命錯誤
  - SOT 同步錯誤

PHY 電源控制支援三種 PHY 類型：
- `ipu6_isys_mcd_phy_set_power()` - MCD PHY (Tiger Lake)
- `ipu6_isys_dwc_phy_set_power()` - DWC PHY (Meteor Lake+)
- `ipu6_isys_jsl_phy_set_power()` - JSL PHY (Jasper Lake)

### 9.5 緩衝區佇列管理

**原始碼**：`ipu6-isys-queue.c`

驅動程式使用 VB2（Video Buffer 2）框架搭配 DMA-SG 記憶體：

```
緩衝區生命週期：

  VIDIOC_REQBUFS → queue_setup()
      └── 計算緩衝區大小：bytesperline × height
      └── 單一平面，*num_planes = 1

  VIDIOC_QBUF → buf_init() + buf_prepare()
      └── buf_init():
          ├── vb2_dma_sg_plane_desc() → 取得 SG 表
          ├── ipu6_dma_map_sgtable()  → 透過 IPU MMU 映射
          └── ivb->dma_addr = sg_dma_address()  → 韌體用的 IOVA
      └── buf_prepare():
          └── vb2_set_plane_payload(bytesperline * height)

  buf_queue() → 加入 incoming 佇列
      └── buffer_list_get() 從每個活躍視訊節點收集一個緩衝區
      └── 以 ipu6_fw_isys_frame_buff_set_abi 提交給韌體

  韌體完成 → ISR → buf_done()
      └── vb2_buffer_done(VB2_BUF_STATE_DONE)
```

多輸出擷取的緩衝區列表管理：

```
buffer_list_get(stream, bl)
    │
    ├── 對串流中的每個佇列 stream->queues：
    │   ├── 鎖定 aq->lock
    │   ├── 從 aq->incoming 取得第一個緩衝區
    │   ├── 加入緩衝區列表 bl->head
    │   └── 遞增 bl->nbufs
    │
    └── 若任何佇列為空：
        └── 將所有已收集的緩衝區歸還至其 incoming 佇列
```

### 9.6 透過 ipu-bridge 進行感測器發現

**原始碼**：引用自 `ipu6-isys.c:688`

ipu-bridge 模組從 ACPI 表中發現攝影機感測器，並為其建立軟體節點：

```c
// 在 ISYS 初始化期間呼叫
ret = ipu_bridge_init(dev, ipu_bridge_parse_ssdb);
```

bridge 的功能：
1. 掃描 ACPI 命名空間尋找攝影機感測器裝置
2. 解析 SSDB（Sensor Specific Data Block）以取得 CSI-2 設定
3. 為每個感測器建立 `software_node` 項目
4. 設定 `fwnode` 連結：感測器 → CSI-2 連接埠
5. 若存在 VCM（Voice Coil Motor），則實例化 VCM i2c 裝置

---

## 10. 韌體通訊層

### 10.1 Syscom 訊息佇列

**原始碼**：`ipu6-fw-com.h`

驅動程式透過系統通訊（syscom）訊息佇列與 IPU 韌體進行通訊：

```
┌─────────────────────────────────────────────────────────┐
│              Firmware Communication                      │
│                                                          │
│   Driver                              Firmware           │
│  ┌──────────────┐                  ┌──────────────┐     │
│  │              │   Send Queue     │              │     │
│  │  Send Token  ├─────────────────►│  Recv Token  │     │
│  │  (commands)  │                  │  (commands)  │     │
│  │              │                  │              │     │
│  │  Recv Token  │◄─────────────────┤  Send Token  │     │
│  │  (responses) │   Recv Queue     │  (responses) │     │
│  └──────────────┘                  └──────────────┘     │
│                                                          │
│  佇列設定：                                              │
│  ├── 傳送佇列：40 個項目（發送至韌體的命令）             │
│  ├── 接收佇列：40 個項目（來自韌體的回應）               │
│  ├── Proxy 傳送：5 個項目（暫存器存取）                  │
│  └── Proxy 接收：5 個項目（暫存器存取回應）              │
└──────────────────────────────────────────────────────────┘
```

API：

```c
// 生命週期
int  ipu6_fw_com_prepare(config, ...);
int  ipu6_fw_com_open(fwcom);
bool ipu6_fw_com_ready(fwcom);
int  ipu6_fw_com_close(fwcom);
int  ipu6_fw_com_release(fwcom, force);

// 傳送（驅動程式 → 韌體）
void *ipu6_send_get_token(fwcom, stream_id);
void  ipu6_send_put_token(fwcom, stream_id);

// 接收（韌體 → 驅動程式）
void *ipu6_recv_get_token(fwcom, stream_id);
void  ipu6_recv_put_token(fwcom, stream_id);
```

### 10.2 韌體命令類型

**原始碼**：`ipu6-fw-isys.h`

```c
enum ipu6_fw_isys_send_type {
    IPU6_FW_ISYS_SEND_TYPE_STREAM_OPEN     = 0,
    IPU6_FW_ISYS_SEND_TYPE_STREAM_START    = 1,
    IPU6_FW_ISYS_SEND_TYPE_STREAM_CAPTURE  = 2,
    IPU6_FW_ISYS_SEND_TYPE_STREAM_STOP     = 3,
    IPU6_FW_ISYS_SEND_TYPE_STREAM_FLUSH    = 4,
    IPU6_FW_ISYS_SEND_TYPE_STREAM_CLOSE    = 5,
};
```

### 10.3 韌體回應類型

```c
enum ipu6_fw_isys_resp_type {
    IPU6_FW_ISYS_RESP_TYPE_STREAM_OPEN_DONE    = 0,
    IPU6_FW_ISYS_RESP_TYPE_STREAM_START_ACK    = 1,
    IPU6_FW_ISYS_RESP_TYPE_STREAM_START_AND_CAPTURE_ACK = 2,
    IPU6_FW_ISYS_RESP_TYPE_STREAM_CAPTURE_ACK  = 3,
    IPU6_FW_ISYS_RESP_TYPE_STREAM_STOP_ACK     = 4,
    IPU6_FW_ISYS_RESP_TYPE_STREAM_FLUSH_ACK    = 5,
    IPU6_FW_ISYS_RESP_TYPE_STREAM_CLOSE_ACK    = 6,
    IPU6_FW_ISYS_RESP_TYPE_PIN_DATA_READY      = 7,
    IPU6_FW_ISYS_RESP_TYPE_FRAME_SOF           = 8,
    IPU6_FW_ISYS_RESP_TYPE_FRAME_EOF           = 9,
    IPU6_FW_ISYS_RESP_TYPE_STATS_DATA_READY    = 10,
};
```

### 10.4 串流設定 ABI

**原始碼**：`ipu6-fw-isys.h`

```c
struct ipu6_fw_isys_stream_cfg_data_abi {
    struct ipu6_fw_isys_input_pin_info_abi input_pins[MAX_IPINS]; // 最多 4 個
    struct ipu6_fw_isys_output_pin_info_abi output_pins[MAX_OPINS]; // 最多 5 個

    u32 nof_input_pins;
    u32 nof_output_pins;

    struct ipu6_fw_isys_cropping_abi crop;

    u8 send_irq_sof_discarded;
    u8 send_irq_eof_discarded;
    u8 send_resp_sof_discarded;
    u8 send_resp_eof_discarded;

    struct ipu6_fw_isys_linked_receiver_abi linked_receiver;
    u8 src;          // 來源索引（CSI-2 連接埠）
    u8 vc;           // 虛擬通道
    u8 isl_use;
    u8 sensor_type;

    u8 compfmt;
    u8 stream_handle;
};
```

### 10.5 擷取請求 ABI

```c
struct ipu6_fw_isys_frame_buff_set_abi {
    struct ipu6_fw_isys_output_pin_payload_abi output_pins[MAX_OPINS];

    u8 send_irq_sof;
    u8 send_irq_eof;
    u8 send_irq_capture_ack;
    u8 send_irq_capture_done;

    u8 send_resp_sof;
    u8 send_resp_eof;
    u8 send_resp_capture_ack;
    u8 send_resp_capture_done;
};
```

### 10.6 韌體回應

```c
struct ipu6_fw_isys_resp_info_abi {
    u64 buf_id;
    u32 pin_id;
    struct ipu6_fw_isys_output_pin_info_abi pin;

    u32 process_group_light_id;
    u64 timestamp[2];    // 開始與結束時間戳
    u32 error;
    u32 error_details;

    enum ipu6_fw_isys_resp_type type;
    u8 stream_handle;
    u16 frame_counter;
};
```

### 10.7 命令/回應流程

```
串流開啟：
  Driver                                     Firmware
    │                                           │
    │  ipu6_send_get_token()                    │
    │  填寫 stream_cfg_data_abi：               │
    │    - input pins（感測器格式、CSI）        │
    │    - output pins（記憶體格式、stride）    │
    │    - 來源連接埠、虛擬通道                │
    │  ipu6_send_put_token()                    │
    │  ─────────────────────────────────────────►
    │                                           │
    │  wait_for_completion(stream_open)         │
    │  ◄─────────────────────────────────────────
    │  STREAM_OPEN_DONE 回應                    │
    │                                           │

擷取請求：
    │  ipu6_send_get_token()                    │
    │  填寫 frame_buff_set_abi：                │
    │    - output pin payloads（DMA 位址）      │
    │    - IRQ 控制旗標                         │
    │  ipu6_send_put_token()                    │
    │  ─────────────────────────────────────────►
    │                                           │
    │  ISR: PIN_DATA_READY                      │
    │  ◄─────────────────────────────────────────
    │  回應包含：                               │
    │    - buf_id、pin_id、timestamps           │
    │    - 錯誤狀態                             │
    │  vb2_buffer_done() → 回傳至使用者空間    │
```

---

## 11. 完整資料流程：從應用程式到硬體

### 11.1 Intel IPU7 HAL：完整擷取路徑

本節追蹤單一影格擷取從 Android 應用程式一路到硬體再返回的完整過程。

```
┌──────────────────────────────────────────────────────────────────────┐
│                    完整擷取資料流程                                    │
│                    (Intel IPU7 Camera HAL)                           │
│                                                                      │
│  ① 應用程式：CameraCaptureSession.capture(request)                   │
│     │                                                                │
│  ② Binder IPC → CameraService                                       │
│     │                                                                │
│  ③ Camera3Device::processCaptureRequest()                            │
│     │  - 驗證請求                                                    │
│     │  - 取得輸出緩衝區控制代碼                                      │
│     │  - 透過 ICameraDeviceSession 提交至 HAL                        │
│     │                                                                │
│  ④ AIDL IPC → Intel IPU7 HAL                                        │
│     │                                                                │
│  ⑤ RequestThread::processRequest()                                   │
│     │  - 從 CameraContext 取得 DataContext                            │
│     │  - 解析擷取設定（AE 模式、AF 模式等）                          │
│     │  - 將請求加入 mPendingRequests 佇列                            │
│     │                                                                │
│  ⑥ RequestThread::threadLoop()                                       │
│     │  - 等待觸發事件（NEW_REQUEST + NEW_SOF/STATS）                 │
│     │  - fetchNextRequest()                                          │
│     │  - handleRequest()：                                           │
│     │    │                                                           │
│     │    ├── AiqUnit::run3A(ccaId, applyingSeq, frameNumber)         │
│     │    │   │                                                       │
│     │    │   ├── IPAClient::runAec() → AE 參數                      │
│     │    │   │   └── → IPAServer → CcaWorker → Intel CCA lib        │
│     │    │   │       └── 回傳：曝光、增益、閃光燈                    │
│     │    │   │                                                       │
│     │    │   ├── IPAClient::runAiq() → AWB + ISP 參數                │
│     │    │   │   └── → IPAServer → CcaWorker → Intel CCA lib        │
│     │    │   │       └── 回傳：CCM、gamma、tone map                  │
│     │    │   │                                                       │
│     │    │   └── IPAClient::runAic() → ISP 韌體設定                  │
│     │    │       └── → IPAServer → CcaWorker → Intel CCA lib        │
│     │    │           └── 回傳：PSYS 用的 PAL 二進位檔               │
│     │    │                                                           │
│     │    └── 將 3A 結果套用至感測器硬體                              │
│     │        └── SensorHwCtrl：透過 I2C 設定曝光/增益                │
│     │                                                                │
│  ⑦ CaptureUnit::qbuf(port, cameraBuffer)                            │
│     │  - 將 CameraBuffer 轉換為 V4L2 buffer                         │
│     │  - DeviceBase::queueBuffer(sequence)                           │
│     │    └── ioctl(VIDIOC_QBUF) → 核心 V4L2 層                     │
│     │                                                                │
│  ⑧ 核心：ipu6_isys_buf_queue()                                      │
│     │  - ipu6_dma_map_sgtable() → 透過 IPU MMU 映射                 │
│     │  - buffer_list_get() → 收集緩衝區集合                         │
│     │  - ipu6_fw_isys_send_capture()：                               │
│     │    │  ipu6_send_get_token(fwcom, stream_id)                    │
│     │    │  填寫 frame_buff_set_abi：                                │
│     │    │    - output_pins[].addr = DMA 映射的 IOVA                 │
│     │    │    - send_irq_sof/eof = 1                                 │
│     │    │  ipu6_send_put_token(fwcom, stream_id)                    │
│     │    └── 命令寫入 syscom 傳送佇列                                │
│     │                                                                │
│  ⑨ IPU 韌體：                                                       │
│     │  - 從 syscom 佇列讀取命令                                      │
│     │  - 設定 CSI-2 接收器 DMA                                       │
│     │  - 等待感測器的下一個影格                                      │
│     │                                                                │
│  ⑩ MIPI CSI-2 硬體：                                                │
│     │  - 感測器透過 CSI-2 通道傳輸影格資料                           │
│     │  - CSI-2 接收器反序列化資料                                    │
│     │  - 資料流經 ISYS 像素緩衝區 (SRAM)                             │
│     │  - DMA 將影格寫入系統記憶體（IOVA 位址）                       │
│     │                                                                │
│  ⑪ IPU 韌體 → 驅動程式：                                            │
│     │  - SOF 回應 → isys_isr() → SOF 事件傳至 HAL                   │
│     │  - PIN_DATA_READY 回應帶有時間戳                               │
│     │                                                                │
│  ⑫ 核心 ISR：                                                       │
│     │  - isys_isr() → isys_isr_one()                                │
│     │  - ipu6_recv_get_token() → 解析韌體回應                       │
│     │  - vb2_buffer_done(VB2_BUF_STATE_DONE)                        │
│     │  - V4L2 poll 喚醒 → 通知使用者空間                            │
│     │                                                                │
│  ⑬ CaptureUnit::poll() → dequeueBuffer()                            │
│     │  - ioctl(VIDIOC_DQBUF) → 原始影格可用                         │
│     │  - 通知 BufferConsumers (ProcessingUnit)                       │
│     │                                                                │
│  ⑭ ProcessingUnit::processNewFrame()                                 │
│     │  - prepareTask()：                                             │
│     │    ├── 從 CaptureUnit 取得原始來源緩衝區                       │
│     │    ├── 取得輸出目標緩衝區（YUV/JPEG）                          │
│     │    └── 從 AiqResultStorage 取得 ISP 設定                       │
│     │  - dispatchTask()：                                            │
│     │    ├── PipeManager → PSysDevice::addTask()                     │
│     │    │   └── ioctl() → PSYS 核心裝置                             │
│     │    │       └── IPU 韌體處理 raw→YUV                            │
│     │    │                                                           │
│     │    └── 等待任務完成                                            │
│     │        └── PSysDevice::poll() → bufferDone()                   │
│     │                                                                │
│  ⑮ ProcessingUnit::onTaskDone()                                      │
│     │  - onBufferDone() → CameraStream::onBufferAvailable()          │
│     │  - onMetadataReady() → 填寫 CaptureResult 中繼資料            │
│     │                                                                │
│  ⑯ CameraStream → AIDL 回呼                                         │
│     │  - ICameraDeviceCallback::processCaptureResult()               │
│     │  - ICameraDeviceCallback::notify(SHUTTER)                      │
│     │                                                                │
│  ⑰ CameraService → 應用程式                                         │
│     │  - Camera3Device 分發結果                                      │
│     │  - Binder 回呼至應用程式                                       │
│     │                                                                │
│  ⑱ 應用程式接收：                                                    │
│     - onCaptureStarted(timestamp)                                    │
│     - onCaptureCompleted(result metadata)                            │
│     - 影像可在輸出 Surface/ImageReader 中取得                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 11.2 Google Desktop HAL：USB 擷取路徑

```
┌──────────────────────────────────────────────────────────────────────┐
│                    USB 攝影機擷取資料流程                              │
│                    (Google Desktop Camera HAL)                       │
│                                                                      │
│  ① 應用程式：CameraCaptureSession.capture(request)                   │
│     │                                                                │
│  ② Binder IPC → CameraService → Camera3Device                       │
│     │                                                                │
│  ③ AIDL IPC → Google Desktop HAL（Rust 行程）                       │
│     │                                                                │
│  ④ Session：將擷取請求加入佇列                                       │
│     │                                                                │
│  ⑤ Worker 執行緒：                                                   │
│     │  - crossbeam channel 接收 Capture 任務                         │
│     │  - V4L2CaptureDevice::dequeue_buffer()                        │
│     │    └── ioctl(VIDIOC_DQBUF) → 來自 UVC 的 MJPEG/YUYV 影格     │
│     │                                                                │
│  ⑥ 影格處理（在 Worker 中）：                                        │
│     │  - MJPEG 解碼 → NV12/YUV                                      │
│     │  - 必要時進行格式轉換                                          │
│     │  - 為 BLOB 串流進行 JPEG 編碼                                  │
│     │  - 複製到輸出 Gralloc 緩衝區                                   │
│     │                                                                │
│  ⑦ V4L2CaptureDevice::queue_buffer()                                │
│     │  └── ioctl(VIDIOC_QBUF) → 將緩衝區歸還至 UVC 驅動程式        │
│     │                                                                │
│  ⑧ 透過 AIDL 進行結果回呼                                           │
│     │  - processCaptureResult(已填寫的緩衝區 + 中繼資料)             │
│     │  - notify(SHUTTER, timestamp)                                  │
│     │                                                                │
│  ⑨ CameraService → 應用程式                                         │
│     - onCaptureCompleted / 影像可用                                  │
└──────────────────────────────────────────────────────────────────────┘
```

### 11.3 時序與延遲考量

```
Sensor          ISYS            3A              PSYS            App
  │               │               │               │               │
  │ Frame N       │               │               │               │
  ├──SOF─────────►│               │               │               │
  │               │ SOF event     │               │               │
  │               ├──────────────►│               │               │
  │               │               │ run3A(N)      │               │
  │               │               │ (for N+1)     │               │
  │ pixel data    │               │               │               │
  ├──────────────►│               │               │               │
  │               │ DMA to DRAM   │               │               │
  │               │               │               │               │
  ├──EOF─────────►│               │               │               │
  │               │ PIN_DATA_RDY  │               │               │
  │               ├───────────────┼──────────────►│               │
  │               │               │               │ Process raw   │
  │               │               │               │ → YUV/JPEG    │
  │               │               │               │               │
  │               │               │               │ Task done     │
  │               │               │               ├──────────────►│
  │               │               │               │  Result       │
  │               │               │               │               │

每影格的典型延遲：
  感測器曝光：10-33ms（30-100fps）
  ISYS 擷取：~1ms（DMA 傳輸）
  3A 處理：~5-10ms（平行執行）
  PSYS 處理：~5-15ms（ISP 管線）
  管線總計：3A 收斂需 2-3 個影格的延遲
```

---

## 12. 緩衝區管理與 DMA

### 12.1 緩衝區類型與配置

系統使用多種緩衝區管理策略：

```
┌──────────────────────────────────────────────────────────────────┐
│                    緩衝區管理層次                                   │
│                                                                  │
│  應用程式層：                                                     │
│  ┌──────────────────────────────────────────┐                   │
│  │ Gralloc Buffers（Android 圖形配置）      │                   │
│  │ - 由 ION/DMA-BUF heap 支援              │                   │
│  │ - 透過 NativeHandle（fd + metadata）共享 │                   │
│  └─────────────────┬────────────────────────┘                   │
│                    │ importBuffer()                               │
│  HAL 層：          │                                             │
│  ┌─────────────────▼────────────────────────┐                   │
│  │ CameraBuffer（icamera 內部）             │                   │
│  │ - 封裝 Gralloc 或內部配置               │                   │
│  │ - 追蹤 DMA 位址、序列號、連接埠        │                   │
│  └─────────────────┬────────────────────────┘                   │
│                    │ V4L2 QBUF/DQBUF                             │
│  核心層：          │                                             │
│  ┌─────────────────▼────────────────────────┐                   │
│  │ VB2 DMA-SG Buffers                       │                   │
│  │ - 用於實體頁面的 scatter-gather 列表    │                   │
│  │ - 透過 IPU MMU 映射供韌體使用           │                   │
│  │                                          │                   │
│  │ ipu6_isys_buf_init()：                   │                   │
│  │   sg = vb2_dma_sg_plane_desc(vb, 0)     │                   │
│  │   ipu6_dma_map_sgtable(adev, sg)        │                   │
│  │   ivb->dma_addr = sg_dma_address()      │                   │
│  │   （這是 IPU 位址空間中的 IOVA）        │                   │
│  └─────────────────┬────────────────────────┘                   │
│                    │ IOVA 位址                                    │
│  韌體：            │                                             │
│  ┌─────────────────▼────────────────────────┐                   │
│  │ 影格緩衝區 payload（DMA 目標）           │                   │
│  │ - 韌體將感測器資料 DMA 至 IOVA          │                   │
│  │ - IPU MMU 將 IOVA 轉換為實體位址        │                   │
│  └──────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
```

### 12.2 IPU DMA 操作

**原始碼**：`ipu6-dma.h`

IPU 使用自訂 DMA 操作，透過其內部 MMU 而非系統 IOMMU 進行路由：

```c
struct ipu6_dma_mapping {
    struct ipu6_mmu_info *mmu_info;    // IPU MMU 頁表
    struct iova_domain iovad;           // IOVA 配置器
};

// 為每個 auxiliary bus 裝置註冊的自訂 DMA 操作：
ipu6_dma_alloc()      // 配置 IOVA + 實體頁面
ipu6_dma_free()       // 釋放 IOVA + 實體頁面
ipu6_dma_mmap()       // 用於使用者空間存取的 mmap
ipu6_dma_map_sg()     // 透過 IPU MMU 映射 scatter-gather 列表
ipu6_dma_unmap_sg()   // 從 IPU MMU 解除映射
ipu6_dma_map_sgtable()   // 映射 SG 表（由 VB2 使用）
ipu6_dma_unmap_sgtable() // 解除映射 SG 表
```

### 12.3 PSYS 緩衝區註冊

對於 PSYS 處理，緩衝區透過 PSysDevice 向核心註冊：

```cpp
// HAL 向 PSYS 核心裝置註冊緩衝區
int PSysDevice::registerBuffer(TerminalBuffer* buf) {
    struct ipu_psys_buffer psysBuf;
    psysBuf.base.fd = buf->fd;      // DMA-BUF 檔案描述符
    psysBuf.base.flags = buf->flags;
    // ioctl(mFd, IPU_IOC_REGISTER_BUFFER, &psysBuf)
}

// 使用已註冊的緩衝區提交處理任務
int PSysDevice::addTask(const PSysTask& task) {
    // 為每個 terminal 填寫 ipu_psys_term_buffers
    // ioctl(mFd, IPU_IOC_TASK_ADD, taskBuffers)
}
```

---

## 13. 電源管理與效能

### 13.1 iWake 水位標記系統

**原始碼**：`ipu6-isys.c:531`

iWake 系統根據活躍串流的資料速率動態調整電源管理：

```
┌──────────────────────────────────────────────────────────────────┐
│                    iWake 電源管理                                   │
│                                                                  │
│  輸入：具有資料速率的活躍視訊串流                                  │
│                                                                  │
│  計算方式：                                                       │
│  1. 加總所有活躍串流的資料速率 (MB/s)                              │
│  2. calc_fill_time = max_sram_size / total_data_rate             │
│  3. LTR（Latency Tolerance Reporting）：                          │
│     - 增強模式：使用平台 LTR 值                                   │
│     - 傳統模式：基於填充時間的查閱表                               │
│  4. DID（Delay In Data）：                                        │
│     - 增強模式：fill_time × 90%                                   │
│     - 傳統模式：fill_time - LTR                                   │
│  5. iwake_threshold = DID × data_rate >> sram_granularity        │
│  6. critical_threshold = iwake + (buffer_pages - iwake) / 2      │
│                                                                  │
│  輸出：寫入 fabric 控制暫存器：                                    │
│  ┌─────────────────────────────────────────────┐                │
│  │ Fabric Control Register (0x68)               │                │
│  │ bits[9:0]   ltr_val   (LTR 值)             │                │
│  │ bits[12:10] ltr_scale (時間刻度)            │                │
│  │ bits[25:16] did_val   (DID 值)              │                │
│  │ bits[28:26] did_scale (時間刻度)            │                │
│  │ bit[30]     keep_power_in_D0                │                │
│  │ bit[31]     keep_power_override             │                │
│  └─────────────────────────────────────────────┘                │
│                                                                  │
│  韌體暫存器（透過 proxy token）：                                  │
│  - GDA_ENABLE_IWAKE_INDEX (2)                                    │
│  - GDA_IWAKE_THRESHOLD_INDEX (1)                                 │
│  - GDA_IRQ_CRITICAL_THRESHOLD_INDEX (0)                          │
│  - GDA_MEMOPEN_THRESHOLD_INDEX (3)                               │
│                                                                  │
│  電源狀態：                                                       │
│  ┌──────────────┬──────────────┬────────────────────────┐       │
│  │ 狀態         │ LTR/DID      │ 動作                    │       │
│  ├──────────────┼──────────────┼────────────────────────┤       │
│  │ ISYS 關閉    │ 1023 / 1023  │ 最大延遲容忍度          │       │
│  │ ISYS 開啟    │ 20 / 20      │ PKGC-2R 的低延遲       │       │
│  │ iWake 關閉   │ 20 / 20      │ 無活躍串流              │       │
│  │ iWake 開啟   │ 計算值       │ 有活躍串流              │       │
│  │ 增強模式     │ 平台值       │ 增強型 iWake 模式       │       │
│  └──────────────┴──────────────┴────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### 13.2 執行時期電源管理

```
Bus PM Domain 流程：

  應用程式開啟攝影機
      │
      ▼
  pm_runtime_get() 在 ISYS auxiliary device 上
      │
      ▼
  bus_pm_runtime_resume()
      ├── ipu6_buttress_power(ISYS, true)
      │   └── 寫入電源控制暫存器
      │   └── 輪詢電源確認（200ms 逾時）
      ├── pm_generic_runtime_resume()
      │   └── ISYS 驅動程式 probe/resume
      └── set_iwake_ltrdid(LTR_ISYS_ON)

  串流進行中...
      │
      ▼
  update_watermark_setting()（在每次串流啟動/停止時）

  應用程式關閉攝影機
      │
      ▼
  pm_runtime_put() 在 ISYS auxiliary device 上
      │
      ▼
  bus_pm_runtime_suspend()
      ├── pm_generic_runtime_suspend()
      ├── ipu6_buttress_power(ISYS, false)
      └── set_iwake_ltrdid(LTR_ISYS_OFF)
```

### 13.3 QoS 設定

```c
// PM QoS 防止 CPU 在擷取期間進入深度睡眠
#define ISYS_PM_QOS_VALUE 300  // 微秒

// 在串流啟動時設定：
cpu_latency_qos_add_request(&isys->pm_qos, ISYS_PM_QOS_VALUE);

// 在串流停止時移除：
cpu_latency_qos_remove_request(&isys->pm_qos);
```

---

## 14. 關鍵檔案參考索引

### 14.1 Android Camera 框架

| 檔案 | 用途 |
|------|------|
| `frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.h` | HAL provider 發現與管理 |
| `frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp` | 攝影機裝置狀態機與請求路由 |
| `frameworks/av/camera/` | Camera2 API 原生實作 |

### 14.2 AIDL Camera 介面

| 檔案 | 用途 |
|------|------|
| `hardware/interfaces/camera/provider/aidl/ICameraProvider.aidl` | Provider 發現介面 |
| `hardware/interfaces/camera/device/aidl/ICameraDevice.aidl` | 裝置開啟/特性 |
| `hardware/interfaces/camera/device/aidl/ICameraDeviceSession.aidl` | 串流設定、擷取請求 |
| `hardware/interfaces/camera/device/aidl/ICameraDeviceCallback.aidl` | 結果/通知回呼 |

### 14.3 Google Desktop Camera HAL

| 檔案 | 用途 |
|------|------|
| `vendor/google/desktop/camera/hal_usb/src/v4l2_device.rs` | V4L2 擷取裝置封裝 |
| `vendor/google/desktop/camera/hal_usb/src/uvc_device.rs` | UVC 攝影機發現 |
| `vendor/google/desktop/camera/hal_usb/src/session/worker.rs` | 擷取 worker 執行緒 |
| `vendor/google/desktop/camera/hal_usb/src/session/stream.rs` | 串流管理 |
| `vendor/google/desktop/camera/hal_usb/src/session/capture_config.rs` | 擷取設定 |
| `vendor/google/desktop/camera/common/src/lib.rs` | 共享函式庫（AIDL、FMQ、格式） |

### 14.4 Intel IPU7 Camera HAL

| 檔案 | 用途 |
|------|------|
| `vendor/intel/camera/hal/ipu7/src/core/CaptureUnit.h` | ISYS 抽象層 (StreamSource) |
| `vendor/intel/camera/hal/ipu7/src/core/ProcessingUnit.h` | PSYS 抽象層（影像處理） |
| `vendor/intel/camera/hal/ipu7/src/core/PSysDevice.h` | PSYS 核心裝置介面 |
| `vendor/intel/camera/hal/ipu7/src/core/DeviceBase.h` | V4L2 裝置基底類別 |
| `vendor/intel/camera/hal/ipu7/src/core/CameraStream.h` | 應用程式串流 → 管線映射 |
| `vendor/intel/camera/hal/ipu7/src/core/RequestThread.h` | 請求佇列與 3A 編排 |
| `vendor/intel/camera/hal/ipu7/src/core/CameraContext.h` | 每攝影機狀態與結果儲存 |
| `vendor/intel/camera/hal/ipu7/src/core/CameraBuffer.h` | 內部緩衝區表示 |
| `vendor/intel/camera/hal/ipu7/src/core/IProcessingUnit.h` | 處理單元介面 |
| `vendor/intel/camera/hal/ipu7/src/core/StreamSource.h` | 串流來源 (BufferProducer) 介面 |
| `vendor/intel/camera/hal/ipu7/src/core/BufferQueue.h` | 生產者/消費者緩衝區佇列 |
| `vendor/intel/camera/hal/ipu7/src/core/SofSource.h` | Start-of-frame 事件來源 |
| `vendor/intel/camera/hal/ipu7/src/core/IspSettings.h` | ISP 參數儲存 |
| `vendor/intel/camera/hal/ipu7/src/core/IpuPacAdaptor.h` | PAC（Parameter Adaptation Controller） |
| `vendor/intel/camera/hal/ipu7/src/core/SensorHwCtrl.h` | 感測器 I2C 控制 |
| `vendor/intel/camera/hal/ipu7/src/core/LensHw.h` | VCM 鏡頭對焦控制 |
| `vendor/intel/camera/hal/ipu7/src/3a/AiqUnit.h` | 3A 演算法控制單元 |
| `vendor/intel/camera/hal/ipu7/ipa/IPAServer.h` | IPA 演算法伺服器 |
| `vendor/intel/camera/hal/ipu7/ipa/IPAHeader.h` | IPA 命令定義 |
| `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAClient.h` | IPA 客戶端（HAL 端） |
| `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAClientWorker.h` | IPA worker 執行緒 |
| `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAMemory.h` | 共享記憶體管理 |
| `vendor/intel/camera/hal/ipu7/src/v4l2/V4l2DeviceFactory.h` | V4L2 裝置建立 |

### 14.5 IPU 核心驅動程式

| 檔案 | 用途 |
|------|------|
| `drivers/media/pci/intel/ipu6/ipu6.c` | 主要 PCI 驅動程式、probe、SPC 設定 |
| `drivers/media/pci/intel/ipu6/ipu6.h` | 裝置結構、硬體變體、MMU 設定 |
| `drivers/media/pci/intel/ipu6/ipu6-isys.c` | ISYS 驅動程式：初始化、ISR、iWake、V4L2 註冊 |
| `drivers/media/pci/intel/ipu6/ipu6-isys.h` | ISYS 結構、串流設定、水位標記 |
| `drivers/media/pci/intel/ipu6/ipu6-isys-video.c` | V4L2 視訊裝置：格式、ioctl、串流控制 |
| `drivers/media/pci/intel/ipu6/ipu6-isys-video.h` | 視訊裝置與串流結構 |
| `drivers/media/pci/intel/ipu6/ipu6-isys-csi2.c` | CSI-2 接收器：時序、錯誤、PHY |
| `drivers/media/pci/intel/ipu6/ipu6-isys-csi2.h` | CSI-2 結構與子裝置 |
| `drivers/media/pci/intel/ipu6/ipu6-isys-queue.c` | VB2 緩衝區佇列：初始化、準備、排隊、完成 |
| `drivers/media/pci/intel/ipu6/ipu6-fw-isys.h` | 韌體 ABI：命令、回應、設定 |
| `drivers/media/pci/intel/ipu6/ipu6-fw-com.h` | 韌體 syscom：基於 token 的 IPC |
| `drivers/media/pci/intel/ipu6/ipu6-mmu.c` | MMU：頁表、TLB、IOVA |
| `drivers/media/pci/intel/ipu6/ipu6-buttress.c` | Buttress：電源、CSE IPC、TSC 同步 |
| `drivers/media/pci/intel/ipu6/ipu6-buttress.h` | Buttress 結構與暫存器定義 |
| `drivers/media/pci/intel/ipu6/ipu6-bus.c` | Auxiliary bus：裝置建立、PM domain |
| `drivers/media/pci/intel/ipu6/ipu6-bus.h` | Bus 裝置結構 |
| `drivers/media/pci/intel/ipu6/ipu6-cpd.c` | CPD 韌體：解析、驗證、pkg_dir |
| `drivers/media/pci/intel/ipu6/ipu6-cpd.h` | CPD 標頭與項目結構 |
| `drivers/media/pci/intel/ipu6/ipu6-dma.h` | IPU bus 的自訂 DMA 操作 |

### 14.6 關鍵常數參考

```
IPU6_ISYS_MAX_STREAMS        = 16    （最大韌體串流數）
IPU6_ISYS_SIZE_SEND_QUEUE    = 40    （命令佇列深度）
IPU6_ISYS_SIZE_RECV_QUEUE    = 40    （回應佇列深度）
IPU6_ISYS_MIN_WIDTH          = 2     （最小擷取寬度）
IPU6_ISYS_MAX_WIDTH          = 4672  （最大擷取寬度）
IPU6_ISYS_MIN_HEIGHT         = 1     （最小擷取高度）
IPU6_ISYS_MAX_HEIGHT         = 3416  （最大擷取高度）
IPU6_MAX_SRAM_SIZE           = 200KB （IPU6 像素緩衝區）
IPU6SE_MAX_SRAM_SIZE         = 96KB  （IPU6SE 像素緩衝區）
IPU6_DEVICE_GDA_NR_PAGES     = 128   （GDA 實體頁面數）
IPU6_MMU_MAX_DEVICES          = 4    （最大 MMU 單元數）
ISP_PAGE_SIZE                 = 4KB  （MMU 頁面大小）
ISP_L1PT_PTES                 = 1024 （L1 頁表項目數）
ISP_L2PT_PTES                 = 1024 （L2 頁表項目數）
MAX_NODE_NUM (PSYS)           = 5    （最大處理節點數）
MAX_LINK_NUM (PSYS)           = 10   （最大節點連結數）
MAX_TASK_NUM (PSYS)           = 8    （最大並行任務數）
MAX_TERMINAL_NUM (PSYS)       = 26   （每節點最大 terminal 數）
ISYS_PM_QOS_VALUE             = 300  （CPU 延遲 QoS，微秒）
BUTTRESS_POWER_TIMEOUT_US     = 200ms
BUTTRESS_CSE_BOOTLOAD_TIMEOUT = 5s
BUTTRESS_CSE_AUTHENTICATE_TIMEOUT = 10s
```

---

## 文件結尾

本文件涵蓋完整的 Android IPU7 攝影機架構，從 Camera2 應用程式 API、經過 CameraService、AIDL HAL 介面、兩種 HAL 實作（Google Desktop 和 Intel IPU7）、IPU 核心驅動程式、韌體通訊，到硬體暫存器程式設計。

相關文件請參閱：
- `alos_grub/ES9356_Integration_Technical_Document.md` - ES9356 觸控螢幕整合
- `alos_grub/PTL_GCS_RT712_RT1320_Audio_Fix.md` - 音訊子系統文件
