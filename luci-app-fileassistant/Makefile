# From https://github.com/DarkDean89/luci-app-filebrowser
# From https://github.com/stuarthua/oh-my-openwrt/tree/master/stuart/luci-app-fileassistant
# This is free software, licensed under the Apache License, Version 2.0 .

include $(TOPDIR)/rules.mk

LUCI_TITLE:=LuCI support for Fileassistant
LUCI_DEPENDS:=+luci-base +luci-lib-nixio
PKG_LICENSE:=Apache-2.0
PKG_LICENSE_FILES:=LICENSE
LUCI_PKGARCH:=all
PKG_VERSION:=1.0
PKG_RELEASE:=3

ifeq ($(wildcard $(TOPDIR)/feeds/luci/luci.mk),)
include ../../luci.mk
else
include $(TOPDIR)/feeds/luci/luci.mk
endif

# call BuildPackage - OpenWrt buildroot signature
