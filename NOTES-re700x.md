# TP-Link RE700X EU v1.0 WIP Notes

Status: experimental OpenWrt bring-up only. Do not flash. Current goal is DTS,
build integration, initramfs/FIT generation, and RAM boot via U-Boot/TFTP.

## Hardware

- Device: TP-Link RE700X
- Hardware: EU v1.0
- SoC: Qualcomm IPQ5018 / IPQ0509
- RAM: 256 MiB
- Flash: 128 MiB SPI-NAND, GigaDevice F50D1G41LB
- Stock FIT default configuration: `config@mp03.3-c2`
- Stock FIT kernel: `kernel@1`
- Stock FIT FDT: `fdt@mp03.3-c2`
- Stock FIT load address used from U-Boot: `0x44000000`
- Generated OpenWrt FIT kernel load/entry: `0x41000000` (qualcommax default)

## UART and U-Boot

- UART settings: 115200 8N1
- UART SoC pins from stock DTB: GPIO20/GPIO21
- U-Boot: 2016.01
- Break string: `tpl`
- Prompt: `IPQ5018#`
- TFTP works through U-Boot `eth1`
- U-Boot reports `eth1`/MAC1 PHY ID `0x001cc916`, matching Realtek RTL8211F.

## GPIOs

Buttons:

- Reset: GPIO25, active low
- WPS: GPIO31, active low

LEDs:

- Power: GPIO38, active high
- WPS red: GPIO18, active high
- WPS blue: GPIO22, active high
- 2G WLAN: GPIO39, active high
- 5G WLAN: GPIO13, active high

## Flash Layout from SMEM/MIBIB

Partitions are expected from Qualcomm SMEM/MIBIB. Do not hard-code fixed DTS
partitions for this WIP port.

```text
0:SBL1           0x00000000 0x00080000
0:MIBIB          0x00080000 0x00080000
0:BOOTCONFIG     0x00100000 0x00040000
0:BOOTCONFIG1    0x00140000 0x00040000
0:QSEE           0x00180000 0x00100000
0:DEVCFG         0x00280000 0x00040000
0:CDT            0x002c0000 0x00040000
0:APPSBLENV      0x00300000 0x00080000
0:APPSBL         0x00380000 0x00140000
0:ART            0x004c0000 0x00100000
0:TRAINING       0x005c0000 0x00080000
rootfs           0x00640000 0x02a00000
rootfs_1         0x03040000 0x02a00000
0:ETHPHYFW       0x05a40000 0x00080000
factory_data     0x05ac0000 0x00900000
runtime_data     0x063c0000 0x01100000
```

## RAM Boot Test Only

Use U-Boot/TFTP only. Do not write flash.

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 openwrt-qualcommax-ipq50xx-tplink_re700x-initramfs-uImage.itb
bootm 0x44000000
```

Current local test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-lan-dhcp.itb
bootm 0x44000000
```

## Current DTS Bring-Up Assumptions

- SPI-NAND is described through QPIC with `qcom,smem-part`.
- Factory/sysupgrade images are intentionally not defined.
- Wi-Fi is intentionally left disabled until calibration/BDF details are
  validated.
- Ethernet is based on the stock DTB's two NSS-DP/MDIO links, not the
  WN-DAX3000GR QCA8337 switch topology.
- External Ethernet on `lan` uses the Realtek RTL8211F at MDIO1 address 6.
- RAM-boot test with `re700x-rtl8211f.itb` links at 1000 Mbps full duplex and
  can ping the TFTP host with a static address on `lan`.
- Board default network config uses `lan` as a DHCP client, matching the
  single-port repeater/AP use case.
- RAM-boot test with `re700x-lan-dhcp2.itb` creates `br-lan`, brings `lan` up
  through netifd, and obtains an IPv4 DHCP lease from the upstream router.
- `factory_data` is mounted read-only during preinit for TP-Link RE700X so
  board scripts can read `/tmp/factory_data/default-mac`.
- Runtime validation: `/proc/mtd` exposes all 16 SMEM partitions, device-tree
  compatible is `tplink,re700x`, and LEDs enumerate as `green:power`,
  `blue:wps`, `red:wps`, `green:wlan2g`, and `green:wlan5g`.
