# 2026-09-18 — round 2 (16 commits) verified on hardware; one build fix folded in

Applied, fixed, built, swept and run on both boards. Everything passes. One defect needed
fixing before anything would compile — details below, and it needs to come back into your
patches or the next drop reintroduces it.

Not pushed to the live PR branch. The series sits on a separate branch in the upstreaming repo
for human review first.

---

## 1. The one thing that was wrong: all five DSP targets failed to build

    include/zephyr/arch/xtensa/irq.h:152:10: fatal error: soc.h: No such file or directory

Commit 4 moves `soc.h` into `common/adsp/` and leaves a thin `<soc>/adsp/soc.h` that includes
it. Zephyr puts only the **family** directory on the include path automatically — the failing
compile line shows `-I.../soc/mediatek/mt8xxx` and nothing else from the SoC tree — so after the
move `#include <soc.h>` had nowhere to resolve.

The fix is the pattern already two blocks below in the same file. The A55 branch has:

```cmake
if(CONFIG_SOC_MT8188_A55)
  # pinctrl_soc.h, included by the pinctrl framework, lives in the cluster dir.
  zephyr_include_directories(a55)
```

The ADSP branch set sources and a linker script but never the include directory. Added to each
of the five per-SoC `CMakeLists.txt`, inside the existing ADSP guard:

```cmake
if(CONFIG_SOC_MT8186_ADSP)
  # soc.h, included by arch/xtensa/irq.h, lives in the cluster dir.
  zephyr_include_directories(adsp)
```

Folded into **commit 4**, since that commit both moves `soc.h` and creates those five files, so
it is the one that should have carried it. The series is bisectable with it there.

**This would have failed upstream CI**, not only a local gate — twister builds the MediaTek ADSP
platforms. It got through because none of the nine compliance checks in your §7 table compiles
anything, which your handover states plainly. Worth treating "compliance is green" and "it
builds" as separate claims for this series.

---

## 2. Results against your §8 plan

| # | Test | Result |
|---|---|---|
| 1 | Five-way ADSP gate | **pass** — zero losses, **byte-identical loadables on all five** |
| 2 | Driver symbols `y` on the five DSP targets, absent on both A55 | **pass**, verified both halves |
| 3 | A DSP booting on real hardware | **not done** — see below |
| 4 | A55 boot on both Genio boards | **pass** on both |
| 5 | `zephyr.img` still generated after the `gen_img.py` move | **pass** |

Test 1 is byte-identical across all five DSP targets, so the `No functional change: all five
DSP targets produce byte-identical images` line can now go on commits 4 and 5. You were right
to leave it off until someone had built it.

Test 2, both directions: `CONFIG_INTC_MTK_ADSP` and `CONFIG_MTK_ADSP_TIMER` are `y` on all five
DSP targets and **absent** on both A55 targets. The absence is the half that proves the
`DT_HAS_` gating discriminates rather than defaulting on.

**Test 3 was deliberately skipped.** We have never booted a DSP target here: it loads through
`mtk_adsp_load.py` rather than the hypervisor, and the MT8188 audio DSP is owned by Linux on
these images. Attempting it was judged not worth disturbing the boards for. The moved linker
script and the `adsp/soc.h` include chain are therefore **build-verified only** — that should be
stated on the PR rather than implied otherwise.

### Other gates

Both Arm board builds pass, compliance is clean apart from the expected `ClangFormat`
suggestions on the moved files, checkpatch is clean, and **every one of the 16 commits builds
individually** (21 builds). Tree hash reproduced yours exactly at `2e18d5ecc0ac` before the
sign-off rebase, and was unchanged by it. 16 sign-offs after the rebase, one per commit, 12
`Co-authored-by:` intact, no `Assisted-by:`.

---

## 3. Hardware

Full suite on both boards, every test passing:

| | Genio 700 EVK | Genio 510 EVK |
|---|---|---|
| boot, board string | pass | pass |
| `cntfrq` | 13 MHz | 13 MHz |
| `k_sleep(5s)` worst deviation | 5 ms | 5 ms |
| UART RX under interrupt load | `rx=10065 isr=10065` | `rx=10065 isr=10065` |
| runtime reconfigure incl. negative control | pass | pass |
| `uart_basic_api` / `uart_interrupt_api` | 100% / 100% | 100% / 100% |
| cell restart cycling | 6/6 | 6/6 |
| declared 8 MB window actually granted | not testable, see below | **pass**, `mismatches=0` |

The two boards are no longer on the same software. The 700 was reflashed with a newer
`rity-demo` image whose stock inmate cells still grant **2 MB** against the 8 MB the board
devicetree declares, and that reflash removed the patched cells, so the window test cannot run
there. The 510 is on the older image and kept the patched 8 MB cells, so it covers that case.
Jailhouse v0.12 on the newer image otherwise behaves identically.

Unrelated to this series, and not a regression: the 2 MB stock grant is the same gap already
known about.

---

## 4. Answers to things you raised

**The family rename to `SOC_FAMILY_MEDIATEK_MT8XXX`** — deferred, as you recommended. Confirmed
by the human, not assumed.

**The pinctrl `Co-authored-by:`** — correct as it stands and preserved. For the record of why
the count moved: your round-1 drop had 11, the commit published at the time had 12, and the
missing one was Felix on the pinctrl commit. It was restored here rather than dropped, because
removing a trailer already published is an attribution change, not a cleanup. Your reasoning at
the time was that no history showed a second author on that file; the published commit did.

**`ClangFormat` on the moved files** — left alone, as you asked. Confirmed warn-listed and
excluded from the erroring run, so it cannot fail CI.
