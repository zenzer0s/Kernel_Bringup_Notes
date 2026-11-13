# Deep Architectural Comparison: Google Android Common Kernel (ACK 6.12) vs. PostmarketOS Mainline (Linux 6.19-Alioth)

**Analysis Scope:** Comparative code audit between:
1. **PostmarketOS + Akshat Yadav (@octavia-23) Tree:** `6.19.6-alioth-octavia23` (`alioth-aosp-mainline`)
2. **Google Android Common Kernel (ACK):** `android16-6.12` (`common-android16-6.12`)

---

## 1. Executive Summary & Comparison Matrix

| Component / Subsystem | PostmarketOS + Dev Patches (Linux 6.19) | Google ACK 6.12 (`android16-6.12`) | Status in Google ACK | Verdict & Technical Guidance |
| :--- | :--- | :--- | :---: | :--- |
| **In-Kernel DMA-BUF Heap APIs** | Manually patched `dma-heap.c` to export `dma_heap_find()` & `dma_heap_buffer_alloc_fd()`. | Natively implemented in `dma-heap.c` with tracepoints and vendor hooks. | **ALREADY PRESENT & SUPERIOR** | Google's implementation is cleaner, official, and includes latency tracing. |
| **ION Compatibility Bridge** | Custom in-tree driver `drivers/dma-buf/ion-compat.c` (168 lines) translating ioctls to DMA-BUF. | Completely eradicated. Google dropped ION in Android 12 GKI. | **NOT PRESENT (By Design)** | Necessary only when running legacy downstream Android 11 Qualcomm Gralloc blobs. |
| **DPU Bandwidth Ceiling** | Raised `max_bw_high` from 16.6 to **28.5 Gbps** in `dpu_6_0_sm8250.h`. | Unmodified upstream catalog value: `max_bw_high = 16600000` (16.6 Gbps). | **NOT PRESENT** | **MANDATORY PATCH**: Must be ported to ACK to prevent `-E2BIG` crashes at 120Hz. |
| **FocalTech FT8756 Touchscreen** | `ft8756.c` with pre-allocated 4KB DMA SPI buffers and panel follower lifecycle. | No FT8756 driver in tree (only generic `edt-ft5x06.c`). | **NOT PRESENT** | Must be ported directly from the PostmarketOS tree. |
| **Samsung E4 AMOLED Panel** | `panel-samsung-ams667xx01.c` with DSC 1.1 compression & 120Hz/90Hz modes. | Missing. Only generic and reference panels present. | **NOT PRESENT** | Must be ported directly from the PostmarketOS tree. |
| **Awinic AW8697 Haptics** | `aw8697.c` with `FF_RUMBLE` input interface and auto-resonance calibration. | Missing in `drivers/input/misc/`. | **NOT PRESENT** | Must be ported directly from the PostmarketOS tree. |
| **POCO F3 Device Tree (DTS)** | Complete `sm8250-xiaomi-alioth.dts` with all board mappings. | Missing. Only reference boards (HDK, MTP) and `pipa`/`elish`. | **NOT PRESENT** | Must be ported directly from the PostmarketOS tree. |
| **SELinux Enforcement** | Hardcoded `enforcing_enabled() { return false; }` in `security.h`. | Standard Linux/Android SELinux respecting `androidboot.selinux=permissive`. | **SUPERIOR IN ACK** | ACK's design is cleaner, preserving CTS compatibility on userdebug builds. |
| **Incremental FS (`incfs`)** | Not present in PostmarketOS base. | Natively integrated in `fs/incfs/` with VFS integrity verification. | **SUPERIOR IN ACK** | Required for Android 15/16/17 streaming app installations from Play Store. |
| **Binder IPC Subsystem** | Basic upstream staging Binder (~7,173 lines). | Production Android Binder (~7,640 lines) with vendor hooks & frozen process support. | **SUPERIOR IN ACK** | Prevents transaction deadlocks with Android 16/17 background frozen processes. |
| **Android Vendor Hooks** | None. PostmarketOS is a traditional Linux distro tree. | Hundreds of vendor hooks across scheduler, memory, power, and cpufreq. | **SUPERIOR IN ACK** | Enables modern Android EAS scheduling and HAL power-management extensions. |

---

## 2. In-Depth Subsystem Analysis

### 2.1. DMA-BUF Heaps & Memory Allocation
* **In PostmarketOS + Dev Patch:**  
  Standard upstream Linux does not export `dma_heap_find()` or `dma_heap_buffer_alloc()` to kernel modules or in-tree drivers. To build his `ion-compat.c` driver, the developer manually added these functions into `drivers/dma-buf/dma-heap.c`:
  ```c
  struct dma_heap *dma_heap_find(const char *name);
  int dma_heap_buffer_alloc_fd(struct dma_heap *heap, size_t len, u32 fd_flags, u64 heap_flags);
  EXPORT_SYMBOL_GPL(dma_heap_find);
  EXPORT_SYMBOL_GPL(dma_heap_buffer_alloc_fd);
  ```
* **In Google ACK 6.12:**  
  **Google ACK already has this natively implemented and in a much more robust form.**  
  In [`common-android16-6.12/drivers/dma-buf/dma-heap.c`](file:///serverhive3/zenzer0s/kernel/common-android16-6.12/drivers/dma-buf/dma-heap.c#L56-L109), Google provides:
  ```c
  struct dma_heap *dma_heap_find(const char *name);
  struct dma_buf *dma_heap_buffer_alloc(struct dma_heap *heap, size_t len, u32 fd_flags, u64 heap_flags);
  EXPORT_SYMBOL_GPL(dma_heap_find);
  EXPORT_SYMBOL_GPL(dma_heap_buffer_alloc);
  ```
  Furthermore, Google's version wraps allocations in low-overhead vendor tracepoints (`trace_android_vh_dma_heap_buffer_alloc_start`, `lat_start`, `lat_end`), providing real-time latency monitoring for Android's Choreographer and SurfaceFlinger.
* **Verdict:** **Google ACK is strictly superior.** The manual patch applied to PostmarketOS is redundant on ACK.

---

### 2.2. The ION Compatibility Layer (`ion-compat.c`)
* **In PostmarketOS + Dev Patch:**  
  The developer created [`drivers/dma-buf/ion-compat.c`](file:///serverhive3/zenzer0s/kernel/kebab/alioth-aosp-mainline/drivers/dma-buf/ion-compat.c). When legacy Qualcomm Gralloc binaries call `open("/dev/ion")` and `ioctl(ION_IOC_ALLOC)`, `ion-compat` intercepts the call, finds the `"system"` DMA-BUF heap, allocates the buffer, and exports the descriptor into `data.handle`.
* **In Google ACK 6.12:**  
  **Completely absent.** Google removed ION from modern kernels because modern Android userspace (Android 12+) is mandated to use DMA-BUF heaps directly (`/dev/dma_heap/system`).
* **Verdict:**  
  - If booting an AOSP ROM compiled with an **open-source Gralloc** (e.g. Minigbm or AIDL Allocator V2 using `/dev/dma_heap/system`), `ion-compat.c` is **not needed**.
  - If booting stock Clover / MIUI ROMs that reuse **Xiaomi's closed-source Android 11 blobs**, `ion-compat.c` (or the userspace `libion_shim.so`) must still be included.

---

### 2.3. DPU Bandwidth Ceiling (`dpu_6_0_sm8250.h`)
* **In PostmarketOS + Dev Patch:**  
  In [`dpu_6_0_sm8250.h`](file:///serverhive3/zenzer0s/kernel/kebab/alioth-aosp-mainline/drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_0_sm8250.h), `max_bw_high` was raised from `16600000` (16.6 Gbps) to `28500000` (28.5 Gbps).
* **In Google ACK 6.12:**  
  In [`common-android16-6.12/drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_0_sm8250.h`](file:///serverhive3/zenzer0s/kernel/common-android16-6.12/drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_0_sm8250.h#L351-L358):
  ```c
  static const struct dpu_perf_cfg sm8250_perf_data = {
      .max_bw_low = 13700000,
      .max_bw_high = 16600000,  /* <--- Still unpatched upstream limit! */
      .min_core_ib = 4800000,
  ```
* **Verdict:** **Google ACK DOES NOT have this patch.**  
  If you compile ACK 6.12 for Alioth without modifying this line, SurfaceFlinger client composition at 1080x2400 @ 120Hz **will crash with `-E2BIG` (performance check failed -7)**. **This patch must be ported to ACK.**

---

### 2.4. Touchscreen Driver (`ft8756.c`)
* **In PostmarketOS + Dev Patch:**  
  [`drivers/input/touchscreen/ft8756.c`](file:///serverhive3/zenzer0s/kernel/kebab/alioth-aosp-mainline/drivers/input/touchscreen/ft8756.c) was rewritten:
  1. Replaced per-event `devm_kzalloc()` memory allocation inside SPI read/write handlers with pre-allocated 4KB DMA buffers (`ts->spi_tx_buf`, `ts->spi_rx_buf`).
  2. Integrated DRM panel follower callbacks (`panel_prepared` / `panel_unpreparing`) to synchronize touch sleep/wake states with the AMOLED panel.
  3. Corrected regulator count from 3 to 2 (`vio` @ 3.0V, `ibb` @ 1.8V).
  4. Added touch coordinate filter discarding invalid `0xEF`/`0xFF` packets.
* **In Google ACK 6.12:**  
  **Does not exist.** ACK 6.12 only includes generic upstream touchscreen drivers.
* **Verdict:** **Must be ported from PostmarketOS to ACK.** Without this driver, the POCO F3 has zero touchscreen input.

---

### 2.5. Display Panel & Haptics Drivers
* **Samsung E4 AMOLED Panel (`panel-samsung-ams667xx01.c`):**  
  ACK 6.12 contains no driver for the POCO F3 panel. The driver from PostmarketOS (including DSC 1.1 compression, DCS panel init commands, and 120Hz/90Hz timings) **must be ported to ACK.**
* **Awinic AW8697 Haptics (`aw8697.c`):**  
  ACK 6.12 contains no driver for the AW8697 IC. The driver from PostmarketOS **must be ported to ACK.**
* **Device Tree (`sm8250-xiaomi-alioth.dts`):**  
  ACK 6.12 contains DTs for Xiaomi Pad 6 (`pipa`) and Pad 5 Pro (`elish`), but **none for Alioth**. The DTS must be copied over.

---

### 2.6. SELinux Security Architecture
* **In PostmarketOS + Dev Patch:**  
  In [`security/selinux/include/security.h`](file:///serverhive3/zenzer0s/kernel/kebab/alioth-aosp-mainline/security/selinux/include/security.h), the developer hardcoded:
  ```c
  static inline bool enforcing_enabled(void) {
      return false; /* Force permissive */
  }
  ```
  While effective for early bringup, this breaks Android CTS/VTS testing and disables security boundaries permanently.
* **In Google ACK 6.12:**  
  Retains standard, production-grade SELinux infrastructure. Permissive mode can be toggled cleanly via kernel command line (`androidboot.selinux=permissive`) during development without mutilating kernel header sources.
* **Verdict:** **Google ACK is strictly better.**

---

### 2.7. Android Core Infrastructure (Where Google ACK Dominates)
Google ACK 6.12 incorporates core Android features that are completely absent in the PostmarketOS mainline tree:

1. **Incremental FS (`fs/incfs/`):**  
   - Present in ACK 6.12; **absent** in PostmarketOS.  
   - Essential in modern Android for streaming large APK/AAB game installations from Google Play Store.
2. **Production Android Binder (`drivers/android/binder.c`):**  
   - ACK 6.12 includes Android 16/17 Binder enhancements: frozen process transaction buffering, priority inheritance tracking, and vendor hooks. PostmarketOS has an unoptimized generic upstream staging driver.
3. **Android Vendor Hooks (`include/trace/hooks/`):**  
   - ACK 6.12 features hooks for Energy Aware Scheduling (EAS), CPU frequency governors, memory compaction, and thermal limits. PostmarketOS has zero vendor hooks.
4. **GKI 2.0 Modular Architecture:**  
   - ACK 6.12 builds as a clean, standardized Generic Kernel Image with a protected Kernel Module Interface (KMI).

---

## 3. Migration Roadmap: Bringing Alioth to Google ACK 6.12

To port the POCO F3 (Alioth) onto Google ACK 6.12, only the hardware-specific delta needs to be transplanted. The core Android infrastructure should be left to ACK.

### Step-by-Step Porting Checklist:
1. **Port Display DPU Bandwidth Patch:**  
   In `drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_0_sm8250.h`, change `max_bw_high` to `28500000`.
2. **Copy Hardware Drivers:**  
   - `drivers/gpu/drm/panel/panel-samsung-ams667xx01.c` (Panel)
   - `drivers/input/touchscreen/ft8756.c` (Touchscreen)
   - `drivers/input/misc/aw8697.c` (Haptics)
3. **Copy Device Tree:**  
   - `arch/arm64/boot/dts/qcom/sm8250-xiaomi-alioth.dts`
4. **Update Kconfig and Makefiles:**  
   - Add `CONFIG_DRM_PANEL_SAMSUNG_AMS667XX01=y`
   - Add `CONFIG_TOUCHSCREEN_FT8756=y`
   - Add `CONFIG_INPUT_AW8697=y`
5. **Skip Redundant Patches:**  
   - **Do NOT** patch `dma-heap.c` (ACK already has `dma_heap_find` and `dma_heap_buffer_alloc`).
   - **Do NOT** patch `security.h` (pass `androidboot.selinux=permissive` in cmdline instead).
   - Only include `ion-compat.c` if maintaining backward compatibility with legacy Android 11 vendor blobs.

---

## 4. Qualcomm Kona (SM8250) Silicon & Driver Comparison: Google ACK 6.12 vs. PostmarketOS Base (Excluding Dev Patches)

This section isolates the **Qualcomm Kona silicon baseline** present in **Google ACK 6.12** (`android16-6.12`) versus the **PostmarketOS upstream base** (`commit b0d4bec5d511cc07ed436ca842c59eac3929aa4c`), strictly excluding any patches added by Akshat Yadav (`@octavia-23`).

### 4.1. High-Level Inventory Overview

| Category | PostmarketOS 6.19.6 Base | Google ACK 6.12 | Key Architectural Takeaway |
| :--- | :---: | :---: | :--- |
| **Total Kona/SM8250 Files** | **73 files** | **66 files** | PostmarketOS maintains significantly more real-world consumer hardware trees. |
| **Common Silicon Files** | 63 files | 63 files | Core clock, interconnect, UFS, PCIe drivers are upstream-aligned in both. |
| **Consumer Device DTS** | **11 devices** | **4 devices** | ACK only tracks reference boards + Sony Xperia & Xiaomi Pad. |
| **TLMM Pinctrl (`pinctrl-sm8250.c`)** | Extended (`egpio`) | Standard | PostmarketOS includes extended GPIO routing on pins 146–179. |
| **Sound Machine (`sm8250.c`)** | Full Routing (306 lines) | Minimal (197 lines) | PostmarketOS integrates USB audio offload, DP audio, and TDM slot offsets. |
| **SoC Device Tree (`sm8250.dtsi`)** | 7,084 lines | 7,028 lines | PostmarketOS contains 7 kernel cycles of upstream DTSI refactorings (`refgen`, lowercase labels). |

---

### 4.2. Consumer Device Tree Inventory (DTS)

#### Present in PostmarketOS Base but COMPLETELY ABSENT in Google ACK 6.12:
PostmarketOS targets consumer mobile devices, maintaining dedicated hardware trees:
1. **OnePlus 8T** (`arch/arm64/boot/dts/qcom/sm8250-oneplus-kebab.dts`)
2. **OnePlus 8 Pro** (`arch/arm64/boot/dts/qcom/sm8250-oneplus-instantnoodlep.dts`)
3. **Samsung Galaxy S20 5G** (`arch/arm64/boot/dts/qcom/sm8250-samsung-x1q.dts`)
4. **Samsung Galaxy S20 FE** (`arch/arm64/boot/dts/qcom/sm8250-samsung-r8q.dts`)
5. **Samsung Common DTSI** (`arch/arm64/boot/dts/qcom/sm8250-samsung-common.dtsi`)
6. **Lenovo Tab P12 Pro** (`arch/arm64/boot/dts/qcom/sm8250-lenovo-q706f.dts`)
7. **Xiaomi Pad 5 Pro 5G** (`arch/arm64/boot/dts/qcom/sm8250-xiaomi-enuma.dts`)
8. **Xiaomi Pad 6 Multi-Panel DTS** (`sm8250-xiaomi-pipa-csot.dts`, `sm8250-xiaomi-pipa-tianma.dts`, `pipa-common.dtsi`)

#### Device Trees Present in Google ACK 6.12:
Google ACK tracks Qualcomm reference boards and a narrow set of vendor targets:
1. **Qualcomm Reference Platforms**: `sm8250-hdk.dts` (Hardware Development Kit), `sm8250-mtp.dts` (Mobile Test Platform)
2. **Sony Xperia 1 II & 5 II**: `sm8250-sony-xperia-edo-pdx203.dts`, `sm8250-sony-xperia-edo-pdx206.dts`, `sm8250-sony-xperia-edo.dtsi`
3. **Xiaomi Pad 5 Pro (Wi-Fi)**: `sm8250-xiaomi-elish-boe.dts`, `sm8250-xiaomi-elish-csot.dts`, `sm8250-xiaomi-elish-common.dtsi`
4. **Xiaomi Pad 6**: Unified `sm8250-xiaomi-pipa.dts` (upstreamed in 6.12)

---

### 4.3. Core SoC Device Tree Refactorings (`sm8250.dtsi`)

Between Linux 6.12 (ACK) and Linux 6.19 (PostmarketOS), upstream Linux underwent extensive device tree standardizations:
* **Label Schema Modernization**: PostmarketOS standardizes all labels to lowercase (`cpu0`, `l2_0`, `cpu_pd0`, `aoss_qmp`), whereas ACK 6.12 still retains uppercase legacy labels (`CPU0`, `L2_0`, `CPU_PD0`).
* **USB/DP PHY Reference Generator (`refgen`)**: PostmarketOS includes the dedicated reference generator regulator node:
  ```dts
  refgen: regulator@88e7000 {
      compatible = "qcom,sm8250-refgen-regulator";
      reg = <0x0 0x088e7000 0x0 0x84>;
  };
  ```
  *(Absent in ACK 6.12).*

---

### 4.4. Silicon Driver Differences

#### A. TLMM Pin Controller (`drivers/pinctrl/qcom/pinctrl-sm8250.c`)
* **PostmarketOS Base:** Implements Qualcomm **Extended GPIO (`egpio`)**:
  - Registers `egpio_enable` (bit 12) and `egpio_present` (bit 11).
  - Binds function `msm_mux_egpio` across `egpio_groups` (GPIOs 146 through 179).
* **Google ACK 6.12:** Lacks `egpio` definitions completely.

#### B. Audio Machine Driver (`sound/soc/qcom/sm8250.c`)
* **PostmarketOS Base (306 lines):**
  - **USB Audio Offload Jack**: Sets up `qcom_snd_usb_offload_jack_setup` via `usb_offload_utils.h`.
  - **DisplayPort Audio Jack**: Implements `qcom_snd_dp_jack_setup` for `DISPLAY_PORT_RX`.
  - **TDM Audio Routing**: Configures dedicated TDM slot offsets (`tdm_slot_offset[8] = {0, 4, 8, 12, 16, 20, 24, 28}`) with `TDM_BCLK_RATE = 12288000` to support modern I2S/TDM smart amplifiers.
* **Google ACK 6.12 (197 lines):**
  - Minimal implementation: only configures Qualcomm internal WCD codec jack detection (`qcom_snd_wcd_jack_setup`).

#### C. DRM DPU Display Catalog (`drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_6_0_sm8250.h`)
* **PostmarketOS Base (388 lines):**
  - Adopts Linux 6.19 active control cleanups and mixer layer masks (`MIXER_MSM8998_MASK`).
* **Google ACK 6.12 (413 lines):**
  - Retains older 6.12 active control definitions: `BIT(DPU_CTL_ACTIVE_CFG) | BIT(DPU_CTL_SPLIT_DISPLAY)` and `MIXER_SDM845_MASK`.
* *(Note: Both trees in unpatched state share `.max_bw_high = 16600000`, requiring Akshat's 28.5 Gbps bandwidth patch).*

---

### 4.5. Strategic Takeaway for Porting to ACK 6.12

When porting a Kona/SM8250 target (such as Alioth or Kebab) onto Google ACK 6.12:
1. **What ACK gives you for free:** The robust Android GKI platform, in-kernel DMA-BUF heap APIs (`dma_heap_find`), modern Binder with frozen process support, and Android Vendor Hooks.
2. **What you must cherry-pick from PostmarketOS:**
   - The device DTS file (e.g. `sm8250-xiaomi-alioth.dts` or `sm8250-oneplus-kebab.dts`).
   - Extended GPIO (`egpio`) additions in `pinctrl-sm8250.c`.
   - Advanced audio routing (USB offload & TDM slot configuration) in `sound/soc/qcom/sm8250.c`.
   - The DPU bandwidth ceiling bump (`28.5 Gbps`) in `dpu_6_0_sm8250.h`.

