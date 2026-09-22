# 2026-09-22 — round 3, the UART fixes, is on the live PR

Still 16 commits. The series went from `e8199d66330` to `2224a5f69c9`: **one commit changed**,
the four after it are identical content re-parented, and the first eleven are untouched objects.
Two files, +41/-13, both in `drivers/serial`. The branch the team reviews was re-synced in the
same step.

Everything in the previous notes still stands.

---

## 1. Where the round came from

An automated reviewer left four inline comments on the UART driver. Two of them were the same
issue stated twice, so three distinct findings. **All three were real** — no false positives, and
one of them was a build break our own gates had never exercised.

### a. The configuration path was built only under an optional symbol

`uart_mtk_init()` called `uart_mtk_configure()` unconditionally, but the definition and the header
declaration both sat inside `#ifdef CONFIG_UART_USE_RUNTIME_CONFIGURE`. That symbol is `default y`
but user-settable, so `=n` is a legal configuration, and in it the driver did not compile:

```
uart_mtk_common.c:608:9: error: implicit declaration of function 'uart_mtk_configure'
```

The reviewer offered two fixes and **only one of them is right**. Selecting the symbol from our
own Kconfig would have "fixed" it by deleting the broken configuration, and the symbol's help text
says the opposite is intended: *"If this is disabled, UART controllers rely on UART driver's
initialization function to properly configure the controller."* Simply `#ifdef`-ing out the init
call would have left the controller with no baudrate programmed — a dead console.

The fix builds the configuration path unconditionally and keeps only the `uart_driver_api` members
guarded, because that struct carries them only when the option is on. Both board targets now build
with the option off, and a board running that build brings up its console normally.

We did write the `select` variant first, at the human's request, and then dropped it: the two board
defconfigs already set the symbol, `select` on a prompted symbol is what the Kconfig tips page
advises against, and — decisively — keeping it makes the fix unverifiable, because `=n` can no
longer be built to demonstrate that it works.

### b. A discarded return value

```c
ret = 0;
set_baudrate(dev, cfg);
if (ret != 0) {
```

`ret = 0` followed by testing `ret != 0` is dead code, and `set_baudrate()` returns `-EINVAL` for a
rate outside the supported list. `uart_configure()` therefore reported success for a rate it never
programmed, and the whole-struct `memcpy` at the end stored the rejected value, so `config_get()`
reported it too.

### c. Flow control accepted and ignored

The old code copied the caller's config and then forced `flow_ctrl` to `NONE`, returning success.
A caller asking for RTS/CTS got success, no flow control, and a `config_get()` that contradicted
the request. It now returns `-ENOSYS`, which `uart.h` documents for a configuration the device does
not support.

---

## 2. The important part: two field-assignment bugs, and how they hid

The corrected driver we received wrote its stored configuration field by field instead of with one
`memcpy`. Two of those writes went to the wrong field:

```c
uart_data->uart_cfg.stop_bits = cfg->parity;     /* in set_parity()    */
uart_data->uart_cfg.data_bits = cfg->flow_ctrl;  /* in set_flow_ctrl() */
```

Read on its own that looks like a `config_get()` inaccuracy. It is worse, because of aliasing:
`uart_mtk_init()` passes `&uart_data->uart_cfg` as `cfg`, so source and destination are the same
object. `set_parity()` writes `stop_bits = parity`; parity is `NONE` = 0, so `stop_bits` becomes 0,
which is `UART_CFG_STOP_BITS_0_5`. `set_stop_bits()` then reads that 0, falls through to `default:`
and returns `-EINVAL`. `uart_mtk_configure()` fails during init.

**Measured on a Genio 510, not reasoned about:**

| variant | result |
|---|---|
| the two field bugs, init return value **checked** | no console output at all |
| the two field bugs, init return value **discarded** | boots and prints normally |

So the latent fault plus the dropped return value together looked exactly like working firmware.
Fixing the return value alone would have turned a quiet inaccuracy into a board that never opens
its console; fixing the fields alone would have left the dropped return in place. **Both fixes have
to travel together**, and that is the argument to make if the shape of the fix is questioned.

The in-tree `uart_basic_api` suite is what catches this class of bug — it `memcmp`s the whole
`uart_config` against what `config_get()` returns — and it passes on the pushed series.

One process note on our side: the first attempt at that negative control used the wrong arguments
for the serial logger, captured nothing, and made both variants look dead. It was re-run properly
before any conclusion was drawn. A silent board is the standard signature of a harness mistake on
this bench, so it is worth one deliberate re-check before it is believed.

---

## 3. What was verified on the pushed tree

- both Arm board targets build, 156 KB
- compliance clean; ClangFormat warnings unchanged at 21; checkpatch **0 errors, 0 warnings**
- both targets build with the runtime-configure symbol off, and that build boots on hardware
- **9 of 9 hardware tests on the Genio 510**, including the two in-tree UART ztest suites and the
  runtime reconfigure test with its negative control

Not done, and worth saying plainly: the Genio 700 was not connected, so these changes have hardware
coverage on the 510 only. The 510 is currently running a locally modified kernel; the jailhouse
build underneath it is the shipped one, and the 8 MB window was measured as granted.

---

## 4. A force-push costs both approvals — plan drops around this

The series had collected two approvals before this push. **Both were dismissed the moment it
landed.** This is not incidental, it is configured: the branch ruleset for the upstream default
branch is readable without admin rights and sets `dismiss_stale_reviews_on_push: true` and
`require_last_push_approval: true` alongside a requirement of two approvals. Every push drops the
approvals, and the replacements must be granted *after* that push and by someone other than whoever
pushed.

Do not infer the opposite from a "changes requested" review surviving a force-push. That happened
here and led to the wrong conclusion once: GitHub's stale dismissal applies to approving reviews
only, and the older change-request is still sitting there untouched through two force-pushes.

The practical consequence for drops: **batch everything into one push.** A second push to fix
something small costs two fresh review cycles, not one comment. Workflow runs are also still gated
on a maintainer pressing approve, per push, so each push adds an unpredictable wait before any CI
result exists at all.

---

## 5. The family rename is deferred, deliberately

It is recorded here so it is not rediscovered as an open question.

Reading the diff hunks attached to the three Kconfig naming comments shows every one of them was
about the audio-DSP trait symbol, through its three spellings, and round 2 resolved them by
deleting that symbol in favour of devicetree gating. **No reviewer ever asked for the family to be
renamed.** `SOC_FAMILY_MT8XXX` is already correct: it is the uppercase of the `soc.yml` family
name, which is what the SoC porting guide requires.

If it is ever done: the name is `mediatek_mt8xxx` -> `SOC_FAMILY_MEDIATEK_MT8XXX`, because
`mediatek` is the token in `vendor-prefixes.txt` and every compatible in the series already uses
it. It needs no directory rename — a short directory holding a prefixed family name is common in
tree, on the order of 45 files. It touches about ten lines across three files, and nothing outside
`soc/mediatek` references either the symbol or the family string. Of the family names in tree
roughly 55 are vendor-prefixed and 34 are plain, so the convention supports it but does not compel
it.

It is a separate PR, and it is not a merge blocker.

---

## 6. The actual blocker is not code

The series has **no assignee**. The previous assignee stepped off on 2026-09-22 saying that adding
new targets is not their area, and nobody replaced them. Zephyr does not merge without an
assignee's approval on top of the two required approvals, so this is the long pole — slower to
resolve than any review comment, and it needs a human to ask for a maintainer on the boards and
SoCs side.
