# OpenWrt for the TP-Link RE700X (IPQ5018)

A work-in-progress OpenWrt port for the **TP-Link RE700X** Wi-Fi 6 range
extender (EU v1.0), based on the `qualcommax` target. Built on a full OpenWrt
tree; this README covers only the RE700X-specific port.

> ⚠️ **Status: working but WIP.** Installing currently requires a serial (UART)
> console for the initial flash, and a NAND backup is strongly recommended
> before you start. A plug-and-play image flashable from the stock TP-Link web
> UI is in progress (see *Roadmap*). Don't flash this unless you understand the
> recovery path.

## What works

| Feature | Status |
| --- | --- |
| Boot from NAND (persistent) | ✅ |
| Ethernet (WAN/LAN, RTL8211F) | ✅ |
| 5 front LEDs + 2 buttons | ✅ |
| 2.4 GHz Wi-Fi 6 (IPQ5018) | ✅ |
| 5 GHz Wi-Fi 6 (QCN6122) | ✅ |
| Web-UI-flashable factory image | 🔧 in progress |

Both radios run as APs simultaneously and are stable.

## Hardware

- SoC: Qualcomm **IPQ5018** (2.4 GHz built-in) + **QCN6122** (5 GHz)
- RAM: 256 MiB · Flash: 128 MiB SPI-NAND (GigaDevice F50D1G41LB)
- Ethernet PHY: Realtek RTL8211F · 1× Gigabit port
- Bootloader: TP-Link U-Boot 2016.01 (console 115200 8N1, break string `tpl`)

## Building

This is a standard OpenWrt build. Select the device and build:

```sh
# in the OpenWrt tree
make menuconfig        # Target: Qualcomm Atheros 802.11ax (qualcommax) >
                       #   Qualcomm IPQ50xx > select "TP-Link RE700X"
make -j"$(nproc)"
```

Output in `bin/targets/qualcommax/ipq50xx/`:

- `…tplink_re700x-squashfs-sysupgrade.bin` — upgrade from a running OpenWrt
- `…tplink_re700x-squashfs-factory.ubi` — raw UBI for initial install
- `…tplink_re700x-initramfs-uImage.itb` — RAM boot via U-Boot/TFTP (for testing)

## Installing (current method — needs UART)

1. **Back up the stock NAND first** (via U-Boot or a running system). Keep it.
2. Test in RAM via U-Boot/TFTP before touching flash:
   ```
   setenv serverip <host>; setenv ipaddr <dev>
   tftpboot 0x44000000 …-initramfs-uImage.itb
   bootm 0x44000000
   ```
3. From the booted OpenWrt, install with `sysupgrade -n …-sysupgrade.bin`.

The stock U-Boot selects the FIT config by name, so the build sets
`DEVICE_DTS_CONFIG = config@mp02.1`; the UBI partition is `rootfs` (not
`firmware`).

## The 5 GHz / QCN6122 fix (for other IPQ5018+QCN6122 ports)

The QCN6122 is not a PCIe card — it boots as a protection domain (userpd2) on
the shared Q6/WCSS, and the firmware resets it via a GPIO named in the PIL
**boot-args**. Two board-specific values, both recovered from the stock
firmware, were needed:

1. `&q6v5_wcss` boot-args `<0x1 4 3 15 0 0  0x2 4 2 0x1b 0 0>` — UPD2 (the
   QCN6122) on **PCIE1 + reset GPIO 27 (0x1b)**, a board override of the
   firmware-default GPIO 18. Without it the userpd stalls in init (DOG
   watchdog / `err_smem_ver`) and crashes the Q6, killing 2.4 GHz too.
2. A QCN6122 `board-2.bin` (`package/firmware/ipq-wifi/src/board-tplink_re700x.qcn6122`)
   built from the stock `bdwlan.b60` board data; otherwise ath11k fails with
   board-data load `-12`.

See `NOTES-re700x.md` for the full bring-up log and `RE700X-CHANGELOG.md` for
versioned images.

## Roadmap

- [ ] Factory image flashable from the stock TP-Link web UI (no soldering).
      The stock web upgrade uses `nvrammanager -u` with **no RSA signature** —
      only a `support-list` + `soft_ver` + MD5 check — so a wrapped image is
      feasible.
- [ ] WPA3 (SAE) defaults.
- [ ] Upstreaming to OpenWrt.

## Credits & safety

Reverse-engineered from a stock NAND dump and the stock firmware. **No
device-specific MACs, PINs, serials, or ART/calibration dumps are published in
this repo.** Use at your own risk; have a UART console and a NAND backup ready.
