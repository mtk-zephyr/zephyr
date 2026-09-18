# pr-handover — build and hardware results for the live PR series

Orphan branch. No Zephyr source here, and nothing on this branch is ever merged anywhere.

This is the return path for the PR A work: the authoring machine drops patches into its
`pr-handover/` directory, and results from building and running them on real boards come back
here.

**No PR numbers or links anywhere on this branch.** A reference to the PR turns into a
cross-reference comment on it, and the branches described here are temporary.

## Layout

| Path | What it is |
|---|---|
| `to-authoring/YYYY-MM-DD-<topic>.md` | dated reports, newest is the current state |
| `README.md` | this file |

## Who does what

The authoring machine writes patches and runs the checks that need no toolchain: gitlint,
checkpatch, Kconfig parsing, compliance. It has no SDK and no board.

This machine applies those patches, builds them, runs the five-way audio DSP gate, sweeps every
commit individually, and runs the hardware suite on a Genio 700 EVK and a Genio 510 EVK. It
pushes to the upstreaming repo when the human approves.

Claims in either direction are claims until the other side verifies them. A drop that says
"not built here" means exactly that.
