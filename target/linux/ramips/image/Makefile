# SPDX-License-Identifier: GPL-2.0-only
#
# Copyright (C) 2008-2011 OpenWrt.org

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

DEVICE_VARS += LOADER_TYPE LOADER_FLASH_OFFS LZMA_TEXT_START
DEVICE_VARS += NETGEAR_BOARD_ID NETGEAR_HW_ID
DEVICE_VARS += NETGEAR_ENC_MODEL NETGEAR_ENC_REGION NETGEAR_ENC_HW_ID_LIST NETGEAR_ENC_MODEL_LIST
DEVICE_VARS += BUFFALO_TAG_PLATFORM BUFFALO_TAG_VERSION BUFFALO_TAG_MINOR
DEVICE_VARS += SEAMA_SIGNATURE SEAMA_MTDBLOCK
DEVICE_VARS += SERCOMM_HWNAME SERCOMM_HWID SERCOMM_HWVER SERCOMM_SWVER
DEVICE_VARS += SERCOMM_PAD JCG_MAXSIZE

loadaddr-y := 0x80000000
loadaddr-$(CONFIG_TARGET_ramips_rt288x) := 0x88000000
loadaddr-$(CONFIG_TARGET_ramips_mt7621) := 0x80001000

ldrplatform-y := ralink
ldrplatform-$(CONFIG_TARGET_ramips_mt7621) := mt7621

ldrflashstart-y := 0x1c000000
ldrflashstart-$(CONFIG_TARGET_ramips_mt7621) := 0x1fc00000

LOADADDR := $(loadaddr-y)
LOADER_PLATFORM := $(ldrplatform-y)
LOADER_FLASH_START := $(ldrflashstart-y)

KERNEL_DTB = kernel-bin | append-dtb | lzma

define Build/jcg-header
	$(STAGING_DIR_HOST)/bin/jcgimage -v $(1) \
		$(if $(JCG_MAXSIZE), -m $$(($(call exp_units,$(JCG_MAXSIZE)))),) \
		-u $@ -o $@.new
	mv $@.new $@
endef

define Build/loader-common
	rm -rf $@.src
	$(MAKE) -C lzma-loader \
		PKG_BUILD_DIR="$@.src" \
		TARGET_DIR="$(dir $@)" LOADER_NAME="$(notdir $@)" \
		BOARD="$(BOARDNAME)" PLATFORM="$(LOADER_PLATFORM)" \
		LZMA_TEXT_START=$(LZMA_TEXT_START) \
		LOADADDR=$(LOADADDR) \
		SUBTARGET=$(SUBTARGET) \
		$(1) compile loader.$(LOADER_TYPE)
	mv "$@.$(LOADER_TYPE)" "$@"
	rm -rf $@.src
endef

define Build/loader-kernel
	$(call Build/loader-common,LOADER_DATA="$@")
endef

define Build/loader-okli-compile
	$(call Build/loader-common, \
		FLASH_START=$(LOADER_FLASH_START) \
		FLASH_OFFS=$(LOADER_FLASH_OFFS) \
		FLASH_MAX=0 \
	)
endef

define Build/append-loader-okli
	cat "$(KDIR)/loader-$(word 1,$(1)).$(LOADER_TYPE)" >> "$@"
endef

# combine kernel and rootfs into one image
# mkdlinkfw <type> <optional extra arguments to mkdlinkfw binary>
define Build/mkdlinkfw
	-$(STAGING_DIR_HOST)/bin/mkdlinkfw \
		-k $(IMAGE_KERNEL) \
		-r $(IMAGE_ROOTFS) \
		-o $@ \
		$(if $(DLINK_IMAGE_OFFSET), -O $(DLINK_IMAGE_OFFSET)) \
		-s $(DLINK_FIRMWARE_SIZE)
endef

define Build/mkdlinkfw-factory
	-$(STAGING_DIR_HOST)/bin/mkdlinkfw \
		-m $(DLINK_ROM_ID) -f $(DLINK_FAMILY_MEMBER) \
		-F $@ \
		-o $@.new \
		$(if $(DLINK_IMAGE_OFFSET), -O $(DLINK_IMAGE_OFFSET)) \
		-s $(DLINK_FIRMWARE_SIZE)
	mv $@.new $@
endef

define Build/mkdlinkfw-loader
	-$(STAGING_DIR_HOST)/bin/mkdlinkfw \
		-k $(KDIR)/loader-$(DEVICE_NAME).bin \
		-r $@ \
		-o $@.new \
		$(if $(DLINK_IMAGE_OFFSET), -O $(DLINK_IMAGE_OFFSET)) \
		-s $(DLINK_FIRMWARE_SIZE)
	mv $@.new $@
endef

define Build/netis-tail
	echo -n $(1) >> $@
	echo -n $(UIMAGE_NAME)-yun | $(MKHASH) md5 | \
		sed 's/../\\\\x&/g' | xargs echo -ne >> $@
endef

define Build/poray-header
	$(STAGING_DIR_HOST)/bin/mkporayfw $(1) -f $@ -o $@.new
	mv $@.new $@
endef

define Build/relocate-kernel
	rm -rf $@.relocate
	$(CP) ../../generic/image/relocate $@.relocate
	$(MAKE) -C $@.relocate KERNEL_ADDR=$(if $(1),$(1),$(KERNEL_LOADADDR)) CROSS_COMPILE=$(TARGET_CROSS)
	( \
		dd if=$@.relocate/loader.bin bs=32 conv=sync && \
		perl -e '@s = stat("$@"); print pack("V", @s[7])' && \
		cat $@ \
	) > $@.new
	mv $@.new $@
	rm -rf $@.relocate
endef

define Build/sercom-footer
	$(call Build/sercom-seal,-f)
endef

define Build/sercom-seal
	$(STAGING_DIR_HOST)/bin/mksercommfw \
		-i $@ \
		-b $(SERCOMM_HWID) \
		-r $(SERCOMM_HWVER) \
		-v $(SERCOMM_SWVER) \
		$(1)
endef

define Build/sign-dlink-ru
	sign_dlink_ru $@ $1 $2
	mv $@.new $@
endef

define Build/trx
	-$(STAGING_DIR_HOST)/bin/trx $(1) \
		-o $@ \
		-m $$(($(call exp_units,$(IMAGE_SIZE)))) \
		-f $(IMAGE_KERNEL) \
		-a 4 -f $(IMAGE_ROOTFS)
endef

define Build/uimage-padhdr
	uimage_padhdr $(if $(1),-l $(1)) -i $@ -o $@.new
	mv $@.new $@
endef

define Build/uimage-sgehdr
	uimage_sgehdr -i $@ -o $@.new -m $(DEVICE_MODEL) \
		-h $(DEVICE_VARIANT) -s V1.00000
	mv $@.new $@
endef

define Build/umedia-header
	fix-u-media-header -T 0x46 -B $(1) -i $@ -o $@.new && mv $@.new $@
endef

define Build/wrg-header
	-[ -f "$@" ] && \
	mkwrgimg -i $@ -d "/dev/mtdblock/2" -s $(1) -o $@.new && \
	mv $@.new $@
endef

define Device/Default
  PROFILES = Default
  BLOCKSIZE := 64k
  KERNEL := $(KERNEL_DTB) | uImage lzma
  KERNEL_LOADADDR := $(loadaddr-y)
  LZMA_TEXT_START := 0x81800000
  SOC := $(DEFAULT_SOC)
  DEVICE_DTS_DIR := ../dts
  DEVICE_DTS = $$(SOC)_$(1)
  NETGEAR_ENC_MODEL :=
  NETGEAR_ENC_REGION :=
  NETGEAR_ENC_HW_ID_LIST :=
  NETGEAR_ENC_MODEL_LIST :=
  IMAGES := sysupgrade.bin
  COMPILE :=
  sysupgrade_bin := append-kernel | append-rootfs | pad-rootfs
  IMAGE/sysupgrade.bin := append-kernel | append-rootfs | pad-rootfs | check-size | append-metadata
endef

define Device/netgear_sercomm_nor
  DEVICE_VENDOR := NETGEAR
  IMAGES += factory.img
  IMAGE/default := append-kernel | pad-to $$$$(BLOCKSIZE) | append-rootfs | \
	pad-rootfs
  IMAGE/sysupgrade.bin := $$(IMAGE/default) | check-size | append-metadata
  IMAGE/factory.img := pad-extra $$$$(SERCOMM_PAD) | $$(IMAGE/default) | \
	pad-to $$$$(BLOCKSIZE) | sercom-footer | pad-to 128 | \
	zip $$$$(SERCOMM_HWNAME).bin | sercom-seal
endef

define Device/seama
  SEAMA_MTDBLOCK := 2
  IMAGES += factory.bin

  # 64 bytes offset:
  # - 28 bytes seama_header
  # - 36 bytes of META data (4-bytes aligned)
  IMAGE/default := append-kernel | pad-offset $$$$(BLOCKSIZE) 64 | append-rootfs
  IMAGE/sysupgrade.bin := \
	$$(IMAGE/default) | seama | pad-rootfs | check-size | append-metadata
  IMAGE/factory.bin := \
	$$(IMAGE/default) | pad-rootfs -x 64 | seama | seama-seal | check-size
  SEAMA_SIGNATURE :=
endef

define Device/uimage-lzma-loader
  LOADER_TYPE := bin
  KERNEL/lzma-loader := kernel-bin | append-dtb | lzma | loader-kernel
  KERNEL := $$(KERNEL/lzma-loader) | uImage none
endef

define Device/seama-lzma-loader
  $(Device/seama)
  $(Device/uimage-lzma-loader)
  KERNEL := $$(KERNEL/lzma-loader) | relocate-kernel | lzma -a0
  KERNEL_INITRAMFS := $$(KERNEL/lzma-loader) | uImage none
endef

include $(SUBTARGET).mk

$(eval $(call BuildImage))
