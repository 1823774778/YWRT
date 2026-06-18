#
# Copyright (C) 2006-2014 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk

PKG_NAME:=mkimage
PKG_VERSION:=2026.04

PKG_SOURCE:=u-boot-$(PKG_VERSION).tar.bz2
PKG_SOURCE_URL:= \
	https://github.com/u-boot/u-boot/archive/refs/tags \
	https://mirror.cyberbits.eu/u-boot \
	https://ftp.denx.de/pub/u-boot
PKG_HASH:=f81bc51f067f28f16b07294e81fffb7565b811f30c3c4bfada63c14229051ca9

HOST_BUILD_DIR:=$(BUILD_DIR_HOST)/u-boot-$(PKG_VERSION)

include $(INCLUDE_DIR)/host-build.mk

define Host/Configure
	$(MAKE) -C $(HOST_BUILD_DIR) \
		HOSTCFLAGS="$(HOST_CFLAGS)" \
		HOSTLDFLAGS="$(HOST_LDFLAGS)" \
		PKG_CONFIG_EXTRAARGS="--static" \
		V=$(if $(findstring c,$(OPENWRT_VERBOSE)),1) \
		tools-only_defconfig

	$(HOST_BUILD_DIR)/scripts/config \
		--file $(HOST_BUILD_DIR)/.config \
		--disable TOOLS_KWBIMAGE \
		--disable TOOLS_LIBCRYPTO \
		--disable TOOLS_MKEFICAPSULE
endef

define Host/Compile
	$(MAKE) -C $(HOST_BUILD_DIR) \
		HOSTCFLAGS="$(HOST_CFLAGS)" \
		HOSTLDFLAGS="$(HOST_LDFLAGS)" \
		PKG_CONFIG_EXTRAARGS="--static" \
		V=$(if $(findstring c,$(OPENWRT_VERBOSE)),1) \
		tools-only
endef

define Host/Install
	$(CP) $(HOST_BUILD_DIR)/tools/mkimage $(STAGING_DIR_HOST)/bin/
	$(CP) $(HOST_BUILD_DIR)/tools/mkenvimage $(STAGING_DIR_HOST)/bin/
endef

define Host/Clean
	rm -f $(STAGING_DIR_HOST)/bin/mkimage
	rm -f $(STAGING_DIR_HOST)/bin/mkenvimage
endef

$(eval $(call HostBuild))
