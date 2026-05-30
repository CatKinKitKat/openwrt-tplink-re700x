# TP-Link RE700X EU v1.0 WIP Notes

Status: experimental OpenWrt bring-up only. Do not flash. Current scope is DTS,
build integration, initramfs/FIT generation, and RAM boot via U-Boot/TFTP.

Do not publish device-specific MAC addresses, serial numbers, PINs, or ART
dumps. The ART dump and stock backup files are local recovery inputs only.

## Current Status

- Branch: `tplink-re700x-wip`
- Boot method: U-Boot/TFTP RAM boot only
- Current local test image: `/srv/tftp/re700x-wifi-test1.itb`
- Kernel starts and reaches userspace on initramfs.
- SPI-NAND is detected and SMEM/MIBIB partitions are exposed correctly.
- `factory_data` mounts read-only as UBIFS and provides `default-mac`.
- Ethernet on the external port works through `lan` at 1000 Mbps full duplex.
- `br-lan` and `lan` use the 6-byte `default-mac` from `factory_data`.
- Default network config treats the single external port as DHCP client LAN.
- Wi-Fi is enabled for the first RAM-only ath11k test. Calibration is extracted
  from `0:art`, while BDF/board-2 handling is still expected to need follow-up.
- No factory or sysupgrade image support is considered ready.

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
- Use `setenv` only for RAM-boot tests. Do not `saveenv` as part of bring-up.

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
tftpboot 0x44000000 re700x-factorydata.itb
bootm 0x44000000
```

Current Wi-Fi test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-wifi-test1.itb
bootm 0x44000000
```

Post-boot network sanity checks:

```sh
mount | grep factory_data
ls -l /tmp/factory_data
ip addr show dev br-lan
ip link show dev lan
ping -c 3 192.168.1.1
```

MAC check without printing the real address:

```sh
factory_hex=$(hexdump -v -e '6/1 "%02x"' /tmp/factory_data/default-mac)
lan_hex=$(cat /sys/class/net/lan/address | tr -d ':')
br_hex=$(cat /sys/class/net/br-lan/address | tr -d ':')

[ "$lan_hex" = "$factory_hex" ] && echo "lan equals default-mac" || echo "lan differs from default-mac"
[ "$br_hex" = "$factory_hex" ] && echo "br-lan equals default-mac" || echo "br-lan differs from default-mac"
```

## Bring-Up Log

- Initial DTS/build integration created `tplink,re700x` for
  `qualcommax/ipq50xx` and produced a bootable initramfs FIT using
  `config@mp03.3-c2`.
- First RAM boot reached userspace. The early U-Boot warnings about missing
  `/soc/qpic-nand@79b0000` and PCI did not block Linux boot.
- SMEM partition parsing works. `/proc/mtd` exposes all 16 expected
  partitions from MIBIB/SMEM.
- Initial Ethernet assumptions were corrected: the external port is not
  QCA8081. U-Boot PHY ID `0x001cc916` and runtime tests match Realtek
  RTL8211F on MDIO1 address 6.
- `re700x-rtl8211f.itb` validated external `lan` link at 1000 Mbps full duplex
  and successful ping to the TFTP host with a static IP.
- `re700x-lan-dhcp2.itb` validated default DHCP client config on `br-lan` with
  `lan` as its single bridge port.
- `re700x-macfix.itb` confirmed that MAC setup still failed while
  `factory_data` was not mounted.
- `re700x-factorydata.itb` validated read-only mount of
  `ubi14:ubi_factory_data` at `/tmp/factory_data`; `default-mac` exists as a
  6-byte file; `lan` and `br-lan` both match `default-mac`; DHCP lease and
  ping to the upstream router work.
- `re700x-wifi-test1.itb` enables the internal IPQ5018 radio and the external
  QCN6122 userpd2 radio for RAM-only testing. Caldata extraction uses `0:art`
  offset `0x1000` for IPQ5018 and `0x26800` for QCN6122, with MAC addresses
  derived from `/tmp/factory_data/default-mac`. No device-specific board-2/BDF
  package exists yet, so ath11k may still stop at firmware/board data lookup.

## Current DTS Bring-Up Assumptions

- SPI-NAND is described through QPIC with `qcom,smem-part`.
- Factory/sysupgrade images are intentionally not defined.
- Wi-Fi is enabled only in RAM-test images for calibration/BDF validation.
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
- RAM-boot test with `re700x-factorydata.itb` mounts `factory_data` as
  `ubi14:ubi_factory_data`, reads the 6-byte `default-mac`, assigns it to
  both `lan` and `br-lan`, and obtains a DHCP lease from the upstream router.
- Runtime validation: `/proc/mtd` exposes all 16 SMEM partitions, device-tree
  compatible is `tplink,re700x`, and LEDs enumerate as `green:power`,
  `blue:wps`, `red:wps`, `green:wlan2g`, and `green:wlan5g`.
- Stock DTB indicates two active radios: internal IPQ5018 on userpd1 and
  QCN6122 on userpd2; a third radio is disabled. The current OpenWrt DTS
  mirrors this as `&wifi` and `&wifi1` only.

## Open Items

- Validate first Wi-Fi boot logs from `re700x-wifi-test1.itb`.
- Create or source correct board-2/BDF data for TP-Link RE700X if ath11k needs
  it; do not substitute unrelated device board data as a final solution.
- Confirm generated caldata files and ath11k MAC handling without printing
  real MAC addresses.
- Decide later whether this target needs factory/sysupgrade image generation.
- Keep all tests RAM-boot-only until recovery and install paths are fully
  understood.
