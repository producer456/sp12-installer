# SP12 Linux installer image (Surface Pro 12in, 1st Ed — `qcom,x1p42100`)

A dd-able USB image that installs **Ubuntu 26.04 arm64** on a Microsoft Surface Pro
12-inch (1st Edition) and then brings the hardware up in one command.

> ## ⚠ This image has never been booted on hardware other than the machine that built it — and it cannot be tested there
>
> The reference machine boots from an internal partition, so it is structurally
> unable to test its own installer. **You are the first.** Please treat this as an
> untested artifact: use a spare USB stick, expect to need a recovery path, and do
> not point it at a machine holding data you care about.
>
> Everything in it is assembled from parts that *are* proven on the reference
> machine — the same kernel, the same device tree, the same firmware set, the same
> fix scripts. What is unproven is the **installer path itself**: whether a fresh
> install lands the boot chain correctly on a machine that has never had one.

## Credit

The hard part is not this image. It is [**harrisonvanderbyl/surface-pro-12-inch-linux**](https://github.com/harrisonvanderbyl/surface-pro-12-inch-linux) —
the device tree, the firmware set, and the driver/pen/keyboard work that was
upstreamed into Linux 7.2-rc1 (with the dts in linux-next). Without the dtb from
that repo this machine does not usefully boot at all. This image is a packaging
convenience on top of that work, and it ships the community repo inside it.

## What's in it

- Stock Ubuntu 26.04 arm64 live installer (casper), unmodified
- Mainline **7.2.0-rc3** kernel `.deb`s. The Type Cover keyboard depends on this:
  7.0.x distro kernels drop every keypress (no board entry in
  `surface_aggregator_registry`), so the boot entry pins this kernel by filename.
- The community device tree + the Snapdragon GRUB workarounds
  (`cutmem` with a lockdown guard, `clk_ignore_unused pd_ignore_unused arm64.nopauth`)
- A firmware snapshot (ADSP/CDSP/GPU/battery manager) and the audioreach DSP modules
- `postinstall.sh` — one command after install: audio UCM fix + sink watchdog,
  cpufreq/SCMI DVFS, speaker EQ, sensors, WiFi, boot chain

## Get it

The image is split into three parts because GitHub caps release assets at 2 GiB.

```sh
# download part0..part2 and SHA256SUMS.txt from the release, then:
sha256sum -c SHA256SUMS.txt
cat sp12-installer.img.zst.part0 \
    sp12-installer.img.zst.part1 \
    sp12-installer.img.zst.part2 > sp12-installer.img.zst
zstd -d sp12-installer.img.zst          # -> sp12-installer.img (7.3 GiB)
```

Expected hashes after reassembly:

| file | sha256 |
|---|---|
| `sp12-installer.img.zst` | `091200fbebf3cc43bf25ad07f0a11881d684f4fde4ad9ce64af4b11152d02a4a` |
| `sp12-installer.img` | `5fe2892a340c48b72da955e4f7dc734aa7740975b1a250d52244a667fd565fc9` |

Write it to a stick of **8 GB or more** — this erases the stick:

```sh
sudo dd if=sp12-installer.img of=/dev/sdX bs=4M status=progress conv=fsync
```

## Known hazards, stated up front

- **Suspend hard-hangs this hardware on resume.** A sensor power domain poisons the
  resume path — see [issue #6](https://github.com/harrisonvanderbyl/surface-pro-12-inch-linux/issues/6).
  `postinstall.sh` therefore leaves idle auto-suspend **off**. Recovery from a hang is
  a forced power-off, so save your work before testing suspend deliberately. On the
  reference machine the workable substitute is locking on cover-close instead of
  suspending at all.
- **Hardware video decode (`qcom-iris`) does not work.** The firmware is rejected at
  the TrustZone PAS stage. Video decodes on CPU, which costs battery.
- **The internal microphone records digital zeros.** The capture path is not brought
  up; the source looks healthy in `wpctl`, so believe the silence.
- **`update-grub` will make the machine unbootable.** It regenerates a config without
  the devicetree line and the Snapdragon kernel arguments. The install keeps its own
  `sp12.cfg` for this reason.

## Feedback

Boot-test reports are the entire point of this release — please open an issue here,
or comment on
[surface-pro-12-inch-linux#3](https://github.com/harrisonvanderbyl/surface-pro-12-inch-linux/issues/3),
where this image was offered. What is most useful: whether it boots, whether the
installer completes, and whether the machine comes back up on its own after
`postinstall.sh`.

Firmware blobs are redistributed following the same posture as the community repo,
whose maintainer confirmed they have no objection to how people obtain them.
