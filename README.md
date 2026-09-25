facetimehd — patched branch
===========================

This is a fork of [patjak/facetimehd](https://github.com/patjak/facetimehd). The default branch
carries upstream `master` plus the pull requests listed below, all of them open upstream at the time
of writing. `master` here is kept as a plain mirror of upstream, for rebasing.

Everything here is tested on a **MacBookPro14,1** (sensor `0005 0248`), kernel 7.2.6, and each patch
was also verified on its own before being combined.

| PR | what it fixes |
|---|---|
| [#331](https://github.com/patjak/facetimehd/pull/331) | `ENUM_FRAMESIZES` reports the real range instead of a single size |
| [#334](https://github.com/patjak/facetimehd/pull/334) | auto exposure settles in 200 ms instead of a full second |
| [#338](https://github.com/patjak/facetimehd/pull/338) | the sensor crop is centred before scaling |
| [#340](https://github.com/patjak/facetimehd/pull/340) | the S2 PLL lock loop waited for the wrong bit state; errors now propagate |
| [#342](https://github.com/patjak/facetimehd/pull/342) | `CREATE_BUFS` is bounded by free contexts and memory |
| [#343](https://github.com/patjak/facetimehd/pull/343) | frame rates keep the ISP fixed-point units |
| [#344](https://github.com/patjak/facetimehd/pull/344) | auto exposure may lower the frame rate in low light |
| [#345](https://github.com/patjak/facetimehd/pull/345) | up to eight capture buffers instead of four |
| [#346](https://github.com/patjak/facetimehd/pull/346) | YVYU is no longer advertised; its output is unusable as delivered (see note) |
| [#347](https://github.com/patjak/facetimehd/pull/347) | the native sensor bounds are exposed through `G_SELECTION` |
| [#348](https://github.com/patjak/facetimehd/pull/348) | the set file for sensor `0x248` is loaded, so the camera runs calibrated |
| [#350](https://github.com/patjak/facetimehd/pull/350) | the driver says when the calibration file is missing, and which one it wants |
| [#351](https://github.com/patjak/facetimehd/pull/351) | the MacBook Air set file is picked by product name, not board name |

`v4l2-compliance -d /dev/video0 -s` on this branch: **57 tests, 57 passed, 0 failures, 0 warnings**.
On upstream `master` the same run gives 51 passed and **6 failures** (`Scaling`, and five on the
`CREATE_BUFS` paths).

**Note on #346.** Measured here, the YVYU stream is not invalid data: it is the correct frame shifted
by exactly one byte. Read at a one-byte offset it matches the YUYV frame to the second decimal; at a
three-byte offset the U and V planes swap, as YVYU requires. The offset is constant across 1280x720,
800x600 and 640x480. The driver treats both formats identically apart from the value it sends the
ISP, so there is no offset to correct on the driver side, and the format cannot be delivered
correctly today — which is why dropping it is the right call for now, even though the cause is an
indexing shift rather than broken firmware output.

Calibration files and firmware come from
[facetimehd-firmware](https://github.com/vrilutza/facetimehd-firmware); its default branch carries
two fixes of its own.

---

facetimehd
==========

Linux driver for the Facetime HD (Broadcom 1570) PCIe webcam
found in recent Macbooks.

This driver is experimental. Use at your own risk.

See the [Wiki][wiki] for more information:

[wiki]: https://github.com/patjak/bcwc_pcie/wiki
