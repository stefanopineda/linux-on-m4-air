# Agent instructions

You are helping with **bring-up**, not a product. Read `status.json` and `README.md` before editing or proposing commands. G2b is PASS (copy_process → kernel_init → /init, 2026-09-10); the wall past G2b is the fbcon NULL deref (fixed with CONFIG_FB_SIMPLE=n) and the G5 AIC IPI storm. Pre-copy_process wall in [docs/g2b.md](docs/g2b.md). Past copy_process in [docs/what-came-past-copy-process.md](docs/what-came-past-copy-process.md). Pivot: [docs/ghidra-pivot.md](docs/ghidra-pivot.md). Catch-up path: [docs/getting-started.md](docs/getting-started.md).

## Truth

- Gates in `status.json` win over chat memory.
- **PASS** requires a dated log or hash. Do not mark G2b/G5/G6 PASS because a banner looked pretty.
- QEMU, UTM, VKMS, leftover iBoot framebuffer, Hyprland-on-fake-display → not PASS.
- @wtsnz having AGX on an M4 Max does not move this Air’s G2b.

## Hard bans

- Never `kmutil configure-boot` on **Macintosh HD**.
- Never bless the **2.5 GB stub** as default startup disk (yellow screen; see `docs/step2-failure-matrix.md`).
- Never tap **Recovery / Reinstall macOS** on Apple’s yellow dialog. Path out: Startup Disk → Macintosh HD.
- Never `pmgr_reset` on `DISPEXT*` / `DISP_CPU` without a trace. Mini SError is documented.
- Never USB-reset gadget `0x1209:0x316d` to “unstick” a ghost ACM.
- Never `mkfs` / G9 without an explicit human yes after G5 PASS.
- Never bump macOS past the freeze in `status.json`.
- Never put secrets (passwords, `.env`, USB serials, LAN IPs, volume UUIDs) in git.
- Never skip `copy_process` or `current->nsproxy`. Never skip `numa_default_policy` (tested; did not unblock).
- Never `msr VM_TMR_FIQ_ENA_EL2` (or `VM_TMR_LR`) on t8132 from m1n1.

## How to work

1. One physical boot = one hypothesis = one log line. Unique Image hash per WDT site. Never re-boot a hash in `booted_images`.
2. Prefer Asahi trees (`m1n1`, `AsahiLinux/linux` asahi branch) over inventing MMIO.
3. Dump ADT before guessing hardware. Air panel is `dcp`+`disp0`, HID is MTP/dockchannel, not the mini’s `dcpext0`.
4. M4 idle: `idle=nop` or the kernel looks hung in WFI.
5. Stub 1TR on 26.5.1 is a **dead path**. G1 here was **AsahiHost** (full second macOS).
6. Do not open Asahi PRs that enable `j715ap` without a supported story. See `docs/asahi-pr.md`.
7. After a C change in `copy_process` / `kernel_clone`, reconfirm an earlier 19s site. The hang moves.
8. HV experiments need a **Ghidra cite** ([docs/ghidra-pivot.md](docs/ghidra-pivot.md)); no folklore `wdt_site`. Do not `msr VM_TMR_FIQ_ENA_EL2` on t8132 (EL2h-SYNC). IMP-MSR of ENA/LR is banned.

## Where the live lab lives

This GitHub repo is the public-ready **notebook**. Kernels, HV scripts, and USB automation live on a studio checkout and are not all mirrored here on purpose (size + secrets).
