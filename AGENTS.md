# Agent instructions

You are helping with **bring-up**, not a product. Read `status.json` and `README.md` before editing or proposing commands.

## Truth

- Gates in `status.json` win over chat memory.
- **PASS** requires a dated log or hash in this repo or the lab `evidence/` tree. Do not mark G2b/G5/G6 PASS because a banner looked pretty.
- QEMU, UTM, VKMS, leftover iBoot framebuffer, Hyprland-on-fake-display → not PASS.

## Hard bans

- Never `kmutil configure-boot` on **Macintosh HD**.
- Never bless the **2.5 GB stub** as default startup disk (yellow screen; see `docs/step2-failure-matrix.md`).
- Never tap **Recovery / Reinstall macOS** on Apple’s yellow dialog. Path out: Startup Disk → Macintosh HD.
- Never `pmgr_reset` on `DISPEXT*` / `DISP_CPU` without a trace. Mini SError is documented.
- Never USB-reset gadget `0x1209:0x316d` to “unstick” a ghost ACM.
- Never `mkfs` / G9 without an explicit human yes after G5 PASS.
- Never bump macOS past the freeze in `status.json`.
- Never put secrets (passwords, `.env`, USB serials, LAN IPs) in git.

## How to work

1. One physical boot = one hypothesis = one log line.
2. Prefer Asahi trees (`m1n1`, `AsahiLinux/linux` asahi branch) over inventing MMIO.
3. Dump ADT before guessing hardware. Air panel is `dcp`+`disp0`, HID is MTP/dockchannel, not the mini’s `dcpext0`.
4. M4 idle: `idle=nop` or the kernel looks hung in WFI.
5. Stub 1TR on 26.5.1 is a **dead path** until proven otherwise. G1 here was **AsahiHost** (full second macOS).
6. Do not open Asahi PRs that enable `j715ap` without a supported story. See `docs/asahi-pr.md`.

## Where the live lab lives

This GitHub repo is the public-ready **notebook**. Kernels, HV scripts, and USB automation live on the studio checkout `m4-air-linux` and are not all mirrored here on purpose (size + secrets).
