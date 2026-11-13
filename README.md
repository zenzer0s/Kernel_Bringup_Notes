# Kernel Bringup Notes & Hardware Research

Comprehensive documentation, architectural analysis, and hardware bringup notes for mainline Linux kernel ports on mobile Qualcomm Snapdragon platforms.

## Repository Documentation Index

1. **[POCO F3 / Redmi K40 (alioth) Mainline Linux 6.19 Bringup Guide](bringup_alioth.md)**:
   - Comprehensive system engineering report on running mainline Linux 6.19 on Android 17 (Clover AOSP / SDK 37).
   - In-RAM OverlayFS trampoline architecture (`preinit1` → `mainline_stage2`).
   - Hardware fixes: DPU bandwidth ceiling (28.5 Gbps), FT8756 zero-alloc SPI DMA rewrite, PureSoftKeyMint V5 AIDL service.

2. **[Fastboot Boot Failure Mode & vendor_boot Clarifications](doubts.md)**:
   - Technical root cause of why `fastboot boot` crashes on stock SM8250 (Android Boot Header v3/v4 DTB separation).
   - Why flashing modified `vendor_boot.img` enables `fastboot boot` to work.
   - Analysis of `vendor_boot.img` vs. physical `vendor_mainline.img`.

3. **[Google Android Common Kernel (ACK 6.12) vs. PostmarketOS Base Comparison](ack_vs_pos.md)**:
   - Code audit comparing Google ACK 6.12 (`android16-6.12`) with PostmarketOS Linux 6.19.
   - Analysis of in-kernel DMA-BUF heaps, ION compatibility, DPU bandwidth limits, and SELinux policies.
   - Detailed breakdown of Qualcomm Kona (SM8250) silicon drivers and consumer device trees present in each tree.

4. **[Open-Source vs. Downstream Proprietary Alternatives Matrix (SM8250)](opensource_vs_downstream_sm8250.md)**:
   - Subsystem-by-subsystem audit of open-source alternatives for Qualcomm downstream 4.19 drivers.
   - Breakdown of fully solved subsystems (Mesa Turnip GPU, ANGLE GLES, DRM HWC, in-kernel ath11k Wi-Fi, PureSoftKeyMint).
   - Analysis of hybrid subsystems (ADSP Audio, QRTR Cellular, Sensors) and the remaining camera/VoLTE frontiers.
