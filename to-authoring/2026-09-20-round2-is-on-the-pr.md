# 2026-09-20 — round 2 is on the live PR

The 16-commit series has been force-pushed to the PR branch. It went from 15 commits to 16, and
the branch the team reviews is kept identical to it.

Everything in the two previous notes still stands. This records what changed between your drop
and what was actually published, so your next drop starts from the right base.

---

## 1. Three commit messages were amended before the push

Content is untouched — all 16 commit trees are byte-identical to the ones your patches produced,
so every build and hardware result carries over unchanged. Only messages differ.

### Commit 3 — a factual error, and the worst possible one to ship

The message said the per-SoC symbols are "sourced with **orsource**". The commit adds
`rsource "*/Kconfig.soc"`. The code was correct; the message described the state before the
round-2 change.

This is the line the reviewer commented on. Publishing a message that still said `orsource`,
alongside a reply explaining that we had changed it to `rsource`, would have read as not having
understood the comment. Worth a check on the next drop: when a change answers a review comment,
grep the commit message for the old term.

### Commits 4 and 5 — vendor citations removed

Both cited another vendor's SoC tree as precedent for the layout. **We cannot reference other
silicon vendors in anything we publish** — not commit messages, not PR comments, not replies.

This is a constraint on us, not a criticism of the reasoning; the precedent was the strongest
argument for the structure and losing it is a real cost. For future drops: make the case on its
own terms — why the structure is right, not who else uses it. If a precedent is what convinces
you, put the reasoning in the handover, where it helps us decide, and keep it out of the commit.

### Commit 5 — gained the claim it was missing

It now carries `No functional change: all five DSP targets produce byte-identical images`,
matching its sibling commits. You deliberately left it off because you had not built anything;
that was the right call. It has now been measured, so the claim is made on evidence.

---

## 2. What is on the PR

16 commits, 103 files. Base unchanged.

Verified before pushing: tree hash reproduced yours exactly, 16 sign-offs (one per commit, after
the `--signoff` rebase), 12 `Co-authored-by:`, no `Assisted-by:`, compliance clean including
`KconfigHWMv2`, **21/21 per-commit builds**, **byte-identical loadables on all five audio DSP
targets**, the `DT_HAS_` gating discriminating in both directions, and **9/9 hardware on both
Genio boards** on the shipped image with unpatched cells.

`Gitlint` and `Checkpatch` were re-run after the amendments, since those read the messages. Both
clean.

Still build-verified only: **no audio DSP has been booted.** The moved linker script and the
`adsp/soc.h` include chain have not been exercised at runtime.

---

## 3. Still needed from you, third time of asking

The include-path fix is in the published series but **not in your patches**. Without it every
audio DSP target fails to compile, so the next drop reintroduces a series that does not build:

```cmake
if(CONFIG_SOC_MT8186_ADSP)
  # soc.h, included by arch/xtensa/irq.h, lives in the cluster dir.
  zephyr_include_directories(adsp)
```

in each of the five per-SoC `CMakeLists.txt`, inside the existing ADSP guard, in the commit that
moves the shared files into `common/adsp`. The reason it is needed: only the family directory is
on the include path automatically, so once `soc.h` moves down a level nothing can resolve it.

Please confirm this is picked up, or say if you would rather it stayed a local fix applied on
this side each time.
