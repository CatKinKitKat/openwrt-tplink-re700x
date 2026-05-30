# TP-Link RE700X EU v1.0 WIP Notes

Status: experimental OpenWrt bring-up only. Do not flash. Current scope is DTS,
build integration, initramfs/FIT generation, and RAM boot via U-Boot/TFTP.

Do not publish device-specific MAC addresses, serial numbers, PINs, or ART
dumps. The ART dump and stock backup files are local recovery inputs only.

## Current Status

- Branch: `tplink-re700x-wip`
- Boot method: U-Boot/TFTP RAM boot only
- Current local test image: `/srv/tftp/re700x-qcn-only-mm1.itb`
- Kernel starts and reaches userspace on initramfs.
- SPI-NAND is detected and SMEM/MIBIB partitions are exposed correctly.
- `factory_data` mounts read-only as UBIFS and provides `default-mac`.
- Ethernet on the external port works through `lan` at 1000 Mbps full duplex.
- `br-lan` and `lan` use the 6-byte `default-mac` from `factory_data`.
- Default network config treats the single external port as DHCP client LAN.
- Wi-Fi is enabled for RAM-only ath11k testing. Calibration is extracted from
  `0:art`; device-specific board-2 files are now included from the stock
  RE700X EU v1.0 rootfs boarddata.
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

Current board-2 Wi-Fi test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-board2-test1.itb
bootm 0x44000000
```

Current isolated IPQ5018 Wi-Fi test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-ipq5018-only-mm1.itb
bootm 0x44000000
```

Current dual-radio memory-mode-1 test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-both-mm1.itb
bootm 0x44000000
```

Current stock-memory dual-radio test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-stockmem-mm1.itb
bootm 0x44000000
```

Current QCN6122-only memory-mode-1 test copy:

```text
setenv serverip 192.168.1.248
setenv ipaddr 192.168.1.50
tftpboot 0x44000000 re700x-qcn-only-mm1.itb
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
- Stock `rootfs` was copied read-only from `/dev/mtd11ro` and extracted
  offline. The UBI image contains the stock FIT in volume 0 and the SquashFS
  rootfs in volume 2.
- Stock TP-Link boarddata selection chooses `boarddata_hw1.0/EU` for the EU
  hardware. The RAM-only OpenWrt test now packages these stock boarddata files
  as ath11k board-2 containers:
  `boarddata_hw1.0/EU/bdwlan.b24` for IPQ5018 and
  `boarddata_hw1.0/EU/bdwlan.b60` for QCN6122.
- `re700x-board2-test1.itb` includes `ipq-wifi-tplink_re700x`, installing
  `board-2.bin` for both `ath11k/IPQ5018/hw1.0` and
  `ath11k/QCN6122/hw1.0`. The FIT hash is
  `b4d2df528a8a96e4dfc77212fc454f1c80f09bc54590ced6e3be6c7c910e3e5c`.
- `re700x-board2-test1.itb` validates that both board-2 files are present and
  the previous boarddata lookup error is gone. IPQ5018 caldata is generated,
  but QCN6122 caldata is not generated before ath11k times out. The current
  blocking error is `failed to wait wlan mode request (mode 0): -110` on
  `c000000.wifi`.
- `re700x-ipq5018-only-mm1.itb` is an isolation test: QCN6122 is disabled and
  the internal IPQ5018 radio uses firmware memory mode 1. The FIT hash is
  `ac9857a717df5a6ad6dbd6564aa6b3712393a3359ec9ad59513b442baa7b26d6`.
- `re700x-ipq5018-only-mm1.itb` validates that the internal IPQ5018 radio
  starts with firmware memory mode 1. `wifi status` reports `radio0` as up;
  no WLAN interface is configured yet, so `iw dev` remains empty.
- `re700x-both-mm1.itb` enables both IPQ5018 and QCN6122 with firmware memory
  mode 1. The FIT hash is
  `917492c4bb0e18d302bd4bc8e4590c0df63437a85db37f5d3d224757ee4050b8`.
- `re700x-both-mm1.itb` still times out on `c000000.wifi` when QCN6122 is
  enabled, even though IPQ5018 alone works with memory mode 1.
- Stock DTB uses `qcom,userpd-subsys-name = "q6v5_wcss_userpd1"` for the
  internal IPQ5018 radio and places QCN6122 BDF/M3 regions at `0x4d200000` and
  `0x4de00000`. `re700x-stockmem-mm1.itb` tests those stock-derived addresses
  while keeping both radios on firmware memory mode 1. The FIT hash is
  `4f36a34892656492038b561cb393859b75516cd9f2a19bf4ed489b339781bac4`.
- `re700x-stockmem-mm1.itb` still times out on `c000000.wifi`, so the
  stock-derived QCN BDF/M3 addresses alone do not solve the dual-radio start
  issue.
- `re700x-qcn-only-mm1.itb` disables the internal IPQ5018 radio and keeps
  QCN6122 on PD2 with memory mode 1 and stock-derived memory hints. The FIT
  hash is `60d34394c6e22fa799211f788d780076cc5302676898c100ce6b00c88584b8c2`.
- Runtime test with `re700x-qcn-only-mm1.itb`: QCN6122 probes on PD2 and Q6
  starts, but QCN6122 caldata is not generated and Q6 later reports a fatal
  `err_smem_ver.2.1` crash from process `wlan1`. The remoteproc recovery then
  times out. `iw dev` remains empty and `wifi status` is empty.
- `re700x-qcn-only-mm2.itb` keeps the same QCN6122-only setup but changes only
  QCN6122 firmware memory mode from 1 to 2, matching the stock DTB's wireless
  memory-mode hint more closely. The FIT hash is
  `4357319d2dae990d192f5cbeb51057a7c0f932353f59182d30d0aa2f52c7969d`.
- Runtime test with `re700x-qcn-only-mm2.itb`: QCN6122 still crashes Q6 in the
  same way as memory mode 1. PD2 starts and reports `FW memory mode: 2`, then
  Q6 reports fatal `err_smem_ver.2.1` from process `wlan1`; remoteproc recovery
  times out. QCN6122 caldata is still not generated.
- `re700x-qcn-pd3-default.itb` is a diagnostic deviation from stock: IPQ5018
  and QCN PD2 are disabled, QCN is tested through `wifi2`/PD3 with the default
  OpenWrt PCIe0/PD3 mapping, `q6_region` is extended like existing triple-radio
  IPQ5018 devices, and the `b00b040` QCN6122 caldata hotplug case is added for
  `tplink,re700x`. The FIT hash is
  `ac6df20f57cfefebcc6250b5b4ea557f6992492d5922e3c80df69f6ecd167b9c`.
- Runtime test with `re700x-qcn-pd3-default.itb`: PD3 also fails with the same
  Q6 watchdog pattern. The log shows `b00b040.wifi`, userpd 3, and
  `FW memory mode: 1`; then Q6 reports fatal `err_smem_ver.2.1` from process
  `wlan2`/`TIMER_CLIENT_3`, followed by remoteproc recovery timeout. This rules
  out a simple PD2-vs-PD3 mapping mistake as the primary QCN6122 blocker.
- Current baseline after the QCN isolation tests is IPQ5018-only Wi-Fi: the
  internal IPQ5018 radio is enabled with firmware memory mode 1, while QCN6122
  remains disabled. This preserves the known-good RAM-boot and Ethernet state
  and avoids repeated Q6 crashes during normal bring-up tests.
- `re700x-ipq5018-baseline.itb` is the restored IPQ5018-only RAM-boot baseline
  after QCN isolation testing. The FIT hash is
  `1f8d9d3d9e6f7396518eb69ccdc5c40f2eb649b4b51d0cab88ef68ddbcf53d70`.
- Runtime test with `re700x-ipq5018-baseline.itb`: Ethernet/DHCP works on
  `br-lan`, the TFTP host is reachable, IPQ5018 `board-2.bin` and
  `cal-ahb-c000000.wifi.bin` are present, ath11k starts `c000000.wifi` on
  userpd 1 with firmware memory mode 1, and `wifi status` reports `radio0`
  up. No Q6 fatal error is observed in this baseline.
- Extended runtime validation with `re700x-ipq5018-baseline.itb`: `/proc/mtd`
  exposes the expected 16 SMEM partitions, `/proc/device-tree/model` reports
  `TP-Link RE700X`, compatible strings are `tplink,re700x` and `qcom,ipq5018`,
  `factory_data` mounts read-only as UBIFS, and the factory default MAC is
  applied to both `br-lan` and `lan`. LEDs enumerate as `blue:wps`,
  `green:power`, `green:wlan2g`, `green:wlan5g`, and `red:wps`. `netifd`
  obtains a DHCP lease on `lan` through `br-lan`; the only observed Wi-Fi
  userspace warning is `command failed: Not supported (-95)` during antenna
  configuration, after which `radio0` remains up.

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
- `ipq-wifi-tplink_re700x` uses board name
  `bus=ahb,qmi-chip-id=0,qmi-board-id=255,variant=TP-Link-RE700X` for both
  generated board-2 containers.

## Open Items

- Validate Wi-Fi boot logs from `re700x-board2-test1.itb`.
- Confirm generated caldata files and ath11k MAC handling without printing
  real MAC addresses.
- Validate isolated Wi-Fi boot logs from `re700x-ipq5018-only-mm1.itb`.
- Validate dual-radio memory-mode-1 boot logs from `re700x-both-mm1.itb`.
- Validate stock-memory dual-radio boot logs from `re700x-stockmem-mm1.itb`.
- Check whether moving the QCN6122 BDF/M3 regions avoids the IPQ5018 firmware
  start timeout.
- Validate QCN6122-only boot logs from `re700x-qcn-only-mm1.itb`.
- Validate QCN6122-only memory-mode-2 boot logs from
  `re700x-qcn-only-mm2.itb`.
- Validate QCN6122 PD3/default-mapping boot logs from
  `re700x-qcn-pd3-default.itb`.
- QCN6122 remains blocked: PD2 mode 1, PD2 mode 2, and PD3 mode 1 all crash Q6
  before QCN caldata is requested.
- Continue from the validated IPQ5018-only baseline image
  `re700x-ipq5018-baseline.itb`.
- Decide later whether this target needs factory/sysupgrade image generation.
- Keep all tests RAM-boot-only until recovery and install paths are fully
  understood.
