Linux 7.3.6 for the Xiaomi Pad 5 (nabu)
=======================================

Mainline Linux for the Xiaomi Pad 5 (codename nabu, Snapdragon 860 / SM8150). It runs the tablet on
**VibeOS** (Debian unstable with Mobian's repository) with GNOME and GDM, next to LineageOS.

The tree is [Kumar-Jy's nabu kernel](https://github.com/Kumar-Jy/linux), branch `nabu/7.3.y`, at their
release v7.3.6-0 (commit `97375316e260`), imported as one snapshot commit of their tree (their full history
is in their repository), plus the VibeOS commits listed below. All of the Xiaomi Pad 5 support in this
tree is Kumar-Jy's work, built on the SM8150 work of [sm8150-mainline](https://github.com/sm8150-mainline/linux),
postmarketOS and mainline Linux.

Use the `main` branch.

What works
==========

Checked on a Xiaomi Pad 5 (Wi-Fi model) running VibeOS, November 2026:

| Part | State |
| --- | --- |
| Display | 1600x2560 at 120 Hz over dual DSI; rotation from the accelerometer |
| Touchscreen | NT36523, ready from the initramfs |
| GPU | Adreno 640 with Mesa (Freedreno, Turnip Vulkan); GNOME Shell runs on it |
| Wi-Fi | WCN3998 (`ath10k_snoc`), 2.4 and 5 GHz |
| Speakers | the four CS35L41 amps (VibeOS adds a speaker protection filter in PipeWire) |
| Microphones | through the WCD9340 codec on SLIMbus, every capture (see the revert below) |
| Video | Iris hardware decode of H.264 and HEVC |
| Cameras | OV13B10 rear and OV8856 front through camss and libcamera (software ISP); stills checked with the screen off |
| Sensors | accelerometer and ambient light sensor through the SLPI's sensor core (hexagonrpcd and libssc in userspace) |
| Battery | charge level and rate; USB PD charging at 9 V through the PM8150B (about 8 W into the battery) |
| USB | device mode (gadget network) |
| Suspend | s2idle: wakes on a timer, and Wi-Fi, Bluetooth, USB and the sensors come back |

Not done
========

* **Cameras: no tuning yet.** libcamera has no sensor helpers or tuning files for the OV13B10 and OV8856,
  so its software ISP runs uncalibrated. Video recording and camera apps not checked yet.
* **Fast charging:** the 33 W charge pump (LN8000, `charger@51` in the device tree) does not answer, so
  charging runs on the PM8150B alone.
* **Bluetooth:** the controller powers up and its firmware loads; pairing and audio not checked yet.
* **Backlight:** the KTZ8866 driver is up; changing the brightness not checked yet.
* **Magnetometer:** reads a steady heading; whether it points right not checked yet.
* Not checked: the pen, typing on the keyboard cover (it is detected), USB host mode, USB-C headphones,
  waking from suspend with the power key, and the battery drain asleep.
* No vibration motor driver.

Changes for VibeOS
==================

On top of Kumar-Jy's v7.3.6-0:

* **`arch/arm64/configs/vibeos-nabu.config`**, a fragment merged over `nabu_defconfig`. It names the
  kernel `7.3.6-nabu`, drops DWARF, turns off platforms that cannot be a Xiaomi Pad 5, builds in the GENI
  SPI bus (the touch controller sits on it, and touch must not wait for the root filesystem), and adds the
  options every VibeOS device kernel carries (zram, erofs, uinput, YAMA, lockup detectors, USB HID, Wi-Fi,
  Bluetooth, serial and touch peripherals).
* **Revert "ASoC: codecs: wcd934x: stop capture port before tearing down SLIM channel"** (c5834329a6cb).
  With it, only the first capture after the codec probes carried audio: every later stream read one frozen
  sample per channel, and the codec reported "underflow error on TX port 5" as it started. Writing 0 to the
  TX port's config at stop, before the channel is torn down, leaves the port dead for the next stream. With
  the revert, back-to-back captures all carry audio.
* **camss: every VFE line's pads are set up before its video nodes are registered.**
  `msm_vfe_register_entities()` registered line 0's video node before lines 1 to 3 had their media pads,
  and udev's `v4l_id` opens a node as soon as it appears. `video_open()` then reached `camss_find_sensor()`
  for every line of the VFE and dereferenced the pads of a line not set up yet. That NULL pointer oops left
  the VFE's power lock held, so every later open of a video node hung. Now all 16 video nodes open and
  return.
* **camss: `camss_find_sensor()` returns no sensor for an entity without pads**, as a guard.
* **"media: qcom: camss: Power pipeline only when streaming"** (upstream 164202f68203, in Linux since
  v6.17), applied again: the import of the nabu camss had undone it. With the pipeline powered from
  `open()`, PipeWire's libcamera monitor, which keeps a video node open, held VFE0 powered at its highest
  clock from early boot, and CSID and VFE resets then timed out. Now a node held open costs nothing,
  and the pipeline powers up for streaming only.
* **camss: a CSID that fails to power on puts its VFE back.** `csid_set_power()` took the parent VFE first
  but returned on its error paths without putting it, so each failed CSID power-on left the VFE powered
  for good. The error paths now unwind in order and put the VFE last.

Build
=====

With Debian's cross compiler (`gcc-aarch64-linux-gnu`):

```
make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- nabu_defconfig
scripts/kconfig/merge_config.sh -m -O out out/.config arch/arm64/configs/vibeos-nabu.config
make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LOCALVERSION= Image.gz modules dtbs
make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LOCALVERSION= INSTALL_MOD_PATH=<dir> INSTALL_MOD_STRIP=1 modules_install
```

The kernel release is `7.3.6-nabu`. Pass `LOCALVERSION=` to every make, or it gains a `+`.

Booting it
==========

The tablet's unlocked bootloader starts an Android boot image, header version 0, page size 4096, with
`Image.gz` followed by `qcom/sm8150-xiaomi-nabu.dtb` as the kernel:

```
cat out/arch/arm64/boot/Image.gz out/arch/arm64/boot/dts/qcom/sm8150-xiaomi-nabu.dtb > Image.gz-dtb
mkbootimg --header_version 0 --pagesize 4096 --base 0x0 --kernel_offset 0x8000 --ramdisk_offset 0x1000000 \
	--second_offset 0xf00000 --tags_offset 0x100 --kernel Image.gz-dtb --ramdisk initramfs.cpio.gz \
	--cmdline "..." -o boot.img
```

The touch module (`nt36523_ts`) and its firmware belong in the initramfs.

VibeOS on the tablet
====================

The Xiaomi Pad 5 has A/B slots. VibeOS uses slot b and leaves slot a to LineageOS:

* VibeOS's boot image goes into `boot_b`, and its root filesystem into a partition of its own, `linux`,
  carved out of `userdata`. Slot a's partitions are never written.
* `fastboot set_active b` starts VibeOS, `fastboot set_active a` goes back to LineageOS. Fastboot: with the
  tablet off, hold Volume Down and press Power.
* A slot the bootloader has not seen marked good falls back to the other slot once its retries run out.

Firmware
========

None is in this tree. The tablet needs its vendor firmware: ADSP, CDSP, SLPI, the modem DSP (it runs the
Wi-Fi firmware), Iris, the GPU's zap shader and Bluetooth.

Credits
=======

* **Kumar-Jy**: the Xiaomi Pad 5 kernel this tree is, and the nabu packages
  ([Kumar-Jy/nabu-pkgs](https://github.com/Kumar-Jy/nabu-pkgs)).
* **sm8150-mainline** ([sm8150-mainline/linux](https://github.com/sm8150-mainline/linux)): mainline Linux
  for Snapdragon 855/860 devices.
* **postmarketOS**: the nabu port and its boot image layout.
