
## OpenWrt Development for Xiaomi RA82 AX3000 (Qualcomm IPQ5018)

This repository is a fork of the **official OpenWrt source tree**, dedicated to implementing and testing complete upstream support for the **Xiaomi RA82 AX3000** wireless router. The official repository serves as a mirror of `https://git.openwrt.org/openwrt/openwrt.git` and accepts Pull Requests (PRs) which are merged via staging trees,.

The RA82 AX3000 is built around the **Qualcomm IPQ5018** System-on-Chip (SoC),, utilizing the **armv7** architecture. The device is configured with 512MB of system memory, defined by a memory size of `0x20000000` starting at address `0x40000000`.

### Key Hardware Integration Details

The device tree source (DTS) file integrated here maps the necessary hardware components based on direct extraction from the device:

*   **SoC and Architecture:** The device uses the Qualcomm IPQ5018 model,.
*   **Flash Layout:** The device employs a NAND flash controller compatible with `qcom,ebi2-nandc-bam-v2.1.1`. The DTS defines the dual-root filesystem scheme typical for these devices. The primary partition is **`rootfs`** starting at offset `0xa80000`,, and the redundant partition is **`rootfs_1`** starting at offset `0x2e80000`,. Both partitions are sized `0x2400000`,,.
*   **Boot Configuration:** The kernel command line arguments specify booting using the UBI filesystem: **`ubi.mtd=rootfs root=mtd:ubi_rootfs rootfstype=squashfs rootwait`**,.
*   **Networking Hardware:** The primary switch component is configured using the compatible string `qcom,ess-switch-ipq50xx`.
*   **Peripherals (Buttons/LEDs):** Support for user controls is defined via GPIO mappings,. The **WPS button** is defined on `gpio38`,. System LEDs include **`led_sys_blue`** on `gpio26` and **`led_sys_yellow`** on `gpio27`,,.

### Relationship to Other Projects

This project aligns with and draws knowledge from established OpenWrt development for related hardware, specifically devices in the **Redmi AX3000 / Xiaomi CR880x / Xiaomi CR881x** family, which also use the IPQ5018 platform,.

### Status and Goal

The ultimate goal of this repository is to produce **clean, maintainable patches** that can be submitted as a Pull Request to the main OpenWrt repository, for eventual **upstream merging**.

---
### **Topics/Tags (for GitHub categorization):**

`openwrt` `ipq5018` `xiaomi` `ra82` `ax3000` `device-tree` `firmware` `networking`

![OpenWrt logo](include/logo.png)

OpenWrt Project is a Linux operating system targeting embedded devices. Instead
of trying to create a single, static firmware, OpenWrt provides a fully
writable filesystem with package management. This frees you from the
application selection and configuration provided by the vendor and allows you
to customize the device through the use of packages to suit any application.
For developers, OpenWrt is the framework to build an application without having
to build a complete firmware around it; for users this means the ability for
full customization, to use the device in ways never envisioned.

Sunshine!

## Download

Built firmware images are available for many architectures and come with a
package selection to be used as WiFi home router. To quickly find a factory
image usable to migrate from a vendor stock firmware to OpenWrt, try the
*Firmware Selector*.

* [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)

If your device is supported, please follow the **Info** link to see install
instructions or consult the support resources listed below.

## 

An advanced user may require additional or specific package. (Toolchain, SDK, ...) For everything else than simple firmware download, try the wiki download page:

* [OpenWrt Wiki Download](https://openwrt.org/downloads)

## Development

To build your own firmware you need a GNU/Linux, BSD or macOS system (case
sensitive filesystem required). Cygwin is unsupported because of the lack of a
case sensitive file system.

### Requirements

You need the following tools to compile OpenWrt, the package names vary between
distributions. A complete list with distribution specific packages is found in
the [Build System Setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)
documentation.

```
binutils bzip2 diff find flex gawk gcc-6+ getopt grep install libc-dev libz-dev
make4.1+ perl python3.7+ rsync subversion unzip which
```

### Quickstart

1. Run `./scripts/feeds update -a` to obtain all the latest package definitions
   defined in feeds.conf / feeds.conf.default

2. Run `./scripts/feeds install -a` to install symlinks for all obtained
   packages into package/feeds/

3. Run `make menuconfig` to select your preferred configuration for the
   toolchain, target system & firmware packages.

4. Run `make` to build your firmware. This will download all sources, build the
   cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen
   applications for your target system.

### Related Repositories

The main repository uses multiple sub-repositories to manage packages of
different categories. All packages are installed via the OpenWrt package
manager called `opkg`. If you're looking to develop the web interface or port
packages to OpenWrt, please find the fitting repository below.

* [LuCI Web Interface](https://github.com/openwrt/luci): Modern and modular
  interface to control the device via a web browser.

* [OpenWrt Packages](https://github.com/openwrt/packages): Community repository
  of ported packages.

* [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically
  focused on (mesh) routing.

* [OpenWrt Video](https://github.com/openwrt/video): Packages specifically
  focused on display servers and clients (Xorg and Wayland).

## Support Information

For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)

### Documentation

* [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
* [User Guide](https://openwrt.org/docs/guide-user/start)
* [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
* [Technical Reference](https://openwrt.org/docs/techref/start)

### Support Community

* [Forum](https://forum.openwrt.org): For usage, projects, discussions and hardware advise.
* [Support Chat](https://webchat.oftc.net/#openwrt): Channel `#openwrt` on **oftc.net**.

### Developer Community

* [Bug Reports](https://bugs.openwrt.org): Report bugs in OpenWrt
* [Dev Mailing List](https://lists.openwrt.org/mailman/listinfo/openwrt-devel): Send patches
* [Dev Chat](https://webchat.oftc.net/#openwrt-devel): Channel `#openwrt-devel` on **oftc.net**.

## License

OpenWrt is licensed under GPL-2.0
