# What came past copy_process (the G2b walk, 2026-09-07 → 09-10)

This is the journey through the last `rest_init` stop — `kernel_clone` →
`copy_process` → `kernel_init` → `/init` — that turned G2b from open to
**PASS (2026-09-10T21:34Z)**. It is a lab log, not slogans. Every boot below is
one Image, one hash, one WDT site. **Never re-boot a hash.**

> Banned skips (do not rewind): `copy_process`, `current->nsproxy`,
> `numa_default_policy`. The `copy_process` sandwich was **early** — later clean
> boots walked *past* it into `kernel_init` and `/init`. Do not skip `copy_process`.

---

## 1. The wall we started on (2026-09-08)

The timer/FIQ/`VM_TMR_FIQ_ENA_EL2` wall. Guest never took the Apple FIQ vector;
writing ENA from m1n1 EL2h **EL2h-SYNCs** on this t8132. The 2026-09-07
`kernel_clone`→`copy_process` WDT walk was bracketing this and got stuck:

- **19s** = guest 8s watchdog fired (statement ran) → advance.
- **100s** = EL2 90s watchdog (statement never ran) → hang is the call before it.
- **NO_BANNER** = not a hang, retry as a new hash.

~19s localizes the defect *inside Linux*. It did not prove a single-function C
bug: EL2 can still abort on a load Linux happens to emit.

---

## 2. The `copy_process` era (WDT walk)

38 unique Images in the whole `rest_init` WDT walk (15×19s, 22×100s, 1×NO_BANNER);
17 of those poked or patched inside `copy_process`. 13 around `kernel_clone`.

| Site | Where | Result |
| --- | --- | --- |
| 738 | `kernel_clone` after `valid_signal`, before ptrace block | 19s |
| 740 / 713 | immediately before `copy_process` | 100s |
| 741 | first stmt of `!CLONE_UNTRACED` ptrace block | 100s (block is dead) |
| 742 | skip dead ptrace block; poke still before `copy_process` | 100s |
| 743 | plus `static` `vfork` completion | 100s (~82s, messy) — last boot logged |

Last **19s inside `copy_process` itself** was site 723 (`a3e3295c…`): after
`args->flags`, before `current->nsproxy`. The load of `current` (`mrs SP_EL0`)
is the sticky part.

**The hang moved after edits:** an earlier poke that used to be 19s (before the
call) became 100s. So it tracks **`current` / `SP_EL0`**, which both
`kernel_clone` (`ptrace_event_enabled(current)`) and `copy_process`
(`current->nsproxy`) touch. Compiler hoist can make a "dead" ptrace block still
load `current`.

Kept from earlier `rest_init`: skip `set_mems_allowed`, `update_siblings_masks`,
`init_cpu_topology`, `__always_inline kernel_init_freeable`.
Thrown away: skip numa, skip `kernel_init_freeable`, skip nsproxy, noinline
wrapper, `CLONE_THREAD`/`init_nsproxy` branch. Skip ptrace did not unblock.

---

## 3. Getting past copy_process into kernel_init

Once `copy_process` was cleared, the walk moved into `kernel_init` /
`do_initcalls`. Two things showed up:

1. A **`cpu_attach_domain` WARN** at topology.c — **caught and survived**; the
   boot kept going. Not a crash.
2. Then `simplefb_probe` → `register_framebuffer` → `fbcon_fb_registered` where a
   **NULL pointer deref in `fbcon_startup+0xd4`** killed the boot:

```
Unable to handle kernel NULL pointer dereference at 0x190
Internal error: Oops...
pc : fbcon_startup+0xd4/0x1ec
Kernel panic — Attempted to kill init!
```

This is a **bounded, addressable** bug — a NULL deref in the framebuffer/console
takeover *after* we were already past `copy_process` into driver initcalls.
Not the `copy_process` wall. Not a FIQ miss.

---

## 4. The clean pass (2026-09-10T21:34Z)

`CONFIG_FB_SIMPLE=n` (skip `simplefb`/`fbcon`) avoids the `fbcon_startup` NULL
deref. The next Image (`aa431d99`, pp-v1) then reached `copy_process` →
`kernel_init` → `/init` → `G2_INIT_ALIVE` with **no Oops, no panic**:

```
[    0.001960] g2_pp_v0: G2_PP_V0 registered cpu=0 rating=500
[    0.052755]  kernel_init+0x78/0x200
[    0.210607] Run /init as init process
[    0.236208] G2_INIT_ALIVE j715 t8132 uart-mmio
```

One caught `cpu_attach_domain` WARN at topology.c:694, then continued.
Hang-window PC `400008` (EL0). `hv_life` ~18.6s is the usual 19s WDT after
`/init`, not a miss of `copy_process`. **Do not re-boot `aa431d99`.**

Honest-pass follow-up Image (`998c5a50`, pp-v0) reached `copy_process` then hit
the `fbcon_startup` NULL deref again — proving the fbcon path is the new wall
when `simplefb` is on. pp-v1 (FB_SIMPLE=n) is the clean G2b pass.

---

## 5. The newest frontier (2026-09-12)

- **pp-g5-nvme** (`e88ca65a`, 2026-09-12T02:58Z): past `copy_process` into driver
  initcalls, IPv6 `NET: Registered PF_INET6` at 0.179s. **G5 GPT never ran** —
  hang-window PC `aic_ipi_send_single+0x34`, `fiq_n` storm ~1.74e7. Do not
  re-boot `e88ca65a`. Do not mkfs.
- Still open: **G5** NVMe read-only, **G2c** lid, **G6** Linux-owned scanout.

---

## How guesses improve (still true)

Not a test suite. On 19s, move the poke forward. On 100s, stop and pick **one**
lever:

1. **Move the poke** (default).
2. **Skip-class** — comment out a call only if the *next* poke then fires.
3. **Compiler attributes** — drop `__latent_entropy`, `__no_stack_protector`.
4. **Stack layout** — large locals `static`.
5. **How `current` is loaded** — split, wrapper, branch. All 100s so far.
6. **Sanity reconfirm** — after any of 2–5, re-probe a site that used to be 19s.
   If it is now 100s, the hang moved.

Objdump is the judgment step between boots, not a boot.

---

## Summary

`copy_process` is **passed and not skipped** — the walk went past it into
`kernel_init`, `/init`, and `G2_INIT_ALIVE`. The next wall is the `fbcon_startup`
NULL deref (`simplefb_probe`), fixed with `CONFIG_FB_SIMPLE=n`. The newest boot
is at G5 (NVMe), stuck on an AIC IPI storm before the GPT read. Honest G2b PASS:
[`docs/g2b-pass-20260910.md`](g2b-pass-20260910.md).
