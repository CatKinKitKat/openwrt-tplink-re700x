include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

define Device/Default
	PROFILES := Default
	KERNEL_LOADADDR := 0x41000000
	DEVICE_DTS = $$(SOC)-$(lastword $(subst _, ,$(1)))
	DEVICE_DTS_CONFIG := config@1
	DEVICE_DTS_DIR := ../dts
	IMAGES := sysupgrade.bin
	IMAGE/sysupgrade.bin = sysupgrade-tar | append-metadata
	IMAGE/sysupgrade.bin/squashfs :=
endef

define Device/FitImage
	KERNEL_SUFFIX := -uImage.itb
	KERNEL = kernel-bin | libdeflate-gzip | fit gzip $$(KDIR)/image-$$(DEVICE_DTS).dtb
	KERNEL_NAME := Image
endef

# Same as FitImage but bundles an initrd (the separate-initramfs cpio) into
# the FIT. Requires CONFIG_TARGET_ROOTFS_INITRAMFS_SEPARATE=y (otherwise
# Build/fit-its silently drops the -i flag and no initrd ends up in the FIT).
# Used by devices that need an early-userspace trampoline (e.g. dual-boot
# slot detection on tplink,re700x).
define Device/FitImageInitrd
	KERNEL_SUFFIX := -uImage.itb
	KERNEL = kernel-bin | libdeflate-gzip | fit gzip $$(KDIR)/image-$$(DEVICE_DTS).dtb with-initrd
	KERNEL_NAME := Image
endef

define Device/FitImageLzma
	KERNEL_SUFFIX := -uImage.itb
	KERNEL = kernel-bin | lzma | fit lzma $$(KDIR)/image-$$(DEVICE_DTS).dtb
	KERNEL_NAME := Image
endef

define Device/EmmcImage
	IMAGES += factory.bin
	IMAGE/factory.bin := append-rootfs | pad-rootfs | pad-to 64k
	IMAGE/sysupgrade.bin/squashfs := append-rootfs | pad-to 64k | sysupgrade-tar rootfs=$$$$@ | append-metadata
endef

define Device/UbiFit
	KERNEL_IN_UBI := 1
	IMAGES += factory.ubi
	IMAGE/factory.ubi := append-ubi
endef

include $(SUBTARGET).mk

$(eval $(call BuildImage))
