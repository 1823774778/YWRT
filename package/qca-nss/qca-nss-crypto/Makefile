include $(TOPDIR)/rules.mk

PKG_NAME:=qca-nss-crypto
PKG_RELEASE:=1

PKG_SOURCE_URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-crypto.git
PKG_SOURCE_PROTO:=git
PKG_SOURCE_DATE:=2024-09-16
PKG_SOURCE_VERSION:=60e27b9
PKG_MIRROR_HASH:=74a7f587c9352449f92095cacdd2c014972b1dac0a87f7f840592815003c531f
QSDK_VERSION:=13.1
PKG_VERSION:=$(QSDK_VERSION).$(subst -,.,$(PKG_SOURCE_DATE))~$(PKG_SOURCE_VERSION)

PKG_BUILD_PARALLEL:=1

PKG_CONFIG_DEPENDS:= \
	CONFIG_TARGET_SUBTARGET \
	CONFIG_TARGET_qualcommax_ipq807x \
	CONFIG_TARGET_qualcommax_ipq60xx

include $(INCLUDE_DIR)/kernel.mk
include $(INCLUDE_DIR)/package.mk

define KernelPackage/qca-nss-crypto
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Cryptographic API modules
  DEPENDS:=@(TARGET_qualcommax_ipq807x||TARGET_qualcommax_ipq60xx) \
	+kmod-qca-nss-drv \
	+nss-eip-firmware
  TITLE:=Kernel driver for NSS crypto driver
ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-crypto),)
  FILES:=$(PKG_BUILD_DIR)/$(NSS_CRYPTO_DIR)/src/qca-nss-crypto.ko \
	$(PKG_BUILD_DIR)/$(NSS_CRYPTO_DIR)/tool/qca-nss-crypto-tool.ko
  AUTOLOAD:=$(call AutoLoad,33,qca-nss-crypto)
endif
endef

define KernelPackage/qca-nss-crypto/Description
This package contains a NSS crypto driver for QCA chipset
endef

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-crypto),)
NSS_CRYPTO_SUBTARGET:=$(subst ",,$(CONFIG_TARGET_SUBTARGET))
NSS_CRYPTO_DIR:=v2.0

EXTRA_CFLAGS += \
	-DCONFIG_NSS_DEBUG_LEVEL=4 \
	-I$(STAGING_DIR)/usr/include/qca-nss-crypto \
	-I$(STAGING_DIR)/usr/include/qca-nss-drv \
	-I$(PKG_BUILD_DIR)/$(NSS_CRYPTO_DIR)/include \
	-I$(PKG_BUILD_DIR)/$(NSS_CRYPTO_DIR)/src

MAKE_OPTS:= \
	NSS_CRYPTO_DIR=$(NSS_CRYPTO_DIR) \
	SoC=$(NSS_CRYPTO_SUBTARGET)

define Build/InstallDev
	$(INSTALL_DIR) $(1)/usr/include/qca-nss-crypto
	$(CP) $(PKG_BUILD_DIR)/$(NSS_CRYPTO_DIR)/include/* $(1)/usr/include/qca-nss-crypto
endef

define Build/Compile
	+$(KERNEL_MAKE) $(PKG_JOBS) \
		M="$(PKG_BUILD_DIR)" \
		EXTRA_CFLAGS="$(EXTRA_CFLAGS)" \
		$(strip $(MAKE_OPTS)) \
		modules
endef

else

define Build/Compile
	:
endef

define Build/Install
	:
endef

endif

$(eval $(call KernelPackage,qca-nss-crypto))
