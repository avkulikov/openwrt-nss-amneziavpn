# OpenWrt AX3600 NSS + AmneziaWG Builder Assets

This repository contains the builder assets used by GitHub Actions to produce
OpenWrt images for the Xiaomi AX3600 (Qualcommax/IPQ807x) with NSS and AmneziaWG.

## Build and releases

- Workflow: `.github/workflows/build.yaml`
- Trigger: scheduled checks or manual "Run workflow"
- Release assets:
  - Firmware images:
    - first install: `openwrt-qualcommax-ipq807x-xiaomi_ax3600-*-factory.ubi`
    - upgrade: `openwrt-qualcommax-ipq807x-xiaomi_ax3600-*-sysupgrade.bin`
  - AmneziaWG packages: `kmod-amneziawg`, `amneziawg-tools`,
    `luci-proto-amneziawg` (`*.ipk` or `*.apk`, depending on upstream)
- Local build output (if you build manually):
  `openwrt/bin/targets/qualcommax/ipq807x/`

## Flashing Xiaomi AX3600

### First install (factory image)

Use the `*-factory.ubi` image when flashing from the stock firmware or a
recovery method. If your OEM UI supports firmware upload, select the factory
image there. If you use a recovery tool (for example TFTP), use the same
factory image.

After the first boot, OpenWrt defaults to `192.168.1.1`.

### Upgrade (sysupgrade)

Use the `*-sysupgrade.bin` image when the device already runs OpenWrt.

- LuCI: System -> Backup / Flash Firmware -> Flash image
- CLI:
  - `scp openwrt-*-sysupgrade.bin root@192.168.1.1:/tmp/`
  - `sysupgrade /tmp/openwrt-*-sysupgrade.bin`

## AmneziaWG

The build includes AmneziaWG packages. If you need to reinstall them from the
release assets, copy the package files (`.ipk` or `.apk`) to the router and install.

- If you have `.apk` packages (apk-based OpenWrt):
  - `apk add /tmp/kmod-amneziawg_*.apk /tmp/amneziawg-tools_*.apk /tmp/luci-proto-amneziawg_*.apk`
- If you have `.ipk` packages (opkg-based OpenWrt):
  - `opkg install /tmp/kmod-amneziawg_*.ipk /tmp/amneziawg-tools_*.ipk /tmp/luci-proto-amneziawg_*.ipk`

Enable the interface in LuCI:

- Network -> Interfaces -> Add new interface
- Protocol: `AmneziaWG`
- Import or paste your config (keys, addresses, peers), then Apply

Make sure your firewall allows the UDP port used by AmneziaWG.
