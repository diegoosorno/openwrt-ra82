# OpenWrt Support for Xiaomi Mesh System AX3000 (Model: RA82)

> **Status: BLOCKED BY HARDWARE SECURE BOOT (Project Conclusion)**  
> Comprehensive research and reverse-engineering of the boot chain confirmed that the **Xiaomi Mesh System AX3000 (RA82)** European/Global production models have **Qualcomm Secure Boot permanently enabled in hardware eFuses**. Standard OpenWrt images cannot be booted unattended without manufacturer cryptographic signatures.

---

## Hardware Specifications

| Component | Specification |
| :--- | :--- |
| **Model** | Xiaomi Mesh System AX3000 (Model: `RA82`, Part: `DVB4315GL`) |
| **SoC** | Qualcomm IPQ5018 (2x ARM Cortex-A53 @ 1.0 GHz) |
| **RAM** | 256 MB DDR3L |
| **Flash** | 128 MB SPI-NAND (`GD5F1GQ5UEYIG` / Winbond `W25N01KV`) |
| **Switch** | Qualcomm Atheros QCA8337 Gigabit Switch |
| **2.4 GHz Wi-Fi** | Qualcomm IPQ5018 internal radio (2x2 802.11ax, up to 574 Mbps) |
| **5 GHz Wi-Fi** | Qualcomm QCN6122 via PCIe (2x2 802.11ax, 160 MHz, up to 2402 Mbps) |
| **UART Interface** | 1.8V TTL Logic Only! (Baud: 115200, 8N1) |

### Reverse-Engineered Hardware Pinout & GPIO Mapping

Through DTS analysis and validation against stock firmware binaries:

- **UART Serial**:
  - `TX`: GPIO 20 (`blsp0_uart0_tx`)
  - `RX`: GPIO 21 (`blsp0_uart0_rx`)
  - *Warning*: Uses **1.8V logic levels only**. Connecting 3.3V or 5V USB-TTL adapters can permanently damage the SoC.
- **Ethernet Switch (QCA8337)**:
  - Reset Line: GPIO 39 (`active-low`)
  - MDIO Bus: `mdio@0x39c00000`, Switch MDIO address `0x11` (internal GMAC PHY address `7`)
- **LEDs**:
  - System Blue: GPIO 24 (`active-high`)
  - System Orange: GPIO 25 (`active-high`)
  - Internet Blue: GPIO 26 (`active-high`)
  - Internet Orange: GPIO 27 (`active-high`)
- **Buttons**:
  - Reset Button: GPIO 38 (`active-low`)
  - Mesh / WPS Button: GPIO 23 (`active-low`)

---

## Completed Porting Work

This branch (`support-xiaomi-ra82`) contains a complete OpenWrt target implementation:
1. **Device Tree**: `target/linux/qualcommax/files/arch/arm64/boot/dts/qcom/ipq5018-xiaomi-ra82.dts`
   - Complete pinmux and TLMM configurations.
   - Dual A/B UBI partition map (`rootfs` @ `0xa80000`, `rootfs_1` @ `0x2e80000`).
   - Switch definitions with SGMII interface and port mappings.
2. **Build Definitions**: `target/linux/qualcommax/image/ipq50xx.mk`
   - Image profile generating `initramfs-factory.ubi`, `factory.ubi`, and `sysupgrade.bin`.
   - FIT configuration tag aligned with OEM bootloader expectations (`DEVICE_DTS_CONFIG := config@mp03.1`).

---

## Technical Findings & Blocker Conclusion

### 1. The Qualcomm Secure Boot Mechanism
Stock kernel partitions (`mtd18`/`mtd19`/`mtd22`) are not plain FIT images; they are encapsulated inside a 40-byte **Qualcomm MBN (Multi-Boot-Native) container**:
- **Offset `0x00 - 0x27`**: Qualcomm MBN header (`Image Type = 0x17 / APPS_KERNEL`).
- **Offset `0x28 - 0x3502d3`**: Inner FIT image (Kernel + FDT).
- **Offset `0x3502d4+`**: RSA-2048 digital signature and an X.509 certificate chain rooted in Xiaomi’s OEM hardware public key hash.

### 2. Bootloader Authentication (`mtd12_0APPSBL.bin`)
Disassembly of Xiaomi's U-Boot bootloader (`board/qca/arm/common/cmd_bootmiwifi.c`) revealed the enforced boot decision tree:

```arm
0x4a922d30: movs     r3, #1
0x4a922d32: add.w    r2, sp, #7         ; &sp[7] (buffer for TrustZone response)
0x4a922d36: movs     r1, #7             ; cmd_id = 7 (SECURE_BOOT_IS_ENABLED)
0x4a922d38: movs     r0, #8             ; svc_id = 8 (Qualcomm SCM security service)
0x4a922d3a: bl       scm_call           ; Call TrustZone (QSEE)
0x4a922d46: ldrb.w   r3, [sp, #7]       ; Read result
0x4a922d4a: cmp      r3, #1             ; Is Secure Boot blown in hardware eFuses?
0x4a922d4c: bne      do_boot_unsignedimg; If NOT blown -> boot unsigned FIT
0x4a922d52: bl       do_boot_signedimg  ; If BLOWN -> enforce RSA signature verification
```

On this European/Global model (RA82):
- Qualcomm QFPROM Secure Boot fuses are blown at the factory (`flag_boot_type=2` in NVRAM).
- TrustZone returns `1`, causing U-Boot to always route execution into `do_boot_signedimg` (`0x4a922870`).
- `do_boot_signedimg` calls QSEE (`scm_call(8, 8)`) to cryptographically verify the RSA-2048 signature of the kernel volume.
- OpenWrt images lack Xiaomi's private RSA signature. Authentication fails with:
  ```text
  Kernel image authentication failed 
  BUG: failure at board/qca/arm/common/cmd_bootmiwifi.c:163/do_boot_signedimg()!
  ```
- U-Boot increments the boot failure counter and drops into **TFTP recovery mode (slow blinking orange LED)**.

### 3. Why Domestic Chinese Devices (Redmi AX3000 / CR880x) Boot OpenWrt
Community ports for Redmi AX3000 and CR8806/CR8808 succeed because Xiaomi left the Secure Boot fuses unblown on those production batches. On those devices, `scm_call(8, 7)` returns `0`, and U-Boot executes `do_boot_unsignedimg`, booting unsigned FIT images seamlessly via `bootm`.

### 4. Technical Constraints
- **Signed Bootloader**: `0:APPSBL` (U-Boot) is itself signed and verified by Qualcomm SBL1. Modifying U-Boot in flash to patch the authentication check fails SBL1 verification and results in a permanent hard brick (Qualcomm EDL 9008 mode).
- **Hardcoded Autoboot**: In stock U-Boot, `main_loop` is invoked with `"bootmiwifi"` hardcoded in register `r0`; setting `bootcmd` in NVRAM has no effect on autoboot.
- **Interactive UART Bypass Only**: If autoboot is interrupted manually over a 1.8V UART connection (`Hit any key to stop autoboot: 3`), the standard U-Boot `bootm` command can boot an unsigned image from memory (`bootm 0x44000000#config@mp03.1`). However, this cannot boot unattended on power cycle.

Due to the permanent hardware signature enforcement, development on this firmware target is concluded and closed.

---

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
make4.1+ perl python3.8+ rsync subversion unzip which
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
