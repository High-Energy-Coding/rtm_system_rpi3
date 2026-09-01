# rtm_system_rpi3

[![CI](https://github.com/High-Energy-Coding/rtm_system_rpi3/actions/workflows/ci.yml/badge.svg)](https://github.com/High-Energy-Coding/rtm_system_rpi3/actions/workflows/ci.yml)

A Nerves system for the Raspberry Pi 3 (including the 3A+), maintained by
High-Energy-Coding for the Read to Me office hub.

It is a fork of [nerves-project/nerves_system_rpi3][upstream] cut from tag
**v2.1.1**, and it carries exactly **one** change from upstream.

[upstream]: https://github.com/nerves-project/nerves_system_rpi3

## The one change

```diff
--- a/linux-6.18.defconfig
+++ b/linux-6.18.defconfig
 CONFIG_USB_ACM=m
+CONFIG_USB_PRINTER=y
 CONFIG_USB_STORAGE=y
```

That's it. Everything else — Buildroot config, fwup config, kernel, rootfs
overlay — is upstream v2.1.1 verbatim, apart from the package rename and the CI
changes described below.

### Why

The office hub is a Raspberry Pi 3A+ whose entire job is driving a USB thermal
printer. `CONFIG_USB_PRINTER` builds the kernel's `usblp` driver, which is what
exposes the printer at `/dev/usb/lp0`. No official Nerves system enables it.

**Why built in (`=y`) and not a module.** Nerves is perfectly capable of loading
modules — `CONFIG_MODULES=y` is set, busybox ships `modprobe`, and
`nerves_uevent` autoloads drivers on hotplug. The reason is simpler: one less
moving part on a device whose only job is printing. The printer is present at
boot and is never optional, so there is nothing to gain from deferring it.

**Why `rpi3` and not `rpi3a`.** The 3A+ has a single USB port.
[`nerves_system_rpi3a`][rpi3a] runs that port in **gadget** mode (the board
pretends to be a USB device so you can plug it into a laptop), which means it
cannot host a printer at all. `nerves_system_rpi3` runs the same board in
**host** mode and is the documented base for a 3A+ that needs to talk to USB
peripherals.

[rpi3a]: https://github.com/nerves-project/nerves_system_rpi3a

### Position in the defconfig

`CONFIG_USB_PRINTER` sits between `CONFIG_USB_ACM` and `CONFIG_USB_STORAGE`
because that is where `savedefconfig` puts it: the kernel lists `USB_PRINTER`
directly after `USB_ACM` in `drivers/usb/class/Kconfig`, and sources
`drivers/usb/storage/Kconfig` next. Keeping savedefconfig order means upstream
kernel bumps merge cleanly instead of conflicting on line placement.

## Using it

This system is not published to Hex. Depend on it by git tag; because the repo
is **public**, the prebuilt artifact downloads from GitHub Releases with no
token in the consuming build.

```elixir
# mix.exs of the Nerves app
def deps do
  [
    {:rtm_system_rpi3, github: "High-Energy-Coding/rtm_system_rpi3",
     tag: "v0.1.0", runtime: false, targets: :rpi3}
  ]
end
```

Then `export MIX_TARGET=rpi3` and build as usual.

Verify the printer on a running device:

```
iex> File.stat!("/dev/usb/lp0")
iex> File.write!("/dev/usb/lp0", "hello\n")
```

## Board reference

| Feature        | Description                              |
| -------------- | ---------------------------------------- |
| CPU            | 1.2 GHz quad-core Cortex-A53 (ARMv8, 32-bit userland) |
| Memory         | 512 MB (3A+) / 1 GB (3B, 3B+)            |
| Storage        | MicroSD                                  |
| Linux kernel   | 6.18 w/ Raspberry Pi patches             |
| Buildroot      | 2026.05.1 (via `nerves_system_br` 1.34.1)|
| Erlang/OTP     | 29.0.4                                   |
| USB            | Host mode (this is the point)            |
| IEx terminal   | HDMI and USB keyboard (changeable to UART) |
| GPIO, I2C, SPI | Yes — [Elixir Circuits](https://github.com/elixir-circuits) |
| Ethernet       | 3B/3B+ only                              |
| WiFi           | Yes                                      |
| Audio          | HDMI / stereo out                        |

For anything not covered here — WiFi setup, Bluetooth, camera, the 2.0 firmware
validation change, kernel module details — read [upstream's README][upstream].
It applies unchanged.

## Differences from upstream, in full

1. `CONFIG_USB_PRINTER=y` in `linux-6.18.defconfig` (above).
2. Renamed: `@app :rtm_system_rpi3`, `BR2_NERVES_SYSTEM_NAME="rtm_system_rpi3"`,
   `RtmSystemRpi3.MixProject`. The repo name and `:app` must match — CI runs
   `mix nerves.artifact ${GITHUB_REPOSITORY#*/}`.
3. `artifact_sites` points at `{:github_releases, "High-Energy-Coding/rtm_system_rpi3"}`.
4. Version restarts at `0.1.0` — this fork's own line, not upstream's `2.x`.
5. CI runs on `ubuntu-latest` instead of upstream's `blacksmith-*` runners, and
   `push-to-download-site` is `false` (upstream's S3 source mirror needs
   `secrets.AWS_ROLE`, which this org does not have). No secret beyond the
   automatic `GITHUB_TOKEN` is required.

## Releasing

Push a tag and CI builds and uploads the artifact:

```sh
git tag v0.1.0 && git push origin v0.1.0
```

**The deploy job creates a DRAFT release.** Nerves' artifact resolver cannot see
drafts, so a consuming project will fail to download the artifact until you
publish it:

```sh
gh release edit v0.1.0 --draft=false
```

The release notes come from the `## vX.Y.Z` section of `CHANGELOG.md`. If that
section is missing, the deploy job fails — add it before tagging.

## Pulling in upstream changes

```sh
git remote add upstream https://github.com/nerves-project/nerves_system_rpi3.git
git fetch upstream --tags
git merge v2.2.0          # whichever upstream tag you're moving to
```

Expect conflicts in a small, predictable set of files:

- **the kernel defconfig** — re-add `CONFIG_USB_PRINTER=y` after
  `CONFIG_USB_ACM`. Note that the *filename* changes on kernel bumps
  (`linux-6.18.defconfig` → `linux-6.20.defconfig` etc.); when it does, also
  update the path in `mix.exs` `package_files` and in `REUSE.toml`.
- **`mix.exs`** — keep our `@app`, `@github_organization`, and description.
- **`nerves_defconfig`** — keep `BR2_NERVES_SYSTEM_NAME="rtm_system_rpi3"`.
- **`.github/workflows/ci.yml`** — take upstream's OTP/Elixir/bootstrap version
  bumps, keep our runner labels and `push-to-download-site: false`.
- **`README.md`** — this file.

Then bump `VERSION`, add a `## vX.Y.Z` section to `CHANGELOG.md`, tag, and
publish the draft release.
