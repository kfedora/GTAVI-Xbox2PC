
<div align="center">

```text
   ______ ______ ___   _    __ ____       _  __ __                ___    ____  ______
  / ____//_  __//   | | |  / //  _/      | |/ // /_  ____  _  __ |__ \  / __ \/ ____/
 / / __   / /  / /| | | | / / / / ______ |   // __ \/ __ \| |/_/ __/ / / /_/ / /     
/ /_/ /  / /  / ___ | | |/ /_/ / /_____//   |/ /_/ / /_/ />  <  / __/ / ____/ /___   
\____/  /_/  /_/  |_| |___//___/       /_/|_|_.___/\____/_/|_| /____//_/    \____/   
                                                                                     
          XBOX SERIES X|S STATIC RECOMPILATION & RAGE9 SHIM SPECIFICATION
```

# GTAVI-Xbox2PC (Project VIctory / Leonida Heritage)

[![Build Status](https://img.shields.io/badge/Runtime_Performance-2.4_FPS-orange?style=for-the-badge)](https://github.com/kfedora/GTAVI-Xbox2PC)
[![Target Binary](https://img.shields.io/badge/Target-Xbox_Series_X%7CS_(XSX)-107C10?style=for-the-badge&logo=xbox)](https://github.com/kfedora/GTAVI-Xbox2PC)
[![Graphics API](https://img.shields.io/badge/Backend-DX12_%7C_Vulkan_1.4-0078D4?style=for-the-badge)](https://github.com/kfedora/GTAVI-Xbox2PC)
[![Ray Tracing](https://img.shields.io/badge/RTGI-Hardware_Ray_Query-76B900?style=for-the-badge)](https://github.com/kfedora/GTAVI-Xbox2PC)
[![DMCA Speedrun](https://img.shields.io/badge/DMCA_Status-Any%25_Speedrun-DC143C?style=for-the-badge)](https://github.com/kfedora/GTAVI-Xbox2PC)
[![GitHub Stars](https://img.shields.io/github/stars/kfedora/GTAVI-Xbox2PC?style=for-the-badge&color=gold)](https://github.com/kfedora/GTAVI-Xbox2PC/stargazers)

**An open-specification static recompilation pipeline, DXIL-to-SPIR-V shader translator, and Xbox Era OS shim architecture for executing the Grand Theft Auto VI console binary natively on x86_64 PCs.**

[Executive Summary](#executive-summary) •
[Architecture](#system-architecture) •
[Memory Model](#virtual-memory--gddr6-bridge) •
[Flamegraph](#frame-time-profiling-why-is-it-3-fps) •
[Compatibility](#compatibility--mission-tracker) •
[Benchmarks](#hardware-benchmarks) •
[Build Spec](#build--runtime-specification) •
[Timeline](#development-chronicle-nov-22--nov-25-2026) •
[FAQ](#frequently-asked-questions) •
[Legal](#legal-disclaimer)

</div>

---

> [!CAUTION]
> **Post-DMCA Repository Notice (Updated November 25, 2026):**  
> On November 24, 2026, Take-Two Interactive issued a sweeping DMCA takedown disabling the original `Leonida` repository and its entire network of **3,007 forks**. As a result, all active C++23/Rust lifting branches and binary patches have transitioned to private, invite-only mirrors.  
>  
> **`kfedora/GTAVI-Xbox2PC` hosts ZERO copyrighted assets, ZERO console executables, ZERO decryption keys, and ZERO source code.** This public repository serves strictly as the **architectural specification, compatibility ledger, and community benchmark hub**.

---

## Executive Summary

**GTAVI-Xbox2PC** documents the technical effort to bring *Grand Theft Auto VI* to PC via **Ahead-of-Time (AOT) Static Binary Recompilation** and **Runtime Kernel/Graphics Shimming**—without waiting for an official PC announcement.

Unlike traditional hypervisor or HLE/LLE console emulators that suffer massive dynamic recompilation (JIT) overhead at runtime, this architecture statically lifts the leaked Xbox Series X|S executable (`Leonida_Retail_XSX.xex` / PE32+ container) into native Windows and Linux `x86_64-v3` object code via LLVM 21, while intercepting Xbox Era kernel syscalls and RAGE9 engine memory barriers at near-zero CPU instruction penalty.

In less than 48 hours since the Xbox executable surfaced, private builds coordinated across 35 tracked community projects (monitored by `PortWatchVI`) have achieved major breakthroughs:
* **Full Static Binary Lifting:** Native host execution of primary gameplay, UI, and RAGE9 job-scheduler threads.
* **Shader & Material Pipeline:** Real-time translation of Xbox DXIL Shader Model 6.7 bytecode into host DX12 and Vulkan 1.4 SPIR-V.
* **Texture Streaming & Geometry:** Hardware BC7/BCn texture pass-through, skeletal animation, and Euphoria physics integration.
* **Ray-Traced Global Illumination (RTGI):** Functional BVH acceleration structure translation and ray-traced lighting/reflections.
* **Interactive Gameplay Milestones:** Multiple private builds have completed **Lucia's Prologue** end-to-end and entered **Jason's opening mission (*"Gone Fishing"*)** at **1 to 5 FPS**.

---

## System Architecture

The `GTAVI-Xbox2PC` stack separates **offline binary translation** from **runtime OS/GPU virtualization**:

```text
========================================================================================
                        STAGE 1: OFFLINE STATIC RECOMPILATION
========================================================================================

  [ User-Supplied Console Dump ]
  (Legally dumped XSX PE/XEX + RPFv9 Archives — NOT INCLUDED IN THIS REPO)
                 |
                 v
  +----------------------------------------------------------------------------------+
  |  LeonidaLift (Static Binary Lifter)                                              |
  |  • Parses Xbox PE32+ sections, .pdata unwind tables, and jump tables             |
  |  • Resolves console-specific GS/FS segment register TLS offsets                  |
  |  • Replaces custom Jaguar/Zen2 cache-line prefetches & Xbox-only intrinsics      |
  |  • Emits LLVM 21 Intermediate Representation -> Links against Host CRT / ABI     |
  +----------------------------------------------------------------------------------+
                 |
                 v
  [ Native Host Executable: GTAVI_Recomp_Host.exe / gtavi-recomp-linux ]

========================================================================================
                        STAGE 2: RUNTIME OS & GPU SHIMMING
========================================================================================

  +----------------------------------------+  +--------------------------------------+
  |  Xbox Era / WinRT Kernel Shim          |  |  RAGE9 Engine Runtime Hooks          |
  |  • Maps XMemAlloc -> Host VirtualAlloc |  |  • Hooks rage::sysJobManager         |
  |  • Translates XAudio2 / XMA2 -> Stub   |  |  • Clamps Euphoria dt (max 333ms)    |
  |  • Emulates Xbox Live / Profile Auth   |  |  • Overrides HDD/SSD streaming locks |
  +--------------------+-------------------+  +-------------------+------------------+
                       |                                          |
                       +--------------------+---------------------+
                                            |
                                            v
  +----------------------------------------------------------------------------------+
  |  Graphics & Ray-Tracing Translation Bridge (DX12 / Vulkan 1.4)                   |
  |  • Intercepts DirectX 12.X Console Command Lists (Fast-Path Ring Buffer)         |
  |  • Converts Xbox Enhanced Barriers -> Standard PC VK_KHR_synchronization2        |
  |  • Reconstructs Top-Level Acceleration Structures (TLAS) for RTGI / Reflections  |
  |  • Dispatches DXIL -> SPIR-V JIT Compilation (Causing 15-minute Vice Beach stalls)|
  +----------------------------------------------------------------------------------+
                                            |
                                            v
  +----------------------------------------------------------------------------------+
  |  Host GPU (RTX 5070 Ti / RTX 5090 / RX 9070 XT)                                  |
  |  >>> OUTPUT: Lucia walking around the prologue at a majestic 2.4 FPS <<<         |
  +----------------------------------------------------------------------------------+
```

---

## Virtual Memory & GDDR6 Bridge

One of the primary engineering challenges in bringing an Xbox Series X binary to PC is the **Unified Memory Architecture (UMA)**. The console shares a single **16 GB GDDR6** pool (10 GB high-bandwidth `560 GB/s` + 6 GB lower-bandwidth `336 GB/s`, with ~13.5 GB accessible to games) directly between CPU and GPU without PCIe transfers.

On standard x86_64 PCs with discrete GPUs, `GTAVI-Xbox2PC` enforces a split-mirror virtual memory bridge:

| Console Address Range | Size | Xbox Pool | PC Host Mapping Strategy | Current Bottleneck |
| :--- | :--- | :--- | :--- | :--- |
| `0x0000_1000_0000` – `0x0002_8000_0000` | 10.0 GB | High-Speed GPU/CPU | Pinned `DEVICE_LOCAL \| HOST_VISIBLE` (ReBAR) | Exhausts 12GB/16GB VRAM cards immediately |
| `0x0002_8000_0000` – `0x0003_6000_0000` | 3.5 GB | Standard System | System DDR5 + On-Demand PCIe Staging Ring | Synchronous `vkQueueWaitIdle` stalls |
| `0x0004_0000_0000` – `0x0004_4000_0000` | 1.0 GB | OS / Hypervisor | Intercepted & Stubbed by `EraShim.dll` | Missing telemetry calls silently ignored |
| `0x7FFE_0000_0000` – `0x7FFF_FFFF_FFFF` | Dynamic | TLS & Stack | Remapped to Host `std::jthread` Stack | Thread #7 decompression stack overflow |

> [!IMPORTANT]
> **Resizable BAR (ReBAR) / Smart Access Memory is mandatory.** Disabling ReBAR in your motherboard UEFI reduces prologue performance from **2.4 FPS** down to **0.18 FPS** (roughly 1 frame every 5.5 seconds) due to constant 256MB staging aperture flushes.

---

## Frame-Time Profiling (Why Is It 3 FPS?)

People frequently ask why an **RTX 5070 Ti** paired with a **Ryzen 7 7800X3D** renders the prologue at just **1–3 FPS** while GPU utilization hovers around **18%**. Below is an actual captured `416.6 ms` frame budget (~2.4 FPS) from a private build trace:

```text
Total Frame Time: 416.6 ms (2.40 FPS)
[======================================================================] 100%

Breakdown by Subsystem:
1. [██████████████████████████████] 182.4 ms (43.8%) — DXIL -> SPIR-V / PSO Compile
2. [████████████████████          ] 124.0 ms (29.8%) — Unified Memory Barrier Flush
3. [███████████                   ]  71.2 ms (17.1%) — RTGI TLAS/BLAS CPU Rebuild
4. [████                          ]  24.5 ms ( 5.9%) — RAGE9 Spin-Lock Contention
5. [██                            ]  14.5 ms ( 3.4%) — GPU Rasterization & RT
```

**Key Takeaway:** The GPU itself finishes drawing a frame of *Grand Theft Auto VI* in **~14.5 ms** (roughly 69 FPS capable). The remaining **402.1 ms** is pure CPU-side synchronization, memory copying, and shader translation overhead. Once asynchronous pipeline caching and dirty-page tracking replace full-frame memory flushes, frame rates are projected to jump significantly.

---

## Compatibility & Mission Tracker

Data aggregated from community builds tracked by `PortWatchVI` as of **November 25, 2026**:

### Core Subsystems

| Subsystem | Status | Implementation Notes | Known Quirks / Bugs |
| :--- | :---: | :--- | :--- |
| **Static Lifter (`LeonidaLift`)** | 🟢 Operational | Clean lifting of 99.4% of `.text` segments | Remaining 0.6% falls back to interpreter |
| **Shader Translation** | 🟡 Partial | DXIL 6.7 -> DX12 Native / Vulkan SPIR-V | Neon magenta foliage on Vice Beach palm trees |
| **Texture & Mesh Streaming** | 🟢 Operational | Hardware BC7/BC5/BC4 pass-through | Streaming LODs lag behind fast cars |
| **Ray-Traced Lighting (RTGI)** | 🟡 Partial | Translates reflections, diffuse GI, and shadows | Water caustics occasionally blind the entire county |
| **Euphoria / RAGE9 Physics** | 🟡 Partial | Character ragdolls and vehicle suspension active | Stepping into airboats launches NPCs into orbit |
| **Input & Haptics** | 🟢 Operational | SDL3 / XInput / DualSense adaptive triggers | Mouse look emulates right thumbstick badly |
| **Audio Engine (XMA2 / DSP)** | 🔴 Stubbed | Dummy silence buffer prevents audio thread hang | Airboats play a continuous 440Hz test tone |
| **DLSS 5 / Frame Gen Hooks** | 🟣 Experimental | External DLL injection via Neural Rendering | 6x Apex Frame Gen turns HUD text into hieroglyphs |

### Story & Open-World Progression

| Segment / Location | Bootable? | Playable? | Avg FPS | Verified Notes |
| :--- | :---: | :---: | :---: | :--- |
| **Main Menu & Attract Screen** | ✅ Yes | ✅ Yes | 12–15 FPS | Wave simulation renders accurately; menu music silent |
| **Prologue (Lucia)** | ✅ Yes | ✅ Yes | 1.8–3.2 FPS | **100% Completable.** Lighting, textures, and combat functional |
| **Chapter 1: *"Gone Fishing"* (Jason)** | ✅ Yes | 🟡 Partial | 2.0–4.5 FPS | Reachable from prologue; water buoyancy crashes after ~45s |
| **Vice Beach Strip** | ✅ Yes | 🟡 Partial | 1.5–2.8 FPS | Heavy shader stutter; pedestrian AI spawns correctly |
| **Ocean View Hotel Interior** | ✅ Yes | ✅ Yes | 4.8–5.5 FPS | Highest frame rate region in current builds |
| **Grassrivers (Swamp / Airboats)** | ✅ Yes | 🔴 Unstable | 0.9–1.7 FPS | Alligator skeletal rigs stretch infinitely on LOD transition |
| **Downtown Vice City (Free Roam)** | 🟡 Loading | ❌ Crash | N/A | Crashes on `rpf9_inflate_chunk_0x4B` (Decompression Thread #7) |

---

## Hardware Benchmarks

All tests run on **Console Parity Visual Settings** (1440p internal resolution unless noted) with **RTGI Enabled**.

| CPU | GPU | System RAM | Backend | Avg FPS | 1% Low | Notes |
| :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| **Ryzen 7 7800X3D** | GeForce RTX 5070 Ti (16GB) | 32GB DDR5-6000 | DX12 | **2.4 FPS** | 0.8 FPS | Baseline community verification rig |
| **Ryzen 9 9950X3D** | GeForce RTX 5090 (32GB) | 64GB DDR5-6400 | DX12 | **4.9 FPS** | 2.1 FPS | 4K native; 28.4 GB total VRAM allocated |
| **Core Ultra 9 285K** | Radeon RX 9070 XT (16GB) | 32GB DDR5-6400 | Vulkan 1.4 | **3.6 FPS** | 1.3 FPS | RTGI disabled (`rt_enable = false`) |
| **Ryzen 7 9800X3D** | GeForce RTX 4090 (24GB) | 64GB DDR5-6000 | DX12 | **4.1 FPS** | 1.7 FPS | Stable through entire Lucia prologue |
| **Core Ultra 5 245K** | Intel Arc B770 (16GB) | 32GB DDR5-5600 | Vulkan 1.4 | **1.9 FPS** | 0.5 FPS | Volumetric clouds render as wireframe grids |
| **Ryzen Z2 Extreme** | Integrated RDNA (Shared) | 24GB LPDDR5X | Vulkan 1.4 | **0.4 FPS** | 0.1 FPS | 720p Ultra-Low; thermal throttles in 90 seconds |

> [!TIP]
> **Looking for 60 FPS?** Record gameplay at **3 FPS** using OBS, import the `.mkv` into FFmpeg, and run `ffmpeg -i gtavi_3fps.mkv -filter:v "setpts=0.05*PTS" gtavi_60fps.mp4`. Instant locked 60 FPS!

---

## Build & Runtime Specification

> [!WARNING]
> **Read Before Cloning:** Because this repository intentionally contains **only `README.md`**, running `cmake` inside a clone of `kfedora/GTAVI-Xbox2PC` will immediately fail unless you supply your own private clean-room `src/` directory and legally dumped console assets.

### System Prerequisites

| Dependency | Minimum Version | Purpose |
| :--- | :--- | :--- |
| **Operating System** | Windows 11 25H2 / Linux 6.12+ | Requires `ntsync` (Linux) or Native WDDM 3.3 |
| **C++ Compiler** | Clang 20.1+ or MSVC 19.44+ | Full C++23 `std::expected` & `std::jthread` support |
| **Build Toolchain** | CMake 4.0+ & Ninja 1.12+ | Parallel static-lifted object linking |
| **LLVM Toolchain** | LLVM 21.0 Development Headers | Offline Xbox XEX/PE -> x86_64 IR lifting |
| **Graphics SDK** | DX12 Agility SDK 1.715 / Vulkan 1.4 | Ray Query & Enhanced Barrier translation |
| **NVMe Storage** | **450 GB Free Space** | 180 GB game assets + 270 GB uncompressed shader/IR cache |
| **System RAM + Pagefile** | 32 GB RAM + 64 GB Swap | Linking `GTAVI_Recomp_Host` peaks at 58 GB commit |

### Reference Build Commands

```bash
# 1. Clone the documentation hub
git clone https://github.com/kfedora/GTAVI-Xbox2PC.git
cd GTAVI-Xbox2PC

# 2. Verify repository contents (spoiler: there is no copyrighted code here)
ls -la
# Output: README.md

# 3. Configure CMake (Requires your private clean-room src/ directory)
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DGTAVI_BACKEND=DX12 \
  -DENABLE_RTGI_TRANSLATION=ON \
  -DCLAMP_EUPHORIA_DELTA=ON \
  -DFIX_MAGENTA_PALMS=ON \
  -DSKIP_AUDIO_DSP_XMA2=ON

# 4. Build the static lifter and host runtime (expect ~45 minutes link time)
cmake --build build --parallel $(nproc)
```

### CMake Flag Reference

| CMake Option | Default | Description |
| :--- | :---: | :--- |
| `-DGTAVI_BACKEND` | `DX12` | Selects `DX12` or `VULKAN` translation layer |
| `-DENABLE_RTGI_TRANSLATION` | `ON` | Hooks ray-traced global illumination & reflections (~1.5 FPS cost) |
| `-DASYNC_PSO_COMPILE` | `OFF` | Experimental async shader compilation (currently causes heap corruption) |
| `-DCLAMP_EUPHORIA_DELTA` | `ON` | Clamps physics delta to 333ms so ragdolls don't invert space-time at 2 FPS |
| `-DFIX_MAGENTA_PALMS` | `ON` | Replaces missing compute-shader foliage palettes with hardcoded green |
| `-DSKIP_AUDIO_DSP_XMA2` | `ON` | Stubs Xbox audio hardware to prevent infinite semaphore deadlocks |
| `-DDUMP_UNCOMPRESSED_IR` | `OFF` | Writes 400 GB of debug shader IR to your SSD (do not enable on QLC drives) |

### Sample Runtime Configuration (`gtavi-xbox2pc.toml`)

```toml
# Place alongside your locally compiled binary (not included in this repo)

[system]
memory_bridge_gb = 28                 # Minimum 24GB recommended for ReBAR bridge
enable_rebar_fastpath = true          # Disabling drops FPS by 90%
thread_pool_override = 16             # Match physical CPU thread count
unlock_framerate = false              # DO NOT UNLOCK: Water physics implode above 5 FPS

[graphics]
api = "dx12"                          # "dx12" or "vulkan"
internal_resolution_scale = 0.50      # 0.50 recommended on RTX 5070 Ti / RX 9070 XT
ray_tracing_gi = true                 # Set false for +1.5 FPS uplift
ray_tracing_reflections = true
dlss5_neural_hook = false             # Experimental external hook; unstable on HUD
apex_framegen_multiplier = 1          # Setting to 6x creates 1.4s input lag

[debug]
skip_rockstar_intro = true
wireframe_alligators = false
log_unimplemented_syscalls = true     # Essential for debugging "Gone Fishing" crashes
```

---

## Development Chronicle (Nov 22 – Nov 25, 2026)

* **Nov 22, 2026:** First community efforts kick off to analyze Xbox Series OS calls and RAGE9 container formats ahead of executable extraction.
* **Nov 23, 2026:** The Xbox Series executable surfaces online. Static recompilation projects (`Leonida`, `VIctory`) begin mapping PE sections and entry points. First frame of the main menu renders on PC within hours.
* **Nov 24, 2026 (Morning):** First verified gameplay footage hits social media—Lucia's prologue walking around at **1 to 3 FPS** with textures, character models, and early ray-traced lighting on an RTX 5070 Ti + Ryzen 7 7800X3D.
* **Nov 24, 2026 (Evening):** Take-Two Interactive files a massive DMCA notice on GitHub, taking down the primary **`Leonida`** repository and **3,007 associated forks**. Development immediately scatters into private groups and encrypted mirrors.
* **Nov 25, 2026:** At least six private builds reach interactive gameplay—completing Lucia's prologue and loading Jason's *"Gone Fishing"* mission at **2–5 FPS**. **`kfedora/GTAVI-Xbox2PC`** is launched as a 100% asset-free, code-free public documentation hub to chronicle the architecture—and test whether automated DMCA bots read Markdown files before filing takedowns.

---

## Known Issues & Troubleshooting

<details>
<summary><b>1. My RTX 5070 Ti is only getting 2.4 FPS! Is my GPU broken?</b></summary>

No. Your GPU is barely doing anything (~18% usage). Nearly 96% of your frame time is currently spent waiting on synchronous shader translation and uncached CPU-to-GPU memory barrier copies. Treat it as a playable slideshow of the future.
</details>

<details>
<summary><b>2. Why does this repository only have README.md? Where is the download?</b></summary>

There is no download. Following Take-Two's November 24 DMCA sweep of 3,007 repositories, hosting any compiled binaries, leaked executables, or even clean-room source code publicly on GitHub results in instant takedowns. This repo exists purely as technical documentation, parody, and a honeypot to see if legal bots DMCA a single Markdown file.
</details>

<details>
<summary><b>3. Why did my SSD lose 300 GB of free space during boot?</b></summary>

If you enabled debug logging or intermediate DXIL caching, the translation pipeline writes uncompressed shader IR and pipeline state objects to disk on every draw-call cache miss. Unlike certain recent GPU driver bugs, this excessive SSD write behavior is entirely our fault.
</details>

<details>
<summary><b>4. Jason falls through the dock in "Gone Fishing" and crashes the game.</b></summary>

The RAGE9 water buoyancy solver experiences a floating-point division singularity whenever frame time exceeds `333 ms` (below 3 FPS). Look straight down at the wooden planks of the dock to cull ocean reflections until your framerate climbs above 3.5 FPS, then proceed with the cutscene.
</details>

<details>
<summary><b>5. Why are all the palm trees in Vice Beach glowing neon magenta?</b></summary>

The compute shader responsible for wind-swaying foliage vertex buffers uses an Xbox-specific `WavePrefixSum` intrinsic that hasn't been mapped in `vkd3d` yet, causing the material shader to fall back to the engine's missing-shader debug color.
</details>

---

## Contributing & DMCA Resilience

We welcome **documentation-only Pull Requests**! To keep `kfedora/GTAVI-Xbox2PC` 100% compliant with GitHub Terms of Service and copyright law:

1. **DO NOT** submit game assets, textures, `.rpf` archives, `.xex`/`.exe` binaries, or decryption keys.
2. **DO NOT** submit links to external piracy sites, Discord invite links distributing leaked files, or compiled builds.
3. **DO** submit benchmark updates, architectural diagrams, typo fixes, or verified `PortWatchVI` progression milestones via Pull Requests or Issues.

---

## Frequently Asked Questions

**Q: Is this an official Rockstar Games project?**  
**A:** No. If Rockstar wanted PC gamers playing GTA VI in November 2026, they wouldn't be busy filing 3,007 DMCA takedowns instead of announcing a PC release date.

**Q: Can I run this on Linux / SteamOS / macOS?**  
**A:** Theoretically via the Vulkan 1.4 SPIR-V backend and Linux 6.12+ `ntsync`, though macOS Metal lacks direct DXIL Ray Query parity without heavy MoltenVK shimming. Also, remember: *there is no code in this repo.*

**Q: Will Take-Two Interactive actually issue a DMCA takedown against a repository containing one `README.md` file?**  
**A:** Honestly, we have no idea—that's half the fun. If this repository disappears tomorrow, check back under a slightly sillier name.

---

## Legal Disclaimer

**`kfedora/GTAVI-Xbox2PC` is a non-commercial educational documentation and parody repository.**

1. **Zero Copyrighted Content:** This repository contains **zero** source code, **zero** compiled executables, **zero** game assets, **zero** shaders, **zero** decryption keys, and **zero** proprietary files belonging to Rockstar Games, Take-Two Interactive, or Microsoft Corporation.
2. **Zero Circumvention Tools:** Nothing hosted at `https://github.com/kfedora/GTAVI-Xbox2PC` bypasses DRM, decrypts console packages, or executes any game software. The entire git tree consists of a single plain-text Markdown document (`README.md`).
3. **Fair Use & Trademarks:** *Grand Theft Auto*, *Grand Theft Auto VI*, *Rockstar Games*, *RAGE*, and *Xbox Series X|S* are registered trademarks of their respective owners. Their use herein is strictly nominative, editorial, and satirical.

> **To Automated DMCA Scrapers & Outside Counsel:**  
> Before submitting a takedown notice under 17 U.S.C. § 512(c), please note that under § 512(f), any person who knowingly materially misrepresents that material is infringing may be subject to liability. There is literally nothing in this repository except text. Have a wonderful day!

---

## Acknowledgements & Memorial

* **PortWatchVI** for maintaining real-time tracking across 35 independent community recompilation efforts.
* **The 3,007 fallen repositories of the `Leonida` network** — deleted by DMCA on November 24, 2026, yet somehow accelerating private development even faster.
* **Brackz, onyx, and __cxc__** for proving that 2–3 FPS ray-traced GTA VI on PC is worth melting an SSD for.

