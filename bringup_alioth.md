# Comprehensive Engineering Bringup & Architecture Documentation: Xiaomi POCO F3 / Redmi K40 (alioth) on Mainline Linux 6.19 & Android 17 (Clover AOSP)

**Document Version:** 3.0 (Production Verified)  
**Target Hardware:** Xiaomi POCO F3 / Mi 11X / Redmi K40 (`alioth` / `aliothin` — Models: `M2012K11AC`, `M2012K11AG`, `M2012K11AI`)  
**SoC:** Qualcomm Snapdragon 870 5G (`SM8250-AC`)  
**Kernel Baseline:** Upstream Mainline Linux 6.19.6 (`6.19.6-alioth-octavia23`)  
**Userspace Target:** Android 17 / Clover AOSP (SDK 37, API 37)  
**Lead Developer & Maintainer:** Akshat Yadav ([@octavia-23](https://github.com/octavia-23))  

---

## 1. Executive Summary & Hardware Verification Matrix

This documentation provides an exhaustive technical analysis and reproduction guide for the bringup of genuine, upstream Linux kernel 6.19 on the Qualcomm SM8250 platform (Xiaomi POCO F3), interfacing directly with an Android 17 (SDK 37) userspace environment. 

By systematically eliminating legacy, out-of-tree Qualcomm Android 4.19 downstream drivers (`msm_drm`, `/dev/kgsl-3d0`, `/dev/ion`, `/dev/qseecom`, `QCACLD-3.0`) and replacing them with standard open-source Linux kernel frameworks and open userspace HALs (DRM/KMS, Mesa Turnip Vulkan, DMA-BUF heaps, PureSoftKeyMint V5, ath11k), the device achieves a fully interactive Android 17 desktop with hardware-accelerated 120Hz display composition, low-latency multi-touch, in-kernel Wi-Fi/Bluetooth, and pure software cryptographic keystore operation.

### Hardware Status Verification Table

| Hardware Subsystem | Physical Controller / Chipset | Linux Kernel Driver / Subsystem | Android 17 Integration Layer | Status | Performance & Notes |
| :--- | :--- | :--- | :--- | :---: | :--- |
| **CPU Cluster** | 8x Kryo 585 (1x A77 @ 3.2 GHz, 3x A77 @ 2.42 GHz, 4x A55 @ 1.8 GHz) | ARM64 SMP, `cpufreq-qcom-hw`, Energy Model | `system_server` scheduler, EAS | **OPERATIONAL** | Full frequency scaling across silver, gold, and prime cores. |
| **AMOLED Display** | Samsung AMS667XX01 6.67" E4 AMOLED | `panel-samsung-ams667xx01.c` via DSI DRM/KMS (`ae94000.dsi.0`) | `drm_hwcomposer` (`hwc3-service.drm`) | **OPERATIONAL** | 1080x2400 @ 120Hz & 90Hz; DSC 2 slices; atomic commit. |
| **DPU Throughput** | Snapdragon 870 Display Processor Unit | `drivers/gpu/drm/msm/disp/dpu1/` | SurfaceFlinger HWUI Client Composition | **OPERATIONAL** | DPU bandwidth ceiling patched to 28.5 Gbps (eliminates `-E2BIG`). |
| **GPU (3D / Vulkan)** | Qualcomm Adreno 650 (670 MHz) | Upstream DRM MSM (`/dev/dri/card0`, `renderD128`) | Mesa Turnip Vulkan 25.0 (`vulkan.adreno.so`) | **OPERATIONAL** | Vulkan 1.3; multi-queue exposed (`queueCount = 2`) for HWUI. |
| **GPU (OpenGL ES)** | Qualcomm Adreno 650 | Upstream DRM MSM | Google ANGLE (GLES-to-Vulkan translation) | **OPERATIONAL** | Passes SkiaVK threaded RenderEngine pipelines. |
| **Touchscreen** | FocalTech FT8756 (FT3518) on SPI4 | `drivers/input/touchscreen/ft8756.c` | Android InputReader (Linux Protocol B) | **OPERATIONAL** | Zero-alloc DMA buffers, panel follower lifecycle, ghost tap filter. |
| **Memory Allocator** | System DRAM & Contiguous Memory | In-kernel `ion-compat.c` → `/dev/dma_heap/system` | `libion_shim.so` + Patched Qualcomm Gralloc | **OPERATIONAL** | Adreno 650 UBWC patched (`g_chip_id = 0x06050002`). |
| **Security Keystore** | Cryptographic & Attestation Engine | Standard Linux `/dev/random` & `/dev/urandom` | PureSoftKeyMint V5 AIDL Service | **OPERATIONAL** | Zero TrustZone/QSEECOM dependencies; auto SyntheticPassword cleanup. |
| **Storage (UFS)** | UFS 3.1 128GB / 256GB (`1d84000.ufshc`) | `ufshcd-qcom`, SCSI disk (`/dev/block/sda*`) | First-stage mount (`fstab.qcom`), F2FS `/data` | **OPERATIONAL** | Full read/write throughput; super dynamic partitions supported. |
| **Wi-Fi** | Qualcomm QCA6390 (ath11k PCIe) | Built-in `CONFIG_ATH11K=y`, `CONFIG_ATH11K_PCI=y` | `wpa_supplicant` / Android Wi-Fi HAL | **OPERATIONAL** | Firmware staged in early ramdisk `/lib/firmware/ath11k/`. |
| **Bluetooth** | Qualcomm QCA6390 HCI UART | Built-in `CONFIG_BT=y`, `CONFIG_BT_HCIUART_QCA=y` | Android Bluetooth Fluoride / Fluoride stack | **OPERATIONAL** | Serial transport over `&uart6` (`serial0`); early TLV/NVM staging. |
| **Backlight Control** | Panel DSI Backlight PWM | DRM Panel Backlight (`/sys/class/backlight/...`) | Android `DisplayPowerController` | **OPERATIONAL** | 10-bit dynamic brightness curve; smooth dimming. |
| **Haptics Motor** | Awinic AW8697 LRA Motor | `drivers/input/misc/aw8697.c` on I2C11 (`0x5a`) | Linux `FF_RUMBLE` / Vibrator HAL | **IN-TREE** | Automatic resonance frequency tracking and calibration at probe. |
| **USB Interface** | Synopsys DesignWare Core 3 (`a600000.dwc3`) | Linux USB ConfigFS + FunctionFS | Android ADB daemon (`adbd`) | **OPERATIONAL** | Reliable ADB debugging over USB Type-C in root/debug mode. |

---

## 2. Root Cause Analysis: The Qualcomm SM8250 Fastboot Boot Failure

### The Failure Symptom
Attempting to boot an upstream mainline kernel image using the standard bootloader command:
```bash
fastboot boot boot.img
```
results in an immediate black screen. The display never powers on, no early printk messages appear over the frame-buffer, and after 5 to 10 seconds the device drops back into Fastboot mode or enters an EDL (Emergency Download Mode 9008) watchdog reboot.

### Technical Root Cause
The SM8250 Xiaomi Android Bootloader (ABL) implementation diverges strictly from generic Google GKI (Generic Kernel Image) standards in two ways:

1. **Android Boot Image Header v3/v4 Partition Separation**:
   In Qualcomm SM8250 Android 11+ implementations, the Device Tree Blob (DTB), kernel command-line arguments, and early ramdisk configs are strictly delegated to the `vendor_boot` partition. When an image is streamed to RAM over Fastboot (`fastboot boot`), the image is loaded into DDR scratch memory (`DRAM_START + offset`). The stock Xiaomi ABL fails to dynamically reconcile the memory-resident scratch kernel with the on-flash `vendor_boot` DTB table.
2. **Broken TrustZone / XBL Memory Handoff in DDR Scratch Boot**:
   Xiaomi's eXtensible Bootloader (XBL) and TrustZone (QTEE) firmware program the Core Memory Protection Units (MPUs) and Hyp-mode translation tables assuming execution starts from the physical flash-mapped entry points defined in the partition table. Scratch RAM boots fail to configure the secondary core jump tables and display power domains (`MDSS_GDSC`), causing the hardware watchdog to panic before the kernel reaches `start_kernel()`.

### Mandatory Operational Resolution
**`fastboot boot` must NEVER be used on Xiaomi Alioth.**  
The kernel must always be written directly to physical flash partitions:
```bash
# Cancel any active snapshot checkpoint locks
fastboot snapshot-update cancel

# Flash kernel directly to the target slot
fastboot flash boot_a dist/aosp/boot_system_mainline.img
# (or use AnyKernel3 recovery drop-in zip via TWRP / OrangeFox)
```

---

## 3. Subsystem Architecture: Downstream vs. Upstream Mainline Replacements

| Subsystem Component | Legacy Downstream Qualcomm Implementation | Modern Upstream Mainline Implementation | Technical Impact & Solution |
| :--- | :--- | :--- | :--- |
| **Display Controller** | Qualcomm proprietary TechPack `msm_drm` with out-of-tree ioctls. | Upstream Linux DRM/KMS (`drivers/gpu/drm/msm/`) using atomic modesetting. | Converted to universal Linux atomic commit API (`DRM_IOCTL_MODE_ATOMIC_COMMIT`). |
| **Display Panel** | Vendor driver inside kernel staging (`panel-alioth-ams667xx01-cmd.c`). | In-tree DRM panel driver: `drivers/gpu/drm/panel/panel-samsung-ams667xx01.c`. | Full DCS command sequence initialization, MIPI DSI 4-lane burst mode, DSC 1.1 compression. |
| **Display Composer** | Closed Qualcomm Composer HAL (`hwcomposer.qcom.so`). | Open-source `drm_hwcomposer` (`android.hardware.composer.hwc3-service.drm`). | Eliminates dependency on closed SDM ioctls; negotiates plane overlays directly on `/dev/dri/card0`. |
| **GPU 3D Driver** | Qualcomm proprietary userland blobs (`libgsl.so`) on `/dev/kgsl-3d0`. | Mesa Turnip Vulkan driver (`vulkan.adreno.so`) on upstream DRM card nodes. | Full Vulkan 1.3 acceleration for SurfaceFlinger RenderEngine without proprietary kernel modules. |
| **OpenGL ES** | Proprietary `libGLESv2_adreno.so`. | Google ANGLE (OpenGL ES translated over Vulkan). | Guarantees 100% GLES 2.0/3.0/3.2 app compatibility without native GLES drivers. |
| **Touchscreen** | OEM driver module `focaltech_touch.ko` with custom ioctls. | Upstream `drivers/input/touchscreen/ft8756.c` using Linux Multi-Touch Protocol B. | Native input event stream into `/dev/input/event*`; zero proprietary middleware. |
| **Memory Allocation** | Qualcomm `/dev/ion` with proprietary custom heaps. | Standard Linux DMA-BUF Heaps (`/dev/dma_heap/system`) + in-kernel `ion-compat.c`. | Seamless bridge for legacy Gralloc binaries without requiring downstream ION code. |
| **Security Keystore** | Proprietary Keymaster daemon dependent on TrustZone QSEECOM (`/dev/qseecom`). | PureSoftKeyMint V5 AIDL service (`android.hardware.security.keymint-service`). | Complete software cryptographic service running in userspace; prevents `system_server` deadlock. |
| **Wi-Fi Subsystem** | Proprietary Qualcomm QCACLD-3.0 kernel module (`wlan.ko`). | Upstream in-kernel `ath11k` driver (`ath11k_pci`). | Eliminates module loading issues; binds directly to QCA6390 on PCIe bus 0. |
| **Bluetooth** | Downstream userspace transport daemon (`btqis`). | In-kernel `hci_uart` (`CONFIG_BT_HCIUART_QCA=y`). | Operates over `/dev/ttyHS0` (`&uart6`); auto firmware initialization via Linux BT stack. |

---

## 4. Key Technical Breakthroughs & Engineering Solutions

### 4.1. The DPU Bandwidth Ceiling Patch (`dpu_6_0_sm8250.h`)
* **Problem:** In upstream Linux, the SM8250 DPU hardware catalog defined `max_bw_high` as 16.6 Gbps (`16600000`). When Android 17 SurfaceFlinger performs multi-layer client composition at 1080x2400 @ 120Hz, the DPU driver calculates the required peak interconnect bandwidth to exceed 17.6 Gbps. As a result, the kernel rejected atomic display commits with `-E2BIG`:
  ```text
  [drm:dpu_crtc_atomic_check] crtc89 failed performance check -7
  ```
  This manifested as permanent black screens or severe display freezing upon rendering complex UI animations.
* **Solution:** Raised the performance bandwidth ceiling in `drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_0_sm8250.h`:
  ```c
  static const struct dpu_perf_cfg sm8250_perf_data = {
      .max_bw_low = 13700000,
  -   .max_bw_high = 16600000,
  +   .max_bw_high = 28500000,
      .min_core_ib = 4800000,
      .min_llcc_ib = 0,
      .min_dram_ib = 800000,
  ```
  This allows full 120Hz atomic frame submission without dropped frames.

### 4.2. Mesa Turnip Multi-Queue Exposure for Android HWUI
* **Problem:** The Android 17 HWUI RenderThread initializes the Vulkan backend (`SkiaVK`) and validates that the primary graphics/compute queue family provides at least 2 queues:
  ```text
  Assertion failed: queueProps[i].queueFamilyProperties.queueCount < kRequestedQueueCount
  ```
  Turnip default upstream only reported `queueCount = 1`, causing an immediate abort during early UI startup.
* **Solution:** Patched `src/freedreno/vulkan/tu_device.cc` in Mesa Turnip to expose `queueCount = 2`:
  ```cpp
  static const VkQueueFamilyProperties tu_queue_family_properties = {
     .queueFlags =
        VK_QUEUE_GRAPHICS_BIT | VK_QUEUE_COMPUTE_BIT | VK_QUEUE_TRANSFER_BIT,
  -   .queueCount = 1,
  +   .queueCount = 2,
     .timestampValidBits = 48,
     .minImageTransferGranularity = { 1, 1, 1 },
  };
  ```

### 4.3. Deterministic Binary Patching of `libadreno_utils.so` (UBWC Adreno 650 Fix)
* **Problem:** Qualcomm's proprietary Gralloc allocation library (`libgralloccore.so`) relies on `libadreno_utils.so` to query the GPU chip ID. In downstream kernels, it opens `/dev/kgsl-3d0` and ioctls the GPU. On mainline, `/dev/kgsl-3d0` does not exist, so `g_chip_id` defaulted to `-1`. This caused Gralloc to compute illegal Universal Bandwidth Compression (UBWC) strides, triggering fatal memory faults in `allocator-service`.
* **Solution:** Created an automated build-time Python patcher targeting both 64-bit and 32-bit `libadreno_utils.so` binaries. It calculates the `.data` section offset and overrides the static uninitialized global variable:
  ```python
  # g_chip_id = 0x06050002 (Adreno 650 hardware identifier)
  struct.pack_into("<I", data, expected_off, 0x06050002)
  ```
  This forces Gralloc to recognize the Adreno 650 hardware capabilities offline and calculate correct UBWC tiling parameters.

### 4.4. FocalTech FT8756 Touchscreen Driver Rework
The upstream driver in `drivers/input/touchscreen/ft8756.c` suffered from multiple severe flaws on Alioth:
1. **Dynamic Memory Allocation in Fast Path:** `ft8756_spi_write` and `ft8756_spi_read` called `devm_kzalloc` inside the high-frequency interrupt context for every single SPI transaction, resulting in heap fragmentation, latency spikes, and memory leaks.
   - *Fix:* Replaced dynamic allocations with pre-allocated 4KB DMA-safe buffers (`ts->spi_tx_buf`, `ts->spi_rx_buf`) protected by `ts->spi_lock`.
2. **Ghost Tap Filtering:** Spurious interrupts frequently caused the controller to transmit uninitialized packet payloads (`0xEF` and `0xFF` bytes), resulting in ghost taps and unresponsive touch registers.
   - *Fix:* Added an explicit packet validity filter checking coordinates prior to reporting input events.
3. **Regulator Configuration:** Corrected the regulator count from 3 (`vio`, `lab`, `ibb`) to 2 (`vio` @ 3.0V, `ibb` @ 1.8V), eliminating timeouts waiting for nonexistent `lab` regulators.
4. **DRM Panel Follower Integration:** Added a direct phandle link to the panel in DTS (`panel = <&panel>;`) and implemented `panel_prepared` and `panel_unpreparing` callbacks to automatically put the touch controller to sleep and wake it on display power state changes.

### 4.5. In-Kernel ION to DMA-BUF Heap Compatibility Driver (`ion-compat.c`)
* **Problem:** Stock Android vendor binaries and older Gralloc libraries still execute `ion_open()` and issue `ION_IOC_ALLOC` ioctls against `/dev/ion`. Upstream Linux removed the legacy staging ION driver in favor of standard DMA-BUF heaps (`/dev/dma_heap/system`).
* **Solution:** Developed an in-tree kernel driver (`drivers/dma-buf/ion-compat.c`) that:
  - Registers `/dev/ion` as a dynamic misc device.
  - Catches `ION_IOC_ALLOC` ioctls.
  - Dynamically fetches the `"system"` (or `"linux,cma"`) DMA-BUF heap via `dma_heap_find()`.
  - Allocates a contiguous/scatter-gather buffer using `dma_heap_buffer_alloc_fd()`.
  - Returns the exported file descriptor directly inside the legacy `ion_allocation_data.handle` field.

### 4.6. PureSoftKeyMint V5 & Zero-CPU Stub Daemons
* **Problem:** In Android 17, `system_server` halts early boot if `android.hardware.security.keymint` is unavailable. Stock ROMs require the Qualcomm TrustZone QSEECOM driver (`/dev/qseecom`). Furthermore, daemons like `qseecomd`, `netmgrd`, and `cnd` enter infinite restart loops when their hardware drivers are missing, consuming 100% of CPU cores.
* **Solution:**
  1. Built a native C++ AIDL `PureSoftKeyMint` daemon implementing:
     - `IKeyMintDevice/default` (KeyMint V5)
     - `IRemotelyProvisionedComponent/default` (RKP V3)
     - `ISharedSecret/default` (SharedSecret V1)
     - `ISecureClock/default` (SecureClock V1)
  2. Built `stub_sleeper.c` (a 5KB static binary with zero runtime libc dependencies) that sleeps indefinitely in 3600s cycles, replacing dead QTI binaries and pacifying Android init.
  3. Added `mainline_boot.sh` executed at `post-fs-data`:
     - Clears ephemeral stale SyntheticPassword files (`/data/system_de/0/spblob/*`) and `locksettings.db*`.
     - Waits for `sys.boot_completed=1` and automatically runs `wm dismiss-keyguard` and `input keyevent 82` to present an unlocked home screen.

---

## 5. The Universal In-RAM OverlayFS Trampoline Architecture

To enable genuine mainline Linux 6.19 to boot over any installed stock or custom ROM (MIUI, HyperOS, LineageOS, Clover) **without modifying or corrupting read-only dynamic partitions (`super`)**, an in-ramdisk dual-stage trampoline was engineered:

```
[Mainline Linux 6.19 Kernel Boots from boot_a / boot_b]
                      │
                      ▼
[First-Stage Ramdisk: preinit1]
  ├─ Mounts tmpfs at /postinstall (survives switch_root)
  ├─ Copies mainline_stage2 to /postinstall/ini
  ├─ Copies mainline_vendor_ovr.bin to /postinstall/vendor_ovr.bin
  ├─ Pre-creates UFS block partition symlinks (/dev/block/by-name/*)
  └─ Executes /init.aosp (patched to chain to /postinstall/ini)
                      │
                      ▼
[Second-Stage Trampoline: mainline_stage2 (/postinstall/ini)]
  ├─ Unpacks mainline_vendor_ovr.bin to /postinstall/vendor_ovr
  ├─ Mounts read-only OverlayFS: lowerdir=/postinstall/vendor_ovr:/vendor
  │    ├─ Overlays Mesa Turnip Vulkan (vulkan.adreno.so)
  │    ├─ Overlays DRM HWComposer (hwc3-service.drm on /dev/dri/card0)
  │    ├─ Overlays PureSoftKeyMint V5 & stubs out crashing QTI daemons
  │    └─ Injects updated fstab.qcom and vendor build props
  ├─ Creates /dev/dri/card0 & /dev/dri/renderD128 with graphics SELinux context
  ├─ Creates /dev/input/event0..7 and triggers uevent "add"
  ├─ Patches blocking init.rc directives (keystore wait, odsign wait, bpf wait)
  └─ Executes the real /system/bin/init
                      │
                      ▼
[Real Android /system/bin/init → SurfaceFlinger Boots Cleanly at 120Hz]
```

### Key Advantages
1. **Zero Partition Mutation:** Physical storage partitions (`system`, `vendor`, `product`, `odm`) remain 100% untouched.
2. **Instant Rollback:** Flashing the stock kernel restores stock downstream boot instantly.
3. **Universal Compatibility:** Works regardless of whether underlying partitions use EXT4 or EROFS.

---

## 6. Directory Structure & Complete Extraction Inventory

Every zip archive, boot image, sparse vendor image, and overlay container has been fully extracted and validated on disk in [`/serverhive3/zenzer0s/kernel/alioth/extracted/`](file:///serverhive3/zenzer0s/kernel/alioth/extracted/):

```text
/serverhive3/zenzer0s/kernel/
├── alioth/
│   ├── alioth_mainline_modifications.zip       # Complete source & binary drop-in package
│   ├── AnyKernel3-Alioth-Mainline-Universal-DropIn.zip  # Universal recovery flashing zip
│   ├── boot_clover_aosp_mainline_v38.img       # Production boot image (v38, 114.5MB)
│   ├── vendor_boot.img                         # Android VNDRBOOT v3 image (20.6MB)
│   ├── vendor_mainline.img                     # Android sparse vendor partition (1.38GB)
│   ├── Xiaomi_Alioth_Mainline_Architecture_Report.pdf  # Technical architecture whitepaper
│   │
│   └── extracted/                              # Fully unpacked & verified resources
│       ├── alioth_mainline_modifications/      # Unpacked modifications ZIP
│       │   ├── boot_image/                     # Contains boot_clover_aosp_mainline_v37.img
│       │   ├── firmware/                       # ath11k (QCA6390) and qca BT firmware
│       │   ├── graphics_turnip_hwc/            # Mesa Turnip Vulkan & drm_hwcomposer
│       │   ├── kernel_source_patches/          # ft8756.c, config, dts, dtb, patch
│       │   ├── keymint_hal/                    # SoftKeyMint C++ source & binary
│       │   ├── patches/                        # Gralloc and DRM patched libraries
│       │   ├── scripts/                        # Build scripts v21 through v37
│       │   ├── trampolines/                    # preinit1.c, mainline_stage2.c, stub_sleeper.c
│       │   └── vendor_overlay_tree/            # Unpacked staging vendor overlay tree
│       │
│       ├── anykernel3/                         # Unpacked AnyKernel3 drop-in installer
│       │   ├── anykernel.sh                    # Ramdisk patching & flashing script
│       │   ├── Image                           # Linux 6.19.6 ARM64 kernel Image (62.8MB)
│       │   ├── dtb                             # Compiled sm8250-xiaomi-alioth.dtb
│       │   ├── overlay/                        # mainline_vendor_ovr.bin & data_sparse.bin
│       │   ├── trampoline/                     # Static preinit1 & mainline_stage2 binaries
│       │   └── tools/                          # ak3-core.sh, patch_init, magiskboot
│       │
│       ├── boot_v38/                           # Unpacked boot_clover_aosp_mainline_v38.img
│       │   ├── Image                           # Extracted kernel binary (62,794,240 bytes)
│       │   ├── ramdisk.cpio.gz                 # Compressed ramdisk (51,589,167 bytes)
│       │   └── ramdisk/                        # Fully extracted root CPIO ramdisk tree
│       │       ├── init                        # Static preinit1 entrypoint
│       │       ├── init.aosp                   # Patched AOSP init (/postinstall/ini)
│       │       ├── mainline_stage2             # Secondary OverlayFS trampoline
│       │       ├── mainline_vendor_ovr.bin     # OVR1 overlay binary container
│       │       └── lib/firmware/               # Staged ath11k & bluetooth firmware
│       │
│       ├── boot_v38_vendor_ovr/                # 102 unpacked entries from mainline_vendor_ovr.bin
│       │   ├── bin/hw/                         # hwc3-service.drm, keymint-service, boot-service
│       │   ├── bin/mainline_boot.sh            # Auto-unlock & SyntheticPassword cleanup script
│       │   ├── etc/init/                       # Overlaid init .rc files (stubs & HALs)
│       │   ├── etc/vintf/                      # VINTF compatibility matrix & manifests
│       │   ├── lib64/                          # Patched libadreno_utils, PureSoftKeyMint, libsdedrm
│       │   ├── lib64/hw/vulkan.adreno.so       # 64-bit Mesa Turnip Vulkan driver
│       │   ├── lib/hw/vulkan.adreno.so         # 32-bit Mesa Turnip Vulkan driver
│       │   └── firmware/                       # ath11k & DSP firmware (adsp, cdsp, slpi, zap)
│       │
│       ├── vendor_boot/                        # Unpacked vendor_boot.img (VNDRBOOT v3)
│       │   ├── dtb                             # Qualcomm SM8250 DTB (132,164 bytes)
│       │   ├── vendor_ramdisk.cpio             # Extracted vendor ramdisk (20,437,449 bytes)
│       │   └── ramdisk/                        # Fully extracted first-stage vendor ramdisk
│       │
│       └── vendor_mainline/                    # Unsparsed & 7z extracted vendor partition
│           ├── app/, bin/, etc/, lib/, lib64/  # Complete Android 17 vendor filesystem
│           └── build.prop                      # Target props: Android 17 (SDK 37), Adreno 650
│
└── kebab/
    └── alioth-aosp-mainline/                   # Active Git Repository (Linux 6.19 tree)
        ├── arch/arm64/boot/dts/qcom/
        │   ├── sm8250-xiaomi-alioth.dts        # Device tree for POCO F3
        │   └── sm8250.dtsi                     # Base Snapdragon 870 SoC DTSI
        ├── arch/arm64/configs/
        │   └── alioth_aosp_defconfig           # Production kernel build configuration
        ├── drivers/
        │   ├── dma-buf/
        │   │   └── ion-compat.c                # In-kernel ION to DMA-BUF heap bridge
        │   ├── gpu/drm/
        │   │   ├── msm/disp/dpu1/catalog/
        │   │   │   └── dpu_6_0_sm8250.h        # 28.5 Gbps bandwidth patch
        │   │   └── panel/
        │   │       └── panel-samsung-ams667xx01.c  # E4 AMOLED panel driver
        │   └── input/
        │       ├── misc/aw8697.c               # AW8697 haptic driver
        │       └── touchscreen/ft8756.c        # FT8756 touchscreen driver
        ├── patches/
        │   ├── 0001-drm-msm-dpu-sm8250-raise-max-bw-high.patch
        │   ├── 0001-dt-bindings-input-awinic-aw86927-Add-Awinic-AW8697.patch
        │   ├── 0002-Input-aw8697-Add-Awinic-AW8697-haptics-driver.patch
        │   └── 0002-turnip-expose-queue-count-2-for-hwui.patch
        └── tools/alioth-aosp/
            ├── build_v30.sh / build_v31.sh     # Automated image build scripts
            ├── pack_ovr_v30.py / pack_ovr_v31.py # Overlay packaging scripts
            ├── preinit1.c                      # First-stage CPIO ramdisk entrypoint
            ├── mainline_stage2.c               # Second-stage OverlayFS trampoline
            ├── stub_sleeper.c                  # Zero-CPU daemon stub
            └── tiny_sys.h                      # Raw Linux syscall headers (nostdlib)
```

---

## 7. Step-by-Step Kernel Compilation & Image Packaging

### 7.1. Build Environment Setup
On Ubuntu 22.04 / 24.04 or Debian testing:
```bash
sudo apt update && sudo apt install -y \
    build-essential bison flex libssl-dev libelf-dev dwarves pahole \
    clang lld llvm device-tree-compiler cpio gzip python3
```

### 7.2. Compiling Kernel Image & Device Tree
```bash
cd /serverhive3/zenzer0s/kernel/kebab/alioth-aosp-mainline

# 1. Load the official Alioth AOSP defconfig
make ARCH=arm64 LLVM=1 alioth_aosp_defconfig

# 2. Build kernel Image and compiled Device Tree Blob (DTB)
make -j$(nproc) ARCH=arm64 LLVM=1 Image qcom/sm8250-xiaomi-alioth.dtb
```
The output binaries are generated at:
- `arch/arm64/boot/Image`
- `arch/arm64/boot/dts/qcom/sm8250-xiaomi-alioth.dtb`

### 7.3. Compiling Early Trampolines (`preinit1` & `mainline_stage2`)
Using the Android NDK Clang toolchain (`aarch64-linux-android35-clang`):
```bash
cd tools/alioth-aosp

# Compile stub_sleeper
$NDK/aarch64-linux-android35-clang -O2 -static -nostdlib -fno-builtin \
    -Wall -Wextra -Werror stub_sleeper.c -o stub_sleeper
$NDK/llvm-strip stub_sleeper

# Compile preinit1
$NDK/aarch64-linux-android35-clang -O2 -static -nostdlib -fno-builtin \
    -Wall -Wextra -Werror preinit1.c -o preinit1
$NDK/llvm-strip preinit1

# Compile mainline_stage2
$NDK/aarch64-linux-android35-clang -O2 -static -nostdlib -fno-builtin \
    -Wall -Wextra -Werror mainline_stage2.c -o mainline_stage2
$NDK/llvm-strip mainline_stage2
```

### 7.4. Packing the Universal Boot Image (`mkbootimg`)
```bash
CMDLINE="console=tty0 clk_ignore_unused pd_ignore_unused arm-smmu.disable_bypass=0 \
swiotlb=2048 lpm_levels.sleep_disabled=1 service_locator.enable=1 \
androidboot.usbcontroller=a600000.dwc3 androidboot.usbconfigfs=true loop.max_part=7 \
cgroup.memory=nokmem,nosocket androidboot.hardware=qcom androidboot.fstab_suffix=qcom \
androidboot.boot_devices=soc/1d84000.ufshc,soc@0/1d84000.ufshc,1d84000.ufshc \
androidboot.selinux=permissive panic=0 oops=panic printk.always_kmsg_dump=1 loglevel=8 \
sysctl.kernel.firmware_config.force_sysfs_fallback=1 firmware_class.path=/vendor/firmware"

mkbootimg \
    --header_version 2 \
    --os_version 17.0.0 \
    --os_patch_level 2026-10 \
    --kernel "arch/arm64/boot/Image" \
    --ramdisk "combined_ramdisk.cpio.gz" \
    --dtb "arch/arm64/boot/dts/qcom/sm8250-xiaomi-alioth.dtb" \
    --pagesize 4096 \
    --base 0x00000000 \
    --kernel_offset 0x00008000 \
    --ramdisk_offset 0x20000000 \
    --tags_offset 0x00000100 \
    --dtb_offset 0x22000000 \
    --cmdline "$CMDLINE" \
    --output "boot_clover_aosp_mainline.img"
```

---

## 8. Flashing & Deployment Protocol

### Method A: Universal Recovery Drop-In (TWRP / OrangeFox) — Recommended
This method preserves stock data and automatically modifies the ramdisk without partition mutations:
1. Boot the phone into custom recovery mode (TWRP or OrangeFox).
2. Connect USB and execute:
   ```bash
   adb sideload /serverhive3/zenzer0s/kernel/alioth/AnyKernel3-Alioth-Mainline-Universal-DropIn.zip
   ```
3. Reboot system directly from recovery.

### Method B: Direct Fastboot Partition Flashing
For devices flashing standalone boot and vendor images:
1. Boot device into Fastboot mode (`adb reboot bootloader` or `Volume Down + Power`).
2. Cancel snapshot locks and flash target partition:
   ```bash
   # Cancel virtual A/B snapshot locks
   fastboot snapshot-update cancel

   # Flash mainline kernel boot image
   fastboot flash boot_a /serverhive3/zenzer0s/kernel/alioth/boot_clover_aosp_mainline_v38.img

   # (Optional) Flash standalone mainline vendor image if not using in-RAM overlay
   fastboot flash vendor_a /serverhive3/zenzer0s/kernel/alioth/vendor_mainline.img

   # Set active slot and reboot
   fastboot --set-active=a
   fastboot reboot
   ```
3. The phone boots directly to the Android 17 lockscreen, runs `mainline_boot.sh`, unlocks the display, and presents an interactive 120Hz home screen.

---

## 9. Verification & Diagnostics via ADB

Once the system boots, verify the active mainline subsystems:

```bash
# 1. Verify kernel version and release name
adb shell uname -a
# Expected: Linux localhost 6.19.6-alioth-octavia23 #1 SMP PREEMPT ... aarch64

# 2. Verify DRM KMS display pipeline and modesetting
adb shell dumpsys SurfaceFlinger | grep -E "Display|HWC|Vulkan"
# Expected: 1080x2400 @ 120.0 Hz, HWC composer: drm_hwcomposer

# 3. Check GPU Vulkan rendering backend
adb shell getprop ro.hardware.egl
# Expected: angle
adb shell getprop debug.renderengine.backend
# Expected: skiavkthreaded

# 4. Check multi-touch input device registration
adb shell getevent -lp
# Expected: /dev/input/event*: "focaltech,ft8756-spi" with ABS_MT_POSITION_X/Y

# 5. Check in-kernel Wi-Fi (ath11k) interface
adb shell ip link show wlan0
# Expected: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN

# 6. Check PureSoftKeyMint V5 AIDL service registration
adb shell cmd servicemanager list | grep keymint
# Expected: android.hardware.security.keymint.IKeyMintDevice/default
```

---
*Xiaomi Alioth Mainline Kernel Project • Research & Documentation Completed: November 2026*
