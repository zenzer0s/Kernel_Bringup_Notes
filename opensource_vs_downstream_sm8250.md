# Open-Source vs. Downstream Proprietary Alternatives Matrix: Qualcomm Snapdragon 865 / 870 (SM8250 / Kona)

**Target SoC:** Qualcomm Snapdragon 865 / 865+ / 870 (`SM8250`, `SM8250-AB`, `SM8250-AC` — Code Name: *Kona*)  
**Target Devices:** Xiaomi POCO F3 (`alioth`), OnePlus 8T (`kebab`), OnePlus 8 / 8 Pro (`instantnoodle`), Xiaomi Pad 6 (`pipa`), Sony Xperia 1 II / 5 II (`edo`), Samsung Galaxy S20 FE (`r8q`), etc.  
**Target Operating Systems:** Android 17 AOSP (Clover, LineageOS) & Mainline Linux (PostmarketOS, Alpine, Debian)  

---

## 1. Executive Summary

A long-standing question in modern mobile kernel development is:  
> **"Does a viable, production-grade open-source alternative exist for every legacy downstream Qualcomm driver on the SM8250 platform?"**

**The Reality:** Approximately **90% of all downstream components on SM8250 now have fully functional, often superior open-source upstream replacements.** SM8250 represents one of the most mature, well-supported Qualcomm silicon platforms in the mainline Linux ecosystem.

However, a critical distinction must be drawn between:
1. **Fully Solved Subsystems:** 100% open-source kernel drivers and userspace HALs that match or outperform Qualcomm's legacy code.
2. **Hybrid / Firmware-Dependent Subsystems:** Open-source kernel drivers that communicate with closed firmware running on isolated DSP co-processors.
3. **The Remaining Frontier (Camera & VoLTE):** Complex subsystems still lacking parity with proprietary vendor photography and cellular stacks.

---

## 2. Comprehensive Subsystem Comparison Matrix

| Hardware Subsystem | Legacy Downstream Qualcomm Implementation | Modern Open-Source Upstream Alternative | Open-Source Status | Architectural Verdict & Real-World Experience |
| :--- | :--- | :--- | :---: | :--- |
| **GPU (3D / Vulkan)** | Closed Adreno blobs (`libgsl.so`) on `/dev/kgsl-3d0` | **Mesa Freedreno / Turnip** (`vulkan.adreno.so`) on DRM/KMS | **SUPERIOR** | Modern Vulkan 1.3 support. Turnip is actively maintained and often outperforms Qualcomm's legacy 4.19 blobs in Vulkan titles and Android RenderThread. |
| **GPU (OpenGL ES)** | Closed `libGLESv2_adreno.so` on KGSL | **Google ANGLE** (OpenGL ES translated to Vulkan) | **EXCELLENT** | 100% spec-compliant GLES 2.0 / 3.0 / 3.2. Translates GLES API calls directly into Vulkan pipeline state objects on Turnip. |
| **Display Composition** | Closed `hwcomposer.qcom.so` (SDM) + TechPack `msm_drm` | **`drm_hwcomposer`** (`hwc3-drm`) + Mainline DRM/KMS | **EXCELLENT** | Direct atomic modesetting (`DRM_IOCTL_MODE_ATOMIC_COMMIT`) on `/dev/dri/card0`. Full 120Hz/144Hz multi-plane composition. |
| **Memory Allocator** | Closed Qualcomm `/dev/ion` heaps + `libgralloccore.so` | **Linux DMA-BUF Heaps** (`/dev/dma_heap/system`) | **SUPERIOR** | Clean, standardized Linux memory management. Replaces obsolete ION ioctl mechanics with universal buffer file descriptors. |
| **Touchscreen** | OEM vendor driver (`focaltech_touch.ko`, `goodix_touch.ko`) | **Upstream `ft8756.c` / `goodix.c`** | **EXCELLENT** | Standard Linux Multi-Touch Protocol B. Pre-allocated DMA buffers eliminate allocator delays inside high-frequency interrupts. |
| **Wi-Fi Subsystem** | Closed Qualcomm QCACLD-3.0 (`wlan.ko`) | **In-Kernel `ath11k`** (PCIe QCA6390) | **EXCELLENT** | Native Linux `cfg80211` / `mac80211` wireless stack. Binds directly over PCIe bus 0 with zero out-of-tree module dependencies. |
| **Bluetooth** | Downstream `btqis` transport daemon | **In-Kernel `hci_uart` (QCA)** over UART | **EXCELLENT** | Standard Linux BlueZ / Android Fluoride stack. Communicates over `/dev/ttyHS0` (`&uart6`) with firmware loaded by kernel. |
| **Storage (UFS)** | Downstream `ufshcd-qcom` | **Upstream `drivers/ufs/host/ufs-qcom.c`** | **EXCELLENT** | Full UFS 3.1 bandwidth and sequential/random I/O throughput on SCSI disk nodes (`/dev/block/sda*`). |
| **Security Keystore** | Qualcomm TrustZone QSEECOM (`/dev/qseecom` + `qseecomd`) | **PureSoftKeyMint V5 AIDL Service** | **EXCELLENT** | 100% open-source C++ software keystore. Provides standard cryptographic key generation, AES/RSA signing, and attestation. |
| **Backlight Control** | Closed TechPack PWM backlight driver | **DRM Panel Backlight** (`/sys/class/backlight/`) | **EXCELLENT** | Native Linux 10-bit dynamic brightness control via MIPI DSI DCS commands. |
| **Haptics Motor** | OEM `/sys/class/qcom-haptics/` driver | **Upstream `drivers/input/misc/aw8697.c`** | **EXCELLENT** | Standard Linux `FF_RUMBLE` interface with automatic resonance tracking calibrated during device probe. |
| **Audio Subsystem** | Closed QDSP6 Audio HAL (`audio.primary.kona.so`) + ADSP | **Upstream ASoC QDSP6** (`q6afe`, `q6asm`) + ALSA UCM2 | **HYBRID** | Kernel driver and userspace ALSA routing are 100% open source. However, Qualcomm's proprietary ADSP firmware (`adsp.mbn`) is still required. |
| **Cellular (Data/SMS)** | Closed QMI daemons (`qmuxd`, `netmgrd`, `rmnet_ipa`) | **Upstream Linux QRTR** + ModemManager / open RIL | **HYBRID** | Standard 4G/5G data packet routing and SMS function cleanly. However, carrier VoLTE/IMS calling requires proprietary RIL blobs. |
| **Sensors** | Qualcomm Snapdragon Sensor Core (SSC) on SLPI DSP | **Upstream Linux IIO** or FastRPC into SLPI | **HYBRID** | Raw accelerometer, gyro, and proximity readings work via IIO. Advanced factory calibration still relies on vendor sensor registries. |
| **Camera (ISP)** | Closed Qualcomm CAMX / CHI + Spectra 480 ISP HAL | **Upstream `qcom-camss`** + **Libcamera** | **PARTIAL (The Gap)** | Raw sensor streaming over V4L2 works. However, computational 3A algorithms (HDR, Auto-Exposure, Face Detection) are not yet on par with GCam. |
| **Fingerprint** | QSEECOM TrustZone TA applet (`fp_tz.mbn`) | Physical button FPC drivers (FPC1020) | **PARTIAL** | Physical side-mounted sensors work via open drivers. Optical Under-Display Fingerprint (UDFPS) remains dependent on TrustZone secure display. |

---

## 3. Deep Architectural Breakdown by Tier

### Tier 1: The Fully Conquered Open-Source Stack (Production Ready)

#### 1. Graphics: The Mesa Turnip Revolution
Qualcomm’s downstream graphics stack relies on `/dev/kgsl-3d0`, a proprietary kernel driver that exposes raw GPU ringbuffers to closed userland blobs (`libgsl.so`). 
* **The Open Solution:** Rob Clark, Danylo Piliaiev, and the Freedreno community created **Turnip**, a native Vulkan driver for Adreno 600-series GPUs built into Mesa.
* **Why it Wins:**
  - Turnip runs directly on the upstream DRM/KMS subsystem (`/dev/dri/card0`, `renderD128`).
  - Implements **Vulkan 1.3**, descriptor indexing, dynamic rendering, and modern extensions that Qualcomm's Android 11-era drivers never received.
  - Paired with **Google ANGLE**, it provides flawless OpenGL ES 3.2 support without needing proprietary GLES drivers.

#### 2. Display: Hardware Composer on Linux DRM/KMS
Downstream Android utilizes Qualcomm's closed Display Composer (`hwcomposer.qcom.so`), which communicates via proprietary ioctls with the downstream `msm_drm` driver.
* **The Open Solution:** Linaro's **`drm_hwcomposer`** (`android.hardware.composer.hwc3-service.drm`).
* **Why it Wins:**
  - Standardizes layer overlay composition on universal Linux atomic modesetting APIs (`DRM_IOCTL_MODE_ATOMIC_COMMIT`).
  - Supports 120Hz/144Hz high refresh rate switching, Display Stream Compression (DSC 1.1), and multi-plane hardware blending.

#### 3. Networking: In-Kernel ath11k & HCI UART
Downstream Qualcomm kernels use `QCACLD-3.0` (`wlan.ko`), an out-of-tree monstrosity of over 500,000 lines of proprietary code that frequently breaks across kernel versions.
* **The Open Solution:** The Linux wireless subsystem's **`ath11k`** driver, running natively over PCIe bus 0.
* **Why it Wins:**
  - Fully in-tree, compiled directly into the kernel (`CONFIG_ATH11K=y`, `CONFIG_ATH11K_PCI=y`).
  - Native integration with standard Android `wpa_supplicant` and Linux `NetworkManager`.
  - Bluetooth connects via standard `hci_uart` over serial (`&uart6`), eliminating proprietary transport daemons (`btqis`).

---

### Tier 2: The Hybrid Subsystems (Open Drivers, Closed DSP Firmware)

In modern Qualcomm SoCs, specialized digital signal processors (DSPs) run dedicated operating systems (QuRT):
* **ADSP (Audio DSP):** Runs Hexagon audio algorithms.
* **SLPI (Sensors Low Power Island):** Handles continuous sensor polling.
* **CDSP (Compute DSP):** Handles neural network processing.

#### How Mainline Handles DSPs:
Upstream Linux does **not** attempt to reimplement the DSP operating systems. Instead, it uses open-source communication buses:
1. **Remote Processor (`remoteproc`):** Powers up the DSPs and loads their signed firmware (`adsp.mbn`, `slpi.mbn`, `cdsp.mbn`).
2. **Qualcomm IPC Router (QRTR):** Modern socket-based messaging (`AF_QIPCRTR`) replacing legacy Qualcomm shared memory daemons.
3. **FastRPC (`drivers/misc/fastrpc.c`):** Allows userspace applications to queue computational tasks directly to the Hexagon DSPs.

*Status:* **The drivers, kernel interfaces, and HALs are 100% open source**, but the binaries executing inside the DSP cores remain signed Qualcomm firmware.

---

### Tier 3: The True Open-Source Frontiers (Camera & VoLTE)

#### 1. The Camera Subsystem (Qualcomm Spectra 480 ISP)
* **The Challenge:** While the upstream kernel driver **`qcom-camss`** can configure the CSI-2 receivers and stream raw Bayer sensor data into `/dev/video*`, modern smartphone photography is 90% software computational photography:
  - Complex 3A algorithms (Auto-Exposure, Auto-White Balance, Auto-Focus).
  - Multi-frame High Dynamic Range (HDR) synthesis.
  - Optical Image Stabilization (OIS) control loops.
* **Current State:** **Libcamera** is actively building an open-source 3A pipeline for Qualcomm platforms, but it is not yet capable of matching Google Camera (GCam) or OEM camera quality.

#### 2. Carrier-Certified VoLTE / IMS Calling
* **The Challenge:** Cellular data (4G LTE / 5G NR) and SMS operate cleanly through the open-source **QRTR** interface and `ModemManager`. However, Voice over LTE (VoLTE) and Voice over Wi-Fi (VoWiFi) require proprietary SIP/IMS protocol stacks certified by cellular carriers.
* **Current State:** On Android AOSP, achieving stable VoLTE still requires wrapping Qualcomm's proprietary Radio Interface Layer (`libril-qc-hal-qmi.so`). Pure open-source RIL implementations are currently limited to non-VoLTE voice and data.

---

## 4. Architectural Summary for Android 17 Bringup

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   Android 17 Userspace (AOSP / Clover)                 │
├──────────────────────┬──────────────────────┬──────────────────────────┤
│    Graphics / UI     │  Connectivity / Comms│     Audio & Input        │
│   • Mesa Turnip      │   • ath11k Wi-Fi     │   • ALSA UCM2 / QDSP6    │
│   • Google ANGLE     │   • hci_uart BT      │   • ft8756 Multi-touch   │
│   • drm_hwcomposer   │   • Linux QRTR / RIL │   • aw8697 Haptics       │
├──────────────────────┴──────────────────────┴──────────────────────────┤
│                  Standard Linux Kernel (ACK 6.12 / 6.19)               │
│   • DRM/KMS Subsystem                       • Linux DMA-BUF Heaps      │
│   • Linux Input Subsystem (Protocol B)      • PureSoftKeyMint V5 AIDL  │
└────────────────────────────────────────────────────────────────────────┘
```

### Strategic Conclusion:
On Snapdragon 865/870, **an open-source alternative exists for virtually every component required for a high-performance, smooth, 120Hz daily smartphone experience.** The only areas where proprietary blobs remain unavoidable are carrier-certified VoLTE telephony and high-end computational camera post-processing.
