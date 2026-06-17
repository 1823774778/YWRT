include $(TOPDIR)/rules.mk

PKG_NAME:=qca-nss-phy
PKG_RELEASE:=1

PKG_SOURCE_URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/qca-nss-phy.git
PKG_SOURCE_PROTO:=git
PKG_SOURCE_DATE:=2026-01-11
PKG_SOURCE_VERSION:=85cb19ff9c3905851b5452c7558cc3d94261be4f
PKG_MIRROR_HASH:=7bc4a20a85e5578b44f8f4e6a0f9d48de9faf404d5635eec3fdbeb130f74510e

PKG_FLAGS:=nonshared

include $(INCLUDE_DIR)/package.mk

define Package/qca-nss-phy
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Qualcomm NSS PHY headers
  URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/qca-nss-phy
  DEPENDS:=@TARGET_qualcommax
  BUILDONLY:=1
endef

define Package/qca-nss-phy/Description
  Build-only package exporting QSDK NSS PHY headers for SSDK consumers.
endef

define Build/Compile
endef

define Build/InstallDev
	$(INSTALL_DIR) $(1)/usr/include/qca-nss-phy
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/nss_ext/*.h $(1)/usr/include/qca-nss-phy/
endef

$(eval $(call BuildPackage,qca-nss-phy))
