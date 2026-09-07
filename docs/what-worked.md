# What worked

Short list. Dates are local lab time unless `Z`.

| When | What | Why it mattered |
| --- | --- | --- |
| 2026-09-05 | G0 identity | Chip `0x8132`, board `0x2E`, freeze 26.5.1/25F80. |
| 2026-09-05 | Passwordless sudo + Reduced on **Macintosh HD** | Needed to run Apple tools. Did **not** enroll m1n1 there. |
| 2026-09-05 | IPSW 25F80 identity match for `j715ap` | Stub files could be laid down from the matching restore. |
| 2026-09-05 | `bputil -g` on the **stub** from Macintosh HD 1TR | Unpaired recovery **can** set **Reduced** (`smb0=1`). Cannot set Permissive (`pairing (17)`). |
| 2026-09-05 | **AsahiHost**: full second macOS ~52 GB | Real OS → Apple will 1TR it. Stub 2.5 GB volume would not. |
| 2026-09-05 20:49 | G1: Permissive + `kmutil` m1n1 **on AsahiHost** | USB `m1n1 uartproxy`. ADT dump. |
| 2026-09-06 | Live ADT → `t8132-j715` DTB | Panel `dcp`+`disp0`; HID MTP; NVMe ANS. |
| 2026-09-06 15:58Z | G2: Linux 7.1.9 banner on HV VUART | `idle=nop`, `nr_cpus=1`, skip teardown / locked IMP sysregs. Not kboot (~6s reset). |
| 2026-09-06 | Picker: AsahiHost last → **Right then Enter** | Focus starts on nothing; 3× Left wraps to Recovery. |
| 2026-09-07 | WDT bisection of G2b | Hang walked into `kernel_clone` / `copy_process`. Userspace not up. |

Yellow-screen escape that always worked: **Startup Disk → Macintosh HD**. Never Recovery.
