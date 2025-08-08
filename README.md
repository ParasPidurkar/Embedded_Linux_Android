# Embedded_Linux_Android


# Amlogic SC2 (S905X4-gen) Boot Flow & Memory Map

This README captures the boot sequence and where each stage executes/loads, based on the UART logs you shared. It’s copy-paste ready.

---

## High-Level Flow

BootROM (on-chip)
→ BL2 (vendor preloader)
→ BL2E (vendor extended)
→ BL2X (vendor helper)
→ BL31 (TF-A, EL3)
→ BL32 (TEE, optional, S-EL1)
→ BL33Z (U-Boot decompressor stub)
→ BL33 (U-Boot proper)
→ Kernel (Android/Linux)


**What each stage does (one line):**
- **BootROM**: Probes boot media (eMMC), selects image, prints chip/fuse banner.
- **BL2**: PLL init, **DDR training** (1D/2D), RAM sizing, loads DDR and device FIPs.
- **BL2E/BL2X**: Board/eMMC tuning; parses FIP and hands off.
- **BL31 (TF-A)**: Secure EL3 init, starts **AOCPU** FW, decompresses BL33.
- **BL32 (TEE)**: Secure OS (ATOS/OP-TEE) for keys/DRM/secure services.
- **BL33Z**: Tiny pre-UBoot stub for compressed BL33.
- **BL33 (U-Boot)**: Relocates to DRAM, GPT/A/B/dynamic partitions, **AVB2** checks, loads kernel/DT.

---

## Mermaid Diagram

> If your Markdown viewer supports Mermaid:

```mermaid
flowchart LR
  R[BootROM (on-chip)] --> B2[BL2 (vendor preloader)]
  B2 --> B2E[BL2E (extended)]
  B2E --> B2X[BL2X (helper)]
  B2X --> BL31[BL31 (TF-A, EL3)]
  BL31 --> BL32[BL32 (TEE, S-EL1)]
  BL31 --> BL33Z[BL33Z (U-Boot stub)]
  BL33Z --> BL33[BL33 (U-Boot)]
  BL33 --> K[Kernel]




## SC2 Boot Flow & Memory Map

| Order | Stage   | Component / Name          | Key Actions                                                                 | Runs From (Memory)                                      | Loads From (Storage)                                              | Dest / Addr Seen                          | Log Evidence                                                                                 |
|------:|---------|----------------------------|------------------------------------------------------------------------------|---------------------------------------------------------|-------------------------------------------------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------------|
| 0     | BootROM | ROM code (on-chip)         | Reads eMMC boot area, prints SC2 chip/fuse banner, decides boot path        | On-chip ROM (immutable)                                  | eMMC boot partitions (multiple offsets)                            | –                                         | `SC2:BL:...; FEAT:...; POC:FF; build in ddr magic:ddr4`                                       |
| 1     | BL2     | Vendor preloader           | PLL init, DDR training (1D/2D), size detect, loads DDRFIP                   | On-chip SRAM / IRAM → then uses DRAM after training      | eMMC boot area (DDRFIP @ multiple offsets)                         | `dst f700ab90 (DDRFIP)`                   | `BL2 1.1.1 ...; 1D training succeed; 2D training succeed; DDR size: 3856MB`                   |
| 2     | BL2E    | BL2 Extended (vendor)      | Board/platform setup, eMMC tuning, loads BL3X (FIP payload)                 | On-chip SRAM / IRAM                                      | FIP header/payload @ eMMC offset `0x000a4200`                      | `FIP HDR des 0x00308000; BL3X des 0x00310000` | `BL2E 1.0.0 ... Start to do bl2e platform setup !; aml log : BL2E load BL3X`                  |
| 3     | BL2X    | BL2 helper (vendor)        | Final prep, passes params to BL31                                            | On-chip SRAM / IRAM                                      | –                                                                 | `params addr 0x0100d360`                  | `Hello, we are in BL2X world !; run into bl31`                                                |
| 4     | BL31    | TF-A (ATF)                 | EL3 init, world switch setup, starts AOCPU FW, decompresses BL33            | Secure SRAM/DRAM (platform-specific)                     | From FIP (already in RAM)                                         | `AOCPU FW: 0xf7028000–0xf7034400`         | `NOTICE: BL31 ...; start aocpu; BL33 decompress pass`                                         |
| 5     | BL32    | ATOS / OP-TEE (TEE)        | Secure OS start (S-EL1), secure timer/MMU, services for DRM/keys            | Secure DRAM                                              | From FIP (already in RAM)                                         | –                                         | `E/TC: BL3-2: ATOS-V3.8.4 ...; Secure Timer Initialized`                                      |
| 6     | BL33Z   | U-Boot decompressor stub   | Prepares and jumps to U-Boot proper                                          | IRAM/early DRAM                                          | From FIP (already in RAM, compressed)                             | –                                         | `"Hello world, Now in BL33Z."`                                                                |
| 7     | BL33    | U-Boot proper              | Relocation to DRAM, board init, GPT/A/B/dynamic partitions, AVB2 verification| DRAM (relocated)                                         | Kernel/DT from `boot_a`, `vendor_boot_a`, `dtbo_a` (eMMC user area) | `Relocation: 0xdfe0e000`                  | `U-Boot 2019.01 ...; Relocating to dfe0e000; AVB2 verified with fip key success`              |
