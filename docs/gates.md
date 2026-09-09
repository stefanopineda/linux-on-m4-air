# Gates

Same ladder other M4 bring-ups use. Out of order is allowed except G9 (format) and enrolling the daily macOS.

| Gate | Pass looks like |
| --- | --- |
| G0 | Chip, board, iBoot, macOS build written down. |
| G1 | m1n1 is the enrolled kernel of *some* volume that is **not** Macintosh HD. Other computer sees USB CDC proxy. Python REPL answers. |
| G2 | Guest Linux prints `Linux version …` and a **machine model** string on m1n1 HV VUART. |
| G2b | `/init` (or equivalent) runs. A shell or a known userspace print. |
| G5 | Read GPT / NVMe without format. |
| G6 | Linux owns scanout (modeset). Apple’s leftover boot framebuffer does not count. |
| G7 | Keyboard via MTP/dockchannel. |
| trackpad | Pointer device via MTP. |
| weston | Software Weston (or equivalent) on that scanout + HID. Not VKMS. |
| G4 | USB host beyond the debug gadget. |
| G8 | Wi-Fi. |
| G9 | `mkfs` on a **pre-created empty** Linux slice only, after G5 and a human yes. |
| G10 | Real AGX, not llvmpipe. Overlay/`apple-agx` + DCP swap on the panel counts. Hyprland is G11, not G10. |
| G11 | Omarchy/Hyprland on that stack. |

This Air (2026-09-08): G0–G2 **PASS**, G2b **in progress** (timer/FIQ/ENA wall; not PASS). See `status.json`, [g2b.md](g2b.md), and [ghidra-pivot.md](ghidra-pivot.md).
