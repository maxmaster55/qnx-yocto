# qnx-yocto

A self-contained workspace for building QNX artifacts with bitbake. Everything
here is a submodule; nothing else is needed.

| Path | What |
| --- | --- |
| `poky/` | Yocto, pinned to `scarthgap` (the only branch cloned) |
| `meta-openembedded/` | `meta-oe` + `meta-python`, required by meta-qt6 |
| `meta-qt6/` | stock Qt 6.10.3, cross-compiled for QNX via bbappends |
| `meta-qnx/` | the mechanism: classes, the aarch64le machine, examples |
| `meta-qnx-hyp/` | Raspberry Pi 5 hypervisor **host** — images, disk, board apps |
| `meta-qnx-guest/` | the **guest** image, its rootfs, and Qt |

`meta-qnx-hyp` and `meta-qnx-guest` are project layers; `meta-qnx` itself is
mechanism only and knows nothing about the Pi.

## Clone it elsewhere

```bash
git clone --recurse-submodules <this repo>
```

Already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

## Set up a build directory

`build-qnx/` is **not** tracked — its `conf/` holds absolute paths to this
machine's SDP. Recreate it with:

```bash
TEMPLATECONF=$PWD/meta-qnx/conf/templates/default source poky/oe-init-build-env build-qnx
```

> Use an **absolute** path for `TEMPLATECONF`. `oe-init-build-env` resolves a
> relative one against poky rather than the current directory.

Then in `build-qnx/conf/bblayers.conf`, add the two project layers next to
`meta-qnx` (the template only wires `meta-qnx`):

```
  ##OEROOT##/../meta-qnx-hyp \
  ##OEROOT##/../meta-qnx-guest \
```

And in `build-qnx/conf/local.conf`, point at the SDP and the source tree:

```bitbake
QNX_SDP_ROOT    = "/path/to/qnx800"
QNX_PROJECT_SRC = "/path/to/Qnx_Hypervisor_rbye"
INHERIT += "qnx-toolchain"
LICENSE_FLAGS_ACCEPTED += "qnx-non-commercial"
```

`QNX_PROJECT_SRC` is the monorepo working tree holding application sources and
the RPi5 BSP binaries. Recipes needing it are skipped cleanly when it is unset,
so a build without it still works — it just produces fewer targets.

## Build

```bash
source poky/oe-init-build-env build-qnx

bitbake qnx-ifs-hello      # a bootable IFS -- start here
bitbake qnx-host-image     # hypervisor host IFS for the Pi 5
bitbake qnx-guest-image    # the guest
bitbake qnx-host-disk      # flashable SD image carrying both
bitbake qtbase qtdeclarative  # Qt 6.10.3 for QNX, from stock meta-qt6
```

Inspect what you built without hardware:

```bash
bitbake -c dumpifs qnx-ifs-hello
less build-qnx/tmp/deploy/images/qnx-aarch64le/qnx-host-image.build
```

## Documentation

All in `meta-qnx/docs/`:

- [showcase.md](meta-qnx/docs/showcase.md) — a tour of every feature
- [getting-started.md](meta-qnx/docs/getting-started.md) — setup in detail
- [where-things-come-from.md](meta-qnx/docs/where-things-come-from.md) — the four
  sources QNX components arrive through
- [cookbook.md](meta-qnx/docs/cookbook.md) · [variables.md](meta-qnx/docs/variables.md)
- [sdp.md](meta-qnx/docs/sdp.md) — managing the SDP

## Status

Everything is verified statically — `dumpifs`, `fdisk`, ELF checks, boot-header
comparison against the makefile-built image. **Nothing has been run on hardware
yet.**
