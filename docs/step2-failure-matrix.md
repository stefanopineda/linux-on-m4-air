# Step2 failure matrix — M4 Air j715ap / 26.5.1 (25F80)

> **Later outcome (keep this in mind):** G1 on this Air did **not** come from the stub path documented below. It came from a **full second macOS (AsahiHost)** + `kmutil` on that volume (2026-09-05 20:49). G2 Linux banner happened 2026-09-06. G2b userspace still open. This file remains the log of the **stub 1TR** dead ends so nobody repeats them.

Lab: 15" MacBook Air `Mac16,13` / `j715ap` / T8132 `0x8132` / board `0x2E`.  
Stage1 `m1n1.bin` `0dbde320…`. `boot.bin` `4fb2ee86…`. Volume group IDs omitted on purpose.  
Firmware freeze: **do not** Reinstall Tahoe / bump past 25F80.

This file is the log of **failed and confirmed-dead knobs** on the way into G1 (`kmutil configure-boot` / `coih`).  
0xSero’s mini (`j773`, same T8132) **did** get an authorized m1n1 stub. T8132 can take fuOS. **This Air’s 26.5.1 boot/recovery path will not give the stub’s paired 1TR**, and every workaround we tried either panics, yellows, or hits SEP `pairing (17)`.

Tracking before this file: scattered `CLAUDE.md` notes + `evidence/step2/*.log` + panic dump. Not a matrix. That is on us.

Escape from yellow, every time: **Startup Disk → Macintosh HD**. Never Recovery. Never Reinstall.

---

## Why this is slower than 0xSero

| | 0xSero mini | This Air |
|---|---|---|
| SoC | T8132 | T8132 |
| Board | `j773*` | `j715ap` / `0x2E` |
| What he proved | Authorized stub (G1). Then HV RAM Linux + tg3 SSH. NVMe/DCP still open. | **At the time of this matrix:** stub on disk, IPSW identity matched, `boot.bin` in place, stub `coih` absent. **Later:** G1 via AsahiHost, not via this stub. |
| macOS at install | Not 26.5.1/25F80 (his ANS/DCP work predates our freeze). | **26.5.1 (25F80)** — picker + Full Security verify of an incomplete stub. |
| Step2 | Stock Asahi: bless stub default → hold power → picker **is already stub 1TR** → `.IAPhysicalMedia` launches `step2.sh` → `bputil -nc` + `kmutil`. | Same recipe **never reaches the picker**. iBoot treats the stub as a broken macOS and shows yellow. |

The stub path never yielded m1n1. Fumbling was 100% at **paired 1TR**. G1 later used a complete second macOS (F6).

---

## The catch-22 (load-bearing)

Apple Silicon since macOS 12:

- `bputil -nc` (Permissive) and `kmutil configure-boot` (fuOS / `coih`) require **1TR paired to that OS’s VGID**.
- Hold-power 1TR is the recoveryOS of the **default** boot OS.
- Unpaired recovery (Macintosh HD’s 1TR) **cannot** Permissive-downgrade another VGID. SEP: `pairing (17)`.

On **this** 26.5.1 machine, adding:

- A 2.5 GB stub is Full Security, no SSV, no fuOS.
- Making it default makes iBoot **verify it as macOS** before showing the picker.
- Verify fails → yellow. Booting it as OS → XNU panic `rootvp not authenticated after mounting @bsd_init.c:979`.
- So: **need stub as default to get its 1TR; stub as default cannot show 1TR.**

Asahi’s `.IAPhysicalMedia` trick (hide SystemVersion, picker launches Finish Installation.app inside the stub’s already-running 1TR) assumes the picker appears. It does not.

---

## Historical snapshot (2026-09-05, after E7 — before AsahiHost G1)

| Knob | Value |
|---|---|
| Default boot | Macintosh HD — do not bless stub (C1b later dead) |
| Running recovery when E7 ran | Macintosh HD 1TR (`one true recoveryOS`, **Not Paired** to stub) |
| Stub pairing | **Not Paired** (expected from this recovery) |
| Stub security | **Reduced** (`smb0: 1`) — E7 **PASS** |
| `coih` | **absent** |
| GPT | ISC \| Macintosh HD ~186 GB \| m1n1 proxy 2.5 GB \| free ~56.5 GB \| Recovery last |
| Finish Installation partition | **deleted** |
| Stub SystemVersion | **live** (`26.5.1 (stub)`) |
| Stub Preboot kernelcache | **live** (restored after hide made verify worse) |
| `.IAPhysicalMedia` on stub | **disabled** |

---

## Knob chart

Legend: **Dead** = reproduced, do not retry. **Undone** = we reverted. **Open** = not tried yet. **Trap** = looks like the Asahi docs, actually worse.

### A. Accessory files on the stub (still in macOS)

| # | Knob | What we set | Result | Status | Evidence |
|---|---|---|---|---|---|
| A1 | `.IAPhysicalMedia` ProductVersion | Upstream **12.1 / 00A191** as shipped | Picker ignores installer media; treats stub as OS | Dead | First yellow after stage1 |
| A2 | Same, rewritten **26.5.1 / 25F80** | Match running OS | Still ignored. Stub is an APFS OS volume group; 26.5.1 classifies **OS** first | Dead | `fix-proxy-accessories.log` |
| A3 | Hide System `SystemVersion.plist` (`prepare_for_step2`) | Asahi stock | Necessary for installer-media path **if** picker honors IAPM. Here picker never does | Trap | Asahi `stub.py` |
| A4 | Hide Preboot `restore/SystemVersion.plist` + `RestoreVersion.plist` + restore kernelcache | Hypothesis: picker reads Preboot restore as “real 26.5.1” | Did not make IAPM win. Stub still listed as OS | Dead / undone | `fix-proxy-accessories.log` |
| A5 | Hide Preboot `boot/.../kernelcache` | Stop XNU boot if stub is clicked | **Made verify worse.** Recovery: “Unable to verify startup disk” after auth | Dead / **undone** (restored 06:59) | `restore-stub-for-option-click.log` |
| A6 | Copy IAPM onto Preboot VGID | Picker might look there | iSCPreboot copy SIP-blocked. Preboot copy did not change picker | Dead | `fix-proxy-accessories.log` |
| A7 | Mach-O `step2_launcher` + ad-hoc codesign + richer Info.plist | 26 might refuse shebang app | Irrelevant: app never launched | Dead | same log |
| A8 | `APFSIOC_VOL_BOOTABLE` fsctl | macOS 27 visibility flag | Already 1 on 26.5.1. Not the yellow cause | Dead | same log |
| A9 | Disable stub IAPM, restore all hidden files | Undo A3–A6 so stub looks like an OS again | Restored. Still cannot 1TR the stub | Undone → current | `restore-stub-for-option-click.log` |

### B. Extra installer volume (non-OS)

| # | Knob | What we set | Result | Status | Evidence |
|---|---|---|---|---|---|
| B1 | 256 MB APFS **Finish Installation** after stub, Recovery still last | Only IAPM 26.5.1/25F80 + Finish Installation.app. `step2.sh` finds stub by VGID | Picker treated it as a **startup disk**. Auth then **Unable to verify startup disk** | Dead | `make-installer-volume.log`; user photo path |
| B2 | Hide stub kernelcache so only installer volume is clickable | Combined with B1 | Verify of default/selected volume still failed | Dead / undone | same |
| B3 | Delete installer GPT slice | Remove fake disk from picker | GPT back to ISC / HD / stub / free / Recovery last | Done | `restore-stub-for-option-click.log` |
| B4 | **FAT32** installer volume (Asahi: IAPM works on FAT32) | Never created | **Open.** APFS B1 failed; FAT32 cannot look like an APFS OS VG. **Does not solve pairing:** with Macintosh HD default, IAPM would launch the app **inside Macintosh HD 1TR** → still `pairing (17)` | Open / low value | — |

### C. Who is default boot

| # | Knob | What we set | Result | Status | Evidence |
|---|---|---|---|---|---|
| C1 | `bless --setBoot --mount "/Volumes/m1n1 proxy"` from macOS | Asahi stock after stage1 | Default = stub. Next boot (hold or not) → **yellow**, no three-icon picker | Dead (reproduced **3 times**: after install, after accessory fix, after Recovery bless+shutdown 2026-09-05 ~07:40 and again when user retried) | nvram `boot-volume` pointed at stub; user report |
| C1b | Same bless **after E7 Reduced** (`smb0: 1`), from Macintosh HD 1TR; `getBoot` = `/dev/disk2s2` = `m1n1 proxy` | Hypothesis: yellow was Full Security verify; Reduced might show picker | **Yellow again.** Reduced does not skip iBoot verify of an incomplete OS. Hold-until-icons also shut the machine down; release on “Loading startup options…” | **Dead** | user report + photo 2026-09-05 after `bless --getBoot` `/dev/disk2s2` |
| C2 | Same bless from **Macintosh HD 1TR Terminal** | Thought Recovery bless would be cleaner | Same yellow, no picker | Dead | user report after `bless` + `shutdown -h now` |
| C3 | Startup Disk → Macintosh HD (yellow escape) | Required to get macOS back | Works. Default = Macintosh HD | Live | `bless --getBoot` Macintosh HD |
| C4 | Hold Option + **Always Use** on m1n1 proxy at picker | Apple Help “set default startup volume” | **Not tried.** Would set stub default then auto-restart → predicted yellow/panic. Do not | Open / predicted dead | Apple Help mchl82829c17 |

### D. What you click on the three-icon picker (Macintosh HD default)

Picker **does** appear when Macintosh HD is default: **m1n1 proxy | Macintosh HD | gear**.

| # | Knob | Result | Status | Evidence |
|---|---|---|---|---|
| D1 | Continue on **m1n1 proxy** | iBoot2 loads stub XNU. Panic **`rootvp not authenticated after mounting @bsd_init.c:979`**. Falls back to Macintosh HD login + “restarted because of a problem” | Dead | `evidence/step2/panic-rootvp-not-authenticated.panic` 07:09 |
| D2 | Hold **Option** + Continue on **m1n1 proxy** | Same as D1. Option on this row does **not** mean paired recovery. (Option+**Always Use** is “make default”, C4.) | Dead | user session; panic |
| D3 | Continue on **Macintosh HD** | Boots macOS. Useless for step2 | Dead | — |
| D4 | **Gear**, then Continue | Macintosh HD **1TR**. User list **with no “Select a volume to recover”** on this machine. Confirmed twice | Live path into **wrong** recovery | user report |
| D5 | Gear, then Option-click stub on “select a volume to recover” | **That screen never appears** (gear → user list directly). Cannot execute | Dead (UI missing) | user report, twice |

### E. Inside Macintosh HD 1TR (gear → admin auth → 4 tiles)

`bputil -d` on the stub VGID from this recovery: **OS Type: one true recoveryOS**, **OS Pairing Status: Not Paired**.

| # | Knob | Result | Status | Evidence |
|---|---|---|---|---|
| E1 | `"/Volumes/m1n1 proxy/step2.sh"` without quotes | `/Volumes/m1n1: No such file or directory` (space) | Trap | user report |
| E2 | Same, quoted, stub mounted | Script runs. Prints our patched “Not Paired / do not bless” and stops | Expected | user photo 07:23 |
| E3 | `bputil -nc` on the stub VGID (Permissive + disable CTRR) | Password **accepted**. Then `BYErrorDomain Code=401` “Failed to create local policy”, underlying `com.apple.bootpolicy Code=17 "pairing (17)"` | Dead | user photo |
| E4 | `kmutil configure-boot … -v "/Volumes/m1n1 proxy"` from this 1TR | Not run after E3 (would fail the same pairing). From **full macOS**: `configure-boot must be run from macOS Recovery` | Dead from macOS; unpaired 1TR predicted dead | ssh 07:11 |
| E5 | Startup Security Utility → Macintosh HD | Unlock works. Accessories “Always Allow”. Irrelevant | — | user report |
| E6 | SSU → **m1n1 proxy** | Visible as “macOS 26.5.1”. **Security Policy grayed:** “must be the startup disk”. Accessories Always Allow. Option-click does nothing extra | Dead from this recovery | user report |
| E7 | `bputil -g` on the stub VGID (**Reduced only**) from Macintosh HD 1TR | Password accepted. Policy update succeeded. Stub local policy: **Security Mode Reduced (`smb0: 1`)**. `coih` still absent. Running recovery still **Not Paired**. Helper script missing in this recovery (Data path not mounted); raw `bputil -g` worked | **PASS** | User photo 2026-09-05 Recovery Terminal |
| E8 | `bputil -k` / `-s` (kexts / disable SSV) from unpaired 1TR | Not tried. `-s` is Permissive-class; likely pairing (17) | Open / predicted dead | bputil help |

### F. Things we did **not** try (and should not silently skip)

| # | Knob | Why it is still listed | Do not / do |
|---|---|---|---|
| F1 | `kmutil configure-boot` on **Macintosh HD** | Would replace the macOS kernel. DFU territory | **Never** |
| F2 | Reinstall macOS / yellow Recovery tile | Firmware drift off 25F80 | **Never** |
| F3 | DFU revive/restore | Wipes; breaks freeze | Not unless owner says so |
| F4 | Double-press power (fallback recoveryOS) | Ordinary recovery, **not** 1TR. `step2.sh` would reject “one true recoveryOS” | Low value |
| F5 | Installer run from Recovery Terminal (Asahi expert path) | Creates stub already in some recovery; can land Reduced from unpaired. We already have a stub; this would be a **re-install of stage1 from 1TR** | Open, high cost |
| F6 | Second **complete** macOS 26.5.1 (`25F80`) in the free space, then 1TR **that** volume and `kmutil` m1n1 **there** | Hollow stub cannot be default without yellow. A real OS can. Pairing follows the complete volume, not the 2.5 GB stub | **PASS 2026-09-05 20:49** (AsahiHost). This is G1. |
| F7 | Seal/SSV the stub System volume | Would need Apple’s sealer. We cannot fake `rootvp` auth | Not feasible |

---

## Errors, verbatim

| Where | Text |
|---|---|
| Yellow (stub default or stub selected as OS) | “The version of macOS on the selected disk needs to be reinstalled. Use Recovery to reinstall macOS or select another startup disk.” |
| After auth on fake installer / hollowed stub | “Authentication is required to verify startup disk” → “Unable to verify startup disk” |
| XNU boot of stub | `panic(cpu 6 caller …): rootvp not authenticated after mounting @bsd_init.c:979` — Darwin 25.5.0 `RELEASE_ARM64_T8132`, iBoot `mBoot-18000.120.36` |
| `bputil -nc` from Macintosh HD 1TR | `BYErrorDomain Code=401 "Failed to create local policy"` / `com.apple.bootpolicy Code=17 "pairing (17)"` |
| SSU on stub from Macintosh HD 1TR | Security Policy grayed: must be the startup disk |
| `kmutil` from full macOS | `configure-boot must be run from macOS Recovery` |

---

## What we think is actually true (2026-09-05)

1. Stage1 (GPT stub, IPSW 25F80 identity, `boot.bin`, bless once) **worked**.
2. 26.5.1 on j715ap **does not** honor `.IAPhysicalMedia` on an APFS OS volume group.
3. 26.5.1 **will not show the boot picker** when the incomplete stub is default; it yellows instead. **C1b: still yellow after Reduced.**
4. Macintosh HD 1TR is easy (gear). It is the **wrong pairing** for `bputil -nc` / `kmutil` / SSU Security Policy.
5. Option on the **first** picker row boots the selected OS (D1/D2). It is not “enter that OS’s recovery.”
6. Hiding Preboot kernelcache (A5) was our own foot-gun.

---

Astra (OpenRouter `openai/gpt-6-astra`, 2026-09-05, $0.049): **APPROVE C1b once as diagnostic**, not as a likely fix. Reduced does not disable SSV/`rootvp` auth; yellow vs picker is a different observation than the XNU panic. If picker appears, still verify **Paired** to this VGID before `-nc`/`kmutil`. Yellow again ends C1b. HV still blocked until an accepted m1n1 boot path exists (`coih` is authorization, not USB magic). Full text: `evidence/step2/astra-e7-review.json`.

## What happened next (F6 — done)

**C1b is dead.** Do not bless the 2.5 GB stub as default again.

**F6 completed 2026-09-05 20:49:** complete 26.5.1 (`25F80`) as volume **AsahiHost**, 1TR **that** OS, Permissive + `kmutil` m1n1 there. That is G1.

Notes from the install (2026-09-05 ~19:30 local):

- New GPT slice, volume **AsahiHost**, Recovery still last.
- Fetched `/Applications/Install macOS Tahoe.app` via `softwareupdate --fetch-full-installer --full-installer-version 26.5.1`. Payload **Build 25F80**. Do not use 26.6.x from the catalog.
- `startosinstall --volume /Volumes/AsahiHost` on this Tahoe IA **prints usage and exits 15** (`--volume` is in the binary strings but not accepted by the CLI). Do not `--eraseinstall`.
- GUI installer (`InstallAssistant`). Destination must be **AsahiHost**, never Macintosh HD.

---

## Contribute this

Useful to `#asahi-dev` / installer issues as: **j715ap + 26.5.1 (25F80) cannot complete Asahi step2** because default-stub never reaches 1TR picker (yellow / `rootvp not authenticated`) and foreign 1TR cannot `bputil -nc` (`pairing (17)`). Stock IAPM 12.1 and 26.5.1-matched IAPM both ignored. APFS non-OS IAPM volume still hits verify-startup-disk.
