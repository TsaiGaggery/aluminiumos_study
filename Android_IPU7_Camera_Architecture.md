# Android IPU7 Camera Architecture - Complete Technical Document

## Document Information
- **Platform**: Intel Panther Lake (PTL) with IPU7
- **Android Version**: ALOS (Android Linux on x86)
- **Kernel**: 6.18-based with IPU6/IPU7 driver support
- **Date**: February 2026
- **Scope**: Full camera stack from Android app to hardware registers

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Android Camera2 API and CameraService](#3-android-camera2-api-and-cameraservice)
4. [AIDL Camera Provider/Device Interfaces](#4-aidl-camera-providerdevice-interfaces)
5. [Google Desktop Camera HAL (Rust/V4L2)](#5-google-desktop-camera-hal-rustv4l2)
6. [Intel IPU7 Camera HAL Architecture](#6-intel-ipu7-camera-hal-architecture)
7. [IPU7 3A Engine and IPA Subsystem](#7-ipu7-3a-engine-and-ipa-subsystem)
8. [IPU Kernel Driver Architecture](#8-ipu-kernel-driver-architecture)
9. [V4L2 and Media Controller Integration](#9-v4l2-and-media-controller-integration)
10. [Firmware Communication Layer](#10-firmware-communication-layer)
11. [Complete Data Flow: App to Hardware](#11-complete-data-flow-app-to-hardware)
12. [Buffer Management and DMA](#12-buffer-management-and-dma)
13. [Power Management and Performance](#13-power-management-and-performance)
14. [Key File Reference Index](#14-key-file-reference-index)

---

## 1. Executive Summary

This document provides a comprehensive technical analysis of the camera subsystem on
Intel Panther Lake (PTL) platforms running Android (ALOS). The camera pipeline spans
from the Android Camera2 application API down through the CameraService, AIDL HAL
interfaces, two distinct Camera HAL implementations (Google Desktop HAL and Intel
IPU7 HAL), the IPU kernel driver, and finally to the IPU hardware itself.

The system supports two Camera HAL paths:

1. **Google Desktop Camera HAL** - A Rust-based HAL for USB/UVC cameras that
   communicates directly with V4L2 video capture devices. This is intended for
   desktop-class USB webcams.

2. **Intel IPU7 Camera HAL** - A C++-based HAL for MIPI CSI-2 connected cameras
   using the Intel IPU7 Image Processing Unit. This HAL provides full ISP
   pipeline support including 3A algorithms (AE, AF, AWB), image processing
   (PSYS), and raw capture (ISYS).

Both HALs register as AIDL camera providers and are managed by the Android
CameraService's CameraProviderManager.

### High-Level Architecture Diagram

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

## 2. System Architecture Overview

### 2.1 Component Relationships

The Android camera stack consists of the following major layers, each with
distinct responsibilities:

| Layer | Component | Location | Language |
|-------|-----------|----------|----------|
| App | Camera2 API | android.hardware.camera2 | Java/Kotlin |
| Framework | CameraService | frameworks/av/services/camera/ | C++ |
| Interface | AIDL HAL | hardware/interfaces/camera/ | AIDL→C++ |
| HAL (USB) | Google Desktop HAL | vendor/google/desktop/camera/ | Rust |
| HAL (IPU) | Intel IPU7 HAL | vendor/intel/camera/hal/ipu7/ | C++ |
| Kernel | IPU Driver | drivers/media/pci/intel/ipu6/ | C |
| Firmware | IPU FW | intel/ipu/ipu6_fw.bin | Binary |
| Hardware | IPU7 + Sensor | PCIe + MIPI CSI-2 | Silicon |

### 2.2 Two HAL Architecture

The ALOS platform uniquely supports two simultaneous Camera HAL providers:

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

The CameraProviderManager in CameraService discovers both providers via the
AIDL service manager. Each provider enumerates its cameras (USB devices for
the Desktop HAL, MIPI sensors for the IPU7 HAL), and the combined camera
list is presented to applications.

### 2.3 Source Code Layout

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

## 3. Android Camera2 API and CameraService

### 3.1 CameraService Architecture

The CameraService is the central daemon managing all camera operations. It runs
in the system_server process and handles:

- Camera device enumeration and discovery
- HAL provider lifecycle management
- Request/result routing between apps and HALs
- Multi-client access control and eviction policies

**Key source**: `frameworks/av/services/camera/libcameraservice/`

### 3.2 CameraProviderManager

The CameraProviderManager discovers and manages all Camera HAL providers
registered as AIDL services.

**Source**: `frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.h`

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

The provider manager maintains an internal data structure mapping:

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

Camera3Device is the core class implementing the camera device state machine
and request/result pipeline.

**Source**: `frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp`

Key operations:

1. **initializeCommonLocked()** - Opens the HAL device, creates device session
2. **configureStreamsLocked()** - Sends stream configuration to HAL
3. **processCaptureRequest()** - Submits capture requests
4. **processCaptureResult()** - Receives results from HAL callback

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

### 3.4 Request Processing Pipeline

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

## 4. AIDL Camera Provider/Device Interfaces

### 4.1 Interface Hierarchy

The AIDL camera interfaces define the contract between CameraService and
Camera HAL implementations.

**Source**: `hardware/interfaces/camera/`

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

### 4.2 Key Data Types

#### Stream Configuration

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

#### Capture Request / Result

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

#### HAL Stream Response

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

#### Notification Messages

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

### 4.3 Session Lifecycle

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

### 5.1 Overview

The Google Desktop Camera HAL is a Rust-based implementation designed for
USB/UVC cameras. It uses the V4L2 API directly to capture frames from USB
webcams without requiring an ISP pipeline.

**Source**: `vendor/google/desktop/camera/`

### 5.2 Architecture

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

### 5.3 V4L2 Capture Device

**Source**: `vendor/google/desktop/camera/hal_usb/src/v4l2_device.rs`

The V4L2CaptureDevice wraps the Linux V4L2 video capture interface using the
`v4l2r` Rust crate. It handles:

- Device open/close via `/dev/videoN`
- Format negotiation (MJPEG, YUYV, NV12)
- Buffer allocation via MMAP or DMABUF
- Stream on/off via VIDIOC_STREAMON/STREAMOFF
- Buffer queue/dequeue via VIDIOC_QBUF/DQBUF

The device requires `VIDEO_CAPTURE` capability and uses single-plane buffers.

### 5.4 UVC Device Discovery

**Source**: `vendor/google/desktop/camera/hal_usb/src/uvc_device.rs`

The UVCDevice structure represents a discovered USB camera with:

```rust
struct UVCDevice {
    vendor_id: u16,      // USB vendor ID
    product_id: u16,     // USB product ID
    device_path: String, // e.g., "/dev/video0"
    characteristics: UVCCharacteristics,
}
```

Discovery scans `/sys/class/video4linux/` for video devices with
`VIDEO_CAPTURE` capability and matches them against known USB camera
descriptors.

### 5.5 Worker Thread Architecture

**Source**: `vendor/google/desktop/camera/hal_usb/src/session/worker.rs`

The Worker operates as a dedicated capture thread processing two types
of tasks:

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

The worker uses crossbeam channels for communication with the session
layer. DMA plane layouts are used for zero-copy buffer passing where
possible.

### 5.6 Common Library

**Source**: `vendor/google/desktop/camera/common/src/lib.rs`

The common library provides shared modules:

- **aidl** - AIDL type marshalling for Rust
- **fmq** - Fast Message Queue (FMQ) integration
- **format** - Pixel format conversion utilities
- **geometry** - Image dimension calculations
- **metadata** - Camera metadata handling

---

## 6. Intel IPU7 Camera HAL Architecture

### 6.1 Overview

The Intel IPU7 Camera HAL is a full-featured C++ Camera HAL for MIPI CSI-2
connected image sensors. It provides a complete imaging pipeline including
raw capture (ISYS), image signal processing (PSYS), and 3A algorithm
integration via Intel CCA (Camera Control Algorithm).

**Source**: `vendor/intel/camera/hal/ipu7/`

The HAL operates in the `icamera` namespace and uses a producer/consumer
buffer pipeline architecture.

### 6.2 Top-Level Architecture

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

### 6.3 CaptureUnit (ISYS Abstraction)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/CaptureUnit.h`

CaptureUnit abstracts the ISYS (Input System) function of the IPU. It is the
source of the imaging pipeline - it produces raw frames from the camera sensor.

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

Internal state machine:

```
CAPTURE_UNINIT → CAPTURE_INIT → CAPTURE_CONFIGURE → CAPTURE_START
                                                          │
                                                          ▼
                                                    CAPTURE_STOP
```

The CaptureUnit manages DeviceBase objects, one per V4L2 video node:

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

### 6.4 DeviceBase (V4L2 Device Wrapper)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/DeviceBase.h`

DeviceBase is the foundational class for V4L2 device interaction:

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

Buffer flow through DeviceBase:

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

MainDevice is the primary subclass for pixel-producing video nodes:

```cpp
class MainDevice : public DeviceBase {
    int createBufferPool(const stream_t& config);
    int onDequeueBuffer(shared_ptr<CameraBuffer> buffer);
    bool needQueueBack(shared_ptr<CameraBuffer> buffer);
};
```

### 6.5 ProcessingUnit (PSYS Abstraction)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/ProcessingUnit.h`

ProcessingUnit runs the Image Signal Processing algorithms in the PSYS
(Processing System). It consumes raw frames from CaptureUnit and produces
processed output frames (YUV, JPEG, etc.).

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

Key internal components:

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

Processing flow:

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

### 6.6 PSysDevice (PSYS Kernel Interface)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/PSysDevice.h`

PSysDevice provides the userspace interface to the IPU PSYS kernel device.
It operates with a graph-based processing model:

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

Key data structures:

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

The PSYS graph defines processing nodes connected by links. Each node has
terminals (input/output ports) with associated buffers. A task specifies
which node to execute and which buffers to use for a given frame sequence.

### 6.7 CameraStream (Application Stream)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/CameraStream.h`

CameraStream bridges the application stream to the internal pipeline:

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

Buffer flow from app to hardware:

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

### 6.8 RequestThread (Request Orchestration)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/RequestThread.h`

The RequestThread manages the request queue and coordinates 3A execution
with frame capture:

```cpp
class RequestThread : public Thread, public EventSource, public EventListener {
    int processRequest(int bufferNum, camera_buffer_t **ubuffer);
    int waitFrame(int streamId, camera_buffer_t **ubuffer);
    int wait1stRequestDone();
    int configure(const stream_config_t *streamList);
    void handleEvent(EventData eventData);
};
```

Event-driven processing model:

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

Sequence tracking fields:

```
mLastCcaId      - CCA algorithm instance ID
mLastEffectSeq  - Sequence where last 3A results took effect
mLastAppliedSeq - Sequence where last 3A results were applied
mLastSofSeq     - Last Start-of-Frame sequence received
mBlockRequest   - Blocks 2nd/3rd request until 1st 3A event
```

---

## 7. IPU7 3A Engine and IPA Subsystem

### 7.1 AiqUnit (3A Algorithm Control)

**Source**: `vendor/intel/camera/hal/ipu7/src/3a/AiqUnit.h`

The AiqUnit manages the Intel CCA (Camera Control Algorithm) library for
3A processing (Auto Exposure, Auto Focus, Auto White Balance).

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

AiqUnit state machine:

```
AIQ_UNIT_NOT_INIT → AIQ_UNIT_INIT → AIQ_UNIT_CONFIGURED
                                          │
                                    AIQ_UNIT_START
                                          │
                                    AIQ_UNIT_STOP
```

Internal components:

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

### 7.2 IPA Architecture (Image Processing Algorithm)

The IPA subsystem provides sandboxed 3A algorithm execution, separating the
algorithm library from the main HAL process for security.

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

**Source**: `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAClient.h`

The IPAClient is a singleton that communicates with the IPA server process
via the libcamera IPA proxy mechanism:

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

**Source**: `vendor/intel/camera/hal/ipu7/ipa/IPAServer.h`

The IPAServer runs in the IPA process (or sandboxed thread) and hosts
the actual CCA algorithm instances:

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

Each CcaWorker wraps an Intel CCA library instance and processes algorithm
requests for a specific camera and tuning mode combination.

### 7.5 3A Processing Flow

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

### 7.6 CameraContext (Per-Camera State)

**Source**: `vendor/intel/camera/hal/ipu7/src/core/CameraContext.h`

CameraContext is a singleton per camera ID that stores all per-frame state:

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

DataContext captures all per-frame parameters:

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

## 8. IPU Kernel Driver Architecture

### 8.1 Overview

The IPU kernel driver is located at:
`drivers/media/pci/intel/ipu6/`

Despite the "ipu6" naming, this driver supports IPU6 through IPU7 hardware
variants. The driver is a PCI device driver that creates two auxiliary bus
devices: ISYS (Input System) and PSYS (Processing System).

### 8.2 Driver Module Structure

```
drivers/media/pci/intel/ipu6/
│
├── ipu6.c / ipu6.h              # Main PCI driver, device init
│   └── ipu6_pci_probe()          # PCI probe entry point
│   └── ipu6_configure_spc()      # Signal Processing Cell config
│
├── ipu6-isys.c / ipu6-isys.h    # Input System (ISYS)
│   └── isys_isr()                # ISYS interrupt handler
│   └── isys_setup_hw()           # Hardware initialization
│   └── update_watermark_setting() # iWake power management
│
├── ipu6-isys-video.c / .h       # V4L2 video device nodes
│   └── ipu6_isys_pfmts[]        # Supported pixel formats
│   └── video_open/close/ioctl    # V4L2 file operations
│
├── ipu6-isys-csi2.c / .h        # CSI-2 receiver subdevice
│   └── CSI-2 timing calculation
│   └── DPHY/DWC PHY control
│   └── Error detection and reporting
│
├── ipu6-isys-queue.c             # VB2 buffer queue management
│   └── ipu6_isys_buf_init()      # DMA mapping for buffers
│   └── buffer_list_get()          # Collect buffers for capture
│
├── ipu6-fw-isys.h                # Firmware ABI definitions
│   └── Command/response enums
│   └── Stream config structures
│   └── Frame buffer structures
│
├── ipu6-fw-com.h / .c           # Firmware communication layer
│   └── Token-based message passing
│
├── ipu6-mmu.c                    # MMU (IOMMU) for IPU
│   └── Two-level page table (L1/L2)
│   └── TLB invalidation
│   └── IOVA management
│
├── ipu6-buttress.c / .h         # Buttress (power/security/IPC)
│   └── CSE IPC for firmware auth
│   └── Power state management
│   └── TSC synchronization
│
├── ipu6-bus.c / .h              # Auxiliary bus device management
│   └── PM domain for ISYS/PSYS
│
├── ipu6-cpd.c / .h             # CPD firmware parsing
│   └── Package directory parsing
│   └── Manifest/metadata extraction
│
└── ipu6-dma.h                   # DMA operations
    └── Custom DMA alloc/map for IPU bus
```

### 8.3 PCI Probe and Device Initialization

**Source**: `ipu6.c:ipu6_pci_probe()`

```
ipu6_pci_probe()
    │
    ├── 1. PCI setup
    │   ├── pci_enable_device()
    │   ├── pci_set_master()
    │   ├── pci_request_regions()
    │   └── pci_ioremap_bar()  → isp->base
    │
    ├── 2. Determine hardware version
    │   └── Based on PCI device ID:
    │       IPU6_VER_6      (0x9a19) - Tiger Lake
    │       IPU6_VER_6SE    (0x4e19) - Jasper Lake
    │       IPU6_VER_6EP    (0x465d) - Alder/Raptor Lake
    │       IPU6_VER_6EP_MTL(0x7d19) - Meteor Lake
    │
    ├── 3. Initialize internal platform data
    │   └── ipu6_internal_pdata_init()
    │       ├── Set CSI-2 port count (4-8 per version)
    │       ├── Set SRAM granularity and max size
    │       ├── Set LTR/memopen thresholds
    │       └── Configure enhanced iWake settings
    │
    ├── 4. Initialize buttress (power/IPC)
    │   └── ipu6_buttress_init()
    │       ├── Setup IPC channels to CSE
    │       ├── Register interrupt handler
    │       └── Initialize mutex/completion structures
    │
    ├── 5. Initialize ISYS
    │   └── ipu6_isys_init()
    │       ├── Configure ISYS MMU (3 MMU units)
    │       ├── ipu_bridge_init() → ACPI sensor enumeration
    │       ├── ipu6_bus_initialize_device("isys")
    │       └── ipu6_bus_add_device()
    │
    ├── 6. Initialize PSYS
    │   └── ipu6_psys_init()
    │       ├── Configure PSYS MMU (4 MMU units)
    │       ├── ipu6_bus_initialize_device("psys")
    │       └── ipu6_bus_add_device()
    │
    └── 7. Load and authenticate firmware
        └── Request firmware: intel/ipu/ipu6_fw.bin
            └── ipu6_cpd_parse()  # Parse CPD firmware container
```

### 8.4 ISYS (Input System) Architecture

**Source**: `ipu6-isys.c`, `ipu6-isys.h`

The ISYS captures raw image data from camera sensors via MIPI CSI-2.

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

**Source**: `ipu6-isys.h:127`

```c
struct ipu6_isys {
    struct media_device media_dev;     // Media controller device
    struct v4l2_device v4l2_dev;       // V4L2 device
    struct ipu6_bus_device *adev;      // Auxiliary bus device

    int power;                         // Power state
    spinlock_t power_lock;
    u32 isr_csi2_bits;                 // Active CSI-2 IRQ bits
    u32 csi2_rx_ctrl_cached;

    spinlock_t streams_lock;
    struct ipu6_isys_stream streams[16]; // Max 16 FW streams
    int streams_ref_count[16];

    void *fwcom;                       // Firmware communication handle
    u32 phy_termcal_val;               // PHY termination calibration

    bool need_reset;
    bool icache_prefetch;
    unsigned int ref_count;
    unsigned int stream_opened;

    struct mutex mutex;
    struct mutex stream_mutex;

    struct ipu6_isys_pdata *pdata;     // Platform data
    struct ipu6_isys_csi2 *csi2;       // CSI-2 receiver array

    struct isys_iwake_watermark iwake_watermark; // Power management
};
```

#### Stream Initialization

**Source**: `ipu6-isys.c:144`

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

Each stream has completion objects for firmware command synchronization.
The stream lifecycle follows: OPEN → START → CAPTURE (repeated) → STOP → CLOSE.

#### ISR (Interrupt Service Routine)

**Source**: `ipu6-isys.c:344`

The ISYS ISR handles two types of interrupts:

1. **CSI-2 interrupts** (per-port): Frame start (SOF), frame end (EOF), errors
2. **Software interrupts**: Firmware response ready

```
isys_isr(adev)
    │
    ├── Read ctrl0_status (CSI-2 IRQ status)
    ├── Read UNISPART IRQ status (SW IRQ)
    │
    ├── For each CSI-2 port with pending IRQ:
    │   └── ipu6_isys_csi2_isr(csi2)
    │       ├── Read per-VC status bits
    │       ├── FS_VC(i) → ipu6_isys_csi2_sof_event_by_stream()
    │       └── FE_VC(i) → ipu6_isys_csi2_eof_event_by_stream()
    │
    └── If SW IRQ pending:
        └── isys_isr_one(adev)
            └── Process firmware response from syscom queue
```

### 8.5 Supported Pixel Formats

**Source**: `ipu6-isys-video.c:39`

The driver supports these V4L2 pixel formats:

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

Resolution range: 2×1 to 4672×3416 (step: 2×2)
```

### 8.6 MMU (Memory Management Unit)

**Source**: `ipu6-mmu.c`

The IPU has its own IOMMU separate from the system IOMMU. It uses a
two-level page table:

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

Key operations:

```
tlb_invalidate(mmu)
    └── For each MMU unit:
        ├── Read REG_L1_PHYS (workaround for HW bug)
        ├── Write 0xFFFFFFFF to REG_TLB_INVALIDATE
        └── wmb() (write memory barrier)

map_single(mmu_info, ptr)
    └── dma_map_single() for page table entries

get_dummy_page(mmu_info)
    └── Allocates a zeroed page for unmapped IOVA regions
        └── All L2 entries initially point to dummy page
```

The MMU configuration varies by hardware version. ISYS uses 3 MMU units
and PSYS uses 4 MMU units. Each MMU has configurable L1/L2 stream caches
with ZLW (Zero Length Write) prefetch support.

### 8.7 Buttress (Power and Security)

**Source**: `ipu6-buttress.c`

The Buttress is the interface between the IPU and the rest of the SoC.
It handles power management, CSE IPC (for firmware authentication),
and interrupt routing.

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

IPC Reset protocol (CSE communication):

```
Phase 1: Set ENTRY → Wait for ENTRY|EXIT response
Phase 2: Clear ENTRY|EXIT → Set QUERY
Phase 3: Wait for EXIT → Clear all → Set EXIT
    └── Success: IPC channel ready
    └── Retry up to 2000 times (400-500us per iteration)
```

### 8.8 Auxiliary Bus Device Model

**Source**: `ipu6-bus.c`

ISYS and PSYS are registered as auxiliary bus devices with their own
PM domain:

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

### 8.9 CPD Firmware Container

**Source**: `ipu6-cpd.c`

The IPU firmware uses the CPD (Code Partition Directory) container format:

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

## 9. V4L2 and Media Controller Integration

### 9.1 Media Controller Topology

The ISYS driver creates the following media controller topology:

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

Each CSI-2 port creates `NR_OF_CSI2_SRC_PADS` video capture devices:
- One for pixel data (V4L2_BUF_TYPE_VIDEO_CAPTURE)
- One for metadata (V4L2_BUF_TYPE_META_CAPTURE)

### 9.2 Media Link Creation

**Source**: `ipu6-isys.c:197`

```c
// CSI-2 subdevice → Video capture device links
for (i = 0; i < csi2_pdata->nports; i++) {
    for (j = 0; j < NR_OF_CSI2_SRC_PADS; j++) {
        struct ipu6_isys_video *av = &isys->csi2[i].av[j];
        media_create_pad_link(sd, CSI2_PAD_SRC + j,
                              &av->vdev.entity, 0, 0);
        av->csi2 = &isys->csi2[i];
    }
}

// Sensor → CSI-2 subdevice link (in notifier bound callback)
media_create_pad_link(&sd->entity, src_pad_idx,
                      &isys->csi2[csi2->port].asd.sd.entity, 0,
                      MEDIA_LNK_FL_ENABLED | MEDIA_LNK_FL_IMMUTABLE);
```

### 9.3 V4L2 Video Device Operations

**Source**: `ipu6-isys-video.c`

Video device registration:

```c
// Device naming: "Intel IPU6 ISYS Capture N"
snprintf(av->vdev.name, sizeof(av->vdev.name),
         IPU6_ISYS_ENTITY_PREFIX " ISYS Capture %u",
         i * NR_OF_CSI2_SRC_PADS + j);
```

V4L2 ioctl operations:

```
VIDIOC_QUERYCAP      → ipu6_isys_vidioc_querycap
VIDIOC_ENUM_FMT      → ipu6_isys_vidioc_enum_fmt
VIDIOC_ENUM_FRAMESIZES → ipu6_isys_vidioc_enum_framesizes
VIDIOC_G_FMT         → standard V4L2 format get
VIDIOC_S_FMT         → standard V4L2 format set
VIDIOC_REQBUFS       → VB2 queue_setup
VIDIOC_QBUF          → VB2 queue/prepare
VIDIOC_DQBUF         → VB2 dequeue
VIDIOC_STREAMON      → stream_start (firmware STREAM_OPEN + START)
VIDIOC_STREAMOFF     → stream_stop (firmware STOP + CLOSE)
```

### 9.4 CSI-2 Receiver

**Source**: `ipu6-isys-csi2.c`

The CSI-2 receiver handles MIPI CSI-2 protocol with:

- **Supported bus formats**: RGB565, RGB888, UYVY, YUYV, Bayer 8/10/12,
  META 8/10/12/16/24
- **Timing calculation** based on link frequency:
  ```
  register_value = (A/1e9 + B * UI) / COUNT_ACC
  where UI = 1 / (2 * F), COUNT_ACC = 0.125ns
  ```
- **Error detection**: 20 error types including:
  - Single/multiple packet header errors
  - Payload CRC errors
  - FIFO overflow
  - Frame/line sync errors
  - DPHY fatal errors
  - SOT sync errors

PHY power control supports three PHY types:
- `ipu6_isys_mcd_phy_set_power()` - MCD PHY (Tiger Lake)
- `ipu6_isys_dwc_phy_set_power()` - DWC PHY (Meteor Lake+)
- `ipu6_isys_jsl_phy_set_power()` - JSL PHY (Jasper Lake)

### 9.5 Buffer Queue Management

**Source**: `ipu6-isys-queue.c`

The driver uses VB2 (Video Buffer 2) framework with DMA-SG memory:

```
Buffer lifecycle:

  VIDIOC_REQBUFS → queue_setup()
      └── Calculates buffer size: bytesperline × height
      └── Single plane, *num_planes = 1

  VIDIOC_QBUF → buf_init() + buf_prepare()
      └── buf_init():
          ├── vb2_dma_sg_plane_desc() → get SG table
          ├── ipu6_dma_map_sgtable()  → map through IPU MMU
          └── ivb->dma_addr = sg_dma_address()  → IOVA for firmware
      └── buf_prepare():
          └── vb2_set_plane_payload(bytesperline * height)

  buf_queue() → Adds to incoming queue
      └── buffer_list_get() collects one buffer per active video node
      └── Submits to firmware as ipu6_fw_isys_frame_buff_set_abi

  Firmware completion → ISR → buf_done()
      └── vb2_buffer_done(VB2_BUF_STATE_DONE)
```

Buffer list management for multi-output capture:

```
buffer_list_get(stream, bl)
    │
    ├── For each queue in stream->queues:
    │   ├── Lock aq->lock
    │   ├── Get first buffer from aq->incoming
    │   ├── Add to buffer list bl->head
    │   └── Increment bl->nbufs
    │
    └── If any queue is empty:
        └── Return all collected buffers to their incoming queues
```

### 9.6 Sensor Discovery via ipu-bridge

**Source**: Referenced from `ipu6-isys.c:688`

The ipu-bridge module discovers camera sensors from ACPI tables and
creates software nodes for them:

```c
// Called during ISYS initialization
ret = ipu_bridge_init(dev, ipu_bridge_parse_ssdb);
```

The bridge:
1. Scans ACPI namespace for camera sensor devices
2. Parses SSDB (Sensor Specific Data Block) for CSI-2 configuration
3. Creates `software_node` entries for each sensor
4. Sets up `fwnode` links: sensor → CSI-2 port
5. Instantiates VCM (Voice Coil Motor) i2c devices if present

---

## 10. Firmware Communication Layer

### 10.1 Syscom Message Queues

**Source**: `ipu6-fw-com.h`

The driver communicates with IPU firmware through system communication
(syscom) message queues:

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
│  Queue Configuration:                                    │
│  ├── Send queue: 40 entries (commands to FW)             │
│  ├── Recv queue: 40 entries (responses from FW)          │
│  ├── Proxy send: 5 entries (register access)             │
│  └── Proxy recv: 5 entries (register access responses)   │
└──────────────────────────────────────────────────────────┘
```

API:

```c
// Lifecycle
int  ipu6_fw_com_prepare(config, ...);
int  ipu6_fw_com_open(fwcom);
bool ipu6_fw_com_ready(fwcom);
int  ipu6_fw_com_close(fwcom);
int  ipu6_fw_com_release(fwcom, force);

// Send (driver → firmware)
void *ipu6_send_get_token(fwcom, stream_id);
void  ipu6_send_put_token(fwcom, stream_id);

// Receive (firmware → driver)
void *ipu6_recv_get_token(fwcom, stream_id);
void  ipu6_recv_put_token(fwcom, stream_id);
```

### 10.2 Firmware Command Types

**Source**: `ipu6-fw-isys.h`

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

### 10.3 Firmware Response Types

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

### 10.4 Stream Configuration ABI

**Source**: `ipu6-fw-isys.h`

```c
struct ipu6_fw_isys_stream_cfg_data_abi {
    struct ipu6_fw_isys_input_pin_info_abi input_pins[MAX_IPINS]; // Up to 4
    struct ipu6_fw_isys_output_pin_info_abi output_pins[MAX_OPINS]; // Up to 5

    u32 nof_input_pins;
    u32 nof_output_pins;

    struct ipu6_fw_isys_cropping_abi crop;

    u8 send_irq_sof_discarded;
    u8 send_irq_eof_discarded;
    u8 send_resp_sof_discarded;
    u8 send_resp_eof_discarded;

    struct ipu6_fw_isys_linked_receiver_abi linked_receiver;
    u8 src;          // Source index (CSI-2 port)
    u8 vc;           // Virtual channel
    u8 isl_use;
    u8 sensor_type;

    u8 compfmt;
    u8 stream_handle;
};
```

### 10.5 Capture Request ABI

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

### 10.6 Firmware Response

```c
struct ipu6_fw_isys_resp_info_abi {
    u64 buf_id;
    u32 pin_id;
    struct ipu6_fw_isys_output_pin_info_abi pin;

    u32 process_group_light_id;
    u64 timestamp[2];    // Start and end timestamps
    u32 error;
    u32 error_details;

    enum ipu6_fw_isys_resp_type type;
    u8 stream_handle;
    u16 frame_counter;
};
```

### 10.7 Command/Response Flow

```
Stream Open:
  Driver                                     Firmware
    │                                           │
    │  ipu6_send_get_token()                    │
    │  Fill stream_cfg_data_abi:                │
    │    - input pins (sensor format, CSI)      │
    │    - output pins (memory format, stride)  │
    │    - source port, virtual channel         │
    │  ipu6_send_put_token()                    │
    │  ─────────────────────────────────────────►
    │                                           │
    │  wait_for_completion(stream_open)         │
    │  ◄─────────────────────────────────────────
    │  STREAM_OPEN_DONE response                │
    │                                           │

Capture Request:
    │  ipu6_send_get_token()                    │
    │  Fill frame_buff_set_abi:                 │
    │    - output pin payloads (DMA addresses)  │
    │    - IRQ control flags                    │
    │  ipu6_send_put_token()                    │
    │  ─────────────────────────────────────────►
    │                                           │
    │  ISR: PIN_DATA_READY                      │
    │  ◄─────────────────────────────────────────
    │  Response contains:                       │
    │    - buf_id, pin_id, timestamps           │
    │    - error status                         │
    │  vb2_buffer_done() → return to userspace  │
```

---

## 11. Complete Data Flow: App to Hardware

### 11.1 Intel IPU7 HAL: Full Capture Path

This section traces a single frame capture from the Android application
all the way to the hardware and back.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    COMPLETE CAPTURE DATA FLOW                        │
│                    (Intel IPU7 Camera HAL)                           │
│                                                                      │
│  ① Application: CameraCaptureSession.capture(request)                │
│     │                                                                │
│  ② Binder IPC → CameraService                                       │
│     │                                                                │
│  ③ Camera3Device::processCaptureRequest()                            │
│     │  - Validate request                                            │
│     │  - Acquire output buffer handles                               │
│     │  - Submit to HAL via ICameraDeviceSession                      │
│     │                                                                │
│  ④ AIDL IPC → Intel IPU7 HAL                                        │
│     │                                                                │
│  ⑤ RequestThread::processRequest()                                   │
│     │  - Acquire DataContext from CameraContext                       │
│     │  - Parse capture settings (AE mode, AF mode, etc.)             │
│     │  - Enqueue request to mPendingRequests                         │
│     │                                                                │
│  ⑥ RequestThread::threadLoop()                                       │
│     │  - Wait for trigger event (NEW_REQUEST + NEW_SOF/STATS)        │
│     │  - fetchNextRequest()                                          │
│     │  - handleRequest():                                            │
│     │    │                                                           │
│     │    ├── AiqUnit::run3A(ccaId, applyingSeq, frameNumber)         │
│     │    │   │                                                       │
│     │    │   ├── IPAClient::runAec() → AE parameters                 │
│     │    │   │   └── → IPAServer → CcaWorker → Intel CCA lib        │
│     │    │   │       └── Returns: exposure, gain, flash              │
│     │    │   │                                                       │
│     │    │   ├── IPAClient::runAiq() → AWB + ISP parameters          │
│     │    │   │   └── → IPAServer → CcaWorker → Intel CCA lib        │
│     │    │   │       └── Returns: CCM, gamma, tone map               │
│     │    │   │                                                       │
│     │    │   └── IPAClient::runAic() → ISP firmware config           │
│     │    │       └── → IPAServer → CcaWorker → Intel CCA lib        │
│     │    │           └── Returns: PAL binary for PSYS                │
│     │    │                                                           │
│     │    └── Apply 3A results to sensor hardware                     │
│     │        └── SensorHwCtrl: set exposure/gain via I2C             │
│     │                                                                │
│  ⑦ CaptureUnit::qbuf(port, cameraBuffer)                            │
│     │  - Convert CameraBuffer → V4L2 buffer                         │
│     │  - DeviceBase::queueBuffer(sequence)                           │
│     │    └── ioctl(VIDIOC_QBUF) → kernel V4L2 layer                 │
│     │                                                                │
│  ⑧ Kernel: ipu6_isys_buf_queue()                                     │
│     │  - ipu6_dma_map_sgtable() → map through IPU MMU               │
│     │  - buffer_list_get() → collect buffer set                      │
│     │  - ipu6_fw_isys_send_capture():                                │
│     │    │  ipu6_send_get_token(fwcom, stream_id)                    │
│     │    │  Fill frame_buff_set_abi:                                  │
│     │    │    - output_pins[].addr = IOVA from DMA mapping           │
│     │    │    - send_irq_sof/eof = 1                                 │
│     │    │  ipu6_send_put_token(fwcom, stream_id)                    │
│     │    └── Command written to syscom send queue                    │
│     │                                                                │
│  ⑨ IPU Firmware:                                                     │
│     │  - Reads command from syscom queue                             │
│     │  - Programs CSI-2 receiver DMA                                 │
│     │  - Waits for next frame from sensor                            │
│     │                                                                │
│  ⑩ MIPI CSI-2 Hardware:                                              │
│     │  - Sensor transmits frame data over CSI-2 lanes                │
│     │  - CSI-2 receiver deserializes data                            │
│     │  - Data flows through ISYS pixel buffer (SRAM)                 │
│     │  - DMA writes frame to system memory (IOVA address)            │
│     │                                                                │
│  ⑪ IPU Firmware → Driver:                                            │
│     │  - SOF response → isys_isr() → SOF event to HAL               │
│     │  - PIN_DATA_READY response with timestamp                      │
│     │                                                                │
│  ⑫ Kernel ISR:                                                       │
│     │  - isys_isr() → isys_isr_one()                                │
│     │  - ipu6_recv_get_token() → parse firmware response             │
│     │  - vb2_buffer_done(VB2_BUF_STATE_DONE)                        │
│     │  - V4L2 poll wakeup → userspace notified                      │
│     │                                                                │
│  ⑬ CaptureUnit::poll() → dequeueBuffer()                            │
│     │  - ioctl(VIDIOC_DQBUF) → raw frame available                  │
│     │  - Notify BufferConsumers (ProcessingUnit)                     │
│     │                                                                │
│  ⑭ ProcessingUnit::processNewFrame()                                 │
│     │  - prepareTask():                                              │
│     │    ├── Get raw source buffer from CaptureUnit                  │
│     │    ├── Get output destination buffers (YUV/JPEG)               │
│     │    └── Get ISP settings from AiqResultStorage                  │
│     │  - dispatchTask():                                             │
│     │    ├── PipeManager → PSysDevice::addTask()                     │
│     │    │   └── ioctl() → PSYS kernel device                       │
│     │    │       └── IPU firmware processes raw→YUV                  │
│     │    │                                                           │
│     │    └── Wait for task completion                                │
│     │        └── PSysDevice::poll() → bufferDone()                   │
│     │                                                                │
│  ⑮ ProcessingUnit::onTaskDone()                                      │
│     │  - onBufferDone() → CameraStream::onBufferAvailable()          │
│     │  - onMetadataReady() → fill CaptureResult metadata             │
│     │                                                                │
│  ⑯ CameraStream → AIDL callback                                     │
│     │  - ICameraDeviceCallback::processCaptureResult()               │
│     │  - ICameraDeviceCallback::notify(SHUTTER)                      │
│     │                                                                │
│  ⑰ CameraService → Application                                      │
│     │  - Camera3Device dispatches result                             │
│     │  - Binder callback to app                                      │
│     │                                                                │
│  ⑱ Application receives:                                             │
│     - onCaptureStarted(timestamp)                                    │
│     - onCaptureCompleted(result metadata)                            │
│     - Image available in output Surface/ImageReader                  │
└──────────────────────────────────────────────────────────────────────┘
```

### 11.2 Google Desktop HAL: USB Capture Path

```
┌──────────────────────────────────────────────────────────────────────┐
│                    USB CAMERA CAPTURE DATA FLOW                      │
│                    (Google Desktop Camera HAL)                       │
│                                                                      │
│  ① Application: CameraCaptureSession.capture(request)                │
│     │                                                                │
│  ② Binder IPC → CameraService → Camera3Device                       │
│     │                                                                │
│  ③ AIDL IPC → Google Desktop HAL (Rust process)                      │
│     │                                                                │
│  ④ Session: Enqueue capture request                                  │
│     │                                                                │
│  ⑤ Worker thread:                                                    │
│     │  - crossbeam channel receives Capture task                     │
│     │  - V4L2CaptureDevice::dequeue_buffer()                        │
│     │    └── ioctl(VIDIOC_DQBUF) → MJPEG/YUYV frame from UVC       │
│     │                                                                │
│  ⑥ Frame processing (in Worker):                                     │
│     │  - MJPEG decode → NV12/YUV                                    │
│     │  - Format conversion if needed                                 │
│     │  - JPEG encoding for BLOB streams                             │
│     │  - Copy to output Gralloc buffer                               │
│     │                                                                │
│  ⑦ V4L2CaptureDevice::queue_buffer()                                │
│     │  └── ioctl(VIDIOC_QBUF) → return buffer to UVC driver         │
│     │                                                                │
│  ⑧ Result callback via AIDL                                          │
│     │  - processCaptureResult(filled buffers + metadata)             │
│     │  - notify(SHUTTER, timestamp)                                  │
│     │                                                                │
│  ⑨ CameraService → Application                                      │
│     - onCaptureCompleted / Image available                           │
└──────────────────────────────────────────────────────────────────────┘
```

### 11.3 Timing and Latency Considerations

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

Typical latency per frame:
  Sensor exposure: 10-33ms (30-100fps)
  ISYS capture:    ~1ms (DMA transfer)
  3A processing:   ~5-10ms (parallel)
  PSYS processing: ~5-15ms (ISP pipeline)
  Total pipeline:  2-3 frame latency for 3A convergence
```

---

## 12. Buffer Management and DMA

### 12.1 Buffer Types and Allocation

The system uses several buffer management strategies:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Buffer Management Layers                       │
│                                                                  │
│  Application Layer:                                              │
│  ┌──────────────────────────────────────────┐                   │
│  │ Gralloc Buffers (Android graphics alloc) │                   │
│  │ - Backed by ION/DMA-BUF heap            │                   │
│  │ - Shared via NativeHandle (fd + metadata)│                   │
│  └─────────────────┬────────────────────────┘                   │
│                    │ importBuffer()                               │
│  HAL Layer:        │                                             │
│  ┌─────────────────▼────────────────────────┐                   │
│  │ CameraBuffer (icamera internal)          │                   │
│  │ - Wraps Gralloc or internal allocation   │                   │
│  │ - Tracks DMA address, sequence, port     │                   │
│  └─────────────────┬────────────────────────┘                   │
│                    │ V4L2 QBUF/DQBUF                             │
│  Kernel Layer:     │                                             │
│  ┌─────────────────▼────────────────────────┐                   │
│  │ VB2 DMA-SG Buffers                       │                   │
│  │ - Scatter-gather list for physical pages │                   │
│  │ - Mapped through IPU MMU for firmware    │                   │
│  │                                          │                   │
│  │ ipu6_isys_buf_init():                    │                   │
│  │   sg = vb2_dma_sg_plane_desc(vb, 0)     │                   │
│  │   ipu6_dma_map_sgtable(adev, sg)        │                   │
│  │   ivb->dma_addr = sg_dma_address()      │                   │
│  │   (This is IOVA in IPU address space)   │                   │
│  └─────────────────┬────────────────────────┘                   │
│                    │ IOVA address                                 │
│  Firmware:         │                                             │
│  ┌─────────────────▼────────────────────────┐                   │
│  │ Frame buffer payload (DMA target)        │                   │
│  │ - Firmware DMAs sensor data to IOVA      │                   │
│  │ - IPU MMU translates IOVA → physical     │                   │
│  └──────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
```

### 12.2 IPU DMA Operations

**Source**: `ipu6-dma.h`

The IPU uses custom DMA operations that route through its internal MMU
rather than the system IOMMU:

```c
struct ipu6_dma_mapping {
    struct ipu6_mmu_info *mmu_info;    // IPU MMU page tables
    struct iova_domain iovad;           // IOVA allocator
};

// Custom DMA ops registered per auxiliary bus device:
ipu6_dma_alloc()      // Allocate IOVA + physical pages
ipu6_dma_free()       // Free IOVA + physical pages
ipu6_dma_mmap()       // mmap for userspace access
ipu6_dma_map_sg()     // Map scatter-gather list through IPU MMU
ipu6_dma_unmap_sg()   // Unmap from IPU MMU
ipu6_dma_map_sgtable()   // Map SG table (used by VB2)
ipu6_dma_unmap_sgtable() // Unmap SG table
```

### 12.3 PSYS Buffer Registration

For PSYS processing, buffers are registered with the kernel via PSysDevice:

```cpp
// HAL registers buffer with PSYS kernel device
int PSysDevice::registerBuffer(TerminalBuffer* buf) {
    struct ipu_psys_buffer psysBuf;
    psysBuf.base.fd = buf->fd;      // DMA-BUF file descriptor
    psysBuf.base.flags = buf->flags;
    // ioctl(mFd, IPU_IOC_REGISTER_BUFFER, &psysBuf)
}

// Submit processing task with registered buffers
int PSysDevice::addTask(const PSysTask& task) {
    // Fill ipu_psys_term_buffers for each terminal
    // ioctl(mFd, IPU_IOC_TASK_ADD, taskBuffers)
}
```

---

## 13. Power Management and Performance

### 13.1 iWake Watermark System

**Source**: `ipu6-isys.c:531`

The iWake system dynamically adjusts power management based on active
stream data rates:

```
┌──────────────────────────────────────────────────────────────────┐
│                    iWake Power Management                         │
│                                                                  │
│  Input: Active video streams with data rates                     │
│                                                                  │
│  Calculation:                                                    │
│  1. Sum data rates of all active streams (MB/s)                  │
│  2. calc_fill_time = max_sram_size / total_data_rate             │
│  3. LTR (Latency Tolerance Reporting):                           │
│     - Enhanced mode: use platform LTR value                      │
│     - Legacy mode: lookup table based on fill time               │
│  4. DID (Delay In Data):                                         │
│     - Enhanced: fill_time × 90%                                  │
│     - Legacy: fill_time - LTR                                    │
│  5. iwake_threshold = DID × data_rate >> sram_granularity        │
│  6. critical_threshold = iwake + (buffer_pages - iwake) / 2      │
│                                                                  │
│  Output: Written to fabric control register:                     │
│  ┌─────────────────────────────────────────────┐                │
│  │ Fabric Control Register (0x68)               │                │
│  │ bits[9:0]   ltr_val   (LTR value)           │                │
│  │ bits[12:10] ltr_scale (time scale)          │                │
│  │ bits[25:16] did_val   (DID value)           │                │
│  │ bits[28:26] did_scale (time scale)          │                │
│  │ bit[30]     keep_power_in_D0                │                │
│  │ bit[31]     keep_power_override             │                │
│  └─────────────────────────────────────────────┘                │
│                                                                  │
│  Firmware registers (via proxy token):                           │
│  - GDA_ENABLE_IWAKE_INDEX (2)                                    │
│  - GDA_IWAKE_THRESHOLD_INDEX (1)                                 │
│  - GDA_IRQ_CRITICAL_THRESHOLD_INDEX (0)                          │
│  - GDA_MEMOPEN_THRESHOLD_INDEX (3)                               │
│                                                                  │
│  Power states:                                                   │
│  ┌──────────────┬──────────────┬────────────────────────┐       │
│  │ State        │ LTR/DID      │ Action                  │       │
│  ├──────────────┼──────────────┼────────────────────────┤       │
│  │ ISYS OFF     │ 1023 / 1023  │ Max latency tolerance   │       │
│  │ ISYS ON      │ 20 / 20      │ Low latency for PKGC-2R │       │
│  │ iWake OFF    │ 20 / 20      │ No streams active       │       │
│  │ iWake ON     │ Calculated   │ Streams active          │       │
│  │ Enhanced     │ Platform     │ Enhanced iWake mode     │       │
│  └──────────────┴──────────────┴────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### 13.2 Runtime Power Management

```
Bus PM Domain Flow:

  App opens camera
      │
      ▼
  pm_runtime_get() on ISYS auxiliary device
      │
      ▼
  bus_pm_runtime_resume()
      ├── ipu6_buttress_power(ISYS, true)
      │   └── Write power control register
      │   └── Poll for power ack (200ms timeout)
      ├── pm_generic_runtime_resume()
      │   └── ISYS driver probe/resume
      └── set_iwake_ltrdid(LTR_ISYS_ON)

  Streaming active...
      │
      ▼
  update_watermark_setting()  (on each stream start/stop)

  App closes camera
      │
      ▼
  pm_runtime_put() on ISYS auxiliary device
      │
      ▼
  bus_pm_runtime_suspend()
      ├── pm_generic_runtime_suspend()
      ├── ipu6_buttress_power(ISYS, false)
      └── set_iwake_ltrdid(LTR_ISYS_OFF)
```

### 13.3 QoS Configuration

```c
// PM QoS prevents CPU from entering deep sleep during capture
#define ISYS_PM_QOS_VALUE 300  // microseconds

// Set during stream start:
cpu_latency_qos_add_request(&isys->pm_qos, ISYS_PM_QOS_VALUE);

// Removed during stream stop:
cpu_latency_qos_remove_request(&isys->pm_qos);
```

---

## 14. Key File Reference Index

### 14.1 Android Camera Framework

| File | Purpose |
|------|---------|
| `frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.h` | HAL provider discovery and management |
| `frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp` | Camera device state machine and request routing |
| `frameworks/av/camera/` | Camera2 API native implementation |

### 14.2 AIDL Camera Interfaces

| File | Purpose |
|------|---------|
| `hardware/interfaces/camera/provider/aidl/ICameraProvider.aidl` | Provider discovery interface |
| `hardware/interfaces/camera/device/aidl/ICameraDevice.aidl` | Device open/characteristics |
| `hardware/interfaces/camera/device/aidl/ICameraDeviceSession.aidl` | Stream config, capture requests |
| `hardware/interfaces/camera/device/aidl/ICameraDeviceCallback.aidl` | Result/notification callbacks |

### 14.3 Google Desktop Camera HAL

| File | Purpose |
|------|---------|
| `vendor/google/desktop/camera/hal_usb/src/v4l2_device.rs` | V4L2 capture device wrapper |
| `vendor/google/desktop/camera/hal_usb/src/uvc_device.rs` | UVC camera discovery |
| `vendor/google/desktop/camera/hal_usb/src/session/worker.rs` | Capture worker thread |
| `vendor/google/desktop/camera/hal_usb/src/session/stream.rs` | Stream management |
| `vendor/google/desktop/camera/hal_usb/src/session/capture_config.rs` | Capture configuration |
| `vendor/google/desktop/camera/common/src/lib.rs` | Shared library (AIDL, FMQ, format) |

### 14.4 Intel IPU7 Camera HAL

| File | Purpose |
|------|---------|
| `vendor/intel/camera/hal/ipu7/src/core/CaptureUnit.h` | ISYS abstraction (StreamSource) |
| `vendor/intel/camera/hal/ipu7/src/core/ProcessingUnit.h` | PSYS abstraction (image processing) |
| `vendor/intel/camera/hal/ipu7/src/core/PSysDevice.h` | PSYS kernel device interface |
| `vendor/intel/camera/hal/ipu7/src/core/DeviceBase.h` | V4L2 device base class |
| `vendor/intel/camera/hal/ipu7/src/core/CameraStream.h` | App stream → pipeline mapping |
| `vendor/intel/camera/hal/ipu7/src/core/RequestThread.h` | Request queue and 3A orchestration |
| `vendor/intel/camera/hal/ipu7/src/core/CameraContext.h` | Per-camera state and result storage |
| `vendor/intel/camera/hal/ipu7/src/core/CameraBuffer.h` | Internal buffer representation |
| `vendor/intel/camera/hal/ipu7/src/core/IProcessingUnit.h` | Processing unit interface |
| `vendor/intel/camera/hal/ipu7/src/core/StreamSource.h` | Stream source (BufferProducer) interface |
| `vendor/intel/camera/hal/ipu7/src/core/BufferQueue.h` | Producer/consumer buffer queue |
| `vendor/intel/camera/hal/ipu7/src/core/SofSource.h` | Start-of-frame event source |
| `vendor/intel/camera/hal/ipu7/src/core/IspSettings.h` | ISP parameter storage |
| `vendor/intel/camera/hal/ipu7/src/core/IpuPacAdaptor.h` | PAC (Parameter Adaptation Controller) |
| `vendor/intel/camera/hal/ipu7/src/core/SensorHwCtrl.h` | Sensor I2C control |
| `vendor/intel/camera/hal/ipu7/src/core/LensHw.h` | VCM lens focus control |
| `vendor/intel/camera/hal/ipu7/src/3a/AiqUnit.h` | 3A algorithm control unit |
| `vendor/intel/camera/hal/ipu7/ipa/IPAServer.h` | IPA algorithm server |
| `vendor/intel/camera/hal/ipu7/ipa/IPAHeader.h` | IPA command definitions |
| `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAClient.h` | IPA client (HAL-side) |
| `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAClientWorker.h` | IPA worker threads |
| `vendor/intel/camera/hal/ipu7/pipeline/ipaclient/IPAMemory.h` | Shared memory management |
| `vendor/intel/camera/hal/ipu7/src/v4l2/V4l2DeviceFactory.h` | V4L2 device creation |

### 14.5 IPU Kernel Driver

| File | Purpose |
|------|---------|
| `drivers/media/pci/intel/ipu6/ipu6.c` | Main PCI driver, probe, SPC config |
| `drivers/media/pci/intel/ipu6/ipu6.h` | Device structures, HW variants, MMU config |
| `drivers/media/pci/intel/ipu6/ipu6-isys.c` | ISYS driver: init, ISR, iWake, V4L2 registration |
| `drivers/media/pci/intel/ipu6/ipu6-isys.h` | ISYS structures, stream config, watermark |
| `drivers/media/pci/intel/ipu6/ipu6-isys-video.c` | V4L2 video device: formats, ioctl, stream ctrl |
| `drivers/media/pci/intel/ipu6/ipu6-isys-video.h` | Video device and stream structures |
| `drivers/media/pci/intel/ipu6/ipu6-isys-csi2.c` | CSI-2 receiver: timing, errors, PHY |
| `drivers/media/pci/intel/ipu6/ipu6-isys-csi2.h` | CSI-2 structures and subdevice |
| `drivers/media/pci/intel/ipu6/ipu6-isys-queue.c` | VB2 buffer queue: init, prepare, queue, done |
| `drivers/media/pci/intel/ipu6/ipu6-fw-isys.h` | Firmware ABI: commands, responses, configs |
| `drivers/media/pci/intel/ipu6/ipu6-fw-com.h` | Firmware syscom: token-based IPC |
| `drivers/media/pci/intel/ipu6/ipu6-mmu.c` | MMU: page tables, TLB, IOVA |
| `drivers/media/pci/intel/ipu6/ipu6-buttress.c` | Buttress: power, CSE IPC, TSC sync |
| `drivers/media/pci/intel/ipu6/ipu6-buttress.h` | Buttress structures and register defs |
| `drivers/media/pci/intel/ipu6/ipu6-bus.c` | Auxiliary bus: device create, PM domain |
| `drivers/media/pci/intel/ipu6/ipu6-bus.h` | Bus device structure |
| `drivers/media/pci/intel/ipu6/ipu6-cpd.c` | CPD firmware: parse, validate, pkg_dir |
| `drivers/media/pci/intel/ipu6/ipu6-cpd.h` | CPD header and entry structures |
| `drivers/media/pci/intel/ipu6/ipu6-dma.h` | Custom DMA ops for IPU bus |

### 14.6 Key Constants Reference

```
IPU6_ISYS_MAX_STREAMS        = 16    (max firmware streams)
IPU6_ISYS_SIZE_SEND_QUEUE    = 40    (command queue depth)
IPU6_ISYS_SIZE_RECV_QUEUE    = 40    (response queue depth)
IPU6_ISYS_MIN_WIDTH          = 2     (min capture width)
IPU6_ISYS_MAX_WIDTH          = 4672  (max capture width)
IPU6_ISYS_MIN_HEIGHT         = 1     (min capture height)
IPU6_ISYS_MAX_HEIGHT         = 3416  (max capture height)
IPU6_MAX_SRAM_SIZE           = 200KB (IPU6 pixel buffer)
IPU6SE_MAX_SRAM_SIZE         = 96KB  (IPU6SE pixel buffer)
IPU6_DEVICE_GDA_NR_PAGES     = 128   (GDA physical pages)
IPU6_MMU_MAX_DEVICES          = 4    (max MMU units)
ISP_PAGE_SIZE                 = 4KB  (MMU page size)
ISP_L1PT_PTES                 = 1024 (L1 page table entries)
ISP_L2PT_PTES                 = 1024 (L2 page table entries)
MAX_NODE_NUM (PSYS)           = 5    (max processing nodes)
MAX_LINK_NUM (PSYS)           = 10   (max node links)
MAX_TASK_NUM (PSYS)           = 8    (max concurrent tasks)
MAX_TERMINAL_NUM (PSYS)       = 26   (max terminals per node)
ISYS_PM_QOS_VALUE             = 300  (CPU latency QoS, us)
BUTTRESS_POWER_TIMEOUT_US     = 200ms
BUTTRESS_CSE_BOOTLOAD_TIMEOUT = 5s
BUTTRESS_CSE_AUTHENTICATE_TIMEOUT = 10s
```

---

## End of Document

This document covers the complete Android IPU7 camera architecture from
the Camera2 application API through the CameraService, AIDL HAL interfaces,
both HAL implementations (Google Desktop and Intel IPU7), the IPU kernel
driver, firmware communication, and hardware register programming.

For related documentation, see:
- `alos_grub/ES9356_Integration_Technical_Document.md` - ES9356 touchscreen integration
- `alos_grub/PTL_GCS_RT712_RT1320_Audio_Fix.md` - Audio subsystem documentation

