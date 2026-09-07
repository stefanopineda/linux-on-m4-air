# Linux on a 15" M4 MacBook Air

**Status (2026-09-07):** this is a **lab log**, not a distro. Apple’s installer still says M4 is unsupported. We have a working **m1n1 USB debugger** on this Air and have seen a **Linux kernel banner** over a virtual UART. We do **not** have disk Linux, a usable display owned by Linux, or Omarchy.

If you dual-boot Windows and Linux on a PC, this document is the Apple Silicon version of “what actually happens at boot” — plus what we tried, what died, and what an AI coding agent should read first.

---

## Read this in 90 seconds

| You might think | What is actually true |
| --- | --- |
| Install Linux and it replaces the Mac bootloader | **You cannot replace iBoot.** Apple’s boot ROM still runs. You add a tiny extra “OS” whose *kernel* is a program called **m1n1**. |
| This is like GRUB on the EFI partition | Closer to: “install a second, fake macOS, then tell Apple’s tools that its kernel is our debugger.” |
| The USB cable boots Linux from the other Mac | **No USB boot** on Apple Silicon. The cable is a **debug probe** after m1n1 is already running on the Air. |
| Omarchy (Arch + Hyprland) is the first step | Omarchy is **userspace**. It needs a kernel, a console, and eventually a GPU. We are still in kernel bring-up. |
| QEMU / UTM / a Linux VM counts | **No.** Those are not this project. Honesty bar is below. |

**Honesty bar:** QEMU, UTM, Parallels, VKMS, leftover Apple boot logo, and “Hyprland on a fake display” do **not** count. A real step has a log, a date, and a hash.

---

## What this machine is

| Field | Value |
| --- | --- |
| Product | 15" MacBook Air (2025), `Mac16,13` |
| Apple board name | `j715ap` |
| Chip | T8132 (M4), board id `0x2E` |
| macOS we froze | **26.5.1 (25F80)** — do not “just update” |
| Parallel work | Same chip, different boards: [0xSero’s M4 mini](https://github.com/0xSero/mac-mini-m4-linux), @wtsnz on an M4 Max |

Official Asahi: M1/M2 are the supported laptops. M3 is limited. **M4 is bring-up.** We are not claiming Asahi support.

---

## What m1n1 is (the important mental model)

On a PC, GRUB or Windows Boot Manager is just a file on disk you can replace.

On Apple Silicon:

1. **SecureROM** (in the chip) starts.
2. **iBoot** (Apple-signed) continues. You do not replace this.
3. iBoot looks up a **boot policy** in the Secure Enclave: “which volume, which kernel hash, how paranoid.”
4. It loads that kernel. For macOS, that is **XNU**. For us, we enroll **m1n1** as a *fully untrusted OS image* (**fuOS**). Apple’s name, not ours.

**m1n1** is a small program (Asahi Linux) that:

- Speaks USB to another computer (CDC “uartproxy”).
- Can dump the **ADT** (Apple Device Tree — the firmware’s map of the SoC).
- Can run a **hypervisor**: Apple’s kernel or Linux as a guest, while m1n1 traces MMIO.

It is **not** Linux. It is the first code *you* control after iBoot.

```
hold power → picker → boot "AsahiHost" (tiny enrolled macOS)
        → iBoot loads m1n1 instead of XNU
        → Air shows a black or Apple-ish panel (that is OK)
        → USB device "m1n1 uartproxy" appears on the other Mac
        → Python client: poke memory, dump ADT, boot Linux in RAM
```

### The proxy is not “inside Finder”

When the Air is running **normal macOS**, there is no m1n1 proxy. When it is running **m1n1**, macOS is not running.

Typical setup:

- **Air** — boots m1n1 (debuggee).
- **Studio / another Mac** — Python (`m1n1` proxy client) over USB-C.

That is how we “explore the components”: not by attaching to macOS, but by talking to the SoC the way a JTAG probe would, using maps Apple already published in the ADT.

---

## How far we are (gates)

Think of these as checkpoints, not a distro installer.

| Gate | Meaning | This Air | Date |
| --- | --- | --- | --- |
| **G0** | We know chip, board, firmware | **PASS** | 2026-09-05 |
| **G1** | m1n1 enrolled, USB proxy talks | **PASS** (via a real second macOS named **AsahiHost**, not the tiny stub) | 2026-09-05 20:49 |
| **G2** | Linux kernel prints its banner (RAM, under m1n1 HV) | **PASS** — `Linux version 7.1.9` + `Machine model: Apple MacBook Air (15-inch, M4, 2025)` on HV VUART | 2026-09-06 15:58Z |
| **G2b** | First userspace process (`/init`) | **not yet** — hang in `kernel_clone` / `copy_process` | 2026-09-07 |
| **G5** | NVMe read-only | not started |  |
| **G6** | Linux-owned display | not started (leftover Apple framebuffer does not count) |  |
| **G7 / trackpad** | Keyboard / trackpad (MTP) | not started |  |
| **Weston** | Pointer moving on a real compositor | not started |  |
| **G9** | Format a Linux partition | **blocked** until G5 + a spoken yes |  |
| **G10 / G11** | Real GPU / Omarchy | blocked on a kernel console + GPU ISA |  |

**Compared with others (same month, public):** [0xSero](https://github.com/0xSero/mac-mini-m4-linux) got G1 + RAM Linux + Ethernet SSH on an M4 mini; NVMe and display still failing there. @wtsnz on an M4 Max has gone further (internal NVMe, Linux-owned scanout, software Weston, trackpad). We are **ahead of “no m1n1”** and **behind Weston+trackpad** by: userspace, DCP display, MTP HID, then Weston. NVMe/Wi-Fi/GPU are *not* required for a pointer-on-Weston demo.

---

## Two boot paths we learned the hard way

### Path that **failed**: tiny 2.5 GB “stub macOS”

Stock Asahi on M1/M2: the installer creates a 2.5 GB fake macOS, you hold power, pick it, a “Finish Installation” app runs in **that volume’s** recovery, and `kmutil` enrolls m1n1.

On **this Air, macOS 26.5.1**, that picker never cooperates. Details: [docs/step2-failure-matrix.md](docs/step2-failure-matrix.md).

Dead ends (do not retry):

- Bless the stub as the default startup disk → Apple yellow screen “macOS needs to be reinstalled.” **Startup Disk → Macintosh HD**, never tap Recovery (that invites a full reinstall and firmware drift).
- Click the stub in the picker (even holding Option) → XNU panic `rootvp not authenticated`.
- `bputil -nc` (Permissive) from **Macintosh HD’s** recovery → `pairing (17)`. Wrong recovery cannot change the stub.
- Hiding the stub’s kernelcache, rewriting `.IAPhysicalMedia`, extra APFS “Finish Installation” volume.

What *did* work on the stub, from Macintosh HD recovery: `bputil -g` (**Reduced** only). Permissive still needs the stub’s own 1TR.

### Path that **worked**: a real second macOS (**AsahiHost**)

Install a **complete** extra macOS on the internal disk (we used ~52 GB). From **its** recovery (it is a real OS, so Apple will 1TR it):

1. Set **Permissive** security **only on AsahiHost** (Macintosh HD stays Full/Reduced — DRM on your daily macOS is unchanged).
2. `kmutil configure-boot` of m1n1 **on AsahiHost**, never on Macintosh HD.
3. Hold power → pick **AsahiHost** → m1n1 USB proxy.

That is G1 on this machine.

---

## Getting started (if you are repeating this)

You need: the M4 Mac, a **second computer**, a USB-C **data** cable (not charge-only), backups, and a willingness to brick-recover via DFU if you enroll m1n1 on the **wrong** volume.

1. **Freeze macOS** on a known build. We used 26.5.1 (25F80). Updates change firmware ABI.
2. Keep a healthy **Macintosh HD**. Never run `kmutil configure-boot` against it.
3. Do **not** follow the 2.5 GB stub + “Finish Installation” recipe on 26.5.1 until someone shows a picker that actually launches step2 (see the failure matrix).
4. Prefer a **real second macOS** (AsahiHost) for enrollment.
5. Second computer: Asahi `m1n1` repo, Python proxy client, USB-C into the Air’s **rightmost left-side** port (DFU/proxy lore; avoid hubs).
6. After G1: dump the **ADT** before guessing hardware. Then Linux in RAM under the hypervisor (`idle=nop` on M4 or the kernel looks “dead” in WFI).
7. Agents: read [AGENTS.md](AGENTS.md) and `status.json` before writing code.

We are **not** publishing a one-liner installer. Stock `curl https://alx.sh | sh` will refuse this Air on purpose.

---

## Repo map (people and agents)

| Path | What |
| --- | --- |
| [AGENTS.md](AGENTS.md) | Rules for coding agents (bans, honesty, where truth lives) |
| [status.json](status.json) | Machine-readable gates and hashes |
| [docs/step2-failure-matrix.md](docs/step2-failure-matrix.md) | Every stub-1TR knob we turned, with errors |
| [docs/what-worked.md](docs/what-worked.md) | Short list of things that actually passed |
| [docs/asahi-pr.md](docs/asahi-pr.md) | Why we are **not** opening an Asahi installer PR yet |
| [docs/gates.md](docs/gates.md) | Gate definitions |

This GitHub repo is **documentation**. The live lab (kernels, 20 GB trees, USB scripts) stays on the studio machine until something is worth hashing into `status.json`.

---

## Credits

- [Asahi Linux](https://asahilinux.org/) — m1n1, the installer model, DCP/HID/NVMe on M1/M2.
- Sven Peter, Yureka Lilian, and the Asahi kernel/m1n1 trees (T8132 PMGR, HV GXF/SPTM, 7.2 ANS/SMP notes).
- [0xSero/mac-mini-m4-linux](https://github.com/0xSero/mac-mini-m4-linux) — same SoC, different board; landmines we did not want to rediscover.
- @wtsnz — public M4 Max checkpoints (scanout, Weston, trackpad).

We are not Asahi and not Omarchy. Omarchy is [omarchy.org](https://omarchy.org) / [omacom/omarchy](https://github.com/omacom/omarchy).

---

## License

Notes in this repo: MIT. Upstream Asahi/m1n1/Linux keep their own licenses. Do not paste Apple firmware into git.
