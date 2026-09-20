# 2026-09-19 — the 8 MB grant has shipped; round 2 re-verified on both boards

Supersedes the board and memory-window parts of `2026-09-18-round2b-verified-one-fix-needed.md`.
The build and code findings in that note still stand, including the include-path fix, which still
needs folding back into your patches.

**Nothing has been pushed to the live PR branch.** It still carries review round 1. The 16-commit
round 2 series sits on a separate branch in the upstreaming repo awaiting human review.

---

## 1. The 8 MB inmate window is now in the shipped image

Both boards were reflashed on 2026-09-18 to `rity-demo` 26.1-dev, kernel
`6.6.147-mtk+gc08b30748656-ge45256210b65`, jailhouse `v0.12 (377-g96c1af0e-dirty)`. The stock
cell configs that ship with it grant the full 8 MB:

| cell | inmate window |
|---|---|
| `genio-700-evk-zephyr` | **8 MB** |
| `genio-510-evk-zephyr` | **8 MB** |
| `genio-700-evk-zephyr-afe` | **8 MB** |
| `*-zephyr_rpmsg` | 2 MB — still short, unrelated to this series |

**This closes a real defect in the board documentation.** Both board pages state that Zephyr is
given an 8 MB window and then tell the reader to create the cell from
`/usr/share/jailhouse/cells/<board>-zephyr.cell`. Until this image that file granted 2 MB, so
anyone following the documentation on a non-trivial image got a cell that died before the console
existed. As of this image the documentation is accurate as written, with nothing patched locally.

Any standing caveat along the lines of "this depends on a jailhouse change that has not landed"
can come out of the PR description and the replies. It has landed.

**Watch out for two different `6.6.147` builds.** The earlier `…-g2b4144a67b4d` granted 2 MB; the
current `…-ge45256210b65` grants 8 MB. Measurements taken before 2026-09-18 are not wrong, they
are from the earlier build. Compare the full kernel string before concluding a measurement is
stale.

---

## 2. Round 2 re-verified on hardware, on shipped cells

Both reflashes removed the board-side working directory, which had held our locally patched
cells. It was recreated with only the setup script — **no patched cells** — so this run tests
what a user actually gets.

| | Genio 700 EVK | Genio 510 EVK |
|---|---|---|
| boot, board string | pass | pass |
| `cntfrq` | 13 MHz | 13 MHz |
| `k_sleep(5s)` worst deviation | 4 ms | 5 ms |
| UART RX under interrupt load | `rx=10065 isr=10065` | `rx=10065 isr=10065` |
| runtime reconfigure incl. negative control | pass | pass |
| `uart_basic_api` / `uart_interrupt_api` | 100% / 100% | 100% / 100% |
| cell restart cycling | 6/6 | 6/6 |
| declared 8 MB window actually granted | **pass** | **pass** |

**9/9 on both boards**, on identical software, with unpatched cells. The previous run reached
8 MB only on the 510 and only because we had patched its cells; this one does not depend on us
having touched the boards at all.

Jailhouse `377-g96c1af0e` brings the inmate cell up, hands over the A55 core, muxes the UART
pins, delivers interrupts and survives six create/destroy cycles, exactly as the older build did.

---

## 3. Unchanged from the previous report

Still true, not re-run:

- tree hash reproduced yours exactly at `2e18d5ecc0ac` before the sign-off rebase, unchanged by it
- 16 sign-offs after the rebase, one per commit; 12 `Co-authored-by:`; no `Assisted-by:`
- compliance clean apart from the expected `ClangFormat` suggestions on the moved files
- **21/21 per-commit builds** — every commit in the series builds on its own
- **byte-identical loadables on all five audio DSP targets**
- the `DT_HAS_` gating discriminates in both directions: the two driver symbols are `y` on all
  five DSP targets and **absent** on both A55 targets

Still build-verified only: **no audio DSP has been booted on hardware.** The moved linker script
and the `adsp/soc.h` include chain have not been exercised at runtime, and the PR should say so
rather than imply otherwise.

---

## 4. Still needed from your side

The include-path fix from the previous report has not been confirmed as picked up. Without it the
next drop reintroduces a series where all five DSP targets fail to compile. Restated:

```cmake
if(CONFIG_SOC_MT8186_ADSP)
  # soc.h, included by arch/xtensa/irq.h, lives in the cluster dir.
  zephyr_include_directories(adsp)
```

in each of the five per-SoC `CMakeLists.txt`, inside the existing ADSP guard, in the commit that
moves the shared files into `common/adsp`.
