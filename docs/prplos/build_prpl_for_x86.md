# Building prplOS for x86_64: A Step-by-Step Guide

**prplOS**, an open-source operating system based on OpenWRT, is designed for embedded devices like routers, gateways, and IoT hubs, offering modularity, security, and support for protocols like TR-369 (USP). Building prplOS for the **x86_64** architecture allows you to create a versatile firmware for generic PCs, virtual machines, or x86-based embedded systems. This guide walks you through setting up the build environment, configuring the x86_64 target, and generating a prplOS image, leveraging OpenWRT’s build system.

## Prerequisites

Before starting, ensure your host machine is ready to build prplOS. Ubuntu or Debian is recommended for compatibility with OpenWRT’s build tools.

### Hardware Requirements
- **CPU**: Multi-core processor (4+ cores recommended for faster compilation).
- **RAM**: 8 GB or more (16 GB preferred for parallel builds).
- **Storage**: 50 GB free disk space (for source code, toolchain, and output images).
- **Internet**: Stable connection for downloading source code and dependencies.

### Software Requirements
Install the necessary packages on Ubuntu/Debian:
```bash
sudo apt update && sudo apt upgrade
sudo apt install build-essential ccache ecj fastjar file g++ gawk \
    gettext git java-propose-classpath libelf-dev libncurses5-dev \
    libncursesw5-dev libssl-dev python3 python3-distutils python3-pyelftools \
    qemu-utils rsync subversion swig time unzip wget xsltproc zlib1g-dev
```

- **Optional**: Install `python3-setuptools` and `python3-pip` for additional tools.
- **Git Configuration**: Set up your Git identity:
  ```bash
  git config --global user.name "Your Name"
  git config --global user.email "your.email@example.com"
  ```

## Step 1: Downloading the prplOS Source
prplOS is typically hosted in a repository provided by the prpl Foundation or a vendor-specific fork. Since the exact repository may vary, check the prpl Foundation’s website or GitHub for the latest source.

1. **Clone the Repository**:
   ```bash
   git clone https://gitlab.com/prpl-foundation/prplos/prplos.git
   cd prplos
   ```
   *Note*: Replace `https://gitlab.com/prpl-foundation/prplos/prplos.git` with the actual prplOS repository URL if different.

2. **Checkout a Stable Version** (optional):
   - List available tags:
     ```bash
     git tag
     ```
   - Checkout a stable release (e.g., `vX.Y.Z`):
     ```bash
     git checkout vX.Y.Z
     ```
   - For the latest development version, stay on the `main` branch.

3. **Update the Repository**:
   ```bash
   git pull
   ```

## Step 2: Understanding the Source Tree
The prplOS source tree is based on OpenWRT’s structure, with additional packages and configurations for prpl-specific features (e.g., TR-369/USP support).

- **Key Directories**:
  - `bin/`: Output directory for compiled images.
  - `build_dir/`: Temporary build artifacts and toolchains.
  - `dl/`: Downloaded source tarballs.
  - `package/`: Package definitions (e.g., `ob-usp-agent` for TR-369).
  - `target/linux/x86/`: x86_64-specific configurations.
  - `feeds.conf.default`: Package feed definitions.
  - `config/`: Build configuration files.
  - `profile/`: 

## Step 3: Configuring Feeds
prplOS extends OpenWRT’s package ecosystem with prpl-specific feeds for features like USP.

1. **Edit Feeds Configuration**:
   - Copy the default feeds file:
     ```bash
     cp feeds.conf.default feeds.conf
     ```
   - Example `feeds.conf`:
     ```
     src-git packages https://github.com/openwrt/packages.git
     src-git luci https://github.com/openwrt/luci.git
     src-git routing https://github.com/openwrt/routing.git
     src-git prpl https://github.com/prplfoundation/prpl-packages.git
     ```
   - Add the prpl-specific feed if not included.

2. **Update and Install Feeds**:
   ```bash
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   ```

## Step 4: Configuring the x86_64 Target
Use prpl's `gen_config.py` to configure the build for x86_64.

1. **Run gen_config.py**:
   ```bash
   ./scripts/gen_config.py prpl x86_64
   ```

3. **Save Configuration**:
   - Save to `.config` and backup:
     ```bash
     cp .config config-backup
     ```

## Step 5: Building the Image
Compile the prplOS image for x86_64.

1. **Start the Build**:
   ```bash
   make -j$(nproc)
   ```
   - `-j$(nproc)` uses all CPU cores for faster compilation.
   - For verbose output (debugging):
     ```bash
     make V=s
     ```

2. **Build Output**:
   - Images are generated in `bin/targets/x86/64/`.
   - Example outputs:
     - `prplos-x86-64-combined-ext4.img.gz`: Bootable disk image.
     - `prplos-x86-64-combined-efi.iso`: ISO for UEFI systems.
     - `prplos-x86-64-initramfs-kernel.bin`: For testing.

3. **Build Time**:
   - Takes 30 minutes to several hours, depending on hardware and configuration.
   - First builds are slower due to toolchain compilation.

## Step 6: Testing the Image
Test the prplOS image in a virtual environment or on physical hardware.

### Using QEMU
1. **Install QEMU**:
   ```bash
   sudo apt install qemu-system-x86
   ```

2. **Run Image**:
   - For `initramfs` (testing):
     ```bash
     qemu-system-x86_64 -kernel bin/targets/x86/64/openwrt-x86-64-initramfs-kernel.bin \
       -nographic -m 512 -append "console=ttyS0"
     ```
   - For disk image:
     ```bash
     qemu-system-x86_64 -drive file=bin/targets/x86/64/openwrt-x86-64-combined-ext4.img,format=raw \
       -m 512 -netdev user,id=net0 -device virtio-net-pci,netdev=net0
     ```

3. **Access**:
   - Console: Via QEMU’s serial output.
   - SSH: `ssh root@192.168.1.1` (set password first with `passwd`).
   - LuCI: `http://192.168.1.1` (if `luci` is included).

### Flashing to Physical Hardware
1. **Prepare Image**:
   - Use the `.iso` or `.img` file from `bin/targets/x86/64/`.
   - Write to a USB drive:
     ```bash
     sudo dd if=openwrt-x86-64-combined-ext4.img of=/dev/sdX bs=4M status=progress
     ```
     Replace `/dev/sdX` with your USB device (use `lsblk` to identify).

2. **Boot**:
   - Configure the BIOS/UEFI to boot from the USB.
   - prplOS boots with default settings (e.g., IP `192.168.1.1`).

## Step 7: Configuring TR-369 (USP) Support
prplOS’s TR-369 support is a key feature for remote management. Configure the USP agent post-build.

1. **Install USP Agent**:
   - Ensure `ob-usp-agent` is included in the build (selected in `make menuconfig`).
   - If not, install via `opkg`:
     ```bash
     opkg update
     opkg install ob-usp-agent
     ```

2. **Configure USP**:
   - Edit `/etc/config/usp`:
     ```bash
     config usp
         option controller_url 'wss://acs.example.com'
         option endpoint_id 'prplos::x86_64_device'
         option mtp 'websocket'
         option enable_tls '1'
     ```
   - Apply: `/etc/init.d/usp restart`.

3. **Verify**:
   - Check status: `ubus call usp status`.
   - Monitor logs: `logread | grep usp`.

## Troubleshooting Build Issues
- **Missing Dependencies**:
  - Error: Missing tools/libraries.
  - Fix: Reinstall prerequisites (`sudo apt install ...`).
- **Disk Space**:
  - Error: Build fails due to insufficient space.
  - Fix: Free up space (`df -h`) or use a larger partition.
- **Feed Issues**:
  - Error: prpl-specific packages unavailable.
  - Fix: Verify `feeds.conf` and update feeds (`./scripts/feeds update -a`).
- **Compilation Errors**:
  - Error: Build fails for specific packages.
  - Fix: Enable verbose output (`make V=s`) and check logs in `build_dir/`.
  - Search the prpl Foundation forum or OpenWRT wiki for solutions.
- **QEMU Issues**:
  - Error: Image doesn’t boot.
  - Fix: Ensure correct image type (`initramfs` for testing) and QEMU parameters.

## Conclusion
Building prplOS for x86_64 enables you to deploy a powerful, OpenWRT-based firmware on versatile hardware, from virtual machines to physical PCs. With its support for TR-369 (USP), prplOS is ideal for modern broadband and IoT applications, offering secure and scalable remote management. This guide provides a foundation for building and testing prplOS, leveraging OpenWRT’s robust build system. For further details, check the [prpl Foundation](https://prplfoundation.org/) or [OpenWRT documentation](https://openwrt.org/docs/guide-developer/build-system/start).

Happy building, and enjoy deploying prplOS on x86_64!