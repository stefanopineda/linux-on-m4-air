# Linux on a 15″ M4 MacBook Air

**Status (2026-09-08):** this is a **lab log**, not a distro. Apple’s installer still says M4 is unsupported. This Air has a working **m1n1 USB debugger** and has printed a **Linux kernel banner** (`7.1.9`) over a virtual UART. It does **not** yet run userspace, own the panel, talk to the trackpad, or run Omarchy. G2 is PASS. **G2b is not.** The 2026-09-07 `copy_process` sandwich was early; the wall now is timer/FIQ/`VM_TMR_FIQ_ENA_EL2`.

If you dual-boot Windows and Linux on a PC, this is the Apple Silicon version of “what actually happens at boot” — plus what we tried, what died, and how to reach the same wall we are on now.

Live kernels and USB scripts stay in a separate studio tree. This GitHub repo is the **notebook**.

---

## Read this in 90 seconds

| You might think | What is actually true |
| --- | --- |
| Install Linux and it replaces the Mac bootloader | **You cannot replace iBoot.** Apple’s boot ROM still runs. You add a tiny extra “OS” whose *kernel* is a program called **[m1n1](https://github.com/AsahiLinux/m1n1)**. |
| This is like GRUB on the EFI partition | Closer to: “install a second, fake macOS, then tell Apple’s tools that its kernel is our debugger.” |
| The USB cable boots Linux from the other Mac | **No USB boot** on Apple Silicon. The cable is a **debug probe** after m1n1 is already running on the Air. |
| Omarchy (Arch + Hyprland) is the first step | Omarchy is **userspace**. It needs a kernel, a console, and eventually a GPU. We are still in kernel bring-up. |
| QEMU / UTM / a Linux VM counts | **No.** Those are not this project. Honesty bar is below. |

**Honesty bar:** QEMU, UTM, Parallels, VKMS, leftover Apple boot logo, and “Hyprland on a fake display” do **not** count. A real step has a log, a date, and a hash.

---

## Where we are (this Air, 2026-09-08 / 09)

| Gate | Meaning | This Air | When |
| --- | --- | --- | --- |
| **G0** | Chip, board, firmware written down | **PASS** — `Mac16,13` / `j715ap` / T8132 `0x8132` / board `0x2E` / macOS **26.5.1 (25F80)** freeze | 2026-09-05 |
| **G1** | m1n1 enrolled, USB proxy talks | **PASS** via a full second macOS (**AsahiHost**). The 2.5 GB Asahi stub path is **dead** on 26.5.1 | 2026-09-05 20:49 |
| **G2** | Linux prints its banner (RAM, under m1n1 HV) | **PASS** — `Linux version 7.1.9` + `Machine model: Apple MacBook Air (15-inch, M4, 2025)` on HV VUART | 2026-09-06 15:58Z |
| **G2b** | First userspace process (`/init`) | **NOW** — guest never takes the Apple FIQ vector; `msr VM_TMR_FIQ_ENA_EL2` from m1n1 EL2h-SYNCs. Ghidra pivot on 25F80 dumps. **Not PASS.** Do not skip `copy_process` | 2026-09-08 |
| **G5** | NVMe read-only | not started |  |
| **G6** | Linux-owned display (`dcp`+`disp0`) | not started (leftover Apple framebuffer does not count) |  |
| **G7 / trackpad** | Keyboard / trackpad (MTP + dockchannel) | not started |  |
| **Weston** | Pointer on a real compositor | not started |  |
| **G9** | Format a Linux partition | **blocked** until G5 + a spoken yes |  |
| **G10** | Real AGX (not llvmpipe) | not started here |  |
| **G11** | Omarchy / Hyprland | blocked on a kernel console + GPU |  |

**The current wall, in one sentence:** guest Linux never takes the Apple FIQ vector; writing `VM_TMR_FIQ_ENA_EL2` from m1n1 SYNCs on this t8132. The 2026-09-07 hang on `kernel_clone`→`copy_process` was early in the walk — do not rewind it, and do not skip `copy_process`. Later clean boots (guest exception class 783–786, T/V/W probes, live-window FIQ inject) showed the cut is timer/FIQ delivery, not “we skipped too much kernel.” More `wdt_site` folklore cannot read Apple’s binaries. **~19s** still means the guest poke fired; **~100s** means it never ran. Details: [docs/g2b.md](docs/g2b.md), [docs/ghidra-pivot.md](docs/ghidra-pivot.md).

Machine-readable copy: [`status.json`](status.json). Gate definitions: [docs/gates.md](docs/gates.md).

---

## Where we lag (and where we do not)

This is the honest scoreboard against public M4 work in the same month. **We are behind.** That is the point of writing it down.

| Checkpoint | Asahi (upstream) | [0xSero](https://github.com/0xSero/mac-mini-m4-linux) (M4 mini) | [@wtsnz](https://x.com/wtsnz) (M4 Max) | This Air |
| --- | --- | --- | --- | --- |
| m1n1 USB proxy (G1) | production on M1/M2 | PASS (authorized stub) | PASS | **PASS** (AsahiHost, not stub) |
| Linux banner (G2) | production on M1/M2 | PASS (HV RAM Linux) | PASS | **PASS** (HV VUART, 2026-09-06) |
| Userspace / shell (G2b) | production | PASS (Ethernet SSH) | PASS (remote shell 2026-08-25) | **stuck here** |
| Internal NVMe | production | failing / open | PASS (boots off internal, 2026-08-26) | not started |
| Linux-owned panel (G6) | production (DCP) | open / failing | PASS (2026-08-30) | not started |
| Keyboard | production (SPI/HID) | — | PASS (2026-08-25) | not started (Air is **MTP**, not SPI) |
| Trackpad | production | — | PASS ([video 2026-09-06](https://x.com/wtsnz/status/2096472670909120650)) | not started |
| Software Weston | production-ish | Hyprland on **VKMS does not count** | PASS ([photo 2026-09-06](https://x.com/wtsnz/status/2096396349499642033)) | not started |
| Wi-Fi | production (M1/M2) | — | PASS (2026-08-29) | not started |
| Real AGX (G10) | production (M1/M2) | not started | **PASS** ([AGX+DCP fractal 2026-09-07](https://x.com/wtsnz/status/2096815939027349574); Hyprland still open) | not started |
| Omarchy / Hyprland on real GPU | Fedora Asahi, not Omarchy | invalid (RAM+VKMS) | **not yet** (his words) | not started |

**Gap to wtsnz’s 2026-09-06 Weston+trackpad demo:** five gates — G2b, G6, G7, trackpad, software Weston. NVMe, Wi-Fi, and AGX are **off that path**. His 2026-09-07 GPU clip is a *further* checkpoint (G10), still with Hyprland unfinished.

**Why the Air is slower than the mini on G1:** same SoC (T8132), different board and a **26.5.1 picker** that treats the 2.5 GB Asahi stub as a broken macOS. 0xSero’s mini got an authorized stub. We could not. The workaround was a **complete second macOS**. Log: [docs/step2-failure-matrix.md](docs/step2-failure-matrix.md).

**Why we are slower than wtsnz on everything after G2:** he had a shell on 2026-08-25. We still do not. Until `/init` runs, DCP, MTP, and AGX are not the experiment.

---

## Prior work (stand on this, do not redo it)

Bring-up here is a thin layer on other people’s years.

- **[Asahi Linux](https://asahilinux.org/)** — m1n1, the fuOS installer model, DCP, HID, NVMe, AGX, and the [linux](https://github.com/AsahiLinux/linux) / [m1n1](https://github.com/AsahiLinux/m1n1) trees. Official support is **M1/M2** laptops; M3 is limited; **M4 is bring-up**. Stock `curl https://alx.sh | sh` will refuse this Air on purpose. The Asahi tree already has `t8132-j715.dts` (this board). We did not invent the SoC map.
- **Sven Peter ([@svenpeter42](https://x.com/svenpeter42))** — m1n1, DCP, and the “proxy is a JTAG that speaks USB” workflow.
- **Yureka Lilian ([@yuyuyureka](https://x.com/yuyuyureka))** — T8132 PMGR, HV GXF/SPTM, 7.2 ANS/SMP notes. Use the Asahi kernel; do not invent MMIO.
- **Janne Grunau and the Asahi kernel releases** — `asahi-7.1.x` tags. This lab is on **7.1.9** (`77cb8f24c`). 7.2 exists; we have not A/B’d it on G2b.
- **[0xSero/mac-mini-m4-linux](https://github.com/0xSero/mac-mini-m4-linux)** — same T8132, **Mac mini** (`j773`). Public HEAD last noted `e0d67e3` / 2026-08-28. Do not DM. Landmines we copied on purpose: do not `pmgr_reset(DISPEXT*)` (SError), skip locked IMP sysregs, kboot teardown vs HV. Different display (`dcpext0` vs our `dcp`+`disp0`). Hyprland-on-VKMS is documented there as not completion; we agree.
- **[@wtsnz](https://x.com/wtsnz)** — public M4 **Max** MacBook Pro checkpoints (Aug–Sep 2026): console, keyboard, NVMe, USB Ethernet, Wi-Fi, Linux-owned scanout, software Weston, trackpad, then AGX+DCP. No public tree we are tracking; the videos are the evidence. This Air is still G2b.
- **Eryk Wieliczko ([@ewninjaofficial](https://x.com/ewninjaofficial))** — [Ghidra + macOS snapshots + many Linux reboots](https://x.com/ewninjaofficial/status/2093963487449841675) to build missing T8132 pieces (GPU compute demo on an MBA M4). This lab is following that **method** (dump 25F80 binaries, decompile, then experiment), not claiming his GPU result. [docs/ghidra-pivot.md](docs/ghidra-pivot.md).
- **Not taken:** [@kwargq](https://x.com/kwargq/status/2097351557717074307) ran an Eryk / Aurora Silicon `.pkg` installer (desktop, still llvmpipe). The lab forbade installer-consume.
- **Phoronix, 2026-07** — [initial M4 device-tree patches](https://www.phoronix.com/news/Apple-M4-DT-Linux). Start of the public M4 DT story, not a laptop bring-up.
- **Omarchy** is [omarchy.org](https://omarchy.org) / [omacom/omarchy](https://github.com/omacom/omarchy). We are not Omarchy. We are not Asahi. The end-state *this* lab wants is Omarchy on this Air; the path is Asahi-shaped bring-up first.

---

## What m1n1 is (the important mental model)

On a PC, GRUB or Windows Boot Manager is just a file on disk you can replace.

On Apple Silicon:

1. **SecureROM** (in the chip) starts.
2. **iBoot** (Apple-signed) continues. You do not replace this.
3. iBoot looks up a **boot policy** in the Secure Enclave: which volume, which kernel hash, how paranoid.
4. It loads that kernel. For macOS, that is **XNU**. For us, we enroll **m1n1** as a *fully untrusted OS image* (**fuOS**). Apple’s name, not ours.

**m1n1** is a small program (Asahi) that:

- Speaks USB to another computer (CDC `uartproxy`).
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

When the Air is running **normal macOS**, there is no m1n1 proxy. When it is running **m1n1**, macOS is not running. The other computer is a debug host, not a USB installer.

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

What *did* work on the stub, from Macintosh HD recovery: `bputil -g` (**Reduced** only). Permissive still needs the stub’s own 1TR, which we never got.

### Path that **worked**: a real second macOS (**AsahiHost**)

Install a **complete** extra macOS on the internal disk. From **its** recovery (it is a real OS, so Apple will 1TR it):

1. Set **Permissive** security **only on AsahiHost** (Macintosh HD stays Full/Reduced — DRM on your daily macOS is unchanged).
2. `kmutil configure-boot` of m1n1 **on AsahiHost**, never on Macintosh HD.
3. Hold power → pick **AsahiHost** → m1n1 USB proxy.

That is G1 on this machine. Picker habit: icons left→right, focus starts on **nothing**. If AsahiHost was last booted, **Right then Enter**. Do **not** 3× Left (wraps to Options/Recovery).

---

## Catch up to where this lab is now

A full runbook: [docs/getting-started.md](docs/getting-started.md). The short version:

1. **Freeze macOS** on a known build. We used 26.5.1 (25F80). Updates change firmware ABI. Do not “just update.”
2. Keep a healthy **Macintosh HD**. Never `kmutil configure-boot` against it.
3. Skip the 2.5 GB stub recipe on 26.5.1. Use a **complete second macOS** for enrollment.
4. Second computer + USB-C **data** cable. Asahi `m1n1` proxy client. Dump the **ADT** before guessing hardware.
5. Linux in **RAM under the hypervisor**, not direct kboot (kboot still ~6s-resets on this Air). Bootargs that got us a banner: `earlycon keep_bootcon console=none nr_cpus=1 idle=nop`. **`idle=nop` is load-bearing** — without it the kernel looks dead in WFI.
6. After the banner: **printk hangs** past the dummy console. Mute the console, then walk `rest_init` with a guest watchdog poke (19s vs 100s). Do not debug G2b with more bootargs.
7. Topology calls that hung and were skipped (kept): `set_mems_allowed`, `update_siblings_masks`, `init_cpu_topology`. Do **not** skip `copy_process`, `current->nsproxy`, or `numa_default_policy` (those skips did not unblock).
8. You are caught up when you can reproduce: banner on HV VUART; the 2026-09-07 `copy_process` sandwich as **history** (do **not** skip `copy_process` / `current->nsproxy`); then the timer/FIQ wall — guest exception class never entered after INIT, CNTP FIQ at EL2, CNTV pending, ENA already `f`, synthetic FIQ inject at `VBAR+0x300` still MUTED. Then the Ghidra encodings in [docs/ghidra-pivot.md](docs/ghidra-pivot.md). Do not IMP-MSR ENA/LR.

We are **not** publishing a one-liner installer.

---

## What we have that is ours

Not a kernel fork worth upstreaming. A **repro log**:

- 26.5.1 stub-1TR is a dead path on this laptop (matrix above). AsahiHost is the G1 that worked.
- Air ADT classification: panel **`dcp`+`disp0`** (not the mini’s `dcpext0`); HID **MTP + dockchannel** (not SPI); NVMe ANS; AIC3; USB DRD; Wi-Fi `bcm4387`.
- G2 under HV with skip-teardown m1n1, locked IMP sysregs skipped, `idle=nop`, `nr_cpus=1`.
- G2b method: one unique `Image.gz` hash per watchdog site; never re-boot the same hash. 38 unique Images in the `rest_init` walk; 17 of those poked or patched inside `copy_process`. That sandwich is **history**. [docs/g2b.md](docs/g2b.md).
- 2026-09-08 timer/FIQ wall + Ghidra pivot: 25F80 from-dir dumps (`kernelcache.release.mac16g`, SPTM, `armfw_g16g`); ENA is `sys_reg(3,5,15,1,3)` MRS `0xD53DF160` / MSR `0xD51DF160`; kernelcache has 5× MSR ENA, 0× MRS ENA, 0× `VM_TMR_LR`; vmentry writes `ENA=0xF`. [docs/ghidra-pivot.md](docs/ghidra-pivot.md).
- We did **not** open an Asahi installer PR that allowlists `j715ap`. Issue-shaped data only: [docs/asahi-pr.md](docs/asahi-pr.md). We did **not** consume the kwargq / Aurora Silicon `.pkg`.

---

## Repo map

| Path | What |
| --- | --- |
| [docs/getting-started.md](docs/getting-started.md) | Repeat the path to G1/G2 and join the G2b walk |
| [docs/g2b.md](docs/g2b.md) | G2b wall: `copy_process` history, then the 2026-09-08 timer/FIQ poke mill |
| [docs/ghidra-pivot.md](docs/ghidra-pivot.md) | Eryk method vs installer; 25F80 dump list; ENA encodings; outstanding volume dump |
| [docs/step2-failure-matrix.md](docs/step2-failure-matrix.md) | Every stub-1TR knob, with errors |
| [docs/what-worked.md](docs/what-worked.md) | Short list of things that actually passed |
| [docs/gates.md](docs/gates.md) | Gate definitions |
| [docs/asahi-pr.md](docs/asahi-pr.md) | Why there is no installer enablement PR yet |
| [AGENTS.md](AGENTS.md) | Rules for coding agents (bans, honesty) |
| [status.json](status.json) | Machine-readable gates and pins |

---

## Hard bans (humans and agents)

- Never `kmutil configure-boot` on **Macintosh HD**.
- Never bless the **2.5 GB stub** as default startup disk.
- Never tap **Recovery / Reinstall macOS** on Apple’s yellow dialog. Escape: Startup Disk → Macintosh HD.
- Never `pmgr_reset` on `DISPEXT*` / `DISP_CPU` (mini SError).
- Never USB-reset gadget `0x1209:0x316d` to “unstick” a ghost ACM. Hold-power → AsahiHost.
- Never `mkfs` without an explicit human yes after G5 PASS.
- Never bump macOS past **26.5.1 (25F80)** on this machine.
- Never `msr VM_TMR_FIQ_ENA_EL2` or `VM_TMR_LR` on t8132 from m1n1 (MSR ENA EL2h-SYNCs; LR is unused in 25F80 XNU).
- Never consume a third-party M4 Linux `.pkg` (Aurora Silicon / installer-consume path).
- Never put secrets (passwords, `.env`, USB serials, LAN IPs, volume UUIDs) in git.

---

## Credits

Asahi Linux, Sven Peter, Yureka Lilian, Janne Grunau, [0xSero](https://github.com/0xSero), [@wtsnz](https://x.com/wtsnz), Eryk Wieliczko ([@ewninjaofficial](https://x.com/ewninjaofficial)). This notebook is **the lab** ([@themartiano](https://x.com/themartiano)). Mistakes in this log are ours.

---

## License

Notes in this repo: MIT. Upstream Asahi / m1n1 / Linux keep their own licenses. Do not paste Apple firmware into git.
