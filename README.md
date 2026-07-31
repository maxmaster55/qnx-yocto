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

The generated `local.conf` lists the rest, commented out, with the wifi
credentials file (`QNX_HOST_CONF_WIFI`) and the board's network addresses among
them. [configuration.md](meta-qnx/docs/configuration.md) has the full set —
including the handful of values that appear in more than one file and have to
agree, which is where these images have historically gone wrong.

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
bitbake -c dumpifs   qnx-ifs-hello    # what ended up in the image
bitbake -c dumpbuild qnx-ifs-hello    # the build file that asked for it
```

`dumpbuild` works on any IFS, rootfs or disk target and does not build the image
first, so it also works when the build is what failed. When something is missing
from an image, that file is where the reason is.

## Documentation

All in `meta-qnx/docs/`:

- [showcase.md](meta-qnx/docs/showcase.md) — a tour of every feature
- [getting-started.md](meta-qnx/docs/getting-started.md) — setup in detail
- [configuration.md](meta-qnx/docs/configuration.md) — every setting, where it
  lives, and **which values have to agree with each other**
- [adding-a-recipe.md](meta-qnx/docs/adding-a-recipe.md) — new software into an
  image, end to end
- [where-things-come-from.md](meta-qnx/docs/where-things-come-from.md) — the four
  sources QNX components arrive through
- [cookbook.md](meta-qnx/docs/cookbook.md) — recipe text per build system ·
  [variables.md](meta-qnx/docs/variables.md) — full variable reference
- [sdp.md](meta-qnx/docs/sdp.md) — managing the SDP

## Status

Runs on a Raspberry Pi 5. The host boots, Screen comes up on the panel, and
`gles2-gears` renders at 60 FPS; SPI publishes `/dev/io-spi`; the guest launches
under `qvm` with its paravirtual GPU.

Wifi is the one part still being brought up — the driver, firmware and
configuration are all in place and match QNX's own reference, but it has not
associated yet.

Everything is also verified statically — `dumpifs`, `fdisk`, ELF checks, and
boot-header comparison against the makefile-built image.
