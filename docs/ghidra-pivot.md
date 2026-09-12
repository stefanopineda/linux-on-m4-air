# Ghidra pivot (2026-09-08)

Stop the G2b poke mill. Dump frozen **25F80** binaries, decompile the timer/AIC/ENA path, then experiment. G2b is **not PASS**.

This is the **method** [@ewninjaofficial](https://x.com/ewninjaofficial) published for missing T8132 pieces — not a claim of his GPU result.

---

## Two public paths (this lab took one)

| Path | What it is | This lab |
| --- | --- | --- |
| **Eryk / Ghidra + snapshots** | [GPU compute on MBA M4](https://x.com/ewninjaofficial/status/2093963487449841675): dump macOS, many Linux reboots, Ghidra on GPU drivers/firmware. | **Following the method** (dump 25F80, decompile, then experiment). Not claiming GPU compute. |
| **Installer consume** | [@kwargq](https://x.com/kwargq/status/2097351557717074307) ran an Eryk / Aurora Silicon `.pkg` (desktop, still **llvmpipe**). | **Not taken.** Do not consume that installer. |

Asahi already has m1n1, fuOS, and `t8132-j715.dts`. Official support is M1/M2; M4 is bring-up.

**Side note (progress):** G2b was crossed 2026-09-10 — `copy_process` → `kernel_init` → `/init` (`G2_INIT_ALIVE`). This pivot was the stop before that. See [what-came-past-copy-process.md](what-came-past-copy-process.md). [0xSero/mac-mini-m4-linux](https://github.com/0xSero/mac-mini-m4-linux) is the same SoC on a mini (`j773`); public HEAD last noted `e0d67e3` / 2026-08-28 — do not DM. [@wtsnz](https://x.com/wtsnz) on an M4 Max is further along (shell, NVMe, Weston, AGX clips). This Air is still G2b.

---

## What was dumped (25F80)

IPSW-class / **from-dir**. **Not** yet a live Macintosh HD volume copy.

| Artifact | Notes |
| --- | --- |
| `kernelcache.release.mac16g` | XNU kernelcache for `mac16g` / this Air class. Timer/AIC/ENA decompile target. |
| `sptm.t8132.release.im4p` | SPTM. Unpacked Mach-O ~1.2 MB. |
| `armfw_g16g.im4p` | AGX firmware blob (GPU follow-up; not the current wall). |

Studio tools: **Ghidra 12.1.3**, `ipsw`, OpenJDK 21.

**Outstanding (planned Macintosh HD boot, not another guest poke):** live volume dump of `BootKernelExtensions.kc` (AppleAIC kext) plus a snapshot. Do not treat the from-dir kernelcache as that kext.

Do not paste Apple firmware into git. Hashes and encodings below are notes, not blobs.

---

## Encodings (load-bearing)

Ghidra `findBytes` first used the **wrong** encoding: **s2_5** / op0=2 (`0xD535F160`). That produced a false “zero hits ⇒ steering not in XNU.”

Real register is `sys_reg(3,5,15,1,3)` (`s3_5_c15_c1_3`):

| Name | Encoding | Word |
| --- | --- | --- |
| `VM_TMR_FIQ_ENA_EL2` MRS | `sys_reg(3,5,15,1,3)` | `0xD53DF160` |
| `VM_TMR_FIQ_ENA_EL2` MSR | same | `0xD51DF160` |
| `VM_TMR_LR_EL2` MRS | `sys_reg(3,5,15,1,2)` | `0xD53DF140` |
| `VM_TMR_LR_EL2` MSR | same | `0xD51DF140` |

Ground truth: llvm-mc + Capstone, then a Ghidra pass with the corrected words.

---

## Findings (kernelcache + SPTM)

Capstone, then Ghidra on `kernelcache.release.mac16g`:

| Hunt | Count | What it means |
| --- | ---: | --- |
| MSR `VM_TMR_FIQ_ENA_EL2` | **5** | Apple does write ENA from EL2. |
| MRS ENA | **0** | XNU never reads it. |
| MSR/MRS `VM_TMR_LR` | **0** | LR is not in the macOS vtimer protocol. Locked here; do not write it. |

The five MSR ENA sites:

- **Four clones** (EL2 / FIQ entry / guest-exit): `mov x0,#0; msr ENA` then `ICH_HCR.En=0`. Same shape: per-cpu TPIDR_EL2, SP_EL0/SP_EL1 save, stack switch, then ENA=0 + ICH_HCR.En=0.
- **One vmentry arm:** `tbnz w8,#0` (skip flag) → `mov w8,#0xf; msr ENA` then restore `ICH_HCR`. Lab peek `ENA f` is **Apple’s 4-bit vmentry mask**, not stuck junk. Asahi names bits 0–1 as CNTV/CNTP; Apple still writes `0xF`. Bits 2–3 are unnamed. Do not invent them. Do not MSR `0xF` from m1n1.

Whole-macho (same kernelcache):

| Sysreg | MSR | MRS | Note |
| --- | ---: | ---: | --- |
| `CNTV_CTL` / `CNTV_CVAL` / `CNTVOFF` | 0 | 0 | XNU never programs guest CNTV in this macho. |
| `HCR_EL2` | **0** | 8 | Never an MSR. MRS sites are EL2 vector stubs. |
| `CNTHCTL_EL2` | 1 | 1 | Splice is **`EL1NVPCT\|EL1NVVCT` only**, not `EL1TVT`. |

SPTM: **1×** `msr ENA, xzr` (clear) on an EL-conditional path next to `msr tpidr_el2, xzr`. Not the AIC path.

Implication for G2b: `ENA f` is necessary and **not sufficient**. Guest timer delivery is not “write ENA from m1n1.” Writing ENA from m1n1 **EL2h-SYNCs** on this t8132 (gate-revert boot). MRS of ENA is legal; iBoot already opened the gate.

---

## What this does not claim

- Not G2b PASS. `/init` has not run.
- Not Omarchy. Not AGX. Not Weston. wtsnz having AGX on an M4 Max does not move this Air.
- Not “we skipped too much kernel.” Guest exception class 783–786: `aic_handle_fiq` / `aic_handle_irq` / `irq_enter_rcu` / EL1h FIQ vector never entered after INIT.
- Not a live Macintosh HD dump yet. Next public-facing work is that volume copy + snapshot, then Ghidra on AppleAIC — not another `wdt_site`.

HV experiments after this note need a **Ghidra cite**. No folklore `wdt_site`. Do not `msr VM_TMR_FIQ_ENA_EL2` on t8132. Do not skip `copy_process` / `current->nsproxy` / `numa_default_policy`.
