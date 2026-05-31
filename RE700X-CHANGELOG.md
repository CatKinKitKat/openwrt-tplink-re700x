# TP-Link RE700X (IPQ5018) — Image Changelog

Tracks flashable OpenWrt images for the RE700X EU v1.0 port on branch
`tplink-re700x-wip`. Versions are git-tagged (`re700x-vX.Y`). The `sha256` is of
the `squashfs-sysupgrade.bin` built from that commit. Doc-only changes
(NOTES/CHANGELOG) do not alter the firmware image, so a tagged image's sha256
stays valid across later documentation commits.

## v1.1 — 2026-05-31 (tag `re700x-v1.1`)

Adds a **web-UI-flashable factory image** — install OpenWrt on a stock RE700X
straight from the TP-Link "Firmware Upgrade" web page (no UART, no soldering).
Firmware is the v1.0 port; this release is the install-path tooling.

- sha256 (sysupgrade): `41d5366da07ef2f1a9a6019b247ef936e8bd68ae488b52687d5ebb35f9e1ad0c`
- New tool `re700x-factory-pack.py` (+ bundled `re700x-fwdata/`) wraps the rootfs
  UBI into the stock `nvrammanager` upload format (TP-Link safeloader + the
  reverse-engineered `FwUpTbl` partition table). Build it with:
  ```
  ./re700x-factory-pack.py \
     --os bin/targets/qualcommax/ipq50xx/openwrt-qualcommax-ipq50xx-tplink_re700x-squashfs-factory.ubi \
     --bump-version "9.9.9 Build 20991231 Rel. 99999" -o re700x-factory.bin
  ```
- Format fully reverse-engineered from the stock `nvrammanager`; see
  `RE700X-FACTORY-IMAGE-PROBLEM.md`.
- Proven end-to-end on real hardware: `nvrammanager -c`/`-u` **and** the stock
  web GUI flash the rootfs into the inactive dual-boot slot and reboot into
  OpenWrt 6.12.91.

## v1.0 — 2026-05-31 (commit 20fd56b47a)

First fully working image: all hardware up.

- sha256 (sysupgrade): `a3873d326429e6bd094efeb99d1cfc80f56df3a03a8b8c7396a31a0082ab5edf`
- **5G Wi-Fi 6 (QCN6122) now works** — the last open item:
  - `q6v5_wcss` boot-args set to the stock values: UPD2 on PCIE1 + reset
    GPIO 27 (0x1b). This is what lets the QCN6122 userpd finish init instead
    of stalling in the DOG watchdog / `err_smem_ver` and crashing the Q6.
  - Added `board-tplink_re700x.qcn6122` (ath11k `board-2.bin` built from stock
    `bdwlan.b60`, board_id 0x60, variant `TP-Link-RE700X`) → fixes the ath11k
    board-data load failure (-12).
- Both radios run as APs simultaneously and stay stable.
- Carries forward everything from v0.9.

## v0.9 — 2026-05-30 (commits 7f70639f7e, f15057cc87)

First flashable image that boots OpenWrt from NAND.

- sha256 (sysupgrade, 2.4G image): `09981a4a…`
- Boots persistently from NAND (no more U-Boot/TFTP RAM-boot dance).
- Ethernet (WAN/LAN), 5 LEDs, 2 buttons.
- **2.4G Wi-Fi 6 (IPQ5018)** works.
- 5G/QCN6122 not yet working (userpd stalled in init).
- Key flash fixes: `DEVICE_DTS_CONFIG=config@mp02.1`, platform.sh
  `CI_UBIPART=rootfs`, `CONFIG_CMDLINE_FORCE` + `coherent_pool`.
