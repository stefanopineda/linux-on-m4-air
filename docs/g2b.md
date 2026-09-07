# G2b — the wall (2026-09-07)

**Goal:** first userspace process (`/init`) after the Linux banner. **Not there yet.**

UART is dead after we mute printk (the next `printk` after dummy-console init hangs). So this is not a “read the oops” problem. It is a “did this C statement run?” problem.

---

## How we know where we are

One compile-time poke, `g2_wdt_short(8)`, in the guest. Rebuild a **unique** `Image.gz`. Boot under the same m1n1 hypervisor. Never re-boot the same hash.

| `hv_life` | Meaning |
| --- | --- |
| **~19s** | Guest 8s watchdog fired. That statement **ran**. Advance. |
| **~100s** | EL2 90s watchdog. That statement **never ran**. Hang is the call or prologue immediately before. |
| **NO_BANNER** | Not a hang. Retry as a new hash (one glitch did this; same poke on a new Image was 19s). |

Held constant across this walk: one skip-teardown m1n1, one DTB (`t8132-j715`), one initramfs, HV (not kboot), bootargs `earlycon keep_bootcon console=none nr_cpus=1 idle=nop`, printk muted, three topology skips, `__always_inline kernel_init_freeable`.

**Isolation vs the other kernels on the path:** 19s can only come from the *guest* poke. iBoot and m1n1 do not produce 19s. That localizes *inside Linux*. It does **not** prove the defect is a pure C bug in one function: EL2 can still take an abort on a load Linux happens to emit.

---

## Current sandwich

| Site | Where | Result | Image (prefix) |
| --- | --- | --- | --- |
| 738 | `kernel_clone` after `valid_signal`, before the ptrace block | **19s** | `6deb11b5…` |
| 740 / 713 | immediately before `copy_process` | **100s** | `89f2654f…` |
| 741 | first statement of `!CLONE_UNTRACED` ptrace block | **100s** | this spawn is `CLONE_UNTRACED` — block is dead |
| 742 | skip that dead ptrace block; poke still before `copy_process` | **100s** | did not unblock |
| 743 | plus `static` `vfork` completion (off stack) | **100s** (`hv_life` ~82s, messy) | last boot logged |

Last **19s inside `copy_process` itself** was site 723 (`a3e3295c…`): after `args->flags`, before `current->nsproxy`. The load of `current` (`mrs SP_EL0`) is the sticky part.

**Do not skip `copy_process`.** **Do not skip `current->nsproxy`.** Both were either banned or tested and failed to unblock.

The hang **moved** after we changed `copy_process` codegen: an earlier poke that used to be 19s (immediately before the call) became 100s. So this is not “a bug isolated to `copy_process`.” It tracks **`current` / `SP_EL0`**, which both `kernel_clone` (`ptrace_event_enabled(current)`) and `copy_process` (`current->nsproxy`) touch. Compiler hoist can make a “dead” ptrace block still load `current`.

---

## How guesses improve

Not a test suite. On 19s, move the poke forward. On 100s, stop and pick **one** lever:

1. **Move the poke** (default).
2. **Skip-class** — comment out a call only if the *next* poke then fires. Topology skips passed. `numa_default_policy`, `kernel_init_freeable`, `nsp=NULL` did not.
3. **Compiler attributes** — drop `__latent_entropy`, `__no_stack_protector`, avoid `noinline` wrappers when objdump shows PAC / canary / `mrs` *before* the poke.
4. **Stack layout** — large locals `static` so the prologue stops zeroing them before the poke.
5. **How `current` is loaded** — split, wrapper, `CLONE_THREAD` else `&init_nsproxy`. All 100s so far.
6. **Sanity reconfirm** — after any of 2–5, re-probe a site that used to be 19s. If it is now 100s, the hang moved.

Objdump is the judgment step between boots, not a boot.

---

## How many versions we actually booted

From the lab `wdt-bisect` log (not slogans):

| Scope | Unique Images |
| --- | ---: |
| Inside `copy_process()` (poke or C change in that function) | **17** |
| `kernel_clone` around the call | **13** |
| Whole `rest_init` WDT walk | **38** (15×19s, 22×100s, 1×NO_BANNER) |
| Broader G2/G2b HV ticks (banner, mute, topology, …) | **103** ticks |

Themes of the clone/`copy_process` era (overlapping; one Image often combines a poke with a kept attribute):

| Theme | Count | What happened |
| --- | ---: | --- |
| Probe placement only | 14 | default walk |
| Compiler attributes | 4 | dropping `__latent_entropy` **kept** (19s); `noinline` wrapper failed |
| Stack layout (`static delayed` / `static vfork`) | 2 | `delayed` **kept** (19s); `vfork` did not unblock 713 |
| Skip-class in this era | 4 | **0 succeeded** (nsproxy, ptrace) |
| `current` / nsproxy load shape | 5 | all 100s |
| Sanity reconfirm of an earlier site | 6 | 713 flipping 19s→100s after `copy_process` edits is the important one |

**Kept from earlier in `rest_init`:** skip `set_mems_allowed`, skip `update_siblings_masks`, skip `init_cpu_topology`, `__always_inline kernel_init_freeable`.

**Thrown away:** skip numa, skip `kernel_init_freeable`, skip nsproxy, noinline wrapper, `CLONE_THREAD`/`init_nsproxy` branch. Skip ptrace did not unblock the call.

---

## What this is not

- Not a second kernel. `copy_process` is one C function in guest Linux (`kernel/fork.c`). The other kernels on the path (iBoot2, m1n1 EL2) stay pinned.
- Not Omarchy. Not a display bug. Not AGX. Those need `/init`.
- Not “try 7.2 / kboot / more CPUs” until a 713 reconfirm after a `current` isolation experiment. We have not A/B’d those during this walk.

---

## Next honest experiment

Isolate **`mrs SP_EL0` / `current`** on the `kernel_clone` side without skipping `copy_process`, then **reconfirm 713**. If 713 stays 100s, the hang moved again and the last C change is implicated.
