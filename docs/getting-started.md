# Getting started — catch up to this lab

This is not an installer. It is the shortest path that reaches **the same wall we are on**: Linux 7.1.9 banner under m1n1 HV, then the timer/FIQ/`VM_TMR_FIQ_ENA_EL2` wall (guest never takes the Apple FIQ vector). The 2026-09-07 `kernel_clone` → `copy_process` sandwich is history on the way there — do **not** skip `copy_process`.

If you only want the mental model, the [README](../README.md) is enough. If you want to stand on the same hardware checkpoint, read this, then [g2b.md](g2b.md) and [ghidra-pivot.md](ghidra-pivot.md).

**Hardware we used:** 15″ MacBook Air (2025), `Mac16,13` / `j715ap` / T8132. A **second computer** (another Mac is easiest). A USB-C **data** cable, not charge-only. Avoid hubs.

**You can brick-recover via DFU** if you enroll m1n1 on the wrong volume. That is why Macintosh HD is sacred.

---

## 0. Freeze firmware

Write down: product, board (`j715ap`), chip (`0x8132`), macOS version **and build**.

We froze **26.5.1 (`25F80`)**. Do not install a newer IPSW on a machine you care about matching this log. iBoot and SEP policy change across builds; the stub-1TR deaths are 26.5.1-specific.

Keep a healthy **Macintosh HD**. Never `kmutil configure-boot` against it.

---

## 1. Do not follow the 2.5 GB stub recipe on 26.5.1

Stock Asahi (`curl https://alx.sh | sh`) will refuse M4. Even if you force a stub:

- Bless stub default → yellow “macOS needs to be reinstalled.”
- Click stub in the picker → XNU panic `rootvp not authenticated`.
- Macintosh HD recovery cannot Permissive-downgrade the stub (`pairing (17)`).

Escape from yellow, every time: **Startup Disk → Macintosh HD**. Never Recovery. Never Reinstall.

The full dead-end log is [step2-failure-matrix.md](step2-failure-matrix.md). Skip to step 2 unless you are writing an installer.

---

## 2. G1 — enroll m1n1 on a *complete* second macOS

Call it **AsahiHost** (any name). It must be a real macOS 26.5.1 (`25F80`) volume, not a 2.5 GB stub.

1. Install that OS. Boot it **once** so Apple will give it its own recovery.
2. Hold power → pick **AsahiHost** → enter **its** 1TR (paired to that volume).
3. `bputil -nc` (**Permissive**) **only on AsahiHost**. Macintosh HD stays Full/Reduced.
4. `kmutil configure-boot` of a stage-1 **m1n1** image **on AsahiHost**.
5. Reboot, hold power, pick **AsahiHost**.

**Pass looks like:** the other computer sees a USB CDC gadget named something like `m1n1 uartproxy`. Python `proxyclient` answers. macOS is **not** running on the Air.

Picker habit on this machine: focus starts on nothing. If AsahiHost was last: **Right then Enter**. Do not 3× Left.

If the USB node exists but NOP times out (ghost ACM): **do not** host-reset vendor `0x1209` / product `0x316d`. Hold-power → AsahiHost.

Our stage-1 pin: m1n1 sha256 `0dbde320b863b5cd84ebc1a152456ed2b0cd1818880d4324866c32d534e74a35`. Use a current Asahi m1n1; the hash is “this is what G1 was,” not a required binary.

---

## 3. Dump the ADT before guessing hardware

From the proxy host, dump the **Apple Device Tree**. Classify nodes. Do not copy a Mac mini overlay onto an Air.

On **this** Air the ADT said:

| Block | What we found |
| --- | --- |
| Panel | `dcp` + `disp0` (laptop). **Not** `dcpext0` (mini). |
| HID | MTP + dockchannel. **Not** SPI. |
| Storage | NVMe ANS |
| IRQ | AIC3 |
| USB | DRD (the debug gadget is not USB host) |
| Wi-Fi | `bcm4387` |

Asahi linux already ships `t8132-j715.dts` for this board. Prefer that over a hand-rolled DTB.

---

## 4. G2 — Linux banner under the hypervisor

Direct **kboot** (m1n1 jumps to Linux, tears down, USB dies) still ~6s-resets on this Air. Apple has **no EL3 PSCI**; Linux `reboot()` cannot reset the SoC. Timing on kboot cannot tell “reached init” from “died in teardown.”

What worked: **m1n1 hypervisor**, skip framebuffer/MMU teardown that `memcpy128`s after USB is down, skip **locked T8132 IMP sysregs** (writing secondary RVBAR SError’d). Guest Linux on VUART.

Kernel: Asahi `linux` **asahi-7.1.9** (`77cb8f24c` here). Bootargs that produced a banner:

```
earlycon keep_bootcon console=none nr_cpus=1 idle=nop
```

- **`idle=nop` is load-bearing.** Without it the guest looks hung in WFI.
- **`nr_cpus=1`** for the first banner. SMP is a later fight (t8132 secondaries, RVBAR locked).
- **`console=none`** — `console=tty0` hung at `Console [tty0] enabled`. SoC UART (`ttySAC0`) is not the USB proxy.

**Pass looks like:** `Linux version 7.1.9…` and `Machine model: Apple MacBook Air (15-inch, M4, 2025)` on the HV virtual UART. Leftover Apple logo on the panel does **not** count.

Our G2 PASS: 2026-09-06T15:58:11Z.

---

## 5. After the banner: mute printk, then walk with a watchdog

The next printk after dummy-console init **hangs**. More bootargs will not help. Mute the console (`suppress_printk` / equivalent), then you get `G2_MUTED` and silence.

Userspace is still not up. `rest_init` has not finished. `/init` has not run.

Debug method that actually moved: plant one `g2_wdt_short(8)` behind a compile-time site id, rebuild a **unique** `Image.gz`, boot once.

| Guest lifetime | Meaning |
| --- | --- |
| ~19s | The 8s guest poke ran. That C statement executed. Advance. |
| ~100s | EL2 90s watchdog. That statement never ran. Hang is the call or prologue immediately before. Stop and look at the binary. |
| Same sha256 as a previous boot | Do not re-boot. The hash *is* the experiment id. |

Keep:

- skip `set_mems_allowed()`
- skip `update_siblings_masks()`
- skip `init_cpu_topology()`
- `__always_inline kernel_init_freeable`
- drop `__latent_entropy` and stack protector on `copy_process`
- `static` the large `delayed` local in `copy_process` (off the stack)

Do **not** skip: `copy_process`, `current->nsproxy`, `numa_default_policy`, `kernel_init_freeable` (skipping the last two was tested; they did not unblock).

The 2026-09-07 catch-up was: last **19s** poke after `valid_signal` in `kernel_clone`, poke immediately before `copy_process` is **100s**. Keep that as history; do **not** skip `copy_process`. You are caught up *now* when you have also reproduced the timer/FIQ wall (exception class silent after INIT; ENA already `f`; MSR ENA SYNCs) and the encodings in [ghidra-pivot.md](ghidra-pivot.md). That is [g2b.md](g2b.md).

---

## 6. What not to spend a week on yet

Off the path to a pointer on software Weston (the public checkpoint we are chasing):

- NVMe format / Omarchy installer
- Wi-Fi
- Real AGX (wtsnz has this on an M4 Max; we do not have userspace)
- Direct kboot instead of HV
- `display_start_dcp` / `pmgr_reset(DISP*)` from m1n1
- Live-patching RX `.text` on m1n1 (EL2 data abort)

On the path, in order: **G2b `/init` → G6 dcp+disp0 → G7 keyboard → trackpad → software Weston**.

---

## Agent checklist

Read [AGENTS.md](../AGENTS.md) and [`status.json`](../status.json) before proposing commands. One physical boot = one hypothesis = one log line.
