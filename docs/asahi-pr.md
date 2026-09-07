# Asahi PR recommendation — lab installer fork (2026-09-06)

Scope: local uncommitted diff in `src/asahi-installer` (branch `j715ap-t8132`) vs `origin/main` — 6 files, +145/−25, plus untracked `src/step2/step2_launcher.c`.

## Verdict: **issue only**

No code PR now. The only candidate for code contribution is the macOS 26 `RestoreBundlePath` fallback in `stub.py` — but the observed upstream failure (issues #407 → dup of **#404**, `BYErrorDomain Code=112 "Preserved restore bundle in preboot is missing"`) is a *bless-time validation* failure on an already-supported chip (M2, j413ap, macOS 26.2), not a Python `KeyError: RestoreBundlePath`. We cannot confirm our fix addresses #404's actual failure mode: we never hit Code 112 — our step2 failed earlier, at the picker. Until the lab reproduces or reasonably explains #404, a PR would be speculative.

High-value deliverable is the **failure matrix as an issue/comment** (`docs/step2-failure-matrix.md`): it documents, on hardware Asahi does not yet support, that step2's stock `.IAPhysicalMedia` recipe never reaches the picker on macOS 26.x — with reproducible dead knobs, verbatim errors, and a panic dump. That is data they cannot generate themselves (no supported M4 path exists).

## Change table

| Local change | File | Upstream-worthy | Why |
|---|---|---|---|
| CHIP_MIN_VER `0x8132` | `src/main.py` | **No** | M4 enablement without a working supported path; also depends on kernel/driver work that is not ours to gate. Asahi adds chips with the enablement, not before. |
| DEVICES j623/j624/j773g/j604/j713/j715ap | `src/main.py` | **No** | Same. Expert-only flag does not soften it; upstream explicitly refuses unsupported devices. |
| IPSW 26.5.1 / 25F80 entry | `src/main.py` | **No** | Lab freeze pin (do-not-bump-past-25F80 is a lab constraint, not upstream policy); upstream picks its own min versions. |
| `RestoreBundlePath` → `restore` fallback | `src/stub.py` | **Maybe, later** | Only real gift. Matches the documented Tahoe behavior (bless2 `bootcaches.plist` dropped the key; live Preboot uses `<vgid>/restore/`). But upstream symptom is Code 112, not KeyError — needs verification against #404 first. Small (~15 lines incl. `restore_bundle_relpath()` + `_restore_bundle_dir()`), lab-hack-free. Keep out of any allowlist PR. |
| `LAB_NOSHUTDOWN` env | `src/main.py` | **No** | Lab automation hook; upstream would want nothing of the sort. |
| `EXPERT=1` env | `src/main.py` | **No** | Non-interactive convenience for SSH bootstrap; upstream deliberately prompts. |
| IAPM plist 26.5.1/25F80 | `src/step2/IAPhysicalMedia.plist` | **No (as-is)** | Hardcoding the host build is wrong upstream (every user's OS differs); correct fix would be dynamic stamping — but A2 showed 26.5.1 ignores IAPM on an APFS OS VG entirely, so the mechanism itself is dead on 26.x. Issue material, not code. |
| step2.sh VGID finder | `src/step2/step2.sh` | **Marginal** | Finding the stub by VGID instead of `$0`-relative path is more robust and harmless — but it only matters in the installer-volume flow (B1) that is dead on 26.x. Could ride along with a verified RestoreBundlePath PR if maintainers want it; do not lead with it. |
| Mach-O `step2_launcher` | `src/step2/step2_launcher.c` + Info.plist | **No** | Hypothesis fix for "26 refuses shebang app" — never launched, never validated (A7: dead). Delete or keep lab-only. |

## Expected Asahi reaction

- M4 installer enablement PRs without a working supported path: rejected; enablement follows kernel/DT/firmware work in the Linux tree, and they have an explicit policy against upstreaming AI-tainted kernel work.
- `RestoreBundlePath` fallback: plausible welcome on **already-supported** chips hitting macOS 26 — but only if it demonstrably fixes #404's symptom (Code 112 at `bless --setBoot`), is ~10 lines, and contains zero lab hacks (no env vars, no hardcoded builds).
- The failure matrix: good issue content for `#asahi-dev` / installer tracker — "26.x picker ignores `.IAPhysicalMedia` on APFS OS volume groups; default-stub never reaches 1TR (yellow / `rootvp not authenticated`); foreign 1TR `bputil -nc` → `pairing (17)`". Frame as observation + repro, ask whether the installer-volume (non-OS VG) approach or F6 (complete second OS) is the intended 26.x path.

## If we ever PR (human steps)

1. Fork `AsahiLinux/asahi-installer` on GitHub as `stefanopineda`.
2. Topic branch off upstream `main` (not our `j715ap-t8132`); cherry-pick only the `stub.py` restore-bundle change.
3. One concern per PR: RestoreBundlePath fallback alone. **Never** include the j715ap allowlist, IPSW entry, or any env-var hooks in the same PR.
4. Before opening: reproduce or explain #404 (`BYErrorDomain Code=112`) on a supported chip with macOS 26; if our fix doesn't touch that path, don't open a PR — comment on #404 instead.
5. Sign-off: Stefano Pineda; conventional commit message; link #404.
6. `gh pr create` only with explicit user approval; never push from this lab session.

## Runbook pointer

`docs/step2-failure-matrix.md` is the artifact for an **issue**, not a code PR: knob chart A–F, verbatim errors, panic dump reference, and the "Contribute this" section drafted for `#asahi-dev`.
