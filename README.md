# Blueprint: a Mavericks-only Apple Silicon Mac

Don't port Mavericks to the hardware; make the hardware impersonate a 2013 Intel Mac. A hidden layer owns every real device and translates x86-64 to ARM64 underneath an OS that never knows, the way Transmeta's Crusoe impersonated an x86 chip.

Two facts make this viable:

- **Headroom.** An M-series core is roughly 2–4x a 2013 Haswell laptop core, so a translator at 30–50% of native already matches the machines Mavericks shipped on. Rosetta 2 reaches about 70–80% on user code.
- **A frozen target.** 10.9.5 will never change, so every private interface you reverse-engineer stays valid, and the whole OS can be translated ahead of time, once.

```
┌──────────────────────────────────────────────────────────────┐
│ Mavericks userland: stock x86-64 binaries                    │
│   + new GL driver bundle and 2D accel plug-in                │
├──────────────────────────────────────────────────────────────┤
│ XNU 2422 (x86-64): stock kexts + new Mav* kexts              │
│   later rebuilt with a paravirtual machine layer             │
╞══════════════ fictional 2013 Intel Mac ══════════════════════╡
│ Translating VMM: x86-64→ARM64 code morphing, shadow MMU,     │
│   virtual chipset / ACPI / SMC / xHCI, device backends       │
├──────────────────────────────────────────────────────────────┤
│ Minimal Linux + KVM with Asahi drivers (GPU, display, NVMe…) │
├──────────────────────────────────────────────────────────────┤
│ m1n1 ← iBoot ← SecureROM (Apple firmware, never an OS)       │
└──────────────────────────────────────────────────────────────┘
```

## 1. Host and boot

**What stays on disk.** iBoot, recoveryOS and a ~2.5 GB stub container are the machine's firmware and supply the coprocessor firmware blobs. Treat them as the BIOS; no macOS userland ever runs.

**Host choice.** I'd use minimal Linux with KVM rather than a from-scratch type-1 hypervisor. Asahi already drives the GPU, display coprocessor, NVMe, Wi-Fi, audio, input and SMC, and with one process it is functionally the hypervisor. Target an M1 or M2 machine; Asahi coverage beyond that was incomplete as far as I know, so check before choosing.

**Boot chain.**
- From recovery, register m1n1 once as the custom boot object under Permissive Security.
- Append the Linux kernel and initramfs directly to m1n1 as payloads, skipping U-Boot and GRUB.
- Build a monolithic kernel with only this model's drivers and a read-only root.
- Run the VMM as PID 1, with only iwd, PipeWire, speakersafetyd and Mesa beside it.

**Instant-on.**
- Never cold-boot the guest: the VMM maps a saved guest-RAM snapshot and faults it in lazily, so the desktop appears a second or two after the host kernel.
- Lid close puts the guest into S3, then the host into s2idle.
- The only fixed cost is iBoot's few seconds.
- m1n1 repaints iBoot's logo as the grey Mavericks boot screen and holds that framebuffer until the guest's first frame, so no Linux is ever visible.

## 2. Hardware translation

Don't translate legacy I/O Kit calls to Apple Silicon hardware at all. Cut each device class at its narrowest stable interface, preferring:

1. Emulating a real 2013 device at the bus-protocol level, so Apple's stock driver works.
2. Writing a small new x86 kext against the 10.9 SDK, backed by a shared-memory channel to the host.
3. For the GPU only, also replacing the user-mode driver plug-ins.

| Device | What Mavericks sees | New guest code | Host backend |
|---|---|---|---|
| Keyboard, trackpad | 2013 Apple internal USB keyboard and multitouch trackpad | None | Raw touch frames re-encoded into that trackpad's USB packet format (documented by Linux's `bcm5974` driver) |
| External USB | Intel 8-series xHCI | None | Request-level USB passthrough |
| Bluetooth | Apple USB HCI module | None | Raw HCI from the real Broadcom controller |
| Camera | UVC "FaceTime HD" | None | Asahi ISP driver via V4L2 |
| Storage | Paravirtual disk | `MavDisk.kext` | Raw HFS+ partition on NVMe, TRIM passed through |
| Wi-Fi | AirPort card | `MavAirPort.kext`, an `IO80211Controller` subclass (private family; AirportItlwm shows it can be done) | Scans and joins proxied to iwd |
| Audio | Internal speakers and mic | `MavAudio.kext` on the open IOAudioFamily | PipeWire, then Asahi's DSP and speakersafetyd (speaker protection is software, so never bypass it) |
| SMC, battery, lid, sensors | AppleSMC plus ACPI tables you author | None | Host SMC and power drivers; charging and thermals stay host-side |

The virtual SMC also has to answer the key query Mavericks' protected-binary kext makes at boot; QEMU's `isa-applesmc` models this.

**Graphics.** Every accelerated consumer in Mavericks (Quartz Compositor, Core Animation, Core Image) is an OpenGL client, since Metal does not exist yet. A vendor supplies three pieces, and you implement them once:

- **`MavGPU.kext`**: an `IOFramebuffer` (modes, cursor, vblank, brightness, the real panel's EDID) plus an `IOAccelerator` subclass. Advertise a HiDPI mode that ends below the notch.
- **GL driver bundle and 2D plug-in**: these implement the private plug-in interface that OpenGL.framework loads. Instead of hardware command buffers, they write a compact GL command stream into a shared ring.
- **Host replayer**: it feeds Mesa's AGX driver, which is OpenGL 4.6 conformant, a superset of the 4.1 Mavericks can request.

Unified memory helps here. Each window's backing store is an IOSurface in guest RAM, which is host RAM the GPU can map directly. Allocate surfaces 16K-aligned, import them as textures without copying, and put the composited result on a display plane for direct scanout.

The VMsvga2 project shipped this same kext-plus-plug-ins triple for VMware's virtual GPU in the Snow Leopard era. Start with an unaccelerated framebuffer, since WindowServer falls back to software compositing, then layer GL on.

## 3. CPU translation

Translation must be system-level, because the kernel and kexts are x86 too. That is normally slow (QEMU's emulation manages roughly 10–20% of native) because every memory access goes through a software TLB. The fixes, in order of payoff:

- **Hardware MMU for guest memory.** Run the translator inside a KVM guest and use real ARM page tables as shadows of the guest's x86 tables. x86-64 paging and ARM64's 4K granule have identical 9-9-9-9-12 geometry and the same 2 MB and 1 GB large pages, and Apple cores support 4K pages in guests. Stage-2 protection catches self-modifying code and traps MMIO. The Captive research hypervisor demonstrated this technique.
- **Apple's Rosetta hardware.** Use TSO mode for x86 memory ordering without barriers, the flag-manipulation instructions plus an undocumented parity/adjust flag mode, and the alternate floating-point mode for SSE semantics.
- **Static register mapping.** The 16 general and 16 XMM registers map one-to-one onto ARM registers with spares left over. Pair translated CALL/RET with real BL/RET so the return predictor stays accurate.
- **A modest virtual CPU.** Advertise a Westmere-class CPUID (SSE4.2, AES-NI, no AVX). Mavericks supports pre-AVX Macs, so no 256-bit vectors need translating.
- **Tiering and a persistent cache.** A quick block translator runs first, and an optimizing trace compiler runs on the E-cores. The cache is keyed by code-page content hash, and the kernel, kexts, dyld shared cache and bundled apps are pre-translated at install time.
- **Paravirtual kernel.** XNU 2422's source is public. Rebuild it with the `osfmk/i386` machine layer talking to the VMM: page-table updates and TLB shootdowns become hypercalls, interrupt masking a memory flag, idle a hypercall. The kext interface is unchanged, so stock kexts still load.
- **Native islands.** Intercept entry to known hot functions (memcpy, corecrypto, Accelerate, zlib, the whole GL driver) and run ARM64 implementations, as Box64 and FEX do.

**Battery.** A guest HLT blocks the vCPU thread, so an idle Mavericks is an idle host. Mavericks' timer coalescing and App Nap cut wakeups, and vCPUs float across P and E cores under the host scheduler. Translation costs perhaps 1.3–2x energy per unit of work (my estimate), on a workload sized for 2013.

## 4. Execution roadmap

| Phase | Build | Done when |
|---|---|---|
| 0. Proof of life | Asahi plus stock QEMU (q35, Penryn CPU, e1000, applesmc, OpenCore) | The desktop appears, slow and unaccelerated |
| 1. Appliance | m1n1 direct boot, VMM as PID 1, raw partition, snapshot resume, sleep | Power button leads to Mavericks with no visible Linux |
| 2. Translating VMM | Shadow-MMU translator, tiering, persistent cache, whole-OS pre-translation | It beats a 2013 MacBook Air on period benchmarks |
| 3. Device kexts | Disk, audio, AirPort, virtual trackpad, Bluetooth, camera, authored ACPI | Every menu-bar item and preference pane works |
| 4. Graphics | Framebuffer, then accelerator and 2D plug-in, then GL bundle, then zero-copy scanout | Translucency and Mission Control run at 60 fps |
| 5. Paravirtual kernel | Rebuilt XNU, native islands | Compile and file-I/O workloads land within 2x of native |
| 6. Power | Idle tuning, s2idle, thermals, external displays | All-day battery under light use |

Run phases 2–4 as parallel teams once phase 1 exists. Build guest kexts with Xcode 5 inside Mavericks itself. From day one, the translator team needs a differential-testing oracle: the same guest execution replayed on real Intel hardware, with architectural state compared.

## Endgame: move the boundary up

With truly unlimited effort, move the ISA boundary from below the kernel to the syscall line, which is Rosetta's architecture:

- Port XNU 2422 to ARM64, using modern open-source XNU's arm64 layer as the reference.
- Translate only userland, at user-mode speeds.
- Reimplement the closed x86 kexts, or host them as translated islands behind an ARM64EC-style ABI with x86-compatible layouts and thunks at the kernel interface.
- Rewrite the Asahi drivers as native I/O Kit drivers, at which point Linux disappears and Mavericks is native.

## Where the risk is

- **The private GL plug-in interface.** This is the largest reverse-engineering item, though it only has to be done once.
- **System-level translator correctness.** Precise faults, self-modifying code and kernel-mode code are where it breaks, which is why the oracle matters.
- **Host power management.** Linux idle and sleep power on this hardware, not translation overhead, will decide whether battery life is viable.
