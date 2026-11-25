# 🌴 Leonida-Recomp (Project VIctory)

<div align="center">

![Status](https://img.shields.io/badge/Status-Pre--Alpha_(2--5_FPS)-orange?style=for-the-badge)
![Target](https://img.shields.io/badge/Target-Xbox_Series_X%7CS_Executable-107C10?style=for-the-badge&logo=xbox)
![Backend](https://img.shields.io/badge/Graphics-DX12_%7C_Vulkan_1.4-blue?style=for-the-badge)
![Ray Tracing](https://img.shields.io/badge/RTGI-Partial_Support-purple?style=for-the-badge)
![DMCA](https://img.shields.io/badge/DMCA_Status-Pending_Takedown-red?style=for-the-badge)

**An experimental static recompilation, shader translation, and OS-shim layer for running the Grand Theft Auto VI console binary natively on x86_64 PCs.**

[Architecture](#-architecture-overview) •
[Progress](#-compatibility--feature-matrix) •
[Benchmarks](#-hardware-benchmarks) •
[Build Guide](#-building-from-source) •
[FAQ](#-frequently-asked-questions) •
[Legal](#-legal-disclaimer)

</div>

---

## ⚠️ Important Notice (November 25, 2026)

Following the November 24 DMCA wave that disabled the original `Leonida` repository and its 3,007 network forks, **active C++/Rust recompilation development has migrated to private decentralized git mirrors.** 

This public repository serves as the **official documentation hub, compatibility tracker, and build specification reference** for independent researchers working on Xbox Series static recompilation and RAGE9 runtime shimming.

> **Note:** This repository contains **zero copyrighted assets, zero game binaries, zero decryption keys, and zero Rockstar Games source code.** In fact, following yesterday's takedowns, it doesn't even contain any `src/` files—only documentation. You must supply your own legally acquired console dump to use any compatible local toolchain.

---

## 📖 Overview

**Leonida-Recomp** is not an emulator in the traditional sense. Rather than emulating the full Xbox Series hypervisor and custom AMD RDNA architecture instruction-by-instruction at runtime, this project documents the **Ahead-of-Time (AOT) Static Recompilation** pipeline used to lift the leaked Xbox Series executable (`Leonida_Retail_XSX.xex` / PE image) into native PC x86_64 object code.

With less than 48 hours since initial executable analysis began, early private builds have already moved far beyond spinning logos and black screens:
* **Full texture streaming and character model skinning** are operational.
* **Ray-Traced Global Illumination (RTGI)** and reflections translate to native DXR / Vulkan Ray Query pipelines.
* **Lucia's Prologue** is fully completable from start to finish.
* **Jason's first mission (*"Gone Fishing"*)** successfully loads into interactive gameplay.
* Current frame pacing sits around a cinematic **1 to 5 FPS** depending on scene geometry, shader compilation stalls, and VRAM pressure.

---

## 📊 Compatibility & Feature Matrix

Tracked in coordination with community benchmarks from `PortWatchVI`:

| Subsystem | Status | Implementation Details | Known Quirks |
| :--- | :---: | :--- | :--- |
| **Executable Lifter (`LIFT-VI`)** | 🟢 Working | Static x86_64-v3 lifting + TLS register remapping | Requires 64GB pagefile during initial link |
| **Memory & Virtual Allocator** | 🟢 Working | Custom unified GDDR6-to-System-RAM/VRAM bridge | Occasional OOM crash on 16GB RAM systems |
| **Shader Translation** | 🟡 Partial | Xbox DXIL bytecode -> SPIR-V / Native DX12 PSOs | 15-minute shader stutter on Vice Beach |
| **Textures & Streaming** | 🟢 Working | Hardware BC7/BCn pass-through | Some palm trees render as neon magenta cubes |
| **Ray-Traced Lighting (RTGI)** | 🟡 Partial | BVH construction hooked to `VK_KHR_ray_tracing_pipeline` | Massive CPU overhead (~1.2 FPS penalty) |
| **Physics (RAGE9 / Euphoria)** | 🟡 Partial | Fixed-timestep hooked to high-precision query timer | Water buoyancy explodes above 4 FPS |
| **Input Subsystem** | 🟢 Working | XInput / SDL3 Gamepad mapping (Keyboard WIP) | Aiming sensitivity tied to frame delta |
| **Audio DSP / Radio** | 🔴 Stubbed | XMA2 decoder stubbed to prevent thread deadlocks | Airboats emit a continuous 440Hz sine wave |
| **DLSS 5 / Frame Gen** | 🟣 Experimental | External hook via Neural Rendering / Apex FG | HUD flickers into alternate dimensions |

### Mission Progression Tracker

* [x] **Boot & Main Menu** — Renders at ~12 FPS (background waves occasionally T-pose).
* [x] **Prologue (Lucia)** — Fully playable start-to-finish at 2–3 FPS. Cutscenes desync audio by ~14 seconds.
* [x] **Chapter 1: *"Gone Fishing"* (Jason)** — Playable up to the boat dock sequence.
* [ ] **Open World Free Roam (Downtown Vice City)** — Crashes during high-speed vehicle streaming due to unimplemented decompression thread #7.

---

## 🖥️ Hardware Benchmarks

Because the translation layer currently lacks asynchronous PSO compilation and performs redundant CPU-to-GPU memory copies every frame, **performance is heavily CPU single-thread and memory-bandwidth bound.**

| CPU | GPU | RAM | Resolution | Settings | Avg FPS | 1% Low | Notes |
| :--- | :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| Ryzen 7 7800X3D | GeForce RTX 5070 Ti | 32GB DDR5-6000 | 1440p | Console Parity + RT | **2.4 FPS** | 0.8 FPS | Verified prologue walkthrough |
| Ryzen 9 9950X3D | GeForce RTX 5090 | 64GB DDR5-6400 | 4K | Console Parity + RT | **4.9 FPS** | 1.9 FPS | GPU utilization sits at 14% |
| Core Ultra 9 285K | Radeon RX 9070 XT | 32GB DDR5-6400 | 1080p | RT Disabled | **3.6 FPS** | 1.1 FPS | Vulkan SPIR-V backend |
| Ryzen Z2 Extreme | Integrated RDNA | 24GB LPDDR5X | 720p | Potato | **0.4 FPS** | 0.1 FPS | Basically a PowerPoint presentation |

> **Pro Tip:** Enabling **6x Apex Frame Generation** on a 3 FPS base framerate technically yields 18 FPS, though input latency reaches approximately 1.4 seconds and NPCs begin predicting your inputs before you press them.

---

## 🏗️ Architecture Overview

The recompilation pipeline separates static binary translation from runtime OS/Hardware shimming:

```text
+-----------------------------------------------------------------------+
|                    Legally Dumped Console Assets                      |
|               (User-Provided Executable & RPF Archives)               |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|                     1. Static Recompiler (LeonidaLift)                |
|   • Parses Xbox Series PE/XEX sections & unwind metadata              |
|   • Remaps console-specific gs/fs segment registers                   |
|   • Emits intermediate LLVM 21 IR -> Native Windows/Linux x86_64      |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|                     2. RAGE9 & Xbox OS Shim Layer                     |
|   • Intercepts WinRT / Xbox Era kernel syscalls                       |
|   • Translates unified memory barriers to host explicit barriers      |
|   • Hooks RAGE9 job scheduler to host std::jthread pool               |
+-----------------------------------+-----------------------------------+
                                    |
              +---------------------+---------------------+
              |                                           |
              v                                           v
+---------------------------+               +---------------------------+
|   Graphics Bridge (DX12)  |               |  Graphics Bridge (Vulkan) |
| • Direct DXIL pass-through|               | • DXIL -> SPIR-V (vkd3d)  |
| • DXR 1.2 Ray Tracing     |               | • KHR Ray Query RTGI      |
+-------------+-------------+               +-------------+-------------+
              |                                           |
              +---------------------+---------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|                Your PC Rendering Lucia at 2.4 FPS                     |
+-----------------------------------------------------------------------+
```

---

## 🛠️ Building from Source

> **⚠️ Reminder:** Per the November 24 DMCA notice, the `src/` directory is **not hosted on GitHub**. The instructions below apply only if you are maintaining your own clean-room local fork or testing compatible toolchain modules.

### Prerequisites
* **OS:** Windows 11 (25H2+) or Linux (Kernel 6.12+ with `ntsync` enabled)
* **Compiler:** Clang 20+ / MSVC 19.44+ with full C++23 support
* **Build System:** CMake 4.0+ & Ninja
* **SDKs:**
  * DirectX 12 Agility SDK (1.715+) or Vulkan SDK 1.4.x
  * LLVM 21 Development Headers (for static lifting pass)
* **Storage:** At least **450 GB free NVMe space** (180 GB for assets + 270 GB for uncompressed runtime shader caches and debug symbols)

### Build Commands (Reference)

```bash
# 1. Clone the documentation reference
git clone https://github.com/YOUR_USERNAME/Leonida-Recomp.git
cd Leonida-Recomp

# 2. Verify repository contents
ls -la
# Output: README.md (and literally nothing else)

# 3. Configure CMake (Requires your local private src/ tree)
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DENABLE_DX12_BACKEND=ON \
  -DENABLE_EXPERIMENTAL_RTGI=ON \
  -DTARGET_FPS_CAP=5

# 4. Compile
cmake --build build --parallel $(nproc)
```

### Runtime Configuration (`leonida.toml`)

If your local build reaches initialization, place a `leonida.toml` configuration file next to your compiled binary:

```toml
[graphics]
backend = "dx12"                # Options: "dx12", "vulkan"
ray_tracing = true              # Disable if you want +1.5 FPS boost
resolution_scale = 0.50         # Recommended for sub-5090 GPUs
async_shader_compile = false    # Currently causes instant heap corruption

[system]
memory_allocation_gb = 28       # Minimum 24GB recommended
unlock_framerate = false        # Do NOT unlock; physics engine breaks above 5 FPS
log_syscalls = true             # Useful for debugging "Gone Fishing" crashes

[debug]
skip_intro_logos = true
wireframe_alligators = false
```

---

## 🐛 Known Issues & Troubleshooting

<details>
<summary><b>1. The game runs at 2 FPS on my brand-new RTX 5070 Ti!</b></summary>

This is expected behavior right now. We are currently performing synchronous CPU-side command list validation and unoptimized DXIL-to-SPIR-V translation on every draw call. Think of it less as "lag" and more as "high-resolution interactive concept art."
</details>

<details>
<summary><b>2. My SSD lost 300 GB of free space while compiling shaders.</b></summary>

Unlike recent GPU driver bugs that caused excessive SSD writes, this one is actually our fault. Our temporary pipeline cache dumps uncompressed intermediate LLVM IR to disk on every frame boundary. Do not run this on a QLC drive you care about.
</details>

<details>
<summary><b>3. Why is the repository empty except for README.md?</b></summary>

Because Take-Two Interactive's legal automation took down 3,007 repositories yesterday, and we wanted to see if a completely empty repository containing zero code, zero assets, and one well-formatted Markdown file still triggers a DMCA takedown notice.
</details>

<details>
<summary><b>4. Characters freeze in Jason's "Gone Fishing" mission.</b></summary>

The water surface buoyancy solver attempts to divide by zero when frame delta exceeds `333ms` (i.e., below 3 FPS). You can temporarily bypass this by staring directly at the floorboards of the boat until your framerate rises to 4 FPS.
</details>

---

## 🗺️ Roadmap

* **Phase 1: Proof of Concept** *(Completed)*
  * [x] Decrypt & map Xbox Series executable entry points.
  * [x] Boot to title screen without immediate kernel panic.
  * [x] Complete Lucia's prologue at 1–3 FPS.
* **Phase 2: Playability & Optimization** *(In Progress - Private Branches)*
  * [ ] Fix RAGE9 job scheduler thread contention (Target: **10 FPS**).
  * [ ] Implement asynchronous PSO caching (eliminate 10-second freezes when turning the camera).
  * [ ] Fix XMA2 audio decoding so Vice City radio stations work.
  * [ ] Complete *"Gone Fishing"* without falling through the ocean.
* **Phase 3: Release Parity** *(TBD)*
  * [ ] Achieve stable 30 FPS on midrange hardware.
  * [ ] Native Ultrawide (21:9 / 32:9) HUD scaling.
  * [ ] Survive long enough for Rockstar to officially announce the actual PC version.

---

## ❓ Frequently Asked Questions

**Q: Can I download the compiled `.exe` or leaked Xbox files here?**  
**A:** **No.** Sharing copyrighted game files, executables, or assets is strictly prohibited. This repository contains only a single `README.md` file. Do not ask for links in Issues or Pull Requests.

**Q: Why didn't Rockstar just launch GTA VI on PC day one?**  
**A:** Good question. Ask Take-Two.

**Q: Where did all the active developers go?**  
**A:** Following the November 24 GitHub DMCA purge, groups have moved into private Discord channels, self-hosted Forgejo instances, and invite-only tracking groups monitored by `PortWatchVI`.

**Q: Will Take-Two DMCA a repository that literally only has a README file?**  
**A:** Let's find out together. Place your bets in the Issues tab.

---

## ⚖️ Legal Disclaimer

This repository (`Leonida-Recomp`) is an **educational documentation and parody project**. 

1. **No Copyrighted Material:** This repository hosts **no** source code, binaries, shaders, textures, audio, decryption keys, or proprietary data belonging to Rockstar Games, Take-Two Interactive, or Microsoft Corporation.
2. **No Circumvention:** This repository provides no tools capable of bypassing DRM, decrypting console packages, or executing any software whatsoever. Literally the only file in the git tree is `README.md`.
3. **Trademarks:** *Grand Theft Auto*, *Grand Theft Auto VI*, *Rockstar Games*, *RAGE*, and *Xbox* are registered trademarks of their respective owners. Their mention here is purely descriptive/editorial.

If you are an automated DMCA bot or legal representative reading this: congratulations, you have successfully scanned a plain-text Markdown file. Please tell your supervisor that there is nothing here to take down.

---

## 🙏 Acknowledgements

* **PortWatchVI** for tracking all 35 community recompilation efforts in real time.
* **The 3,007 fallen forks of `Leonida`** — gone, not forgotten.
* **RTX 5070 Ti owners** bravely sacrificing their SSDs for 2.4 FPS gameplay footage.
```

---
