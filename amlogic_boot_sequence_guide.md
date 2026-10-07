# Exhaustive Amlogic SC2 Android STB Bootloader & Kernel Architecture Manual

This manual serves as the **single authoritative technical reference** for the complete boot sequence, low-level hardware initialization, memory layout, storage drivers, display pipelines, and Linux kernel startup on the **Amlogic SC2 (S905X4 / SC2_S905X4_AH212) Android Set-Top Box (STB) platform**.

---

## 1. Complete Multi-Stage Boot Sequence Architecture

The boot chain transitions through **six distinct execution environments** from silicon power-on to the Android TV launcher:

```
  ┌────────────────────────┐
  │ Stage 1: Mask ROM      │  <-- On-Chip Silicon ROM (BL1) [EL3]
  └───────────┬────────────┘
              │
              ▼
  ┌────────────────────────┐
  │ Stage 2: BL2 / BL2E    │  <-- Closed-Source Vendor Blobs (DDR Training)
  └───────────┬────────────┘
              │
              ▼
  ┌────────────────────────┐
  │ Stage 3: BL31 / BL32   │  <-- ARM Trusted Firmware (ATF) & OP-TEE OS [EL3 / Secure EL1]
  └───────────┬────────────┘
              │
              ▼
  ┌────────────────────────┐
  │ Stage 4: BL33 (U-Boot) │  <-- Open-Source U-Boot v2019 [EL2]
  └───────────┬────────────┘
              │
              ▼
  ┌────────────────────────┐
  │ Stage 5: Linux Kernel  │  <-- Linux 5.15 (GKI) [EL1]
  └───────────┬────────────┘
              │
              ▼
  ┌────────────────────────┐
  │ Stage 6: Android /init │  <-- Android User Space [EL0]
  └────────────────────────┘
```

---

### **Detailed Stage-by-Stage Breakdown**

#### **Stage 1: Mask ROM (BL1)**
* **Execution Level**: Exception Level 3 (EL3).
* **Source**: Etched directly into the Amlogic SC2 silicon die during semiconductor fabrication (Closed Source).
* **Responsibilities**:
  1. Executes immediately when power is applied to the SoC.
  2. Configures minimal system clocks (24 MHz Crystal Oscillator).
  3. Checks hardware strap pins to select boot medium order (eMMC 5.1 $\rightarrow$ SD Card $\rightarrow$ USB DFU Burn Mode).
  4. Reads the initial 64 KB header of **BL2** into internal SRAM (`0xD9000000`).
  5. Authenticates BL2 using the SoC root public key stored in eFuse.
  6. Branches execution to BL2.

#### **Stage 2: BL2, BL2E, and BL2X (Vendor Extension Stage)**
* **Execution Level**: Secure EL1 / EL3.
* **Source**: Closed-source Amlogic vendor binaries (`bl2_new.bin`, `bl2e.bin`, `bl2x.bin`).
* **Responsibilities**:
  1. **PLL Clock Initialization**: Scales CPU clock to 1.2 GHz and DDR clock to 912 MHz, later ramping up to 1320 MHz.
  2. **DRAM Hardware Training**:
     * Runs **LPDDR4 PHY 1D & 2D Training** (`LPDDR4_PHY_SC2_0_1_32`).
     * Determines Rank 0 (CS0: 2048 MB) and Rank 1 (CS1: 2048 MB) capacities.
     * Total Physical DRAM detected: **3856 MB** (after reserving secure region).
  3. **FIP (Firmware Image Package) Extraction**:
     * Reads FIP package from eMMC offset `0x00000000`.
     * Validates SHA-256 integrity checksums (`SHA CHK OK!`).
  4. **Payload Loading**:
     * Loads **BL31** to RAM `0x007FFFF0`.
     * Loads **BL32 (OP-TEE)** to RAM `0x00FFFFFF`.
     * Loads **BL33 (U-Boot)** to RAM `0x01000000`.
  5. Jumps to BL31.

#### **Stage 3: ARM Trusted Firmware (BL31) & OP-TEE (BL32)**
* **Execution Level**: EL3 (BL31) and Secure EL1 (BL32).
* **Source**: Open-source ARM Trusted Firmware + Amlogic SoC hooks.
* **Responsibilities**:
  1. **BL31 (ATF)**:
     * Sets up the EL3 Secure Monitor.
     * Configures Power Management (PSCI v1.0) function IDs.
     * Starts the **AOCPU (Always-On CPU)** core running FreeRTOS (`AOCPU image version='bl-3.5.15'`).
     * Configures PMP (Physical Memory Protection) for range `0xF7028000 ~ 0xF7034000`.
  2. **BL32 (OP-TEE v3.8 / ATOS)**:
     * Initializes the Secure World OS.
     * Sets up the Secure Timer.
     * Manages eFuse keys, Widevine DRM Keymaster decryption, and RPMB (Replay Protected Memory Block) storage.
  3. Transfers control to **BL33 (U-Boot)** at Non-Secure EL2.

#### **Stage 4: U-Boot v2019 (BL33)**
* **Execution Level**: Non-Secure Exception Level 2 (EL2).
* **Source**: Open-source in workspace ([`bootloader/uboot-repo/bl33/v2019/`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/bootloader/uboot-repo/bl33/v2019/)).
* **Responsibilities**: Full hardware setup, eMMC partition parsing, HDMI EDID negotiation, AVB 2.0 verification, DTBO overlay patching, loading kernel to RAM, and executing `booti`.

#### **Stage 5: Linux Kernel 5.15**
* **Execution Level**: Non-Secure Exception Level 1 (EL1).
* **Source**: Open-source GKI in workspace ([`common/common14-5.15/common/`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/common/common14-5.15/common/)).
* **Responsibilities**: Memory management (CMA), driver probing (`do_initcalls`), SMP multi-core wake-up via PSCI, mounting Android partitions (`system_a`, `vendor_a`).

#### **Stage 6: Android `/init`**
* **Execution Level**: User Space (EL0).
* **Responsibilities**: Launches Android init system, starts `servicemanager`, `surfaceflinger`, `zygote`, and boots the Android TV Launcher UI.

---

## 2. Deep Dive into U-Boot Source Code & Execution Flow

### **2.1 Low-Level Assembly Entry Point (`start.S`)**

* **File Path**: [`bootloader/uboot-repo/bl33/v2019/arch/arm/cpu/armv8/start.S`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/bootloader/uboot-repo/bl33/v2019/arch/arm/cpu/armv8/start.S)

When BL31 hands execution to U-Boot, execution begins at the symbol `_start` in `start.S`.

```armasm
.globl _start
_start:
    b   reset

reset:
    /* Step 1: Save boot parameters passed from BL31 in registers x0-x3 */
    bl  save_boot_params

    /* Step 2: Position-Independent Code (PIE) Fixup */
    bl  pie_fixup

    /* Step 3: Set up Exception Vector Base Address Register (VBAR_EL2) */
    adr x0, vectors
    msr vbar_el2, x0

    /* Step 4: Apply ARM Cortex-A55 Hardware Errata Fixes */
    bl  apply_core_errata

    /* Step 5: Master/Slave CPU Check (MPIDR_EL1 Register) */
    mrs x0, mpidr_el1
    and x0, x0, #0xFF        /* Extract CPU ID (Bits [7:0]) */
    cbz x0, master_cpu       /* If CPU ID == 0 -> Jump to master_cpu */

slave_cpu:
    wfe                      /* If CPU ID != 0 -> Put Cores 1, 2, 3 to sleep! */
    b   slave_cpu

master_cpu:
    /* Core 0 continues to C runtime entry point _main in crt0_64.S */
    bl  _main
```

#### **Why Secondary CPU Cores Sleep (`slave_cpu` / `wfe`):**
1. **Race Condition Prevention**: Memory locks (`spin_lock`, `mutex`) and Cache Coherency (L1/L2 snooping) are not yet enabled. If 4 cores executed U-Boot concurrently, they would overwrite shared RAM registers and crash.
2. **eMMC Bus Bottleneck**: U-Boot is I/O-bound (reading eMMC at ~100 MB/s max). Additional CPU cores cannot speed up storage bus throughput.
3. **Power & Thermal Stability**: Prevents current draw spikes that cause voltage droops at power-on.

---

### **2.2 Assembly to C Bridge (`crt0_64.S`)**

* **File Path**: [`bootloader/uboot-repo/bl33/v2019/arch/arm/lib/crt0_64.S`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/bootloader/uboot-repo/bl33/v2019/arch/arm/lib/crt0_64.S)

1. Sets up the initial C stack pointer (`sp`).
2. Reserves global data structure (`gd_t`).
3. Calls `board_init_f()`.
4. Sets up final stack and calls `board_init_r()`.

---

### **2.3 Early C Initialization (`common/board_f.c`)**

* **File Path**: `bootloader/uboot-repo/bl33/v2019/common/board_f.c`
* **Function**: `board_init_f()`

Executes a sequential array of initialization function pointers (`init_sequence_f`):

1. **`init_baud_rate`**: Reads serial baud rate from environment (default: `115200` / `921600`).
2. **`serial_init`**: Powers on Amlogic UART0 hardware controller (`In: serial@a000`, `Out: serial@a000`).
3. **`dram_init`**: Queries memory controller and registers total DRAM size (`DRAM: 3.5 GiB`).
4. **`reserve_uboot` & `reserve_malloc`**: Calculates top-of-RAM relocation address (`Relocating to dfe0e000`).

---

### **2.4 Late C Initialization (`common/board_r.c`)**

* **File Path**: `bootloader/uboot-repo/bl33/v2019/common/board_r.c`
* **Function**: `board_init_r()` & `board_late_init()`

Executes `init_sequence_r`:

1. **`initr_emmc`**: Initializes eMMC host controller and reads GPT partition tables (`emmc probe success`).
2. **`initr_env`**: Reads environment parameters from storage (`Loading Environment from STORAGE... OK`).
3. **`board_late_init()`**:
   * Detects active boot slot (`active slot = 0` $\rightarrow$ Slot A).
   * Verifies Keymaster DRM keys in eFuse (`[KM]Msg:rawhead hash check successful`).
   * Initializes VPU/VPP display engines (`vpu: set clk: 666667000Hz`).
   * Performs HDMI HPD detection and TV EDID handshake over I2C.
   * Probes DesignWare Ethernet MAC controller (`designware_eth_probe, ret=0`).
4. Calls `main_loop()`.

---

### **2.5 Main Command Loop & Autoboot (`common/main_loop.c`)**

* **File Path**: `bootloader/uboot-repo/bl33/v2019/common/main_loop.c`
* **Function**: `main_loop()`

1. Checks for user serial input during the 3-second countdown (`Hit any key to stop autoboot: 0`).
2. **If key pressed**: Enters interactive shell (`sc2_ah212#`).
3. **If countdown reaches 0**: Executes the `storeboot` script:
   ```bash
   run storeboot
   ```
   `storeboot` reads `boot_a` and `vendor_boot_a` from eMMC, verifies AVB 2.0 signatures, applies DTBO overlays, and executes `booti 0x01080000 0x13000000 0x01000000`.

---

## 3. Comprehensive Memory Map & Android Partition Architecture

### **3.1 Complete System DRAM Memory Map**

The 3.5 GB physical DRAM space is mapped into precise functional regions:

```text
  Physical RAM Address Range      Size        Assigned Function
 ───────────────────────────────────────────────────────────────────────────────────
  0x00000000 - 0x01000000        16 MB       Reserved for Secure OS (BL31 ATF / OP-TEE)
  0x01000000 - 0x01080000       512 KB       Flattened Device Tree Blob ($fdt_addr_r)
  0x01080000 - 0x05000000        63.5 MB     Linux Kernel Image ($kernel_addr_r)
  0x05000000 - 0x07300000        35 MB       Secure Monitor Buffer (linux,secmon)
  0x07300000 - 0x07400000         1 MB       Crash Log Buffer (ramoops@0x08400000)
  0x13000000 - 0x30000000       464 MB       Vendor Ramdisk & Drivers ($ramdisk_addr_r)
  0x30000000 - 0x30800000         8 MB       Audio DSP Firmware Buffer (linux,dsp_fw)
  0x58400000 - 0x6F000000       364 MB       Video Decoder CMA Buffer (codec_mm_cma)
  0x6F000000 - 0x74800000        90 MB       Mali GPU Graphics CMA Buffer (heap-gfx)
  0x74800000 - 0x79800000        80 MB       PCIe DMA Operations Buffer (pcie_dma_ops)
  0x79800000 - 0x7D000000        56 MB       Display Framebuffer Pool (heap-fb)
  0x7D000000 - 0x7D800000         8 MB       System CMA Pool (linux,cma)
  0x7D800000 - 0x7E000000         8 MB       Video Input 1 CMA Buffer (vdin1_cma)
  0x7E200000 - 0x7F200000        16 MB       Secure Video Decoder (secure_vdec_reserved)
  0x7F800000 - 0x80000000         8 MB       Boot Logo OSD Framebuffer (meson-fb)
  0xDFE00000 - 0xE0000000        32 MB       Relocated U-Boot High Execution Memory
```

---

### **3.2 Android Dynamic Partitions Deep Dive (`super` & `dm-linear`)**

#### **Static vs Dynamic Partition Architecture**

* **Legacy Static Partitions (pre-Android 10)**: Every partition (`system`, `vendor`, `product`) had a fixed size hardcoded directly into the eMMC GPT header. If a software update increased `/system` size by 100 MB, the update failed even if `/vendor` had 500 MB of unused empty space!
* **Android Dynamic Partitions (Android 10+)**: Replaces static GPT partitions with **ONE single physical container partition named `super`**.

```text
 Physical eMMC GPT Table:
 └── super (3.2 GB Physical Container)
       │
       ▼ Contains LPMetadata (Logical Partition Metadata)
       │
 Runtime Linux Kernel 'dm-linear' Virtual Device Mapping:
 ├── /dev/block/dm-0  -->  /system   (Dynamically allocated from super)
 ├── /dev/block/dm-1  -->  /vendor   (Dynamically allocated from super)
 ├── /dev/block/dm-2  -->  /product  (Dynamically allocated from super)
 └── /dev/block/dm-3  -->  /odm      (Dynamically allocated from super)
```

#### **How Android `/init` Builds Dynamic Partitions:**
1. The Linux Kernel eMMC driver reads the GPT table and creates the physical device node `/dev/block/mmcblk0p15` (`super`).
2. Android **`/init`** reads the **LPMetadata (Logical Partition Metadata)** stored in the first block of `super`.
3. `/init` calls the kernel **`dm-linear` (Device Mapper)** driver.
4. `dm-linear` creates virtual block devices in RAM (`/dev/block/dm-0` to `dm-3`) mapping logical sector offsets.

---

### **3.3 Complete Partition Breakdown & Detailed Usage**

Defined in workspace file: [`device/amlogic/k14_2gb/part_table_5_15.txt`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/device/amlogic/k14_2gb/part_table_5_15.txt)

| Partition Name | Type | Size | Complete Technical Usage & Description |
| :--- | :--- | :--- | :--- |
| **`reserved`** | Physical GPT | 64 MB | Reserved space for eMMC sector alignment and primary bootloader metadata. |
| **`env`** | Physical GPT | 8 MB | Stores persistent U-Boot environment variables (e.g. `bootargs`, `hdmimode`, `mac`). |
| **`frp`** | Physical GPT | 2 MB | **Factory Reset Protection**: Stores Google Account lock tokens to prevent unauthorized device use after hard reset. |
| **`factory`** | Physical GPT | 8 MB | Stores factory calibration metrics, Wi-Fi/Bluetooth MAC addresses, and HDCP DRM key certificates. |
| **`vendor_boot_a / _b`** | Physical GPT | 64 MB | **Hardware Vendor Boot**: Contains hardware Device Tree (`.dtb`) + Vendor Ramdisk (`.ko` kernel drivers like `amhdmitx.ko`, `dovi.ko`). |
| **`bootloader_a / _b`** | Physical GPT | 8 MB | Backup copies of U-Boot binary images. |
| **`tee`** | Physical GPT | 32 MB | **TrustZone / OP-TEE Storage**: Persistent encrypted storage for Widevine L1 DRM keys, Keymaster, and RPMB data. |
| **`logo`** | Physical GPT | 8 MB | Stores splash screen BMP graphics displayed on TV screen by U-Boot during early bootup. |
| **`misc`** | Physical GPT | 2 MB | **Boot Control Block (BCB)**: Stores active slot flags (`active_slot = 0` for Slot A), recovery commands, and OTA reboot flags. |
| **`dtbo_a / _b`** | Physical GPT | 2 MB | **Device Tree Overlays**: Contains board-specific hardware patches dynamically merged into main DTB at boot. |
| **`odm_ext_a / _b`** | Physical GPT | 16 MB | Original Design Manufacturer extensions and board customization configs. |
| **`boot_a / _b`** | Physical GPT | 64 MB | **Generic Kernel Image (GKI)**: Contains generic uncompressed/compressed Linux Kernel (`Image.lz4`) + generic Android ramdisk. |
| **`metadata`** | Physical GPT | 64 MB | Stores File-Based Encryption (FBE) master keys used to mount `/data`. |
| **`vbmeta_a / _b`** | Physical GPT | 2 MB | **Android Verified Boot (AVB 2.0)**: Contains RSA-2048/4096 public keys, descriptors, and cryptographic SHA-256 root hashes. |
| **`super`** | Physical GPT | 3.2 GB | **Dynamic Partition Container**: Physical GPT container holding `system`, `vendor`, `product`, and `odm` dynamic partitions. |
| **`system`** *(inside `super`)* | Dynamic Logical | ~1.8 GB | **Android OS Framework**: Java APIs, ART Runtime, System Services (`system_server`), and system apps (`Settings`, `Launcher`). |
| **`vendor`** *(inside `super`)* | Dynamic Logical | ~500 MB | **Hardware Abstraction Layer (HAL)**: C++ shared libraries (`.so`), vendor daemons (`systemcontrol`), and hardware HAL binaries. |
| **`product`** *(inside `super`)* | Dynamic Logical | ~400 MB | **Product Customization**: OEM branding, app packages, custom sound effects, and vendor media extractors. |
| **`odm`** *(inside `super`)* | Dynamic Logical | ~100 MB | **Board Hardware Customization**: Board-level custom configuration files, display profiles, and sensor configs. |
| **`userdata`** | Physical GPT | Remaining (~10.5 GB) | **User Data (`/data`)**: Mounted at `/data`. Stores installed user apps, app databases, app caches, user settings, encrypted via FBE. |

---

### **3.4 eMMC Hardware Block Registration & UnifyKey Authentication**

During kernel bootup, the storage subsystem completes key authentication and registers physical hardware device nodes:

```text
[1.476592] [mmc]: emmc key: emmc_key_init:522 ok.
[1.518443] [mmc]: emmc_key_read:566, read ok
[1.519648] [efuse-unifykey]: rawhead hash check successful
[1.521999] [efuse-unifykey]: attach success!
[1.588196] mmcblk0: p1 p2 p3 p4 p5 p6 p7 p8 p9 p10 p11 p12 p13 p14 p15 p16 p17 p18 p19 p20 p21 p22 p23 p24 p25 p26 p27 p28 p29
[1.591029] mmcblk0boot0: mmc0:0001 A3A551 4.00 MiB
[1.592839] mmcblk0boot1: mmc0:0001 A3A551 4.00 MiB
[1.595744] mmcblk0rpmb: mmc0:0001 A3A551 16.0 MiB, chardev (236:0)
```

1. **`efuse-unifykey`**: Authenticates hardware identity certificates and Widevine DRM keymaster hashes in eFuse (`rawhead hash check successful`).
2. **GPT Partition Registration**: Reads the eMMC table and creates **29 physical partition nodes** (`p1` through `p29`).
3. **Special Hardware Blocks**:
   * `mmcblk0boot0` & `mmcblk0boot1`: 4 MB eMMC hardware boot partitions.
   * `mmcblk0rpmb`: 16 MB **Replay Protected Memory Block** secure storage partition for DRM keys.

---

### **3.5 eMMC Driver Comparison (U-Boot vs Linux Kernel)**

| Feature | U-Boot eMMC Driver | Linux Kernel eMMC Driver (`sdhci-amlogic`) |
| :--- | :--- | :--- |
| **Bus Mode** | High-Speed SDR (~100 MB/s) | **HS400 Dual-Data-Rate (400 MB/s @ 200 MHz)** |
| **Operation** | Polling / Basic DMA | Interrupt-Driven Multi-queue Async DMA |
| **Driver Scope** | Loads `boot_a` & `vendor_boot_a` into RAM | Creates `/dev/block/mmcblk0` & mounts `system`, `vendor`, `data` |
| **Lifecycle** | Destroyed upon jumping to kernel | Runs permanently during entire OS runtime |

---

### **3.6 Over-The-Air (OTA) Background Update Sequence**

```text
 [ Running System: Slot A ]
            │
            ├─► 1. Android 'update_engine' downloads OTA payload in background
            ├─► 2. Writes updated files to Slot B inside 'super' via Kernel eMMC driver:
            │      - system_b
            │      - vendor_b
            │      - boot_b
            │      - vendor_boot_b
            ├─► 3. Writes 'active_slot = 1' into 'misc' partition
            │
            ▼
 [ User Clicks Reboot ]
            │
            ├─► 4. U-Boot reads 'misc', detects active_slot = 1
            ├─► 5. Boots Slot B
            │
            └─► [ Boot Success ] ──► System running on Slot B!
                [ Boot Failure ] ──► U-Boot rolls back to Slot A automatically.
```

---

## 4. Display, HDMI, Dolby Vision, V4L2 & ALSA Deep Dive

### **4.1 HDMI HPD & EDID Handshake**

1. **HPD Detection**: U-Boot and Kernel probe the HDMI Hot-Plug Detect line (`hpd_state=1`).
2. **EDID Parsing**: Reads 256 bytes over I2C DDC from the connected TV/Monitor:
   * **Header**: `00 ff ff ff ff ff ff 00`
   * **Manufacturer & Product ID**: `4c 2d 1a 0d` (Samsung Display)
   * **Product Name**: `S22F350` (Samsung 22" 1080p Monitor)
   * **CEA-861 Extension Block**: Validates supported VIC modes (`VIC 16` = 1080p @ 60Hz YUV444 8-bit).

---

### **4.2 Dolby Vision 4-Layer Architecture**

```text
 ┌────────────────────────────────────────────────────────┐
 │ Layer 4: Android App & Media Framework                 │
 │ • Apps (Netflix / YouTube / Disney+)                   │
 │ • MediaCodec & DisplayManagerService                   │
 └───────────────────────────┬────────────────────────────┘
                             │ Exposes HDR_TYPE_DOLBY_VISION
                             ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 3: SystemControl C++ HAL Daemon                  │
 │ • CDolbyVision.cpp & DisplayMode.cpp                   │
 │ • Manages 'Adaptive' vs 'Always' HDR policy            │
 └───────────────────────────┬────────────────────────────┘
                             │ Writes to sysfs / ioctl
                             ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 2: Linux Kernel Drivers                          │
 │ • amdovi.c & amhdmitx.ko                               │
 │ • Controls IPT Tunneling vs SDR Bypass (/sys/class/...)│
 └───────────────────────────┬────────────────────────────┘
                             │ Configures hardware registers
                             ▼
 ┌────────────────────────────────────────────────────────┐
 │ Layer 1: U-Boot Display Engine                         │
 │ • Reads TV EDID Dolby Vision VSDB Block (0x00D046)     │
 │ • Sets 'dv disabled' if monitor lacks DV support       │
 └───────────────────────────┴────────────────────────────┘
```

* **`CDolbyVision.cpp`**: [`vendor/amlogic/common/frameworks/services/systemcontrol/PQ/CDolbyVision.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/vendor/amlogic/common/frameworks/services/systemcontrol/PQ/CDolbyVision.cpp)
* **`DisplayMode.cpp`**: [`vendor/amlogic/common/frameworks/services/systemcontrol/DisplayMode.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/vendor/amlogic/common/frameworks/services/systemcontrol/DisplayMode.cpp)

---

### **4.3 V4L2 vs ALSA Frameworks**

* **V4L2 (Video for Linux 2)**: The Linux Kernel subsystem managing video decoders, camera sensors, and capture inputs. Handles buffer queuing (`VIDIOC_QBUF`/`VIDIOC_DQBUF`) and color formats (YUV420, NV12, H.265).
* **ALSA (Advanced Linux Sound Architecture)**: The Linux Kernel subsystem managing audio hardware. Handles PCM digital audio streams (48kHz 16-bit stereo/surround), volume mixing, and audio clock synchronization.

---

## 5. Linux Kernel Initialization & Driver Probing (`main.c` & `setup.c`)

### **5.1 Entry Point: `start_kernel()`**

* **File Location**: [`common/common14-5.15/common/init/main.c`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/common/common14-5.15/common/init/main.c#L958) (Line 958)

```c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    char *command_line;
    char *after_dashes;

    set_task_stack_end_magic(&init_task);
    smp_setup_processor_id();
    debug_objects_early_init();
    init_vmlinux_build_id();

    cgroup_init_early();

    local_irq_disable();
    early_boot_irqs_disabled = true;

    boot_cpu_init();
    page_address_init();
    pr_notice("%s", linux_banner);     /* Prints: Linux version 5.15.153... */
    early_security_init();
    setup_arch(&command_line);        /* Parses Device Tree & Memory zones */
    setup_boot_config();
    setup_command_line(command_line);
    setup_nr_cpu_ids();
    setup_per_cpu_areas();
    smp_prepare_boot_cpu();
    boot_cpu_hotplug_init();

    build_all_zonelists(NULL);
    page_alloc_init();

    pr_notice("Kernel command line: %s\n", saved_command_line);
    jump_label_init();
    parse_early_param();
    
    setup_log_buf(0);
    vfs_caches_init_early();
    sort_main_extable();
    trap_init();
    mm_init();                        /* Initializes Memory Allocators */
    ftrace_init();
    sched_init();                     /* Initializes Task Scheduler */

    early_boot_irqs_disabled = false;
    local_irq_enable();

    console_init();                   /* Enables printk serial output */
    
    lockdep_init();
    sched_clock_init();
    calibrate_delay();

    arch_cpu_finalize_init();
    pid_idr_init();
    cred_init();
    fork_init();
    proc_caches_init();
    vfs_caches_init();
    cgroup_init();

    /* Spawns PID 1 (kernel_init -> /init) */
    arch_call_rest_init();
}
```

---

### **5.2 Device Tree Parsing (`setup.c`)**

* **File Location**: [`common/common14-5.15/common/arch/arm64/kernel/setup.c`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/common/common14-5.15/common/arch/arm64/kernel/setup.c#L298)

1. **Line 314 (`setup_machine_fdt(__fdt_pointer)`)**:
   Maps the physical address of the DTB passed in CPU register `x0` from U-Boot into kernel memory space.
2. **Line 354 (`unflatten_device_tree()`)**:
   Unpacks the raw binary DTB blob into live C data structures (`struct device_node`) representing every device component on the board.

---

### **5.3 Video Driver RDMA Register Tracing (`aml_vecm` & `rdma_write_reg`)**

During driver probing (`do_one_initcall`), the kernel registers the Amlogic Video Enhancement driver (`aml_vecm`):

#### **Driver Call Stack Trace:**
```text
aml_vecm_probe -> init_pq_setting -> cm_init_config -> am_set_regmap -> VSYNC_WR_MPEG_REG -> rdma_write_reg
```

1. **`aml_vecm_probe`**: Driver binds to VPU hardware.
2. **`init_pq_setting`**: Initializes Contrast, Saturation, Brightness, Sharpness, and Gamma matrices.
3. **`rdma_write_reg`**: Writes color registers directly into VPU memory using **RDMA (Remote Direct Memory Access)**.
4. **Trace Output**: Developer debug calls (`dump_stack()`) print `rdma: rdma_write(1)(swapper/0)` to serial logs to verify every RDMA color register write.

---

### **5.4 Multi-Core SMP CPU Wakeup**

Once `mm_init()`, spinlocks, and cache coherency are initialized, the kernel executes `smp_init()`, issuing PSCI firmware commands to wake up secondary CPUs:

```text
[    0.013314][0 T1     ..] smp: Bringing up secondary CPUs ...
[    0.014020][0 T0     ..] Detected VIPT I-cache on CPU1
[    0.014077][0 T0     ..] CPU1: Booted secondary processor
[    0.014796][0 T0     ..] Detected VIPT I-cache on CPU2
[    0.014844][0 T0     ..] CPU2: Booted secondary processor
[    0.015538][0 T0     ..] Detected VIPT I-cache on CPU3
[    0.015578][0 T0     ..] CPU3: Booted secondary processor
[    0.015653][0 T1     ..] smp: Brought up 1 node, 4 CPUs
```

---

### **5.5 Reclaiming Early Boot Memory & Handover to Android (`/init`)**

```text
Freeing unused kernel memory: 1600K
Run /init as init process
```

1. **`Freeing unused kernel memory: 1600K`**: Memory occupied by early setup routines marked `__init` is freed and returned to the OS pool (~1.6 MB).
2. **`rest_init()`** creates process **PID 1 (`kernel_init`)**.
3. **`kernel_init()`** mounts the root filesystem (`initramfs`) and calls `run_init_process("/init")`.
4. **CPU Execution Transition**: Switches from **Kernel Space (EL1)** to **User Space (EL0)**.

---

## 6. Android First-Stage `/init` Lifecycle, Dynamic Mounting & SELinux

Once execution enters User Space (EL0), Android **First-Stage `/init`** executes the following critical sequence inside the Android source tree:

### **6.1 Source File Map for Init & SELinux Subsystems**

| Task / Feature | Source File Path in Workspace | Primary Function |
| :--- | :--- | :--- |
| **First-Stage Main Entry** | [`system/core/init/first_stage_main.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/system/core/init/first_stage_main.cpp) | `main(int argc, char** argv)` |
| **Kernel Module Loader** | [`system/core/init/first_stage_init.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/system/core/init/first_stage_init.cpp) | `FirstStageMain(int argc, char** argv)` |
| **Dynamic Partitions (`dm-linear`)** | [`system/core/fs_mgr/fs_mgr_dm_linear.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/system/core/fs_mgr/fs_mgr_dm_linear.cpp#L139) | `CreateLogicalPartitions()` |
| **SELinux Setup & Compiler** | [`system/core/init/selinux.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/system/core/init/selinux.cpp#L972) | `SetupSelinux(char** argv)` |
| **Kernel `dm-linear` Driver** | [`common/common14-5.15/common/drivers/md/dm-linear.c`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/common/common14-5.15/common/drivers/md/dm-linear.c) | `linear_map()` / `linear_ctr()` |
| **Kernel `dm-verity` Driver** | [`common/common14-5.15/common/drivers/md/dm-verity.c`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/common/common14-5.15/common/drivers/md/dm-verity.c) | `verity_map()` / `verity_ctr()` |

---

### **6.2 First-Stage GKI Module Loading (`/lib/modules/`)**

Android `/init` loads modular kernel drivers required before mounting filesystems:

```text
[1.802618] init: Loading module /lib/modules/amlogic-hwspinlock.ko
[1.806046] init: Loading module /lib/modules/aml_smmu.ko
[1.811639] init: Loading module /lib/modules/amlogic-input.ko
[1.827797] init: Loading module /lib/modules/amlogic-usb.ko
```

* **`amlogic-hwspinlock.ko`**: Hardware mutex locks for multi-core synchronization.
* **`aml_smmu.ko`**: System MMU driver allocating the 76 MB PCIe DMA pool (`0x74800000`).
* **`amlogic-input.ko`**: Initializes GPIO keys and IR Remote Control input (`ir_keypad0_0`).
* **`amlogic-usb.ko`**: Probes the xHCI USB 2.0/3.0 Host Controller (`xhci-hcd-meson`).

---

### **6.3 Dynamic Partition Mounting (EROFS + `dm-verity`)**

Android `/init` mounts read-only system partitions using high-performance **EROFS (Enhanced Read-Only File System)** compressed images authenticated via ARMv8 hardware-accelerated **`dm-verity`**:

```text
[1.952013] device-mapper: verity: sha256 using implementation "sha256-ce"
[1.956890] erofs: (device dm-8): mounted with root inode @ nid 59.
[1.968815] erofs: (device dm-9): mounted with root inode @ nid 39.
[1.975025] erofs: (device dm-10): mounted with root inode @ nid 38.
[1.981645] erofs: (device dm-11): mounted with root inode @ nid 44.
```

1. **`sha256-ce` Acceleration**: Uses ARMv8 Crypto Extensions (`ce`) to execute cryptographic SHA-256 hash checks at wire speed.
2. **Logical EROFS Partition Mounts**:
   * **`dm-8`** $\rightarrow$ **`/system`**
   * **`dm-9`** $\rightarrow$ **`/system_ext`**
   * **`dm-10`** $\rightarrow$ **`/vendor`**
   * **`dm-11`** $\rightarrow$ **`/product`**
   * **`dm-12`** $\rightarrow$ **`/odm`**
3. **EXT4 Data Mount**:
   * Mounts `mmcblk0p14` as **`/data`** (`userdata`).

---

### **6.4 SELinux Setup, Policy Compilation & Domain Transition (`selinux.cpp`)**

In [`system/core/init/selinux.cpp`](file:///mnt/ebs5/p-parasp/new_neha_sdmc/system/core/init/selinux.cpp#L972), `/init` compiles and loads SELinux policy:

```cpp
// Source: system/core/init/selinux.cpp (Line 972)
int SetupSelinux(char** argv) {
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);

    MountMissingSystemPartitions();
    SelinuxSetupKernelLogging();

    // Step 1: Prepares APEX SEPolicy packages
    PrepareApexSepolicy();

    // Step 2: Reads precompiled policy or compiles split policy via /system/bin/secilc
    std::string policy;
    ReadPolicy(&policy);

    // Step 3: Loads SELinux policy into Linux Kernel (/sys/fs/selinux)
    LoadSelinuxPolicy(policy);

    // Step 4: Sets Enforcing (1) vs Permissive (0) mode based on kernel cmdline
    SelinuxSetEnforcement();

    // Step 5: Restores SELinux security contexts on /system/bin/init
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    // Step 6: Transitions from Kernel Domain (kernel) to User Init Domain (u:r:init:s0)
    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));

    return 1;
}
```

---

## 7. APEX Container Initialization, Peripheral Driver Probing & Android ServiceManager

Between timestamps `2.66s` and `5.38s`, the system initializes APEX containers, hardware peripherals, and Android system daemons.

### **7.1 Raw Serial Log Trace (Timestamps `2.669s` to `5.387s`)**

```text
[    2.669375][2 T158   ..] cutils-trace: Error opening trace file: No such file or directory (2)
[    2.670617][2 T158   ..] apexd-bootstrap: Scanning /system/apex for pre-installed ApexFiles
[    2.672217][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.amlogic.mediaextractor.apex
[    2.676037][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.android.adbd.capex
[    2.678697][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.android.adservices.capex
[    2.681216][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.android.apex.cts.shim.apex
[    2.683827][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.android.appsearch.capex
[    2.686488][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.android.art.capex
[    2.689214][2 T158   ..] apexd-bootstrap: Found pre-installed APEX /system/apex/com.android.btservices.apex
[    3.623052][0 T196   ..] loop0: detected capacity change from 0 to 34880
[    3.626724][1 T199   ..] loop1: detected capacity change from 0 to 1616
[    3.630806][1 T198   ..] loop2: detected capacity change from 0 to 7456
[    3.631415][0 T197   ..] loop3: detected capacity change from 0 to 68064
[    3.650941][3 T199   ..] EXT4-fs (loop1): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    3.656418][0 T199   ..] apexd (199) used greatest stack depth: 11888 bytes left
[    3.657318][2 T196   ..] EXT4-fs (loop0): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    3.662152][0 T197   ..] EXT4-fs (loop3): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    3.665475][0 T198   ..] EXT4-fs (loop2): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    3.682713][0 T158   ..] printk: apexd: 49 output lines suppressed due to ratelimiting
[    3.683200][0 T158   ..] apexd (158) used greatest stack depth: 11376 bytes left
[    3.788437][1 T156   ..] amlogic_fbc_lib: module license 'Copyright (C) 2015 Amlogic, Inc. All rights reserved.' taints kernel.
[    3.789193][1 T156   ..] Disabling lock debugging due to kernel taint
[    3.790433][1 T156   ..] register_amlogic_afbc_dec_fun
[    3.884140][2 T156   ..] cuva alg: 2024-04-28 V-0.1, modify division by zero,cuva init set 0 init ok
[    3.886170][2 T156   ..] hdr10_tmo_alg: hdr10_tmo_alg_driver_init insmod ok. v1.0_2023-2-14
[    3.920064][0 T212   ..] meson-tsensor fe020000.p_tsensor: Does not support reset func..
[    3.920691][0 T212   ..] [thermal]: r1p1_tsensor_read  valid cnt is 0, tvalue:0
[    3.934564][2 T212   ..] meson-tsensor fe022000.d_tsensor: Does not support reset func..
[    3.935171][2 T212   ..] [thermal]: r1p1_tsensor_read  valid cnt is 0, tvalue:0
[    3.952941][2 T212   ..] [thermal]: thermal: register cpucore failed
[    3.953066][2 T212   ..] [thermal]: meson_cdev one or more cooldev register fail
[    3.969012][3 T212   ..] aml_crypto_dev fe440400.aml_dma:crypto: Aml crypto device (irq)
[    3.972627][0 T212   ..] [wireless]: [wifi_dev_probe] wifi_pwm_tee = 0
[    3.972764][0 T212   ..] [wireless]: [wifi_dev_probe] use double channel
[    3.973851][0 T212   ..] [wireless]: [wifi_dev_probe] buf_level is :2
[    3.974435][0 T212   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_init_wlan_mem : 101.10.361.36 (wlan=r892223-20231107-1)
[    3.975716][0 T212   ..] [wireless]: [dhd] STATIC-MSG) dhd_init_wlan_mem : 101.10.361.36 (wlan=r892223-20231107-1)
[    3.977753][0 T212   ..] [wireless]: [dhd] STATIC-MSG) dhd_init_wlan_mem : prealloc ok for index 0: 8459264(8261K)
[    3.991304][3 T212   ..] [wireless]: amlogic rfkill init
[    3.991838][0 T53    ..] [wireless]: request_irq error ret=-22
[    3.992094][0 T53    ..] [wireless]: dev_pm_set_wake_irq failed: -22
[    3.992886][0 T53    ..] input: input_btrcu as /devices/platform/aml_bt/input/input5
[    4.031847][2 T212   ..] zram: Added device: zram0
[    4.069832][1 T212   ..] usbcore: registered new device driver r8152-cfgselector
[    4.070125][1 T212   ..] usbcore: registered new interface driver r8152
[    4.080884][0 T212   ..] usbcore: registered new interface driver asix
[    4.084479][0 T212   ..] usbcore: registered new interface driver ax88179_178a
[    4.088186][0 T212   ..] usbcore: registered new interface driver cdc_ether
[    4.092399][0 T212   ..] usbcore: registered new interface driver cdc_ncm
[    4.095703][0 T212   ..] usbcore: registered new interface driver r8153_ecm
[    4.098111][0 T212   ..] i2c_dev: i2c /dev entries driver
[    4.222476][1 T212   ..] aml dvb init
[    4.235779][1 T212   ..] dwc_otg: usb0: type: 2 speed: 0,
[    4.235783][1 T212   ..] config: 0, dma: 0, id: 0,
[    4.235813][1 T212   ..] phy: fe03a000, ctrl: 0
[    4.237162][0 T212   ..] dwc_otg: Core Release: 3.30a
[    4.237580][0 T212   ..] dwc_otg: Setting default values for core params
[    4.238439][0 T212   ..] dwc_otg: curmode: 0, host_only: 0
[    4.242256][0 T65    ..] lcd vlock_en=0, vlock_mode=0x8 i=101
[    4.251295][0 T212   ..] dwc_otg: Using Buffer DMA mode
[    4.251319][0 T212   ..] dwc_otg: OTG VER PARAM: 1, OTG VER FLAG: 1
[    4.252167][0 T212   ..] dwc_otg: Working on port type = SLAVE
[    4.252761][0 T212   ..] dwc_otg: Dedicated Tx FIFOs mode
[    4.258075][1 T212   ..] meson-vrtc fe010288.rtc: registered as rtc0
[    4.258212][1 T212   ..] meson-vrtc fe010288.rtc: setting system clock to 1970-01-01T00:00:04 UTC (4)
[    4.284428][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: host bridge /soc/pcie@f5000000 ranges:
[    4.284850][3 T52    ..] amlogic-pcie-v2 f5000000.pcie:       IO 0x00f5600000..0x00f56fffff -> 0x0000000000
[    4.286051][3 T52    ..] amlogic-pcie-v2 f5000000.pcie:      MEM 0x00f5700000..0x00f6ffffff -> 0x00f5700000
[    4.287424][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: GPIO normal: assert reset
[    4.302303][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: iATU unroll: enabled
[    4.302507][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: Detected iATU regions: 4 outbound, 4 inbound
[    4.318718][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: Link up, GEN1,link width is x1
[    4.319032][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: Link up
[    4.319949][3 T52    ..] amlogic-pcie-v2 f5000000.pcie: PCI host bridge to bus 0000:00
[    4.320757][3 T52    ..] pci_bus 0000:00: root bus resource [bus 00-ff]
[    4.321573][3 T52    ..] pci_bus 0000:00: root bus resource [io  0x0000-0xfffff]
[    4.322523][3 T52    ..] pci_bus 0000:00: root bus resource [mem 0xf5700000-0xf6ffffff]
[    4.323520][3 T52    ..] pci 0000:00:00.0: [16c3:abcd] type 01 class 0x060400
[    4.324400][3 T52    ..] pci 0000:00:00.0: reg 0x38: [mem 0x00000000-0x0000ffff pref]
[    4.325404][3 T52    ..] pci 0000:00:00.0: supports D1
[    4.325995][3 T52    ..] pci 0000:00:00.0: PME# supported from D0 D1 D3hot D3cold
[    4.330311][3 T52    ..] pci 0000:01:00.0: [14e4:449d] type 00 class 0x028000
[    4.330587][3 T52    ..] pci 0000:01:00.0: reg 0x10: [mem 0x00000000-0x0000ffff 64bit]
[    4.331541][3 T52    ..] pci 0000:01:00.0: reg 0x18: [mem 0x00000000-0x003fffff 64bit]
[    4.332575][3 T52    ..] pci 0000:01:00.0: Max Payload Size set to 256 (was 128, max 512)
[    4.333809][3 T52    ..] pci 0000:01:00.0: supports D1 D2
[    4.334181][3 T52    ..] pci 0000:01:00.0: PME# supported from D0 D1 D2 D3hot D3cold
[    4.349232][2 T52    ..] pci 0000:00:00.0: BAR 14: assigned [mem 0xf5800000-0xf5dfffff]
[    4.349558][2 T52    ..] pci 0000:00:00.0: BAR 6: assigned [mem 0xf5700000-0xf570ffff pref]
[    4.350650][2 T52    ..] pci 0000:01:00.0: BAR 2: assigned [mem 0xf5800000-0xf5bfffff 64bit]
[    4.351675][2 T52    ..] pci 0000:01:00.0: BAR 0: assigned [mem 0xf5c00000-0xf5c0ffff 64bit]
[    4.352721][2 T52    ..] pci 0000:00:00.0: PCI bridge to [bus 01-ff]
[    4.353486][2 T52    ..] pci 0000:00:00.0:   bridge window [mem 0xf5800000-0xf5dfffff]
[    4.354597][2 T52    ..] OF: /soc/pcie@f5000000: no iommu-map translation for id 0x0 on (null)
[    4.355903][2 T52    ..] pcieport 0000:00:00.0: PME: Signaling with IRQ 74
[    4.362403][2 T52    ..] pcieport 0000:00:00.0: AER: enabled with IRQ 74
[    4.381098][0 T212   ..] amlogic_host fe340000.hifidsp0: this device not support hifi4dsp0
[    4.381508][0 T212   ..] amlogic_host: probe of fe340000.hifidsp0 failed with error -22
[    4.412932][2 T216   ..] aml_init_mm: ffffffdb09c30d58
[    4.438285][3 T53    ..] aw9523_led 3-005b: aw9523_i2c_probe: there is no aw9523 ret=-22
[    4.438629][3 T53    ..] aw9523_led: probe of 3-005b failed with error -22
[    4.441446][3 T212   ..] g12a-mdio_mux fe028000.mdio-multiplexer: wzh failed to get ethrmii clock
[    4.466502][3 T212   ..] meson8b-dwmac fdc00000.ethernet: IRQ eth_wake_irq not found
[    4.466795][3 T212   ..] meson8b-dwmac fdc00000.ethernet: IRQ eth_lpi not found
[    4.467951][3 T212   ..] meson8b-dwmac fdc00000.ethernet: PTP uses main clock
[    4.470676][1 T212   ..] meson8b-dwmac fdc00000.ethernet: User ID: 0x11, Synopsys ID: 0x37
[    4.471038][1 T212   ..] meson8b-dwmac fdc00000.ethernet:    DWMAC1000
[    4.471821][1 T212   ..] meson8b-dwmac fdc00000.ethernet: DMA HW capability register supported
[    4.472894][1 T212   ..] meson8b-dwmac fdc00000.ethernet: RX Checksum Offload Engine supported
[    4.473964][1 T212   ..] meson8b-dwmac fdc00000.ethernet: COE Type 2
[    4.474777][1 T212   ..] meson8b-dwmac fdc00000.ethernet: TX Checksum insertion supported
[    4.475774][1 T212   ..] meson8b-dwmac fdc00000.ethernet: Wake-Up On Lan supported
[    4.476814][1 T212   ..] meson8b-dwmac fdc00000.ethernet: Normal descriptors
[    4.477595][1 T212   ..] meson8b-dwmac fdc00000.ethernet: Ring mode enabled
[    4.478529][1 T212   ..] meson8b-dwmac fdc00000.ethernet: Enable RX Mitigation via HW Watchdog Timer
[    4.479600][1 T212   ..] meson8b-dwmac fdc00000.ethernet: device MAC address b0:b3:69:14:d0:8f
[    4.502266][3 T212   ..] aml_cust_setting
[    4.502311][3 T212   ..] no gpio wol 0
[    4.502553][3 T212   ..] set default cali_val as 0
[    4.503486][3 T212   ..] set rgmii pinmux
[    4.504096][3 T212   ..] input: input_ethrcu as /devices/platform/soc/fdc00000.ethernet/input/input6
[    4.505686][0 T53    ..] g12a-mdio_mux fe028000.mdio-multiplexer: wzh failed to get ethrmii clock
[    4.510917][2 T53    ..] mdio_bus 0.0: ethernet-phy@0 has invalid PHY address
[    4.511131][2 T53    ..] mdio_bus 0.0: scan phy ethernet-phy at address 0
[    4.529452][0 T53    ..] [mdio-g12a]: tx_amp_addr fe010330
[    4.529510][0 T53    ..] [mdio-g12a]: txamp 0x34
[    4.530028][0 T53    ..] [mdio-g12a]: use default st_mode
[    4.600870][1 T212   ..] usbcore: registered new interface driver aml_usbcam
[    4.687247][3 T212   ..] audio-ddr-manager fe330000.audiobus:ddr_manager: 0, irqs frddr 32
[    4.687608][3 T212   ..] audio-ddr-manager fe330000.audiobus:ddr_manager: 1, irqs frddr 33
[    4.688633][3 T212   ..] audio-ddr-manager fe330000.audiobus:ddr_manager: 2, irqs frddr 34
[    4.689661][3 T212   ..] audio-ddr-manager fe330000.audiobus:ddr_manager: 3, irqs frddr 35
[    4.701831][3 T212   ..] snd_pdm fe330000.audiobus:pdm: Can't get pdm pinmux
[    4.702122][3 T212   ..] snd_pdm fe330000.audiobus:pdm: Can't retrieve xtal_clk clock
[    4.703066][3 T212   ..] snd_pdm fe330000.audiobus:pdm: no clk_src_cd clock for 44k case
[    4.704876][3 T212   ..] [snd-soc]: failed to get data_lb_ratec
[    4.734473][3 T212   ..] input: vad_keypad as /devices/platform/soc/fe330000.audiobus/fe330000.audiobus:vad/input/input7
[    4.737175][0 T53    ..] asoc-aml-card auge_sound: aml_card_dai_link_of, error dai-link idx:1, error getting codec dai, ret -517
[    4.737943][0 T53    ..] [snd-soc]: aml_card_probe error ret:-517
[    4.750928][1 T53    ..] asoc-aml-card auge_sound: aml_card_dai_link_of, error dai-link idx:1, error getting codec dai, ret -517
[    4.751696][1 T53    ..] [snd-soc]: aml_card_probe error ret:-517
[    4.758660][0 T212   ..] aml_codec_T9015 fe01a000.t9015: aml_T9015_audio_codec_probe
[    4.759183][0 T212   ..] [snd-codec-t9015]: T9015 acodec tdmout index:1
[    4.761250][1 T53    ..] asoc-aml-card auge_sound: IRQ audio_exception64 not found
[    4.762098][1 T53    ..] [snd-codec-t9015]: call standard reset interface
[    5.031387][3 T53    ..] aml_codec_T9015 fe01a000.t9015: ASoC: source widget Left DAC overwritten
[    5.032641][3 T53    ..] [snd-soc]:
[    5.032641][3 T53    ..] loopback_dai_set_sysclk, 0, 12288000, 0
[    5.033148][3 T53    ..] [snd-soc]: asoc loopback_dai_set_fmt, 0x4010, 0000000077fb6a28
[    5.035527][3 T53    ..] [snd-soc]: no node audio_effect for eq/drc info!
[    5.157034][2 T212   ..] modules_load (212) used greatest stack depth: 11088 bytes left
[    5.296349][1 T224   ..] servicemanager: Starting sm instance on /dev/binder
[    5.304145][1 T224   ..] SELinux: Multiple same specifications for tv_remote.
[    5.305760][1 T224   ..] SELinux: SELinux: Loaded service context from:
[    5.305996][1 T224   ..] SELinux:            /system/etc/selinux/plat_service_contexts
[    5.307674][1 T224   ..] SELinux:            /system_ext/etc/selinux/system_ext_service_contexts
[    5.308909][1 T224   ..] SELinux:            /product/etc/selinux/product_service_contexts
[    5.309239][1 T224   ..] SELinux:            /vendor/etc/selinux/vendor_service_contexts
[    5.364053][2 T156   ..] [efuse-unifykey]: already inited!
[    5.376554][2 T156   ..] mali_kbase: loading out-of-tree module taints kernel.
[    5.387018][3 T222   ..] logd.auditd: start
```

---

## 8. GPU Bifrost DDK, OP-TEE VDEC TA, VINTF Manifest & UserData Mount (Timestamps `5.387s` to `7.059s`)

Between timestamps `5.387s` and `7.059s`, the OS initializes graphics GPU hardware, Dolby Vision engine, OP-TEE secure video firmware, VINTF HAL manifest, and mounts the user partition:

### **8.1 Raw Serial Log Trace (Timestamps `5.387s` to `7.059s`)**

```text
[    5.387074][3 T222   ..] logd.klogd: 5372846377
[    5.389249][1 T53    ..] mali fe400000.bifrost: Kernel DDK version r47p0-01eac0
[    5.389528][1 T53    ..] mali fe400000.bifrost: GPU metrics tracepoint support enabled
[    5.391704][1 T53    ..] clk mali have enabled
[    5.393050][1 T53    ..] mali fe400000.bifrost: Register LUT 00070000 initialized for GPU arch 0x00070009
[    5.397549][1 T53    ..] mali fe400000.bifrost: GPU identified as 0x3 arch 7.0.9 r0p0 status 0
[    5.398147][1 T53    ..] mali fe400000.bifrost: No priority control manager is configured
[    5.398990][1 T53    ..] mali fe400000.bifrost: Large page support was disabled at compile-time!
[    5.400132][1 T53    ..] mali fe400000.bifrost: No memory group manager is configured
[    5.402759][2 T53    ..] mali fe400000.bifrost: Probed as mali0
[    5.408488][0 T222   ..] logd: Loaded bug_map file: /system_ext/etc/selinux/bug_map
[    5.409178][2 T156   ..] [dovi_sc2_5_15_stb26]: *** amlogic_dolby_vision_init dv: sc2 ***
[    5.409514][0 T222   ..] logd: Loaded bug_map file: /vendor/etc/selinux/selinux_denial_metadata
[    5.409791][2 T156   ..] *** register_dv_stb2.6_functions.***
[    5.411800][3 T222   ..] logd: Loaded bug_map file: /system/etc/selinux/bug_map
[    5.411826][2 T156   ..] [dovi_sc2_5_15_stb26]: Creating DV mp success
[    5.413506][2 T156   ..] [dovi_sc2_5_15_stb26]: Creating DV mp success
[    5.414125][2 T156   ..] enable DV HLG when stb v2.6. policy 41
[    5.414895][2 T156   ..] efuse_mode=0 reg_value = 0x18
[    5.415496][2 T156   ..] dv capability 7
[    5.423977][2 T156   ..] init: wait for '/dev/audio_utils' took 0ms
[    5.426327][2 T156   ..] init: wait for '/dev/dolby_fw' took 0ms
[    5.426707][2 T156   ..] init: wait for '/odm/lib/ms12' took 0ms
[    5.544412][3 T224   ..] cutils-trace: Error opening trace file: No such file or directory (2)
[    5.659197][1 T156   ..] [media_clock]: No find node.
[    5.659239][1 T156   ..] [media_clock]: get dos dev failed, id 50(0), try to search
[    5.660101][1 T156   ..] [media_clock]: dos_device_search_data, get major 28 dos dev data success
[    5.661208][1 T156   ..] [media_clock]: initial_dos_device end, chip 50(0)
[    5.667029][1 T156   ..] [firmware]: not find node
[    5.699311][3 T156   ..] vdec vdec: Get pwrc-vdec-2 failed, pm-domain: 0
[    5.699554][3 T156   ..] vdec vdec: Get pwrc-hevcb failed, pm-domain: 0
[    5.700293][3 T156   ..] vdec vdec: Get pwrc-wave failed, pm-domain: 0
[    5.718404][3 T156   ..] Amlogic A/V streaming port init
[    5.762331][3 T1     ..] EXT4-fs (mmcblk0p13): Ignoring removed nomblk_io_submit option
[    5.783392][0 T1     ..] EXT4-fs (mmcblk0p13): recovery complete
[    5.783937][3 T1     ..] EXT4-fs (mmcblk0p13): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit. Quota mode: journalled.
[    5.854044][3 T1     ..] e2fsck: e2fsck 1.46.6 (1-Feb-2023)
[    5.861568][3 T1     ..] e2fsck: Pass 1: Checking inodes, blocks, and sizes
[    5.866850][0 T224   ..] servicemanager: getDeviceHalManifest: Reading VINTF information.
[    5.880641][1 T1     ..] e2fsck: Pass 2: Checking directory structure
[    5.884249][3 T1     ..] e2fsck: Pass 3: Checking directory connectivity
[    5.884508][0 T1     ..] e2fsck: Pass 4: Checking reference counts
[    5.890356][0 T1     ..] e2fsck: Pass 5: Checking group summary information
[    5.897638][3 T1     ..] e2fsck: /dev/block/by-name/param: 18/4096 files (11.1% non-contiguous), 1821/4096 blocks
[    5.903427][0 T1     ..] EXT4-fs (mmcblk0p13): Ignoring removed nomblk_io_submit option
[    5.913671][0 T224   ..] servicemanager: getDeviceHalManifest: Successfully processed VINTF information
[    5.938732][2 T1     ..] EXT4-fs (mmcblk0p13): mounted filesystem with ordered data mode. Opts: nodelalloc,nomblk_io_submit,errors=panic. Quota mode: journalled.
[    5.942004][2 T1     ..] EXT4-fs (mmcblk0p7): Ignoring removed nomblk_io_submit option
[    5.947202][2 T156   ..] EXT4-fs (mmcblk0p7): recovery complete
[    5.947585][3 T1     ..] EXT4-fs (mmcblk0p7): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit. Quota mode: none.
[    5.987656][1 T1     ..] e2fsck: e2fsck 1.46.6 (1-Feb-2023)
[    5.995024][0 T1     ..] e2fsck: Pass 1: Checking inodes, blocks, and sizes
[    6.006380][1 T1     ..] e2fsck: Pass 2: Checking directory structure
[    6.015402][2 T1     ..] EXT4-fs (mmcblk0p7): Ignoring removed nomblk_io_submit option
[    6.042342][3 T1     ..] EXT4-fs (mmcblk0p7): mounted filesystem with ordered data mode. Opts: nodelalloc,nomblk_io_submit,errors=panic. Quota mode: none.
[    6.069358][2 T1     ..] FAT-fs (mmcblk0p4): Volume was not properly unmounted. Some data may be corrupt. Please run fsck.
[    6.126302][2 T53    ..] [TEE] E/TA:   MM-module-name:VDEC TA,Version:1.0.21-g006fb97(build:6823)
[    6.126733][2 T53    ..] [TEE] M/TA: support scs verify chip. 50
[    6.127476][2 T53    ..] [TEE] E/TA:   fw_check_pack_version:301 the package has 17 fws totally.
[    6.128571][2 T53    ..] [TEE] E/TA:   fw_check_pack_version:318 The TA ver is v1.0
[    6.129523][2 T53    ..] [TEE] E/TA:   fw_check_pack_version:319 The fw ver is v0.4
[    6.138107][2 T53    ..] [TEE] E/TA:   fw_data_insert:403 the fw with 368 KB will be loaded.
[    6.138737][2 T53    ..] [TEE] M/TA: TEE_Video_Load_FW success
[    6.222314][2 T273   ..] vdc (273) used greatest stack depth: 10896 bytes left
[    6.418388][3 T284   ..] android.hardware.boot-service: boot_ctrl.roll_flag =
[    6.428831][1 T284   ..] android.hardware.boot-service: IBootControl AIDL service running...
[    6.489842][3 T240   ..] Checkpoint: No magic
[    6.493024][2 T291   ..] vdc: Command: checkpoint restoreCheckpoint /dev/block/by-name/userdata Failed: Status(-8, EX_SERVICE_SPECIFIC): '22: No magic'
[    6.557806][3 T284   ..] android.hardware.boot-service: sys.boot_completed:
[    6.683197][0 T240   ..] EXT4-fs (dm-50): Ignoring removed nomblk_io_submit option
[    6.683474][0 T240   ..] EXT4-fs (dm-50): Using encoding defined by superblock: utf8-12.1.0 with flags 0x0
[    6.721109][0 T240   ..] EXT4-fs (dm-50): 1 orphan inode deleted
[    6.721186][0 T240   ..] EXT4-fs (dm-50): recovery complete
[    6.724014][2 T240   ..] EXT4-fs (dm-50): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit. Quota mode: journalled.
[    6.745453][0 T240   ..] e2fsck: e2fsck 1.46.6 (1-Feb-2023)
[    6.770214][0 T240   ..] e2fsck: Pass 1: Checking inodes, blocks, and sizes
[    7.030079][0 T240   ..] e2fsck: Inode 373946 extent tree (at level 1) could be shorter.  Optimize? yes
[    7.030679][0 T240   ..] e2fsck:
[    7.058973][0 T240   ..] e2fsck: Inode 374162 extent tree (at level 1) could be shorter.  Optimize? yes
[    7.059546][0 T240   ..] e2fsck: 
```

---

## 9. OP-TEE Keymaster HW KDF, ZRAM Swap Creation, Video Composer 4K Capabilities & Second-Stage APEX Mounts (Timestamps `7.150s` to `8.689s`)

Between timestamps `7.150s` and `8.689s`, execution handles hardware key derivation in TrustZone, creates ZRAM compressed swap space, initializes Amlogic Video Composer 4K channels, and completes second-stage APEX container mounting:

### **9.1 Raw Serial Log Trace (Timestamps `7.150s` to `8.689s`)**

```text
[    7.150278][2 T52    ..] [TEE] M/TA: KeymasterTA (info): app/ipc/keymaster_ipc.cpp, Line 1149: Amlogic KEYMINT! Build Time: Dec  5 2024 17:13:03 version: 51a16ec5 TDK: 3c1d5a6
[    7.151558][2 T52    ..] [TEE] M/TA: KeymasterTA (warn): app/ipc/keymaster_ipc.cpp, Line 531: Dispatching GET_VERSION_2, size: 4
[    7.152994][2 T52    ..] [TEE] M/TA: KeymasterTA (info): app/trusty_keymaster_context.cpp, Line 831: master key doesn't exist in storage: 80000100. res = ffff0008
[    7.154844][2 T52    ..] [TEE] M/TA: KeymasterTA (info): app/trusty_keymaster_context.cpp, Line 839: master key doesn't exist in storage: 80000000. res = ffff0008
[    7.156614][2 T52    ..] [TEE] M/TA: KeymasterTA (info): app/trusty_keymaster_context.cpp, Line 847: Master key is derived from KDF
[    7.669270][1 T240   ..] e2fsck: Pass 1E: Optimizing extent trees
[    7.669716][2 T240   ..] e2fsck: Pass 2: Checking directory structure
[    7.865791][3 T240   ..] e2fsck: Pass 3: Checking directory connectivity
[    7.866727][3 T240   ..] e2fsck: Pass 4: Checking reference counts
[    8.227451][0 T240   ..] EXT4-fs (dm-50): Ignoring removed nomblk_io_submit option
[    8.227732][0 T240   ..] EXT4-fs (dm-50): Using encoding defined by superblock: utf8-12.1.0 with flags 0x0
[    8.231458][0 T240   ..] EXT4-fs (dm-50): mounted filesystem with ordered data mode. Opts: nodelalloc,nomblk_io_submit,resgid=1065,errors=panic. Quota mode: journalled.
[    8.260750][0 T1     ..] init: Control message: Could not find 'aidl/SurfaceFlingerAIDL' for ctl.interface_start from pid: 224 (/system/bin/servicemanager)
[    8.265193][0 T1     ..] zram0: detected capacity change from 0 to 1951152
[    8.289577][0 T1     ..] Adding 975572k swap on /dev/block/zram0.  Priority:-2 extents:1 across:975572k SS
[    8.301041][2 T1     ..] init: Control message: Could not find 'aidl/android.hardware.graphics.composer3.IComposer/default' for ctl.interface_start from pid: 224 (/system/bin/servicemanager)
[    8.303306][2 T1     ..] init: Control message: Could not find 'aidl/SurfaceFlingerAIDL' for ctl.interface_start from pid: 224 (/system/bin/servicemanager)
[    8.355413][0 T1     ..] init: Control message: Could not find 'aidl/android.hardware.graphics.composer3.IComposer/default' for ctl.interface_start from pid: 224 (/system/bin/servicemanager)
[    8.371669][2 T324   ..] vdin0 req vs irq 64
[    8.377713][0 T1     ..] fscrypt: AES-256-CTS-CBC using implementation "cts-cbc-aes-ce"
[    8.382485][0 T1     ..] fscrypt: AES-256-XTS using implementation "xts-aes-ce"
[    8.390016][0 T324   ..] vt session 324-0 create
[    8.451036][1 T240   ..] vold: keystore2 Keystore earlyBootEnded returned service specific error: -68
[    8.477982][1 T290   ..] video_composer_open iminor(inode) =0
[    8.478316][1 T290   ..] vc:[0]vd_render_index_get: render_index is 0.
[    8.480348][1 T290   ..] vc:[0]get capability: min 64 64; max 4096 2160
[    8.480532][1 T290   ..] video_composer_open iminor(inode) =1
[    8.481411][1 T290   ..] vc:[1]vd_render_index_get: render_index is 1.
[    8.482385][1 T290   ..] vc:[1]get capability: min 64 64; max 4096 2160
[    8.484278][3 T233   ..] logd: logd reinit
[    8.486912][3 T233   ..] logd: FrameworkListener: read() failed (Connection reset by peer)
[    8.489124][1 T290   ..] vc:[0]get capability: min 64 64; max 4096 2160
[    8.489277][1 T290   ..] vc:[1]get capability: min 64 64; max 4096 2160
[    8.498527][0 T341   ..] apexd: Scanning /system/apex for pre-installed ApexFiles
[    8.499502][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.amlogic.mediaextractor.apex
[    8.500957][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.adbd.capex
[    8.501995][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.adservices.capex
[    8.503317][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.apex.cts.shim.apex
[    8.504626][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.appsearch.capex
[    8.505653][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.art.capex
[    8.507129][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.btservices.apex
[    8.508400][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.configinfrastructure.capex
[    8.509569][0 T341   ..] apexd: Found pre-installed APEX /system/apex/com.android.conscrypt.capex
[    8.559182][2 T1     ..] selinux: SELinux: Skipping restorecon on directory(/data/system/shutdown-checkpoints)
[    8.642343][0 T1     ..] init: Control message: Could not find 'aidl/SurfaceFlingerAIDL' for ctl.interface_start from pid: 224 (/system/bin/servicemanager)
[    8.657613][2 T363   ..] loop4: detected capacity change from 0 to 536
[    8.658202][3 T364   ..] loop5: detected capacity change from 0 to 10392
[    8.659042][0 T365   ..] loop6: detected capacity change from 0 to 25264
[    8.660042][2 T366   ..] loop7: detected capacity change from 0 to 20400
[    8.686721][3 T366   ..] EXT4-fs (loop7): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.689534][3 T366   ..] loop8: detected capacity change from 0 to 38408
```

---

## 10. APEX Container Enumeration, HDMI Un-mute, SELinux AVC Audit Denials & Interactive Shell Launch (Timestamps `8.695s` to `10.257s`)

Between timestamps `8.695s` and `10.257s`, execution completes APEX container mounting, un-mutes HDMI AV output, audits SELinux security denials, spawns the interactive UART terminal console, registers the V4L2 decoder `/dev/video26` node, and loads eBPF network tethering:

### **10.1 Raw Serial Log Trace (Timestamps `8.695s` to `10.257s`)**

```text
[    8.695164][3 T365   ..] EXT4-fs (loop6): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.695581][2 T363   ..] EXT4-fs (loop4): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.696067][1 T364   ..] EXT4-fs (loop5): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.697195][0 T365   ..] loop9: detected capacity change from 0 to 576
[    8.699973][0 T364   ..] loop10: detected capacity change from 0 to 2224
[    8.701104][3 T363   ..] loop11: detected capacity change from 0 to 4320
[    8.703224][1 T366   ..] EXT4-fs (loop8): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.706046][1 T366   ..] loop12: detected capacity change from 0 to 7456
[    8.710611][3 T365   ..] EXT4-fs (loop9): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.712864][3 T365   ..] loop13: detected capacity change from 0 to 68064
[    8.715110][1 T364   ..] EXT4-fs (loop10): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.717131][1 T364   ..] loop14: detected capacity change from 0 to 1048
[    8.722312][1 T366   ..] EXT4-fs (loop12): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.724014][0 T366   ..] loop15: detected capacity change from 0 to 1616
[    8.726644][1 T363   ..] EXT4-fs (loop11): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.729010][1 T363   ..] loop16: detected capacity change from 0 to 34880
[    8.734377][1 T365   ..] EXT4-fs (loop13): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.736213][1 T365   ..] EXT4-fs (loop17): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.743441][1 T364   ..] EXT4-fs (loop14): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.743510][0 T366   ..] EXT4-fs (loop15): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.746210][2 T364   ..] loop18: detected capacity change from 0 to 8896
[    8.746935][1 T366   ..] loop19: detected capacity change from 0 to 39832
[    8.750069][0 T365   ..] EXT4-fs (loop17): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.750322][1 T363   ..] EXT4-fs (loop16): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.752534][0 T365   ..] loop20: detected capacity change from 0 to 5656
[    8.753507][3 T363   ..] loop21: detected capacity change from 0 to 50632
[    8.758359][1 T290   ..] [hdmitx:] avmute_store -1
[    8.761962][0 T364   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.762986][0 T236   ..] type=1400 audit(9.000:4): avc:  denied  { read } for  comm="android.hardwar" name="u:object_r:system_prop:s0" dev="tmpfs" ino=343 scontext=u:r:hal_graphics_composer_default:s0 tcontext=u:object_r:system_prop:s0 tclass=file permissive=0
[    8.765956][0 T236   ..] type=1400 audit(9.000:5): avc:  denied  { read } for  comm="android.hardwar" name="u:object_r:system_prop:s0" dev="tmpfs" ino=343 scontext=u:r:hal_graphics_composer_default:s0 tcontext=u:object_r:system_prop:s0 tclass=file permissive=0
[    8.772265][2 T363   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.773566][1 T366   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.781897][3 T365   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.815050][2 T364   ..] EXT4-fs (dm-49): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.815059][3 T366   ..] EXT4-fs (dm-48): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.817420][3 T366   ..] loop22: detected capacity change from 0 to 6680
[    8.821077][1 T363   ..] EXT4-fs (dm-46): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.821936][1 T364   ..] loop23: detected capacity change from 0 to 8576
[    8.828084][1 T363   ..] loop24: detected capacity change from 0 to 16576
[    8.834698][2 T365   ..] EXT4-fs (dm-47): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.838851][1 T366   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.839413][0 T365   ..] loop25: detected capacity change from 0 to 1512
[    8.858661][3 T363   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.867903][1 T364   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.871676][1 T365   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.880130][0 T366   ..] EXT4-fs (dm-45): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.885356][0 T363   ..] EXT4-fs (dm-40): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.886290][0 T366   ..] loop26: detected capacity change from 0 to 536
[    8.890013][2 T363   ..] loop27: detected capacity change from 0 to 6016
[    8.896194][3 T364   ..] EXT4-fs (dm-43): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.896385][0 T365   ..] EXT4-fs (dm-36): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.899549][0 T364   ..] loop28: detected capacity change from 0 to 17432
[    8.900412][1 T365   ..] loop29: detected capacity change from 0 to 5920
[    8.913876][1 T366   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.917841][1 T364   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.918734][3 T365   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.923559][1 T363   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.934800][0 T366   ..] EXT4-fs (dm-39): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.938135][0 T366   ..] loop30: detected capacity change from 0 to 6496
[    8.947768][2 T364   ..] EXT4-fs (dm-35): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.950779][3 T364   ..] loop31: detected capacity change from 0 to 49208
[    8.951784][0 T363   ..] EXT4-fs (dm-34): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.954475][2 T363   ..] loop32: detected capacity change from 0 to 4648
[    8.960703][0 T365   ..] EXT4-fs (dm-33): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    8.963237][0 T365   ..] loop33: detected capacity change from 0 to 42112
[    8.965948][3 T366   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.977155][3 T364   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.981859][3 T363   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.985729][3 T365   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    8.999099][1 T366   ..] EXT4-fs (dm-31): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.001986][2 T366   ..] loop34: detected capacity change from 0 to 536
[    9.005317][1 T364   ..] EXT4-fs (dm-30): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.008274][2 T364   ..] loop35: detected capacity change from 0 to 34544
[    9.010718][3 T363   ..] EXT4-fs (dm-29): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.013949][1 T363   ..] loop36: detected capacity change from 0 to 30080
[    9.014381][2 T365   ..] EXT4-fs (dm-25): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.034104][2 T366   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    9.034185][3 T363   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    9.034191][1 T364   ..] device-mapper: verity: sha256 using implementation "sha256-ce"
[    9.035649][1 T236   ..] type=1400 audit(9.276:6): avc:  denied  { search } for  comm="BootAnimation" name="data" dev="dm-50" ino=373521 scontext=u:r:bootanim:s0 tcontext=u:object_r:system_data_file:s0:c512,c768 tclass=dir permissive=0
[    9.062622][3 T363   ..] EXT4-fs (dm-20): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.062622][2 T366   ..] EXT4-fs (dm-22): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.062622][1 T364   ..] EXT4-fs (dm-24): mounted filesystem without journal. Opts: (null). Quota mode: none.
[    9.485119][3 T1     ..] selinux: SELinux: Skipping restorecon on directory(/data)
[    9.788008][1 T457   ..] fs-verity: sha256 using implementation "sha256-ce"
[   10.023223][1 T459   ..] apexd: Snapshot DE subcommand detected
[   10.025422][1 T459   ..] apexd-snapshotde: Marking APEXd as ready
[   10.073243][3 T156   ..] [media_sync]:mediasync_policy_manager_init(ffffffc0089af000) initiation success.
console:/ $ [   10.087943][0 T156   ..] aml-vcodec-dec vcodec_dec: v4ldec registered as /dev/video26
[   10.108714][1 T465   ..] init: failed to set task profiles
[   10.200396][2 T1     ..] selinux: SELinux: Skipping restorecon on directory(/data/dalvik-cache/arm)
[   10.201185][2 T1     ..] selinux: SELinux:  Could not stat /data/dalvik-cache/arm64: No such file or directory.
[   10.214446][3 T156   ..] init: Not setting encryption policy on: /data/media
[   10.254487][3 T473   ..] LibBpfLoader: Section bpfloader_min_ver value is 2 [0x2]
[   10.254891][3 T473   ..] LibBpfLoader: Section bpfloader_max_ver value is 25 [0x19]
[   10.256049][3 T473   ..] LibBpfLoader: Section size_of_bpf_map_def value is 120 [0x78]
[   10.256835][3 T473   ..] LibBpfLoader: Section size_of_bpf_prog_def value is 92 [0x5c]
[   10.257686][3 T473   ..] LibBpfLoader: BpfLoader version 0x00026 ignoring ELF object /apex/com.android.tethering/etc/bpf/test.o with max ver 0x00019
```

---

## 11. OTA Update Slot Verification, VINTF System HAL Manifest Resolution, DRM Key Provisioning & Boot Completion (Timestamps `10.259s` to `15.942s`)

Between timestamps `10.259s` and `15.942s`, execution performs post-boot A/B slot verification, registers major Android TV system HALs, audits OP-TEE DRM key persistence, performs HDMI HDCP link authentication, routes ALSA audio streams, and notifies `apexservice` of boot completion:

### **11.1 Raw Serial Log Trace (Timestamps `10.259s` to `15.942s`)**

```text
[   10.259547][3 T473   ..] bpfloader: Loaded object: /apex/com.android.tethering/etc/bpf/test.o
[   10.262466][1 T474   ..] update_verifier: Started with arg 1: nonencrypted
[   10.264017][3 T473   ..] LibBpfLoader: Section bpfloader_min_ver value is 2 [0x2]
[   10.285016][2 T474   ..] update_verifier: Using AIDL version of IBootControl
[   10.286376][1 T284   ..] android.hardware.boot-service: sys.boot_completed:
[   10.286915][2 T474   ..] update_verifier: Booting slot 0: isSlotMarkedSuccessful=1
[   10.287599][2 T474   ..] update_verifier: Leaving update_verifier.
[   10.323632][3 T156   ..] Mass Storage Function, version: 2009/09/11
[   10.323743][3 T156   ..] LUN: removable file: (no medium)
[   10.331363][3 T156   ..] file system registered
[   10.336843][3 T156   ..] using random self ethernet address
[   10.336887][3 T156   ..] using random host ethernet address
[   10.444057][3 T156   ..] 0: amvdec_vc1 module init
[   10.448888][3 T156   ..] 0: amvdec_vc1 module init
[   10.581098][0 T156   ..] have register same node[av1-v4l] on decoder before
[   10.586182][2 T156   ..] Registered frame rate driver success.
[   10.848264][3 T224   ..] servicemanager: Found android.hardware.drm.IDrmFactory/clearkey in device VINTF manifest.
[   10.882920][2 T224   ..] servicemanager: Found android.hardware.power.IPower/default in device VINTF manifest.
[   10.889746][0 T512   ..] [pm]: early_suspend_state=0
[   10.900073][0 T224   ..] servicemanager: Found android.hardware.hdmi.IHdmiFactory/default in device VINTF manifest.
[   10.967721][3 T493   ..] healthd: No battery devices found
[   10.971721][0 T493   ..] healthd: battery none chg=a
[   11.003323][3 T224   ..] servicemanager: Found android.hardware.usb.gadget.IUsbGadget/default in device VINTF manifest.
[   11.011734][2 T224   ..] servicemanager: Found android.hardware.tv.hdmi.connection.IHdmiConnection/default in device VINTF manifest.
[   11.041382][1 T224   ..] servicemanager: Found android.hardware.gatekeeper.IGatekeeper/default in device VINTF manifest.
[   11.082953][0 T224   ..] servicemanager: Found android.hardware.tv.hdmi.cec.IHdmiCec/default in device VINTF manifest.
[   11.091398][1 T224   ..] servicemanager: Found android.hardware.memtrack.IMemtrack/default in device VINTF manifest.
[   11.109040][1 T224   ..] servicemanager: Found android.hardware.usb.IUsb/default in device VINTF manifest.
[   11.111667][0 T224   ..] servicemanager: Found android.hardware.cas.IMediaCasService/default in device VINTF manifest.
[   11.246286][0 T53    ..] [TEE] M/TA: gatekeeper_messages: 414: Amlogic GATEKEEPER! Build Time: Dec  5 2024 17:08:34 version: 8848b72 TDK: Unknow
[   11.247166][2 T156   ..] map_store:rm default
[   11.249091][0 T156   ..] map_store:add default decoder ppmgr deinterlace amvideo
[   11.253611][0 T156   ..] [snd-soc]: drc high cut scale set to 0%
[   11.254999][0 T156   ..] [snd-soc]: drc low boost scale set to 0%
[   11.256541][1 T156   ..] [snd-soc]: drc mode set to RF
[   11.310567][2 T156   ..] usbcore: registered new interface driver wifi_usb_common
[   11.310833][2 T156   ..] aml_usb_common->
[   11.310837][2 T156   ..] aml_wifi_usb_insmod(138) aml common driver insmod
[   11.313385][2 T156   ..] *****************aml sdio common driver is insmoded********************
[   11.314477][2 T156   ..] aml_wifi_sdio_insmod(279) start...
[   11.449259][1 T473   ..] printk: bpfloader: 598 output lines suppressed due to ratelimiting
[   11.656419][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/carrier_changes:  No such file or directory
[   11.657643][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/testing:  No such file or directory
[   11.662973][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/carrier:  No such file or directory
[   11.664087][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/dev_id:  No such file or directory
[   11.665654][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/carrier_down_count:  No such file or directory
[   11.667274][1 T204   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/queues/tx-0/byte_queue_limits/hold_time:  No such file or directory
[   11.669619][1 T204   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/inflight:  No such file or directory
[   11.671292][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/address:  No such file or directory
[   11.674539][1 T203   ..] selinux: SELinux: Could not set context for /sys/devices/virtual/net/ipsec_test/operstate:  No such file or directory
[   12.164061][1 T656   ..] read descriptors
[   12.164233][1 T656   ..] read strings
[   12.270294][1 T53    ..] [TEE] I/TA:
[   12.270335][1 T53    ..] [TEE] MM-module-name:Widevine TA,Version:18.7-r1.3-g291e32a(build:7403)
[   12.271191][1 T53    ..] [TEE] E/TA:   OPTEE_PlatformInit:151 time 12 < 2022
[   12.272066][1 T53    ..] [TEE] I/TA: TA_CloseSessionEntryPoint
[   12.272792][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0x11 WIDEVINE_KEY] already exists
[   12.273919][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0x31 HDCP_TX14_KEY] already exists
[   12.278347][1 T53    ..] [TEE] I/TA: [TA_CreateEntryPoint:104] Playready TA Version
[   12.278650][1 T53    ..] [TEE] MM-module-name:Playready TA,Version:4.4.0.6211-r45.0-g82119ca(build:5449)
[   12.279838][1 T53    ..] [TEE] I/TA: [TA_CreateEntryPoint:121] Playready doesn't backup STORAGE_TKGENERATION
[   12.281062][1 T53    ..] [TEE] I/TA: [TA_CreateEntryPoint:123] Playready doesn't backup STORAGE_TKGENERATION failed
[   12.283989][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0x21 PLAYREADY_PRIVATE_KEY] already exists
[   12.284614][1 T53    ..] [TEE] I/TA: [TA_CreateEntryPoint:104] Playready TA Version
[   12.285515][1 T53    ..] [TEE] MM-module-name:Playready TA,Version:4.4.0.6211-r45.0-g82119ca(build:5449)
[   12.286748][1 T53    ..] [TEE] I/TA: [TA_CreateEntryPoint:121] Playready doesn't backup STORAGE_TKGENERATION
[   12.287929][1 T53    ..] [TEE] I/TA: [TA_CreateEntryPoint:123] Playready doesn't backup STORAGE_TKGENERATION failed
[   12.289807][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0x22 PLAYREADY_PUBLIC_KEY] already exists
[   12.290811][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0x32 HDCP_TX22_KEY] already exists
[   12.770629][0 T381   ..] [hdmitx:] *HDMITX_ERROR* E: ddc_read_1byte hdcp 0x3a 0x50
[   12.818012][3 T381   ..] [hdmitx:] system: hdcp: set mode as 1
[   12.818080][3 T381   ..] [hdmitx:] *HDMITX_ERROR* Record HDMI error: hdmitx_hdcp_auth_read_bksv_error
[   12.818080][3 T381   ..]
[   12.830258][3 T205   .s] [hdmitx:] hdcp14: instat: 0x1
[   12.950243][3 T478   .s] [hdmitx:] hdcp14: instat: 0x81
[   12.962244][3 T0     .s] [hdmitx:] hdcp14: instat: 0x1
[   13.114423][3 T89    ..] [hdmitx:] hdcptx: 1  auth: 1
[   13.133729][2 T485   ..] [hdmitx:] hdmitx20_ext_get_audio_status[1815] val = 1
[   13.141922][2 T485   ..] [debug]: audio_utils_ioctl SET_LIB_SIZE 4298116
[   13.142092][2 T485   ..] [debug]: audio_utils_ioctl WRITE_LIB
[   13.153521][2 T485   ..] [snd-soc]: spk_mute_set: mute flag = 1
[   13.188508][2 T673   ..] audio_ddr_mngr: frddrs[0] registered by device fe330000.audiobus:tdm@1
[   13.189253][2 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 9, use_vadtop 0
[   13.189871][2 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 13, use_vadtop 0
[   13.190934][2 T673   ..] [snd-soc]: hdmitx src switch to spdif 0
[   13.191578][2 T673   ..] [snd-soc]: sharebuffer_spdifout_prepare  get_hdmitx_audio_src(rtd->card):0  spdif_id:0
[   13.192836][2 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:0 size:0 chs:0 i2s_ch_mask:0 aud_src_if:0
[   13.194208][2 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn setting, must be 2, 4, 6 or 8, Rst as def
[   13.195401][2 T673   ..] [hdmitx:] aout notify format CT_PCM
[   13.198172][2 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   13.198273][2 T673   ..] [hdmitx:] set audio param
[   13.198838][2 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:3 size:1 chs:2 i2s_ch_mask:1 aud_src_if:0
[   13.200195][2 T673   ..] [hdmitx:] aout notify format CT_PCM
[   13.203022][2 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   13.203082][2 T673   ..] [hdmitx:] set audio param
[   13.203841][3 T110   ..] [snd-codec-t9015]: aml_T9015_audio_set_bias_level
[   13.204682][3 T110   ..] [snd-codec-t9015]: aml_T9015_audio_set_bias_level
[   13.219984][2 T673   ..] [snd-soc]: release_spdif_same_src(), 1039 src sel 3
[   13.220192][2 T673   ..] [snd-soc]: ss_free() samesrc 3, lvl 1
[   13.221192][2 T673   ..] audio_ddr_mngr: frddrs[1] registered by device fe330000.audiobus:spdif@0
[   13.224241][2 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 9, use_vadtop 0
[   13.224525][2 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 13, use_vadtop 0
[   13.225524][2 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:3 rate:3 size:1 chs:2 i2s_ch_mask:0 aud_src_if:1
[   13.226901][2 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn msk, must larger than 0
[   13.227816][2 T673   ..] [hdmitx:] aout notify format CT_PCM
[   13.230677][2 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   13.230738][2 T673   ..] [hdmitx:] set audio param
[   13.235456][0 T381   ..] [hdmitx:] avmute_store -1
[   13.294270][1 T53    ..] [TEE] E/TA:   [EFUSE-TA] [Error] The Efuse object(DGPK2) already written
[   13.294703][1 T53    ..] [TEE] I/TA: [EFUSE-TA] The same Efuse object(DGPK1_DGPK2_CID) already exists
[   13.295849][1 T53    ..] [TEE] E/TA:   keybox_write:1283 [PROVISION-TA] TEE_InvokeTACommand(cmd: 0x1, subcmd: 0x1) Failed
[   13.297212][1 T53    ..] [TEE] E/TA:   provision_key_store:2015 [PROVISION-TA] keybox_write failed, return 0xFFFF0003
[   13.298582][1 T53    ..] [TEE] M/TA: KeymasterTA (info): app/ipc/keymaster_ipc.cpp, Line 1149: Amlogic KEYMINT! Build Time: Dec  5 2024 17:13:03 version: 51a16ec5 TDK: 3c1d5a6
[   13.300485][1 T53    ..] [TEE] M/TA: KeymasterTA (info): app/ipc/keymaster_ipc.cpp, Line 1161: Goodbye Keymint!
[   13.301752][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0x42 KEYMASTER3_KEY] already exists
[   13.302942][1 T53    ..] [TEE] E/TA:   parse_keybox:1766 [PROVISION-TA] v1 keybox is no longer supported
[   13.304073][1 T53    ..] [TEE] E/TA:   provision_key_store:1996 [PROVISION-TA] parse data failed, res = 0xFFFF000A
[   13.305358][1 T53    ..] [TEE] I/TA: MM-module-name:Netflix TA,Version:2.2.5-gc882a60(build:6886)
[   13.306524][1 T53    ..] [TEE] M/TA: [PROVISION-TA] the same key [0xA2 NETFLIX_MGKID] already exists
[   13.334067][3 T485   ..] [snd-soc]: spk_mute_set: mute flag = 0
[   15.941272][3 T224   ..] servicemanager: Notifying apexservice they do (previously: don't) have clients when service is guaranteed to be in use
[   15.942525][0 T447   ..] AidlLazyServiceRegistrar: Process has 1 (of 1 available) client(s) in use after notification apexservice has clients: 1
```

---

## 12. ApexService Package Enumeration, Android TV VINTF Manifest Resolution & Hardware Exclusions, MediaServer SELinux Property Denial, MediaProxy Inter-Process IPC, 364MB Decoder CMA Pool Status, and Amlogic Hardware H.264 V4L2 Decoder Initialization (Timestamps `15.943s` to `21.174s`)

Between timestamps `15.943s` and `21.174s`, execution enumerates installed APEX packages, resolves framework VINTF HAL manifests, handles STB-specific hardware feature exclusions, logs a MediaServer SELinux property denial, allocates MediaProxy kernel ring buffers, queries Video Decoder CMA memory status, and initializes the hardware H.264 V4L2 decoder:

### **12.1 Raw Serial Log Trace (Timestamps `15.943s` to `21.174s`)**

```text
[   15.943081][3 T446   ..] apexd: getActivePackages received by ApexService
[   15.944882][0 T447   ..] AidlLazyServiceRegistrar: Shutdown prevented by forcePersist override flag.
[   16.007537][0 T224   ..] servicemanager: Could not find android.hardware.power.stats.IPowerStats/default in the VINTF manifest.
[   16.015234][1 T224   ..] servicemanager: Found android.frameworks.stats.IStats/default in framework VINTF manifest.
[   16.016758][1 T224   ..] servicemanager: Found android.hardware.memtrack.IMemtrack/default in device VINTF manifest.
[   16.216040][1 T224   ..] servicemanager: Found android.hardware.power.IPower/default in device VINTF manifest.
[   16.229658][2 T224   ..] servicemanager: Found android.hardware.light.ILights/default in device VINTF manifest.
[   16.712286][3 T446   ..] apexd: getAllPackages received by ApexService
[   19.533713][1 T224   ..] servicemanager: Could not find android.hardware.sensors.ISensors/default in the VINTF manifest.
[   19.539014][0 T224   ..] servicemanager: Could not find android.hardware.health.IHealth/default in the VINTF manifest.
[   19.547511][3 T493   ..] healthd: battery none chg=a
[   19.578128][2 T224   ..] servicemanager: Found vendor.amlogic.hardware.droidaudio.IDroidAudio/default in framework VINTF manifest.
[   19.818766][1 T224   ..] servicemanager: Could not find android.hardware.vibrator.IVibratorManager/default in the VINTF manifest.
[   19.974520][3 T236   ..] type=1400 audit(1789140427.124:9): avc:  denied  { read } for  comm="AmNuPlayerDrive" name="u:object_r:vendor_default_prop:s0" dev="tmpfs" ino=378 scontext=u:r:mediaserver:s0 tcontext=u:object_r:vendor_default_prop:s0 tclass=file permissive=0
[   19.979221][3 T236   ..] type=1400 audit(1789140427.128:10): avc:  denied  { read } for  comm="AmNuPlayerDrive" name="u:object_r:vendor_default_prop:s0" dev="tmpfs" ino=378 scontext=u:r:vendor_default_prop:s0 tcontext=u:object_r:vendor_default_prop:s0 tclass=file permissive=0
[   20.311492][3 T446   ..] AidlLazyServiceRegistrar: Process has 0 (of 1 available) client(s) in use after notification apexservice has clients: 0
[   20.312452][3 T446   ..] AidlLazyServiceRegistrar: Shutdown prevented by forcePersist override flag.
[   21.053622][1 T224   ..] servicemanager: Found android.hardware.thermal.IThermal/default in device VINTF manifest.
[   21.057846][3 T224   ..] servicemanager: Found android.hardware.usb.gadget.IUsbGadget/default in device VINTF manifest.
[   21.063940][1 T815   ..] read descriptors
[   21.063987][1 T815   ..] read strings
[   21.064341][1 T815   ..] read descriptors
[   21.064725][1 T815   ..] read strings
[   21.077601][3 T915   ..] mediaproxy produce init [aml-vcodec-dec]
[   21.077693][3 T915   ..] Mediaproxy is opened name[aml-vcodec-dec]\n
[   21.077697][3 T915   ..] Mediaproxy is opened name[aml-vcodec-dec] end\n
[   21.078548][3 T915   ..] mediaproxy produce start connect
[   21.080004][3 T915   ..] success to alloc kfifo size:[32] i=[2]
[   21.080720][3 T915   ..] find p kfifo size:[32][2]
[   21.081109][2 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.081315][3 T915   ..] mediaproxy_get_fifo get idx[2]
[   21.081319][3 T915   ..] Session producer-2 is connected fifo idx [2]
[   21.083974][3 T915   ..] mediaproxy produce connect [2]
[   21.084619][3 T915   ..] mediaproxy produce msg type :0x40  module_name:aml-vcodec-dec
[   21.088745][1 T915   ..] deinit session
[   21.088788][1 T915   ..] deinit session producer-2
[   21.089211][1 T915   ..] deinit has_unusing_fifo [1]
[   21.093664][0 T87    ..] p_fifo[2] name[aml-vcodec-dec] state[0] to  MP_FIFO_UNUSED
[   21.094689][0 T915   ..] mediaproxy produce init [aml-vcodec-dec]
[   21.094778][0 T915   ..] Mediaproxy is opened name[aml-vcodec-dec]\n
[   21.094781][0 T915   ..] Mediaproxy is opened name[aml-vcodec-dec] end\n
[   21.095590][0 T915   ..] mediaproxy produce start connect
[   21.097080][0 T915   ..] find p kfifo size:[32][2]
[   21.097665][0 T915   ..] mediaproxy_get_fifo get idx[2]
[   21.098700][0 T915   ..] Session producer-2 is connected fifo idx [2]
[   21.099121][0 T915   ..] mediaproxy produce connect [2]
[   21.099768][0 T915   ..] mediaproxy produce msg type :0x40  module_name:aml-vcodec-dec
[   21.101410][1 T915   ..] deinit session
[   21.101443][1 T915   ..] deinit session producer-2
[   21.101839][1 T915   ..] deinit has_unusing_fifo [1]
[   21.102519][1 T87    ..] p_fifo[2] name[aml-vcodec-dec] state[0] to  MP_FIFO_UNUSED
[   21.106646][2 T915   ..] codec mem info:
[   21.106689][2 T915   ..]     total codec mem size:364 MB
[   21.107237][2 T915   ..]     alloced size= 0 MB
[   21.109170][2 T915   ..]     max alloced: 0 MB
[   21.109211][2 T915   ..]     CMA:0,RES:0,TVP:0,SYS:0,COHER:0,VMAPED:0 MB
[   21.110057][2 T915   ..]     RES_EXT:0 MB
[   21.110335][2 T915   ..]     [4]CMA size:364 MB:alloced: 0 MB,free:364 MB
[   21.111633][2 T915   ..] codec mem info:
[   21.111667][2 T915   ..]     total codec mem size:364 MB
[   21.112208][2 T915   ..]     alloced size= 0 MB
[   21.112762][2 T915   ..]     max alloced: 0 MB
[   21.113723][2 T915   ..]     CMA:0,RES:0,TVP:0,SYS:0,COHER:0,VMAPED:0 MB
[   21.114064][2 T915   ..]     RES_EXT:0 MB
[   21.114564][2 T915   ..]     [4]CMA size:364 MB:alloced: 0 MB,free:364 MB
[   21.116920][2 T925   ..] mediaproxy produce init [aml-vcodec-dec]
[   21.117009][2 T925   ..] Mediaproxy is opened name[aml-vcodec-dec]\n
[   21.117012][2 T925   ..] Mediaproxy is opened name[aml-vcodec-dec] end\n
[   21.118468][2 T925   ..] mediaproxy produce start connect
[   21.119863][2 T925   ..] find p kfifo size:[32][2]
[   21.119903][2 T925   ..] mediaproxy_get_fifo get idx[2]
[   21.126535][2 T925   ..] Session producer-2 is connected fifo idx [2]
[   21.126666][2 T925   ..] mediaproxy produce connect [2]
[   21.129008][2 T925   ..] mediaproxy produce msg type :0x40  module_name:aml-vcodec-dec
[   21.131571][2 T925   ..] deinit session
[   21.131608][2 T925   ..] deinit session producer-2
[   21.132209][2 T925   ..] deinit has_unusing_fifo [1]
[   21.132610][1 T87    ..] p_fifo[2] name[aml-vcodec-dec] state[0] to  MP_FIFO_UNUSED
[   21.133055][2 T925   ..] mediaproxy produce init [aml-vcodec-dec]
[   21.135195][2 T925   ..] Mediaproxy is opened name[aml-vcodec-dec]\n
[   21.135209][2 T925   ..] Mediaproxy is opened name[aml-vcodec-dec] end\n
[   21.135314][2 T925   ..] mediaproxy produce start connect
[   21.137048][2 T925   ..] find p kfifo size:[32][2]
[   21.137450][2 T925   ..] mediaproxy_get_fifo get idx[2]
[   21.138080][2 T925   ..] Session producer-2 is connected fifo idx [2]
[   21.138910][2 T925   ..] mediaproxy produce connect [2]
[   21.140299][2 T673   ..] audio_ddr_mngr: frddrs[1] released by device fe330000.audiobus:spdif@0
[   21.141333][2 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 9, use_vadtop 0
[   21.141691][2 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 13, use_vadtop 0
[   21.142710][2 T673   ..] [snd-soc]: hdmitx src switch to spdif 0
[   21.143377][2 T673   ..] [snd-soc]: sharebuffer_spdifout_prepare  get_hdmitx_audio_src(rtd->card):0  spdif_id:0
[   21.144631][2 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:0 size:0 chs:0 i2s_ch_mask:0 aud_src_if:0
[   21.146004][2 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn setting, must be 2, 4, 6 or 8, Rst as def
[   21.147190][2 T673   ..] [hdmitx:] aout notify format CT_PCM
[   21.147969][2 T925   ..] mediaproxy produce msg type :0x40  module_name:aml-vcodec-dec
[   21.149905][3 T925   ..] [amlv4l]:[3]: DW:10, TW:0, Margin:6
[   21.149971][2 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   21.150704][2 T673   ..] [hdmitx:] set audio param
[   21.151272][2 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:3 size:1 chs:2 i2s_ch_mask:1 aud_src_if:0
[   21.152627][2 T673   ..] [hdmitx:] aout notify format CT_PCM
[   21.152804][3 T932   ..] [amlcom]:vdec_init, dev_name:ammvdec_h264_v4l, vdec_type=VDEC_TYPE_FRAME_BLOCK, format: 2, total: 1
[   21.155464][2 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   21.155515][2 T673   ..] [hdmitx:] set audio param
[   21.162879][2 T673   ..] [snd-soc]: release_spdif_same_src(), 1039 src sel 3
[   21.163085][2 T673   ..] [snd-soc]: ss_free() samesrc 3, lvl 1
[   21.164309][3 T673   ..] audio_ddr_mngr: frddrs[1] registered by device fe330000.audiobus:spdif@0
[   21.168154][3 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 9, use_vadtop 0
[   21.168435][3 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 13, use_vadtop 0
[   21.169437][3 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:3 rate:3 size:1 chs:2 i2s_ch_mask:0 aud_src_if:1
[   21.170840][3 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn msk, must larger than 0
[   21.171725][3 T673   ..] [hdmitx:] aout notify format CT_PCM
[   21.173738][3 T915   ..] [amlv4l]:[3]: set vf duration: c80
[   21.174616][3 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   21.174666][3 T673   ..] [hdmitx:] set audio param
```

---

## 13. H.264 1080p Stream Parsing & Picture Buffer Allocation, Gralloc IAllocator Binding, De-Interlacer & VPP Pipeline Routing, U-Boot Boot Splash Screen 8MB Memory Reclaim, Realtek RTL8211FD Gigabit Ethernet RGMII Setup, and Broadcom PCIe Wi-Fi DHD Driver Power-On Sequence (Timestamps `21.183s` to `22.100s`)

Between timestamps `21.183s` and `22.100s`, execution parses the incoming H.264 video stream, binds the Gralloc graphics memory allocator HAL, routes video planes through the De-interlacer (DIM) and VPP hardware pipelines, reclaims 8MB of RAM occupied by U-Boot boot logo, initializes the Realtek Gigabit Ethernet RGMII PHY, and powers on the Broadcom PCIe Wi-Fi chip:

### **13.1 Raw Serial Log Trace (Timestamps `21.183s` to `22.100s`)**

```text
[   21.183375][2 T932   ..] [dv_inst_map]map id 0
[   21.198650][2 T305   ..] [amlv4l]:[3]: Parse from ucode, visible(1920 x 1080), coded(1920 x 1088), scan:P, bitdepth(0), dw(10)
[   21.198671][1 T932   ..] [amlv4l]:[3]: Picture buffer count: dec:5, vpp:0, ge2d:0, margin:6, total:11
[   21.207054][3 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.209484][0 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.220572][2 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.223343][1 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.230769][1 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.232813][2 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   21.429892][1 T224   ..] servicemanager: Could not find android.hardware.tv.input.ITvInput/default in the VINTF manifest.
[   21.711124][1 T386   ..] di_process_open iminor(inode) =0
[   21.711921][2 T386   ..] dim:reg:use[0]ms
[   21.711991][1 T386   ..] dim:ins:create:ch[2],tmode[3]
[   21.712383][1 T386   ..] dim:        out:0x80000000
[   21.716157][1 T386   ..] dp:[0]no hdr10+ data.
[   21.716196][1 T386   ..] dp:[0]no meta_data.
[   21.716559][1 T386   ..] dp:[0]invalid ud param.
[   21.717229][1 T97    ..] dim:value reg:ch[2]:fix_buf:0;ponly <0,0> post_nub=5
[   21.717473][0 T386   ..] vc:[0]vc: set enable index=0, val=1
[   21.718030][1 T97    ..] dim:di_cnt_i_buf1:tvp:0
[   21.718763][0 T386   ..] VID: VD1 off
[   21.718771][0 T386   ..] VID: VD1 set global output as 1
[   21.718798][0 T386   ..] VID: store VD0 path_id changed -1->2
[   21.719304][1 T97    ..] dim:di_cnt_pst_afbct:cfg post nub:5
[   21.721896][0 T386   ..] vc:[0]sideband_type =-1
[   21.726059][1 T97    ..] dim:pre_sec:use[7]ms
[   21.726103][1 T97    ..] dim:pre_sec_alloc:cma size:10,0xfffffffe01f44600,0x7d118000
[   21.728005][1 T97    ..] dim:pst_sec:use[2]ms
[   21.728051][1 T97    ..] dim:pst_sec_alloc:cma:size:30,0xfffffffe01f44880,0x7d122000
[   21.728848][1 T97    ..] dim:di_reg_setting:ch[2]:for first ch reg:
[   21.730494][1 T97    ..] dim:ch[2]:reg:mem cfg[0][0][0]
[   21.730555][1 T97    ..] dim:ch[2]:bypass change:i:0->1:0x60
[   21.736882][1 T101   ..] free_reserved_mem 295 logo start_addr=7f800000, end=80000000
[   21.739679][1 T101   ..] [memory-debug]: Freeing fb-memory memory: 8192K
[   21.739873][1 T101   ..] [drm] am_meson_free_logo_memory, free none_cma memory: addr:0x0x000000007f800000,size:0x800000
[   21.746175][0 T236   ..] type=1400 audit(1789140428.892:11): avc:  denied  { read } for  comm="CDevicePollChec" name="u:object_r:system_prop:s0" dev="tmpfs" ino=343 scontext=u:r:system_control:s0 tcontext=u:object_r:system_prop:s0 tclass=file permissive=0
[   21.996125][1 T553   ..] [wireless]: [wifi_mac_write] wifi rand mac is 1c:a4:10:b9:b4:c5
[   22.022324][1 T620   ..] meson8b-dwmac fdc00000.ethernet eth0: PHY [0.0:00] driver [RTL8211FD-VD-CG Gigabit Ethernet] (irq=75)
[   22.028129][0 T620   ..] meson8b-dwmac fdc00000.ethernet eth0: Register MEM_TYPE_PAGE_POOL RxQ-0
[   22.039761][0 T620   ..] meson8b-dwmac fdc00000.ethernet eth0: No Safety Features support found
[   22.040211][0 T620   ..] meson8b-dwmac fdc00000.ethernet eth0: PTP not supported by HW
[   22.044174][0 T620   ..] meson8b-dwmac fdc00000.ethernet eth0: configuring for phy/rgmii link mode
[   22.076754][1 T553   ..] [dhd] _dhd_module_init: in Dongle Host Driver, version 101.10.591.68.32 (20240712-1)(429fcb0)
[   22.076754][1 T553   ..] /mnt/ebs6/p-anam/release/common/common14-5.15/out/bazel/output_user_root/e9418113fad0ecb82762087d7c04db8b/sandbox/linux-sandbox/26/execroot/__main__/out/android14-5.15/common/..//driver_modules/wifi_bt/wifi/broadcom/ap6xxx/bcmdhd.101.10.591.x/dhd_pcie compiled on Feb 26 2025 at 06:02:12
[   22.076754][1 T553   ..]
[   22.090070][1 T553   ..] [dhd] ANDROID_VERSION = 14
[   22.094724][1 T553   ..] [dhd] dhd_wlan_init_gpio: WL_HOST_WAKE=-1, oob_irq=72, oob_irq_flags=0x1
[   22.095159][1 T553   ..] [dhd] dhd_wlan_init_gpio: WL_REG_ON=-1
[   22.096393][1 T553   ..] [dhd] dhd_wifi_platform_load: Enter
[   22.096920][1 T553   ..] [dhd] wifi_platform_bus_enumerate device present 1
[   22.097654][1 T553   ..] [dhd] ======== Card detection to detect PCIE card! ========
[   22.098958][1 T553   ..] [dhd] Power-up adapter 'DHD generic adapter'
```

---

## 14. ApexService APEX Session Queries, Graphics Composer SELinux Audit Denials, and Broadcom BCM4359 / AP6xxx PCIe Wi-Fi Driver Enumeration, BAR Memory Enablement, and 4MB DMA Preallocation (Timestamps `22.196s` to `22.449s`)

Between timestamps `22.196s` and `22.449s`, execution handles APEX update session queries, logs graphics composer SELinux property denials, enumerates the Broadcom BCM4359 / AP6xxx PCIe Wi-Fi chip, enables PCIe BAR bus access, and pre-allocates 4MB of kernel DMA buffer memory:

### **14.1 Raw Serial Log Trace (Timestamps `22.196s` to `22.449s`)**

```text
[   22.196073][3 T446   ..] AidlLazyServiceRegistrar: Process has 1 (of 1 available) client(s) in use after notification apexservice has clients: 1
[   22.196347][1 T742   ..] apexd: getSessions() received by ApexService
[   22.197107][3 T446   ..] AidlLazyServiceRegistrar: Shutdown prevented by forcePersist override flag.
[   22.212610][0 T386   ..] [hdmitx:] avmute_store -1
[   22.214504][0 T236   ..] type=1400 audit(1789140429.364:12): avc:  denied  { read } for  comm="binder:290_3" name="u:object_r:system_prop:s0" dev="tmpfs" ino=343 scontext=u:r:hal_graphics_composer_default:s0 tcontext=u:object_r:system_prop:s0 tclass=file permissive=0
[   22.217835][3 T236   ..] type=1400 audit(1789140429.368:13): avc:  denied  { read } for  comm="binder:290_3" name="u:object_r:system_prop:s0" dev="tmpfs" ino=343 scontext=u:r:hal_graphics_composer_default:s0 tcontext=u:object_r:system_prop:s0 tclass=file permissive=0
[   22.306313][3 T553   ..] [dhd] wifi_platform_set_power = 1, sleep done: 200 msec
[   22.306568][3 T553   ..] [dhd] wifi_platform_bus_enumerate device present 1
[   22.307960][3 T553   ..] [dhd] ======== Card detection to detect PCIE card! ========
[   22.308522][3 T553   ..] Failed to set up IOMMU for device 0000:01:00.0; retaining platform DMA ops
[   22.309570][3 T553   ..] [dhd] dhdpcie_pci_probe : no mutex held
[   22.312271][3 T553   ..] [dhd] dhdpcie_pci_probe : set mutex lock
[   22.312519][3 T553   ..] [dhd] PCI_PROBE:  bus 0x1, slot 0x0,vendor 0x14E4, device 0x449D(good PCI location)
[   22.313738][3 T553   ..] [dhd] dhdpcie_init: found adapter info 'DHD generic adapter'
[   22.314814][3 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 3, size 139264
[   22.315855][3 T553   ..] [dhd] succeed to alloc static buf
[   22.316518][3 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 4, size 0
[   22.317619][3 T553   ..] pcieh 0000:01:00.0: enabling device (0000 -> 0002)
[   22.319144][3 T553   ..] [dhd] Disable CTO
[   22.319965][3 T553   ..] [dhd] DHD: dongle ram size is set to 1310720(orig 1310720) at 0x170000
[   22.320779][3 T553   ..] [dhd] dhd_bus_aspm_enable_rc_ep: NOT ASPM  CAPABLE rc_ep_aspm_cap: 0
[   22.321593][3 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 7, size 44352
[   22.322674][3 T553   ..] [dhd] dhd_conf_set_chiprev : devid=0x449d, chip=0xaae8, chiprev=2, svid=0x14e4, ssid=0xaae8
[   22.336550][3 T553   ..] [dhd] dhd_check_htput_chip: htput_support:1
[   22.336670][3 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 0, size 5152
[   22.354806][3 T553   ..] [dhd] dhd_pktid_map_init:2730: pktid_audit init succeeded 1025
[   22.355238][3 T553   ..] [dhd] dhd_pktid_map_init:2730: pktid_audit init succeeded 4097
[   22.360975][3 T553   ..] [dhd] dhd_pktid_map_init:2730: pktid_audit init succeeded 36865
[   22.428832][0 T553   ..] [dhd] dhd_ioctl_entry_local invalid parameter net 0000000000000000 dev_priv 0000000092e01f97
[   22.429523][0 T553   ..] [dhd] CFG80211-ERROR) wl_is_fils_supported : FILS NOT supported, err -22
[   22.437992][0 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 5, size 65536
[   22.444712][0 T553   ..] [dhd] CFG80211-ERROR) wl_is_fils_supported : FILS NOT supported, err -19
[   22.445384][1 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 19, size 65720
[   22.446520][1 T553   ..] [dhd] dhd_log_dump_init: kernel log buf size = 64KB; logdump_prsrv_tailsize = 80KB; limit prsrv tail size to = 9KB
[   22.447839][1 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 15, size 4194304
[   22.449673][1 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 16, size 8192
```

---

## 15. Broadcom AP6275P / BCM4359 PCIe Wi-Fi Firmware Loading, NVRAM Parsing, ARM Core Handshake, 64 Flowrings Creation, and `wlan0` Interface Registration (Timestamps `22.450s` to `23.028s`)

Between timestamps `22.450s` and `23.028s`, execution loads binary firmware and NVRAM settings into the Broadcom Wi-Fi chip SRAM, executes an ARM core out-of-reset handshake, allocates 64 PCIe flowrings, loads the Country Location Matrix (CLM), sets regulatory domain `CN`, and registers network interface `wlan0`:

### **15.1 Raw Serial Log Trace (Timestamps `22.450s` to `23.028s`)**

```text
[   22.450273][1 T553   ..] [dhd] dhd_os_attach_pktlog(): dhd_os_attach_pktlog attach
[   22.451349][1 T553   ..] [dhd] dhd_attach(): thread:dhd_watchdog_thread:3d0 started
[   22.452002][1 T553   ..] [dhd] dhd_deferred_work_init: work queue initialized
[   22.452886][1 T553   d.] [dhd] dhd_tcpack_suppress_set: TCP ACK Suppress mode 0 -> mode 3
[   22.453897][1 T553   d.] [dhd] dhd_tcpack_suppress_set: TCPACK_INFO_MAXNUM=40, TCPDATA_INFO_MAXNUM=40
[   22.455317][1 T553   ..] [dhd] dhd_cpumasks_init CPU masks primary(big)=0xf0 secondary(little)=0xe
[   22.456210][0 T22    ..] [dhd] dhd_cpu_startup_callback(): LB data is not initialized yet.
[   22.457916][1 T23    ..] [dhd] dhd_cpu_startup_callback(): LB data is not initialized yet.
[   22.459979][2 T30    ..] [dhd] dhd_cpu_startup_callback(): LB data is not initialized yet.
[   22.460443][3 T37    ..] [dhd] dhd_cpu_startup_callback(): LB data is not initialized yet.
[   22.461479][1 T553   ..] [dhd] dhd_get_memdump_info: MEMDUMP ENABLED = 3
[   22.462212][1 T553   ..] [dhd] dhdpcie_bus_attach: making DHD_BUS_DOWN
[   22.463021][1 T553   ..] [dhd] dhdpcie_init: rc_dev from dev->bus->self (16c3:abcd) is 0000000000000000
[   22.464255][1 T553   ..] [dhd] dhdpcie_init: rc_ep_aspm_cap: 1 rc_ep_l1ss_cap: 1
[   22.465245][1 T553   ..] [dhd] dhdpcie_request_irq: MSI enabled, irq=77
[   22.465966][1 T553   ..] [dhd] dhd_bus_download_firmware: firmware path=../../etc/wifi/43752a2/fw_bcm43752a2_pcie_ag.bin, nvram path=../../etc/wifi/43752a2/nvram_ap6275p.txt
[   22.467928][1 T553   ..] [dhd] dhd_conf_set_path_params : Final fw_path=../../etc/wifi/43752a2/fw_bcm43752a2_pcie_ag.bin
[   22.469207][1 T553   ..] [dhd] dhd_conf_set_path_params : Final nv_path=../../etc/wifi/43752a2/nvram_ap6275p.txt
[   22.470722][1 T553   ..] [dhd] dhd_conf_set_path_params : Final clm_path=../../etc/wifi/43752a2/clm_bcm43752a2_pcie_ag.blob
[   22.471863][1 T553   ..] [dhd] dhd_conf_set_path_params : Final conf_path=../../etc/wifi/43752a2/config_bcm43752a2_pcie_ag.txt
[   22.474123][1 T553   ..] [dhd] dhd_os_get_img_fwreq: ../../etc/wifi/43752a2/config_bcm43752a2_pcie_ag.txt (1612 bytes) open success
[   22.475178][1 T553   ..] [dhd] dhd_conf_read_country : 132 country in list
[   22.475905][1 T553   ..] [dhd] dhd_conf_read_pm_params : PM = 0
[   22.476656][1 T553   ..] [dhd] dhd_conf_read_pkt_filter : dhd_master_mode = 1
[   22.477412][1 T553   ..] [dhd] dhd_conf_read_pm_params : suspend_mode = 1
[   22.478291][1 T553   ..] [dhd] dhd_conf_read_pm_params : insuspend = 0x6
[   22.479102][1 T553   ..] [dhd] dhd_conf_read_pkt_filter : pkt_filter_del id = 100 102 103 104 105 106 107
[   22.480291][1 T553   ..] [dhd] dhd_conf_read_pkt_filter : pkt_filter_add[0][] = 141 0 0 0 0xFFFFFFFFFFFF 0x000000000000
[   22.481633][1 T553   ..] [dhd] dhd_conf_read_pm_params : insuspend = 0x7
[   22.482602][1 T553   ..] [dhd] dhd_conf_read_others : wl_suspend = 3=0
[   22.483288][1 T553   ..] [dhd] dhd_conf_read_others : wl_resume = 2=0
[   22.484097][1 T553   ..] [dhd] d2h_intr_method -> PCIE_MSI(1); d2h_intr_control -> HOST_IRQ(1)
[   22.485229][1 T553   ..] [dhd] dhdpcie_download_code_file: dhd_tcm_test_enable 0, dhd_tcm_test_status 0, dhd_tcm_test_mode 2
[   22.486938][1 T553   ..] [dhd] dhdpcie_download_code_file: download firmware ../../etc/wifi/43752a2/fw_bcm43752a2_pcie_ag.bin
[   22.499411][3 T553   ..] [dhd] dhd_os_get_img_fwreq: ../../etc/wifi/43752a2/fw_bcm43752a2_pcie_ag.bin (934977 bytes) open success
[   22.597726][3 T553   ..] [dhd] dhd_os_get_img_fwreq: ../../etc/wifi/43752a2/nvram_ap6275p.txt (7888 bytes) open success
[   22.598812][3 T553   ..] [dhd] # AP6275PR3_NVRAM_V1.4_20230131A
[   22.601564][3 T553   ..] [dhd] dhdpcie_bus_write_vars: Download, Upload and compare of NVRAM succeeded.
[   22.602068][3 T553   ..] [dhd] dhdpcie_bus_write_vars: New varsize is 6212, length token(nvram_csm)=0xf9ee0611
[   22.605419][3 T553   ..] [dhd] Download and compare of TLV 0xfeedc0de succeeded (size 128, addr 2ae730).
[   22.606533][3 T553   ..] [dhd] dhdpcie_bus_download_state: Took ARM out of Reset
[   22.701232][3 T553   ..] [dhd] dhdpcie_readshared: addr=0x20c0f4 nvram_csm=0xf9ee0611
[   22.703427][3 T553   ..] [dhd] ### Total time ARM OOR to Readshared pass took 97231 usec ###
[   22.704463][3 T553   ..] [dhd] dhdpcie_readshared: PCIe shared addr (0x0020c0f4) read took 90000 usec before dongle is ready
[   22.706079][3 T553   ..] [dhd] FW supports DAR ? N
[   22.708828][3 T553   ..] [dhd] dhdpcie_readshared: max H2D queues 64
[   22.708946][3 T553   ..] [dhd] FW supports MD ring ? N
[   22.714934][3 T553   ..] [dhd] dhd_bus_init: Enabling bus->intr_enabled
[   22.715086][3 T553   ..] [dhd] dhdpcie_oob_intr_register OOB irq=72 flags=0x1
[   22.716029][3 T553   ..] [dhd] dhdpcie_oob_intr_register: enable_irq_wake
[   22.716855][3 T553   ..] [wireless]: [dhd] STATIC-MSG) bcmdhd_mem_prealloc : section 9, size 32896
[   22.717940][3 T553   ..] [dhd] dhd_prot_init:4169: h2d_max_txpost = 512
[   22.718846][3 T553   ..] [dhd] dhd_prot_init:4175: h2d_htput_max_txpost = 2048
[   22.719659][3 T553   ..] [dhd] dhd_prot_init: max_rxbufpost:511 rx_buf_burst:64 rx_bufpost_threshold:64
[   22.720983][3 T553   ..] [dhd] ENABLING DW:0
[   22.721644][3 T553   ..] [dhd] dhd_prot_d2h_sync_init(): D2H sync mechanism is XORCSUM
[   22.722543][3 T553   ..] [dhd] dhd_bus_hostready : Read PCICMD Reg: 0x00100406
[   22.740310][3 T553   ..] [dhd] dhd_bus_hostready: Ring Hostready:1
[   22.740406][3 T553   ..] [dhd] Attach flowrings pool for 64 rings
[   22.744300][3 T553   ..] [dhd] trying to send create d2h info ring: id 70
[   22.745459][3 T553   d.] [dhd] dhd_send_d2h_ringcreate ringid: 3 idx: 70 max_h2d: 67
[   22.746121][0 T243   .s] [dhd] dhd_prot_process_d2h_ring_create_complete ring create Response status = 0 ring 3, id 0xfffc
[   22.747163][0 T243   .s] [dhd] info buffer post after ring create
[   22.748070][3 T553   ..] [dhd] wlc_ver_major 12, wlc_ver_minor 1
[   22.748632][3 T553   ..] [dhd] dhd_get_memdump_info: MEMDUMP ENABLED = 3
[   22.751584][0 T553   ..] [dhd] dhd_sync_with_dongle: GET_REVINFO device 0x449d, vendor 0x14e4, chipnum 0xaae8
[   22.754622][0 T553   ..] [dhd] dhd_sync_with_dongle: RxBuf Post : 2048
[   22.754764][0 T553   ..] [dhd] dhd_sync_with_dongle: RxBuf Post Alloc : 2048
[   22.760751][0 T553   ..] [dhd] dhd_preinit_ioctls: preinit_status IOVAR not supported, use legacy preinit
[   22.761272][0 T553   d.] [dhd] dhd_tcpack_suppress_set 397: already set to 3
[   22.764740][3 T553   ..] [dhd] dhd_legacy_preinit_ioctls: hostwake_oob enabled
[   22.767175][2 T553   ..] [dhd] dhd_legacy_preinit_ioctls: use firmware generated mac_address c0:f5:35:cb:53:67
[   22.768905][3 T553   ..] [dhd] dhd_os_get_img_fwreq: ../../etc/wifi/43752a2/clm_bcm43752a2_pcie_ag.blob (28865 bytes) open success
[   22.772749][0 T553   ..] [dhd] dhd_check_current_clm_data: ----- This FW is not included CLM data -----
[   22.803244][2 T553   ..] [dhd] dhd_apply_default_clm: CLM download succeeded
[   22.805918][2 T553   ..] [dhd] dhd_check_current_clm_data: ----- This FW is included CLM data -----
[   22.817016][3 T553   ..] [dhd] Firmware up: op_mode=0x0005, MAC=c0:f5:35:cb:53:67
[   22.837253][0 T553   ..] [dhd] dhd_legacy_preinit_ioctls: event_log_max_sets: 26 ret: 0
[   22.850785][3 T553   ..] [dhd] arp_enable:1 arp_ol:0
[   22.850832][3 T553   ..] [dhd] dhd_conf_add_pkt_filter : 141 0 0 0 0xFFFFFFFFFFFF 0x000000000000
[   22.874394][2 T553   ..] [dhd]   Driver: 101.10.591.68.32 (20240712-1)
[   22.874394][2 T553   ..] [dhd]   Firmware: wl0: Apr 30 2024 11:12:44 version 18.35.387.23.237 (gbcd01fd4) FWID 01-37a6ac2c
[   22.874394][2 T553   ..] [dhd]   CLM: 9.9.5_SS (2020-02-17 20:44:22)
[   22.881187][1 T553   ..] [dhd] dhd_pno_init: Support Android Location Service
[   22.887321][3 T553   ..] [dhd] rtt_do_get_ioctl: failed to send getbuf proxd iovar (CMD ID : 1), status=-23
[   22.887864][3 T553   ..] [dhd] dhd_rtt_init : FTM is not supported
[   22.888625][3 T553   ..] [dhd] dhd_rtt_init EXIT, err = 0
[   22.934626][3 T553   ..] [dhd] dhd_legacy_preinit_ioctls: d3_hostwake_delay IOVAR not present, proceed
[   22.966836][0 T978   .s] [dhd] dhd_rx_frame: net device is NOT registered. drop event packet
[   22.967287][0 T978   .s] [dhd] dhd_rx_frame: net device is NOT registered. drop event packet
[   22.968292][0 T978   ds] [dhd] dhd_update_interface_flow_info: ifindex:0 previous role:0 new role:0
[   22.969420][0 T978   .s] [dhd] dhd_rx_frame: net device is NOT registered. drop event packet
[   22.972904][2 T553   ..] [dhd] dhd_conf_set_country : set country CN, revision 0
[   22.977345][3 T553   ..] [dhd] dhd_conf_set_country : Country code: CN (CN/0)
[   22.988874][2 T553   ..] [dhd] Dongle Host Driver, version 101.10.591.68.32 (20240712-1)(429fcb0)
[   22.988874][2 T553   ..] /mnt/ebs6/p-anam/release/common/common14-5.15/out/bazel/output_user_root/e9418113fad0ecb82762087d7c04db8b/sandbox/linux-sandbox/26/execroot/__main__/out/android14-5.15/common/..//driver_modules/wifi_bt/wifi/broadcom/ap6xxx/bcmdhd.101.10.591.x/dhd_pcie compiled on Feb 26 2025 at 06:02:12
[   22.988874][2 T553   ..]
[   23.028357][3 T553   ..] [dhd] Register interface [wlan0]  MAC: c0:f5:35:cb:53:67
```

---

## 16. Wi-Fi `wlan1` Virtual Interface Registration, `wlan0` Interface Activation, `wpa_supplicant` & P2P Service Launch, Regulatory Domain Override to US, Network Stack SELinux Audit, Broadcom Bluetooth Controller Power-On, and Audio ALSA Re-configuration (Timestamps `23.038s` to `26.122s`)

Between timestamps `23.038s` and `26.122s`, execution registers virtual network interface `wlan1`, activates primary interface `wlan0`, starts `wpa_supplicant` and Wi-Fi Direct (P2P), overrides the regulatory country code to `US`, audits SELinux network permissions, powers on the Broadcom Bluetooth radio, and re-configures ALSA audio output:

### **16.1 Raw Serial Log Trace (Timestamps `23.038s` to `26.122s`)**

```text
[   23.028357][3 T553   ..]
[   23.038961][0 T553   ..] [dhd] Register interface [wlan1]  MAC: c2:f5:35:cb:53:67
[   23.038961][0 T553   ..]
[   23.039554][0 T553   ..] [dhd] dhdpcie_pci_probe : mutex is released.
[   23.081449][1 T553   ..] [dhd] _dhd_module_init: Exit err=0
[   23.200062][1 T1025  ..] [dhd] [wlan0] dhd_open : Enter
[   23.200111][1 T1025  ..] [dhd] Dongle Host Driver, version 101.10.591.68.32 (20240712-1)(429fcb0)
[   23.200111][1 T1025  ..] /mnt/ebs6/p-anam/release/common/common14-5.15/out/bazel/output_user_root/e9418113fad0ecb82762087d7c04db8b/sandbox/linux-sandbox/26/execroot/__main__/out/android14-5.15/common/..//driver_modules/wifi_bt/wifi/broadcom/ap6xxx/bcmdhd.101.10.591.x/dhd_pcie compiled on Feb 26 2025 at 06:02:12
[   23.200111][1 T1025  ..]
[   23.211414][1 T1025  ..] [dhd] ANDROID_VERSION = 14
[   23.211462][1 T1025  ..] [dhd] dhd_open: ######### called for ifidx=0 #########
[   23.246509][0 T1010  ds] [dhd] dhd_update_interface_flow_info: ifindex:0 previous role:0 new role:0
[   23.246977][0 T1010  .s] [dhd] dhd_rx_frame: net device is NOT registered. drop event packet
[   23.265199][3 T1025  ..] [dhd] [wlan0] wl_cfg80211_up : Roam channel cache enabled
[   23.270545][0 T1025  ..] [dhd] [wlan0] dhd_open : Exit ret=0
[   23.290033][0 T1025  ..] [dhd] dhd_pri_open : no mutex held
[   23.290081][0 T1025  ..] [dhd] dhd_pri_open : set mutex lock
[   23.290821][0 T1025  ..] [dhd] [wlan0] dhd_open : Primary net_device is already up
[   23.291929][0 T1025  ..] [dhd] [wlan0] dhd_pri_open : tx queue started
[   23.310560][0 T1025  ..] [dhd] [wlan0] custom_xps_map_set : Done. mapping cpu
[   23.310777][0 T1025  ..] [dhd] dhd_pri_open : mutex is released.
[   23.440835][1 T627   ..] [dhd] dhd_ioctl_entry_local bad ifidx
[   23.756141][1 T553   ..] netlink: 'binder:514_1': attribute type 15 has an invalid length.
[   23.970828][3 T1083  ..] capability: warning: `wpa_supplicant' uses 32-bit capabilities (legacy support in use)
[   24.087093][0 T1083  ..] [dhd] P2P interface registered
[   24.139829][0 T1083  ..] [dhd] P2P interface started
[   24.259677][2 T1083  ..] [dhd] wl_android_priv_cmd: Android private cmd "COUNTRY US" on wlan0
[   24.263783][1 T1083  ..] [dhd] dhd_conf_set_country : set country US, revision 0
[   24.268491][2 T1083  ..] [dhd] dhd_conf_set_country : Country code: US (US/0)
[   24.299163][0 T1083  ..] [dhd] wldev_set_country: set country for US as US rev 0
[   24.299848][2 T70    ..] [dhd] CFG80211-ERROR) wl_cfg80211_reg_notifier : Set country code US from User
[   24.302605][2 T70    ..] [dhd] dhd_conf_same_country : country code = US/0 is already configured
[   24.869244][0 T236   ..] type=1400 audit(1789140432.020:14): avc:  granted  { read } for  comm="rkstack.process" name="psched" dev="proc" ino=4026531995 scontext=u:r:network_stack:s0 tcontext=u:object_r:proc_net:s0 tclass=file
[   24.871790][0 T236   ..] type=1400 audit(1789140432.020:15): avc:  granted  { read open } for  comm="rkstack.process" path="/proc/1096/net/psched" dev="proc" ino=4026531995 scontext=u:r:network_stack:s0 tcontext=u:object_r:proc_net:s0 tclass=file
[   24.880679][0 T236   ..] type=1400 audit(1789140432.020:16): avc:  granted  { getattr } for  comm="rkstack.process" path="/proc/1096/net/psched" dev="proc" ino=4026531995 scontext=u:r:network_stack:s0 tcontext=u:object_r:proc_net:s0 tclass=file
[   25.159290][0 T932   ..] [amlv4l]:[3]: reset mode: 1, es frames buffering: 0
[   25.162307][0 T932   ..] [amlv4l]:[3]: reset mode: 1, es frames buffering: 0
[   25.312994][1 T446   ..] AidlLazyServiceRegistrar: Process has 0 (of 1 available) client(s) in use after notification apexservice has clients: 0
[   25.314956][1 T446   ..] AidlLazyServiceRegistrar: Shutdown prevented by forcePersist override flag.
[   25.576256][3 T796   ..] [snd-soc]: spk_mute_set: mute flag = 1
[   25.744357][0 T486   ..] [wireless]: [wifi_power_ioctl] CUR status: 0
[   25.771652][2 T486   ..] [wireless]: BT_RADIO going: on
[   25.771694][2 T486   ..] [wireless]: AML_BT: going ON,btpower_evt=1
[   26.051138][0 T969   ..] [snd-soc]: drc high cut scale set to 0%
[   26.051564][0 T969   ..] [snd-soc]: drc low boost scale set to 0%
[   26.053161][3 T969   ..] [snd-soc]: drc mode set to RF
[   26.107503][3 T673   ..] audio_ddr_mngr: frddrs[1] released by device fe330000.audiobus:spdif@0
[   26.108529][3 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 9, use_vadtop 0
[   26.108871][3 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 13, use_vadtop 0
[   26.109872][3 T673   ..] [snd-soc]: hdmitx src switch to spdif 0
[   26.110635][3 T673   ..] [snd-soc]: sharebuffer_spdifout_prepare  get_hdmitx_audio_src(rtd->card):0  spdif_id:0
[   26.111840][3 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:0 size:0 chs:0 i2s_ch_mask:0 aud_src_if:0
[   26.113210][3 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn setting, must be 2, 4, 6 or 8, Rst as def
[   26.114405][3 T673   ..] [hdmitx:] aout notify format CT_PCM
[   26.117182][3 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   26.117244][3 T673   ..] [hdmitx:] set audio param
[   26.117847][3 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:3 size:1 chs:2 i2s_ch_mask:1 aud_src_if:0
[   26.119267][3 T673   ..] [hdmitx:] aout notify format CT_PCM
[   26.122034][3 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   26.122105][3 T673   ..] [hdmitx:] set audio param
```

---

## 17. HDMI Dolby Digital Bitstream Audio Handshake & N-Clock Recovery, V4L2 Decoder & De-interlacer (DIM) Teardown, Binder IPC Transient Allocation, ApexService `markBootCompleted()` Event, SELinux User 0 Credential-Encrypted (`CE`) Storage Initialization, and Secure VDEC 16MB DMABUF Reservation (Timestamps `26.138s` to `27.981s`)

Between timestamps `26.138s` and `27.981s`, execution configures HDMI Dolby Digital bitstream audio, resets the V4L2 video decoder and DIM de-interlacer pipelines, handles a transient Binder IPC buffer allocation, fires the ApexService `markBootCompleted()` event, restores SELinux contexts for per-user credential-encrypted storage, and reserves secure decoder DMA memory:

### **17.1 Raw Serial Log Trace (Timestamps `26.138s` to `27.981s`)**

```text
[   26.138853][3 T673   ..] audio_ddr_mngr: frddrs[1] registered by device fe330000.audiobus:spdif@1
[   26.139448][3 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:0 size:0 chs:0 i2s_ch_mask:0 aud_src_if:0
[   26.140659][3 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn setting, must be 2, 4, 6 or 8, Rst as def
[   26.141803][3 T673   ..] [hdmitx:] aout notify format CT_PCM
[   26.144686][3 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   26.144747][3 T673   ..] [hdmitx:] set audio param
[   26.145696][3 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:1 rate:0 size:0 chs:0 i2s_ch_mask:0 aud_src_if:0
[   26.146761][3 T673   ..] [hdmitx:] *HDMITX_ERROR* audio chn setting, must be 2, 4, 6 or 8, Rst as def
[   26.147860][3 T673   ..] [hdmitx:] aout notify format CT_PCM
[   26.150682][3 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   26.150742][3 T673   ..] [hdmitx:] set audio param
[   26.151338][3 T673   ..] [hdmitx:] hdmitx_audio_notify_callback[804] type:10 rate:3 size:1 chs:2 i2s_ch_mask:1 aud_src_if:0
[   26.152717][3 T673   ..] [hdmitx:] aout notify format CT_DOLBY_D
[   26.153493][3 T673   ..] [hdmitx:] update audio N 23296
[   26.155597][3 T673   ..] [hdmitx:] audio: reset audio fifo_rst
[   26.156305][3 T673   ..] [hdmitx:] set audio param
[   26.159103][3 T673   ..] [snd-soc]: release_spdif_same_src(), 1039 src sel 3
[   26.159308][3 T673   ..] [snd-soc]: ss_free() samesrc 3, lvl 1
[   26.160473][3 T673   ..] audio_ddr_mngr: frddrs[2] registered by device fe330000.audiobus:spdif@0
[   26.164319][3 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 9, use_vadtop 0
[   26.164619][3 T673   ..] [snd-soc]: aml_audio_reset, reg 0xa, shift 13, use_vadtop 0
[   27.020416][3 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   27.023463][3 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   27.208827][0 T932   ..] 0#3-V4L2Decoder (932) used greatest stack depth: 9680 bytes left
[   27.209668][3 T925   ..] [amlv4l]:[3]: reset mode: 1, es frames buffering: 0
[   27.214049][3 T925   ..] deinit session
[   27.214077][3 T925   ..] deinit session producer-2
[   27.214499][3 T925   ..] deinit has_unusing_fifo [1]
[   27.215085][0 T87    ..] p_fifo[2] name[aml-vcodec-dec] state[0] to  MP_FIFO_UNUSED
[   27.288101][1 T969   ..] [snd-soc]: dts_dec_control/0x0
[   27.298300][2 T97    ..] dim:unreg:2keep[0], msk:[0x0,0x0]
[   27.298355][2 T97    ..] dim:hf:r:ch[2]:cnt[0]
[   27.298861][2 T97    ..] dim:di_unreg_setting:
[   27.301889][1 T386   ..] dim:unreg:use[20]ms
[   27.301931][1 T386   ..] dim:ins:destroy:ch[2]:end
[   27.302824][1 T386   ..] di process release
[   27.306481][0 T224   ..] servicemanager: Found android.hardware.power.IPower/default in device VINTF manifest.
[   27.402071][1 T386   ..] vc:[0]vc: set enable index=0, val=0
[   27.402127][1 T386   ..] VID: store VD0 path_id changed 2->-1
[   27.402926][1 T386   ..] VID: VD1 off
[   27.403273][1 T386   ..] VID: VD1 set global output as 0
[   27.403940][1 T386   ..] common_vf_unreg_provider video_render.0: vd1 used:true, vd2 used:false, vd3 used:false, f_wait:true (0 0), b_out:0, cur_buf:00000000195660c3
[   27.424912][1 T386   ..] dp_put_file_ext:put_file=ffffff8015d62f00, file_count=4
[   27.425507][1 T386   ..] dv_inst_unmap 0
[   27.425693][1 T386   ..] dv_inst_unmap 0 OK
[   27.426207][1 T386   ..] dp_buf:[0:3]di buf_mgr free
[   27.466375][3 T680   ..] binder_alloc: 243: binder_alloc_buf, no vma
[   27.466499][3 T680   ..] binder: 550:680 transaction failed 29189/-3, size 556-0 line 3457
[   27.524071][0 T65    ..] binder: undelivered transaction 35307, process died.
[   27.743187][1 T224   ..] servicemanager: Found android.hardware.wifi.IWifi/default in device VINTF manifest.
[   27.754406][0 T224   ..] servicemanager: Notifying apexservice they do (previously: don't) have clients when service is guaranteed to be in use
[   27.755582][3 T446   ..] AidlLazyServiceRegistrar: Process has 1 (of 1 available) client(s) in use after notification apexservice has clients: 1
[   27.756970][3 T446   ..] AidlLazyServiceRegistrar: Shutdown prevented by forcePersist override flag.
[   27.757874][2 T742   ..] apexd: markBootCompleted() received by ApexService
[   27.759446][2 T224   ..] servicemanager: Found android.hardware.wifi.IWifi/default in device VINTF manifest.
[   27.777589][3 T224   ..] servicemanager: Found android.hardware.wifi.IWifi/default in device VINTF manifest.
[   27.801910][3 T224   ..] servicemanager: Found android.hardware.wifi.IWifi/default in device VINTF manifest.
[   27.838153][3 T224   ..] servicemanager: Found android.hardware.wifi.IWifi/default in device VINTF manifest.
[   27.844171][2 T1461  ..] selinux: SELinux: Skipping restorecon on directory(/data/system_ce/0)
[   27.851902][1 T1462  ..] selinux: SELinux: Skipping restorecon on directory(/data/vendor_ce/0)
[   27.856481][1 T1463  ..] selinux: SELinux: Skipping restorecon on directory(/data/misc_ce/0)
[   27.860874][1 T224   ..] servicemanager: Found android.hardware.wifi.IWifi/default in device VINTF manifest.
[   27.980781][0 T156   ..] can't find node dmabuf_manage's  config:secure_vdec_def_size_bytes
[   27.981151][0 T156   ..] set config failed dmabuf_manage.secure_vdec_def_size_bytes=16777216
```

---

## 18. System Boot Completion (`sys.boot_completed=1`), APEX Lazy AIDL Daemon Shutdown, Bluetooth Remote Pairing, 5GHz 80MHz Wi-Fi Connection to "3Meadows", OP-TEE Widevine L1 & Secmem Timer Execution, SELinux System App Write Audit, and NOHZ Tickless Idle Warnings (Timestamps `28.180s` to `63.188s`)

Between timestamps `28.180s` and `63.188s`, execution asserts `sys.boot_completed=1`, shuts down idle APEX AIDL daemons, pairs the Bluetooth remote, connects to a 5GHz 80MHz Wi-Fi network, re-enters OP-TEE Widevine L1 with Secmem timer active, audits SELinux system app write denials, and handles CPU tickless idle (`NOHZ`) softirq states:

### **18.1 Raw Serial Log Trace (Timestamps `28.180s` to `63.188s`)**

```text
[   28.180797][3 T627   ..] [dhd] [wlan0] wl_run_escan : LEGACY_SCAN sync ID: 0, bssidx: 0
[   28.471565][2 T284   ..] android.hardware.boot-service: sys.boot_completed: 1
[   28.739351][3 T742   ..] apexd: destroyCeSnapshotsNotSpecified() received by ApexService user_id : 0 retain_rollback_ids : []
[   29.119150][1 T969   d.] meson-ir fe084040.ir: please set valid keymap name first
[   29.127794][0 T341   ..] apexd: Deleting unused dm device com.amlogic.mediaextractor
[   29.130039][1 T341   ..] apexd: Didn't generate uevent for [com.amlogic.mediaextractor] removal
[   29.130523][1 T341   ..] apexd: Failed to delete dm-device com.amlogic.mediaextractor
[   29.133159][1 T341   ..] apexd: Deleting unused dm device com.android.apex.cts.shim
[   29.135674][1 T341   ..] apexd: Didn't generate uevent for [com.android.apex.cts.shim] removal
[   29.136111][1 T341   ..] apexd: Failed to delete dm-device com.android.apex.cts.shim
[   30.423143][0 T236   ..] type=1400 audit(1789140437.572:17): avc:  denied  { search } for  comm="nalyticsservice" name="vendor" dev="tmpfs" ino=2 scontext=u:r:system_app:s0 tcontext=u:object_r:mnt_vendor_file:s0 tclass=dir permissive=0
[   30.487346][3 T66    ..] input: ruwido BLE Keyboard as /devices/virtual/misc/uhid/0005:119B:2197.0001/input/input8
[   30.488146][3 T66    ..] hid-generic 0005:119B:2197.0001: input,hidraw0: BLUETOOTH HID v1.11 Keyboard [ruwido BLE] on 22:22:ed:bc:60:4e
[   30.589726][2 T1083  ..] [dhd] wl_android_priv_cmd: Android private cmd "BTCOEXSCAN-START" on wlan0
[   30.630364][1 T1083  ..] [dhd] wl_android_priv_cmd: Android private cmd "BTCOEXSCAN-START" on wlan0
[   31.231557][2 T1     ..] selinux: SELinux: Skipping restorecon on directory(/data/misc_ce/0/apexdata/com.android.wifi)
[   32.511985][1 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   32.515974][0 T224   ..] servicemanager: Found android.hardware.graphics.allocator.IAllocator/default in device VINTF manifest.
[   32.750653][2 T110   ..] [TEE] E/TC:? 00 tee_efuse_read_obj:143 secmon_call cmd 0x8200003b fail with 0x65
[   32.751176][2 T110   ..] [TEE] E/TA:   get_efuse_object:232 [ATTEST-TA] Get FEAT_DISABLE_CERTCHAIN_UNIQUE object failed 0xffff000e
[   32.752632][2 T110   ..] [TEE] I/TA: MM-module-name:Netflix TA,Version:2.2.5-gc882a60(build:6886)
[   33.744101][3 T492   ..] amhdmitx_compat_ioctl20
[   33.744113][3 T492   ..] amhdmitx_ioctl20
[   33.751120][1 T492   ..] amhdmitx_compat_ioctl20
[   33.751161][1 T492   ..] amhdmitx_ioctl20
[   33.751747][1 T492   ..] HDMI_GET_MANUFACTURER_NAME
[   33.764715][3 T492   ..] amhdmitx_compat_ioctl20
[   33.764761][3 T492   ..] amhdmitx_ioctl20
[   33.772866][3 T492   ..] amhdmitx_compat_ioctl20
[   33.772909][3 T492   ..] amhdmitx_ioctl20
[   33.773264][3 T492   ..] HDMI_GET_MANUFACTURER_NAME
[   34.572361][3 T1083  ..] [dhd] [wlan0] wl_handle_assoc_hints : replace chspec 0xd028 to previous scan chanspec 0xd926
[   34.579707][0 T1083  ..] [dhd] do_iovar_aml_enable aml failed -23
[   34.584554][3 T1083  ..] [dhd] CFG80211-ERROR) wl_set_multi_akm : Invalid MultiAKM combination 0x8080
[   34.603069][0 T1083  ..] [dhd] [wlan0] wl_ext_set_chanspec : channel 5g-40(0xe12a 80MHz)
[   34.608365][0 T1083  ..] [dhd] [wlan0] wl_conn_debug_info : Connecting with a8:5b:f7:56:b3:70 ssid "3Meadows", len (8), channel=5g-40(chan_cnt=1), sec=wpa2/psk/mfpc/aes, rssi=-50
[   34.701293][0 T2143  ds] [dhd] dhd_update_interface_flow_info: ifindex:0 previous role:0 new role:0
[   34.701759][0 T2143  ds] [dhd] dhd_update_multicilent_flow_rings: ifindex 0
[   34.707458][1 T305   ..] [dhd] [wlan0] wl_iw_event : Link UP with a8:5b:f7:56:b3:70
[   34.707746][1 T305   ..] [dhd] [wlan0] wl_ext_iapsta_link : [S] Link UP with a8:5b:f7:56:b3:70
[   34.713463][1 T304   ..] [dhd] [wlan0] wl_bss_connect_done : Report connect result - connection succeeded
[   34.721369][3 T41    ds] [dhd] dhd_prot_flow_ring_create: Send Flow Create Req flow ID 61 for peer a8:5b:f7:56:b3:70 prio 3 ifindex 0 items 512
[   34.722749][0 T722   .s] [dhd] dhd_prot_flow_ring_create_response_process: Flow Create Response status = 0 Flow 61
[   34.731079][1 T1083  ..] [dhd] [wlan0] wl_add_keyext : key index (0) for a8:5b:f7:56:b3:70
[   34.784606][0 T1083  ..] [dhd] [wlan0] wl_cfg80211_set_suspend_bcn_li_dtim : bcn_li_dtim:0 lpas:0 bcn_to_dly:0
[   35.314678][1 T224   ..] servicemanager: Notifying apexservice they don't (previously: do) have clients when we now have no record of a client
[   35.316272][1 T341   ..] AidlLazyServiceRegistrar: Process has 0 (of 1 available) client(s) in use after notification apexservice has clients: 0
[   35.317266][1 T341   ..] AidlLazyServiceRegistrar: Trying to shut down the service. No clients in use for any service in process.
[   35.322155][1 T224   ..] servicemanager: Unregistering apexservice
[   35.322333][1 T224   ..] BpBinder: onLastStrongRef automatically unlinking death recipients:
[   35.323574][1 T341   ..] AidlLazyServiceRegistrar: Unregistered all clients and exiting
[   35.327588][3 T742   ..] printk: binder:341_4: 161 output lines suppressed due to ratelimiting
[   35.832065][3 T41    ds] [dhd] dhd_prot_flow_ring_create: Send Flow Create Req flow ID 65 for peer 33:33:ff:f4:60:28 prio 0 ifindex 0 items 2048
[   35.833488][0 T2281  .s] [dhd] dhd_prot_flow_ring_create_response_process: Flow Create Response status = 0 Flow 65
[   36.089691][2 T553   ..] [dhd] dhd_dev_apf_add_filter: id 200
[   36.851182][1 T553   ..] [dhd] dhd_dev_apf_add_filter: id 200
[   37.443471][3 T41    ds] [dhd] dhd_prot_flow_ring_create: Send Flow Create Req flow ID 60 for peer 2e:b8:ed:6c:ce:1f prio 2 ifindex 0 items 512
[   37.444846][0 T1582  .s] [dhd] dhd_prot_flow_ring_create_response_process: Flow Create Response status = 0 Flow 60
[   38.894349][2 T110   ..] [TEE] I/TA:
[   38.894389][2 T110   ..] [TEE] MM-module-name:Widevine TA,Version:18.7-r1.3-g291e32a(build:7403)
[   38.895243][2 T110   ..] [TEE] I/TA: [TA_CreateEntryPoint:523]
[   38.895990][2 T110   ..] [TEE] MM-module-name:Secmem TA,Version:3.0.12-g9d28732(build:6846)
[   38.897044][2 T110   ..] [TEE] [Secmem_TimerEnable:112] Secmem platform support hardware cur check timer
[   38.907420][2 T110   ..] [TEE] I/TA:
[   38.907459][2 T110   ..] [TEE] MM-module-name:libteesmp.a, v2
[   38.908014][2 T110   ..] [TEE] 8ab190369deb7c8877a104816087bda5ed3db23e
[   38.908907][2 T110   ..] [TEE] I494578c4dea68a0b8b3edda347e9fcab83cd7a2d
[   40.372093][3 T66    ..] binder: undelivered TRANSACTION_COMPLETE
[   40.372212][3 T66    ..] binder: undelivered transaction 61955, process died.
[   42.890067][1 T224   ..] servicemanager: Found android.hardware.security.keymint.IRemotelyProvisionedComponent/default in device VINTF manifest.
[   45.754914][0 T236   ..] type=1400 audit(1790584998.118:18): avc:  denied  { write } for  comm="com.brdgwtr.dms" name="com.brdgwtr.dms-Dd7aD61naHvM3YlYfWcY6g==" dev="dm-50" ino=260352 scontext=u:r:system_app:s0 tcontext=u:object_r:apk_data_file:s0 tclass=dir permissive=0
[   47.022196][0 T944   ..] binder: 713:944 transaction failed 29189/-22, size 116-0 line 3270
[   50.184743][0 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3209 b tail=0 logMask=4 pid=0 start=1790604802446000000ns deadline=0ns
[   50.630071][0 T236   ..] type=1400 audit(1790585003.086:19): avc:  denied  { write } for  comm="dms:pdssService" name="com.brdgwtr.dms-Dd7aD61naHvM3YlYfWcY6g==" dev="dm-50" ino=260352 scontext=u:r:system_app:s0 tcontext=u:object_r:apk_data_file:s0 tclass=dir permissive=0
[   51.484827][3 T41    ds] [dhd] dhd_prot_flow_ring_create: Send Flow Create Req flow ID 59 for peer 2e:b8:ed:6c:ce:1f prio 1 ifindex 0 items 512
[   51.486197][0 T58    .s] [dhd] dhd_prot_flow_ring_create_response_process: Flow Create Response status = 0 Flow 59
[   60.881889][1 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   60.882358][1 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   63.181818][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   63.182286][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   63.185561][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   63.186027][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   63.187337][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   63.188304][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
```

---

## 19. Post-Boot Application Execution, Package List Monitoring, App SELinux Security Denials for Philo/Tubi/Netflix, Binder IPC Teardown, and Periodic Log Daemon Socket Polling (Timestamps `68.366s` to `128.868s`)

Between timestamps `68.366s` and `128.868s`, execution tracks post-boot app initialization, monitors installed app package UIDs, logs SELinux sandbox security denials for third-party streaming apps (Philo, Tubi, Netflix), handles transient Binder IPC transaction errors, and processes periodic log daemon socket polling:

### **19.1 Raw Serial Log Trace (Timestamps `68.366s` to `128.868s`)**

```text
[   68.366617][1 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[   79.481600][2 T70    ..] binder: undelivered transaction 97903, process died.
[   79.481823][2 T70    ..] binder: undelivered transaction 97923, process died.
[   79.637658][0 T1128  ..] binder: 1128:1128 transaction failed 29189/-22, size 140-8 line 3270
[   80.026393][1 T2130  ..] binder: 2114:2130 ioctl c0306201 a49c60b8 returned -14
[   81.500270][1 T236   ..] type=1400 audit(1790585033.702:20): avc:  denied  { read } for  comm="pool-14-thread-" name="version" dev="proc" ino=4026532006 scontext=u:r:untrusted_app:s0:c132,c256,c512,c768 tcontext=u:object_r:proc_version:s0 tclass=file permissive=0 app=com.philo.philo.google
[   81.702421][1 T4247  ..] logd: start watching /data/system/packages.list ...
[   82.117693][0 T4247  ..] logd: ReadPackageList, total packages: 145
[   85.193235][1 T236   ..] type=1400 audit(1790585037.654:21): avc:  denied  { read } for  comm="pool-21-thread-" name="version" dev="proc" ino=4026532006 scontext=u:r:untrusted_app:s0:c97,c256,c512,c768 tcontext=u:object_r:proc_version:s0 tclass=file permissive=0 app=com.tubitv.ott_bridgewater
[   87.240147][0 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[   90.445972][1 T236   ..] type=1400 audit(1790585042.906:22): avc:  denied  { open } for  comm="NBP_POOL0" path="/proc/loadavg" dev="proc" ino=4026532002 scontext=u:r:priv_app:s0:c512,c768 tcontext=u:object_r:proc_loadavg:s0 tclass=file permissive=0 app=com.netflix.ninja
[   91.806795][1 T236   ..] type=1400 audit(1790585044.266:23): avc:  denied  { open } for  comm="NBP_POOL0" path="/proc/loadavg" dev="proc" ino=4026532002 scontext=u:r:priv_app:s0:c512,c768 tcontext=u:object_r:proc_loadavg:s0 tclass=file permissive=0 app=com.netflix.ninja
[   93.829074][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   93.829543][2 T0     d.] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #08!!!
[   96.740209][0 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[  102.362434][1 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[  107.780537][0 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[  113.053034][1 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[  118.312284][1 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[  123.603459][1 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
[  128.868640][1 T231   ..] logd: logdr: UID=1000 GID=1000 PID=3009 n tail=0 logMask=4 pid=0 start=0ns deadline=0ns
```

---

### **19.2 Line-by-Line Technical Analysis**

1. **Log Reader Socket Connections (`logd: logdr`) (`68.366s`, `87.240s`, `96.740s` - `128.868s`)**:
   * Process `PID 3009` running under System UID `1000` (system server or telemetry diagnostic service) connects to the `/dev/socket/logdr` socket every **~5.3 seconds** to read system logcat streams (`logMask=4`).

2. **Binder IPC Teardown & Buffer Error (`79.481s` - `80.026s`)**:
   * A dying background process leaves undelivered transactions `97903` and `97923`. The kernel binder driver automatically reclaims these allocations.
   * Binder ioctl `c0306201` (`BINDER_WRITE_READ`) returns `-EFAULT` (`-14`), indicating an invalid memory buffer pointer passed during process exit.

3. **App Package List Watcher (`81.702s` - `82.117s`)**:
   * `logd` begins watching `/data/system/packages.list`, identifying **145 total installed app packages** to map UIDs to app package names.

4. **App SELinux Sandbox Security Denials (`81.500s`, `85.193s`, `90.445s` - `91.806s`)**:
   * **Philo TV (`com.philo.philo.google`) & Tubi TV (`com.tubitv.ott_bridgewater`)**: Attempt to read `/proc/version` (`proc_version`). SELinux blocks `untrusted_app` domains from inspecting kernel versions to prevent exploit targeting.
   * **Netflix TV Client (`com.netflix.ninja`)**: Attempts to read `/proc/loadavg` (`proc_loadavg`) via worker thread `NBP_POOL0`. SELinux blocks `priv_app` domains from querying CPU load average to prevent side-channel timing attacks.

5. **Dynamic Tick Idle Warning (`93.829s`)**:
   * CPU 2 logs a non-fatal tickless idle warning (`NOHZ tick-stop error`) while timer softirq work (`handler #08`) remains queued.
