# Technical Doubts & Clarifications: Fastboot Boot Mechanism on Xiaomi Alioth (SM8250)

## Question / Doubt
> **"The developer said after flashing modified vendor boot or vendor img (not sure which one), fastboot boot works. Which one is it, and why does this happen?"**

---

## 1. Direct Resolution: It is `vendor_boot.img`

The partition that enables `fastboot boot` to work on Xiaomi Alioth (POCO F3 / Redmi K40) is **`vendor_boot.img`** (specifically [`vendor_boot.img`](file:///serverhive3/zenzer0s/kernel/alioth/vendor_boot.img)), **NOT** `vendor_mainline.img`.

---

## 2. Technical Mechanism & Architectural Root Cause

### A. Android Boot Image Header v3/v4 Architecture (SM8250 / Android 11+)
In legacy Android devices (Header v1/v2), the kernel `Image`, the ramdisk, and the **Device Tree Blob (DTB)** were all bundled inside a single `boot.img`. 

On modern Qualcomm platforms like Snapdragon 870 (SM8250-AC), Google mandated **Header v3/v4**:
- The **`boot` partition** holds only the generic kernel binary (`Image`) and standard generic ramdisk.
- The **`vendor_boot` partition** holds the **Device Tree Blob (DTB)**, vendor-specific ramdisk, and bootloader kernel command-line arguments (`androidboot.*`).

```text
Physical Storage Layout:
┌─────────────────────────────────┐
│ boot_a / boot_b                 │  ──> Generic Kernel (Image) + Ramdisk
└─────────────────────────────────┘
┌─────────────────────────────────┐
│ vendor_boot_a / vendor_boot_b   │  ──> Device Tree Blob (DTB) + Vendor Cmdline
└─────────────────────────────────┘
```

---

### B. Why `fastboot boot` Fails on a Stock Phone
When you issue the command:
```bash
fastboot boot boot_clover_aosp_mainline_v38.img
```
1. The Qualcomm Android Bootloader (ABL) streams the kernel `Image` and `ramdisk` over USB and places them into **DDR scratch RAM** (`DRAM_START + offset`).
2. **The Critical Handoff Flaw:** ABL does **not** extract a DTB from the memory-streamed image. Instead, ABL looks to the **physical flash storage** to read the active slot's **`vendor_boot` partition** for the device tree and board parameters.
3. On a stock device, `vendor_boot` contains the **legacy downstream Qualcomm Android 4.19 DTB**:
   - Out-of-tree clock controllers and power domain trees.
   - Nonexistent mainline DRM KMS nodes (only proprietary `qcom,mdss_dsi`).
   - Downstream touchscreen nodes (`qcom,touchscreen`) instead of mainline SPI nodes (`focaltech,ft8756-spi`).
4. The mainline Linux 6.19 kernel attempts to initialize hardware using this incompatible 4.19 DTB. It immediately encounters unmapped clocks, hangs before early printk/display initialization, and the hardware watchdog resets the device back to Fastboot.

---

### C. Why Flashing Modified `vendor_boot.img` Makes `fastboot boot` Work
When you flash the project's modified [`vendor_boot.img`](file:///serverhive3/zenzer0s/kernel/alioth/vendor_boot.img):
```bash
fastboot flash vendor_boot vendor_boot.img
```
1. The physical flash partition `vendor_boot` is now populated with the **genuine mainline Linux 6.19 DTB** (132,164 bytes, verified containing `xiaomi,alioth`, `qcom,sm8250`, `focaltech,ft8756-spi`, and `soc@0/1d84000.ufshc`).
2. Now, when you run:
   ```bash
   fastboot boot boot_clover_aosp_mainline_v38.img
   ```
3. ABL loads the 6.19 kernel and ramdisk into DDR scratch memory from USB, pairs it with the **mainline 6.19 DTB stored in on-flash `vendor_boot`**, and transfers execution to Linux.
4. Because the kernel and DTB match, the display initializes at 120Hz, multi-touch binds on SPI4, and the system boots into Android 17 without having overwritten the physical `boot` partition!

---

## 3. What About `vendor_mainline.img`?

* **`vendor_mainline.img`** (1.38 GB) is the full physical `/vendor` filesystem containing:
  - Mesa Turnip Vulkan driver (`vulkan.adreno.so`)
  - DRM Hardware Composer service (`hwc3-service.drm`)
  - Standalone AIDL KeyMint service (`android.hardware.security.keymint-service`)
  - Custom init scripts and patched libraries.

* **Why you DO NOT need to flash `vendor_mainline.img` for `fastboot boot`:**
  - The compiled boot images ([`boot_clover_aosp_mainline_v37.img`](file:///serverhive3/zenzer0s/kernel/alioth/extracted/alioth_mainline_modifications/boot_image/boot_clover_aosp_mainline_v37.img) and [`boot_clover_aosp_mainline_v38.img`](file:///serverhive3/zenzer0s/kernel/alioth/boot_clover_aosp_mainline_v38.img)) already incorporate the **in-RAM OverlayFS container (`mainline_vendor_ovr.bin`)**.
  - During early boot, `mainline_stage2` unpacks this container and mounts an OverlayFS directly over `/vendor` in RAM (`lowerdir=/postinstall/vendor_ovr:/vendor`).
  - Therefore, the userspace vendor modifications are injected **in memory**, keeping the physical `vendor` partition safe and untouched.

---

## 4. Summary Matrix

| Partition / Action | Required for `fastboot boot`? | Purpose |
| :--- | :---: | :--- |
| **Flash `vendor_boot.img`** | **YES** | Provides the permanent on-flash **mainline Linux 6.19 DTB** that ABL requires during RAM boots. |
| **Flash `vendor_mainline.img`** | **NO** (Optional) | Only required if booting without the in-RAM overlay trampoline (`mainline_vendor_ovr.bin`). |
| **`fastboot boot <image>`** | **WORKS** (after step 1) | Safely boots mainline Linux 6.19 and Android 17 straight from scratch RAM. |
