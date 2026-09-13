# Linux on the ASUS Vivobook S14 (S3407QA)

Device tree and notes for running Linux on the ASUS Vivobook S 14 S3407QA —
Snapdragon X (X1P-42-100 / "Purwa"), ARM64, Copilot+ PC.

There is no upstream device tree for this model. The closest mainline file is
`x1p42100-asus-vivobook-s15.dts`; this DTS is derived from it.

Tested on Ubuntu 26.04.1 (resolute), kernel 7.0.0-31-generic.

## Status

| Component | State |
|---|---|
| Keyboard, touchpad | works |
| Panel (Samsung ATNA40KW01, 1920x1200 OLED) | works |
| GPU (Adreno X1-45, turnip/Mesa) | works |
| NVMe | works |
| Battery + ASUS charge thresholds | works |
| CPU frequency scaling | works, needs `scmi-cpufreq` loaded manually |
| Suspend | works |
| USB-A (SuperSpeed), USB-C data | works |
| HDMI (Parade PS185HDM bridge) | works |
| Audio: speakers, headphones, jack detect | works |
| Bluetooth | works |
| Wi-Fi (WCN6855) | works ~3 boots in 5 — see below |
| Microphone | device present, captures silence |
| CPU thermal throttling | measurement only, no passive trip points |
| Fan control | none — firmware/EC only |
| Webcam | not working |

## Wi-Fi: intermittent

`qcom-qmp-pcie-phy 1c0e000.phy: phy initialization timed-out` -> the PCIe root
complex for the Wi-Fi card never comes up, and the whole domain is missing from
`lspci`. Resembles upstream commit 6cb8c1f (PHY powered down by firmware), but
that fix is already in this kernel, so it is a related residual case.

There is no runtime recovery: `qcom-pcie` sets `suppress_bind_attrs`, so the
controller cannot be rebound. Only a reboot helps.

`ath11k_pci` is blacklisted on the kernel command line and loaded late by a
systemd unit — probing it early caused unbootable systems.

Board file: see `wifi/board-2-fallback.md`.

## Firmware

No firmware blobs are redistributed here. Extract them from your own Windows
installation; `notes/firmware-inventory.txt` lists the DriverStore paths.

Install to `/lib/firmware/updates/qcom/x1p42100/ASUSTeK/vivobook-s14/`:
`qcdxkmsucpurwa.mbn` (GPU zap shader), `qcadsp8380.mbn` + `adsp_dtbs.elf`,
`qccdsp8380.mbn` + `cdsp_dtbs.elf`.

## Building

    cp dts/x1p42100-asus-vivobook-s14.dts <kernel>/arch/arm64/boot/dts/qcom/
    make ARCH=arm64 qcom/x1p42100-asus-vivobook-s14.dtb

`dts/base/` holds the includes this was built against. `reference/` holds
mainline snapshots for comparison — they are NOT the build inputs and differ
(see `notes/provenance.md5`).

## Booting

Kernel command line:

    quiet splash arm64.nopauth cma=128M efi=noruntime modprobe.blacklist=ath11k_pci

The DTB must be loaded with GRUB's `devicetree` command **before** `linux` —
Ubuntu's `10_linux` emits it after `initrd`, which does not work. Use a custom
entry in `/etc/grub.d/40_custom`:

    search --no-floppy --set=root --fs-uuid <root-uuid>
    devicetree /boot/dtb
    linux /boot/vmlinuz root=UUID=<root-uuid> ro <params above>
    initrd /boot/initrd.img

A hand-written `linux` line does not inherit `GRUB_CMDLINE_LINUX`; omitting
`cma=128M efi=noruntime` gives a black screen.

The firmware refuses to boot external USB media (Secured-core PC). Put the
bootloader on the internal ESP and add a boot entry from Windows with
`bcdedit /copy {bootmgr}`.

## Credits

The DTS is derived from `x1p42100-asus-vivobook-s15.dts` (Qualcomm
Innovation Center, Xilin Wu) and `x1-asus-vivobook-s15.dtsi` (Maud
Spierings); the HDMI bridge block follows the PS8830/PS185HDM work in
that file. SoC includes `purwa.dtsi` and `hamoa.dtsi` are Qualcomm/Linaro.
Build inputs in `dts/base/` come from Jens Glathe's Ubuntu X1E kernel tree.
`wifi/ath11k-bdencoder` is from Qualcomm's qca-swiss-army-knife.

## License

BSD-3-Clause, see LICENSE. `wifi/ath11k-bdencoder` is ISC.