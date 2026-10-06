facetimehd — patched branch
===========================

This is a fork of [patjak/facetimehd](https://github.com/patjak/facetimehd). The default branch
carries upstream `master` plus the changes from the open pull request #355 by pschatzmann, listed
below. The other patches previously listed here are now part of upstream. `master` here is kept
as a plain mirror of upstream, for rebasing.

The validation below was recorded on revision `bed0636`, on a **MacBookPro14,1** (sensor `0005 0248`),
kernel 7.2.8. Synchronizing the upstream merges leaves every tracked file except this README
unchanged from that revision. The recorded results describe only that configuration.

| PR | what it fixes |
|---|---|
| [#355](https://github.com/patjak/facetimehd/pull/355) | `USERPTR` buffers that start inside a page are accepted again, with the offset carried to the hardware (by pschatzmann; see note) |

`v4l2-compliance -d /dev/video0 -s` on this branch: **57 tests, 57 passed, 0 failures, 0 warnings**,
with the two `USERPTR` streaming tests actually exercised rather than reported as not supported.
`DMABUF` is not tested here (`v4l2-compliance` needs an exporting device for it), and none of these
tests exercises PipeWire's import of buffers.

**Note on #355.** Upstream `master` refuses `USERPTR` entirely, through #333 — a patch of mine. #333
stopped a real corruption: the driver dropped the offset of a buffer that does not start on a page
boundary, and the camera wrote up to a page early. But it removed the mode instead of carrying the
offset, which the hardware accepts. Going by the trace in #355 and PipeWire's source, that breaks
PipeWire's import of buffers an application supplies; the import itself was not reproduced here. #355
carries the offset instead. Tested here with guard bytes at in-page offsets 0, 0x40, 0x100, 0x800 and
1000: no canary modification was detected in the guards, and the image lines up in luminance, where the
same tree with #333 reverted shifted it by the offset. Details and limits are in
[the review thread](https://github.com/patjak/facetimehd/pull/355).

**Not included: #356**, the warmup-frame patch. It is a draft, and whether it should apply to every
sensor or only the one measured here is an open question for the maintainer.

Installing
----------

On Debian and derivatives; adapt the package names elsewhere.

```
sudo apt install build-essential linux-headers-$(uname -r) dkms git curl xz-utils cpio v4l-utils
```

**1. Firmware and calibration first.** The driver needs firmware to initialize and loads the
calibration file if one is available for the sensor:

```
git clone https://github.com/vrilutza/facetimehd-firmware.git
cd facetimehd-firmware
make                 # downloads from Apple and verifies every file against a known hash
sudo make install    # into /lib/firmware/facetimehd/
cd ..
```

Prevent `bdc_pci` from binding to the camera on future boots, for either installation method:

```
printf 'blacklist bdc_pci\n' | sudo tee /etc/modprobe.d/facetimehd.conf
```

If `bdc_pci` is already loaded, reboot after installing the driver.

**2. The driver.** Clone the source before choosing either installation method:

```
git clone https://github.com/vrilutza/facetimehd.git
cd facetimehd
```

For a plain build:

```
make
sudo make install
sudo depmod -a
sudo modprobe facetimehd
```

or, to survive kernel upgrades, through DKMS:

```
V=0.7.2+$(git rev-parse --short=12 HEAD)
sudo mkdir -p /usr/src/facetimehd-$V
git archive HEAD | sudo tar -x -C /usr/src/facetimehd-$V     # source only, no build leftovers
sudo sed -i "s/^PACKAGE_VERSION=.*/PACKAGE_VERSION=$V/" /usr/src/facetimehd-$V/dkms.conf
sudo dkms install -m facetimehd -v $V
sudo modprobe facetimehd
```

The version includes the source revision so an update does not reuse an older DKMS build.
That installs it for the kernel you are running. DKMS rebuilds it for later kernels when their
headers and the distribution's DKMS hooks are available. If you keep an older kernel around as a
fallback, install it there too:
`sudo dkms install -m facetimehd -v $V -k <that kernel>`.

**3. Check it came up:**

```
v4l2-ctl --list-devices
dmesg | grep facetimehd | grep -E 'set file|S2 PLL'
```

The log reports the PLL status and `loaded set file facetimehd/NNNN_01XX.dat` when calibration loads.
The PLL timing can vary; these messages alone do not verify a working capture. If the set file is
missing, check that the installed calibration files include the name requested in the log.

Firmware and calibration
------------------------

Calibration files and firmware come from
[facetimehd-firmware](https://github.com/vrilutza/facetimehd-firmware); its default branch carries
the extraction changes in [#14](https://github.com/patjak/facetimehd-firmware/pull/14), still open upstream.
Neither repository contains the binaries themselves — they are Apple's, and the
tool extracts them from your own download, verifying each one against a known hash.

This branch is used daily here with firmware **5.60.0**, which the tool fetches by default and which
identifies itself as `S2ISP-01.57.00`. The older 1.43.0 works just as well: same formats, same sizes,
same `1571_01XX.dat` calibration. The difference is in the image, measured on a static scene in low
light over three interleaved rounds of 160 frames at the same exposure: 5.60.0 has **58 % less noise**
but **33 % less real detail**. Cleaner and softer against grainier and sharper. For switching firmware
versions, follow the firmware repository's instructions in a separate checkout.
The driver always requests `facetimehd/firmware.bin`; installing the other version replaces that file.

---

facetimehd
==========

Linux driver for the Facetime HD (Broadcom 1570) PCIe webcam
found in recent Macbooks.

This driver is experimental. Use at your own risk.

See the [Wiki][wiki] for more information:

[wiki]: https://github.com/patjak/bcwc_pcie/wiki
