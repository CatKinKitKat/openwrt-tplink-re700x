# OpenWrt for the TP-Link RE700X (IPQ5018)

A work-in-progress OpenWrt port for the **TP-Link RE700X** Wi-Fi 6 range
extender (EU v1.0), based on the `qualcommax` target. Built on a full OpenWrt
tree; this README covers only the RE700X-specific port.

> ⚠️ **Status: working — but the stock web-GUI install is EXPERIMENTAL and has
> bricked a unit.** Once OpenWrt is installed it runs great, and `sysupgrade`
> (OpenWrt→OpenWrt) is safe and repeatable. **But the *initial* flash from the
> stock web GUI bricked a second, healthy device** (clean stock, no UART) — it
> booted nothing and is recoverable only via UART. This bootloader has **no
> button/TFTP recovery**. **Do not flash from stock without UART access and a
> full NAND backup.** Suspected cause under investigation (dual-boot slot /
> `ubi.mtd=` cmdline mismatch — see §*Install from stock*).

## What works

| Feature | Status |
| --- | --- |
| Boot from NAND (persistent) | ✅ |
| Ethernet (WAN/LAN, RTL8211F) | ✅ |
| 5 front LEDs + 2 buttons | ✅ |
| 2.4 GHz Wi-Fi 6 (IPQ5018) | ✅ (single radio) |
| 5 GHz Wi-Fi 6 (QCN6122) | ✅ (single radio) |
| **Both radios at once** | ❌ OOM on 256 MB — see *Known issue: RAM* |
| Web-UI-flashable factory image | ⚠️ experimental (bricked a unit) |

Each radio works. Running **both radios + full default services** exhausts the
256 MB RAM and the OOM-killer reboots the device after ~30–60 s. A single radio
(2.4 *or* 5 GHz) is stable.

## Known issue: RAM / out-of-memory with both radios

On this 256 MB device (~184 MB usable after firmware reservations) the mainline
**ath11k + nss-dp** network stack alone consumes ~90–110 MB, leaving too little
for userspace → OOM-kill reboot loop. Pinned via `/proc/allocinfo` (kernel
`CONFIG_MEM_ALLOC_PROFILING`); the dominant consumers are:

- **`__page_frag_cache_refill` ≈ 54 MB and growing** — network-driver RX page
  fragments (nss-dp/EDMA Ethernet + ath11k RX).
- `__dma_direct_alloc_pages` ≈ 19 MB + `atomic_pool_expand` ≈ 16 MB — ath11k /
  coherent DMA.
- `alloc_slab_page` ≈ 20 MB, skb ≈ 5 MB.

These live outside the normal `/proc/meminfo` counters, which is why `free`
showed ~100 MB "missing". It is **not** a userspace/package problem and **not**
fixed by `qcom,ath11k-fw-memory-mode` (mainline ath11k ignores that DT property).
The real fix is reducing the EDMA/ath11k RX-buffer footprint (driver/DT) — open
work; the `allocinfo` data above is the evidence for an upstream/community report.
**Workaround: run a single radio.**

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

## Install from stock via web GUI (EXPERIMENTAL — have UART ready)

> ⚠️ **This has bricked a device.** It worked on one unit and left a second,
> healthy unit dead (booted nothing) after a clean stock web-GUI flash with no
> UART attached. There is **no button/TFTP recovery** on this bootloader, so a
> failed flash needs UART to recover. Suspected cause (pending a serial log):
> the image's kernel cmdline hardcodes `ubi.mtd=rootfs` (dual-boot slot 0); if
> the stock flasher writes OpenWrt into the *other* slot (`rootfs_1`) and boots
> it, the kernel attaches the wrong UBI and root is not found → no boot.
> **Only do this with UART access and a full NAND backup, and treat it as a
> recoverable experiment — not plug-and-play.**

The flow — flash from the **stock** TP-Link web interface:

1. Grab the factory image from the latest [release](../../releases)
   (`re700x-v1.1-factory-webflash.bin`), or build it yourself (below).
2. On the stock RE700X: web UI → **System → Firmware Upgrade**.
3. Upload `re700x-v1.1-factory-webflash.bin` and start the upgrade.
4. The device writes OpenWrt to the inactive dual-boot slot and reboots into it.
   Your stock firmware stays in the other slot as a fallback.

Requirements: the device must be running **stock firmware** with an installed
version ≤ the image's `soft_ver` (the released image is bumped to `9.9.9` so it
always passes the stock anti-downgrade check). Verify the download against
`SHA256SUMS` before flashing.

### Build the factory image yourself

```sh
# in the OpenWrt tree, after a normal build
./re700x-factory-pack.py \
  --os bin/targets/qualcommax/ipq50xx/openwrt-qualcommax-ipq50xx-tplink_re700x-squashfs-factory.ubi \
  --bump-version "9.9.9 Build 20991231 Rel. 99999" \
  -o re700x-factory.bin
```

This wraps the rootfs UBI into the stock `nvrammanager` upload format (TP-Link
safeloader + the reverse-engineered `FwUpTbl` partition table). The format was
reverse-engineered from the stock `nvrammanager` and the whole flash path is
verified on real hardware — details in `RE700X-FACTORY-IMAGE-PROBLEM.md`.

## Alternative: UART / TFTP (recovery or development)

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

- [~] Factory image flashable from the stock TP-Link web UI (no soldering) —
      `re700x-factory-pack.py` works (one unit installed fine), **but bricked a
      second unit** → experimental until fixed.
- [ ] Fix the dual-boot brick: make the kernel cmdline slot-aware (stop
      hardcoding `ubi.mtd=rootfs`) so OpenWrt boots from whichever slot the stock
      flasher writes. Confirm the cause via #2's UART serial log.
- [ ] Integrate the `FwUpTbl` format into `tplink-safeloader` so `make` emits a
      ready-to-flash `factory.bin` directly (instead of the separate packer).
- [ ] WPA3 (SAE) defaults.
- [ ] Upstreaming to OpenWrt.

## Credits & safety

Reverse-engineered from a stock NAND dump and the stock firmware. **No
device-specific MACs, PINs, serials, or ART/calibration dumps are published in
this repo.** Use at your own risk; have a UART console and a NAND backup ready.
