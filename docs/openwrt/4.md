# Running OpenWRT on Virtual and Physical Devices

This guide covers running OpenWRT in a virtual environment using QEMU and on physical devices, including installation, access, and recovery procedures.

## Introduction to QEMU for Embedded Emulation

QEMU is an open-source emulator and virtualizer that supports various architectures, making it ideal for testing OpenWRT on virtual hardware without physical devices.

### Purpose:
- Emulate embedded architectures (e.g., MIPS, ARM) to test OpenWRT builds.
- Debug firmware or configurations in a controlled environment.
- Simulate router hardware for development or prototyping.

### Advantages:
- No physical hardware required.
- Safe environment for testing experimental builds.
- Supports multiple architectures (e.g., malta for MIPS, virt for ARM).

### Limitations:
- Limited emulation of specific hardware features (e.g., Wi-Fi radios, switch chips).
- Performance may not match physical devices.
- Requires correct QEMU target and image type (e.g., initramfs).

## Preparing the QEMU Target for OpenWRT

To run OpenWRT in QEMU, you need to prepare the build and QEMU environment.

### Prerequisites:
- QEMU installed on the host (Ubuntu/Debian):
  ```bash
  sudo apt install qemu-system-mips qemu-system-arm
  ```
- OpenWRT source code and build environment (see Module 3).
- A QEMU-compatible target (e.g., malta for MIPS, virt for ARM).

### Steps:
- Configure OpenWRT for QEMU:
  - Run `make menuconfig`.
  - Select a QEMU-compatible target:
    - **Target System**: MIPS Malta (for MIPS) or ARMvirt (for ARM).
    - **Subtarget**: Generic (if applicable).
    - **Target Profile**: QEMU or Generic.
  - Enable necessary packages (e.g., `luci` for web GUI, `kmod-virtio` for virtual I/O).
- Build an initramfs image for QEMU:
  - Enable **Target Images > ramdisk** in `make menuconfig`.
  - Build the image:
    ```bash
    make -j$(nproc)
    ```
- Locate the Image:
  - Find the initramfs image in `bin/targets/<target>/<subtarget>/` (e.g., `openwrt-malta-be-initramfs-kernel.bin`).
- Prepare QEMU Parameters:
  - Choose the architecture (e.g., `qemu-system-mips` for Malta).
  - Allocate RAM (e.g., 128 MB) and CPU cores.
  - Configure virtual network interfaces (e.g., `virtio-net` for networking).

### Notes:
- The `malta` target is widely used for MIPS-based testing.
- Ensure the image matches the QEMU architecture.

## Installing and Booting OpenWRT in QEMU

Run OpenWRT in QEMU using the built initramfs image.

### Basic QEMU Command (MIPS Malta example):
```bash
qemu-system-mips -kernel bin/targets/malta/be/openwrt-malta-be-initramfs-kernel.bin \
  -nographic -m 128 -M malta -append "console=ttyS0"
```
- `-kernel`: Specifies the OpenWRT initramfs image.
- `-nographic`: Uses console output (no GUI).
- `-m 128`: Allocates 128 MB RAM.
- `-M malta`: Sets the machine type.
- `-append "console=ttyS0"`: Configures the kernel console.

### Networking Setup (optional):
- Add a virtual network interface:
  ```bash
  qemu-system-mips -kernel bin/targets/malta/be/openwrt-malta-be-initramfs-kernel.bin \
    -nographic -m 128 -M malta -append "console=ttyS0" \
    -netdev user,id=net0 -device virtio-net-device,netdev=net0
  ```
  This enables a user-mode network for testing (e.g., DHCP, SSH access).

### Booting:
- QEMU boots the initramfs image, loading OpenWRT into RAM.
- Access the console via the terminal (serial console).
- Configure networking (e.g., via `uci` or LuCI) if needed.

### Tips:
- Use `-snapshot` to avoid modifying the image.
- For GUI access, install `luci` and access via a browser (requires network setup).
- Check the OpenWRT wiki for target-specific QEMU parameters.

## Flashing Images to Physical Devices (U-Boot, TFTP, Serial)

Flashing OpenWRT to physical devices replaces the stock firmware. Methods vary by device.

### Preparation:
- Identify the device model and confirm OpenWRT support (check the OpenWRT Table of Hardware).
- Download or build the correct image (`factory` for initial flash, `sysupgrade` for updates).
- Backup the stock firmware (if possible) to restore in case of issues.

### Flashing Methods:
- **Web Interface (Vendor Firmware)**:
  - Access the router’s web interface (e.g., `http://192.168.1.1`).
  - Navigate to the firmware upgrade section.
  - Upload the factory image (e.g., `openwrt-xxx-factory.bin`).
  - Wait for the device to reboot with OpenWRT.
  - **Note**: Some devices require specific image formats or headers.
- **U-Boot (Bootloader)**:
  - Access the bootloader via serial console or by interrupting the boot process (e.g., pressing a key during startup).
  - Use U-Boot commands to load and flash the image:
    ```bash
    tftpboot 0x80060000 openwrt-xxx-factory.bin
    erase 0x9f050000 +0x7b0000
    cp.b 0x80060000 0x9f050000 0x7b0000
    boot
    ```
  - Requires a TFTP server (see below) and correct memory addresses (device-specific).
- **TFTP (Trivial File Transfer Protocol)**:
  - Set up a TFTP server on the host:
    ```bash
    sudo apt install tftpd-hpa
    sudo systemctl start tftpd-hpa
    cp openwrt-xxx-factory.bin /var/lib/tftpboot/
    ```
  - Boot the device into TFTP recovery mode (often via a reset button or bootloader).
  - Configure the device to download the image from the TFTP server (IP and filename vary by device).
  - Example (U-Boot):
    ```bash
    setenv ipaddr 192.168.1.1
    setenv serverip 192.168.1.100
    tftpboot 0x80060000 openwrt-xxx-factory.bin
    bootm
    ```
- **Serial Console**:
  - Connect a USB-to-serial adapter to the device’s serial port (requires hardware modification on some devices).
  - Use a terminal emulator (e.g., `minicom`, `picocom`):
    ```bash
    sudo minicom -D /dev/ttyUSB0 -b 115200
    ```
  - Interrupt the bootloader and flash via TFTP or direct image transfer.

### Device-Specific Notes:
- Check the OpenWRT wiki for device-specific instructions (e.g., TP-Link, Netgear).
- Verify flash partition layouts to avoid bricking.
- Ensure the correct image type (`factory` vs. `sysupgrade`).

## Console Access and SSH Login

Accessing OpenWRT for configuration or debugging is done via console or SSH.

### Console Access:
- **Serial Console**:
  - Connect a USB-to-serial adapter to the device’s serial port.
  - Use a terminal emulator:
    ```bash
    sudo minicom -D /dev/ttyUSB0 -b 115200
    ```
  - Log in (default: no password for `root` on first boot).
- **QEMU Console**:
  - If running in QEMU with `-nographic`, the terminal acts as the console.
  - For graphical QEMU, use the virtual serial console or network-based access.

### SSH Login:
- Ensure the device has an IP address (default: `192.168.1.1`).
- Enable SSH (included by default with `dropbear`):
  ```bash
  ssh root@192.168.1.1
  ```
- On first login, set a root password to enable SSH:
  ```bash
  passwd
  ```
- If LuCI is installed, access the web interface at `http://192.168.1.1`.

### Tips:
- Configure network settings via `/etc/config/network` if SSH is inaccessible.
- Use `uci` commands for CLI configuration (e.g., `uci set network.lan.ipaddr='192.168.1.1'`).

## Recovering from Failed Flash / Soft Bricking

A soft-bricked device (firmware fails to boot but bootloader is intact) can often be recovered.

### Symptoms:
- Device fails to boot OpenWRT or becomes unresponsive.
- Bootloader (e.g., U-Boot) is still accessible.
- LEDs indicate abnormal behavior (e.g., constant blinking).

### Recovery Methods:
- **Bootloader Recovery (U-Boot/TFTP)**:
  - Access the bootloader via serial console or button press.
  - Use TFTP to load a working initramfs or factory image:
    ```bash
    tftpboot 0x80060000 openwrt-xxx-initramfs-kernel.bin
    bootm
    ```
  - Flash a new sysupgrade image from the running system:
    ```bash
    sysupgrade -v openwrt-xxx-sysupgrade.bin
    ```
- **Safe Mode/Failsafe Mode**:
  - Many devices support a failsafe mode triggered during boot (check device wiki).
  - Example: Press a reset button during boot to enter failsafe.
  - Access via `telnet` (not SSH):
    ```bash
    telnet 192.168.1.1
    ```
  - Restore a working image or reset configurations:
    ```bash
    sysupgrade -v openwrt-xxx-sysupgrade.bin
    ```
- **Serial Console Recovery**:
  - Use a serial connection to interact with the bootloader.
  - Load and flash a new image via TFTP or direct transfer.
- **Revert to Stock Firmware**:
  - If a backup of the stock firmware exists, flash it via TFTP or the bootloader.
  - Check the OpenWRT wiki for vendor-specific recovery procedures.

### Prevention Tips:
- Always verify image compatibility before flashing.
- Backup the stock firmware and configurations.
- Test builds in QEMU before flashing physical devices.
- Avoid interrupting the flash process.

### Hard Brick Warning:
- If the bootloader is corrupted (hard brick), recovery may require JTAG or hardware reprogramming, which is complex and device-specific.
- Consult the OpenWRT forum or device documentation for advanced recovery.