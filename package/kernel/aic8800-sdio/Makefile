#
# LOCAL PACKAGE -- not upstream OpenWrt.
#
# Out-of-tree vendor driver for the AIC8800D80 SDIO WiFi chip found on the
# LB-Link BL-M8800DS2 module of the Radxa Cubie A5E.
#
# There is no mainline driver for this chip and there never will be, so the
# vendor fullMAC driver (binds cfg80211 directly, not mac80211) is the only
# option.  Source is pinned to the exact same radxa-pkg/aic8800 commit that
# /etc/nixos/machines/switchboard/hw-config/aic8800-sdio.nix uses, and the
# patch series in ./patches/ is debian/patches/series from that same commit,
# renumbered in series order.
#
# Kernel 6.18 note: the series has no 6.18 patch because 6.18 needs nothing
# beyond 6.17.  fix-linux-6.19-build.patch is guarded with
# "LINUX_VERSION_CODE <= KERNEL_VERSION(6,18,0)" and fix-linux-6.17-build.patch
# with "< KERNEL_VERSION(6,17,0)", so applying the whole series on 6.18 is
# correct.
#
# Structured after github.com/firtel-t/aic8800-sdio-openwrt.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=aic8800-sdio
PKG_VERSION:=5.0
PKG_RELEASE:=2

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/radxa-pkg/aic8800.git
PKG_SOURCE_VERSION:=7f42b22913b462ab6c658dfc075bae1dbfe9a71a
PKG_SOURCE_DATE:=2026-04-29
# The git-tarball hash is not stable enough to be worth pinning for an
# experiment; "skip" is honoured by include/download.mk.
PKG_MIRROR_HASH:=skip

PKG_LICENSE:=GPL-2.0
PKG_LICENSE_FILES:=LICENSE
PKG_BUILD_PARALLEL:=1

# The driver must be compiled against the mac80211 *backports* headers, not
# the in-tree kernel ones.  See the NOSTDINC_FLAGS comment below.
PKG_BUILD_DEPENDS:=mac80211
STAMP_CONFIGURED_DEPENDS:=$(STAGING_DIR)/usr/include/mac80211-backport/backport/autoconf.h

include $(INCLUDE_DIR)/package.mk

# Everything below lives under src/SDIO/... in the vendor tree.
AIC_SDIO_DIR:=$(PKG_BUILD_DIR)/src/SDIO/driver_fw
AIC_DRV_DIR:=$(AIC_SDIO_DIR)/driver/aic8800
AIC_FW_DIR:=$(AIC_SDIO_DIR)/fw/aic8800D80

# Must match CONFIG_AIC_FW_PATH as set by 010-fix-sdio-firmware-path.patch,
# otherwise the driver will not find its firmware and no aic_fw_path module
# parameter is being passed.
AIC_FW_TARGET:=/lib/firmware/aic8800_fw/SDIO/aic8800D80

define KernelPackage/aic8800-sdio
  SUBMENU:=Wireless Drivers
  TITLE:=AIC8800D80 SDIO WiFi driver
  URL:=https://github.com/radxa-pkg/aic8800
  DEPENDS:=+kmod-cfg80211 +kmod-mmc +aic8800-sdio-firmware
  FILES:= \
	$(AIC_DRV_DIR)/aic8800_bsp/aic8800_bsp.ko \
	$(AIC_DRV_DIR)/aic8800_fdrv/aic8800_fdrv.ko
  AUTOLOAD:=$(call AutoProbe,aic8800_bsp aic8800_fdrv)
endef

define KernelPackage/aic8800-sdio/description
 Vendor fullMAC cfg80211 driver for the AIC8800D80 SDIO WiFi/BT combo chip.

 Known driver limitations (carry these into /etc/config/wireless):
  * No ACS -- the channel MUST be pinned explicitly, "auto" hangs hostapd.
  * Do not set a country code in hostapd; use the cfg80211.ieee80211_regdom
    kernel parameter instead, otherwise hostapd stalls 90+ seconds.
  * Single BSS, no multi-SSID, no AP+STA concurrency.
  * One radio only: 2.4 GHz or 5 GHz, never both at once.
  * No WPA3/802.11w. Use WPA2-CCMP.
  * HT40 is broken in the driver; use SHORT-GI-20 only.
endef

define Package/aic8800-sdio-firmware
  SECTION:=firmware
  CATEGORY:=Firmware
  TITLE:=AIC8800D80 SDIO firmware
  URL:=https://github.com/radxa-pkg/aic8800
endef

define Package/aic8800-sdio-firmware/description
 Firmware blobs and configuration tables for the AIC8800D80 SDIO chip,
 installed into $(AIC_FW_TARGET).
endef

# NOTE: the vendor driver's own Makefiles set "EXTRA_CFLAGS" (the pre-2.6.36
# kbuild variable name). Kernel 6.18's scripts/Makefile.build has dropped
# EXTRA_CFLAGS support entirely (grep confirms no reference to it), so that
# variable is silently ignored -- none of the defines below, and critically
# not -Wno-error, ever reached the compiler. Kernel 6.18 also unconditionally
# sets a blanket -Werror for external module builds (see
# scripts/mod/empty.o's recorded command line), which promotes the vendor
# code's pre-existing -Wmissing-prototypes/-Wmissing-declarations warnings
# (rwnx_msg_tx.c etc. define several non-static functions with no prototype
# in a header) into hard errors. subdir-ccflags-y is still honoured (it is
# appended to KBUILD_CFLAGS at scripts/Makefile.build:42, i.e. after the
# blanket -Werror) and nothing in the vendor tree sets it, so passing it on
# the command line is safe and does not clobber any vendor "+=" flags the
# way overriding ccflags-y directly would.
EXTRA_CFLAGS:= \
	-DCONFIG_AIC8800_WLAN_SUPPORT=1 \
	-DAICWF_SDIO_SUPPORT \
	-Wno-error \
	-Wno-missing-prototypes \
	-Wno-missing-declarations

# CRITICAL -- do not remove.
#
# OpenWrt does not use the kernel's in-tree cfg80211.  CONFIG_CFG80211 is off
# in the kernel config and kmod-cfg80211 is built by package/kernel/mac80211
# from the openwrt/backports snapshot, which carries a *newer* wireless stack
# than the 6.18 kernel it is loaded into.  The two disagree about struct
# layout: backports' struct ieee80211_channel has an extra
# "unsigned long dfs_state_last_available" field, making it 72 bytes where the
# in-tree 6.18 definition is 64.
#
# Building this driver against the in-tree headers therefore produces a module
# whose idea of every cfg80211 struct is wrong.  Observed on hardware: phy0
# came up advertising a garbage channel list (only array indices 0 and 8 of
# rwnx_2ghz_channels landed on a real center_freq -- exactly the 72/64 = 9/8
# stride ratio -- and indices 2 and 10 reported "30 MHz", which is the
# max_power=30 field of the CHAN() macro read at the wrong offset), and any
# real radio operation (scan, or netifd's AP-mode setup) walked off the end of
# the mis-sized structs and hung the kernel hard enough to need the watchdog.
#
# The fix is OpenWrt's standard idiom for out-of-tree cfg80211 drivers: put
# the backports include paths in NOSTDINC_FLAGS, which scripts/Makefile.lib
# places *before* LINUXINCLUDE on the compiler command line, so <net/cfg80211.h>
# resolves to the backports copy that cfg80211.ko was actually built from.
# Compare package/kernel/rtl8812au-ct/Makefile, which does the same thing.
NOSTDINC_FLAGS:= \
	$(KERNEL_NOSTDINC_FLAGS) \
	-I$(AIC_DRV_DIR) \
	-I$(STAGING_DIR)/usr/include/mac80211-backport \
	-I$(STAGING_DIR)/usr/include/mac80211-backport/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211 \
	-I$(STAGING_DIR)/usr/include/mac80211/uapi \
	-include backport/backport.h

# CONFIG_PLATFORM_* are passed on the command line so they override the
# vendor Makefile's ":=" assignments deterministically.  The ALLWINNER
# platform branch in the vendor Makefile hardcodes Android SDK paths and is
# not what we want -- UBUNTU is the generic mainline-kernel branch.
# BTLPM is the Bluetooth low-power-mode module; not needed here.
AIC_MAKE_OPTS:= \
	$(KERNEL_MAKE_FLAGS) \
	M="$(AIC_DRV_DIR)" \
	KERNELDIR="$(LINUX_DIR)" \
	NOSTDINC_FLAGS="$(NOSTDINC_FLAGS)" \
	subdir-ccflags-y="$(EXTRA_CFLAGS)" \
	CONFIG_PLATFORM_UBUNTU=y \
	CONFIG_PLATFORM_ALLWINNER=n \
	CONFIG_AIC8800_BTLPM_SUPPORT=n

define Build/Compile
	+$(MAKE) -C "$(LINUX_DIR)" $(AIC_MAKE_OPTS) modules
endef

define Build/Install
endef

define Package/aic8800-sdio-firmware/install
	$(INSTALL_DIR) $(1)$(AIC_FW_TARGET)
	$(CP) $(AIC_FW_DIR)/* $(1)$(AIC_FW_TARGET)/
endef

$(eval $(call KernelPackage,aic8800-sdio))
$(eval $(call BuildPackage,aic8800-sdio-firmware))
